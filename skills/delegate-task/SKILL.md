---
name: delegate-task
description: >-
  Token-efficient development workflow using isolated research, implementation,
  and validation agents. Use for non-trivial coding tasks where isolating the
  phases improves correctness or keeps the main context small. Not for bug
  discovery, review, planning, research, simplification, or PR CI and reviewer
  feedback, which have their own skills. Supports --interactive and
  --autonomous; otherwise infers the mode from context.
argument-hint: "[--interactive | --autonomous] <task>"
---

# Delegate Task

For non-trivial coding tasks where isolating investigation, implementation, and
review improves correctness or keeps the main context small.

Read every `AGENTS.md` and `CLAUDE.md` that applies and follow them; they
override this skill's defaults, and when two conflict, ask which wins. Give each
sub-agent the rules it needs.

Workflow: `Research -> Implement -> Validate`, with a fresh agent for each phase
that is needed. Carry conclusions forward, never transcripts. When the harness
cannot start sub-agents, run the phases inline, keep validation as free of
implementation reasoning as possible, and report the reduced independence.

## Modes

- `--interactive`: research, show the plan, wait for approval, then implement
  and validate. When research is skipped, a user-supplied plan or approved
  artifact counts as approval; otherwise show the Task Contract plus the
  intended files and wait.
- `--autonomous`: research, implement, and validate without pausing, unless a
  material decision needs the user.
- Both flags: ask which one.
- No flag: choose interactive when requirements are materially ambiguous,
  approaches carry real trade-offs, the work touches architecture, public API,
  schema or migrations, auth or security, destructive behavior, or broad
  repository changes, or the user wants to approve the approach. Choose
  autonomous when the expected behavior and acceptance criteria are clear and
  the change is a localized fix, narrow feature, or constrained refactor. When
  unsure, interactive for high-impact decisions, autonomous otherwise.

An autonomous run that uncovers a decision that could change behavior beyond
the Task Contract, or API, data, security, or scope, pauses for the user; minor
choices settled by project conventions do not.

Agents have overhead. A typo, localized rename, one-line fix, or mechanical
change with no design uncertainty takes the direct path: implemented directly
with the relevant check run, after showing the Task Contract and waiting under
`--interactive`. Validation is skipped only on the direct path, and
high-impact, security, and data-changing work gets independent validation even
there.

## Task Contract

Distill the request into Goal, Scope, Constraints, Acceptance criteria, and
Non-goals when they prevent scope creep. Each agent gets this contract plus the
minimum phase handoff, and inspects files, tests, git state, and build output
itself. Prefer paths and symbols over pasted source. Handoffs are the shortest
text the next phase can act on without re-reading what the previous phase
read.

Start each phase agent with an explicit model and effort level that fits the
phase: highest for implementation, fix, and validation, lower for simple
read-only research.

## Workflow boundary

This skill is for implementing a change. Proactive bug discovery is `bughunt`,
structural review is `structure-review`, contract design is `api-design`,
substantial external research is `source-research`, investigating current
behavior is `repository-research`, planning is `plan-implementation`, PR CI
failures are `fix-pr`, PR reviewer feedback is `respond-pr`, pure
simplification is `simplify-code`, and standalone validation is
`code-validation`; route those requests there.

When the user asks to implement an artifact from one of those
(`.plan-implementation/`, `BUG_REPORT.md`, `.bughunt/`, `STRUCTURE_REVIEW.md`
or `STRUCTURE_REVIEW-<scope-slug>.md`, `.api-design/`, `.research/`,
`.repository-research/`), treat it as input and do not repeat its analysis. A
fresh read-only agent (the research agent, when research runs) confirms it
still applies to the current code. On drift, ask in interactive mode; in
autonomous mode, re-research minor drift and stop on material drift. A saved
plan lets research be skipped.

In a `bughunt` report, an answered `Desired solution:` block is the approved
design; an unanswered one is empty or holds only the placeholder `// answer`.
An `.api-design/` artifact counts as approved only when it says
`Status: Approved`; a `Draft` goes back to `api-design`. An unanswered block or
a plan's `Open decisions` leave the design unsettled: in both modes, pause and
show the options and the recommendation before implementing.

## Research

Skip it when the implementation is obvious, the user supplied an exact plan,
tests define the fix, or an approved artifact already settles the design. When
needed, a fresh read-only research agent inspects the relevant code, finds the
current behavior or root cause, and picks the smallest correct approach that
reuses existing conventions and abstractions. For a bug fix it reproduces or
otherwise establishes the failure first, preferring an existing failing test or
a minimal reproduction, and states the limitation when CI evidence has to stand
in. Temporary reproduction files go outside the working tree, or are removed by
the research agent before it hands off; the handoff names any left. When the
failure cannot be established, stop and report the limitation in both modes
unless the user supplies a design.

```text
Research Handoff
- Current/root cause: ...
- Reproduction: <test or command; observed result>
- Approach: ...
- Files/symbols: ...
- Constraints/edge cases: ...
- Acceptance criteria: ...
- Open decisions: <only if any>
- Left behind: <temporary files, only if any>
```

In interactive mode, show this handoff and wait; in autonomous mode, Open
decisions still pause for the user. A small adjustment updates the affected
items without repeating research.

## Implementation

Before implementation, record the baseline: `git status --porcelain` for the
changes that already exist, and which relevant checks already fail, run by the
implementation agent before it edits. Pre-existing changes are untouchable and
pre-existing failures are out of scope for every later agent unless the task
targets them. Outside a Git repository, the Implementation Handoff's Changed
list is the only record of the diff.

A fresh agent with the Task Contract, the approved handoff (or the user's plan
or artifact when research was skipped), and repository access makes the
smallest correct change: existing conventions, no unrelated refactors or
speculative improvements, only the files the task needs plus directly related
breakage, tests added or updated where needed, and the narrowest useful checks
run. It inspects the working tree before editing and changes it only: no
commits, pushes, branches, or history edits, and unrelated uncommitted work is
preserved.

```text
Implementation Handoff
- Baseline failures: <relevant checks failing before any edit>
- Changed: <files/symbols>
- Key decisions: <only non-obvious ones>
- Checks: <what ran + result>
- Unresolved: <only if applicable>
```

Material Unresolved items, including an approved approach that does not work,
pause for the user; the rest are carried into the output. An implementer that
changes nothing ends the run with its reason; an empty diff is never validated.
The validator gets only the baseline and the Changed list from this handoff,
never its decisions or reasoning, unless it needs a fact the repository cannot
provide.

## Validation

A fresh read-only agent with the Task Contract, the acceptance criteria, the
Reproduction, the baseline, and the Changed list, without the implementer's
reasoning or self-assessment, reviews the changed paths and any other change
absent from the baseline in `git diff HEAD` plus untracked files from
`git status`. It checks correctness and missed requirements, regressions and
edge cases, unintended API or behavior changes, security, performance, and
dependency implications where relevant, unnecessary complexity, whether a
refactor reduced complexity or merely moved it, and the relevant tests. Passing
checks are evidence, not proof. It never edits, and correct code is not
rewritten for style; only the fix agent changes code.

```text
Validation
1. [High] <problem> — <file/symbol> — <impact and required correction>
2. [Medium] ...
3. [Low] ...
```

or `Validation passed. No actionable issues found.`

- **High**: incorrect behavior, a concrete data or security risk, a broken
  contract, a serious regression, or a material performance failure.
- **Medium**: a meaningful edge-case failure, an incomplete requirement, a
  dependency or compatibility problem, or a likely correctness or maintenance
  problem.
- **Low**: a real but limited issue worth fixing.

## Fix loop

High and Medium findings go to a fresh fix agent with the Task Contract, the
findings, the baseline, the Changed list, and repository access, under the
implementation agent's constraints: working tree only, no commits, pushes,
branches, or history edits, and pre-existing changes untouched. It fixes only
those findings and directly related breakage, runs the checks, and returns
`Not fixed: <evidence the finding is wrong>` for a finding it rejects; the
orchestrator decides whether that stands. A fresh validator then runs again. At
most two fix and validate rounds by default; if High or Medium findings remain,
stop and report them as blockers. Low findings go into the output.

## Parallel research

Use several research agents only for genuinely independent uncertainty, such as
separate subsystems, and merge their findings into one Research Handoff before
implementation.

## Output

Interactive: first the Research Handoff for approval, or, when research is
skipped without a user-supplied plan or approved artifact, the Task Contract
plus the intended files. Both modes: the implementation result, checks run,
validation result, Low findings, and any remaining blockers. Direct path: the
change and the check result. Never expose agent transcripts.
