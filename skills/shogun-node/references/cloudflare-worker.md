# @almighty-shogun/cloudflare-worker

Typed routing, responses and cron scheduling for a Cloudflare Worker module.

Verify signatures against the copy installed in the project, resolving the
declaration entry from the package's own `exports` rather than assuming a path.

Docs: https://node.docs.shogun.ms/cloudflare-worker/

Re-exports **every** `@almighty-shogun/http-core` export from its own root, so a
route file imports `HttpStatus`, `queryInteger` or `MissingParameterError` from
this package. See `http-core.md`. Do not install `http-core` separately.

## Setup

```sh
bun add @almighty-shogun/cloudflare-worker
```

Needs Wrangler 4+, and `@cloudflare/workers-types` as an optional peer, since the
published types reference `ExecutionContext` and `ScheduledController`.

The entry file's **default export** must be the value `createWorker()` returns,
because Cloudflare calls `fetch` and `scheduled` on it.

```ts
import { createWorker } from '@almighty-shogun/cloudflare-worker';
import * as routes from './routes';

export default createWorker({ routes });
```

`WorkerEnv` is an empty interface you augment with your bindings. The file doing
the augmenting **must be a module**, so it needs a top-level `import` or
`export`. Without one, TypeScript reads `declare module` as an ambient
declaration that replaces the package's types, and every import from the package
silently resolves to `any`.

```ts
export {};

declare module '@almighty-shogun/cloudflare-worker' {
    interface WorkerEnv {
        CACHE: KVNamespace;
        ASSETS: Fetcher;
    }
}
```

## What is where

| Need | Reach for |
|---|---|
| Export the worker | `createWorker({ routes })` as the default export |
| Define a route | `defineRoute(path, method, handler)` |
| Schedule a task | `defineScheduled(cron, handler)` |
| Declare bindings | augment `WorkerEnv` |
| Serve static assets | `assets: 'BINDING_NAME'` |
| Handle a thrown error | `onError(error, request, env)` |
| Return a response | `HttpResponse` statics, `response.from(upstream)` for a native one |
| Inspect compiled output | `compileRoutes`, `compileScheduled` |
| Status codes, query readers, HTTP errors | re-exported from `http-core`, see `http-core.md` |

## Task recipes

### Define a route

`defineRoute(path, method, handler)`. The path is a literal type, so `params` is
typed from it: `/users/:id` gives `params.id`.

```ts
import { defineRoute } from '@almighty-shogun/cloudflare-worker';

export const user = defineRoute('/users/:id', 'GET', (request, response) => {
    return response.json({ id: request.params.id });
});
```

The handler receives `(request, response, env)`:

- `request` is a `Request` plus `params` and `ctx`, the `ExecutionContext`, so
  `request.ctx.waitUntil(...)` works.
- `response` is the `HttpResponse` class itself, not an instance. Call statics on
  it: `response.json(...)`, `response.notFound()`, `response.from(upstream)`.
- `env` is your augmented `WorkerEnv`.

It must resolve to an `HttpResponse`. Returning a native `Response` throws
`InvalidHandlerResultError` at request time.

### Build the collection

A collection is an object of named exports, one entry per export, each a route or
an array of routes. A namespace import is the normal way to build it.

```ts
// routes/index.ts
export * from './users';
export * from './health';

// worker.ts
import * as routes from './routes';

export default createWorker({ routes });
```

Export names are only used for error messages and ordering; they never affect
matching.

### Schedule a task

`defineScheduled(cron, handler)`, collected the same way as routes, passed as
`scheduled`.

```ts
import { defineScheduled } from '@almighty-shogun/cloudflare-worker';

export const nightly = defineScheduled('5 2 * * *', async (run, env) => {
    await env.CACHE.delete('report');
});
```

The handler receives `(run, env)`, where `run` holds `controller`, `ctx` and
`cron`. Register the same expression in `wrangler.toml`:

```toml
[triggers]
crons = ["5 2 * * *"]
```

Several tasks may share one expression; they are collected into one entry and run
in the order their exports were collected, which is export name order.

### Serve static assets

`assets` takes the **name of a binding key** on `WorkerEnv`, not the binding.

```ts
export default createWorker({ routes, assets: 'ASSETS' });
```

Assets are served only when **no route matches the path at all**. A path that
matches a route but not its method gives `405`, never the asset.

### Handle errors centrally

`onError(error, request, env)` returns an `HttpResponse` to send instead, or
`null` to fall through to the generated response.

```ts
export default createWorker({
    routes,
    onError: (error, request) => {
        console.error(request.url, error);

        return null;
    }
});
```

The generated error response never includes the thrown message, so an internal
failure cannot leak through a public route. `defaultErrorResponse` controls its
shape: `'json'`, `'text'`, or `null` for an empty body.

## Behavior worth knowing

**Route order is computed, not declaration order.** Compiled routes sort segment
by segment, static before dynamic, then alphabetically, and the first match wins.
So `/users/me` beats `/users/:id` regardless of which was declared first.

**Matching is exact on segment count.** There are no optional or wildcard
segments. `/users/:id` does not match `/users`, and nothing matches a trailing
extra segment. Parameters are `decodeURIComponent`-ed.

**`HEAD` and `OPTIONS` are automatic**, both defaulting on. A `HEAD` with no
handler runs the `GET` handler and returns its status, status text and headers
with a null body. An `OPTIONS` with no handler returns `204` with `Allow`. A
`405` also carries `Allow`. Turn either off with `automaticHead: false` or
`automaticOptions: false`.

**Collection problems throw at `createWorker()` time**, not per request, because
compilation happens once at module scope. A duplicate method and path throws
`DuplicateRouteError`; two paths differing only in parameter name, such as
`/users/:id` and `/users/:userId`, throw `ConflictingRoutePathsError`.

**Scheduled tasks run sequentially and awaited.** A task that throws does not
cancel the ones after it. When exactly one failed, that error is rethrown as-is;
when several did, they are wrapped in a `ScheduledTasksFailedError` carrying
`cron` and, as an `AggregateError`, every individual failure in `.errors`.

**A trigger with no matching expression does nothing**, silently. Matching is
exact string equality against `controller.cron` after trimming, so the string in
`wrangler.toml` must match the one passed to `defineScheduled`.

## Traps

**Cron Triggers run in UTC and ignore daylight saving.** "Every day at 18:05
Europe/Amsterdam" has no correct cron expression: `5 16 * * *` is right in summer
and an hour out in winter. Either accept the shift, or schedule hourly and check
the local time inside the handler. Ask which; there is no safe default.

**`onError` does not cover scheduled tasks.** It is only wired into `fetch`. An
error thrown by a scheduled handler propagates to the runtime, which is what makes
Cloudflare record the failure and retry, so do not treat that as a bug to catch.

**`response` is the class, not an instance.** `response.json(body)` is a static
call. There is no `new`, and no instance to configure first.

**Two copies of `http-core` break responses.** The worker checks handler results
with `instanceof HttpBaseResponse`. Two resolved versions make a perfectly valid
response fail that check and raise `InvalidHandlerResultError`. Keep exactly one
resolved copy, and do not add `http-core` as a direct dependency.

**Returning a native `Response` compiles until it does not.** The handler type
requires an `HttpBaseResponse`, but a handler typed loosely, or one returning
`fetch(...)` directly, only fails at request time. Wrap upstream responses with
`HttpResponse.from(upstream)`.

**An empty array in a collection is an error, not an empty group.**
`export const admin = []` throws `EmptyScheduledExportError` or
`EmptyRouteExportError` naming that export, on the theory that an export
registering nothing is a mistake rather than an intentional no-op.

**Assets do not shadow a bad method.** If you expect `/index.html` to fall
through to assets but have a route on that exact path, only the method mismatch
path applies, and you get `405`.

## Exports

Runtime: `createWorker`, `defineRoute`, `compileRoutes`, `defineScheduled`,
`compileScheduled`, `HttpResponse`.

Errors: `InvalidScheduledCollectionError`, `EmptyScheduledCollectionError`,
`EmptyScheduledExportError` (carries `exportName`), `EmptyCronExpressionError`,
`ScheduledTasksFailedError` (extends `AggregateError`, carries `cron`).

Types: `WorkerEnv` (augment it), `CreateWorkerOptions`, `WorkerModule`,
`WorkerErrorHandler`, `AssetsBinding`, `RouteRequest`, `RouteHandler`,
`RouteDefinition`, `RouteExport`, `RouteCollection`, `CompiledRoute`,
`CompiledRouteCollection`, `ScheduledRun`, `ScheduledHandler`,
`ScheduledDefinition`, `ScheduledExport`, `ScheduledCollection`,
`CompiledScheduled`, `CompiledScheduledCollection`.

Plus everything from `http-core`.

`compileRoutes` and `compileScheduled` are exported for inspecting compiled
output or building a custom dispatcher. `createWorker` calls both for you.
