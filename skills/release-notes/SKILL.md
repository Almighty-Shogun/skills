---
name: release-notes
description: >-
  Generate concise, evidence-based GitHub release notes from a repository diff.
  Uses the standard emoji house format by default, supports monorepos with one
  top-level section per changed package, can persist notes inside the repository
  with --file, and can regenerate an existing GitHub release with --redo <tag>.
  Read-only for Git; an existing release is edited only after explicit approval.
argument-hint: "[project] [base-ref] [--file] [--redo <tag>]"
---

# Release Notes

Generate release-ready notes by comparing a resolved base and head, inspecting the
actual changes, and describing only release-relevant behavior.

This skill does not create releases, tags, commits, or publish operations.

Its only remote mutation is an explicitly approved edit to an existing GitHub
release through `--redo <tag>`.

## Invocation

```text
release-notes
release-notes --file
release-notes <base-ref>
release-notes <base-ref> --file
release-notes --redo <tag>
release-notes --redo <tag> --file
```

Workspace/project selection may precede these arguments when the current working
directory contains several repositories.

## Modes

### Direct

Without `--redo`:
- resolve the repository;
- resolve base/head;
- generate notes;
- return the notes in chat;
- when `--file` is supplied, also write them inside the repository.

### Redo existing release

With:

```text
--redo <tag>
```

the skill regenerates notes for an existing GitHub release.

Without `--file`:
1. verify the release exists;
2. generate replacement notes;
3. show the current notes and exact proposed replacement;
4. ask explicit approval;
5. update the GitHub release only after approval.

With `--file`:
- generate the replacement;
- write it inside the repository;
- report the path;
- do not edit GitHub.

`--redo` does not create a missing release.

## Repository instructions

Read applicable:
- `AGENTS.md`;
- `CLAUDE.md`;
- release-specific instructions in the repository.

Repository instructions may alter:
- release-unit/package detection;
- comparison/base rules;
- whether commit/PR numbers are required;
- domain terminology;
- additional required release information.

The emoji house format remains the default release-note style unless repository
instructions explicitly require something incompatible.

Recent release history may be inspected for:
- terminology;
- package/component naming;
- expected amount of detail;
- migration-note conventions.

Do not copy a previous release's non-emoji structure merely because it exists.

## Resolve repository

Determine:
- `<dir>` — repository root;
- `<slug>` — GitHub `owner/name`;
- `<default-branch>` — GitHub default branch.

Use every Git command with:

```text
git -C <dir> ...
```

and GitHub commands with:

```text
--repo <slug>
```

### Workspace mode

When the current directory is not itself the target repository or contains
several sibling repositories:
- discover repositories dynamically;
- never hardcode project names;
- if a project argument clearly identifies one repository, use it;
- otherwise show the candidates and ask which single repository to target.

One repository per run.

## Resolve comparison range

Refresh refs:

```bash
git -C <dir> fetch origin --tags --prune
```

### Normal generation

Head defaults to:

```text
origin/<default-branch>
```

Base is:
- the user-provided ref, when supplied;
- otherwise the most recent published release appropriate to the requested
  release context;
- if no release exists, the repository's first commit.

Verify both refs resolve.

### `--redo <tag>`

Verify:
- the GitHub release exists;
- the tag resolves locally.

Use:
- head = `<tag>`;
- base = nearest earlier published release;
- if none exists, the repository's first commit.

For an existing release, compare-link head is the existing tag.

### Release workflow handoff

When another release workflow asks this skill to generate notes, accept an
explicit resolved:
- base;
- head;
- target version/tag;
- prerelease/stable context.

Do not depend on numbered steps from another skill.

If base and head resolve to the same commit, report that there are no changes
and stop.

## Gather evidence

Start with:

```bash
git -C <dir> log --oneline --no-merges <base>..<head>
git -C <dir> diff --name-status <base>..<head>
git -C <dir> diff --stat <base>..<head>
```

Commit subjects are discovery signals, not authoritative classification.

Read actual diffs whenever needed to determine:
- user-visible behavior;
- public API changes;
- package exports;
- configuration changes;
- dependency effects;
- publishing/build changes;
- documentation changes;
- migrations;
- accessibility;
- styling;
- performance;
- compatibility.

Do not classify a change solely because the commit prefix says `feat`, `fix`,
`chore`, or anything else.

## Breaking changes

Mark a change as breaking only when evidence supports an actual compatibility
break.

Examples:
- removed/renamed public API;
- removed/renamed public parameter, property, prop, event, response field, or
  endpoint;
- incompatible serialized/configuration format change;
- changed behavior that existing consumers cannot continue using as before.

A deleted file is not automatically breaking.

Before calling something breaking:
- establish that it is part of the public/consumer contract;
- inspect relevant exports/consumers/docs/types where needed.

Do not inflate severity from speculation.

When a migration path is known and verified, state it briefly.

## Release relevance

Account for the complete diff, but do not create a bullet for every commit/file.

Include information that matters to:
- users;
- consumers;
- integrators;
- package maintainers;
- operators when the release changes deployment/build/runtime behavior.

Usually omit:
- internal test refactors;
- formatting-only churn;
- mechanical cleanup;
- CI housekeeping with no release effect;
- internal renames invisible to consumers.

A notable maintainer-facing change may still go under `🧹 Chores`.

## Classification

Use these house sections where they have content:

| Change | Section |
|---|---|
| Proven compatibility break | `⚠️ Breaking changes` |
| Newly added public package/component/module/API | `✨ New <domain items>` |
| New consumer-visible behavior | `🚀 Features` |
| Accessibility behavior | `♿ Accessibility` |
| Corrected behavior | `🐛 Fixes` |
| Meaningful performance improvement | `⚡ Performance` |
| Styling/theme/token-only or visual-system behavior | `🎨 Styles` |
| Release-relevant documentation | `📚 Documentation` |
| Maintenance/build/dependency/publishing/internal upkeep worth mentioning | `🧹 Chores` |

### Important classification rules

- Accessibility changes belong under Accessibility even if implemented as a
  feature.
- User-visible corrected behavior belongs under Fixes.
- New user-visible behavior belongs under Features.
- A compatibility break belongs under Breaking changes even if introduced by a
  `feat` or `fix` commit.
- Dependency/build/publishing/CI changes belong under Chores unless they create
  user-visible behavior, fix consumer-visible breakage, or introduce a breaking
  change.
- Documentation-only corrections/features belong under Documentation when they
  are worth mentioning.
- Pure internal churn can be omitted entirely.

Deduplicate by resulting behavior, not commit text.

Several commits/PRs that collectively implement one feature should normally
produce one release-note item.

## Monorepos and multiple packages

Monorepo package separation is intentional.

Detect release units from repository/package structure and repository
instructions.

For every package with release-relevant changes, render its own top-level:

```markdown
# `<Package>`
```

Do not collapse several changed packages into one generic heading merely to make
the notes shorter.

Within each package, use the applicable emoji sections.

If a change genuinely spans the repository/workspace rather than belonging to
one package, place it under an explicit top-level area such as:

```markdown
# Workspace
```

or the repository's established equivalent.

Do not duplicate the same cross-package behavior under every package unless each
package has a distinct consumer-facing effect that deserves its own entry.

For a single-package repository, omit the top-level package heading and begin
with the emoji sections.

## House format

The default section order is:

```text
⚠️ Breaking changes
✨ New <domain items>
🚀 Features
♿ Accessibility
🐛 Fixes
⚡ Performance
🎨 Styles
📚 Documentation
🧹 Chores
```

Only render sections that contain content.

Every section heading must include its house emoji.

Examples:

```markdown
## ⚠️ Breaking changes
## ✨ New components
## 🚀 Features
## ♿ Accessibility
## 🐛 Fixes
## ⚡ Performance
## 🎨 Styles
## 📚 Documentation
## 🧹 Chores
```

The noun after `✨ New` should match the domain when obvious:
- components;
- packages;
- APIs;
- modules;
- commands;
- integrations.

Do not use generic GitHub boilerplate such as:
- `What's Changed`;
- `What's new`;
- bare `Added`;
- bare `Changed`;
- bare `Removed`;
- bare `Fixed`.

Do not append PR/commit numbers by default.

Include them only when repository instructions explicitly require them.

## Writing style

Write in English unless the user explicitly asks otherwise.

Prefer concrete descriptions of observable change.

Good:

```text
- `usePersistentRef` now preserves explicit null values during deserialization.
```

Avoid:

```text
- Various improvements and fixes.
- This exciting release brings...
- We've been hard at work...
- We're thrilled to announce...
```

Do not copy Conventional Commit prefixes into the notes.

Do not create release-note bullets merely to exhaustively mirror the commit log.

Preserve technical accuracy over style.

Punctuation is not governed by arbitrary bans; use normal readable prose.

## Summary line

A quote-style summary line is optional.

Use it only when the release has a real coherent theme worth summarizing.

Example:

```markdown
> Adds package-level cache configuration and fixes stale cache invalidation.
```

When breaking changes exist, the summary may mention that, but do not force a
summary solely to say "contains breaking changes".

## Full changelog

When GitHub refs make the comparison meaningful, end with:

```markdown
**Full Changelog:** https://github.com/<slug>/compare/<base>...<compare-head>
```

Use:
- future target tag when invoked by a release workflow;
- existing `<tag>` for `--redo`;
- resolved head ref/tag for direct historical comparison.

## Rendering example

### Multi-package

```markdown
# `@example/core`

## ⚠️ Breaking changes

- `CacheStore` no longer accepts string expiration values. Pass a duration value instead.

## 🚀 Features

- Cache entries can now opt into stale-while-revalidate behavior.

## 🐛 Fixes

- Cache invalidation now removes dependent keys after a namespace is cleared.

# `@example/vue`

## ✨ New composables

- `useCachedQuery` exposes cache-aware query state for Vue consumers.

## 📚 Documentation

- Added examples for cache invalidation and stale query refreshes.

# Workspace

## 🧹 Chores

- Publishing now validates workspace package versions before creating artifacts.

**Full Changelog:** https://github.com/example/repo/compare/v1.2.0...v1.3.0
```

### Single package

```markdown
## 🚀 Features

- Added configurable request retry policies.

## 🐛 Fixes

- Failed retries now preserve the original request cancellation reason.

## 📚 Documentation

- Documented retry limits and timeout interaction.

**Full Changelog:** https://github.com/example/repo/compare/v1.2.0...v1.3.0
```

## `--file`

When `--file` is supplied, always write the generated markdown **inside the
target repository**.

Use:

```text
<dir>/.release-notes/<descriptive-head-or-version>.md
```

Examples:

```text
.release-notes/v1.4.0.md
.release-notes/main-since-v1.3.0.md
.release-notes/v2.0.0-rc.1.md
```

Sanitize `/`, whitespace, and unsafe path characters.

Do not fall back to a harness scratchpad merely because the path is not ignored.

If the generated file is not ignored by Git, warn the user that it appears in
the working tree, but still keep it inside the repository as requested.

Never commit or remove the file automatically.

Before overwriting an existing release-notes file for the same label:
- inspect it;
- update/replace it only when it clearly represents the same requested output;
- do not overwrite unrelated notes.

## `--redo` GitHub update

For:

```text
release-notes --redo <tag>
```

without `--file`:

1. fetch the current GitHub release notes;
2. generate the complete replacement notes;
3. show both current and proposed notes;
4. ask explicit approval;
5. only then run the equivalent of:

```text
gh release edit <tag> --notes-file <temporary-file>
```

Use a temporary file only as transport for the GitHub CLI.

Remove that temporary transport file after a successful update.

Do not treat approval to generate notes as approval to edit GitHub.

## Repository safety

Except for `.release-notes/` when `--file` is supplied, this skill does not
modify the repository.

Do not:
- modify source/tests/configuration;
- stage;
- commit;
- tag;
- push;
- create a release;
- publish packages;
- reset/stash/clean/rebase/amend;
- overwrite unrelated user work.

The approved `--redo` GitHub edit changes only the notes of the specified
existing release.

## Final response

Keep the response compact.

Direct generation:
- provide the complete release notes;
- state the resolved comparison range;
- include the saved path when `--file` was used;
- mention material unresolved classification uncertainty, if any.

After an approved `--redo`:
- confirm which release was updated;
- include its URL when available.

Do not add generic offers, investigation transcripts, or command dumps.
