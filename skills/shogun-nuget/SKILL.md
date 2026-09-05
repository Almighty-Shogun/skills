---
name: shogun-nuget
description: >-
  Use when work materially depends on an AlmightyShogun.* NuGet package: using or
  changing one of its APIs, deciding which package owns a concern, wiring the
  ASP.NET packages together, or resolving the installed contract. Covers Utils,
  AspNet.Core, AspNet.Localization, AspNet.Auth, AspNet.Auth.Credentials,
  AspNet.RequestValidation, AspNet.MaintenanceMode, ConsoleCommands,
  RemoteCommands, Hangfire.RecurringJobs, Hosting.ConsoleLifetime, Mail.Resend,
  Serilog and EntityFrameworkCore.ModelBuilding. Do not trigger merely because an
  unrelated AlmightyShogun using directive appears in a file. NOT for the
  @almighty-shogun npm or Maven packages.
argument-hint: "[package] <what you want>"
license: MIT (skill content); the AlmightyShogun.* NuGet packages are MIT, by Shogun
---

# Shogun NuGet packages

Use this skill when the task materially depends on the `AlmightyShogun.*` NuGet
ecosystem.

Do not load it merely because a file happens to contain an unrelated
`AlmightyShogun.*` using directive.

## Package ownership

Fourteen packages, released together from one monorepo. The `AlmightyShogun.`
prefix is dropped below.

| Package | Owns | Depends on, in scope |
| --- | --- | --- |
| `Utils` | assembly scanning and convention registration, validated options binding, JSON helpers, console helpers | |
| `AspNet.Localization` | JSON message files resolved in the caller's language, `IMessageResolver`, `Content-Language` middleware | `Utils` |
| `AspNet.Core` | the one error body every web package writes, exception mapping vocabulary, request metadata, CORS, Cloudflare forwarded headers | `AspNet.Localization` |
| `AspNet.RequestValidation` | attribute, fluent and custom request rules, reported together as one `422` | `AspNet.Core`, `AspNet.Localization` |
| `AspNet.Auth` | JWT bearer authentication, permission policies, refresh-token cookies, host to audience resolution | `Utils`, `AspNet.Core`, `AspNet.Localization` |
| `AspNet.Auth.Credentials` | password login, refresh sessions with rotation, reset, lockout, TOTP two-factor, EF Core storage | `Utils`, `AspNet.Core`, `AspNet.Auth`, `AspNet.Localization`, `AspNet.RequestValidation` |
| `AspNet.MaintenanceMode` | file-backed maintenance windows and the middleware that enforces them | `Utils`, `AspNet.Core` |
| `ConsoleCommands` | attribute-discovered commands dispatched from a console input loop | `Utils` |
| `RemoteCommands` | length-prefixed JSON over TCP, listener and client | `Utils` |
| `Hangfire.RecurringJobs` | Hangfire setup plus schedules declared on the job class | `Utils` |
| `Hosting.ConsoleLifetime` | a host that ignores an accidental Ctrl+C but stops on SIGTERM | `Utils` |
| `Mail.Resend` | Resend sending with HTML and text templates | `Utils` |
| `Serilog` | Serilog registration plus a coloring console formatter | |
| `EntityFrameworkCore.ModelBuilding` | `ModelBuilder` helpers for relationships, indexes and enums | |

Docs: https://nuget.docs.shogun.ms

Source of truth: `~/Development/nuget-packages`

Do not guess package ownership when the source or an installed assembly can be
checked.

## Reference routing

Read only the package reference needed for the task.

| Area | Primary reference | Additional reference only when needed |
| --- | --- | --- |
| `Utils` | `references/utils.md` | |
| `AspNet.Core` | `references/asp-net-core.md` | `references/asp-net-localization.md` when messages are involved |
| `AspNet.Localization` | `references/asp-net-localization.md` | |
| `AspNet.RequestValidation` | `references/asp-net-request-validation.md` | `references/request-validation-rules.md` for the full rule catalogue |
| `AspNet.Auth` | `references/asp-net-auth.md` | `references/asp-net-core.md` when the task crosses into error responses |
| `AspNet.Auth.Credentials` | `references/asp-net-auth-credentials.md` | `references/asp-net-auth.md` for tokens, cookies and audiences |
| `AspNet.MaintenanceMode` | `references/asp-net-maintenance-mode.md` | |
| `ConsoleCommands` | `references/console-commands.md` | |
| `RemoteCommands` | `references/remote-commands.md` | |
| `Hangfire.RecurringJobs` | `references/hangfire-recurring-jobs.md` | |
| `Hosting.ConsoleLifetime` | `references/hosting-console-lifetime.md` | |
| `Mail.Resend` | `references/mail-resend.md` | |
| `Serilog` | `references/serilog.md` | |
| `EntityFrameworkCore.ModelBuilding` | `references/ef-core-model-building.md` | |
| Any `appsettings.json` section | `references/configuration.md` | the owning package reference |

Start with the directly relevant package. Inspect another guide only when the
actual implementation crosses that boundary.

## Wiring order is load bearing

The web packages share one error body and one exception pipeline, so
registration order changes behavior:

- `AddMessageLocalization` must be registered whenever `AspNet.Core` writes an
  error, because the handlers and `UseHttpErrorResponses` resolve
  `IMessageResolver`.
- `AddHttpErrorResponseWriter` must be registered separately.
  `AddExceptionHandling` does not register it.
- Exception handlers run in registration order and the fallback registered by
  `AddExceptionHandling` answers every exception, so every other handler,
  including the ones `AddAuth` and `AddAuthCredentials` register, must be
  registered **before** `AddExceptionHandling`.

See `references/asp-net-core.md` for the full pipeline and middleware order.

## Installed contract wins

The restored package version is the contract that consuming code must compile
against, not the current monorepo source.

Resolve it from repository evidence: `PackageReference` entries in the project
file, `Directory.Packages.props` when `ManagePackageVersionsCentrally` is true,
`obj/project.assets.json` for what actually restored, and the extracted package
under `~/.nuget/packages/<id>/<version>/`.

That package holds `lib/<tfm>/<Id>.dll` and the `.nuspec`, which names the
version and the dependencies. Releases have shipped without an XML documentation
file beside the assembly, so check for `<Id>.xml` before relying on it, and
otherwise confirm a signature from the assembly through the IDE, or from the
monorepo source at the tag matching the restored version.

Priority: the installed package and restored assets, then the monorepo source at
the matching tag, then the matching bundled reference guide, then the
documentation site.

If a guide and the installed contract disagree, follow the installed contract and
mention the mismatch when it materially affects the answer or the change.

## Legacy package ids

These packages were renamed once. A project referencing an id in the left column
predates the rename:

| Legacy id | Current id |
| --- | --- |
| `AlmightyShogun.AspNet.JwtAuth` | `AlmightyShogun.AspNet.Auth` |
| `AlmightyShogun.AspNet.Utils` | `AlmightyShogun.AspNet.Core` and `AlmightyShogun.AspNet.Localization` |
| `AlmightyShogun.EntityFrameworkCore.Utils` | `AlmightyShogun.EntityFrameworkCore.ModelBuilding` |
| `AlmightyShogun.Hangfire.Utils` | `AlmightyShogun.Hangfire.RecurringJobs` |
| `AlmightyShogun.Hosting.Utils` | `AlmightyShogun.Hosting.ConsoleLifetime` |
| `AlmightyShogun.Logging` | `AlmightyShogun.Serilog` |
| `AlmightyShogun.Resend.Utils` | `AlmightyShogun.Mail.Resend` |

`Utils`, `ConsoleCommands` and `RemoteCommands` kept their ids.
`AspNet.Auth.Credentials`, `AspNet.RequestValidation` and
`AspNet.MaintenanceMode` came later and have no legacy id.

A rename is not an API-compatibility claim, and the split of `AspNet.Utils` into
`AspNet.Core` and `AspNet.Localization` moved and reshaped types rather than
copying them. When a project references a legacy id, verify against that package,
not against the reference guides here.

Delete this section once no project you work on references a legacy id.

## Host repository conventions win

Read the applicable host-project `AGENTS.md`, `CLAUDE.md`, and any project or
directory-local instructions. The consuming project's conventions take priority
over development conventions from the `AlmightyShogun.*` package repository.

## Working inside the package monorepo

When the task edits `~/Development/nuget-packages` itself rather than a consumer,
that repository's own `CLAUDE.md` and `docs/CLAUDE.md` govern, and they are
stricter than this skill:

- Every member carries a full XML documentation block, checked by the build with
  `GenerateDocumentationFile` on and no `NoWarn`.
- `max_line_length` is 140 columns, counting the `///` prefix.
- `<since>` records when a member was added, never when it changed.
- Documentation pages under `docs/` are written by hand after reading the source.
- Packages are versioned together, and CI publishes on a GitHub release.

Use the `csharp-docs` skill for documentation work there. Do not change code to
make an existing doc block true.

## Implementation permission

An explicit request for a scoped implementation already authorizes the ordinary
code edits it requires. Do not ask to proceed before normal source changes.

Ask first only when a material requirement is unresolved, several materially
different behaviors are possible, a dependency must be added or changed,
consequential configuration must be changed, or a database schema change would
follow, such as anything touching `AuthDbContext<TUser>` or its entities.

## Missing dependencies

Inspect the project's `.csproj`, `Directory.Packages.props` and
`Directory.Build.props` where they exist, `obj/project.assets.json` for what
actually restored, and the solution file when several projects are involved.

Presence in the restore graph does **not** authorize direct use. These packages
reference each other, so `Utils` is restored by almost every other one and
`AspNet.Localization` comes in with `AspNet.Core`. When project source uses a
package's API directly, it belongs on that project as a direct
`PackageReference`.

When one is missing, verify it is actually necessary, identify the correct
project, check whether central package management is in use, because the version
then belongs in `Directory.Packages.props` and the reference carries no
`Version`, show the exact change or `dotnet add package` command, and get
explicit approval before changing the dependency.

An ASP.NET package also needs the web SDK or
`<FrameworkReference Include="Microsoft.AspNetCore.App"/>` in the consuming
project. `AspNet.Auth.Credentials` additionally needs an EF Core provider, which
it does not supply.

Do not install dependencies unasked.

## Configuration changes

Every configurable package binds one `appsettings.json` section through
`AddConfiguration`, which turns on data-annotation validation and validation on
start, so a missing or invalid section fails at startup rather than at first use.
`references/configuration.md` lists every section, key and default.

Do not change runtime, build or hosting configuration merely to make a package
convenient. When a consequential change is genuinely required, explain why, show
the exact intended change, and get approval before applying it.

## Runtime and platform uncertainty

Do not invent answers for platform semantics that materially affect behavior:
ASP.NET Core middleware and exception-handler ordering, EF Core provider
differences including which providers support the `ExecuteUpdateAsync` and
transaction patterns these packages use, Hangfire storage and queue behavior,
Data Protection key persistence that two-factor secrets depend on, or the time
zone ids available on the host.

Use repository-local evidence and the installed assemblies first. When
substantial external research is required, use `source-research` rather than
bloating this skill.

## Verification

Use the host project's real validation commands, discovered from the solution and
project files, CI configuration or repository instructions. Inside the package
monorepo the checks are:

```sh
dotnet build packages/<Package>/<Package>.csproj --no-incremental
dotnet build packages.sln
bun run docs:build
```

Run the narrowest check that covers the change, and do not run publish commands.
Passing checks do not justify ignoring a clear package-contract mismatch.

## Scope

This is an implementation and reference skill, not a Git workflow: the global
instructions on Git and outward-facing actions apply unchanged, and completed Git
work goes to the `commit` skill. The worktree may be dirty, so protect existing
user changes.

Keep context small. Load one package guide, use `references/request-validation-rules.md`
only for an exact rule lookup, inspect installed assemblies or monorepo source
directly rather than pasting them into parent context, and do not enumerate a
package's whole API to answer a one-symbol question.
