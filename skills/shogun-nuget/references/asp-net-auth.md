# AlmightyShogun.AspNet.Auth

JWT bearer authentication, permission-based authorization, refresh-token cookie
helpers, and host-to-audience resolution for an API that serves several apps
from one deployment.

Docs: https://nuget.docs.shogun.ms/asp-net-auth/

Depends on `AlmightyShogun.Utils`, `AlmightyShogun.AspNet.Core`,
`AlmightyShogun.AspNet.Localization`, and the Microsoft JWT bearer stack.

This package issues and validates tokens. It does not store users or verify
passwords; that is `AlmightyShogun.AspNet.Auth.Credentials`.

## What is where

| Need | Reach for |
| --- | --- |
| Register everything | `AddAuth(configuration)` |
| Issue an access token | `IAuthTokenGenerator.Generate(claims, audience?)` |
| Require a permission on an endpoint | `[AuthPermission("users.read")]` |
| Read the current user id | `User.GetCurrentUserId()` or `TryGetCurrentUserId()` |
| Read the refresh-token cookie | `request.GetRefreshTokenCookie()` or `TryGetRefreshTokenCookie()` |
| Set or clear the refresh-token cookie | `response.SetRefreshTokenCookie(token, days)`, `response.DeleteAuthCookies()` |
| Know which app the current host is | `IAppHostResolver.Resolve()` or `TryResolve(out app)` |
| Claim and policy names | `AuthClaimTypes`, `AuthPolicies`, `CookieNames` |

## Apps and audiences

Every token carries an audience and the audience is always validated. Two shapes:

- **single app**: set `Auth:DefaultApp`, leave `Auth:Hosts` empty. Every token
  gets that audience and no host lookup happens;
- **scoped**: fill `Auth:Hosts` with `host` to `app` pairs. The audience comes
  from the request host, and `Auth:LocalhostApp` covers `localhost` and loopback
  addresses during development.

`AddAuth` validates at startup that at least one audience resolved. Building the
audience list throws when a `Hosts` entry has a blank value, or when `Hosts` is
empty and `DefaultApp` is unset.

Every policy the app resolves gains an audience requirement, so an otherwise
valid token minted for another app is rejected on a host that maps elsewhere.

## Permissions

`[AuthPermission("users.read")]` is an `AuthorizeAttribute` whose policy name is
`permission:users.read`. A custom policy provider builds that policy on demand,
requiring an authenticated user plus the permission.

A granted permission satisfies a required one when it matches case-insensitively,
or when it ends in `.*` and the required permission starts with the same prefix,
so `users.*` grants `users.read`.

Permission claims are read from the `permission` claim type; the user id from
`userId`, falling back to `ClaimTypes.NameIdentifier`.

## Task recipes

### Register it

```csharp
builder.Services
    .AddMessageLocalization(builder.Configuration)
    .AddHttpErrorResponseWriter()
    .AddAuth(builder.Configuration)
    .AddExceptionHandling();
```

`AddAuth` binds `Auth`, adds the JWT bearer scheme and authorization, replaces
`IAuthorizationPolicyProvider`, registers the permission and audience handlers,
and by default registers its exception handler. Keep it before
`AddExceptionHandling`, or pass `registerExceptionHandler: false`.

### Issue a token pair

```csharp
AuthToken access = tokenGenerator.Generate(claims);
response.SetRefreshTokenCookie(refreshToken, authSettings.RefreshTokenDays);
```

`Generate` resolves the audience from the current host when scoped, otherwise
from `DefaultApp`, unless you pass one explicitly.

### Cookie behavior

`SetRefreshTokenCookie` writes `refreshToken` at path `/`, HTTP only, `Secure`
only when the request is HTTPS, and `SameSite` from `Auth:SameSite` (default
`Lax`). `DeleteAuthCookies` deletes it with the same path, SameSite and Secure.

## Traps

**`Auth:Secret` must encode to at least 32 bytes.** Both the options data
annotation and the signing-key factory enforce it, so a short secret fails the
host at startup with an explicit message.

**A scoped setup rejects an unmapped host.** `IAppHostResolver.Resolve()` throws
`UnknownAppException` when `Hosts` is non-empty and the request host matches
nothing, including `localhost` when `LocalhostApp` is unset.

**`TryResolve` returns true with a null app in the unscoped case.** True means
"resolution succeeded", not "an app was found", so check the out parameter.

**The resolved app is cached in `HttpContext.Items`** for the request, so
changing the host mid-request changes nothing.

**`GetCurrentUserId` throws `MissingUserIdClaimException`** when the claim is
absent or is not a `Guid`. `TryGetCurrentUserId` returns null instead. Both are
extensions on `ClaimsPrincipal`, so they work on `User` in a controller.

**`GetRefreshTokenCookie` throws `MissingRefreshTokenException`** on a blank or
absent cookie; use the `Try` variant on paths where absence is normal.

**The three exceptions map to fixed responses.** `MissingUserIdClaimException`
and `MissingRefreshTokenException` become 401, `UnknownAppException` becomes 403,
each with a message key under `auth.*`. The mapper is internal, so a different
status means writing your own handler ahead of this one.

**A permission wildcard only matches a `.` boundary as written.** The check
strips the final `*` and compares the prefix, so `users.*` also matches
`users.readonly`, and `users*` is not a wildcard at all.

**`ClockSkewSeconds` defaults to 30**, not the Microsoft default of five
minutes, so tokens expire closer to their stated lifetime.

## Public surface

Registration: `AddAuth(IConfiguration, bool registerExceptionHandler = true)`.

Tokens: `IAuthTokenGenerator.Generate(IEnumerable<Claim>, string? audience)`,
`AuthToken` (`Token`, `ExpiresAt`).

Apps: `IAppHostResolver` (`TryResolve(out string?)`, `Resolve()`,
`TryResolveAppFromHost(string?, out string)`, `ResolveAppFromHost(string?)`).

Authorization: `AuthPermissionAttribute`, `AuthPolicies.PermissionPrefix`,
`AuthClaimTypes.UserId`, `AuthClaimTypes.Permission`.

Extensions: `ClaimsPrincipal.GetCurrentUserId()`, `TryGetCurrentUserId()`,
`HttpRequest.GetRefreshTokenCookie()`, `TryGetRefreshTokenCookie()`,
`HttpResponse.SetRefreshTokenCookie(string, int)`, `DeleteAuthCookies()`,
`CookieNames.RefreshToken`.

Exceptions: `MissingUserIdClaimException`, `MissingRefreshTokenException`,
`UnknownAppException` (carries `Host`).

Configuration: `AuthSettings` (`Issuer`, `Secret`, `AccessTokenMinutes`,
`RefreshTokenDays`, `ClockSkewSeconds`, `DefaultApp`, `LocalhostApp`, `Hosts`,
`SameSite`, computed `ValidAudiences`, `IsScoped()`).
