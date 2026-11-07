---
name: plan-implementation
description: Produce an implementation-ready, repository-grounded plan for an already-understood change without modifying code. Reuses relevant design/research artifacts, identifies exact files and symbols, sequences changes, and optionally persists the plan with --save.
argument-hint: "[--save] <change to plan>"
---

# Implementation Plan

Decide how a chosen feature, fix, refactor, or migration should be implemented,
before any code changes. Without `--save`, the plan is returned in chat; with
it, it is also written to `.plan-implementation/<descriptive-topic>.md`.

This assumes the direction is chosen. An unresolved public contract (request,
response, error, or retry semantics) is designed in `api-design` first;
understanding current behavior is `repository-research`; external facts are
`source-research`; discovering structural problems is `structure-review`.

## Investigation

Reduce the request to a Task Contract: Goal, Scope, Constraints, Known
requirements, Acceptance criteria when supplied, and Non-goals when they prevent
scope creep. For a plan that needs real exploration, hand that contract to a
fresh planning agent with repository access, so its exploration stays out of the
parent context; a small, well-located change can be planned directly.

The planner inspects the relevant code, tests, configuration, and conventions;
picks the smallest correct implementation that reuses existing patterns;
identifies the exact files and symbols to change and their order where it
matters; settles details the conventions already decide; and accounts for
regressions, security, compatibility, performance, dependencies, migrations,
and data effects when relevant. For a bug fix, it establishes the failure or
root cause first, from a failing test, a minimal reproduction, or a verified
`bughunt` report. It stops exploring once the plan is actionable.

Reuse an artifact the user references or that clearly belongs to the change
(`.api-design/`, `.research/`, `.repository-research/`, `BUG_REPORT.md`,
`.bughunt/`, `STRUCTURE_REVIEW.md`), or a trustworthy handoff already in the
conversation, instead of repeating its analysis; still verify that its paths,
symbols, and conclusions are current.

## Ambiguity

Settle minor choices from conventions and never ask the user to pick between
equivalent low-level details. When approaches differ materially in API,
architecture, data, security, compatibility, or maintenance, present the
smallest useful set of choices and the trade-off.

## Output

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
- <only decisions that need user input>
```

Drop sections a small task does not need. The plan is specific enough that a
fresh implementation agent can execute it without repeating broad research:
exact paths and symbols (a line number such as `src/Foo.cs line 42 —
FooService.ExecuteAsync` is a navigation hint; path and symbol must suffice if
lines move), concrete behavior changes, the abstractions to reuse, acceptance
criteria, and narrow validation commands. No full implementations, large
excerpts, investigation history, speculative refactors, or vague steps such as
"update the service"; pseudocode only to disambiguate behavior.

## Persistence and boundaries

With `--save`, update an existing plan for the same topic rather than
duplicating it, and never overwrite an unrelated one; persist the plan, not the
investigation. That file is the only write: leave source, tests, configuration,
and Git state alone. The plan is for review or later execution, never
implemented automatically.
