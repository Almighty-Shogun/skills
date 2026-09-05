# @almighty-shogun/bun-server

Typed routing, responses and server setup for a Bun HTTP server.

Verify signatures against the copy installed in the project, resolving the
declaration entry from the package's own `exports` rather than assuming a path.

Docs: https://node.docs.shogun.ms/bun-server/

Re-exports **every** `@almighty-shogun/http-core` export from its own root, so a
route file imports `HttpStatus`, `queryInteger` or `MissingParameterError` from
this package. See `http-core.md`. Do not install `http-core` separately.

## Setup

```sh
bun add @almighty-shogun/bun-server
```

Bun only. `@types/bun` is an optional peer, since the published types reference
`Bun.Server`, `Bun.BunRequest` and `Bun.HTMLBundle`.

```ts
import { createServer } from '@almighty-shogun/bun-server';
import * as routes from './routes';

createServer({ routes, port: 3000 });
```

`createServer` forwards every option it does not own straight to `Bun.serve()`,
so `port`, `hostname`, `tls`, `websocket` and the rest behave natively.

## What is where

| Need | Reach for |
|---|---|
| Start a server | `createServer({ routes, port })` |
| Define a route | `defineRoute(path, method, handler)` |
| Serve an HTML bundle | `defineHtmlRoute(paths, bundle)` |
| Return a response | `HttpResponse` statics, called on the `response` argument |
| Return a file | `response.file(body, { contentType })` |
| Pass Bun's own route shape straight through | `routeMode: 'native'` |
| Handle a thrown error | Bun's `error` option on `createServer` |
| Inspect or drive the compiled map yourself | `compileRoutes` |
| Status codes, query readers, HTTP errors | re-exported from `http-core`, see `http-core.md` |

## Task recipes

### Define a route

`defineRoute(path, method, handler)`. The path is a literal type, so `params` is
typed from it: `/users/:userId/posts/:postId` gives both keys.

```ts
import { defineRoute } from '@almighty-shogun/bun-server';

export const user = defineRoute('/users/:id', 'GET', (request, response) => {
    return response.json({ id: request.params.id });
});
```

The handler receives `(request, response, server)`:

- `request` is Bun's `BunRequest`, a `Request` carrying decoded `params`.
- `response` is the `HttpResponse` **class**, not an instance. Statics only.
- `server` is the live `Bun.Server`, for `server.upgrade(request)` and similar.

It must resolve to an `HttpResponse`. Anything else throws
`InvalidHandlerResultError` at request time.

### Serve an HTML bundle

`defineHtmlRoute(path, bundle)` takes Bun's `HTMLBundle` import, and one path or
an array of paths.

```ts
import index from './index.html';
import { defineHtmlRoute } from '@almighty-shogun/bun-server';

export const app = defineHtmlRoute(['/', '/dashboard'], index);
```

Bun serves these itself. They never reach a handler, so no method routing,
automatic `HEAD` or `OPTIONS` applies to them.

### Build the collection

An object of named exports, each a route, an HTML route, or an array of either.

```ts
// routes/index.ts
export * from './users';
export * from './app';

// server.ts
import * as routes from './routes';

createServer({ routes, port: 3000 });
```

### Return a file

`HttpResponse.file(body, options)` is the one factory this package adds on top of
`http-core`.

```ts
return response.file(Bun.file('./report.pdf'), {
    contentType: 'application/pdf'
});
```

### Use Bun's native route format

When routes are already in `Bun.serve({ routes })` shape, set `routeMode:
'native'` and they are passed through untouched.

```ts
createServer({ routeMode: 'native', routes: { '/': new Response('ok') } });
```

Native mode adds none of this package's behavior: no automatic `HEAD`, no
automatic `OPTIONS`, no generated `405`, no `Allow` headers. The types enforce
this, so `automaticHead` and `automaticOptions` are `never` in that mode.

## Behavior worth knowing

**Bun owns path matching**, unlike the Worker package. Routes compile to an
object keyed by path, handed to `Bun.serve()`, so specificity and ordering follow
Bun's rules rather than anything this package computes.

**`HEAD` and `OPTIONS` are automatic**, both on by default. A `HEAD` with no
handler runs the `GET` handler and returns its status, status text and headers
with a null body. An `OPTIONS` with no handler returns `204` with `Allow`. A
`405` carries `Allow` too.

**Collection problems throw at `createServer()` time**, not per request, since
compilation happens once at startup. A duplicate method and path throws
`DuplicateRouteError`; two paths differing only in parameter name throw
`ConflictingRoutePathsError`; the same path registered as both an HTML route and
a method route throws `ConflictingRouteTypeError`.

**A thrown error is caught by Bun's own `error` option.** `createServer` installs
a default that answers `500` shaped by `defaultErrorResponse`. Pass your own
`error` to replace it entirely.

## Traps

**`response` is the class, not an instance.** `response.json(body)` is a static
call. No `new`, nothing to configure first.

**HTML routes bypass everything.** They are not handlers, so a path served by
`defineHtmlRoute` gets no automatic `HEAD`, no `OPTIONS` and no `405`. Bun serves
the bundle directly.

**Error handling is Bun's `error` option, not an `onError` option.** That is the
opposite of `cloudflare-worker`, which owns its dispatch and so takes `onError`.
Here the hook is passed straight through to `Bun.serve()`.

**`file()` returns `HttpBaseResponse`, not a Bun response.** Return it as-is from
a handler; do not unwrap it yourself.

**Two copies of `http-core` break responses.** Handler results are checked with
`instanceof HttpBaseResponse`. Two resolved versions make a valid response fail
and raise `InvalidHandlerResultError`. Keep one resolved copy, and do not add
`http-core` as a direct dependency.

**An empty path array is an error.** `defineHtmlRoute([], bundle)` throws
`EmptyHtmlRoutePathsError` during compilation, on the theory that a route
registering nothing is a mistake.

**Native mode silently drops the package's behavior.** Switching `routeMode` to
`'native'` to fix one route removes automatic `HEAD`, `OPTIONS` and `405` from
every route at once.

**`defineHtmlRoute` freezes what it returns**, and copies a path array, so
mutating the array afterwards does not change the route.

## Exports

Runtime: `createServer`, `defineRoute`, `defineHtmlRoute`, `compileRoutes`,
`HttpResponse`.

Errors: `EmptyHtmlRoutePathsError`, `DuplicateHtmlRouteError` (carries `path`),
`ConflictingRouteTypeError` (carries `path`).

Types: `RouteHandler`, `RouteDefinition`, `HtmlRouteDefinition`, `RouteExport`,
`RouteCollection`, `NativeRouteCollection`, `CompileRoutesOptions`,
`CreateServerOptions`, `CreateServerDefinedOptions`, `CreateServerNativeOptions`.

Plus everything from `http-core`.

`compileRoutes` is exported for inspecting the compiled map or driving
`Bun.serve()` yourself. `createServer` calls it for you.
