# AlmightyShogun.Hosting.ConsoleLifetime

A host lifetime for a long-running console process that should not die from a
stray Ctrl+C, while still stopping cleanly for an orchestrator.

Docs: https://nuget.docs.shogun.ms/hosting-console-lifetime/

Depends on `AlmightyShogun.Utils` and `Microsoft.Extensions.Hosting`.

## What is where

| Need | Reach for |
| --- | --- |
| Replace the default lifetime | `UseCustomConsoleLifetime()` |
| Set shutdown timeout and background-service behavior | `ConfigureHostOptions(timeout, behavior)` |

Each is available on `IServiceCollection`, `IHostApplicationBuilder` and
`IHostBuilder`, so it fits both the builder styles.

## Behavior

```csharp
builder.UseCustomConsoleLifetime();
builder.ConfigureHostOptions(TimeSpan.FromSeconds(30), BackgroundServiceExceptionBehavior.Ignore);
```

The lifetime cancels Ctrl+C so the host keeps running, and on non-Windows
registers a SIGTERM handler that cancels the default behavior and calls
`StopApplication`, giving an orderly shutdown for a container or a service
manager.

The exception is a debugger session: when `DOTNET_RUNNING_IN_IDE` is set, Ctrl+C
is left alone, so stopping from the IDE still works.

## Traps

**Ctrl+C does nothing in production by design.** Stop the process with SIGTERM,
or from inside the application through `IHostApplicationLifetime.StopApplication`
or a command. Nothing else stops it.

**Windows gets no SIGTERM registration**, so the host relies on the normal
Windows shutdown path there.

**The lifetime is registered by replacing `IHostLifetime`**, through
`ReplaceService` from `AlmightyShogun.Utils`, so anything registering its own
lifetime after this call wins.

**`ConfigureHostOptions` here sets exactly two options**, `ShutdownTimeout` and
`BackgroundServiceExceptionBehavior`, and both parameters are required. It is a
convenience over `Configure<HostOptions>`, not a full options surface.

**`CustomConsoleLifetime` is internal.** Use the extensions; there is no type to
subclass or resolve.

## Public surface

`UseCustomConsoleLifetime()` and
`ConfigureHostOptions(TimeSpan, BackgroundServiceExceptionBehavior)`, each on
`IServiceCollection`, `IHostApplicationBuilder` and `IHostBuilder`.
