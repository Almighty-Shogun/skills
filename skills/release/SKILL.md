---
name: release
description: >-
  Cut one GitHub release from a bump keyword or an explicit version. Explicit
  versions are literal release identifiers: 1.2.3 creates title/tag 1.2.3 and
  v1.2.3 creates title/tag v1.2.3. Validates the exact release tree, generates
  persistent release notes under .release-notes/, requires approval before
  release creation, and leaves artifact publishing to repository automation.
argument-hint: "[project] <major|minor|patch [beta]|stable|version>"
disable-model-invocation: true
---

# Release

Create one GitHub release safely.

This skill:
- resolves one repository and one release identifier;
- validates release metadata before committing it;
- validates the exact tree that will be released;
- generates release notes through `release-notes`;
- requires explicit approval before GitHub release creation;
- creates the GitHub release pinned to one exact SHA.

It never publishes packages/artifacts directly.

## Contract

Possible mutations are limited to:

1. An explicitly approved release-metadata commit when repository instructions
   define required release metadata changes. Use the `commit` skill.
2. Creation of `.release-notes/<release-id>.md` inside the repository.
3. An explicitly approved GitHub release.

Do not:
- publish npm/NuGet/Composer/etc. packages directly;
- push tags manually;
- overwrite an existing tag/release;
- reset, stash, clean, rebase, amend, or force-push;
- modify unrelated files;
- create a release without explicit approval.

## Invocation

```text
release
release major
release minor
release patch

release major beta
release minor beta
release patch beta

release stable

release 1.4.0
release 1.4.0-rc.1
release v1.4.0
release v1.4.0-rc.1
```

A workspace/project selector may precede the release spec.

If no release spec is supplied, resolve the repository first and ask what to
release. Do not infer a bump silently.

Reject extra release-spec tokens instead of silently ignoring them.

## Repository instructions

Read applicable:
- `AGENTS.md`;
- `CLAUDE.md`;
- release-specific repository instructions.

Repository instructions may define:
- release unit/package;
- default branch override;
- whether prereleases are allowed;
- release metadata markers;
- metadata commit message;
- exact validation commands;
- publishing workflow expectations.

If instructions conflict materially, ask rather than guessing.

## 1. Resolve one repository / release unit

Determine:
- `<dir>` repository root;
- `<slug>` GitHub `owner/name`;
- `<default-branch>`;
- release unit/package when the repository is a monorepo.

Use:

```text
git -C <dir> ...
gh ... --repo <slug>
```

### Workspaces / monorepos

Never release several independent repositories in one run.

For monorepos:
- if repository instructions define independently versioned packages, resolve
  the requested package/release unit;
- if the repository clearly uses one repository-wide release version, use that;
- if several independently versioned release units are plausible and the user
  did not choose one, ask which unit to release.

Do not silently assume all monorepos share one version.

## 2. Resolve the release identifier

Refresh refs first:

```bash
git -C <dir> fetch origin --tags --prune
```

The resolved value is `<release-id>`.

### Explicit version

Accept a semver-like explicit identifier with an optional literal leading `v`:

```text
1.2.3
1.2.3-beta.1
1.2.3+meta
v1.2.3
v1.2.3-beta.1
```

Validation shape:

```regex
^v?[0-9]+\.[0-9]+\.[0-9]+(-[0-9A-Za-z.-]+)?(\+[0-9A-Za-z.-]+)?$
```

**Use the user's value verbatim.**

Examples:

```text
input: 1.2.3
release-id: 1.2.3
tag: 1.2.3
release title: 1.2.3
```

```text
input: v1.2.3
release-id: v1.2.3
tag: v1.2.3
release title: v1.2.3
```

Do not:
- add `v`;
- remove `v`;
- normalize casing;
- rewrite the prerelease suffix;
- infer a repository prefix.

The Git tag and GitHub release title are always exactly `<release-id>`.

### `major` / `minor` / `patch`

Bump modes retain the existing bare-semver convention.

Find the highest pure stable bare-semver tag:

```bash
git -C <dir> tag -l |
  grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' |
  sort -V |
  tail -1
```

Then:

```text
major → (X+1).0.0
minor → X.(Y+1).0
patch → X.Y.(Z+1)
```

The generated `<release-id>` is bare semver.

If no suitable bare-semver base exists, stop and ask the user to provide an
explicit version rather than guessing a prefix/history convention.

### `<bump> beta`

For:

```text
major beta
minor beta
patch beta
```

first calculate the stable bump target, then find:

```text
<target>-beta.N
```

and produce the next beta number.

No existing beta:

```text
<target>-beta.0
```

Bump-generated beta identifiers remain bare semver.

### `stable`

`stable` finalizes an existing bare-semver beta line.

If exactly one plausible active beta line exists, strip its `-beta.N` suffix.

If several plausible beta lines exist, show them and ask which one to finalize.

If no beta exists, stop and ask for:
- `major`;
- `minor`;
- `patch`;
- or an explicit version.

### Prerelease status

Determine prerelease status from the semantic portion after an optional leading
`v`.

Examples:

```text
1.2.3          → stable
v1.2.3         → stable
1.2.3-beta.1   → prerelease
v1.2.3-rc.1    → prerelease
```

Respect repository instructions that forbid prereleases.

## 3. Hard existence safeguards

The release identifier must be new.

Check both:

```bash
git -C <dir> rev-parse -q --verify "refs/tags/<release-id>"
gh release view "<release-id>" --repo <slug>
```

If either already exists, stop.

Updating the notes of an existing release belongs to:

```text
release-notes --redo <release-id>
```

`release` never re-cuts or overwrites an existing tag/release.

## 4. Resolve the initial target SHA

Default target:

```text
origin/<default-branch>
```

Capture it:

```bash
git -C <dir> rev-parse "origin/<default-branch>"
```

Call this `<target-sha>`.

The release is pinned to an exact SHA, not to a moving branch name.

Do not release the currently checked-out feature branch merely because it is
active locally.

Repository instructions may explicitly define another release target.

## 5. Inspect working-copy safety

Inspect:

```bash
git -C <dir> status --short
git -C <dir> branch --show-current
```

Uncommitted user changes are never silently included in the release.

Do not:
- reset;
- stash;
- clean;
- move unrelated changes between branches;
- overwrite them to prepare a release.

If no release metadata must be changed, a dirty local tree does not prevent
validating/releasing the remote target SHA because validation can use an
isolated worktree.

If metadata edits are required and local state prevents safely preparing them
on the release base branch, stop and explain the blocking paths.

## 6. Prepare required release metadata

Run this section only when repository instructions explicitly define pending
release metadata/markers.

Examples:
- replacing `Unreleased`;
- updating required release metadata files;
- updating a repository-specific version marker.

Do not invent metadata edits merely because a release is being created.

### Prepare on the release base

Metadata changes must be based on the current remote release branch.

Safely switch/update only when doing so does not disturb user work:

```bash
git -C <dir> switch "<default-branch>"
git -C <dir> pull --ff-only origin "<default-branch>"
```

Never make release metadata commits on an unrelated feature branch.

Apply only the configured metadata changes.

Show the resulting diff.

### Validate before committing/pushing

Before any metadata commit is created, run the repository's release validation
against the resulting working tree.

If validation fails:
- stop;
- do not commit;
- do not push;
- do not create the release.

Then show the exact metadata changes and ask explicit approval to commit/push
them.

If approved, invoke `commit --auto-all` with scope restricted to the metadata
changes.

If your harness cannot invoke one skill from another, read `../commit/SKILL.md`
and follow its rules inline for the same scope, as if invoked with `--auto-all`.

The release workflow's explicit approval already covers both the metadata commit
and its push, so do not ask for the same approvals again. `--auto-all` skips only
the duplicate confirmations; the `commit` skill must still inspect, group,
stage narrowly, validate, and preserve unrelated work.

The metadata commit must not include:
- `.release-notes/`;
- unrelated local changes;
- source edits not required by the configured release metadata rules.

After the commit skill pushes:

```bash
git -C <dir> fetch origin --tags --prune
git -C <dir> rev-parse "origin/<default-branch>"
```

Recapture `<target-sha>`.

## 7. Resolve release validation

Determine the repository's release validation commands in this order:

1. explicit `## Releasing` / repository instructions;
2. actual CI/release workflow configuration;
3. unambiguous repository manifests/scripts;
4. ask when no reliable validation can be determined.

Typical evidence may include:
- build;
- test;
- typecheck;
- lint;
- package-specific build;
- monorepo dependency-ordered build.

Do not run publishing commands as validation.

Do not silently skip validation.

## 8. Validate the exact target SHA

The tree being validated must be the tree that `<target-sha>` identifies.

### Validate in-place only when exact

Use the current checkout only if:
- `HEAD == <target-sha>`;
- the working tree is clean;
- validation will not depend on unrelated generated/local state.

Otherwise create a temporary detached worktree:

```bash
git -C <dir> worktree add --detach "<temporary-worktree>" "<target-sha>"
```

Run the resolved validation commands there.

Always remove the temporary worktree on every exit path, including validation
failure:

```bash
git -C <dir> worktree remove "<temporary-worktree>"
```

Remove only the worktree created by this workflow. Never leave release-validation
worktrees behind.

Never delete/reset/stash the user's actual checkout to make validation easier.

If exact-SHA validation fails:
- stop;
- show the failing command and useful output;
- do not create the release.

### Metadata validation note

When release metadata was changed, pre-commit validation in step 6 prevents a
broken metadata tree from being pushed.

This exact-SHA validation still runs after the metadata commit so the final
release SHA itself is proven.

## 9. Resolve release-notes range

Use `release-notes` conceptually; do not depend on its numbered internal steps.

If your harness cannot invoke one skill from another, read
`../release-notes/SKILL.md` and follow its rules inline.

Provide it:
- repository `<dir>` / `<slug>`;
- resolved `<release-id>`;
- exact `<target-sha>`;
- prerelease/stable state;
- resolved base.

### Stable release base

Use the latest published non-prerelease release before the target.

### Prerelease base

Use the most recent published release of any kind before the target.

This means successive betas normally describe changes since the previous beta,
while the final stable release can describe the full change since the prior
stable release.

If no prior published release exists, use the repository's first commit.

A user-supplied explicit base overrides automatic base resolution.

## 10. Generate persistent release notes

Generate the raw Markdown through `release-notes`.

The release notes must use the established release-note house style, including
the default emoji sections:

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

Monorepo package sections remain separate according to the `release-notes`
skill.

Write the exact GitHub release body inside the repository:

```text
<dir>/.release-notes/<safe-release-id>.md
```

Examples:

```text
.release-notes/1.4.0.md
.release-notes/v1.4.0.md
.release-notes/1.5.0-beta.0.md
```

Create this file only **after** any release-metadata commit has completed so it
cannot accidentally become part of that commit.

The file remains after release creation for inspection.

Do not commit it automatically.

If it is not ignored by Git, warn the user that it appears in the working tree.

## 11. Approval gate

Show the consequential release plan:

```text
Repository: owner/repo
Release: 1.4.0
Tag: 1.4.0
Type: stable
Target: abc1234
Base: 1.3.0
Notes: .release-notes/1.4.0.md
```

For explicit `v` input:

```text
Release: v1.4.0
Tag: v1.4.0
```

Then show the **exact release notes** that will be submitted.

Ask:

```text
Create release <release-id>?
```

Do not create the GitHub release until the user explicitly approves.

Do not treat earlier approval of metadata changes as approval to create the
release.

If the user edits the notes:
- update `.release-notes/<safe-release-id>.md`;
- show the exact revised notes again;
- obtain release approval against the revised content.

## 12. Re-check immediately before GitHub mutation

After approval, refresh remote state.

Re-check:
- `<release-id>` tag is still absent;
- `<release-id>` GitHub release is still absent;
- `<target-sha>` still resolves;
- release-note base assumptions remain valid.

Also resolve the current release branch SHA:

```bash
git -C <dir> fetch origin --tags --prune
git -C <dir> rev-parse "origin/<default-branch>"
```

If the release branch advanced after `<target-sha>` was validated/approved,
surface:

```text
Approved target: abc1234
Current origin/main: def5678
```

Do not silently retarget.

Ask whether to:
- create the already-reviewed release at `<target-sha>`;
- or stop so notes/validation can be regenerated for the newer SHA.

## 13. Create the GitHub release

After final approval/re-check:

```bash
gh release create "<release-id>" \
  --repo <slug> \
  --target "<target-sha>" \
  --title "<release-id>" \
  --notes-file "<dir>/.release-notes/<safe-release-id>.md" \
  <release-flag>
```

Where:

```text
stable     → --latest
prerelease → --prerelease
```

Hard rule:

```text
tag == release title == <release-id>
```

For an explicit version, `<release-id>` is exactly what the user provided.

Do not:
- push a tag manually;
- create a second tag;
- normalize the release identifier.

`gh release create` owns creation of the tag/release at the pinned target SHA.

## 14. Verify creation

Verify from GitHub:

```bash
gh release view "<release-id>" --repo <slug> \
  --json tagName,name,url,isPrerelease,targetCommitish
```

Confirm:
- expected tag;
- expected title;
- prerelease/stable state;
- release URL.

Do not infer success only from command exit/output.

## 15. Publishing automation

Inspect repository instructions and release-triggered workflows before claiming
what happens next.

If a release-triggered workflow is identified, report its actual role
concisely.

Example:

```text
Release workflow: Publish Packages
```

If none can be established:

```text
No release-triggered publishing workflow was identified.
```

Do not claim that npm/NuGet/deployment will happen merely because a GitHub
release exists.

Never run publishing commands locally unless the user explicitly asks for a
separate publishing task.

## Final response

Keep the result compact.

Example:

```text
Created 1.4.0
Target: abc1234
Notes: .release-notes/1.4.0.md
GitHub: <release URL>

Release workflow: Publish Packages
```

If metadata was committed, mention the commit briefly.

If automation was not identified, say so.

## Safety summary

- One repository/release unit per run.
- Explicit versions are literal; preserve a leading `v` when supplied.
- Bump-generated versions remain bare semver.
- Tag and release title always match exactly.
- Existing tags/releases are hard stops.
- Validate metadata before committing it.
- Validate the exact final release SHA.
- Never disturb unrelated local work to perform validation.
- Release notes stay inspectable under `.release-notes/`.
- Release creation always requires explicit approval.
- Never silently retarget when the release branch advances.
- Never publish packages directly.
- Never reset, stash, clean, rebase, amend, or force-push.
