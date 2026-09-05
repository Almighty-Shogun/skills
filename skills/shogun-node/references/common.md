# @almighty-shogun/common

Vue 3 application primitives: composables, refs, router helpers and module-level
i18n access.

Verify signatures against the copy installed in the project, resolving the
declaration entry from the package's own `exports` rather than assuming a path.

Docs: https://node.docs.shogun.ms/common/

## Setup

```sh
bun add @almighty-shogun/common vue vue-router
```

`vue` and `vue-router` are **peer** dependencies, so the project supplies them.
`@almighty-shogun/utils` arrives as a direct dependency and is used at runtime,
not only for types, so `serialize`, `deserialize` and `setDarkTheme` are already
on disk.

Every export is a named export from the package root. Everything is Vue-scoped,
including the i18n helpers, which read the current component instance.

## What is where

| Need | Reach for |
|---|---|
| Open/closed, loading, interval, clipboard | `useOpen`, `useLoaded`, `useInterval`, `useClipboard` |
| Paging a list | `usePagination`, or `useDataTable` when you also hold the items |
| A form with a reset | `useForm` |
| Persisted state | `useLocalStorage`, `usePersistentRef` |
| Theme and locale on the document | `useDarkTheme`, `useWebsiteLocale` |
| DOM events, outside clicks, hotkeys, scroll | `useEventListener`, `useClickOutside`, `useHotKey`, `useScrollPosition` |
| Page title and icon | `usePageHeader` |
| Reading the current route | `useRouteName`, `useRouteNames`, `useIsRoute`, `useRouteParam`, `useRouteMeta` |
| Guarding a route before entering it | `defineMiddleware`, plus `registerMiddleware` once at startup |
| Typed refs | `requiredRef`, `nullableRef`, `componentRef`, `debouncedRef` |
| Unwrapping a template ref | `unwrapElement`, `unwrapTarget` |
| Translating outside a component | `registerI18n`, `translate`, `translationExists`, `clearRegisteredI18n` to unregister, mainly between tests |

## Task recipes

### Open and close state

```ts
const { isOpen, open, close, toggle } = useOpen();
const { isOpen } = useOpen(true);            // start open
```

### Await something with a loading flag

`load` accepts a promise or a function returning one, and clears `isLoading`
whether it resolves or rejects.

```ts
const { isLoading, load } = useLoaded();

const users = await load(() => fetchUsers());
```

### Page a list you already hold

```ts
const items = ref<User[]>([]);
const { filteredItems, page, perPage, total, setPage } = useDataTable(items, 25);
```

`useDataTable` slices `items` for you into `filteredItems`. `usePagination` is
the same shape without the slicing, for when the server pages the data.

### Persist state

```ts
const settings = useLocalStorage('settings', { theme: 'dark' });
const token = usePersistentRef('token', null);
```

`useLocalStorage` always holds a value and takes a non-null default;
`usePersistentRef` is nullable, and setting it to `null` removes the key.

Both return a plain `Ref` you assign to normally. Both no-op safely during
server-side rendering, returning a plain ref when `window` is undefined.

### Theme and locale

```ts
const { darkMode, toggle } = useDarkTheme();
const { locale, setLocale } = useWebsiteLocale();
```

Both persist to local storage and apply to the document **immediately on
creation**, so a returning visitor's choice is applied on first render without
any startup call.

### React to the route

```ts
const name = useRouteName();                    // leaf route name, or null
const names = useRouteNames();                  // every matched record's name
const isSettings = useIsRoute('settings');      // exact name match
const isSection = useIsRoute('settings', false); // prefix match
const isRoute = useIsRoute();                    // matcher: isRoute('settings')
const id = useRouteParam<number>('id', 0);      // typed, deserialized
const meta = useRouteMeta();                    // merged meta, deeply readonly
```

### Guard a route

`defineMiddleware(name, handler)` builds it, `meta.middleware` attaches it, and
`registerMiddleware(router)` runs it. Without that one registration call the
middleware is inert.

```ts
// middleware/auth.ts
export default defineMiddleware('auth', () => {
    if (!isAuthenticated()) {
        return { name: 'login' };
    }
});

// router/index.ts
registerMiddleware(router);

// routes
{ path: '/admin', meta: { middleware: [auth] } }
```

Return nothing to continue, `false` to cancel, or a route location to redirect,
exactly as a Vue Router guard does.

Middleware is collected from **every matched record, outermost first**, so a
parent runs before its children. Apply one everywhere with `global`, and exempt
routes by name:

```ts
registerMiddleware(router, { global: [auth], except: ['login', 'register'] });
```

A route drops something it would otherwise inherit with `meta.skipMiddleware`,
which works against both a parent's middleware and the global list:

```ts
{ path: '/public', meta: { middleware: [guest], skipMiddleware: [auth] } }
```

Parameterized middleware is a factory returning a definition, with the name built
from the argument so each variant is distinct:

```ts
export default (permission: string) => defineMiddleware(
    `permission:${permission}`,
    () => { /* ... */ }
);
```

### Translate outside a component

Register once during startup, then `translate` works from anywhere, including
stores and plain modules.

```ts
import { createI18n } from 'vue-i18n';
import { registerI18n, translate } from '@almighty-shogun/common';

const i18n = createI18n({ /* ... */ });

registerI18n(i18n.global);

translate('user.greeting', { name: 'Ada' });
```

### Typed refs

```ts
const user = requiredRef<User>();          // throws until assigned
const selected = nullableRef<User>();      // starts null
const input = componentRef(MyInput);       // typed template ref
const search = debouncedRef('', 300);      // writes settle after 300ms
```

## Traps

**`requiredRef` throws on read before assignment.** That is the point: it models
"this exists by the time anything reads it" without `!` everywhere. Reading it in
a synchronous `setup` before the assignment throws
`Required ref accessed before initialization`.

**`useRouteMeta` is deeply readonly.** Writing to a nested key is a compile
error, and the value is a copy, so it shares nothing with your route records. It
merges every matched record key by key at any depth, unlike Vue Router's own
`route.meta`, which replaces a nested object as soon as a child declares the same
key.

**`useIsRoute` compares route *names*, not paths or segments.** It is true when a
matched record is literally named that. `useIsRoute('settings')` is not true on a
route named `settings.profile` unless a parent record is named `settings`. Pass
`false` as the second argument for prefix matching, which does make
`settings.profile` match.

**`useRouteParam` deserializes only when a default is given.** With a non-null
default it runs `deserialize` against the raw string, so a numeric default gives
a number back. With no default you get the raw string typed as `T`, which is a
lie if `T` is not `string`.

**`usePersistentRef` typing depends on the default.** A `null` default gives
`Ref<Nullable<string>>`, since there is nothing to deserialize against. A typed
default gives `Ref<Nullable<T>>`.

**Storage failures warn, they do not throw.** A blocked or full `localStorage`
logs a warning and falls back to the default, so state silently stops persisting
rather than breaking the app.

**Nothing is written during server-side rendering.** `useLocalStorage`,
`usePersistentRef`, `useDarkTheme` and `useWebsiteLocale` all check for `window`
or `document` first and degrade to plain refs.

**Middleware does nothing without `registerMiddleware`.** The definitions are
valid and typed, `meta.middleware` type-checks, and no guard exists to run them.
Nothing warns.

**Middleware is identified by its `name`, not by object identity.** Two
definitions sharing a name are one middleware: the second is dropped from the
chain, and skipping either skips both. This is what makes a factory work, since
each call builds a new object, and it is why a factory's name must be built from
its argument. Two unrelated middleware sharing a name silently collapse.

**Middleware runs on every navigation that matches its record**, not only on
first entry, so a parent's check re-runs when moving between its children.

**`except` is only type-checked with Vue Router's typed routes configured.**
Without that it accepts any string, so a renamed route leaves a name that matches
nothing and silently stops exempting. Prefer `skipMiddleware`, which holds
definitions, when a mistake would leave a route unguarded.

**`translate` needs `registerI18n` first.** Call it once during startup. Without
it, translations fall back rather than resolving, and the failure is quiet.

**The i18n helpers still assume Vue.** They read the current instance, so
"module-level" means outside a component *file*, not outside a Vue app.

## Exports

Composables: `useOpen`, `useLoaded`, `useInterval`, `useClipboard`, `useForm`,
`usePagination`, `useDataTable`, `useLocalStorage`, `usePersistentRef`,
`useDarkTheme`, `useWebsiteLocale`, `usePageHeader`, `useEventListener`,
`useClickOutside`, `useHotKey`, `useScrollPosition`.

Router: `useRouteName`, `useRouteNames`, `useIsRoute`, `useRouteParam`,
`useRouteMeta`.

Middleware: `defineMiddleware`, `registerMiddleware`.

Refs and utilities: `requiredRef`, `nullableRef`, `componentRef`, `debouncedRef`,
`unwrapElement`, `unwrapTarget`.

i18n: `registerI18n`, `clearRegisteredI18n`, `translate`, `translationExists`.

Types: `MiddlewareDefinition`, `MiddlewareHandler`, `MiddlewareResult`,
`RegisterMiddlewareOptions`. The package also augments Vue Router's `RouteMeta`
with `middleware` and `skipMiddleware`, so neither needs declaring in the
project.

Composable return shapes (`UseOpen`, `UseDataTable<T>`, `UsePagination`, and so
on) are not exported; annotate with `ReturnType<typeof useOpen>` if you need to
name one.
