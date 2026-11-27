---
name: release
description: >-
  Cut one GitHub release from a bump keyword or an explicit version. Explicit
  versions are literal release identifiers: 1.2.3 creates title/tag 1.2.3 and
  v1.2.3 creates title/tag v1.2.3. Validates the exact release tree, commits
  and pushes configured release metadata after approval, generates persistent
  release notes under .release-notes/, requires approval before release
  creation, and leaves artifact publishing to repository automation. Use only
  when the user invokes it by name.
argument-hint: "[project] <major|minor|patch [beta]|stable|version>"
disable-model-invocation: true
---

# Release

Create one GitHub release, pinned to one exact validated SHA, with notes from
`release-notes`. Publishing is left to repository automation.

## Contract

The only mutations are an approved release-metadata commit and its push
(through the `commit` skill, and only when repository instructions define
metadata changes), the switch to and `pull --ff-only` of the local
`<release-branch>` that prepares it, temporary validation worktrees outside the
repository, the file `.release-notes/<release-id>.md`, and an approved GitHub
release. Never publish packages directly, push tags manually, overwrite an
existing tag or release, reset, stash, clean, rebase, amend, force-push, or
touch unrelated files.

## Invocation

```text
release [project] major | minor | patch
release [project] major beta | minor beta | patch beta
release [project] stable
release [project] 1.4.0 | 1.4.0-rc.1 | v1.4.0 | v1.4.0-rc.1
```

`[project]` picks one repository when the working directory holds several, or
one independently versioned package in a monorepo. Without a release spec,
resolve the repository and ask what to release; never infer a bump. Reject
extra tokens instead of ignoring them.

Read `AGENTS.md`, `CLAUDE.md`, and release-specific instructions. They may
define the release unit, release branch, whether prereleases are allowed,
metadata markers, the metadata commit message, validation commands, and the
publishing workflow. Ask when they conflict materially.

## 1. Resolve one repository and release unit

Derive `<dir>`, `<slug>`, `<default-branch>`, and `<release-branch>`; use
`git -C <dir>` and `--repo <slug>` throughout. `<slug>` comes from `origin`;
when it is a fork, ask whether to release there:

```bash
gh repo view "$(git -C <dir> remote get-url origin)" --json nameWithOwner,isFork
```

`<release-branch>` is `<default-branch>` unless instructions name another.
Never release several repositories in one run.

In a monorepo, use the requested package when instructions define
independently versioned units, the single repository-wide version when that is
clearly the convention, and ask when several units are plausible. Never assume
every monorepo shares one version. A package unit has its own tag pattern (such
as `<package>@<version>`), from instructions or its existing tags; ask when
neither shows one. Its `<release-id>` is that pattern filled with the version,
and every tag lookup below (bump base, newness, highest version, notes base)
considers only tags of that pattern, by their version part.

## 2. Resolve the release identifier

Refresh refs first with `git -C <dir> fetch origin --tags --prune`; if it fails
(for example, a tag that moved on the remote "would clobber existing tag"),
stop and report it. The fetch never prunes local tags, so read existing tags
from the remote (`<remote-tags>`):

```bash
git -C <dir> ls-remote --tags --refs origin | sed 's|.*refs/tags/||'
```

**An explicit version** matches the semver pattern below, with an optional
leading `v`, and is used **verbatim** (filling a package unit's tag pattern):
`1.2.3` releases tag and title `1.2.3`, and `v1.2.3` releases tag and title
`v1.2.3`. Never add or remove a `v`, change casing, rewrite the prerelease
suffix, or infer a prefix.

```text
^v?(0|[1-9][0-9]*)\.(0|[1-9][0-9]*)\.(0|[1-9][0-9]*)(-(0|[1-9][0-9]*|[0-9]*[A-Za-z-][0-9A-Za-z-]*)(\.(0|[1-9][0-9]*|[0-9]*[A-Za-z-][0-9A-Za-z-]*))*)?(\+[0-9A-Za-z-]+(\.[0-9A-Za-z-]+)*)?$
```

**`major`, `minor`, `patch`** bump the highest pure stable bare-semver tag in
`<remote-tags>` and produce bare semver:

```bash
git -C <dir> ls-remote --tags --refs origin | sed 's|.*refs/tags/||' \
  | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -1
```

`major` → `(X+1).0.0`, `minor` → `X.(Y+1).0`, `patch` → `X.Y.(Z+1)`. With no
such base tag, stop and ask for an explicit version.

**`<bump> beta`** computes the stable target, then the next `<target>-beta.N`,
starting at `<target>-beta.0`.

**`stable`** finalizes an existing bare-semver beta line by stripping
`-beta.N`. With several plausible lines, show them and ask; with none, ask for
a bump or an explicit version.

A release is a prerelease when its version carries a prerelease suffix.
Respect instructions that forbid prereleases. The new version must be higher,
by semver precedence with any leading `v` ignored, than every version tag in
`<remote-tags>`; otherwise (a hotfix below a newer release, a bump from bare
tags while newer ones are `v`-prefixed) show the highest and ask.

## 3. The identifier must be new

```bash
git -C <dir> rev-parse -q --verify "refs/tags/<release-id>"
gh release view "<release-id>" --repo <slug>
```

Check the other `v` spelling too, since `1.4.0` and `v1.4.0` are the same
version. If a release exists, stop: updating its notes belongs to
`release-notes --redo <release-id>`. If only the tag exists (pushed by hand),
stop and report that this skill never releases an existing tag, so the user
creates that release by hand or picks a new version.

## 4. Target SHA

The target is `origin/<release-branch>`:
`git -C <dir> rev-parse "origin/<release-branch>"` gives `<target-sha>`. The
release is pinned to that SHA, never a moving branch name, and never to the
locally checked-out feature branch just because it is active.

Resolve the notes base (step 9) for `<target-sha>` now. If it points at
`<target-sha>`, nothing is new since that release, so stop.

## 5. Working-copy safety

Check `git status --short` and the current branch. Uncommitted changes are
never included in a release. Without metadata edits, a dirty tree does not
block releasing the remote SHA, because validation can run in a separate
worktree. If metadata edits are required and local state prevents preparing
them safely on the release branch, stop and name the blocking paths.

## 6. Release metadata

Only when instructions define pending release markers (such as replacing
`Unreleased` or bumping a version file); never invent metadata edits. When the
configured metadata at `<target-sha>` already matches `<release-id>` (a release
PR already bumped it), skip this step.

Prepare on `<release-branch>`, never on a feature branch, and only when
switching cannot disturb user work. Record the current branch first as
`<original-branch>` (the commit, when HEAD is detached):

```bash
git -C <dir> switch "<release-branch>"
git -C <dir> pull --ff-only origin "<release-branch>"
```

If the pull fails (the branches diverged), stop and report it. Before editing
metadata, this count must be 0:

```bash
git -C <dir> rev-list --count "origin/<release-branch>..<release-branch>"
```

Otherwise the local branch holds unpushed commits the metadata push would
carry along, so stop and name them
(`git -C <dir> log --oneline "origin/<release-branch>..<release-branch>"`).
Also stop and name the paths when any metadata path has staged, unstaged, or
untracked changes (`git -C <dir> status --short -- <metadata paths>`).

When this step ends, on success or stop, switch back with
`git -C <dir> switch "<original-branch>"` (`switch --detach` for a commit)
unless uncommitted metadata edits remain. User changes carried along that do
not collide do not block the switch back. If metadata edits remain uncommitted,
or Git refuses the switch, stay on `<release-branch>` and report both branches;
never carry the edits over.

Apply only the configured changes and show the diff. Run the release validation
(step 7) in a temporary worktree at `HEAD` with the edited metadata files
copied in (step 8's worktree rules), so user changes in the main tree cannot
affect it. If it fails, stop with no commit, push, or release. Otherwise
resolve `<metadata-message>`: the commit message instructions define, else a
commit-style subject such as `chore: release <release-id>`. Show the exact
changes and `<metadata-message>`, and ask to commit and push them. If the user
declines, stop with no release and report the uncommitted edits and both
branches.

On approval, record `HEAD`, then run
`commit --auto-all --dir <dir> --message "<metadata-message>"` with the
metadata file paths as its paths, or follow `../commit/SKILL.md` inline as if
invoked so when the harness cannot invoke skills; if neither works, stop with
no release and say that `commit` is missing. This approval
covers the commit and its push, so `--auto-all` skips the duplicate
confirmations, but `commit` still inspects, stages narrowly, validates, and
preserves unrelated work. The commit excludes `.release-notes/`, unrelated
changes, and anything beyond the configured metadata.

Afterwards, continue only when `HEAD` has moved and
`git -C <dir> status --short -- <metadata paths>` is clean; otherwise
`commit` created nothing or left metadata behind, so stop with no release.
Record `HEAD` as `<metadata-sha>`, fetch again, and confirm the push landed:

```bash
git -C <dir> merge-base --is-ancestor "<metadata-sha>" "origin/<release-branch>"
```

If it fails (for example, a protected branch rejected the push), stop: create
no release, and report that the metadata commit is local and unpushed.
Otherwise recapture `<target-sha>`. From here on, every stop also reports the
pushed metadata commit and that no release was created.

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
# install dependencies, then run the validation commands there
git -C <dir> worktree remove --force "<temporary-worktree>"
```

`<temporary-worktree>` is a new path in the scratchpad or system temp
directory, never under `<dir>`. It starts without dependencies or generated
files, so first run the repository's install or restore step from the
instructions or CI. Remove it on every exit path, failure included (`--force`,
since validation may leave files behind), and remove only the one this workflow
created. If validation fails, stop, show the failing command and its useful
output, and create nothing. After a metadata commit this still runs, so the
final release SHA itself is proven.

## 9. Release-notes range

Hand `release-notes` the repository, the unit, `<release-id>`, `<target-sha>`,
the prerelease state, and the base (omitted for the empty tree); if the
harness cannot invoke skills, follow `../release-notes/SKILL.md` inline; if
neither works, stop with no release and say that `release-notes` is missing:

```text
release-notes [<unit>] [<base>] --head <target-sha> --tag <release-id> [--prerelease]
```

Resolve the base with its base rule (Range)
for `<target-sha>`, the unit, and this prerelease state, so a later `--redo`
yields the same range.

## 10. Release notes

Generate the notes through `release-notes` in its house format, monorepo
package sections included, and write the exact release body to
`<dir>/.release-notes/<release-id>.md` (for example
`.release-notes/v1.4.0.md`). If that file already exists (from an aborted run),
show it and ask before replacing it. Create it only **after** any metadata
commit so it cannot end up in that commit. It stays for inspection and is never
committed automatically; if Git does not ignore it, warn that it shows in the
working tree.

## 11. Approval

First decide `<release-flag>`: `--prerelease` for a prerelease. For a stable
release it is `--latest` only when `<release-id>` is higher than every existing
stable release, compared by semver precedence with any leading `v` ignored;
otherwise it is `--latest=false`:

```bash
gh release list --repo <slug> --exclude-pre-releases --exclude-drafts \
  --limit 10000 --json tagName -q '.[].tagName'
```

When `<release-id>` or an existing stable tag does not parse as semver (for
example monorepo `pkg@1.2.3` tags), ask whether this release should be Latest;
never guess.

```text
Repository: owner/repo
Release: 1.4.0
Tag: 1.4.0
Type: stable
Latest: yes
Target: abc1234
Base: 1.3.0
Notes: .release-notes/1.4.0.md
```

With the empty-tree base, show `Base: none (full history)`. Show the **exact**
notes too, then ask `Create release <release-id>?`. Approval of metadata
changes is not approval of the release. If the user edits the notes, update the
file, show the revised notes, and get approval again.

## 12. Re-check before creating

Fetch again and confirm the tag and release are still absent, the notes' base
still holds, and `<target-sha>` is still on the branch:

```bash
git -C <dir> merge-base --is-ancestor "<target-sha>" "origin/<release-branch>"
```

If it is not (a force-push removed it), stop. If `origin/<release-branch>` has
moved past the approved target, show both SHAs and ask whether to release the
reviewed `<target-sha>` or stop so notes and validation can be regenerated.
Never retarget silently.

## 13. Create the release

```bash
gh release create "<release-id>" \
  --repo <slug> \
  --target "<target-sha>" \
  --title "<release-id>" \
  --notes-file "<dir>/.release-notes/<release-id>.md" \
  <release-flag>
```

`<release-flag>` is the one decided in step 11. Tag and title always equal
`<release-id>`; `gh release create` creates the tag at the pinned SHA, so never
push one manually.

## 14. Verify

```bash
gh release view "<release-id>" --repo <slug> \
  --json tagName,name,url,isPrerelease
git -C <dir> ls-remote origin "refs/tags/<release-id>"
gh api "repos/<slug>/releases/latest" --jq .tag_name
```

Confirm the tag, title, prerelease state, and URL; that the tag points at
`<target-sha>` (GitHub ignores `--target` when the tag appeared after step 12);
and that the Latest release is `<release-id>` exactly when `<release-flag>` is
`--latest`. Report any mismatch; command output alone is not proof.

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
