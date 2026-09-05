---
name: respond-pr
description: Process reviewer feedback on a pull request. Resolve the target PR, surface only unresolved actionable feedback, let the user choose what to address, delegate independent changes to isolated sub-agents, validate the resulting diff, and require approval before posting replies or resolving GitHub threads.
argument-hint: "[<number-or-url>]"
disable-model-invocation: true
---

# Respond PR

Use this skill to process review feedback that already exists on a pull request.

This skill is responsible for:
- locating the target PR;
- collecting unresolved reviewer feedback;
- filtering and grouping related comments;
- letting the user choose what to address;
- verifying whether reviewer requests are technically valid;
- applying selected local code changes;
- independently validating those changes;
- drafting concise replies;
- posting replies or resolving threads only after explicit approval.

It does not perform a general PR review. Use `review-pr` to discover new issues.

## Invocation

Supported forms:

```text
respond-pr
respond-pr <PR number>
respond-pr <PR URL>
```

Resolution order:

1. If the user provides a PR URL, use that PR.
2. If the user provides a PR number, use that PR in the current repository.
3. Otherwise resolve the PR associated with the current branch.
4. If no clear PR can be resolved, ask the user for a PR number or URL.

Do not guess between multiple possible PRs.

## Core workflow

```text
Resolve PR
   ↓
Fetch unresolved feedback
   ↓
Filter / classify / group
   ↓
Show concise numbered list
   ↓
User selects items
   ↓
Independent implementation sub-agents
   ↓
Fresh validation
   ↓
Draft replies
   ↓
User approval
   ↓
Post replies / resolve threads
```

## Feedback collection

Prefer:
- unresolved review threads;
- actionable review comments;
- PR-level feedback that clearly requests a change.

Ignore by default:
- resolved threads;
- approvals with no requested action;
- bot chatter;
- CI notifications;
- conversational comments with no actionable request.

Inspect resolved or non-actionable context only when needed to understand an unresolved item.

## Classification

Classify each relevant item internally as one of:

```text
Actionable
Suggestion
Question / clarification
Incorrect / incompatible
Already addressed
```

Classification determines the workflow:

- `Actionable` → candidate for code changes.
- `Suggestion` → candidate for code changes if the user selects it.
- `Question / clarification` → usually needs a reply, not code.
- `Incorrect / incompatible` → do not apply blindly; explain why.
- `Already addressed` → do not make redundant changes; a reply may still be appropriate.

## Group related feedback

Group comments that clearly request the same underlying change.

Example:

```text
3. Simplify authentication error handling
   Covers 3 related review threads.
```

Preserve the original thread/comment references internally so replies and thread resolution map back correctly.

Do not create separate tasks for duplicate symptoms of the same requested change.

## User selection

Always show a concise numbered list before modifying code.

Example:

```text
1. Reuse the existing date parser
2. Remove the duplicated null check
3. Simplify authentication error handling (3 related threads)
4. Add coverage for expired tokens
```

Allow natural selections such as:

```text
all
1, 2 and 4
everything except 3
```

Do not modify code for unselected feedback.

## Verify reviewer feedback

Never assume a reviewer is correct.

Before applying a selected request, verify it against:
- current code;
- relevant tests;
- repository rules and instructions;
- public contracts;
- compatibility requirements;
- relevant documentation where needed.

If the request is technically incorrect or incompatible, do not apply it.

Return a concise explanation instead.

Example:

```text
Not applied:
The requested null-check removal would violate the public contract, which
explicitly permits null here and is covered by an existing test.
```

Do not turn this verification into broad unrelated research.

## Questions and clarifications

Comments such as:

```text
Why is this needed?
Can you explain this behavior?
```

should not trigger code changes automatically.

Prepare a concise reply based on the current implementation and repository context.

If answering the question reveals a genuine code problem, surface that separately instead of silently changing code.

## Sub-agent strategy

Use isolated sub-agents for independent selected feedback.

Example:

```text
Item 1 → sub-agent A
Item 2 → sub-agent B
Item 4 → sub-agent C
```

Related comments that belong to one underlying change should be handled by one sub-agent.

Each implementation sub-agent receives only:
- the exact selected review request;
- relevant thread context;
- necessary surrounding code;
- repository constraints required for the change.

Do not give every sub-agent the entire PR discussion.

Preferred implementation handoff:

```text
Status:
Applied | Not applied | Already addressed

Files changed:
<only relevant files>

What changed:
<concise summary>

Checks:
<relevant local validation>

Reason not applied:
<only when applicable>
```

Do not return:
- investigation transcripts;
- unrelated PR context;
- lengthy implementation walkthroughs;
- speculative refactors.

## Local code changes

Once the user selects feedback items, local code changes do not require another approval step.

Changes must:
- stay within the selected feedback scope;
- follow repository conventions;
- avoid unrelated refactors;
- preserve public contracts unless the selected request explicitly requires changing them;
- protect unrelated user work.

Never:
- reset unrelated changes;
- stash unrelated changes;
- clean the repository destructively;
- checkout over user work;
- overwrite unrelated uncommitted files.

Do not commit or push automatically.

Do not create/delete branches, amend, rebase, or perform other Git-history mutations.

Use the existing `commit` skill when the user wants commits created or pushed.

## Independent validation

After selected code changes are applied, use a fresh validation sub-agent.

The validator should inspect:
- the original reviewer request;
- the actual resulting diff;
- relevant tests/checks;
- regressions;
- whether the requested change was genuinely addressed.

Do not rely on the implementation agent's explanation.

If validation finds a concrete problem:
- fix only that problem using an isolated implementation sub-agent;
- validate again.

Avoid open-ended repair loops.

## Already-addressed feedback

If the current branch already satisfies a review request:
- classify it as `Already addressed`;
- do not make redundant edits;
- prepare a concise reply when useful.

## Conflicting reviewer feedback

If reviewer requests conflict, do not choose silently.

Surface the conflict concisely and ask the user which direction to follow.

Example:

```text
Reviewers requested incompatible behaviors for the same API.
Choose which direction to follow before I modify it.
```

## Out-of-scope requests

If a reviewer asks for work that materially expands the PR beyond its current purpose, identify that clearly.

Example:

```text
This request appears to expand the PR beyond its current scope.
```

Ask the user whether they want to include that work before modifying code for it.

Do not silently broaden the PR.

## CI and local checks

Run relevant local checks after changes.

Examples:
- targeted tests;
- build;
- lint;
- type-check;
- formatter validation where appropriate.

Do not turn this skill into full CI debugging.

If GitHub CI fails for a deeper or unrelated reason, surface that and use `fix-pr` instead.

## Reply writing style

Use the same concise, human writing style as the repository's PR skills.

Replies should:
- explain what changed or why a request was not applied;
- be short and direct;
- avoid implementation diaries;
- avoid unnecessary headings;
- avoid boilerplate;
- avoid generated-by/tool attribution;
- avoid narrating that the change is focused, scoped, minimal, or clean.

Good:

```text
Updated this to reuse the existing parser and added coverage for invalid input.
```

Bad:

```text
I have now carefully implemented the requested change by making several focused
updates across the codebase. The following modifications were performed...
```

## Draft replies

After validation, prepare concise replies for handled feedback.

For each reply, show the exact text that would be posted.

Example:

```text
Thread 3

Proposed reply:
Updated this to reuse the existing parser and added coverage for invalid input.
```

Do not post replies automatically.

## GitHub write approval

Every GitHub mutation requires explicit user approval.

This includes:
- posting review-thread replies;
- posting PR-level replies;
- resolving review threads.

Local code changes do not require additional approval after the user selected the feedback items.

Do not treat approval to post a reply as approval to resolve the thread unless the user explicitly approves both.

## Resolving threads

Thread resolution is separate from replying.

After a selected request has been:
- addressed or validly declined;
- independently validated where code changed;
- replied to when appropriate;

offer to resolve the relevant thread.

Require explicit approval before resolution.

Never silently mark feedback resolved.

## Exact-post behavior

When asking for approval, show the exact reply text that will be submitted.

If the user approves it, post exactly that text unless they request an edit.

Do not regenerate approved text afterward.

## Final summary

Keep the final summary compact.

Useful information:

```text
Addressed: 3
Already addressed: 1
Not applied: 1
Replies posted: 3
Threads resolved: 2
```

Also mention:
- any unresolved conflict;
- any out-of-scope request awaiting a decision;
- any validation failure that remains unresolved.

Do not repeat full review comments or full implementation details.

## Repository and GitHub conventions

Follow the same repository-resolution, PR-resolution, writing, safety, and GitHub conventions used by the existing PR skills where applicable.

Do not make `respond-pr` invoke `review-pr` as a prerequisite. They are separate workflows:

```text
review-pr
→ independently inspect a PR
→ read-only

respond-pr
→ process existing reviewer feedback
→ may modify local code
```

## Token efficiency

- Fetch only feedback relevant to unresolved/actionable work.
- Group related comments before delegation.
- Give each sub-agent only candidate-specific context.
- Do not forward one sub-agent's full reasoning to another.
- Prefer actual repository state and diff inspection over prose summaries.
- Keep implementation handoffs compact.
- Keep validation independent.
- Do not repeat GitHub discussion history unless it is needed for context.
