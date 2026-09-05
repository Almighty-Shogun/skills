# AlmightyShogun.RemoteCommands

Length-prefixed JSON over raw TCP, dispatched to typed handlers. One package
holds both ends: a listener for the process that owns the commands, and a client
for whatever drives it.

Docs: https://nuget.docs.shogun.ms/remote-commands/

Depends on `AlmightyShogun.Utils` and the `Microsoft.Extensions.*` abstractions.

## What is where

| Need | Reach for |
| --- | --- |
| Register the listener | `AddRemoteCommands(configuration)` |
| Discover and register the commands | `RegisterRemoteCommands()` or the assembly overload |
| Run the listener | `IRemoteCommandHandler.StartAsync(cancellationToken)` |
| Stop the listener | `IRemoteCommandHandler.Stop()` |
| Declare a command | `[RemoteCommand("name")]` on a `RemoteCommand<TMessage>` |
| Answer the caller | `ICommandResponse.WriteAsync(data)` |
| Call a command | `RemoteCommandClient.SendAsync<TMessage, TResponse>(name, message)` |

## The protocol

Each frame is a four-byte big-endian length followed by that many bytes of UTF-8
JSON, serialized with web defaults (camelCase).

A request frame is `RemoteCommandPayload`: `command`, `data`, and an optional
`secret`. A response frame is `RemoteCommandResponse`: either `data`, or a
`refusal` from `RemoteCommandRefusal`.

The connection stays open for further frames until it idles out, the caller
disconnects, or the listener stops.

## Writing a command

```csharp
[RemoteCommand("restart", "Restarts a worker.")]
public sealed class RestartCommand(IWorkerPool pool) : RemoteCommand<RestartMessage>
{
    public override async Task HandleCommandAsync(
        RestartMessage message,
        ICommandResponse response,
        CancellationToken cancellationToken = default
    )
    {
        await pool.RestartAsync(message.WorkerId, cancellationToken);

        await response.WriteAsync(new { restarted = true }, cancellationToken);
    }
}
```

The command resolves from a scope per frame, so scoped dependencies work. Writing
no response is fine: the listener sends an empty envelope when the handler
returns without writing.

## Calling a command

```csharp
await using RemoteCommandClient client = new("127.0.0.1", 5001, secret);

RestartResult? result = await client.SendAsync<RestartMessage, RestartResult>(
    "restart",
    new RestartMessage { WorkerId = 3 },
    cancellationToken
);
```

The client connects lazily and reuses the connection across calls. The overload
without a response type discards the payload.

## Refusals

The listener answers with a refusal rather than an exception:

| Refusal | Cause |
| --- | --- |
| `MalformedPayload` | the frame was not readable JSON |
| `MissingCommandName` | the payload named no command |
| `Unauthorized` | the pre-shared key was missing or wrong |
| `CommandNotFound` | no command registered under that name |
| `InvalidMessage` | the data did not match the command's message type |
| `Other` | the handler threw, or a reason the client does not recognize |

The client turns a refusal into `RemoteCommandRefusedException` carrying the
`Reason`, an unreachable server into `RemoteCommandUnreachableException` with
host and port, and a closed connection into
`RemoteCommandDisconnectedException`, all deriving from `RemoteCommandException`.

## Traps

**The whitelist is mandatory in practice.** A connection whose address matches no
`RemoteServer:Whitelisted` entry is closed without a response, which the client
reports as `RemoteCommandDisconnectedException`, whose message says the address
may not be whitelisted. An empty whitelist rejects everyone.

**Whitelist entries are addresses or CIDR ranges**, and an unparseable entry
throws when the handler is constructed.

**`RemoteServer:Address` is the bind address**, defaulting to `127.0.0.1`, so the
listener is loopback only until you change it. It must parse as an IP address,
not a host name.

**The secret is optional and is a pre-shared key, not a session.** With
`Secret` unset every whitelisted caller is authorized. It is compared as a
SHA-256 hash in fixed time, but the wire carries the secret in clear text, so use
it on a trusted network or behind a tunnel.

**Command names are matched case-sensitively**, unlike `ConsoleCommands`.
Duplicates log a warning and the later class never dispatches.

**A response can be written once.** A second `WriteAsync` on the same frame
throws `InvalidOperationException`.

**Timeouts are per frame.** `IdleTimeout` bounds the wait for the next frame and
`ReadTimeout` bounds handling one; both are seconds, and hitting the idle one
simply closes the connection.

**`MaxPayloadBytes` bounds the listener, and the client has its own fixed
one-megabyte cap.** A declared length outside the accepted range is a protocol
error, not a refusal.

**`MaxConcurrentConnections` is a semaphore**, so past the limit new connections
wait rather than being rejected.

**`StartAsync` twice is refused with an error log**, and on shutdown the listener
waits at most five seconds for in-flight handlers.

**A command class must inherit `RemoteCommand<T>`** and carry the attribute;
either omission throws during registration or handler construction.

## Public surface

Registration: `AddRemoteCommands(IConfiguration)`, `RegisterRemoteCommands()`,
`RegisterRemoteCommands(Assembly[])` on `IServiceCollection`.

Authoring: `RemoteCommand<T>` with `HandleCommandAsync`,
`RemoteCommandAttribute(name, description)`, `ICommandResponse.WriteAsync<T>`.

Runtime: `IRemoteCommandHandler` (`StartAsync`, `Stop`), `RemoteCommandClient`
(`SendAsync<TMessage, TResponse>`, `SendAsync<TMessage>`, `DisposeAsync`).

Protocol: `RemoteCommandPayload`, `RemoteCommandResponse`,
`RemoteCommandRefusal`.

Exceptions: `RemoteCommandException` and its `RemoteCommandRefusedException`,
`RemoteCommandUnreachableException`, `RemoteCommandDisconnectedException`,
`RemoteCommandProtocolException`.

Configuration: `RemoteServerSettings` with `Address`, `Port`, `Whitelisted`,
`Secret`, `EnableReceiveLog`, `MaxPayloadBytes`, `ReadTimeout`, `IdleTimeout`,
`MaxConcurrentConnections`.
