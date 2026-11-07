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
commits, rebase, reset, stash, amend, force-push, squash, close without merging,
or create releases, tags, or version commits. Blockers that need code changes
go to `review-pr`, `respond-pr`, `fix-pr`, or `delegate-task`. If the
environment blocks a Git or GitHub mutation, report it rather than working
around it.

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
headRefName,headRepositoryOwner,headRepository,baseRefName,url,headRefOid,\
statusCheckRollup,comments,reviews
```

The merge target is the pull request's `baseRefName`, which need not be the
default branch.

## 2. Local working-copy safety

Do not switch branches yet. Run `git status --short`, `git branch
--show-current`, and `git fetch origin`. A dirty tree does not block the remote
merge, since the merge does not touch the local branch, but work out whether the
post-merge switch and cleanup would collide with local work: tracked
modifications against the target base, and untracked files whose paths exist on
it. Never reset, stash, clean, discard, or carry user modifications onto
another branch because Git happens to allow it.

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

```text
Merging #42 <title>

Base: development
Head: feat/cache
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

Use the repository's configured merge method; otherwise a merge commit, which
preserves the branch commits:

```bash
gh pr merge <number> --repo <slug> --merge
```

Never `--squash`, `--rebase`, or force options. Delete the remote head branch
only for an ordinary same-repository disposable branch the user owns that is
neither protected nor required to be kept. Leave fork branches, protected or
long-lived branches, and uncertain ownership intact, and say why.

## 10. Verify the merge

```bash
gh pr view <number> --repo <slug> --json \
  state,mergedAt,mergeCommit,url,headRefName,baseRefName
```

Require `state = MERGED` and record the merge commit. A quiet command is not
proof of success.

## 11. Local cleanup

Only after the remote merge succeeded, and only when switching cannot collide
with local work:

```bash
git -C <dir> switch <base-branch>
git -C <dir> pull --ff-only origin <base-branch>
git -C <dir> branch -d <head-branch>
```

Delete the local head branch only when it exists and is not checked out in
another worktree. Use `--ff-only` unless the repository requires otherwise, and
never create a local merge commit to update the base. Use `-d`, never `-D`; if it refuses,
stop cleanup and report it. When cleanup is skipped or incomplete, keep
"PR merged successfully, local cleanup incomplete" distinct from "PR merge
failed", and name the paths or reason.

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
