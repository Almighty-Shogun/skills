---
name: plan-implementation
description: Produce an implementation-ready, repository-grounded plan for an already-understood change without modifying code. Reuses relevant design/research artifacts, identifies exact files and symbols, sequences changes, and optionally persists the plan with --save.
argument-hint: "[--save] <change to plan>"
disable-model-invocation: true
---

# Implementation Plan

Use when the user wants to decide how a feature, fix, refactor, migration, or architectural change should be implemented before touching code.

This skill is plan-only. Do not modify source, tests, configuration, Git state, or implementation files.

Use `--save` when the user wants the plan persisted for later execution. Without `--save`, return the plan in chat only.

## Core rule

Delegate repository investigation and planning to a fresh planning agent. Keep exploratory context inside that agent and return only the implementation-ready plan.

## Task Contract

Before delegation, reduce the request to:

- Goal
- Scope
- Constraints
- Known requirements
- Non-goals, only when they materially prevent scope creep
- Acceptance criteria, if supplied

Do not forward unrelated conversation history.

## Planning agent

Give the agent only the Task Contract and repository access.

It should:

- inspect relevant existing code, tests, configuration, conventions, and abstractions
- determine the smallest correct implementation
- prefer existing project patterns over new abstractions
- identify exact files/symbols likely to change
- sequence changes in dependency/order-of-execution order when that matters
- use line numbers only as optional navigation hints; path + symbol must remain sufficient if lines move
- resolve implementation details that are inferable from repository conventions
- surface only decisions that genuinely require user choice
- identify important edge cases and validation needs
- account for correctness, regressions, security, compatibility, meaningful performance impact, dependency changes, migrations, or data effects when relevant
- not modify files

Stop exploring once the plan is actionable.

## Output

Return an implementation-ready plan, normally under ~700 tokens:

```text
Implementation Plan

Goal
- ...

Changes
1. <file/symbol> — <specific change and why>
2. ...

Tests / validation
- ...

Constraints / edge cases
- ...

Open decisions
- <only decisions that materially require user input>
```

Use fewer sections when the task is small. Correctness is more important than the token ceiling.

## Plan quality

A good plan should be specific enough that a fresh implementation agent can execute it without repeating broad repository research.

Prefer:

- exact paths and symbols, optionally with a current line-number hint such as `src/Foo.cs line 42 — FooService.ExecuteAsync`
- concrete behavior changes
- existing abstractions to reuse
- explicit acceptance criteria
- narrow validation commands or test areas

Avoid:

- pseudocode unless needed to disambiguate behavior
- full code implementations
- large source excerpts
- investigation history
- discarded approaches unless the choice matters
- speculative refactors
- vague steps such as "update the service" without saying what changes

## Ambiguity

Resolve minor implementation choices from repository conventions.

If there are multiple materially different approaches with meaningful API, architecture, data, security, compatibility, or maintenance tradeoffs, present only the smallest useful choice set and explain the tradeoff concisely.

Do not ask the user to decide between equivalent low-level implementation details.

## Research and artifact reuse

Reuse relevant, user-selected or clearly applicable artifacts instead of repeating completed analysis.

Examples:

```text
.api-design/*.md
.research/*.md
BUG_REPORT.md
.bughunt/*.md
STRUCTURE_REVIEW.md
```

Do not blindly consume every report in the repository. Use an artifact when the user references it or when it clearly belongs to the requested change.

If the conversation already contains a concise, trustworthy handoff, use it rather than repeating the same investigation. Still inspect repository state directly when needed to verify paths, symbols, or whether prior conclusions remain current.

Never require the full research transcript.

For bug-fix planning, establish the failing behavior/root cause where practical before producing implementation steps. Prefer an existing failing test, minimal reproduction, or verified `bughunt` report. Do not build an elaborate plan around an unverified assumption.

## Boundaries with other skills

This skill assumes the desired outcome is already a chosen implementation/change.

- `api-design` decides what a public or cross-boundary contract should be. If material request/response/error/retry semantics are unresolved, design them first instead of inventing them here.
- `structure-review` discovers structural/architectural improvements. This skill may plan an architectural change that has already been chosen.
- `repository-research` explains how the current repository works.
- `source-research` establishes external/framework/specification facts.
- `plan-implementation` turns the chosen direction into exact executable repository steps.

The planning agent may perform the small amount of repository inspection needed to make the plan executable, but should not become a deep research workflow.

## Boundary with repository research

This skill answers primarily:

- What should we change?
- Where should we change it?
- In what order?
- How will we know it is correct?

Use `repository-research` when the user's primary goal is understanding current behavior rather than planning a change.

## Persistence

Without `--save`, return the finished plan in chat only.

With `--save`, persist it under:

```text
.plan-implementation/<descriptive-topic>.md
```

Before writing, check for an existing plan for the same topic. Update it when clearly appropriate and avoid overwriting unrelated plans.

The persisted artifact must contain the implementation-ready plan, not the investigation transcript.

## Repository safety

Planning is read-only except for the optional `.plan-implementation/` artifact created by `--save`.

Do not:
- modify source, tests, or configuration;
- stage or commit;
- create/delete branches;
- reset, stash, clean, rebase, or amend;
- overwrite unrelated user work.

## User-facing behavior

Return the plan for review or later execution. Do not implement it automatically.
