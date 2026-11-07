---
name: respond-pr
description: Process reviewer feedback on a pull request. Resolve the target PR, surface only unresolved actionable feedback, let the user choose what to address, delegate independent changes to isolated sub-agents, validate the resulting diff, and require approval before posting replies or resolving GitHub threads.
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

## Collect and group feedback

Collect unresolved review threads, actionable review comments, and PR-level
comments that clearly request a change. Skip resolved threads, bare approvals,
bot chatter, CI notifications, and conversation, reading them only to
understand an unresolved item.

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
current code, tests, repository rules, public contracts, and compatibility
requirements. A request that is technically wrong or incompatible is not
applied; explain why instead:

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
and keep public contracts unless the request explicitly changes them.

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
user asks for an edit. Resolving a thread is a separate approval: offer it once
a request has been addressed or validly declined, validated where code changed,
and replied to where appropriate. Never mark feedback resolved silently.

## Final summary

```text
Addressed: 3
Already addressed: 1
Not applied: 1
Replies posted: 3
Threads resolved: 2
```

Also name any unresolved conflict, out-of-scope request awaiting a decision, or
validation failure that remains.
