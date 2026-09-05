# AlmightyShogun.Utils

Framework-agnostic infrastructure: assembly scanning and convention
registration, validated options binding, two JSON helpers, and console helpers.
No ASP.NET dependency, so it is the one package a console app or a worker can
take on its own.

Docs: https://nuget.docs.shogun.ms/utils/

Depends only on `Microsoft.Extensions.*` abstractions: options, configuration
binder, options data annotations, DI abstractions.

Usually already restored: every other package in the scope except `Serilog` and
`EntityFrameworkCore.ModelBuilding` references it. Declare it directly when your
own code calls into it.

## What is where

| Need | Reach for |
| --- | --- |
| Register every implementation of a base type or interface | `RegisterOnInherit<T>` |
| Group one feature's registrations behind a type | `IServiceRegistry` plus `AddService<T>` |
| Bind and validate an options section | `AddConfiguration<T>` |
| Swap a framework registration | `ReplaceService<TService, TImplementation>` |
| Opt one class out of auto registration | `[SkipAutoRegistration]` |
| Find types without registering them | `TypeDiscovery.FindAssignableTypes<T>` |
| Parse JSON that may be malformed | `json.TryDeserialize<T>(out value)` |
| Read a JSON stream | `stream.DeserializeAsync<T>()` |
| Prompt on the console | `ConsoleUtils.AskQuestionAsync` |
| Keep Ctrl+C from killing the process | `ConsoleUtils.PreventCancellation` |

## Task recipes

### Register every implementation of an interface

```csharp
using AlmightyShogun.Utils;

builder.Services.RegisterOnInherit<IJob>();
```

The parameterless overload scans the **calling** assembly only. Pass assemblies
explicitly when the types live elsewhere:

```csharp
builder.Services.RegisterOnInherit<IJob>(
    [typeof(SomeJob).Assembly],
    ServiceLifetime.Scoped,
    registerAsBaseType: false
);
```

`registerAsBaseType: true`, the default, registers every implementation under
`IJob`, so `IEnumerable<IJob>` resolves them all. `false` registers each concrete
type under itself, which is what you want when a consumer resolves a specific
command or job type. An optional `filter` narrows the set further.

### Group a feature's registrations

```csharp
public sealed class MailRegistry : IServiceRegistry
{
    public void ConfigureService(IServiceCollection serviceCollection)
        => serviceCollection.AddSingleton<IMailer, Mailer>();
}

builder.Services.AddService<MailRegistry>();
```

`AddService<T>` requires a public parameterless constructor and instantiates the
registry itself, so a registry cannot take dependencies.

### Bind an options section with validation

```csharp
builder.Services.AddConfiguration<MailSettings>(builder.Configuration.GetSection("Mail"));
```

Data-annotation validation and validation on start are both on by default, so a
missing `[Required]` key fails the host at startup. Pass
`validateDataAnnotations: false` or `validateOnStart: false` to turn either off.
This is the call every other package in the scope uses to bind its own section.

### Parse JSON defensively

```csharp
if (json.TryDeserialize(out Payload? payload))
{
    // payload is not null here
}

Payload? fromStream = await stream.DeserializeAsync<Payload>(cancellationToken: cancellationToken);
```

Both default to `JsonSerializerDefaults.Web`, so camelCase property names and
case-insensitive matching, unless you pass your own options.

## Traps

**`RegisterOnInherit<T>()` and `TypeDiscovery.FindAssignableTypes<T>()` scan the
calling assembly.** Called from a library on behalf of an application, they scan
the library, not the application. Pass the assemblies explicitly whenever the
call site is not the assembly that holds the types.

**Discovery skips interfaces and abstract classes**, and `RegisterOnInherit`
additionally skips anything carrying `[SkipAutoRegistration]`. The attribute is
only honored on the type itself, not on a base type.

**A load failure is swallowed.** `TypeDiscovery` catches
`ReflectionTypeLoadException` and returns the types that did load, so a broken
dependency shows up as a missing registration rather than an exception.

**`TryDeserialize` returns `false` for the JSON literal `null`.** It only catches
`JsonException`, so it reports failure for malformed JSON and for a payload that
deserializes to null, and lets anything else propagate.

**`ReplaceService` defaults to `ServiceLifetime.Singleton`** and replaces only
the first matching descriptor, following `ServiceCollectionDescriptorExtensions.Replace`.

**`ConsoleUtils.RemoveLastLine` is silent when output is redirected**, and
swallows the cursor exceptions a non-interactive console throws.

**`PreventCancellation` is a one-way, process-wide switch.** It is idempotent
through an interlocked flag, but nothing removes the handler afterwards. Prefer
`AlmightyShogun.Hosting.ConsoleLifetime` when a host should still stop on
SIGTERM.

## Public surface

Registration: `IServiceRegistry`, `AddService<T>`, `AddConfiguration<T>`,
`ReplaceService<TService, TImplementation>`, `RegisterOnInherit<T>` (two
overloads), `SkipAutoRegistrationAttribute`.

Discovery: `TypeDiscovery.FindAssignableTypes<T>()` for the calling assembly,
one assembly, or an assembly array.

JSON: `string.TryDeserialize<T>(out T?, JsonSerializerOptions?)`,
`Stream.DeserializeAsync<T>(JsonSerializerOptions?, CancellationToken)`.

Console: `ConsoleUtils.Title`, `ConsoleUtils.RemoveLastLine`,
`ConsoleUtils.AskQuestionAsync`, `ConsoleUtils.PreventCancellation`.
