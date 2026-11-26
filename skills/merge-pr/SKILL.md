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

Merge one pull request per run, then clean up its branch when safe.

## Contract

Allowed: merge the selected pull request, delete its remote head branch when
appropriate, fast-forward its base branch locally, and delete the local head
branch with `git branch -d`. Never edit code, fix review or CI findings, post
replies or review comments (`review-pr --comment` owns that flow), create
commits, rebase, reset, stash, amend, force-push, squash or rebase-merge unless
the user chooses it in step 7, close without merging, or create releases, tags,
or version commits. Blockers that need code changes go to `review-pr`,
`respond-pr`, `fix-pr`, or `delegate-task`. If the environment blocks a Git or
GitHub mutation, report it rather than working around it.

## Invocation

```text
merge-pr [<number> | <PR URL>] [--no-review] [--comment] [--auto-merge]
```

- No PR: list open pull requests and ask which.
- `--no-review` skips the fresh `review-pr` pass, visibly.
- `--comment` is forwarded to `review-pr`, which still asks before posting.
- `--auto-merge` skips only the final merge confirmation, once every gate has
  passed. It never means "ignore unsafe state".

Read `AGENTS.md` and `CLAUDE.md`; they may set the base branch convention, merge
method, branch deletion policy, whether review may be skipped, and extra gates.
Ask when they conflict materially.

## 1. Resolve the repository and PR

Derive `<dir>`, `<slug>`, and `<default-branch>`; use `git -C <dir>` and
`--repo <slug>` throughout. For a URL, resolve that repository and number;
never guess from an unrelated current repository. Without a PR:

```bash
gh pr list --repo <slug> --state open \
  --json number,title,isDraft,headRefName,baseRefName,url
```

Read the chosen one:

```bash
gh pr view <number> --repo <slug> --json \
  number,title,state,isDraft,mergeable,mergeStateStatus,reviewDecision,\
headRefName,headRepositoryOwner,headRepository,isCrossRepository,baseRefName,\
url,headRefOid,statusCheckRollup,comments,reviews
```

The merge target is the pull request's `baseRefName`, which need not be the
default branch.

Local steps (2, 6, 11, and the remote branch deletion in 10) need `<dir>` to be
a clone whose `origin` is `<slug>` (`git -C <dir> remote get-url origin`). For
a URL from another repository with no such clone, skip them and report each as
skipped.

## 2. Local working-copy safety

Do not switch branches yet. Run `git status --short`, `git branch
--show-current`, `git fetch origin`, and `git fetch origin pull/<number>/head`;
the last makes `<head-sha>` local even for a fork pull request, whose commits
no `origin` branch carries. A dirty tree does not block the remote merge, since
the merge does not touch the local branch, but work out whether the post-merge
switch and cleanup would collide with local work: tracked modifications against
the target base, and untracked files whose paths exist on it. Never reset,
stash, clean, discard, or carry user modifications onto another branch because
Git happens to allow it.

If local work blocks only cleanup, the merge may proceed once the user knows
cleanup will stay incomplete. If local state makes the merge itself ambiguous,
stop and name the paths.

## 3. Eligibility

Stop on any hard blocker:

- state is not `OPEN`, or the pull request is a draft;
- `mergeable` is `CONFLICTING`;
- the review decision is `CHANGES_REQUESTED`;
- branch protection or `mergeStateStatus` blocks it;
- a required check failed, timed out, was cancelled, or is still queued,
  running, pending, or expected;
- the state is too unknown to establish merge safety.

Interpret `mergeStateStatus`: `DIRTY` is a conflict; `BLOCKED` is a protection,
review, or check requirement to inspect; `UNSTABLE` needs its cause found
before proceeding; `BEHIND` is reported, never auto-updated; `UNKNOWN` is
never assumed safe. Repository-specific semantics may override these. An
empty check rollup means **no checks ran**, not that they passed. Never weaken
or bypass branch protection.

The rollup does not mark required checks; read them with
`gh pr checks <number> --repo <slug> --required`. A non-zero exit is data:
exit 8 means pending; exit 1 means a failure or, with the message
`no required checks reported`, that the base requires none. Exit 1 with
`no checks reported` means nothing ran; treat it like the empty rollup. Exit 1
from an API error is a command failure, not data. A failed non-required check
is not a hard blocker, but name it in the proposal; `--auto-merge` does not
skip approval while one fails, and `review-pr` may still block on it.

## 4. Review feedback

Show only feedback still relevant to merging: unresolved actionable requests or
questions, change requests, and material concerns. Leave out resolved threads,
approvals, bot chatter, CI notifications, and conversation. Acting on feedback
belongs to `respond-pr`.

## 5. Fresh review

Unless `--no-review` was given, run `review-pr` against this pull request and
its actual base, forwarding `--comment` only when supplied. If the harness
cannot invoke one skill from another, read `../review-pr/SKILL.md` and follow
it inline. Proceed only when it reports **no blocking findings**; otherwise
stop and report them. With `--no-review`, show `review: skipped by --no-review`
so an unreviewed merge never looks reviewed.

## 6. Base relationship

```bash
git -C <dir> merge-base origin/<base-branch> <head-sha>
git -C <dir> rev-list --count origin/<base-branch>..<head-sha>
git -C <dir> rev-list --count <head-sha>..origin/<base-branch>
```

Report unusual ancestry. Never rebase, merge the base into the head, or update
the pull request branch.

## 7. Propose the merge

Read the allowed merge methods:

```bash
gh repo view <slug> --json mergeCommitAllowed,squashMergeAllowed,rebaseMergeAllowed
```

The method is a merge commit, which preserves the branch commits. If
`mergeCommitAllowed` is false, or instructions set another method, stop and
ask which allowed method to use; never pick `--squash` or `--rebase` yourself.

```text
Merging #42 <title>

Base: development
Head: feat/cache
Method: merge commit
Mergeable: yes
Checks: passed
Review: no blocking findings
After merge: delete head branch when safe
```

Include any unresolved material feedback. Without `--auto-merge`, ask for
approval; with it, still show the plan.

## 8. Re-check immediately before merging

Approval goes stale. Fetch the pull request again and compare state, draft
status, head SHA (`headRefOid`), mergeability, `mergeStateStatus`, review
decision, required checks, and base branch. If the head SHA or any material
state changed since the review or approval, stop and show the new state.

## 9. Merge

Merge with the method from step 7; replace `--merge` only with the `--squash`
or `--rebase` the user chose there:

```bash
gh pr merge <number> --repo <slug> --merge \
  --match-head-commit <headRefOid>
```

`<headRefOid>` is the head SHA that was reviewed and re-checked in step 8, so
GitHub rejects the merge if anything was pushed since. If it does, stop and
show the new state; never drop `--match-head-commit` or retry automatically.

Never use force options or `--delete-branch`, which also deletes the local
branch and switches branches. Delete the remote head branch (step 10) only for
an ordinary same-repository disposable branch the user owns that is neither
protected nor required to be kept. Leave fork branches, protected or long-lived
branches, and uncertain ownership intact, and say why.

## 10. Verify the merge and delete the remote branch

```bash
gh pr view <number> --repo <slug> --json \
  state,mergedAt,mergeCommit,url,headRefName,baseRefName
```

Require `state = MERGED` and record the merge commit. A quiet command is not
proof of success.

When step 9 allows deleting the remote head branch, check that it still
exists, since GitHub's auto-delete may already have removed it (exit 2 means
it is gone), and only then delete it:

```bash
git -C <dir> ls-remote --exit-code origin refs/heads/<head-branch>
git -C <dir> push origin --delete refs/heads/<head-branch>
```

## 11. Local cleanup

Run these only after the remote merge succeeded and only when switching cannot
collide with local work, each per the rules below it:

```bash
git -C <dir> switch <base-branch>
git -C <dir> pull --ff-only origin <base-branch>
git -C <dir> branch -d <head-branch>
```

Switch only when the head branch is checked out; skip the switch when the base
already is. On any other branch, stay there: update an existing local base with
`git -C <dir> fetch origin <base-branch>:<base-branch>`, which refuses a
non-fast-forward, and keep the local head branch, since `-d` would test it
against its upstream or HEAD rather than the base.

Delete the local head branch only when it exists, the base is checked out, and
it is not checked out in another worktree. After a user-chosen squash or rebase
merge, delete it only when `git -C <dir> merge-base --is-ancestor <head-branch>
HEAD` passes, HEAD being the updated base; otherwise keep it and report it.
Use `--ff-only` unless the repository requires otherwise, and never create a
local merge commit to update the base. Use `-d`, never `-D`; if it or the
fetch refuses, stop cleanup and report it. When cleanup is skipped or
incomplete, keep "PR merged successfully, local cleanup incomplete" distinct
from "PR merge failed", and name the paths or reason.

## 12. Report

Check `git status --short --branch` and `git branch`, then report each part
separately:

```text
Merged #42: Feature: Add cache support
Merge commit: abc1234

Checks: passed
Review: no blocking findings
Remote branch: deleted
Local base: development updated
Local branch: deleted
Current: development, clean
```

Never claim the local repository is clean or current when cleanup was skipped.
