---
name: plan-implementation
description: >-
  Produce an implementation-ready, repository-grounded plan for a chosen
  change, bug fixes included, without lasting code changes. Reuses relevant
  design/research artifacts, identifies exact files and symbols, sequences
  changes, and optionally persists the plan with --save. Not for designing an
  unresolved public contract (api-design) or implementing the change
  (delegate-task).
argument-hint: "[--save] <change to plan>"
---

# Plan Implementation

Decide how a chosen feature, fix, refactor, or migration should be implemented,
before any code changes. Without `--save`, the plan is returned in chat; with
it, it is also written to `.plan-implementation/<descriptive-topic>.md` under
the repository root, and the reply ends with that path.

Read every `AGENTS.md` and `CLAUDE.md` that applies and follow them; they
override this skill's defaults, and when two conflict, ask which wins. Give each
sub-agent the rules it needs.

This assumes the direction is chosen. An unresolved public contract (request,
response, error, or retry semantics) is designed in `api-design` first. An
`.api-design/` artifact counts as approved only when it says
`Status: Approved`; a `Draft` goes back to `api-design`, where the user answers
or waives its open questions. Understanding current behavior is
`repository-research`; external facts are `source-research`; discovering
structural problems is `structure-review`. `api-design` and `structure-review`
are only named to the user, never invoked from here; work routed to
`source-research` always writes its `.research/<topic>.md`.

## Investigation

Reduce the request to a Task Contract: Goal, Scope, Constraints, Known
requirements, Acceptance criteria when supplied, and Non-goals when they prevent
scope creep. For a plan that needs real exploration, hand that contract to a
fresh planning agent with repository access, so its exploration stays out of the
parent context; a small, well-located change can be planned directly. Start the
agent with an explicit model and effort level that fits its task: lower for a
simple read-only plan, higher for one that settles design or a root cause.

The planner inspects the relevant code, tests, configuration, and conventions;
picks the smallest correct implementation that reuses existing patterns;
identifies the exact files and symbols to change and their order where it
matters; settles details the conventions already decide; and accounts for
regressions, security, compatibility, performance, dependencies, migrations,
and data effects when relevant. For a bug fix, it establishes the failure or
root cause first, from a failing test, a minimal reproduction, or a verified
`bughunt` report; when it cannot, it says so, plans no fix, and lists the
missing evidence as an Open decision. It stops exploring once the plan is
actionable.

Reuse an artifact the user references or that clearly belongs to the change
(`.api-design/`, `.research/`, `.repository-research/`, `.plan-implementation/`,
`BUG_REPORT.md`, `.bughunt/`, `STRUCTURE_REVIEW.md` or
`STRUCTURE_REVIEW-<scope-slug>.md`, `SIMPLIFICATION_REPORT.md`; look for the
structure reviews and `.api-design/` at the repository root), or a trustworthy
handoff already in the conversation, instead of repeating its analysis. Verify
that the paths, symbols, and conclusions of findings are current, and check a
design for conflict with the code, not for symbols it has yet to create. Name a
stale or conflicting artifact and re-derive that part or
raise an Open decision. In a `bughunt` report, an answered `Desired solution:`
block is the chosen design; an unanswered one becomes an Open decision. A run
without `--save` that supersedes a saved plan says so.

## Ambiguity

Settle minor choices from conventions and never ask the user to pick between
equivalent low-level details. When approaches differ materially in
architecture, data, security, compatibility, or maintenance, present the
smallest useful set of choices and the trade-off; a choice between public
contracts goes to `api-design`.

## Output

```text
Implementation Plan

Goal
- ...

Root cause
- <bug fixes only: the established cause and its evidence>

Changes
1. <file/symbol> — <specific change and why>
2. ...

Tests / validation
- ...

Acceptance criteria
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
investigation. A saved plan with Open decisions carries `Status: Draft` under
its title. The plan is a working note, committed only when the user asks. That
file and any `.research/<topic>.md` from `source-research` are the only lasting
writes: leave source, tests, configuration, and Git state alone. To reproduce a
failure, prefer commands that do not rewrite tracked files (`npm ci`, no
snapshot updates), and revert any tracked-file change the run caused. The agent
that creates temporary instrumentation or a harness removes it before handing
off and reports anything left; remove only what this run created. The plan is
for review or later execution, never implemented
automatically.
