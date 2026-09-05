# @almighty-shogun/utils, full index

Every export with its exact signature. Read `utils.md` first for what to reach
for and what to watch out for; this file is the lookup.

Signatures are transcribed from a published build. Verify against the copy
installed in the project, which is the contract that actually applies.

## Dates

Every date helper takes and returns a Luxon `DateTime`, never a native `Date`.
`locale` is optional everywhere and falls back to the document's.

```ts
declare function formatDate(date: DateTime, locale?: Undefinable<string>): string;
declare function formatDateTime(date: DateTime, locale?: Undefinable<string>): string;
declare function formatFullDate(date: DateTime, locale?: Undefinable<string>): string;
declare function formatTime(date: DateTime, locale?: Undefinable<string>): string;
declare function formatHour(date: DateTime, locale?: Undefinable<string>): string;

declare function formatMonth(
    date: DateTime,
    isLong?: boolean,
    locale?: Undefinable<string>
): string;

declare function formatWeekday(
    date: DateTime,
    isLong?: boolean,
    locale?: Undefinable<string>
): string;

declare function formatRemainingTime(time: DurationLikeObject): string;
declare function addTime(time: DurationLike): DateTime;
declare function isToday(dateTime: DateTime, today?: Undefinable<DateTime>): boolean;
declare function isBeforeNow(date: DateTime): boolean;
declare function getToday(): string;
```

| Export | Note |
|---|---|
| `formatDate` | day, long month, year |
| `formatDateTime` | date plus time |
| `formatFullDate` | includes the weekday |
| `formatTime` | hours and minutes |
| `formatHour` | hour only |
| `formatMonth` | `isLong` defaults to `true`, so `January` over `Jan` |
| `formatWeekday` | `isLong` defaults to `true` |
| `formatRemainingTime` | a Luxon duration object as human text |
| `addTime` | adds a duration to **now**, it takes no base date |
| `isToday` | `today` is injectable, which is what makes it testable |
| `getToday` | `yyyy-MM-dd` **string**, not a `DateTime` |

## Numbers, money, temperature

```ts
declare function formatNumber(
    value: number,
    decimals?: number,
    locale?: Undefinable<string>
): string;

declare function formatCurrency(
    value: number,
    currency?: Undefinable<string>,
    locale?: Undefinable<string>
): string;

declare function formatPercentage(value: number, locale?: Undefinable<string>): string;

declare function formatDutchNumber(value: number, decimals?: number): string;
declare function formatDutchCurrency(value: number): string;
declare function formatDutchPercentage(value: number): string;

declare function formatCelsius(temperature: number): string;
declare function formatFahrenheit(temperature: number): string;
```

The `formatDutch*` trio is locale-fixed to `nl-NL` and ignores the document
locale. The others follow it.

## Values and null handling

```ts
declare function hasValue<T>(value: NullableOrUndefinable<T>): value is T;
declare function emptyOrNull<T>(value: NullableOrUndefinable<T>): Nullable<T>;

declare function optional<T, U extends {}>(value: U[], callback: (value: U) => T): T[];
declare function optional<T, U extends {}>(value: U, callback: (value: U) => T): T;
declare function optional<T, U>(
    value: NullableOrUndefinable<U[]>,
    callback: (value: U) => T
): Nullable<T[]>;
declare function optional<T, U>(
    value: NullableOrUndefinable<U>,
    callback: (value: U) => T
): Nullable<T>;

declare function upperFirst(value: string): string;
```

`hasValue` and `emptyOrNull` both treat `''` and `[]` as absent, not only `null`
and `undefined`. `optional` has four overloads so an array input keeps an array
output and a non-null input keeps a non-null output.

## Serialization

```ts
declare function serialize<T>(value: T): string;
declare function deserialize<T>(value: string, defaultValue: T): T;
```

Type-preserving rather than JSON. Handles `string`, `number`, `boolean`,
`bigint`, `null`, `undefined`, `DateTime`, `Date`, `URL`, `Set` and `Map`, and
falls back to JSON for plain objects. The `defaultValue` selects the handler, so
it is a type hint as much as a fallback, and an unparseable value returns it
rather than throwing.

## Locale data

```ts
declare function getLanguage(code: LanguageCode): Nullable<Language>;
declare function getLanguages(): Language[];
```

`Language` and `LanguageCode` are **not exported**. `Language` is:

```ts
type Language = {
    name: string;
    code: string;
    flag: {
        plain: string;
        rounded: string;
    };
};
```

## Browser and DOM

All of these touch browser globals and are unguarded, so wrap them for
server-side rendering.

```ts
declare function initApplication(config: ApplicationConfig): void;
declare function setDarkTheme(isDark: boolean): void;
declare function setWebsiteLocale(locale?: Undefinable<string>): void;
declare function disableZoom(): void;
declare function reload(): void;

declare function scrollToTop(
    element?: Undefinable<HTMLElement>,
    options?: Undefinable<ScrollToOptions>
): void;

declare function copyToClipboard(value: string): Promise<boolean>;
declare function prefersDarkScheme(): boolean;
declare function prefersReducedMotion(): boolean;
declare function isHtmlElement(element: unknown): element is HTMLElement;
```

`ApplicationConfig` is **not exported**:

```ts
type ApplicationConfig = {
    locale?: Undefinable<string>;
    isDarkTheme?: Undefinable<boolean>;
    isZoomDisabled?: Undefinable<boolean>;
};
```

`copyToClipboard` resolves `false` on failure rather than rejecting.
`scrollToTop` scrolls the window when no element is given.

## Functions and timing

```ts
declare function delay(ms: number): Promise<void>;
declare function runBefore<T extends (...args: any[]) => any>(before: () => void): (fn: T) => T;
declare function runAfter<T extends (...args: any[]) => any>(after: () => void): (fn: T) => T;
declare function isClassInstance(value: unknown): value is object;
```

`runBefore` and `runAfter` are curried: the hook first, then the function to
wrap, as in `runAfter(cleanup)(doWork)`.

## Constants

```ts
declare const MDASH: '—';
declare const NDASH: '–';
declare const BULLET: '•';
declare const CELSIUS: '°C';
declare const FAHRENHEIT: '°F';
```

## Types

```ts
type Nullable<T> = T | null;
type Undefinable<T> = T | undefined;
type NullableOrUndefinable<T> = Nullable<T> | Undefinable<T>;
type Arrayable<T> = T | T[];
type Promisable<T> = T | Promise<T>;
type PromiseGetter<T> = () => Promise<T>;
type PromiseOrGetter<T> = Promise<T> | PromiseGetter<T>;
type HTMLTarget = HTMLElement | Window | Document;
```

These are the shared vocabulary every other package in the scope reuses.
