---
name: respond-pr
description: >-
  Process reviewer feedback on a pull request. Resolve the target PR, surface
  only unresolved actionable feedback, let the user choose what to address,
  delegate independent changes to isolated sub-agents, validate the resulting
  diff, and require approval before posting replies or resolving GitHub
  threads.
argument-hint: "[<number-or-url>]"
---

# Respond PR

Process review feedback that already exists on one pull request. Discovering new
issues belongs to `review-pr`, which this skill does not require as a
prerequisite; CI failures belong to `fix-pr`.

## Contract

Selected feedback may change the working tree without further approval. Every
GitHub mutation (a thread reply, a PR-level reply, resolving a thread) needs
explicit approval of the exact text. Commits and pushes go through the `commit`
skill; this skill never commits, pushes, amends, rebases, or creates or deletes
branches.
Protect unrelated uncommitted work.

## Resolve the PR

Use the PR URL when given, else the number in the current repository, else the
pull request for the current branch. If none resolves, or several plausibly
could, ask; never guess. Follow the repository and PR resolution conventions of
`review-pr`.

Check the head first, before any edit and before any local reproduction or
verification:

```bash
gh pr view <number> --repo <slug> \
    --json headRefName,headRefOid,headRepository,headRepositoryOwner
```

Require that the current branch is `headRefName`, `HEAD` is `headRefOid`, and
`origin` is the head repository. A fork pull request passes only when `origin`
is that fork, as when its author runs this. Otherwise stop and ask before any
edit; never run `gh pr checkout` automatically. Reading and verifying feedback
may still proceed read-only, but from CI evidence and `gh pr diff`, never from
local runs on a checkout that is not the PR head.

## Collect and group feedback

Collect unresolved review threads, actionable review comments, and PR-level
comments that clearly request a change. Skip resolved threads, bare approvals,
bot chatter, CI notifications, and conversation, reading them only to
understand an unresolved item.

No `gh` subcommand exposes thread state, so read threads through GraphQL
(`<owner>` and `<name>` from `<slug>`); PR-level comments and review bodies
come from `gh pr view <number> --repo <slug> --json comments,reviews`:

```bash
gh api graphql --paginate -f owner=<owner> -f name=<name> -F number=<number> \
  -f query='
  query($owner: String!, $name: String!, $number: Int!, $endCursor: String) {
    repository(owner: $owner, name: $name) { pullRequest(number: $number) {
      reviewThreads(first: 100, after: $endCursor) {
        pageInfo { hasNextPage endCursor }
        nodes { id isResolved path line
          comments(first: 50) { nodes { databaseId author { login } body } } } }
    } } }'
```

Comments past the first 50 in a thread are not needed to reply to or resolve
it; the thread `id` and its opening comments carry the request.

Classify each item as actionable, suggestion, question, incorrect or
incompatible, or already addressed. Group comments that ask for the same
underlying change, keeping each original thread reference so replies and
resolutions map back correctly.

## Let the user choose

Show a numbered list before touching code:

```text
1. Reuse the existing date parser
2. Remove the duplicated null check
3. Simplify authentication error handling (3 related threads)
4. Add coverage for expired tokens
```

Accept natural selections such as `all`, `1, 2 and 4`, or `everything except 3`.
Unselected feedback gets no code changes.

## Verify before applying

Never assume a reviewer is right. Check each selected request against the
PR's code (the local checkout only when the head check passed), tests,
repository rules, public contracts, and compatibility requirements. A request
that is technically wrong or incompatible is not applied; explain why instead:

```text
Not applied:
The requested null-check removal would violate the public contract, which
explicitly permits null here and is covered by an existing test.
```

Questions get a reply, not code; if answering one reveals a real problem,
surface it separately. Already-addressed requests get no redundant edits, only
a reply when useful. When reviewers ask for incompatible behavior, surface the
conflict and ask which direction to follow. When a request would materially
widen the pull request beyond its purpose, say so and ask before doing it.

## Implement and validate

Give each independent selected item its own implementation sub-agent, and one
sub-agent per group of related comments, with only the selected request, its
thread context, the surrounding code, and the repository constraints. Changes
stay within the selected scope, follow conventions, avoid unrelated refactors,
and keep public contracts unless the request explicitly changes them. The
sub-agents share the working tree, so run them one after another unless their
file sets are disjoint. Start each sub-agent with an explicit model and effort
level that fits its task: highest for implementation, fix, and validation,
lower for simple read-only research.

```text
Status: Applied | Not applied | Already addressed
Files changed: <files>
What changed: <summary>
Checks: <local validation>
Reason not applied: <only when applicable>
```

Run the relevant local checks. Then a fresh validation sub-agent compares the
original request with the actual diff, checks tests and regressions, and judges
whether the request was genuinely addressed, without relying on the
implementer's account. A concrete problem gets one targeted fix and another
validation; avoid open-ended repair loops.

## Replies and resolution

Draft a reply for each handled item and show the exact text:

```text
Thread 3

Proposed reply:
Updated this to reuse the existing parser and added coverage for invalid input.
```

Replies say what changed or why a request was not applied, briefly and
directly, without implementation diaries, boilerplate, attribution, or remarks
about the change being focused or minimal. Not this:

```text
I have now carefully implemented the requested change by making several focused
updates across the codebase. The following modifications were performed...
```

Post exactly the approved text and never regenerate it afterwards unless the
user asks for an edit. Replies that claim no change, such as answers and
declined requests, do not wait. A reply that claims a change ("Updated
this...") waits until the change is pushed and the PR head contains it:

1. Offer the `commit` handoff with the changed files as its paths (plus
   `--dir <dir>` outside the current repository); it owns grouping, commits,
   and the push under its own confirmations.
2. Re-read `headRefOid` with `gh pr view`, run `git -C <dir> fetch origin` so
   later pushes by others are present, then check
   `git -C <dir> merge-base --is-ancestor HEAD <headRefOid>`. `HEAD` is the
   last commit `commit` created, so one check covers several commits. Post only
   when it passes.
3. If the user declines the commit or the push, or the check fails, end the run
   and report each waiting reply in full, its thread and exact text. Approval
   does not carry into a later run, which shows the text for approval again.

Post a thread reply with the GraphQL `addPullRequestReviewThreadReply` mutation
(`pullRequestReviewThreadId` is the thread `id`), or with
`gh api repos/<slug>/pulls/<number>/comments/<comment-id>/replies -F
body=@<body-file>` (`<comment-id>` is the first comment's `databaseId`). Post a
PR-level reply with `gh pr comment <number> --repo <slug> --body-file
<body-file>`. Keep `<body-file>` in the scratchpad or temp directory.

Resolving a thread is a separate approval: offer it once a request has been
addressed or validly declined, validated where code changed, and replied to
where appropriate. A thread resolved by a change also waits until the same
check passes. Resolve with the GraphQL `resolveReviewThread`
mutation (`threadId` is the thread `id`). Never mark feedback resolved silently.

## Final summary

```text
Addressed: 3
Already addressed: 1
Not applied: 1
Replies posted: 3
Threads resolved: 2
Waiting on push: 1
```

Also name any unresolved conflict, out-of-scope request awaiting a decision, or
validation failure that remains, and list each reply waiting on the push with
its thread and exact text.
