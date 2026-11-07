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

Create one GitHub release, pinned to one exact validated SHA, with notes from
`release-notes`. Publishing is left to repository automation.

## Contract

The only mutations are an approved release-metadata commit (through the
`commit` skill, and only when repository instructions define metadata changes),
the file `.release-notes/<release-id>.md`, and an approved GitHub release.
Never publish packages directly, push tags manually, overwrite an existing tag
or release, reset, stash, clean, rebase, amend, force-push, or touch unrelated
files.

## Invocation

```text
release [project] major | minor | patch
release [project] major beta | minor beta | patch beta
release [project] stable
release [project] 1.4.0 | 1.4.0-rc.1 | v1.4.0 | v1.4.0-rc.1
```

Without a release spec, resolve the repository and ask what to release; never
infer a bump. Reject extra tokens instead of ignoring them.

Read `AGENTS.md`, `CLAUDE.md`, and release-specific instructions. They may
define the release unit, default branch, whether prereleases are allowed,
metadata markers, the metadata commit message, validation commands, and the
publishing workflow. Ask when they conflict materially.

## 1. Resolve one repository and release unit

Derive `<dir>`, `<slug>`, and `<default-branch>`; use `git -C <dir>` and
`--repo <slug>` throughout. Never release several repositories in one run. In a
monorepo, use the requested package when instructions define independently
versioned units, the single repository-wide version when that is clearly the
convention, and ask when several units are plausible. Never assume every
monorepo shares one version.

## 2. Resolve the release identifier

Refresh refs first: `git -C <dir> fetch origin --tags --prune`.

**An explicit version** matches
`^v?[0-9]+\.[0-9]+\.[0-9]+(-[0-9A-Za-z.-]+)?(\+[0-9A-Za-z.-]+)?$` and is used
**verbatim**: `1.2.3` releases tag and title `1.2.3`, and `v1.2.3` releases tag
and title `v1.2.3`. Never add or remove a `v`, change casing, rewrite the
prerelease suffix, or infer a prefix.

**`major`, `minor`, `patch`** bump the highest pure stable bare-semver tag and
produce bare semver:

```bash
git -C <dir> tag -l | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -1
```

`major` → `(X+1).0.0`, `minor` → `X.(Y+1).0`, `patch` → `X.Y.(Z+1)`. With no
such base tag, stop and ask for an explicit version.

**`<bump> beta`** computes the stable target, then the next `<target>-beta.N`,
starting at `<target>-beta.0`.

**`stable`** finalizes an existing bare-semver beta line by stripping `-beta.N`. With
several plausible lines, show them and ask; with none, ask for a bump or an
explicit version.

A release is a prerelease when the part after an optional leading `v` carries a
prerelease suffix. Respect instructions that forbid prereleases.

## 3. The identifier must be new

```bash
git -C <dir> rev-parse -q --verify "refs/tags/<release-id>"
gh release view "<release-id>" --repo <slug>
```

If either exists, stop. Updating an existing release's notes belongs to
`release-notes --redo <release-id>`.

## 4. Target SHA

The target is `origin/<default-branch>` unless instructions define another:
`git -C <dir> rev-parse "origin/<default-branch>"` gives `<target-sha>`. The
release is pinned to that SHA, never a moving branch name, and never to the
locally checked-out feature branch just because it is active.

## 5. Working-copy safety

Check `git status --short` and the current branch. Uncommitted changes are
never included in a release. Without metadata edits, a dirty tree does not
block releasing the remote SHA, because validation can run in a separate
worktree. If metadata edits are required and local state prevents preparing
them safely on the release branch, stop and name the blocking paths.

## 6. Release metadata

Only when instructions define pending release markers (such as replacing
`Unreleased` or bumping a version file); never invent metadata edits.

Prepare on the current remote release branch, never on a feature branch, and
only when switching cannot disturb user work:

```bash
git -C <dir> switch "<default-branch>"
git -C <dir> pull --ff-only origin "<default-branch>"
```

Apply only the configured changes, show the diff, and run the release
validation (step 7) against that tree. If it fails, stop with no commit, push,
or release. Otherwise show the exact changes and ask to commit and push them.

On approval, run `commit --auto-all` scoped to the metadata changes, or follow
`../commit/SKILL.md` inline as if invoked so when the harness cannot invoke
skills. This approval covers the commit and its push, so `--auto-all` skips
the duplicate confirmations, but `commit` still inspects, stages narrowly,
validates, and preserves unrelated work. The commit excludes `.release-notes/`,
unrelated changes, and anything beyond the configured metadata.

Afterwards, fetch again and recapture `<target-sha>`.

## 7. Release validation

Resolve the commands from, in order: explicit `## Releasing` instructions, the
CI or release workflow, unambiguous manifests and scripts; ask when none is
reliable. Typical checks are build, test, typecheck, lint, and package or
dependency-ordered builds. Never run publish commands as validation, and never
skip validation silently.

## 8. Validate the exact target SHA

The validated tree must be the tree at `<target-sha>`. Validate in place only
when `HEAD == <target-sha>`, the tree is clean, and no local or generated state
can leak in. Otherwise:

```bash
git -C <dir> worktree add --detach "<temporary-worktree>" "<target-sha>"
# run the validation commands there
git -C <dir> worktree remove "<temporary-worktree>"
```

Remove that worktree on every exit path, failure included, and remove only the
one this workflow created. If validation fails, stop, show the failing command
and its useful output, and create nothing. After a metadata commit this still
runs, so the final release SHA itself is proven.

## 9. Release-notes range

Hand `release-notes` the repository, `<release-id>`, `<target-sha>`, the
prerelease state, and the base; if the harness cannot invoke skills, follow
`../release-notes/SKILL.md` inline. A stable release's base is the latest
published non-prerelease release before the target; a prerelease's is the most
recent published release of any kind, so successive betas describe changes
since the previous beta. With no prior release, use the first commit. A base the
user supplies wins.

## 10. Release notes

Generate the notes through `release-notes` in its house format, monorepo
package sections included, and write the exact release body to
`<dir>/.release-notes/<safe-release-id>.md` (for example
`.release-notes/v1.4.0.md`). Create it only **after** any metadata commit so it
cannot end up in that commit. It stays for inspection and is never committed
automatically; if Git does not ignore it, warn that it shows in the working
tree.

## 11. Approval

```text
Repository: owner/repo
Release: 1.4.0
Tag: 1.4.0
Type: stable
Target: abc1234
Base: 1.3.0
Notes: .release-notes/1.4.0.md
```

Show the **exact** notes too, then ask `Create release <release-id>?`. Approval
of metadata changes is not approval of the release. If the user edits the
notes, update the file, show the revised notes, and get approval again.

## 12. Re-check before creating

Fetch again and confirm the tag and release are still absent, `<target-sha>`
still resolves, and the notes' base still holds. If `origin/<default-branch>`
has moved past the approved target, show both SHAs and ask whether to release
the reviewed `<target-sha>` or stop so notes and validation can be regenerated.
Never retarget silently.

## 13. Create the release

```bash
gh release create "<release-id>" \
  --repo <slug> \
  --target "<target-sha>" \
  --title "<release-id>" \
  --notes-file "<dir>/.release-notes/<safe-release-id>.md" \
  <release-flag>
```

`<release-flag>` is `--latest` for a stable release and `--prerelease` for a
prerelease. Tag and title always equal `<release-id>`; `gh release create`
creates the tag at the pinned SHA, so never push one manually.

## 14. Verify

```bash
gh release view "<release-id>" --repo <slug> \
  --json tagName,name,url,isPrerelease,targetCommitish
```

Confirm the tag, title, prerelease state, and URL; command output alone is not
proof.

## 15. Publishing automation

Read the instructions and release-triggered workflows before saying what
happens next, and report the workflow's actual role (`Release workflow: Publish
Packages`) or `No release-triggered publishing workflow was identified.` Never
claim a package or deployment will publish just because a release exists, and
never run publish commands locally.

## Final response

```text
Created 1.4.0
Target: abc1234
Notes: .release-notes/1.4.0.md
GitHub: <release URL>

Release workflow: Publish Packages
```

Mention a metadata commit briefly when one was made.
