---
name: merge-pr
description: >-
  Safely merge one GitHub pull request after checking repository state, PR
  eligibility, CI, review status, and a fresh code review. Supports PR numbers
  and URLs, preserves merge commits by default, supports legitimate non-default
  base branches, and cleans up the merged branch when appropriate. --auto-merge
  skips only the final confirmation; it never bypasses safety gates.
argument-hint: "[<number-or-url>] [--no-review] [--comment] [--auto-merge]"
disable-model-invocation: true
---

# Merge PR

Merge one pull request and clean up its branch when safe.

One pull request per run.

## Contract

Allowed mutations:
- merge the selected pull request;
- delete its remote head branch when appropriate;
- update the PR's base branch locally after merge;
- delete the local merged head branch with `git branch -d` when safe.

Do not:
- edit source code;
- fix review findings;
- fix CI failures;
- create commits;
- rebase;
- reset;
- stash;
- amend;
- force-push;
- squash;
- close a PR without merging it;
- create releases/tags/version commits.

Use `review-pr`, `respond-pr`, `fix-pr`, or `delegate-task` when a blocker needs actual code changes.

## Invocation

```text
merge-pr
merge-pr 42
merge-pr https://github.com/owner/repo/pull/42
merge-pr 42 --no-review
merge-pr 42 --comment
merge-pr 42 --auto-merge
```

- No PR argument: list open PRs and ask which one to merge.
- PR number or URL: resolve that exact PR.
- `--no-review`: explicitly skips the fresh `review-pr` pass.
- `--comment`: forwards to `review-pr`, which still requires approval before posting.
- `--auto-merge`: skips only the final merge confirmation after every safety gate passes.

`--auto-merge` never means "ignore unsafe state".

## Repository instructions

Read applicable `AGENTS.md` and `CLAUDE.md` files.

Repository instructions may override:
- preferred base/default branch conventions;
- merge method;
- branch deletion policy;
- whether review may be skipped;
- extra merge gates.

If instructions conflict materially, ask rather than guessing.

## 1. Resolve repository and PR

Resolve:
- `<dir>` repository root;
- `<slug>` GitHub `owner/name`;
- `<default-branch>` GitHub default branch.

Use `git -C <dir> ...` for Git commands and `--repo <slug>` for `gh pr` commands.

When no PR is supplied:

```bash
gh pr list --repo <slug> --state open \
  --json number,title,isDraft,headRefName,baseRefName,url
```

Show all open PRs and ask the user which one.

For a supplied URL, extract/resolve the repository and PR number. Do not guess from an unrelated current repository.

Read the chosen PR:

```bash
gh pr view <number> --repo <slug> --json \
  number,title,state,isDraft,mergeable,mergeStateStatus,reviewDecision,\
headRefName,headRepositoryOwner,headRepository,baseRefName,url,headRefOid,\
statusCheckRollup,comments,reviews
```

The PR's actual `baseRefName` is the merge target. It does not need to equal the repository default branch.

## 2. Inspect local working-copy safety

Do not switch branches yet.

Inspect:

```bash
git -C <dir> status --short
git -C <dir> branch --show-current
git -C <dir> fetch origin
```

Preserve all local work.

Never:
- reset;
- stash;
- clean;
- discard;
- carry user modifications onto another branch merely because Git happens to permit it.

A dirty tree does not automatically block the remote merge, because the PR can be merged without changing the local branch.

However, determine whether post-merge switching/updating/cleanup would overwrite or collide with local work.

Check tracked modifications against the target base and relevant branch paths.

For untracked files, check for path collisions with the target branch before switching. Untracked files are not universally safe.

If local work blocks only post-merge cleanup, the remote merge may still proceed after the user understands that cleanup will remain incomplete.

If local state makes the merge itself ambiguous or unsafe, stop and name the exact paths/reason.

`--auto-merge` does not bypass any of these checks.

## 3. Check PR eligibility

Stop when any hard blocker exists:

- PR `state` is not `OPEN`;
- PR is a draft;
- `mergeable` is `CONFLICTING`;
- review decision is `CHANGES_REQUESTED`;
- branch protection or `mergeStateStatus` says the PR is blocked;
- required checks failed;
- required checks are still pending/running/expected;
- PR state is unknown enough that merge safety cannot be established.

Interpret `mergeStateStatus` instead of merely displaying it.

Examples:

- `DIRTY` → merge conflicts; block.
- `BLOCKED` → branch protection/review/check requirement blocks; inspect and block.
- `UNSTABLE` → determine which check/review state causes it before proceeding.
- `BEHIND` → report that the head is behind its base; do not automatically merge/rebase/update the branch.
- `UNKNOWN` → do not assume safety.

Repository-specific required-state semantics may override these defaults.

### Check-state handling

For each relevant required check, require an acceptable terminal state before merging.

Block on states such as:
- failure;
- timed out;
- cancelled;
- queued;
- in progress;
- pending;
- expected/waiting.

An empty check rollup means **no checks ran**, not "checks passed". Report that honestly.

Do not weaken or bypass branch protection.

## 4. Surface relevant review feedback

Use review/comment data only for merge safety and user awareness.

Show only feedback that remains relevant, such as:
- unresolved actionable review requests;
- unresolved questions material to merging;
- change requests;
- material unresolved concerns.

Do not dump:
- resolved threads;
- approvals;
- bot chatter;
- CI notifications;
- non-actionable conversation.

Do not edit code or post replies from this skill.

Use `respond-pr` for review feedback that requires action.

## 5. Run a fresh review

Unless the user supplied `--no-review`, run `review-pr` against the exact PR.

If your harness cannot invoke one skill from another, read `../review-pr/SKILL.md`
and follow its rules inline against the same PR.

Forward `--comment` only when the user supplied it.

The review must evaluate the current PR diff against its actual base branch.

Proceed only when `review-pr` reports no blocking findings.

If blocking findings exist, stop and report them.

When `--no-review` was supplied, visibly state:

```text
review: skipped by --no-review
```

Never make an unreviewed merge look reviewed.

## 6. Verify base relationship

Inspect the PR head/base relationship without modifying history.

Use GitHub/remote refs or equivalent:

```bash
git -C <dir> merge-base origin/<base-branch> <head-ref-or-sha>
git -C <dir> rev-list --count origin/<base-branch>..<head-ref-or-sha>
git -C <dir> rev-list --count <head-ref-or-sha>..origin/<base-branch>
```

Report unusual ancestry when relevant.

Do not:
- rebase;
- merge the base into the head;
- update the PR branch;
- force-push.

A PR targeting a legitimate non-default base is supported.

## 7. Propose the merge

Show a compact plan:

```text
Merging #42 <title>

Base: development
Head: feat/cache
Mergeable: yes
Checks: passed
Review: no blocking findings
After merge: delete head branch when safe
```

Include:
- PR number/title;
- base/head;
- mergeability;
- check state;
- review result or explicit `--no-review`;
- any unresolved material feedback;
- planned branch cleanup.

Without `--auto-merge`, ask explicit approval before merging.

With `--auto-merge`, still show the plan, but the flag itself authorizes the merge once every gate below remains satisfied.

## 8. Re-check immediately before merge

Approval can become stale.

Immediately before mutating GitHub, fetch the PR again and re-check:

- `state`;
- draft status;
- head SHA (`headRefOid`);
- mergeability;
- `mergeStateStatus`;
- review decision;
- required checks;
- base branch.

If the head SHA or any material state changed since the reviewed/approved plan, stop and show the new state.

Do not merge a changed PR under stale approval or stale review.

## 9. Merge

Use the repository's required merge method when explicitly configured.

Otherwise default to a merge commit:

```bash
gh pr merge <number> --repo <slug> --merge
```

Preserve the branch commits.

Do not use:
- `--squash`;
- `--rebase`;
- force options.

### Remote branch deletion

Delete the remote head branch only when appropriate.

Before requesting deletion, determine:
- whether the head branch belongs to the same repository;
- whether the current user/repository owns it;
- whether it is protected or intentionally retained;
- whether repository instructions require keeping it.

For an ordinary same-repository disposable feature branch, deletion is appropriate.

For fork PRs, protected branches, shared long-lived branches, or uncertain ownership, leave it intact and report why.

If safe, use the host-supported branch deletion behavior as part of or after the merge.

## 10. Verify remote merge

After the merge call, verify from GitHub:

```bash
gh pr view <number> --repo <slug> --json \
  state,mergedAt,mergeCommit,url,headRefName,baseRefName
```

Require `state = MERGED`.

Record the merge commit SHA when available.

Do not infer success merely because the merge command returned without obvious output.

## 11. Update local base branch when safe

Only after the remote merge succeeds, update the PR's actual base branch locally.

If switching would overwrite/collide with local work, do **not** disturb the working copy.

Report:

```text
PR merged successfully; local base update/branch cleanup was skipped because:
- <path/reason>
```

If safe:

```bash
git -C <dir> switch <base-branch>
git -C <dir> pull --ff-only origin <base-branch>
```

Use `--ff-only` for this synchronization unless repository instructions require otherwise.

Do not create a local merge commit merely to update the base.

## 12. Delete local merged branch when safe

If a local branch matching the PR head exists and is not currently needed:

```bash
git -C <dir> branch -d <head-branch>
```

Use `-d`, never `-D`.

If `-d` refuses, stop cleanup and report it. Do not force deletion.

A cleanup failure does not mean the PR merge failed.

Distinguish:

```text
PR merged successfully
Local cleanup incomplete
```

from:

```text
PR merge failed
```

## 13. Final verification

Inspect:

```bash
git -C <dir> status --short --branch
git -C <dir> branch
```

Report separately:
- remote PR merge result;
- merge commit SHA;
- check state;
- review result;
- remote branch deletion state;
- local base update state;
- local branch deletion state;
- current local branch/worktree state.

Do not claim the local repository is clean/up-to-date if cleanup was intentionally skipped.

Keep the response compact.

Example:

```text
Merged #42: Feature: Add cache support
Merge commit: abc1234

Checks: passed
Review: no blocking findings
Remote branch: deleted
Local base: development updated
Local branch: deleted
```

## Safety

- One PR per run.
- Never bypass pending/failing required checks.
- Never bypass branch protection.
- Never merge a changed head SHA under stale review/approval.
- Never let `--auto-merge` bypass a safety gate.
- Never force-push, reset, stash, clean, rebase, or amend.
- Never use `git branch -D`.
- Never discard or move unrelated local work.
- Never edit code in response to review or CI findings.
- Never close a PR without merging it.
- Never create releases, tags, or version commits.
- Never skip `review-pr` unless the user supplied `--no-review`.
- Never post review comments directly; `review-pr --comment` owns that flow.
- If a Git/GitHub mutation is blocked by the environment, report the blocked operation rather than working around it.
