---
name: delegate-task
description: Token-efficient development workflow using isolated research, implementation, and validation agents. Supports --interactive and --autonomous; otherwise infers the mode from context.
argument-hint: "[--interactive | --autonomous] <task>"
---

# Delegate Task

Use for non-trivial coding tasks where isolated investigation, implementation, and review improve correctness or reduce context pollution.

Workflow: `Research -> Implement -> Validate`

Use a fresh agent for each needed phase. Carry conclusions forward, never full agent transcripts.

## Modes

- `--interactive`: research first, show the plan, wait for user approval, then implement and validate.
- `--autonomous`: research, implement, and validate without pausing unless a material user decision is required.
- no flag: infer the mode from task risk, ambiguity, scope, and conversation context.

Treat the flags as mode directives in the user's invocation.

### Infer interactive when

- requirements are materially ambiguous
- multiple approaches have meaningful tradeoffs
- architecture, public API, schema/migrations, auth/security, destructive behavior, or broad repository changes are involved
- user preference determines the correct design
- the user asks to approve/review the approach first

### Infer autonomous when

- expected behavior is clear
- acceptance criteria are explicit or obvious from tests/current behavior
- the task is a localized bug fix, narrow feature, or constrained refactor
- existing project conventions make the implementation choice clear

If uncertain, choose interactive for high-impact decisions; otherwise autonomous.

If autonomous research uncovers a decision that could materially change behavior, API, data, security, or scope, pause for the user. Do not pause for minor implementation choices resolvable from project conventions.

## Avoid unnecessary delegation

Sub-agents have overhead. If the task is trivial, skip unnecessary phases.

Examples: typo, obvious rename, one-line config/import fix, mechanical change with no design uncertainty.

For trivial autonomous tasks, implement directly and run the relevant check. Never skip independent validation for high-impact/security/data-changing changes.

## Context rules

Before delegation, distill the request into a compact Task Contract:

- Goal
- Scope
- Constraints
- Non-goals, only when they materially prevent scope creep
- Acceptance criteria

Only include conversation details needed for the task.

For every agent:

- give only the Task Contract plus the minimum phase-specific handoff
- let it inspect repository files, tests, git status/diff, and build output directly
- prefer paths/symbols over pasted source
- never forward full transcripts, reasoning, exploration history, large command output, or repository walkthroughs
- omit discarded approaches unless their rejection matters to the next phase

Default handoff ceilings, not targets:

- Research: ~500 tokens
- Implementation: ~300 tokens
- Validation: ~400 tokens

Use less whenever possible. Correctness beats the token budget.

## Workflow boundary

Use this skill when the intended outcome is an implementation or repository change.

Do not silently turn it into a substitute for specialized discovery/design workflows such as:

- `bughunt` for proactively discovering and proving unknown bugs
- `structure-review` for finding structural/architectural improvements
- `api-design` for designing a public or cross-boundary contract
- `source-research` for substantial external research

If the user explicitly asks to implement an existing artifact or finding from one of those workflows, treat that artifact as input. Verify that it is still applicable, then avoid repeating the completed analysis.

## Research

Use a fresh research agent when investigation is needed.

It must:

- inspect relevant code only
- identify current behavior/root cause
- determine the smallest correct approach
- reuse existing project conventions/abstractions
- identify affected files/symbols
- identify important constraints/edge cases
- refine acceptance criteria if needed
- for bug fixes, reproduce or otherwise prove the reported failure where practical before designing the fix
- reuse prior approved design/research artifacts when applicable instead of redoing their work
- not modify files

Return only:

```text
Research Handoff
- Current/root cause: ...
- Approach: ...
- Files/symbols: ...
- Constraints/edge cases: ...
- Acceptance criteria: ...
```

No investigation diary, long code excerpts, or generic background.

Skip research when the implementation is already obvious, the user supplied an exact plan, tests clearly define the fix, or an approved/relevant repository artifact already settles the design and only needs a quick applicability check.

For bug fixes, prefer an existing failing test or a minimal reproduction. Do not spend significant effort designing a solution for a failure that cannot be reproduced or otherwise established. If reproduction is impractical but repository/CI evidence is decisive, state that constraint briefly.

## Interactive approval

In `--interactive` mode:

1. complete research
2. show the concise Research Handoff
3. wait for approval/adjustments
4. implement only after approval

If the user makes a small adjustment, update only the affected handoff items; do not repeat research unnecessarily.

## Implementation

Use a fresh implementation agent.

Give it only:

- Task Contract
- final/approved Research Handoff, if one exists
- repository access

It must:

- make the smallest correct change
- follow existing conventions
- avoid unrelated refactors/speculative improvements
- preserve unrelated user work
- never reset, stash, clean, checkout over, or otherwise overwrite unrelated changes
- touch only files needed for the task unless directly related breakage requires more
- add/update relevant tests when needed
- run the narrowest useful tests/build/lint/type checks
- not commit, push, create branches, rebase, amend, or perform other Git-history mutations

Return only:

```text
Implementation Handoff
- Changed: <files/symbols>
- Key decisions: <only non-obvious ones>
- Checks: <what ran + result>
- Unresolved: <only if applicable>
```

Do not forward this handoff to validation unless the validator cannot obtain a necessary fact from repository state.

## Validation

Use a fresh validation agent.

Give it:

- Task Contract
- acceptance criteria
- direct access to current code and git diff

Do not give it the implementation agent's reasoning or self-assessment.

Check independently for:

- correctness and missed requirements
- regressions/edge cases
- unintended API/behavior changes
- security implications when relevant
- meaningful performance regressions when relevant
- dependency additions/upgrades and their implications when relevant
- unnecessary complexity
- whether a refactor actually reduces complexity rather than merely moving it
- relevant tests/checks

Passing tests/checks is necessary evidence, not sufficient proof of correctness.

Do not rewrite correct code for stylistic preference.

If issues exist, return only actionable findings ordered by severity:

```text
Validation
1. [High] <issue> — <file/symbol> — <impact>
2. [Medium] ...
```

Otherwise return:

```text
Validation passed. No actionable issues found.
```

## Fix loop

If validation finds issues:

1. use a fresh fix agent
2. give it only Task Contract + validation findings + repository access
3. fix only those issues and directly related breakage
4. run relevant checks
5. use a fresh validator again

Maximum 2 fix/validate iterations by default. If meaningful issues remain, stop and report the blockers instead of looping indefinitely.

## Parallel research

Do not parallelize by default. Use multiple research agents only when the task has genuinely independent uncertainty (for example architecture vs security or separate subsystems).

Synthesize parallel findings into one compact Research Handoff before implementation. Never pass multiple full reports forward.


## Repository safety

This skill modifies the working tree only.

It must not:
- commit;
- push;
- create or delete branches;
- amend or rebase;
- reset;
- stash unrelated work;
- run destructive clean operations;
- checkout over unrelated user changes.

Dedicated Git/PR skills own those operations.

Always inspect the working tree before modification and preserve unrelated changes.

## User-facing output

Interactive mode normally shows:

- Research Handoff for approval
- concise implementation result
- validation result

Autonomous mode normally shows only:

- concise implementation result
- checks performed
- validation result
- remaining blockers, if any

Do not expose internal transcripts or chain-of-thought.
