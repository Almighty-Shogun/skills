---
name: release-notes
description: >-
  Generate concise, evidence-based GitHub release notes from a repository diff
  in the standard emoji house format, with one section per changed package in
  monorepos. --file writes them under .release-notes/; --redo <tag>
  regenerates an existing release, editing it only after approval.
argument-hint: "[project] [base-ref] [--file] [--redo <tag>]"
---

# Release Notes

Compare a resolved base and head, read the actual changes, and describe only
release-relevant behavior.

## Contract

Read-only for Git: never modify source, stage, commit, tag, push, create a
release, or publish. The only writes are `.release-notes/` files with `--file`,
and an approved edit to an existing release's notes with `--redo`.

## Invocation

```text
release-notes [project] [<base-ref>] [--file]
release-notes [project] --redo <tag> [--file]
```

A project selector picks one repository when the working directory holds
several; discover them, never hardcode names, and ask when the argument does
not identify one. One repository per run.

- **Direct**: generate the notes and return them in chat, also writing the file
  with `--file`.
- **`--redo <tag>`**: verify the release exists, generate replacement notes,
  show the current and proposed notes, and update GitHub only after approval.
  With `--file`, write the replacement into the repository instead and leave
  GitHub alone. `--redo` never creates a missing release.

Read `AGENTS.md`, `CLAUDE.md`, and release-specific instructions. They may set
release-unit detection, base rules, whether PR or commit numbers are required,
terminology, and required extra information. The emoji format stays the
default unless instructions require something incompatible. Recent releases
are a guide to terminology, package naming, and level of detail, not to a
non-emoji structure.

## Range

Derive `<dir>`, `<slug>`, and `<default-branch>`; use `git -C <dir>` and
`--repo <slug>` throughout. Run `git -C <dir> fetch origin --tags --prune`.

- Direct: head is `origin/<default-branch>`; base is the given ref, else the
  most recent published release appropriate to the context, else the first
  commit.
- `--redo <tag>`: head is the tag (which must resolve locally); base is the
  nearest earlier published release, else the first commit.
- From the `release` skill: accept its resolved base, head, target tag, and
  prerelease state as given.

Verify both refs resolve. If they are the same commit, report no changes and
stop.

## Evidence

```bash
git -C <dir> log --oneline --no-merges <base>..<head>
git -C <dir> diff --name-status <base>..<head>
git -C <dir> diff --stat <base>..<head>
```

Commit subjects are discovery signals, not classification. Read the diffs to
establish user-visible behavior, public API and export changes, configuration,
dependencies, build and publishing, documentation, migrations, accessibility,
styling, performance, and compatibility.

Call a change breaking only with evidence of an actual compatibility break: a
removed or renamed public API, parameter, property, prop, event, response
field, or endpoint; an incompatible serialized or configuration format; or
behavior existing consumers cannot keep using. A deleted file is not breaking
by itself; confirm it was part of the consumer contract. State a verified
migration path briefly.

Account for the whole diff without a bullet per commit. Write for users,
consumers, integrators, maintainers, and operators when deployment or runtime
behavior changes. Usually omit test refactors, formatting churn, mechanical
cleanup, CI housekeeping, and internal renames; a notable maintainer-facing
change can still go under Chores. Several commits implementing one feature
become one item.

## House format

Sections, in this order, rendered only when they have content:

| Section | For |
|---|---|
| `## ⚠️ Breaking changes` | a proven compatibility break, even from a `feat` or `fix` commit |
| `## ✨ New <domain items>` | a newly added public package, component, module, or API; name the noun for the domain (components, packages, commands, integrations) |
| `## 🚀 Features` | new consumer-visible behavior |
| `## ♿ Accessibility` | accessibility behavior, even when built as a feature |
| `## 🐛 Fixes` | corrected consumer-visible behavior |
| `## ⚡ Performance` | a meaningful performance improvement |
| `## 🎨 Styles` | styling, theme, token, or visual-system changes |
| `## 📚 Documentation` | release-relevant documentation |
| `## 🧹 Chores` | build, dependency, CI, publishing, or upkeep worth mentioning, unless it changes behavior or breaks consumers |

No generic GitHub headings such as `What's Changed`, `Added`, or `Fixed`. No PR
or commit numbers unless instructions require them.

**Monorepos**: every package with release-relevant changes gets its own
top-level `# \`<Package>\`` heading with its sections beneath; never collapse
packages into one heading for brevity. A change spanning the whole workspace
goes under `# Workspace` or the repository's equivalent, not duplicated under
each package unless each has a distinct consumer-facing effect. A
single-package repository omits the package heading.

A quote-style summary line is optional, used only when the release has a real
theme: `> Adds package-level cache configuration and fixes stale cache
invalidation.`

End with `**Full Changelog:** https://github.com/<slug>/compare/<base>...<compare-head>`
when the refs make the comparison meaningful, where `<compare-head>` is the
future tag for a release workflow, the existing tag for `--redo`, or the
resolved head otherwise.

```markdown
# `@example/core`

## ⚠️ Breaking changes

- `CacheStore` no longer accepts string expiration values. Pass a duration value instead.

## 🐛 Fixes

- Cache invalidation now removes dependent keys after a namespace is cleared.

# `@example/vue`

## ✨ New composables

- `useCachedQuery` exposes cache-aware query state for Vue consumers.

# Workspace

## 🧹 Chores

- Publishing now validates workspace package versions before creating artifacts.

**Full Changelog:** https://github.com/example/repo/compare/v1.2.0...v1.3.0
```

Write in English unless asked otherwise, describing concrete observable change:
`` `usePersistentRef` now preserves explicit null values during deserialization.``
No marketing voice ("This exciting release brings..."), vague catch-alls
("Various improvements and fixes"), or Conventional Commit prefixes.

## `--file`

Write to `<dir>/.release-notes/<descriptive-head-or-version>.md`, such as
`.release-notes/v1.4.0.md` or `.release-notes/main-since-v1.3.0.md`, with `/`,
whitespace, and unsafe characters sanitized. Keep it inside the repository even
when Git does not ignore it, warning that it shows in the working tree. Never
commit or remove it. Before overwriting a file with the same name, check that it
is the same requested output.

## `--redo` on GitHub

Fetch the current notes, generate the full replacement, show both, and ask.
Approval to generate is not approval to edit. Only then run
`gh release edit <tag> --repo <slug> --notes-file <temporary-file>`, with the
temporary file removed after a successful update. The edit changes only that
release's notes.

## Final response

Direct: the complete notes, the resolved range, the saved path with `--file`,
and any material classification uncertainty. After `--redo`: which release was
updated, with its URL.
