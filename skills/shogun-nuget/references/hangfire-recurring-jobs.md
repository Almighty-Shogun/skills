# AlmightyShogun.Hangfire.RecurringJobs

Hangfire setup plus recurring schedules declared on the job class instead of in
startup code, with per-job overrides from configuration.

Docs: https://nuget.docs.shogun.ms/hangfire-recurring-jobs/

Depends on `AlmightyShogun.Utils`, Hangfire, `Hangfire.InMemory`, `Cronos` for
validating expressions, and `Newtonsoft.Json`.

## What is where

| Need | Reach for |
| --- | --- |
| Configure Hangfire with sane defaults | `AddCustomHangfire()` |
| Configure Hangfire with your own storage | `AddCustomHangfire(configure)` |
| Discover and schedule the jobs | `RegisterRecurringJobs(configuration?)` |
| Write a job | `IRecurringJob` plus `[RecurringJob("id", cron)]` |
| Common cron strings | `CronSchedules` |
| Read what got scheduled | `IRecurringJobRegistry.Jobs` |

## Writing a job

```csharp
[RecurringJob("nightly-cleanup", CronSchedules.Daily, TimeZone = "Europe/Amsterdam", Queue = "maintenance")]
public sealed class NightlyCleanupJob(IStore store) : IRecurringJob
{
    public Task RunAsync(CancellationToken cancellationToken) => store.CleanupAsync(cancellationToken);
}
```

```csharp
builder.Services
    .AddCustomHangfire()
    .RegisterRecurringJobs(builder.Configuration);
```

`AddCustomHangfire()` with no argument uses in-memory storage at compatibility
level 180, which loses every job on restart; pass a configuration action for real
storage. Both overloads set the simple assembly name type serializer and the
recommended serializer settings, and add a Hangfire server unless
`addServer: false`.

`RegisterRecurringJobs` registers each job type as scoped, binds the
`RecurringJobs` section when a configuration is passed, and adds a hosted service
that calls `AddOrUpdate` for every enabled job at startup.

## Overrides

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

Precedence per field is the configuration override, then the attribute, then the
default. For `Enabled` it is the override, then an explicitly set
`Enabled` on the attribute, then `EnabledByDefault`.

## Traps

**Discovery throws at startup rather than skipping.** An empty job id, an
unparseable cron expression, a time zone this host does not know, two jobs
declaring the same id, or an override naming a job id nothing declares, each
fail the host with an explicit message. The override check is deliberate: a
renamed job leaves a stale override behind and the host tells you.

**A class implementing `IRecurringJob` without the attribute is registered in DI
but never scheduled.** The type registration scans for the interface; scheduling
requires the attribute.

**`RunAsync` must be public**, so implement the interface implicitly. An explicit
implementation is not found and throws when the schedule is built.

**Cron expressions are validated by Cronos**, and the default parser is five
fields (minute precision). `CronSchedules` provides `Minutely`, `Hourly`,
`Daily`, `Weekly` (Monday), `Monthly` and `Yearly`.

**`RegisterRecurringJobs()` scans the calling assembly** unless you pass
assemblies.

**Passing no configuration means no overrides and defaults everywhere**, since
the settings are then bound from nothing.

**The cancellation token the job receives is `CancellationToken.None` at
scheduling time**; Hangfire substitutes its own job token at execution.

**Disabling a job stops it from being registered, it does not remove it.** A job
already stored in Hangfire from a previous run stays there until removed through
Hangfire.

## Public surface

Registration: `AddCustomHangfire(bool addServer = true)`,
`AddCustomHangfire(Action<IGlobalConfiguration>, bool addServer = true)`,
`RegisterRecurringJobs(IConfiguration?)`,
`RegisterRecurringJobs(Assembly[], IConfiguration?)`.

Authoring: `IRecurringJob.RunAsync(CancellationToken)`,
`RecurringJobAttribute(jobId, cronExpression)` with `TimeZone`, `Queue` and
`Enabled`.

Introspection: `IRecurringJobRegistry.Jobs`, `RecurringJobInfo` (`JobId`,
`CronExpression`, `JobType`, `TimeZone`, `Queue`).

Constants: `CronSchedules`.

Configuration: `RecurringJobSettings` (`EnabledByDefault`, `Jobs`),
`RecurringJobOverride` (`Enabled`, `CronExpression`, `TimeZone`, `Queue`).
