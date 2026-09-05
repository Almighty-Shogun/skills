# @almighty-shogun/prototype-extensions

Adds convenience methods to `Array`, `String` and `Number` by augmenting their
prototypes. **Exports nothing.** It is a side-effect import.

Verify the method list against the copy installed in the project, resolving the
declaration entry from the package's own `exports` rather than assuming a path.

Docs: https://node.docs.shogun.ms/prototype-extensions/

No dependencies, no peers.

## Setup

Import it **once**, at the application entry point, for its side effects. There
is nothing to destructure.

```ts
// main.ts
import '@almighty-shogun/prototype-extensions';
```

That single import both patches the prototypes at runtime and brings the
`declare global` block into scope, so every file in the project sees the methods
without importing anything.

A library should not import it. It mutates globals for the whole process, which
is a decision that belongs to the application.

## Methods

### Array

| Method | Returns |
|---|---|
| `first()` | first element |
| `last()` | last element |
| `delete(index)` | a **new** array without that index |
| `removeDuplicates()` | a new array, duplicates removed, order kept |
| `addOrRemove(value)` | a new array with the value toggled in or out |
| `isEmpty()` | `length === 0` |

`removeDuplicates` and `addOrRemove` are constrained to `Array<string | number>`,
because both compare by identity.

### String

| Method | Returns |
|---|---|
| `toSlug()` | lowercase, punctuation stripped, spaces and underscores to hyphens, trimmed |
| `append(value)` | the string with `value` added |
| `limit(length, appending?)` | truncated to `length`, with `appending` added when it was truncated |
| `toNumber()` | `Number(this)` |
| `isEmpty()` | `length === 0` |
| `equals(value)` | strict equality |

### Number

| Method | Returns |
|---|---|
| `add`, `subtract`, `multiply`, `divide` | the arithmetic result |
| `equals(value)` | strict equality |
| `isInRange(min, max)` | inclusive range test |
| `isEven()` | `this % 2 === 0` |
| `isLowerThan(value, allowEqual?)` | comparison, exclusive unless `allowEqual` |
| `isGreaterThan(value, allowEqual?)` | comparison, exclusive unless `allowEqual` |

```ts
['a', 'b', 'a'].removeDuplicates();      // ['a', 'b']
['a', 'b'].addOrRemove('b');             // ['a']
'Hello World!'.toSlug();                 // 'hello-world'
'A long sentence'.limit(6, '...');       // 'A long...'
(4).isInRange(1, 5);                     // true
```

## Traps

**`first()` and `last()` are typed as `T`, not `T | undefined`.** On an empty
array they return `undefined` at runtime while the type says otherwise. Guard
with `isEmpty()` first.

**`delete(index)` does not mutate.** It returns a new array via `toSpliced`, so
`items.delete(0)` on its own does nothing. Assign the result. The name reads like
`Array.prototype.splice`, which does mutate, and that is the easy mistake.

**`removeDuplicates` and `addOrRemove` compare by identity.** They are typed for
`string | number` only, so they will not deduplicate objects, and the type stops
you trying.

**`limit` appends only when it truncates.** A string already within the limit
comes back untouched, with no suffix. The default `appending` is `null`, which
concatenates as the text `"null"` if you pass `null` explicitly rather than
omitting it.

**`toNumber()` returns `NaN` for unparseable text**, since it is `Number(this)`.
It does not throw and does not return null.

**One import, at the entry point.** Importing it in several modules is harmless
but pointless; importing it in none means the methods exist in the types and not
at runtime, which fails only when the code runs.

**`isLowerThan` and `isGreaterThan` are exclusive by default.** Pass `true` as
the second argument for `<=` or `>=`.

## Exports

None. The package has no runtime exports and nothing to name in an import
statement; it works entirely through global augmentation.
