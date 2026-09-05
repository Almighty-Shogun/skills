# AlmightyShogun.Serilog

Serilog registration plus a console formatter that colors the level prefix and
each rendered property, with an inline color spec in the message template.

Docs: https://nuget.docs.shogun.ms/serilog/

Depends on Serilog with the console and async sinks and the configuration
settings package. It references no other package in this scope.

## What is where

| Need | Reach for |
| --- | --- |
| Add logging to a service collection | `AddCustomLogging(configuration?, includeConsoleSink, enableColors?)` |
| Add logging on a host builder | the same extension on `IHostBuilder` |
| Color one property | a color code after `|` in the property's format |

## Registration

```csharp
builder.Services.AddCustomLogging(builder.Configuration);
```

The logger is built with `Enrich.FromLogContext()`, an async-wrapped console sink
using the package formatter, and then, when a configuration is passed,
`ReadFrom.Configuration`, so a `Serilog` section can add sinks and set levels.
The `IServiceCollection` overload registers it through `AddSerilog`; the
`IHostBuilder` overload through `UseSerilog`. Both hand ownership of the logger
to the host, so it is disposed with it.

## Output

Each line is `[HH:mm:ss LVL] message`, where the prefix is colored by level:
information green, warning yellow, error red, fatal bright red, verbose and debug
white. An exception is appended on its own line in dark gray.

Rendered properties are colored by type: strings white, integers and reals cyan,
booleans magenta, null dark gray, anything else white.

Override a property's color with a short code after a pipe in its format:

```csharp
logger.LogInformation("Started listening on {Address:c}:{Port:c}", address, port);
logger.LogWarning("{Name:y} is already registered", name);
logger.LogInformation("Elapsed {Duration:F2|bg} seconds", duration);
```

Codes: `r`, `g`, `b`, `c`, `y`, `m`, and the bright variants `br`, `bg`, `bb`,
`bc`, `by`, `bm`. Anything else falls back to white. Text before the pipe is
still used as the numeric or date format.

## Traps

**Colors are auto-detected once per process.** Output goes uncolored when it is
redirected or when `NO_COLOR` is set. Pass `enableColors: true` or `false` to
decide explicitly.

**A property in the template that is not supplied is written back literally**,
braces, format and all, rather than throwing.

**A format string that the value rejects falls back to `ToString()`**, so a
mistyped numeric format degrades quietly.

**Configuration is applied after the console sink**, so a `Serilog` section adds
to the console sink rather than replacing it. Pass `includeConsoleSink: false`
when configuration should own the console.

**The color codes are an extension of Serilog's format syntax**, understood only
by this formatter. The same template sent to another sink shows `F2|bg` as the
format.

**The formatter and the color table are internal.** There is no public type to
reuse in a custom sink configuration.

## Public surface

`AddCustomLogging(IConfiguration? configuration = null, bool includeConsoleSink =
true, bool? enableColors = null)` on `IServiceCollection` and on `IHostBuilder`.
