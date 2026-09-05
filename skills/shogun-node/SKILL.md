---
name: shogun-node
description: >-
  Use when work materially depends on an @almighty-shogun npm package: using or
  changing one of its APIs, determining package ownership, or resolving its
  installed contract. Covers utils, common, http-core, bun-server,
  cloudflare-worker, webkit-native-bridge, and prototype-extensions. Do not
  trigger merely because an unrelated @almighty-shogun import appears in a file.
  NOT for @almighty-shogun NuGet or Maven packages.
argument-hint: "[package] <what you want>"
license: MIT (skill content); the @almighty-shogun npm packages are MIT, by Shogun
---

# Shogun node packages

Use this skill when the task materially depends on the `@almighty-shogun` npm
ecosystem.

Do not load it merely because a file happens to contain an unrelated
`@almighty-shogun/*` import.

## Package ownership

| Package | Owns | Depends on, in scope |
| --- | --- | --- |
| `utils` | Luxon date helpers, number, currency and locale formatting, serialization, browser and DOM helpers, the shared type vocabulary (`Nullable`, `Undefinable`, `Arrayable`, `Promisable`) | |
| `http-core` | HTTP status and method vocabulary, response factories, query readers, the shared server errors | |
| `bun-server` | Bun server setup and typed routing | `http-core`, re-exported whole, and `utils` |
| `cloudflare-worker` | Worker routing and cron scheduling | `http-core`, re-exported whole, and `utils` |
| `common` | Vue 3 composables, refs, forms, tables, router helpers, i18n | `utils` at runtime; Vue and Vue Router are peers |
| `webkit-native-bridge` | typed request and response bridge to a native WebKit host | `utils`, for types only |
| `prototype-extensions` | methods added to `Array`, `String` and `Number` by prototype mutation | none, and it exports nothing |

`utils` and `http-core` are the two roots, and every other package except
`prototype-extensions` depends on `utils`, so it is in `node_modules` of most
projects in the scope whether or not `package.json` names it. Only
`webkit-native-bridge` uses it for types alone.

Docs: https://node.docs.shogun.ms

Do not guess package ownership when an export can be checked.

## Reference routing

Read only the package reference needed for the task.

| Area | Primary reference | Additional reference only when needed |
| --- | --- | --- |
| `utils` | `references/utils.md` | `references/utils-index.md` for exact exports and signatures |
| `common` | `references/common.md` | `references/common-index.md` for exact exports and return shapes |
| `bun-server` | `references/bun-server.md` | `references/http-core.md` when the task crosses into re-exported HTTP APIs |
| `cloudflare-worker` | `references/cloudflare-worker.md` | `references/http-core.md` when the task crosses into re-exported HTTP APIs |
| `http-core` | `references/http-core.md` | |
| `webkit-native-bridge` | `references/webkit-native-bridge.md` | |
| `prototype-extensions` | `references/prototype-extensions.md` | |

Start with the directly relevant package. Inspect another guide only when the
actual implementation crosses that boundary, and use an index file only when an
exact export, signature or return shape is needed.

## One resolved copy of http-core

`bun-server` and `cloudflare-worker` re-export every `http-core` export from
their own root, so a route file imports `HttpStatus`, `queryInteger` or
`MissingParameterError` from the server package.

Never add `http-core` to `package.json` when either server package is present.
Handler results are checked with `instanceof HttpBaseResponse`, so two resolved
copies make a valid response fail that check and raise
`InvalidHandlerResultError`, at request time, with nothing in the message
pointing at the duplicate.

Install `http-core` directly only when building a server wrapper for a runtime
neither package covers.

## Installed contract wins

The package version installed in the current project is the contract that code
must compile against.

Before relying on a signature or export, inspect the installed package metadata
and declaration entry when practical. Resolve that entry from the package's own
exports under `node_modules/@almighty-shogun/<package>/` rather than assuming a
path such as `dist/index.d.mts`, which is not stable across versions.

Priority: installed declarations and package exports, then the matching bundled
reference guide, then the documentation site.

If a guide and the installed contract disagree, follow the installed contract and
mention the mismatch when it materially affects the answer or the change.

## Host repository conventions win

Read the applicable host-project `AGENTS.md`, `CLAUDE.md`, and any package or
module-local instructions. The consuming project's conventions take priority over
development conventions from the `@almighty-shogun` package repositories.

## Implementation permission

An explicit request for a scoped implementation already authorizes the ordinary
code edits it requires. Do not ask to proceed before normal source changes.

Ask first only when a material requirement is unresolved, several materially
different behaviors are possible, a dependency must be added or changed,
consequential configuration must be changed, or `prototype-extensions` would be
newly introduced.

## Missing dependencies

Inspect `package.json`, workspace manifests where relevant, the lockfile, and
installed `node_modules`.

Presence in `node_modules` does **not** authorize direct use: a package may exist
only because another dependency installed it transitively. When project source
imports an `@almighty-shogun/*` package directly, it normally belongs in the
applicable manifest as a direct dependency.

When one is missing, verify it is actually necessary, identify the correct
manifest, determine the package manager from repository evidence, show the exact
change or command, and get explicit approval before installing anything.

Lockfiles identify the manager of an unfamiliar project; they do not decide what
a new project should use:

| Lockfile | Manager |
| --- | --- |
| `bun.lock`, `bun.lockb` | Bun |
| `pnpm-lock.yaml` | pnpm |
| `yarn.lock` | Yarn |
| `package-lock.json` | npm |

Do not install dependencies unasked.

## Prototype extensions: explicit opt-in

`@almighty-shogun/prototype-extensions` differs from an ordinary helper package
because importing it mutates built-in prototypes by side effect.

Never introduce it merely because a prototype method would be shorter or more
convenient, and never silently replace a normal helper with one.

When the project already uses it intentionally, follow that usage: ordinary
scoped use needs no extra approval. When it does not, explain that the package
extends built-in prototypes, show the dependency and import change, and get
explicit approval first. That approval is required even when the package already
exists transitively in `node_modules`.

## Configuration changes

Do not change runtime, build or tooling configuration merely to make a package
convenient. That covers TypeScript config, Bun settings, Cloudflare config, Vue
and plugin setup, and runtime globals.

When a consequential change is genuinely required, explain why, show the exact
intended change, and get approval before applying it.

## Runtime and platform uncertainty

Do not invent answers for platform semantics that materially affect behavior:
Cloudflare cron time semantics, Bun runtime behavior, Vue lifecycle behavior, or
browser and WebKit bridge constraints.

Use repository-local evidence and installed declarations first. When substantial
external research is required, use `source-research` rather than bloating this
skill.

## Verification

Use the host project's real validation commands, discovered from `package.json`
scripts, workspace scripts, CI configuration or repository instructions. Do not
assume command names.

Run only the checks relevant to the changed scope. Passing checks do not justify
ignoring a clear package-contract mismatch.

## Scope

This is an implementation and reference skill, not a Git workflow: the global
instructions on Git and outward-facing actions apply unchanged, and completed Git
work goes to the `commit` skill.

Keep context small. Load one package guide, use index files only for an exact
lookup, inspect installed declarations directly rather than pasting them into
parent context, and do not enumerate a package's whole API to answer a
one-symbol question.
