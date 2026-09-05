# AlmightyShogun.AspNet.MaintenanceMode

File-backed maintenance windows that survive a restart. Enabled state lives in
`maintenance.json` at the content root, so a process restart, or a second
instance sharing that path, sees the same window.

Docs: https://nuget.docs.shogun.ms/asp-net-maintenance-mode/

Depends on `AlmightyShogun.Utils` and `AlmightyShogun.AspNet.Core`, whose error
writer it uses for the blocked response.

## What is where

| Need | Reach for |
| --- | --- |
| Register it | `AddMaintenanceMode(configuration)` |
| Block requests while a window is active | `UseMaintenanceMode()` |
| Turn it on | `IMaintenanceService.EnableAsync(request)` |
| Turn it off | `IMaintenanceService.DisableAsync()` |
| Read the current window | `IMaintenanceService.GetAsync()` |
| Cheap on/off check | `IMaintenanceService.IsEnabledAsync()` |

## Behavior

Place `UseMaintenanceMode()` early, before the middleware you want blocked.

While a window is active, a request that is not allowed through gets a `503`
with the standard error body, error code `service_unavailable`, and the window's
message as the description. A `Retry-After` header in seconds is added whenever
the window has an end.

A request that accepts `text/html` is redirected to the maintenance path instead,
when `RedirectBlockedRequests` is on.

The maintenance path itself answers `503` with the window details
(`MaintenanceResponse`: message, `StartsAt`, `EndsAt`, `EnabledAt`) while the
window is active, and `404` while it is not.

A window is active when it is enabled and `StartsAt` is null or already passed,
so scheduling ahead is enabling with a future `StartsAt`.

Requests pass through when the path matches `AllowedPaths` exactly, starts with
one of `AllowedPathPrefixes` as a segment prefix, or the remote address matches
`AllowedIpAddresses` exactly.

## Task recipes

### Schedule a window

```csharp
await maintenanceService.EnableAsync(new MaintenanceRequest
{
    Message = "Upgrading the database.",
    StartsAt = DateTimeOffset.UtcNow.AddMinutes(10),
    EndsAt = DateTimeOffset.UtcNow.AddMinutes(40),
    AutoDisableWhenExpired = true,
    AllowedPathPrefixes = ["/admin"]
});
```

Every nullable field on the request falls back to the matching key in the
`Maintenance` configuration section when omitted, so configuration holds the
defaults and the request holds the exceptions.

## Traps

**A list on the request replaces the configured list, it does not add to it.**
Passing `AllowedPaths = []` clears the configured allow list for that window;
omitting it keeps the configured one.

**`EnableAsync` throws `ArgumentException`** when `EndsAt` is at or before
`StartsAt`.

**`AutoDisableWhenExpired` is evaluated on read, not on a timer.** The first
request after the end time clears the state file, using the stored revision so
two instances cannot both clear it. With the flag off, an expired window keeps
blocking until something disables it.

**A corrupt state file fails closed.** If `maintenance.json` cannot be parsed the
package treats the site as enabled with the message "Maintenance file is corrupt,
please resolve this.", so a bad file takes the site down rather than letting
traffic through.

**A file that cannot be read falls back to the last known state**, after three
attempts with a short delay, and logs a warning.

**IP allow entries are exact addresses, not CIDR ranges.** They are compared
after unwrapping IPv4-mapped IPv6, and anything unparseable never matches.

**Path allow entries are normalized**: trimmed, leading slash added, trailing
slash removed. `AllowedPathPrefixes` matches on segment boundaries, so `/admin`
matches `/admin/users` but not `/administration`.

**`MaintenancePath` is validated** against a pattern that rejects whitespace, a
query string, and a fragment, so a bad value fails the host at startup.

**The state file is watched only when the directory exists** at first read, and
the watcher is best effort: a platform that cannot watch logs a warning and an
out-of-band edit then goes unnoticed until the process writes again.

**Writes are atomic through a temp file plus move**, and serialized by a
semaphore inside the process. Two processes writing at once is not coordinated.

## Public surface

Registration: `AddMaintenanceMode(IConfiguration)` on `IServiceCollection`,
`UseMaintenanceMode()` on `IApplicationBuilder`.

Service: `IMaintenanceService` (`GetAsync()`, `IsEnabledAsync()`,
`EnableAsync(MaintenanceRequest)`, `DisableAsync()`), registered as a singleton.

Models: `MaintenanceRequest`, `MaintenanceState` (`IsEnabled`, `Message`,
`StartsAt`, `EndsAt`, `EnabledAt`), `MaintenanceResponse`.

Configuration: `MaintenanceSettings` with `MaintenancePath`, `DefaultMessage`,
`AutoDisableWhenExpired`, `RedirectBlockedRequests`, `AllowedPaths`,
`AllowedPathPrefixes`, `AllowedIpAddresses`.
