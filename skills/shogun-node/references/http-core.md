# @almighty-shogun/http-core

Runtime-agnostic HTTP vocabulary: status and method constants, the
`HttpBaseResponse` factory class, query-string readers and the error classes both
server packages throw.

Verify signatures against the copy installed in the project, resolving the
declaration entry from the package's own `exports` rather than assuming a path.

Docs: https://node.docs.shogun.ms/http-core/

## You usually do not install this

`bun-server` and `cloudflare-worker` re-export **every** export of this package
from their own root. If the project has either, import from that instead and do
**not** add `http-core` to `package.json`. Two resolved copies break the
`instanceof HttpBaseResponse` check a handler result is tested against, turning a
valid response into `InvalidHandlerResultError`.

Install it directly only when building your own server wrapper for a runtime
neither package covers.

## Responses

`HttpBaseResponse` is never constructed. Every response comes from a static
factory, and each returns an `HttpBaseResponse` that a server package unwraps
into a native `Response` for you. Call `.unwrap()` yourself only outside a
handler.

```ts
response.json({ id: 1 });
response.json(user, { status: HttpStatus.Created });
response.text('pong');
response.html('<h1>Hi</h1>');
response.image(bytes, { contentType: 'image/webp' });
response.redirect('/login');
```

Named status helpers, when the status is the point:

```ts
response.ok(body);            response.created(body);
response.accepted(body);      response.noContent();
response.notModified();       response.badRequest(body);
response.unauthorized(body);  response.forbidden(body);
response.notFound(body);      response.notAllowed(body);
response.conflict(body);      response.unprocessableEntity(body);
response.tooManyRequests(body); response.internalServerError(body);
```

`custom(body, status, options)` covers anything else.

Options differ by factory, deliberately:

| Options type | Fields | Used by |
|---|---|---|
| `CoreOptions` | `status`, `headers` | `json`, `html`, `text`, `noContent`, `notModified` |
| `ImageOptions` | `CoreOptions` plus `contentType` | `image` |
| `FixedStatusOptions` | `headers`, `contentType` | every named status helper, and `custom` |
| `RedirectOptions` | `status` limited to `RedirectHttpStatus`, `headers` | `redirect` |

A named status helper takes no `status`, because its name is the status.

## Query helpers

`queryString`, `queryInteger`, `queryNumber`, `queryBoolean`, `queryDate`,
`queryList`, `queryNumericList`. Each takes `(request, name, fallback?)`.

**Omitting `fallback` makes the parameter required.** Missing throws
`MissingParameterError`; present but unparseable throws `InvalidParameterError`.
Pass a fallback and both cases return it instead.

```ts
const page = queryInteger(request, 'page', 1);   // optional, defaults to 1
const id = queryString(request, 'id');           // required, throws if absent
```

`queryBoolean` accepts `1`, `on`, `true`, `yes` and the empty string as true, and
`0`, `false`, `no`, `off` as false, case-insensitively. The empty string being
true means `?debug` alone reads as enabled.

`queryDate` parses ISO 8601 through Luxon and returns a `DateTime`.

`queryList` and `queryNumericList` read repeats and commas together, so
`?id=1,2&id=3` gives three values. Values are trimmed and empties dropped, so
`?id=1,%202,,3` gives the same. `queryNumericList` rejects the whole list if any
value is not finite; it never returns a half-parsed list.

## Status and method

`HttpStatus` is a numeric enum covering the standard codes. `HttpMethod` is a
const object plus a matching string union, so `HttpMethod.Get` and `'GET'` are
both accepted wherever a method is expected.

`RedirectHttpStatus` narrows to the five codes that make sense for a redirect.

## Errors

Every class extends `Error`, sets its own `name`, and carries the values that
describe the failure as readonly fields, so a handler never parses a message.

| Error | Carries | Thrown when |
|---|---|---|
| `MissingParameterError` | `parameter` | a required query parameter is absent |
| `InvalidParameterError` | `parameter`, `expected` | present but unparseable |
| `InvalidJsonBodyError` | original error as `cause` | `json()` cannot serialize the body |
| `InvalidHandlerResultError` | `method`, `pathname` | a handler resolved to something other than a response |
| `InvalidRouteCollectionError` | | the collection is not an object |
| `EmptyRouteCollectionError` | | the collection exports nothing |
| `EmptyRouteExportError` | `exportName` | one export is an empty array |
| `DuplicateRouteError` | `method`, `path` | the same method and path twice |
| `ConflictingRoutePathsError` | `existing`, `incoming` | two paths differ only in parameter name |

```ts
try {
    const id = queryInteger(request, 'id');
} catch (error) {
    if (error instanceof MissingParameterError) {
        return response.badRequest({ field: error.parameter });
    }

    throw error;
}
```

## Traps

**A missing parameter and an unparseable one are different errors.** Catching
only `InvalidParameterError` silently lets a missing parameter escape as a 500.
Catch both, or pass a fallback.

**`?debug` with no value is `true`.** The empty string is in the true list. If
you need bare presence to mean something else, read it with `queryString`.

**`json()` can throw.** A circular reference or a value `JSON.stringify` refuses
raises `InvalidJsonBodyError`, with the original `TypeError` as `cause`. It
happens while building the response, not while sending it.

**`redirect()` puts the target in the first argument**, not in an options field,
and defaults to `302 Found`. `redirect(url, { status: HttpStatus.MovedPermanently })`
for a permanent one.

**Named status helpers reject `status`.** They take `FixedStatusOptions`, which
has no `status` field, so `notFound(body, { status: 410 })` does not compile. Use
`custom(body, HttpStatus.Gone)`.

**The route errors live here, not in the server packages.** `DuplicateRouteError`
and `ConflictingRoutePathsError` are shared, while the HTML-route and scheduling
errors belong to `bun-server` and `cloudflare-worker` respectively.

## Exports

Response: `HttpBaseResponse`.

Query: `queryString`, `queryInteger`, `queryNumber`, `queryBoolean`, `queryDate`,
`queryList`, `queryNumericList`.

Constants: `HttpStatus`, `HttpMethod`.

Errors: `MissingParameterError`, `InvalidParameterError`, `InvalidJsonBodyError`,
`InvalidHandlerResultError`, `InvalidRouteCollectionError`,
`EmptyRouteCollectionError`, `EmptyRouteExportError`, `DuplicateRouteError`,
`ConflictingRoutePathsError`.

Types: `CoreOptions`, `ImageOptions`, `FixedStatusOptions`, `RedirectOptions`,
`DefaultErrorResponse`, `ImageContentType`, `RedirectHttpStatus`.
