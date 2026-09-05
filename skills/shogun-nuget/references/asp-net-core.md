# AlmightyShogun.AspNet.Core

The layer the other web packages build on. It owns the one error body every
package writes, the vocabulary for turning an exception into that body, request
metadata, CORS, and Cloudflare forwarded headers.

Docs: https://nuget.docs.shogun.ms/asp-net-core/

Depends on `AlmightyShogun.AspNet.Localization` and `UAParser`.

## The error body

Every error from every package in the scope is this shape:

```json
{ "code": 404, "error": "not_found", "errorDescription": "..." }
```

`code` is the status code, `error` is the reason phrase snake_cased
(`too_early` for 425, `http_error_<code>` when the framework knows no phrase),
and `errorDescription` is the message resolved from `http-error.<code>`, so
`messages/{language}/http-error.json` must carry a key per status code you
answer.

## What is where

| Need | Reach for |
| --- | --- |
| Write an error body from your own code | `IHttpErrorResponseWriter.WriteAsync` |
| Return an error from a controller action | `new HttpErrorResult(response)` |
| Turn a domain exception into a status and message | implement `IExceptionMapper`, return `ErrorMapping` |
| Register the framework and fallback handlers | `AddExceptionHandling()` |
| Answer non-exception status codes with the same body | `UseHttpErrorResponses()` |
| Client IP and user agent, computed once per request | `httpContext.GetClientContext()` |
| Just the IP, IPv4-mapped addresses unwrapped | `httpContext.GetIpAddress()` |
| Parsed browser, OS, device, bot flag | `httpContext.GetUserAgent()` or `UserAgent.Parse` |
| Credentialed CORS from configuration | `AddCorsPolicy(name, configuration)` |
| Trust Cloudflare's client IP header | `AddCloudflareHeaders()` |
| Delete several cookies | `response.DeleteCookies("a", "b")` |

## Registration order

```csharp
builder.Services
    .AddMessageLocalization(builder.Configuration)   // IMessageResolver
    .AddHttpErrorResponseWriter()                    // IHttpErrorResponseWriter
    .AddAuth(builder.Configuration)                  // its handler, before the fallback
    .AddExceptionHandler<AppExceptionHandler>()      // yours, before the fallback
    .AddExceptionHandling()                          // framework handler + catch-all
    .AddCorsPolicy("DefaultCors", builder.Configuration.GetSection("Cors"));

WebApplication app = builder.Build();

app.UseHttpErrorResponses();
app.UseMessageLocalization();
```

`AddExceptionHandling` registers two handlers: one that answers
`BadHttpRequestException` and turns a client-aborted request into status 499, and
one that answers everything else with a 500 body. Handlers run in registration
order, so anything registered after this call never runs.

`UseHttpErrorResponses` installs an exception handler that does nothing, so the
registered `IExceptionHandler` chain owns the response, then a status-code-pages
handler that renders the same body for a bare status code such as a 404 from
routing.

## Task recipes

### Map a domain exception

The mapper decides status, code, and message key. The handler is yours to write,
which is what lets it decline and let another handler answer:

```csharp
public sealed class AppExceptionMapper : IExceptionMapper
{
    public ErrorMapping? Map(Exception exception) => exception switch
    {
        AccountLockedException locked => new ErrorMapping
        {
            StatusCode = StatusCodes.Status423Locked,
            Code = "account_locked_out",
            MessageKey = "auth.locked-out",
            MessageParameters = [locked.LockoutEnd]
        },
        _ => null
    };
}
```

Return `null` to decline. A mapper is normally a singleton on a failing request,
so keep it a pattern match: no configuration reads, no database calls.

### CORS from configuration

```json
{ "Cors": { "AllowedOrigins": ["https://app.example.com"], "AllowedHeaders": [], "AllowedMethods": [] } }
```

The policy always allows credentials. An empty `AllowedHeaders` or
`AllowedMethods` array means any header or any method. The three keys are read
relative to whatever `IConfiguration` you pass, so passing the root expects them
at the top level.

### Cloudflare in front

```csharp
builder.Services.AddCloudflareHeaders();
```

Configures `ForwardedHeadersOptions` to read `CF-Connecting-IP`, clears the known
proxies and networks, and adds Cloudflare's published ranges. You still have to
call `app.UseForwardedHeaders()`; this only configures the options.

## Traps

**`AddExceptionHandling` does not register the writer.** The handlers take
`IHttpErrorResponseWriter` and `IMessageResolver` by constructor, so
`AddHttpErrorResponseWriter` and `AddMessageLocalization` must both be
registered or the host fails to resolve them.

**Anything registered after `AddExceptionHandling` is dead code.** Its fallback
answers every exception whose response has not started.

**`AddAuth` and `AddAuthCredentials` register their own handlers by default.**
Call them before `AddExceptionHandling`, or pass
`registerExceptionHandler: false` and register your own.

**`AddExceptionHandling` also sets `ApiBehaviorOptions.SuppressMapClientErrors`
to true** by default, so MVC stops rewriting client errors into
`ProblemDetails` and the package's body survives. Pass
`suppressMapClientErrors: false` to keep the framework behavior.

**A wildcard origin throws at startup.** `AddCorsPolicy` rejects `"*"` in
`AllowedOrigins`, because the policy always allows credentials and browsers
reject that combination.

**`AddCloudflareHeaders` clears `KnownProxies` and `KnownIPNetworks`.** Any
proxy you trusted before the call is dropped; pass `additionalNetworks` instead.
The Cloudflare range list is a static snapshot in the package.

**`ClientContext` is cached per request** under a private key in
`HttpContext.Items`. `GetClientContext` builds it from the connection's remote
address and the raw `User-Agent` header on first call, so calling it before
forwarded headers run caches the proxy's address. `SetClientContext` overwrites
the cache.

**`GetUserAgent` parses on every call.** It is not cached, unlike
`ClientContext`, and returns `Unknown` fields rather than null for an empty
header.

**`HttpErrorCodes` is internal.** The error string is derived for you; there is
no public helper to map a status code to it.

## Public surface

Registration: `AddCorsPolicy(string, IConfiguration)`,
`AddCloudflareHeaders(string, IEnumerable<IPNetwork>?, int?)`,
`AddHttpErrorResponseWriter()`, `AddExceptionHandling(bool)` on
`IServiceCollection`; `UseHttpErrorResponses()` on `IApplicationBuilder`.

Errors: `HttpErrorResponse` (`Code`, `Error`, `ErrorDescription`),
`HttpErrorResult : ObjectResult`, `IHttpErrorResponseWriter.WriteAsync`,
`IExceptionMapper.Map`, `ErrorMapping` (`StatusCode`, `Code`, `MessageKey`,
`MessageParameters`).

Request metadata: `ClientContext` (`IpAddress`, `UserAgent`), `UserAgent`
(`Browser`, `Os`, `Device`, `IsBot`, `Parse`), `HttpContext.GetClientContext()`,
`SetClientContext(ClientContext)`, `GetIpAddress()`, `GetUserAgent()`.

Cookies: `HttpResponse.DeleteCookies(params string[])`.

Constants: `CloudflareDefaults.ClientIpHeader`, `CloudflareDefaults.Networks`.
