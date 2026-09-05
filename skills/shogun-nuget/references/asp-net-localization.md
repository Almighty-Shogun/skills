# AlmightyShogun.AspNet.Localization

Request messages resolved from JSON files in the caller's language. Every other
web package writes its user-facing text through `IMessageResolver`, so this
package is registered wherever `AspNet.Core` writes an error body.

Docs: https://nuget.docs.shogun.ms/asp-net-localization/

Depends on `AlmightyShogun.Utils` and the ASP.NET Core framework reference.

## What is where

| Need | Reach for |
| --- | --- |
| Register the resolver and message provider | `AddMessageLocalization(configuration)` |
| Stamp `Content-Language` on every response | `UseMessageLocalization()` |
| Resolve a message | `IMessageResolver.Resolve(key)` or `Resolve(key, parameters)` |
| Know which language answered | `IMessageResolver.ResolveLanguage()` |
| Read the request's languages | `request.GetAcceptLanguage()`, `request.GetAcceptLanguages()` |
| Read or set the response language | `response.GetContentLanguage()`, `response.TrySetContentLanguage(language)` |
| Decide the language differently | implement `ILanguageProvider` |

## Message files

Files live under `messages/{language}/*.json`, searched in this order, first hit
per key winning: the content root, `AppContext.BaseDirectory`, then the current
directory. Within one root the files are read in ordinal name order.

The file name becomes the first key segment, nested objects add segments, and a
final `default` segment is dropped:

```text
messages/en/validation.json
{ "required": "This field is required.",
  "date": { "default": "Must be a date.", "format": "Must match {0}." } }

validation.required
validation.date
validation.date.format
```

A key equal to the file name is not prefixed twice, so `"validation"` inside
`validation.json` stays `validation`.

## Task recipes

### Register it

```csharp
builder.Services.AddMessageLocalization(builder.Configuration);

app.UseMessageLocalization();
```

Binds the `Localization` section, adds `IHttpContextAccessor`, and registers
`ILanguageProvider` and `IMessageResolver` with `TryAdd`, so registering your own
before this call keeps it.

### Resolve a message with parameters

```csharp
string message = messageResolver.Resolve("auth.locked-out", [lockoutEnd]);
```

Parameters go through `string.Format` with the culture of the resolved language,
falling back to the invariant culture when the tag is not a known culture.

### Replace the language source

```csharp
builder.Services.AddSingleton<ILanguageProvider, HeaderThenProfileLanguageProvider>();
builder.Services.AddMessageLocalization(builder.Configuration);
```

The default provider reads `Accept-Language` through `IHttpContextAccessor` and
falls back to `Localization:DefaultLanguage`.

## Traps

**A missing key returns the key itself** and logs a warning. Nothing throws, so a
typo ships as a response body reading `auth.locked-out`.

**`ResolveLanguage` picks the first candidate that has any message at all.** The
candidates are the request's accepted languages, each one's shortened fallbacks
(`nl-BE` then `nl`), then the default. A language whose directory exists but is
empty is skipped, and a language with one unrelated message wins for keys it does
not contain.

**Language tags are validated against `^[A-Za-z]{2,3}(-[A-Za-z0-9]{2,8})*$`.**
A malformed tag is ignored with a warning and yields no messages.

**Messages are cached per language until a file changes**, and only when
`Localization:AutomaticReload` is true does the package watch the message
directories. With it off, editing a file at runtime changes nothing.

**Watching starts on the first `GetMessages` call**, not at startup, and only for
directories that exist at that moment. A `messages` directory created later is
not watched.

**Unreadable or malformed message files are skipped with a warning**, so a broken
JSON file silently reduces the set of keys.

**`TrySetContentLanguage` returns false rather than throwing** when the response
has started, the value is blank, or it contains a control character. The
middleware sets the header in `OnStarting` only when nothing else set it.

## Public surface

Registration: `AddMessageLocalization(IConfiguration)` on `IServiceCollection`,
`UseMessageLocalization()` on `IApplicationBuilder`.

Services: `IMessageResolver` (`Resolve(string)`,
`Resolve(string, IReadOnlyList<object?>)`, `ResolveLanguage()`),
`ILanguageProvider` (`GetLanguage()`, `GetLanguages()` with a default
implementation returning the single language).

Extensions: `HttpRequest.GetAcceptLanguage()`,
`HttpRequest.GetAcceptLanguages()`, `HttpResponse.GetContentLanguage()`,
`HttpResponse.TrySetContentLanguage(string)`.

Configuration: `LocalizationSettings` with `DefaultLanguage` and
`AutomaticReload`.
