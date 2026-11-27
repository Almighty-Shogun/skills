---
name: merge-pr
description: >-
  Safely merge one GitHub pull request after checking repository state, PR
  eligibility, CI, review status, and a fresh code review. Supports PR numbers
  and URLs, preserves merge commits by default, supports legitimate non-default
  base branches, and cleans up the merged branch when appropriate. --auto-merge
  skips only the final confirmation; it never bypasses safety gates. Use only
  when the user invokes it by name.
argument-hint: "[<number-or-url>] [--no-review] [--comment] [--auto-merge]"
disable-model-invocation: true
---

# Merge PR

Merge one pull request per run, then clean up its branch when safe.

## Contract

Allowed: merge the selected pull request, delete its remote head branch when
appropriate, switch to an existing local base branch and fast-forward it, and
delete the local head branch with `git branch -d`. Never edit code, fix review
or CI findings, post replies (`respond-pr` owns them) or a findings comment
(`review-pr --comment`), create commits, rebase, reset, stash, amend,
force-push, squash or rebase-merge unless the user chooses it in step 7, close
without merging, or create releases, tags, or version commits. Blockers that
need code changes go to `respond-pr`, `fix-pr`, or `delegate-task`, then back
through `review-pr`. If the environment blocks a Git or GitHub mutation, report
it rather than working around it.

## Invocation

```text
merge-pr [<number> | <PR URL>] [--no-review] [--comment] [--auto-merge]
```

- No PR: list open pull requests and ask which. Unlike the other PR skills,
  never default to the current branch's pull request, since a merge cannot be
  undone. With none open, say so and stop; with more than four, print them as a
  numbered list and ask for the number.
- `--no-review` skips the fresh `review-pr` pass, visibly.
- `--comment` is forwarded to `review-pr`, which still asks before posting.
  Reject it with `--no-review`, which leaves nothing to post.
- `--auto-merge` skips only the final merge confirmation, once every gate has
  passed. It never means "ignore unsafe state".

Read `AGENTS.md` and `CLAUDE.md`; they may set the base branch convention, merge
method, branch deletion policy, whether review may be skipped, and extra gates.
Ask when they conflict materially.

## 1. Resolve the repository and PR

Derive `<dir>`, `<origin-slug>`, and `<slug>` as `review-pr` does:
`<origin-slug>` is `owner/name` from the `origin` URL, HTTPS or SSH, without
`.git`. With a URL, `<slug>` is the URL's `owner/name`; otherwise, when
`origin` is a fork (`gh repo view <origin-slug> --json isFork,parent`), a
number resolves in the fork's parent, so `<slug>` is the parent's
`owner/name`; else `<slug>` is `<origin-slug>`. A pull request inside the fork
itself needs its URL. Compare slugs case-insensitively. Use `git -C <dir>` and
`--repo <slug>` throughout; never guess from an unrelated current repository.
Without a PR:

```bash
gh pr list --repo <slug> --state open --limit 100 \
  --json number,title,isDraft,headRefName,baseRefName,url
```

Read the chosen one:

```bash
gh pr view <number> --repo <slug> --json \
  number,title,state,isDraft,mergeable,mergeStateStatus,reviewDecision,author,\
headRefName,headRepositoryOwner,headRepository,isCrossRepository,baseRefName,\
url,headRefOid,statusCheckRollup,comments,reviews
```

`<headRefOid>`, `<base-branch>` (`baseRefName`), and `<head-branch>`
(`headRefName`) below come from this read. The merge target is `<base-branch>`,
which need not be the default branch.

Local steps (2, 6, 11, and the remote branch deletion in 10) need `<dir>` to be
a clone whose normalized `<origin-slug>` is `<slug>`. When it is not, as for a
URL from another repository or a number resolved in a fork's parent, skip them
and report each as skipped.

## 2. Local working-copy safety

Do not switch branches yet. Run `git status --short`, `git branch
--show-current`, `git fetch origin`, and `git fetch origin pull/<number>/head`;
the last makes `<headRefOid>` local even for a fork pull request, whose commits
no `origin` branch carries. If `git -C <dir> rev-parse FETCH_HEAD` is not
`<headRefOid>`, the head moved since step 1: read the pull request again and
repeat the fetch.

A dirty tree does not block the remote merge, since the merge does not touch
the local branch, but work out whether it blocks the post-merge switch: any
tracked modification, staged or not, does, since the switch would carry it onto
the base; an untracked file does only when its path exists on the base. Never
reset, stash, clean, discard, or carry user modifications onto another branch
because Git happens to allow it.

A local `<head-branch>` is the pull request's only when `git -C <dir>
merge-base --is-ancestor <head-branch> <headRefOid>` passes and, for a
cross-repository pull request, it tracks the fork: `git -C <dir> config
branch.<head-branch>.merge` is `refs/pull/<number>/head`, or
`branch.<head-branch>.remote` is the head repository's URL or a remote with
that URL. When a local `<head-branch>` fails the ancestor test, report the
commits `<headRefOid>` lacks (`git -C <dir> log --oneline
<headRefOid>..<head-branch>`) before proposing.

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
before proceeding; `BEHIND` is a hard blocker, since `gh pr merge` refuses a
head that is behind its base, and is never auto-updated. GitHub computes
mergeability lazily and often returns `UNKNOWN` right after a push: wait a few
seconds and read again once; a second `UNKNOWN` is never assumed safe.
Repository-specific semantics may override these. An empty check rollup means
**no checks ran**, not that they passed; show `Checks: none ran` in the
proposal and the report. Never weaken or bypass branch protection.

The rollup does not mark required checks; read them with
`gh pr checks <number> --repo <slug> --required --json name,state,bucket`.
Judge each by its `bucket`, not the exit code, which counts a cancelled check
as neither failed nor pending: `fail`, `cancel`, and `pending` are hard
blockers. Exit 1 with `no required checks reported` means no reported check is
marked required, not that the base requires none; a required check that never
reported still shows in `mergeStateStatus`. Exit 1 with `no checks reported`
means nothing ran; treat it like the empty rollup. Exit 1 from an API error is
a command failure, not data. A failing or pending non-required check is not a
hard blocker, but name it in the proposal and the report (`Checks: 1 failing,
not required: coverage`); `--auto-merge` does not skip approval while one
fails.

## 4. Review feedback

`comments` and `reviews` carry no thread state, so read review threads through
GraphQL (`<owner>` and `<name>` from `<slug>`):

```bash
gh api graphql --paginate -f owner=<owner> -f name=<name> -F number=<number> \
  -f query='
  query($owner: String!, $name: String!, $number: Int!, $endCursor: String) {
    repository(owner: $owner, name: $name) { pullRequest(number: $number) {
      reviewThreads(first: 100, after: $endCursor) {
        pageInfo { hasNextPage endCursor }
        nodes { isResolved path line
          comments(first: 50) { nodes { author { login } body } } } }
    } } }'
```

Show only feedback still relevant to merging: unresolved threads
(`isResolved: false`) with actionable requests or questions, change requests,
and material concerns. Leave out resolved threads, approvals, bot chatter, CI
notifications, and conversation. Acting on feedback belongs to `respond-pr`.

## 5. Fresh review

Unless `--no-review` was given, run `review-pr` from `<dir>` against this pull
request and its actual base, passing its URL rather than a bare number so
`review-pr` resolves the same repository, and forwarding `--comment` only when
supplied. If the harness cannot invoke one skill from another, read
`../review-pr/SKILL.md` and follow it inline; if neither works, stop and say
so, since only `--no-review` merges without it. Proceed only when it reports
**no blocking findings**, or when its only blocking findings are non-required
checks that step 3 already named, which the step 7 approval covers; otherwise
stop and report them. With `--no-review`, show `review: skipped by --no-review`
so an unreviewed merge never looks reviewed.

## 6. Base relationship

```bash
git -C <dir> merge-base origin/<base-branch> <headRefOid>
git -C <dir> rev-list --count origin/<base-branch>..<headRefOid>
git -C <dir> rev-list --count <headRefOid>..origin/<base-branch>
```

Report unusual ancestry. Never rebase, merge the base into the head, or update
the pull request branch.

## 7. Propose the merge

Read the allowed merge methods, the base branch's rules, and whether the base
uses a merge queue:

```bash
gh repo view <slug> --json \
  mergeCommitAllowed,squashMergeAllowed,rebaseMergeAllowed,deleteBranchOnMerge
gh api repos/<slug>/rules/branches/<base-branch>
gh api graphql -f owner=<owner> -f name=<name> -F number=<number> -f query='
  query($owner: String!, $name: String!, $number: Int!) {
    repository(owner: $owner, name: $name) {
      pullRequest(number: $number) { isMergeQueueEnabled } } }'
```

The method is a merge commit, which preserves the branch commits. If
`mergeCommitAllowed` is false, a `required_linear_history` rule applies, a
`pull_request` rule's `allowed_merge_methods` omits `merge`, or instructions
set another method, stop and ask which allowed method to use; never pick
`--squash` or `--rebase` yourself. When `isMergeQueueEnabled` is true, step 9
adds the pull request to the base's merge queue instead of merging it: show
`Method: merge queue`.

Delete the remote head branch only for a same-repository branch
(`isCrossRepository` false) whose `author.login` is the user (`gh api user
--jq .login`), that is not protected (`gh api
repos/<slug>/branches/<head-branch> --jq .protected` is false), and that is
neither long-lived nor kept by the instructions. Keep fork branches, protected
or long-lived branches, and uncertain ownership, and say why. When
`deleteBranchOnMerge` is true, GitHub may delete the head at merge time
whatever this skill decides; warn when that hits a branch to keep.

```text
Merging #42 <title>

Base: development
Head: feat/cache
Method: merge commit
Mergeable: yes
Checks: passed
Review: no blocking findings
After merge: delete remote feat/cache (yours, unprotected)
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
branch and switches branches.

## 10. Verify the merge and delete the remote branch

```bash
gh pr view <number> --repo <slug> --json \
  state,mergedAt,mergeCommit,url,headRefName,baseRefName
```

Require `state = MERGED` and record the merge commit. A quiet command is not
proof of success. With a merge queue, `state` stays `OPEN` after a successful
step 9: report `Queued #42 in the merge queue`, distinct from both merged and
failed, and skip the remote deletion and step 11, since nothing merged yet.

When step 7 allows deleting the remote head branch, skip it when `gh pr list
--repo <slug> --head <head-branch> --state open` lists another pull request,
which the deletion would close. Otherwise check it, then delete it:

```bash
git -C <dir> ls-remote --exit-code origin refs/heads/<head-branch>
git -C <dir> push origin \
  --force-with-lease=refs/heads/<head-branch>:<headRefOid> \
  --delete refs/heads/<head-branch>
```

Exit 2 means it is gone, as GitHub's auto-delete may already have removed it;
any other non-zero exit is an error to report, not a deletion. When the SHA it
prints is not `<headRefOid>`, a commit was pushed after the merge: keep the
branch and say so. The lease only guards the deletion against the same race.

## 11. Local cleanup

Run these only after the remote merge succeeded, each per the rules below it:

```bash
git -C <dir> switch --no-guess <base-branch>
git -C <dir> pull --ff-only origin <base-branch>
git -C <dir> branch -d <head-branch>
```

Switch only when the checked-out branch is the pull request's (step 2), the
tree does not block the switch, and a local `<base-branch>` exists; skip the
switch when the base already is checked out. With no local base, keep the head
branch checked out and say so; `--no-guess` keeps Git from creating one. On any
other branch, stay there: update an existing local base with
`git -C <dir> fetch origin <base-branch>:<base-branch>`, which refuses a
non-fast-forward, and keep the local head branch, since `-d` would test it
against its upstream or HEAD rather than the base.

Delete the local head branch only when it exists, it is the pull request's
(step 2), the base is checked out, and it is not checked out in another
worktree; otherwise keep it and say why. After a user-chosen squash or rebase
merge, delete it only when `git -C <dir> merge-base --is-ancestor <head-branch>
HEAD` passes, HEAD being the updated base; otherwise keep it and report it.
Use `--ff-only` unless the repository requires otherwise, and never create a
local merge commit to update the base. Use `-d`, never `-D`. If any cleanup
command fails, stop cleanup and report it. When cleanup is skipped or
incomplete, keep "PR merged successfully, local cleanup incomplete" distinct
from "PR merge failed", and name the paths or reason.

## 12. Report

Check `git status --short --branch` and `git branch`, then report each part
separately. When step 1 skipped the local steps, report the local lines as
skipped instead, since this checkout is not the pull request's repository:

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
