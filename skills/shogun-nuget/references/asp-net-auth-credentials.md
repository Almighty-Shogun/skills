# AlmightyShogun.AspNet.Auth.Credentials

Username and password accounts on top of `AspNet.Auth`: login, registration,
refresh sessions with rotation and reuse detection, password change and reset,
lockout, and TOTP two-factor with recovery codes. Storage is EF Core.

Docs: https://nuget.docs.shogun.ms/asp-net-auth-credentials/

Depends on `AlmightyShogun.Utils`, `AspNet.Core`, `AspNet.Auth`,
`AspNet.Localization`, `AspNet.RequestValidation`, EF Core (core and relational),
and `Otp.NET`. It supplies no EF Core provider and no migrations.

## What is where

| Need | Reach for |
| --- | --- |
| Register the services | `AddAuthCredentials<TDbContext, TUser>(configuration)` |
| Log in | `IAuthUserService<TUser>.LoginAsync(request, httpContext)` |
| Register a user and log them in | `IAuthUserService<TUser>.RegisterAsync(user, password, httpContext)` |
| Create a user without a session | `IAuthUserService<TUser>.CreateUserAsync(user, password)` |
| Rotate a refresh token | `IAuthSessionService<TUser>.RefreshSessionAsync(token, httpContext)` |
| Log out one session | `IAuthSessionService<TUser>.RevokeSessionAsync(token)` |
| Create a session yourself | `IAuthSessionService<TUser>.CreateSessionAsync(user, app, clientContext)` |
| Change a password | `IAuthPasswordService.ChangePasswordAsync(identifier, request, currentRefreshToken?)` |
| Start a reset | `IAuthPasswordService.RequestForgotPasswordAsync(request, ip?)` |
| Finish a reset | `IAuthPasswordService.CompleteForgotPasswordAsync(request)` |
| Enrol in two-factor | `IAuthTwoFactorService<TUser>.BeginEnrolmentAsync` then `CompleteEnrolmentAsync` |
| Check a code or recovery code | `IAuthTwoFactorService<TUser>.VerifyAsync(identifier, code)` |
| Hash a token the way the package does | `TokenHasher.Hash(token)` |

## Model

Derive both types, then register:

```csharp
public sealed class AppUser : AuthUser
{
    public string? DisplayName { get; set; }
}

public sealed class AppDbContext(DbContextOptions options) : AuthDbContext<AppUser>(options);

builder.Services
    .AddDbContext<AppDbContext>(options => options.UseNpgsql(connectionString))
    .AddAuthCredentials<AppDbContext, AppUser>(builder.Configuration);
```

`AuthDbContext<TUser>.OnModelCreating` configures the whole schema: unique
indexes on `Identifier`, `Username`, `Email`, refresh-token and reset-token
hashes, cascade deletes from the user, and a concurrency token on `UserSession`.
Call `base.OnModelCreating(modelBuilder)` from any override.

Tables are snake case: `users`, `user_sessions`, `password_reset_tokens`,
`email_verification_tokens`, `user_two_factors`, `user_lockouts`,
`two_factor_recovery_codes`.

`AuthUser` carries `Id` (int key), `Identifier` (Guid v7, the value that goes
into the token), `Username`, `Email`, `Password` (hash), `Role`, `Permissions`,
`IsActive`, and the `Sessions`, `Lockout` and `TwoFactor` navigations.

## Sessions

`CreateSessionAsync` returns a raw refresh token, stores only its SHA-256 hex
hash, records IP, user agent, browser, OS and device, and deletes that user's
already expired sessions first.

`RefreshSessionAsync` rotates: it looks the session up by hash, moves the old
hash to `PreviousRefreshTokenHash`, writes a new one, refreshes the metadata,
extends the expiry by `Auth:RefreshTokenDays` capped at
`AuthCredentials:AbsoluteSessionLifetimeDays` from `CreatedAt`, and bumps the
concurrency token. Two concurrent refreshes mean one loses on
`DbUpdateConcurrencyException` and gets `InvalidSessionException`.

Presenting a token that was already rotated away is treated as reuse: every
non-revoked session for that user is revoked and a warning is logged, unless the
rotation happened within the 30-second grace window, which absorbs a retry.

Permissions are scoped to the app when one resolves: only claims prefixed
`app:` survive, with the prefix stripped.

## Task recipes

### Log in

```csharp
AuthSessionResult<AppUser> result = await userService.LoginAsync(request, HttpContext, cancellationToken);

Response.SetRefreshTokenCookie(result.RefreshToken, authSettings.RefreshTokenDays);

return Ok(new { result.AccessToken });
```

`LoginRequest.Identifier` matches either the username or the email.

### Password reset

```csharp
string? token = await passwordService.RequestForgotPasswordAsync(request, HttpContext.GetIpAddress(), cancellationToken);

if (token is not null)
    await mailService.SendAsync(request.Email, new ResetMail(token));
```

The returned token is raw and is the only time you see it; only its hash is
stored. One active reset token exists per user, and requesting again overwrites
it. Completing a reset revokes every session for that user.

### Two-factor enrolment

```csharp
AuthTwoFactorResult begin = await twoFactorService.BeginEnrolmentAsync(identifier, "My App", cancellationToken);
// show begin.Uri as a QR code

IReadOnlyList<string> recoveryCodes = await twoFactorService.CompleteEnrolmentAsync(identifier, code, cancellationToken);
// show recoveryCodes once
```

`CompleteEnrolmentAsync` replaces any existing recovery codes and returns the new
ones in the clear; only their hashes are stored.

## Traps

**Two-factor needs Data Protection.** Secrets are protected with an
`IDataProtector` under the purpose
`AlmightyShogun.Auth.Credentials.TwoFactor`. Keys that are not persisted across
restarts or shared across instances make every stored secret undecryptable.

**Nothing here enforces two-factor at login.** `LoginAsync` issues tokens on a
correct password; calling `VerifyAsync` and deciding what to do is yours.

**Lockout is off by default** (`AuthCredentials:Lockout:Enabled`). When it is on,
the failed-attempt counter is claimed **before** the password is verified, so a
successful login clears it and a request that throws after the claim still
counted. `AccountLockedException` carries `LockoutEnd`.

**`CreateUserAsync` and `RegisterAsync` check username and email uniqueness with
a read before writing**, which is a race under concurrency; the unique indexes
are the real guard, and a violation surfaces as `DbUpdateException`, not as
`UsernameTakenException`.

**`RegisterAsync` does not check `IsActive` or lockout**, and `LoginAsync`
checks `IsActive` only after the password verifies.

**Password changes revoke every other session.** Pass the caller's current
refresh token as `currentRefreshToken` to keep the session making the request
alive. A reset revokes all of them with no exception.

**`RequestForgotPasswordAsync` returns null for an unknown email** and pads its
own duration to `ForgotPasswordMinimumMilliseconds`, so do not branch on the
result in a way a client can time.

**These services use transactions and `ExecuteUpdateAsync`.** Providers that do
not support them, including the EF Core in-memory provider, will not run this
package.

**Request records carry validation attributes from `AspNet.RequestValidation`.**
`LoginRequest`, `RegisterRequest`, `CreateUserRequest`, `ChangePasswordRequest`
and the two forgot-password requests only validate when that package is
registered.

**`AddAuthCredentials` registers its exception handler by default.** Keep it
before `AddExceptionHandling`, or pass `registerExceptionHandler: false`.

**`EmailVerificationToken` has a table, a context set and indexes, but no service
writes it.** Issuing and checking those tokens is application work.

## Exception to response map

| Exception | Status | Error code | Message key |
| --- | --- | --- | --- |
| `PasswordMismatchException` | 422 | `password_mismatch` | `passwords.mismatch` |
| `PasswordReusedException` | 422 | `password_reused` | `passwords.reused` |
| `UsernameTakenException` | 422 | `username_taken` | `auth.username-taken` |
| `EmailTakenException` | 422 | `email_taken` | `auth.email-taken` |
| `InvalidCredentialsException` | 401 | `invalid_credentials` | `auth.failed` |
| `InvalidSessionException` | 401 | `invalid_session` | `auth.session-invalid` |
| `InvalidTwoFactorCodeException` | 401 | `invalid_two_factor_code` | `auth.two-factor-invalid` |
| `AccountDisabledException` | 403 | `account_disabled` | `auth.disabled` |
| `InvalidPasswordResetTokenException` | 410 | `invalid_password_reset_token` | `passwords.token-invalid` |
| `AccountLockedException` | 423 | `account_locked_out` | `auth.locked-out` with `{0}` = `LockoutEnd` |

## Public surface

Registration: `AddAuthCredentials<TDbContext, TUser>(IConfiguration, bool)`.

Services: `IAuthUserService<TUser>`, `IAuthSessionService<TUser>`,
`IAuthPasswordService`, `IAuthTwoFactorService<TUser>`, all scoped.

Model: `AuthDbContext<TUser>`, `AuthUser`, `UserSession`, `UserLockout`,
`UserTwoFactor`, `TwoFactorRecoveryCode`, `PasswordResetToken`,
`EmailVerificationToken`.

Requests: `LoginRequest`, `RegisterRequest`, `CreateUserRequest`,
`ChangePasswordRequest`, `ForgotPasswordRequest`,
`CompleteForgotPasswordRequest`.

Results: `AuthSessionResult<TUser>` (`AccessToken`, `RefreshToken`, `User`),
`AuthTwoFactorResult` (`Secret`, `Uri`).

Utilities: `TokenHasher.Hash(string)`, SHA-256 as uppercase hex.

Configuration: `AuthCredentialsSettings` with `Lockout`, `TwoFactor`,
`AbsoluteSessionLifetimeDays`, `PasswordResetMinutes`,
`ForgotPasswordMinimumMilliseconds`.
