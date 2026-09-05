# @almighty-shogun/common, full index

Every export with its exact signature **and its return shape**. Read `common.md`
first for what to reach for and what to watch out for; this file is the lookup.

The return shapes matter here more than the signatures. None of them are
exported, so `UseDataTable<T>` cannot be imported or written by hand. Annotate
with `ReturnType<typeof useDataTable<User>>` when you must name one.

Signatures are transcribed from a published build. Verify against the copy
installed in the project, which is the contract that actually applies.

## State

```ts
declare function useOpen(state?: MaybeRefOrGetter<boolean>): UseOpen;

type UseOpen = {
    readonly isOpen: Ref<boolean>;
    open(): void;
    close(): void;
    toggle(): void;
};

declare function useLoaded(): UseLoaded;

type UseLoaded = {
    readonly isLoading: Ref<boolean>;
    load<T>(task: PromiseOrGetter<T>): Promise<T>;
};

declare function useForm<T>(spec: MaybeRefOrGetter<T>): UseForm<T>;

type UseForm<T> = {
    readonly form: Ref<T>;
    reset(): void;
};

declare function useInterval(ms: MaybeRefOrGetter<number>, fn: Function): UseInterval;

type UseInterval = {
    start(): void;
    stop(): void;
};
```

`load` accepts a promise or a function returning one, and clears `isLoading`
whether it resolves or rejects. `useInterval` does not start on its own.

## Paging

```ts
declare function usePagination(pageSize?: MaybeRefOrGetter<number>): UsePagination;

type UsePagination = {
    readonly limits: Ref<number[]>;
    readonly page: Ref<number>;
    readonly perPage: Ref<number>;
    readonly total: Ref<number>;
    setTotal(total: number): void;
    setPage(page: number): void;
    setPerPage(perPage: number): void;
};

declare function useDataTable<T>(
    items: Ref<T[]>,
    pageSize?: Undefinable<MaybeRefOrGetter<number>>
): UseDataTable<T>;

type UseDataTable<T> = {
    readonly isEmpty: Ref<boolean>;
    readonly total: Ref<number>;
    readonly page: Ref<number>;
    readonly perPage: Ref<number>;
    readonly limits: Ref<number[]>;
    readonly filteredItems: Ref<T[]>;
    setTotal(total: number): void;
    setPage(page: number): void;
    setPerPage(perPage: number): void;
};
```

`useDataTable` is `usePagination` plus `filteredItems` and `isEmpty`, and it
slices the items for you. Use `usePagination` when the server pages the data.

## Persisted state

```ts
declare function useLocalStorage<T extends {}>(
    key: string,
    defaultValue: T,
    options?: UseLocalStorageOptions<T>
): Ref<T>;

type UseLocalStorageOptions<T> = {
    prefix?: Undefinable<string>;
    deserializer?(value: string, defaultValue: T): T;
    serializer?(value: T): string;
};

declare function usePersistentRef(key: string, defaultValue: null): Ref<Nullable<string>>;
declare function usePersistentRef<T extends {}>(key: string, defaultValue: T): Ref<Nullable<T>>;
```

`useLocalStorage` always holds a value; `usePersistentRef` is nullable and
removes the key when set to `null`. Its two overloads mean a `null` default gives
`Ref<Nullable<string>>`, since there is nothing to deserialize against. Both
default to `utils`'s `serialize` and `deserialize`, and both degrade to a plain
ref when `window` is undefined.

## Document

```ts
declare function useDarkTheme(key?: string): UseDarkTheme;

type UseDarkTheme = {
    readonly darkMode: Ref<boolean>;
    toggle(): void;
};

declare function useWebsiteLocale(key?: string): UseWebsiteLocale;

type UseWebsiteLocale = {
    readonly locale: Ref<string>;
    setLocale(newLocale: string): void;
};

declare function usePageHeader<TIcon = string>(
    config?: Undefinable<HeaderData<TIcon>>
): UsePageHeader<TIcon>;

type HeaderData<TIcon = string> = {
    title: string;
    icon: TIcon;
    page?: Undefinable<string>;
};

type UsePageHeader<TIcon = string> = {
    readonly pageTitle: Ref<string>;
    readonly pageIcon: Ref<TIcon>;
};
```

`key` defaults to `application-theme` and `application-locale`. Both apply their
stored value to the document immediately on creation.

## DOM and events

```ts
declare function useEventListener<TEvent extends keyof EventMap>(
    source: MaybeRefOrGetter<NullableOrUndefinable<ComponentTarget>>,
    events: Arrayable<TEvent>,
    handler: (evt: EventMap[TEvent]) => void,
    options?: Undefinable<AddEventListenerOptions | boolean>
): UseEventListener;

declare function useClickOutside(
    targets: Arrayable<MaybeRefOrGetter<NullableOrUndefinable<ComponentElement>>>,
    callback: OutsideClickHandler,
    enabled?: MaybeRefOrGetter<boolean>
): UseClickOutside;

declare function useHotKey(
    hotKeys: Arrayable<string>,
    handler: HotKeyHandler,
    options?: UseHotKeyOptions
): () => void;

declare function useScrollPosition(
    target?: MaybeRefOrGetter<NullableOrUndefinable<ComponentTarget>>
): UseScrollPosition;

declare function useClipboard(
    value: MaybeRefOrGetter<string>,
    onSuccess?: Undefinable<Function>
): UseClipboard;
```

`useEventListener`, `useClickOutside`, `useHotKey` and `useClipboard` all return
a **stop function**, typed `() => void`. They also clean up on scope dispose, so
calling it is only needed to stop early.

```ts
type UseScrollPosition = {
    readonly scrollX: Ref<number>;
    readonly scrollY: Ref<number>;
};

type UseHotKeyOptions = {
    target?: MaybeRefOrGetter<NullableOrUndefinable<ComponentTarget>>;
    enabled?: MaybeRefOrGetter<Undefinable<boolean>>;
    event?: Undefinable<'keydown' | 'keyup'>;
    preventDefault?: Undefinable<boolean>;
    stopPropagation?: Undefinable<boolean>;
    ignoreWhileTyping?: Undefinable<boolean>;
    repeat?: Undefinable<boolean>;
};
```

`ComponentTarget` is `HTMLTarget | ComponentPublicInstance`, `ComponentElement`
is `HTMLElement | ComponentPublicInstance`. Neither is exported, so a template
ref to a component works without unwrapping it first.

## Router

```ts
declare function useRouteName(): ComputedRef<Nullable<string>>;
declare function useRouteNames(): ComputedRef<string[]>;
declare function useIsRoute(): IsRoute;
declare function useIsRoute(route: string, strict?: boolean): ComputedRef<boolean>;
type IsRoute = (route: string, strict?: boolean) => boolean;
declare function useRouteMeta(): ComputedRef<DeepReadonly<RouteMeta>>;

declare function useRouteParam<T = string>(
    name: string,
    defaultValue?: Nullable<T>
): Ref<Nullable<T>>;
```

`useIsRoute` compares route **names**, and `strict` defaults to `true`, meaning
exact equality. Pass `false` for prefix matching. Called with no arguments it
returns a plain matcher function instead of a computed, for checking several
routes in a template without one `useIsRoute` call per route. `IsRoute` is not
exported. `useRouteParam` deserializes only when a non-null default is given.

## Middleware

```ts
declare function defineMiddleware<const Name extends string>(
    name: Name,
    handler: MiddlewareHandler
): MiddlewareDefinition<Name>;

declare function registerMiddleware(
    router: Router,
    options?: RegisterMiddlewareOptions
): () => void;

type MiddlewareResult = void | boolean | RouteLocationRaw;

type MiddlewareHandler = (
    to: RouteLocationNormalized,
    from: RouteLocationNormalized
) => Promisable<MiddlewareResult>;

type MiddlewareDefinition<Name extends string = string> = {
    readonly name: Name;
    readonly handler: MiddlewareHandler;
};

type RegisterMiddlewareOptions = {
    global?: Undefinable<readonly MiddlewareDefinition[]>;
    except?: Undefinable<readonly NonNullable<RouteRecordName>[]>;
};
```

These four types **are** exported, unlike the composable return shapes.

The package also augments Vue Router, so both meta fields are typed without the
project declaring anything:

```ts
declare module 'vue-router' {
    interface RouteMeta {
        middleware?: Undefinable<readonly MiddlewareDefinition[]>;
        skipMiddleware?: Undefinable<readonly MiddlewareDefinition[]>;
    }
}
```

`registerMiddleware` returns the function that removes its guard. `except` takes
route names and narrows to a checked union only when the project configures Vue
Router's typed routes.

## Refs and unwrapping

```ts
declare function requiredRef<T>(): Ref<T>;
declare function nullableRef<T = never>(value?: Undefinable<T>): Ref<Nullable<T>>;

declare function componentRef<TComponent extends ComponentImport>(
    component: TComponent
): Ref<Nullable<InstanceType<TComponent>>>;

declare function debouncedRef<T>(
    initialValue: MaybeRefOrGetter<T>,
    delay: number,
    immediate?: boolean
): Ref<T>;

declare function unwrapElement<TElement extends HTMLElement>(
    elementRef: MaybeRefOrGetter<NullableOrUndefinable<ComponentElement>>
): Nullable<TElement>;

declare function unwrapTarget(
    target: MaybeRefOrGetter<NullableOrUndefinable<ComponentTarget>>
): Nullable<HTMLTarget>;
```

`requiredRef` throws on read before assignment. `componentRef` takes the imported
component itself, so the ref is typed as its instance.

## i18n

```ts
declare function registerI18n(i18n: I18n): void;
declare function clearRegisteredI18n(): void;
declare function translate(key: string, params?: Undefinable<TranslationParams>): string;
declare function translationExists(key: string, subKeys?: string[]): boolean;
```

`I18n` accepts either naming convention, so both `vue-i18n`'s `global` and a
component instance fit:

```ts
type I18n = {
    t?: Undefinable<Translate>;
    $t?: Undefinable<Translate>;
    te?: Undefinable<TranslateExists>;
    $te?: Undefinable<TranslateExists>;
};

type TranslationParams = Record<string, unknown> | (string | number)[];
```

`clearRegisteredI18n` unregisters, which is mainly useful between tests.
