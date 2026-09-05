# @almighty-shogun/utils

Framework-agnostic helpers: Luxon-backed dates, number and currency formatting,
locale data, a type-preserving serializer, browser and DOM helpers, and the
shared type vocabulary the other packages reuse.

Verify signatures against the copy installed in the project, resolving the
declaration entry from the package's own `exports` rather than assuming a path.

Docs: https://node.docs.shogun.ms/utils/

## Setup

```sh
bun add @almighty-shogun/utils
```

Depends on `luxon` at runtime. Every date helper takes and returns a Luxon
`DateTime`, never a native `Date`.

Usually already installed: `common`, `bun-server`, `cloudflare-worker` and
`webkit-native-bridge` all depend on it, so it is in `node_modules` even when it
is not in `package.json`. Add it as a direct dependency when your own code
imports from it.

## What is where

| Need | Reach for |
|---|---|
| Format a date, time or month | `formatDate`, `formatDateTime`, `formatFullDate`, `formatTime`, `formatHour`, `formatMonth`, `formatWeekday` |
| Date arithmetic and predicates | `addTime`, `isToday`, `isBeforeNow`, `getToday`, `formatRemainingTime` |
| Numbers, money, percentages | `formatNumber`, `formatCurrency`, `formatPercentage`, and the `formatDutch*` trio |
| Temperature | `formatCelsius`, `formatFahrenheit`, plus `CELSIUS` / `FAHRENHEIT` |
| Null and empty handling | `hasValue`, `emptyOrNull`, `optional` |
| Persisting a typed value as a string | `serialize`, `deserialize` |
| Language list and flags | `getLanguage`, `getLanguages` |
| Document theme, locale, zoom, scroll, reload | `setDarkTheme`, `setWebsiteLocale`, `disableZoom`, `scrollToTop`, `reload`, `initApplication` |
| User preferences | `prefersDarkScheme`, `prefersReducedMotion` |
| Type guards | `isHtmlElement`, `isClassInstance` |
| Wrapping a function | `runBefore`, `runAfter` |
| Waiting | `delay` |
| Clipboard | `copyToClipboard` |
| Text | `upperFirst`, plus `MDASH` / `NDASH` / `BULLET` |

## Shared types

Used across every package in the scope. Prefer these over writing the union
inline.

```ts
Nullable<T>              // T | null
Undefinable<T>           // T | undefined
NullableOrUndefinable<T> // T | null | undefined
Arrayable<T>             // T | T[]
Promisable<T>            // T | Promise<T>
PromiseGetter<T>         // () => Promise<T>
PromiseOrGetter<T>       // Promise<T> | PromiseGetter<T>
HTMLTarget               // HTMLElement | Window | Document
```

## Task recipes

### Format a date

```ts
import { DateTime } from 'luxon';
import { formatDate, formatDateTime } from '@almighty-shogun/utils';

formatDate(DateTime.now());              // 11 August 2026
formatDate(DateTime.now(), 'nl');        // 11 augustus 2026
formatDateTime(DateTime.now());
```

Locale is optional everywhere and falls back to the document's when omitted.

### Guard a nullable value

`hasValue` is a type guard that also rejects empty strings and empty arrays, not
just `null` and `undefined`.

```ts
if (hasValue(name)) {
    // name is string here, and not ''
}
```

`emptyOrNull` is the same test returning `T | null`, useful for collapsing `''`
and `[]` into `null` before storing.

### Map a value that might be absent

`optional` runs the callback only when the value is present, and maps arrays
element-wise. Four overloads keep the return type accurate.

```ts
const slug = optional(project, p => p.name.toLowerCase());   // string | null
const slugs = optional(projects, p => p.name.toLowerCase()); // string[] | null
```

### Persist a typed value

`serialize` and `deserialize` preserve the type rather than JSON-stringifying
everything. `deserialize` takes a default whose type decides how the string is
read back.

```ts
const raw = serialize(DateTime.now());     // ISO string
const when = deserialize(raw, DateTime.now()); // a DateTime again
```

Handles strings, numbers, booleans, bigints, `null`, `undefined`, `DateTime`,
`Date`, `URL`, `Set` and `Map`, falling back to JSON for plain objects.

### Set up an application

```ts
initApplication({ locale: 'nl', isDarkTheme: true, isZoomDisabled: true });
```

A convenience wrapper over `setWebsiteLocale`, `setDarkTheme` and `disableZoom`.

## Traps

**`hasValue('')` is `false`.** It rejects empty strings and empty arrays, not
just null and undefined. That is usually what you want for form input and
usually not what you want for a flag, so reach for an explicit `!== null` when
emptiness is meaningful.

**Dates are Luxon `DateTime`, never `Date`.** Passing a native `Date` to a format
helper does not compile. Convert with `DateTime.fromJSDate(date)`.

**`deserialize` needs a default of the right type.** The default is what selects
the handler, so `deserialize(raw, '')` reads a serialized number back as a
string. It is a type hint as much as a fallback.

**A number that fails to parse falls back silently.** `deserialize('abc', 0)`
returns `0` rather than `NaN` or throwing, so a corrupted stored value degrades
quietly.

**Browser helpers need a document.** `setDarkTheme`, `setWebsiteLocale`,
`disableZoom`, `scrollToTop`, `reload`, `copyToClipboard`, `prefersDarkScheme`
and `prefersReducedMotion` all touch browser globals. Guard them during
server-side rendering; the package does not.

**The `formatDutch*` helpers are locale-fixed.** They always format as `nl-NL`
and ignore the document locale, unlike `formatNumber` and `formatCurrency`.

**`getToday` returns a `yyyy-MM-dd` string**, not a `DateTime`. It is for keys
and comparisons, not for further date maths.

**`runBefore` and `runAfter` are curried.** They take the hook first and return
a function that wraps yours: `runAfter(cleanup)(doWork)`.

**`MDASH` and `NDASH` are dash characters** exported deliberately, so text can
reference them without typing a literal dash.

## Exports

Dates: `formatDate`, `formatDateTime`, `formatFullDate`, `formatTime`,
`formatHour`, `formatMonth`, `formatWeekday`, `formatRemainingTime`, `addTime`,
`isToday`, `isBeforeNow`, `getToday`.

Numbers: `formatNumber`, `formatCurrency`, `formatPercentage`,
`formatDutchNumber`, `formatDutchCurrency`, `formatDutchPercentage`,
`formatCelsius`, `formatFahrenheit`.

Values: `hasValue`, `emptyOrNull`, `optional`, `serialize`, `deserialize`,
`upperFirst`.

Locale: `getLanguage`, `getLanguages`.

Browser and DOM: `initApplication`, `setDarkTheme`, `setWebsiteLocale`,
`disableZoom`, `scrollToTop`, `reload`, `copyToClipboard`, `prefersDarkScheme`,
`prefersReducedMotion`, `isHtmlElement`.

Functions and timing: `delay`, `runBefore`, `runAfter`, `isClassInstance`.

Constants: `MDASH`, `NDASH`, `BULLET`, `CELSIUS`, `FAHRENHEIT`.

Types: `Nullable`, `Undefinable`, `NullableOrUndefinable`, `Arrayable`,
`Promisable`, `PromiseGetter`, `PromiseOrGetter`, `HTMLTarget`.
