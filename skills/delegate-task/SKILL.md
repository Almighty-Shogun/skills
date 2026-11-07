---
name: delegate-task
description: Token-efficient development workflow using isolated research, implementation, and validation agents. Supports --interactive and --autonomous; otherwise infers the mode from context.
argument-hint: "[--interactive | --autonomous] <task>"
---

# Delegate Task

For non-trivial coding tasks where isolating investigation, implementation, and
review improves correctness or keeps the main context small.

Workflow: `Research -> Implement -> Validate`, with a fresh agent for each phase
that is needed. Carry conclusions forward, never transcripts.

## Modes

- `--interactive`: research, show the plan, wait for approval, then implement
  and validate.
- `--autonomous`: research, implement, and validate without pausing, unless a
  material decision needs the user.
- No flag: choose interactive when requirements are materially ambiguous,
  approaches carry real trade-offs, the work touches architecture, public API,
  schema or migrations, auth or security, destructive behavior, or broad
  repository changes, or the user wants to approve the approach. Choose
  autonomous when the expected behavior and acceptance criteria are clear and
  the change is a localized fix, narrow feature, or constrained refactor. When
  unsure, interactive for high-impact decisions, autonomous otherwise.

An autonomous run that uncovers a decision that could change behavior, API,
data, security, or scope pauses for the user; minor choices settled by project
conventions do not.

Agents have overhead. A typo, obvious rename, one-line fix, or mechanical change
with no design uncertainty is implemented directly with the relevant check run.
High-impact, security, and data-changing work always gets independent
validation.

## Task Contract

Distill the request into Goal, Scope, Constraints, Acceptance criteria, and
Non-goals when they prevent scope creep. Each agent gets this contract plus the
minimum phase handoff, and inspects files, tests, git state, and build output
itself. Prefer paths and symbols over pasted source. Handoffs are the shortest
text the next phase can act on without re-reading what the previous phase
read.

## Workflow boundary

This skill is for implementing a change. Proactive bug discovery is `bughunt`,
structural review is `structure-review`, contract design is `api-design`, and
substantial external research is `source-research`. When the user asks to
implement an artifact from one of those, treat it as input: confirm it still
applies, then do not repeat its analysis.

## Research

Skip it when the implementation is obvious, the user supplied an exact plan,
tests define the fix, or an approved artifact already settles the design. When
needed, a fresh read-only research agent inspects the relevant code, finds the
current behavior or root cause, and picks the smallest correct approach that
reuses existing conventions and abstractions. For a bug fix it reproduces or
otherwise establishes the failure first, preferring an existing failing test or
a minimal reproduction; it does not design a fix for a failure it cannot
establish, and states the limitation when CI evidence has to stand in.

```text
Research Handoff
- Current/root cause: ...
- Approach: ...
- Files/symbols: ...
- Constraints/edge cases: ...
- Acceptance criteria: ...
```

In interactive mode, show this handoff and wait. A small adjustment updates the
affected items without repeating research.

## Implementation

A fresh agent with the Task Contract, the approved handoff, and repository
access makes the smallest correct change: existing conventions, no unrelated
refactors or speculative improvements, only the files the task needs plus
directly related breakage, tests added or updated where needed, and the
narrowest useful checks run. It inspects the working tree before editing and
changes it only: no commits, pushes, branches, or history edits, and unrelated
uncommitted work is preserved.

```text
Implementation Handoff
- Changed: <files/symbols>
- Key decisions: <only non-obvious ones>
- Checks: <what ran + result>
- Unresolved: <only if applicable>
```

The validator does not receive this handoff unless it needs a fact the
repository cannot provide.

## Validation

A fresh agent with the Task Contract, the acceptance criteria, and direct access
to the code and diff, without the implementer's reasoning or self-assessment,
checks correctness and missed requirements, regressions and edge cases,
unintended API or behavior changes, security, performance, and dependency
implications where relevant, unnecessary complexity, whether a refactor reduced
complexity or merely moved it, and the relevant tests. Passing checks are
evidence, not proof. It does not rewrite correct code for style.

```text
Validation
1. [High] <issue> — <file/symbol> — <impact>
2. [Medium] ...
```

or `Validation passed. No actionable issues found.`

## Fix loop

Findings go to a fresh fix agent with the Task Contract, the findings, and
repository access; it fixes only those and directly related breakage, runs the
checks, and a fresh validator runs again. At most two fix and validate rounds by
default; if meaningful issues remain, stop and report the blockers.

## Parallel research

Use several research agents only for genuinely independent uncertainty, such as
separate subsystems, and merge their findings into one Research Handoff before
implementation.

## Output

Interactive: the Research Handoff for approval, then the implementation result
and the validation result. Autonomous: the implementation result, checks run,
validation result, and any remaining blockers. Never expose agent transcripts.
