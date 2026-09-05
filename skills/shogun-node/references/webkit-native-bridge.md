# @almighty-shogun/webkit-native-bridge

A typed request and response bridge between web code running inside a WebKit host
and its native side.

Verify signatures against the copy installed in the project, resolving the
declaration entry from the package's own `exports` rather than assuming a path.

Docs: https://node.docs.shogun.ms/webkit-native-bridge/

## Setup

```sh
bun add @almighty-shogun/webkit-native-bridge
```

No peer dependencies. Uses `@almighty-shogun/utils` for types only, so nothing
extra reaches the runtime.

Requires a WebKit host exposing `window.webkit.messageHandlers`. Outside one,
every request resolves to a transport failure with code `UNAVAILABLE`, and `call`
and `postMessage` throw.

Both sides must agree on two names, and the defaults are:

| Setting | Default | Meaning |
|---|---|---|
| `handlerName` | `nativeBridge` | the `window.webkit.messageHandlers` key |
| `eventName` | `webkit-native-bridge` | the `CustomEvent` native dispatches back |
| `requestTimeout` | `30000` | milliseconds before a request gives up |

## What is where

| Need | Reach for |
|---|---|
| Create the bridge | `createNativeBridge<Requests, Commands>(options?)` |
| Request something with a response | `bridge.request(method, body?, options?)` |
| Fire a command with no response | `bridge.call(command)` |
| Send a raw string | `bridge.postMessage(value)` |
| Check the host is present | `bridge.isAvailable()` |
| Feed a native response in directly | `bridge.handleResponse(detail)` |
| Tear down | `bridge.dispose()` |
| Narrow a failure | `isTransportError`, `isNativeError`, `getErrorDetailsAs` |
| Flatten a result for a UI | `normalizeBridgeResponse`, `mapBridgeError` |

## Task recipes

### Describe the contract, then create the bridge

The request map is the whole point. Declare one entry per method with its body,
response, and optionally its error code and details, and every call is typed from
it.

```ts
import { createNativeBridge } from '@almighty-shogun/webkit-native-bridge';

type Requests = {
    getUser: {
        body: { id: string };
        response: { id: string; name: string };
        errorCode: 'NOT_FOUND' | 'FORBIDDEN';
    };
    getVersion: {
        body: void;
        response: string;
    };
};

type Commands = 'openSettings' | 'closeWindow';

const bridge = createNativeBridge<Requests, Commands>();
```

### Request and read the result

Every request resolves; it does not reject on a native or transport failure.
Check `ok` first.

```ts
const result = await bridge.request('getUser', { id: '42' });

if (result.ok) {
    console.log(result.data.name);
} else {
    console.warn(result.error.code, result.error.message);
}
```

A method whose `body` is `void` or `undefined` takes no second argument, enforced
by overloads:

```ts
const version = await bridge.request('getVersion');
```

Per-call timeout:

```ts
await bridge.request('getUser', { id: '42' }, { timeout: 5000 });
```

### Fire and forget

`call` sends a command with no response. `postMessage` sends a raw string. Both
**throw** rather than resolving, since neither has a response to carry an error.

```ts
bridge.call('openSettings');
```

### Narrow a failure

```ts
if (!result.ok) {
    if (isTransportError(result.error)) {
        // TIMEOUT | UNAVAILABLE | DISPOSED | UNKNOWN
    }

    if (isNativeError(result.error)) {
        const details = getErrorDetailsAs<{ field: string }>(result.error);
    }
}
```

### Flatten a response

`normalizeBridgeResponse` collapses the union into one shape whose error fields
are never null, which is easier to hand to a UI. `mapBridgeError` does the same
for one error on its own.

```ts
const normalized = normalizeBridgeResponse(result);
```

### Clean up

```ts
onUnmounted(() => bridge.dispose());
```

`dispose` removes the response listener and settles every in-flight request as a
`DISPOSED` transport failure.

## Behavior worth knowing

**Failures arrive two ways, deliberately.** A native or transport failure comes
back as a resolved `BridgeResponse` with `ok: false`. Only two things throw:
`NativeBridgeUnavailableError` when the handler is missing, and
`NativeBridgeDisposedError` after `dispose`. Those mean the bridge could not be
used at all, so there is no response to resolve.

**`request` never throws for a missing handler.** It resolves to a transport
failure with code `UNAVAILABLE`. `call` and `postMessage` do throw, because they
have nowhere to put an error.

**Native replies by dispatching a `CustomEvent`** named `eventName`, whose detail
matches `NativeResponseEventDetail`: `requestId`, `ok`, `payload`, `error`. The
bridge correlates on `requestId`. `handleResponse` is exposed so a host can feed
a response in directly instead of dispatching an event.

## Traps

**A disposed bridge cannot be revived.** `dispose` is final; create a new bridge.
Calling `request` on a disposed one throws `NativeBridgeDisposedError`
synchronously, before the promise exists, so a `.catch()` will not see it.

**`isAvailable()` is a snapshot.** The host can appear later during startup, so a
`false` at module scope does not mean the bridge is unusable a second later.
Prefer letting a request resolve to `UNAVAILABLE` over gating on it.

**Handler and event names must match native exactly.** A mismatched
`handlerName` looks identical to running outside the host: every request resolves
`UNAVAILABLE`. A mismatched `eventName` is worse, because requests are sent
successfully and then every one times out after 30 seconds.

**`requestTimeout: null` disables the timeout**, whereas omitting it gives the
30 second default. A request with no timeout hangs forever if native never
replies.

**`body: void` and `body: undefined` change the call signature.** They select the
overload that takes no body argument, so passing one does not compile.

**Ten types are intentionally not exported.** `NativeRequestBody`,
`NativeResponseBody`, `NativeMethodsWithBody` and similar are internal machinery
that appears inside signatures but cannot be imported. Write the request map and
let inference do the rest.

## Exports

Runtime: `createNativeBridge`, `normalizeBridgeResponse`, `mapBridgeError`,
`isNativeError`, `isTransportError`, `getErrorDetailsAs`.

Errors: `NativeBridgeUnavailableError` (carries `handlerName`),
`NativeBridgeDisposedError`.

Types: `NativeBridge`, `NativeBridgeOptions`, `NativeBridgeRequestMap`,
`NativeBridgeWindow`, `NativeRequestOptions`, `NativeRequestResult`,
`NativeResponseEventDetail`, `BridgeResponse`, `BridgeSuccess`, `BridgeFailure`,
`BridgeError`, `ResolvedBridgeError`, `NormalizedBridgeResponse`,
`NativeTransportErrorCode`, `NativeTransportErrorDetails`.
