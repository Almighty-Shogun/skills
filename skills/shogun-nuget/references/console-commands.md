# AlmightyShogun.ConsoleCommands

Attribute-discovered command classes dispatched from a console input loop. A
command is a class, its parameters are the method's parameters, and the loop
binds typed arguments from what the operator typed.

Docs: https://nuget.docs.shogun.ms/console-commands/

Depends on `AlmightyShogun.Utils` for discovery and registration.

## What is where

| Need | Reach for |
| --- | --- |
| Register the loop | `AddConsoleCommands()` |
| Discover and register the commands | `RegisterConsoleCommands()` or `RegisterConsoleCommands(assemblies)` |
| Run the loop | `IConsoleCommandHandler.StartAsync(cancellationToken)` |
| Stop the loop | `IConsoleCommandHandler.Stop()` |
| React to a failing command | `IConsoleCommandHandler.CommandFailed` |
| Declare a command | `[ConsoleCommand("name", "description")]` on a `ConsoleCommandBase` |
| Add alternative names | `[Alias("q", "exit")]` |
| Document usage | `[Example(...)]`, read back through `ConsoleCommandDiscovery` |
| List commands for a help command | `ConsoleCommandDiscovery.GetAllCommands()` |
| Keep one class out of the loop | `[SkipAutoRegistration]` |

## Writing a command

```csharp
[Alias("p")]
[Example("example.com", 3)]
[ConsoleCommand("ping", "Pings a host.")]
public sealed class PingCommand(IPinger pinger) : ConsoleCommandBase
{
    public async Task ExecuteAsync(string host, int times = 1, CancellationToken cancellationToken = default)
        => await pinger.PingAsync(host, times, cancellationToken);
}
```

The rules the base class enforces in its constructor, throwing
`InvalidOperationException` when broken:

- the class carries `ConsoleCommandAttribute`;
- the command name is non-blank and contains no whitespace, because input is
  split on spaces;
- there is exactly one public instance method named `ExecuteAsync`;
- that method returns `Task` or `ValueTask`, never a value.

Commands are resolved from a scope per invocation, so constructor injection of
scoped services works.

## Argument binding

Input is split on spaces, the first token selects the command, the rest bind to
the parameters in order. A trailing `CancellationToken` parameter is not bound
from input; the loop passes its own.

Values convert through `Convert.ChangeType` with the invariant culture, then a
`TypeConverter` fallback. Enums parse case-insensitively and must be a defined
value. `Nullable<T>` binds as `T`.

Too few arguments for the parameters without defaults, or too many unless
`IgnoreExtraArgs` is set, logs a warning and the command does not run. A value
that will not convert logs a warning naming the parameter and the command does
not run.

## Traps

**`RegisterConsoleCommands()` scans the calling assembly.** Pass assemblies
explicitly when commands live elsewhere.

**Registration and dispatch are separate calls.** `AddConsoleCommands` adds the
handler, `RegisterConsoleCommands` adds the commands; the loop dispatches nothing
without the second.

**Names and aliases are matched case-insensitively, first registration wins.** A
duplicate logs a warning and the later class is never reachable under that name.

**A command class that does not inherit `ConsoleCommandBase` throws at handler
construction**, so a misdeclared command fails when the handler is resolved, not
when it is typed.

**`StartAsync` twice is refused with an error log**, not an exception, and so is
`Stop` while not running.

**The loop sets `Console.TreatControlCAsInput = false`** and reads lines until
input ends or the token cancels. A null line, meaning redirected input that ran
out, ends the loop.

**The line just typed is erased** through `ConsoleUtils.RemoveLastLine` before
the command runs, which is why command output is expected to go through a
logger.

**A failing command is logged and raises `CommandFailed`, the loop continues.**
Only a cancellation matching the loop's own token propagates.

**Arguments cannot contain spaces.** There is no quoting: the split discards
empty entries and every token is one argument.

**`Usage` and `Example` are derived, not validated.** `ConsoleCommand.Usage` is
built from the parameter names and types; `[Example]` joins whatever objects you
pass with spaces and is prefixed with the command name.

## Public surface

Registration: `AddConsoleCommands()`, `RegisterConsoleCommands()`,
`RegisterConsoleCommands(Assembly[])` on `IServiceCollection`.

Authoring: `ConsoleCommandBase`, `ConsoleCommandAttribute(name, description,
ignoreExtraArgs)`, `AliasAttribute(params string[])`,
`ExampleAttribute(params object[])`.

Runtime: `IConsoleCommandHandler` (`StartAsync`, `Stop`, `CommandFailed`),
`ConsoleCommandErrorEvent` (`CommandName`, `Exception`).

Introspection: `ConsoleCommandDiscovery.GetAllCommands()` and the assembly
overload, returning `ConsoleCommand` (`Name`, `Description`, `Aliases`, `Usage`,
`Example`).
