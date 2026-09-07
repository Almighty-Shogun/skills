# AlmightyShogun.AspNet.Auth.Credentials

Username and password accounts on top of `AspNet.Auth`: login with a two-step
two-factor handoff, registration, refresh sessions with rotation and reuse
detection, password change and reset, email verification and address change,
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
| Finish a two-factor login | `IAuthUserService<TUser>.CompleteTwoFactorLoginAsync(request, httpContext)` |
| Register a user and log them in | `IAuthUserService<TUser>.RegisterAsync(user, password, httpContext)` |
| Create a user without a session | `IAuthUserService<TUser>.CreateUserAsync(user, password)` |
| Rotate a refresh token | `IAuthSessionService<TUser>.RefreshSessionAsync(token, httpContext)` |
| Log out one session | `IAuthSessionService<TUser>.RevokeSessionAsync(token)` |
| Create a session yourself | `IAuthSessionService<TUser>.CreateSessionAsync(user, app, clientContext)` |
| Change a password | `IAuthPasswordService.ChangePasswordAsync(identifier, request, currentRefreshToken?)` |
| Start a reset | `IAuthPasswordService.RequestForgotPasswordAsync(request, ip?)` |
| Finish a reset | `IAuthPasswordService.CompleteForgotPasswordAsync(request)` |
| Verify the address on file | `IAuthEmailService.RequestVerificationAsync` then `CompleteVerificationAsync` |
| Move a user to a new address | `IAuthEmailService.RequestEmailChangeAsync` then `CompleteEmailChangeAsync` |
| Enrol in two-factor | `IAuthTwoFactorService<TUser>.BeginEnrolmentAsync` then `CompleteEnrolmentAsync` |
| Check a code or recovery code | `IAuthTwoFactorService<TUser>.VerifyAsync(identifier, code)` |
| Turn two-factor off | `IAuthTwoFactorService<TUser>.DisableAsync(identifier)` |
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
indexes on `Identifier`, `Username`, `Email` and on the refresh-token,
reset-token, email-verification and two-factor-challenge hashes, lookup indexes
on `UserSession(UserId, App, ExpiresAt)`, `EmailVerificationToken(UserId,
ExpiresAt)` and `TwoFactorChallenge(UserId, ExpiresAt)`, an int conversion for
`EmailVerificationToken.Purpose`, cascade deletes from the user, and a
concurrency token on `UserSession`. Call `base.OnModelCreating(modelBuilder)`
from any override.

Tables are snake case: `users`, `user_sessions`, `password_reset_tokens`,
`email_verification_tokens`, `two_factor_challenges`, `user_two_factors`,
`user_lockouts`, `two_factor_recovery_codes`.

`AuthUser` carries `Id` (int key), `Identifier` (Guid v7, the value that goes
into the token), `Username`, `Email`, `EmailVerifiedAt`, `Password` (hash),
`Role`, `Permissions`, `IsActive`, and the `Sessions`, `Lockout` and `TwoFactor`
navigations.

One reset token exists per user: `PasswordResetToken` is mapped one to one with
the user, so requesting again rewrites the same row.

## Login is two steps when two-factor is enrolled

`LoginAsync` returns `AuthLoginResult<TUser>`, not a session. When the user has
an enabled two-factor enrolment it comes back carrying a `Challenge` and no
`Session`; otherwise it carries the `Session`.

```csharp
AuthLoginResult<AppUser> result = await userService.LoginAsync(request, HttpContext, cancellationToken);

if (result.RequiresTwoFactor)
    return Ok(new { challenge = result.Challenge });

Response.SetRefreshTokenCookie(result.Session.RefreshToken, authSettings.RefreshTokenDays);

return Ok(new { result.Session.AccessToken });
```

`RequiresTwoFactor` carries `MemberNotNullWhen` on both properties, so the
compiler narrows `Session` on the false branch and `Challenge` on the true one.

The challenge is a raw base64url token seen once; only its SHA-256 hash is
stored. It expires after `AuthCredentials:TwoFactor:ChallengeMinutes` (default
5), is single use, and records the app the login resolved to. Issuing one
retires the user's other unused challenges and deletes their expired ones.

```csharp
AuthSessionResult<AppUser> session =
    await userService.CompleteTwoFactorLoginAsync(request, HttpContext, cancellationToken);
```

`CompleteTwoFactorLoginAsync` refuses with `InvalidTwoFactorChallengeException`
when the challenge is unknown, used, expired, or was issued for a different app
than the current host resolves to. It then checks `IsActive`, verifies the code
through `VerifyAsync`, spends the challenge, clears the lockout and opens the
session.

`LoginRequest.Identifier` matches either the username or the email.

## Sessions

`CreateSessionAsync` returns a raw refresh token, stores only its SHA-256 hex
hash, records IP, user agent, browser, OS and device truncated to their column
widths, and deletes that user's already expired sessions first. The new
session's expiry is `Auth:RefreshTokenDays` out, capped at
`AuthCredentials:AbsoluteSessionLifetimeDays` from `CreatedAt`, so the absolute
lifetime holds from sign-in onward rather than only from the first rotation.

`RefreshSessionAsync` rotates: it looks the session up by hash, filtered to the
resolved app when there is one, refuses with `InvalidSessionException` when the
session is already past its absolute lifetime, then moves the old hash to
`PreviousRefreshTokenHash`, writes a new one, refreshes the metadata, extends
the expiry under the same cap, and bumps the concurrency token. Two concurrent
refreshes mean one loses on `DbUpdateConcurrencyException` and gets
`InvalidSessionException`.

Presenting a token that was already rotated away is treated as reuse: every
non-revoked session for that user is revoked and a warning is logged, unless the
rotation happened within the 30-second grace window, which absorbs a retry. The
rotated row's `PreviousRefreshTokenHash` is cleared at the same time, so one
retired token fires the revocation once. That save is retried up to five times
against a concurrent rotation; if it still loses, an error is logged and the
sessions stand.

Permissions are scoped to the app when one resolves: only claims prefixed
`app:` survive, with the prefix stripped.

## Task recipes

### Password reset

```csharp
string? token = await passwordService.RequestForgotPasswordAsync(request, HttpContext.GetIpAddress(), cancellationToken);

if (token is not null)
    await mailService.SendAsync(request.Email, new ResetMail(token));
```

The returned token is raw and is the only time you see it; only its hash is
stored. Requesting again overwrites the user's single reset row. Completing a
reset revokes every session for that user and retires their unused two-factor
challenges.

### Email verification and address change

```csharp
string token = await emailService.RequestVerificationAsync(identifier, cancellationToken);
await mailService.SendAsync(user.Email, new VerifyMail(token));

await emailService.CompleteVerificationAsync(new CompleteEmailVerificationRequest { Token = token }, cancellationToken);
```

`RequestEmailChangeAsync(identifier, newEmail)` issues the same shape of token
against the proposed address, throwing `EmailTakenException` when another user
already holds it. `CompleteEmailChangeAsync` re-checks availability, writes the
new address, stamps `EmailVerifiedAt`, retires any outstanding registration
token, and revokes the user's sessions except the one whose refresh token you
pass as `currentRefreshToken`.

Both requests retire the user's other unused tokens of the same purpose first.
Tokens live for `AuthCredentials:EmailVerificationMinutes` (default 1440) and
are single use.

### Two-factor enrolment

```csharp
AuthTwoFactorResult begin = await twoFactorService.BeginEnrolmentAsync(identifier, "My App", cancellationToken);
// show begin.Uri as a QR code

IReadOnlyList<string> recoveryCodes = await twoFactorService.CompleteEnrolmentAsync(identifier, code, cancellationToken);
// show recoveryCodes once
```

`CompleteEnrolmentAsync` replaces any existing recovery codes and returns the new
ones in the clear; only their hashes are stored. The pending secret expires after
`AuthCredentials:TwoFactor:PendingSecretMinutes`.

## Traps

**Two-factor needs Data Protection.** Secrets are protected with an
`IDataProtector` under the purpose
`AlmightyShogun.Auth.Credentials.TwoFactor`. Keys that are not persisted across
restarts or shared across instances make every stored secret undecryptable.

**`LoginAsync` no longer returns a session directly.** Check
`RequiresTwoFactor` before touching `Session`, and give the client a route that
calls `CompleteTwoFactorLoginAsync` with the challenge and the code.

**Lockout is off by default** (`AuthCredentials:Lockout:Enabled`). When it is on,
the failed-attempt counter is claimed **before** the password is verified, and
`VerifyAsync` claims against the same budget, so a two-factor attempt counts.
Login releases its own claim when it hands out a challenge, so the two steps of
one login spend one attempt rather than two. A successful login or verification
clears the counter; a request that throws after the claim and before the release
still counted. `AccountLockedException` carries `LockoutEnd`.

**`CreateUserAsync` and `RegisterAsync` check username and email uniqueness with
a read before writing**, which is a race under concurrency; the unique indexes
are the real guard, and a violation surfaces as `DbUpdateException`, not as
`UsernameTakenException`.

**`RegisterAsync` does not check `IsActive` or lockout**, and `LoginAsync`
checks `IsActive` only after the password verifies.

**Password changes revoke every other session.** Pass the caller's current
refresh token as `currentRefreshToken` to keep the session making the request
alive. A reset revokes all of them with no exception. Both also retire the
user's unused two-factor challenges.

**`RequestForgotPasswordAsync` returns null for an unknown email** and pads its
own duration to `ForgotPasswordMinimumMilliseconds`, so do not branch on the
result in a way a client can time.

**`CompleteVerificationAsync` refuses a token whose address is no longer the
user's**, with `InvalidEmailVerificationTokenException`, so a verification
issued before an address change cannot land afterwards.

**`VerifyAsync` returns false rather than throwing** on a wrong code, a spent
recovery code, a replayed TOTP window, an undecryptable secret, or an enrolment
that is not enabled. It still throws: `InvalidCredentialsException` for an
unknown identifier, `InvalidTwoFactorCodeException` when the user has no
enrolment row at all, and `AccountLockedException` when the lockout budget is
already spent.

**These services use transactions and `ExecuteUpdateAsync`.** Providers that do
not support them, including the EF Core in-memory provider, will not run this
package. The reset-token write additionally opens a serializable transaction.

**Request records carry validation attributes from `AspNet.RequestValidation`.**
`LoginRequest`, `RegisterRequest`, `CreateUserRequest`, `ChangePasswordRequest`,
`CompleteTwoFactorLoginRequest`, `CompleteEmailVerificationRequest` and the two
forgot-password requests only validate when that package is registered.

**`AddAuthCredentials` registers its exception handler by default.** Keep it
before `AddExceptionHandling`, or pass `registerExceptionHandler: false`.

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
| `InvalidTwoFactorChallengeException` | 401 | `invalid_two_factor_challenge` | `auth.two-factor-challenge-invalid` |
| `AccountDisabledException` | 403 | `account_disabled` | `auth.disabled` |
| `InvalidPasswordResetTokenException` | 410 | `invalid_password_reset_token` | `passwords.token-invalid` |
| `InvalidEmailVerificationTokenException` | 410 | `invalid_email_verification_token` | `auth.email-verification-token-invalid` |
| `AccountLockedException` | 423 | `account_locked_out` | `auth.locked-out` with `{0}` = `LockoutEnd` |

## Public surface

Registration: `AddAuthCredentials<TDbContext, TUser>(IConfiguration, bool)`.

Services: `IAuthUserService<TUser>`, `IAuthSessionService<TUser>`,
`IAuthPasswordService`, `IAuthTwoFactorService<TUser>`, `IAuthEmailService`, all
scoped.

Model: `AuthDbContext<TUser>`, `AuthUser`, `UserSession`, `UserLockout`,
`UserTwoFactor`, `TwoFactorRecoveryCode`, `TwoFactorChallenge`,
`PasswordResetToken`, `EmailVerificationToken`, `EmailVerificationPurpose`
(`Registration`, `EmailChange`).

Requests: `LoginRequest`, `CompleteTwoFactorLoginRequest` (`Challenge`, `Code`),
`RegisterRequest`, `CreateUserRequest`, `ChangePasswordRequest`,
`ForgotPasswordRequest`, `CompleteForgotPasswordRequest`,
`CompleteEmailVerificationRequest` (`Token`).

Results: `AuthLoginResult<TUser>` (`User`, `Session?`, `Challenge?`,
`RequiresTwoFactor`), `AuthSessionResult<TUser>` (`AccessToken`, `RefreshToken`,
`User`), `AuthTwoFactorResult` (`Secret`, `Uri`).

Utilities: `TokenHasher.Hash(string)`, SHA-256 as uppercase hex.

Configuration: `AuthCredentialsSettings` with `Lockout`, `TwoFactor`,
`AbsoluteSessionLifetimeDays`, `PasswordResetMinutes`,
`EmailVerificationMinutes`, `ForgotPasswordMinimumMilliseconds`.
