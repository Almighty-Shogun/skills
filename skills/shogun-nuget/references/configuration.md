# Configuration sections

Every configurable package binds one `appsettings.json` section through
`AddConfiguration` from `AlmightyShogun.Utils`, which turns on data-annotation
validation and validation on start. A missing required key or an out-of-range
value therefore **fails the host at startup**, not at first use.

Defaults below are the property initializers in the settings records. A key you
do not write takes the default; a key you write as `null` may fail validation.

This file is the source of truth for sections, keys and defaults. Each package
reference ends with a one-line `Configuration:` pointer naming its settings type;
when the two disagree, the settings record in the source decides, and this file
is what gets corrected.

## Localization

Bound by `AddMessageLocalization`.

```json
{
  "Localization": {
    "DefaultLanguage": "en",
    "AutomaticReload": false
  }
}
```

| Key | Default | Notes |
| --- | --- | --- |
| `DefaultLanguage` | `"en"` | required, must match a language tag pattern such as `en` or `nl-BE` |
| `AutomaticReload` | `false` | watch `messages/**/*.json` and drop the cache on change |

## Auth

Bound by `AddAuth`.

```json
{
  "Auth": {
    "Issuer": "https://api.example.com",
    "Secret": "at-least-32-bytes-of-secret-here",
    "AccessTokenMinutes": 60,
    "RefreshTokenDays": 30,
    "ClockSkewSeconds": 30,
    "SameSite": "Lax",
    "DefaultApp": "web",
    "LocalhostApp": "web",
    "Hosts": { "app.example.com": "web", "admin.example.com": "admin" }
  }
}
```

| Key | Default | Notes |
| --- | --- | --- |
| `Issuer` | none | required |
| `Secret` | none | required; the annotation demands 32 characters and the signing key demands 32 UTF-8 bytes |
| `AccessTokenMinutes` | `60` | at least 1 |
| `RefreshTokenDays` | `30` | at least 1 |
| `ClockSkewSeconds` | `30` | at least 1 |
| `DefaultApp` | none | required when `Hosts` is empty |
| `LocalhostApp` | none | audience for `localhost` and loopback |
| `SameSite` | `Lax` | applied to the refresh-token cookie |
| `Hosts` | empty | host to app map; a blank value throws |

Startup fails when no audience resolves at all.

## AuthCredentials

Bound by `AddAuthCredentials`.

```json
{
  "AuthCredentials": {
    "AbsoluteSessionLifetimeDays": 30,
    "PasswordResetMinutes": 60,
    "ForgotPasswordMinimumMilliseconds": 200,
    "Lockout": { "Enabled": false, "MaxFailedAttempts": 5, "DurationMinutes": 15 },
    "TwoFactor": { "Issuer": null, "RecoveryCodeCount": 10, "Digits": 6, "PeriodSeconds": 30, "PendingSecretMinutes": 10 }
  }
}
```

| Key | Default | Notes |
| --- | --- | --- |
| `AbsoluteSessionLifetimeDays` | `30` | caps rotation from the session's creation; null means no cap |
| `PasswordResetMinutes` | `60` | reset-token lifetime |
| `ForgotPasswordMinimumMilliseconds` | `200` | minimum duration of a forgot-password call |
| `Lockout:Enabled` | `false` | lockout is off unless turned on |
| `Lockout:MaxFailedAttempts` | `5` | |
| `Lockout:DurationMinutes` | `15` | |
| `TwoFactor:Issuer` | none | overrides the issuer passed to `BeginEnrolmentAsync` |
| `TwoFactor:RecoveryCodeCount` | `10` | 1 to 50 |
| `TwoFactor:Digits` | `6` | 6 to 8 |
| `TwoFactor:PeriodSeconds` | `30` | 15 to 120 |
| `TwoFactor:PendingSecretMinutes` | `10` | 1 to 60 |

## Maintenance

Bound by `AddMaintenanceMode`.

```json
{
  "Maintenance": {
    "MaintenancePath": "/maintenance",
    "DefaultMessage": null,
    "AutoDisableWhenExpired": false,
    "RedirectBlockedRequests": true,
    "AllowedPaths": [],
    "AllowedPathPrefixes": [],
    "AllowedIpAddresses": []
  }
}
```

| Key | Default | Notes |
| --- | --- | --- |
| `MaintenancePath` | `"/maintenance"` | no whitespace, query string or fragment |
| `DefaultMessage` | none | used when a window supplies none |
| `AutoDisableWhenExpired` | `false` | clear the state file on the first request after the end |
| `RedirectBlockedRequests` | `true` | redirect HTML requests instead of writing the error body |
| `AllowedPaths` | empty | exact paths that pass through |
| `AllowedPathPrefixes` | empty | segment prefixes that pass through |
| `AllowedIpAddresses` | empty | exact addresses, not CIDR |

Each list is the default for a window; a list on the request replaces it.

## RemoteServer

Bound by `AddRemoteCommands`.

```json
{
  "RemoteServer": {
    "Address": "127.0.0.1",
    "Port": 5001,
    "Whitelisted": ["127.0.0.1", "10.0.0.0/8"],
    "Secret": null,
    "EnableReceiveLog": false,
    "MaxPayloadBytes": 1048576,
    "ReadTimeout": 30,
    "IdleTimeout": 120,
    "MaxConcurrentConnections": 100
  }
}
```

| Key | Default | Notes |
| --- | --- | --- |
| `Address` | `"127.0.0.1"` | bind address, must parse as an IP address |
| `Port` | none | required, 1 to 65535 |
| `Whitelisted` | empty | addresses or CIDR ranges; empty rejects every caller |
| `Secret` | none | pre-shared key, compared as a hash in fixed time |
| `EnableReceiveLog` | `false` | log each accepted command |
| `MaxPayloadBytes` | `1048576` | per frame |
| `ReadTimeout` | `30` | seconds to handle one frame |
| `IdleTimeout` | `120` | seconds to wait for the next frame |
| `MaxConcurrentConnections` | `100` | a semaphore, so callers queue past it |

## RecurringJobs

Bound by `RegisterRecurringJobs` when a configuration is passed.

```json
{
  "RecurringJobs": {
    "EnabledByDefault": true,
    "Jobs": {
      "nightly-cleanup": { "Enabled": false, "CronExpression": "0 3 * * *", "TimeZone": "UTC", "Queue": "low" }
    }
  }
}
```

Every override field is optional and falls back to the attribute. An override
naming a job id that no discovered job declares fails the host.

## Email

Bound by `AddResendEmail`.

```json
{
  "Email": {
    "ApiToken": "re_...",
    "FromEmail": "no-reply@example.com",
    "FromName": "Example",
    "BrandName": "Example",
    "LogoUrl": "https://cdn.example.com/logo.png",
    "AppUrl": "https://example.com",
    "Links": {},
    "Template": {
      "CopyrightTextTemplate": "© {app_name}",
      "FooterLinkText": "{app_name}",
      "IgnoreText": ""
    }
  }
}
```

| Key | Default | Notes |
| --- | --- | --- |
| `ApiToken` | none | required, also copied into `ResendClientOptions` |
| `FromEmail` | none | required, must be an email address |
| `FromName` | none | composes `Name <email>` when set |
| `BrandName` | `""` | substituted for `{app_name}` |
| `LogoUrl`, `AppUrl` | none | dropped unless absolute http, https or mailto |
| `Links` | empty | bound but not rendered by the base template |
| `Template:*` | as shown | `{app_name}` and `{app_url}` are substituted |

## CORS

`AddCorsPolicy(name, configuration)` is not bound through `AddConfiguration`. It
reads three keys relative to the `IConfiguration` you pass:

```json
{
  "Cors": {
    "AllowedOrigins": ["https://app.example.com"],
    "AllowedHeaders": [],
    "AllowedMethods": []
  }
}
```

Credentials are always allowed, an empty header or method list means any, and
`"*"` in `AllowedOrigins` throws.

## Serilog

`AddCustomLogging(configuration)` calls `ReadFrom.Configuration`, so the standard
Serilog section applies:

```json
{
  "Serilog": {
    "MinimumLevel": { "Default": "Information", "Override": { "Microsoft": "Warning" } }
  }
}
```

It is applied after the package's console sink, so it adds to that sink rather
than replacing it.
