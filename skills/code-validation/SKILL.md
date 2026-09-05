---
name: code-validation
description: Token-efficient independent code review using an isolated validation agent. Validates current changes against requirements and repository behavior without rewriting correct code.
disable-model-invocation: true
---

# Code Validation

Use when the user wants local/current changes reviewed, verified, audited for regressions, or checked against requirements.

For a GitHub pull request, use `review-pr`. For GitHub Actions diagnosis/fixing, use `fix-pr`.

This skill is validation-only by default. Do not modify code unless the user explicitly asks for fixes after seeing the findings.

## Core rule

Use a fresh validation agent so the review is independent from the implementation process.

The validator should inspect actual repository state and the current diff rather than trusting an implementation summary.

## Task Contract

Reduce the request to:

- Intended behavior / goal
- Scope
- Constraints
- Acceptance criteria

If requirements can be recovered from tests, issue context, or a concise approved plan already present, include only the relevant parts.

Do not forward implementation reasoning or self-assessment unless a necessary fact cannot be recovered from repository state.

## Validation agent

Give the agent:

- Task Contract
- repository access
- direct access to current code, git diff/status, and relevant tests
- relevant repository instructions such as `AGENTS.md` / `CLAUDE.md` when present

It should independently check:

- correctness
- missed requirements
- regressions
- edge cases
- unintended behavior/API changes
- error handling where relevant
- security-sensitive behavior where the change touches trust boundaries,
  authorization/authentication, validation, secrets, deserialization, filesystem,
  network, or command execution
- meaningful performance regressions where the changed path is performance-sensitive
- compatibility with existing conventions
- unnecessary complexity that creates real maintenance or correctness risk
- whether a refactor actually reduces complexity rather than moving it elsewhere
- dependency additions/upgrades when relevant: necessity, actual usage, compatibility,
  and breaking migration/release notes; verify maintenance/license/transitive concerns
  only when they materially affect the project
- missing or insufficient tests/checks

Run the narrowest useful tests, build, lint, or type checks when practical. Passing
checks are necessary evidence, not sufficient proof of correctness; inspect whether
they cover the changed contract and important failure paths.

Do not rewrite correct code for stylistic preference. Technical evidence and
repository contracts outrank reviewer preference.

## Output

If issues exist, return only actionable findings ordered by severity:

```text
Validation
1. [High] <problem> — <file/symbol> — <impact and required correction>
2. [Medium] ...
3. [Low] ...
```

If no meaningful issues exist:

```text
Validation passed. No actionable issues found.
```

Optionally add one compact line for checks run when that information is useful.

Default ceiling: ~500 tokens. Use less when possible.

## Severity

Use severity based on impact, not preference:

- High: incorrect behavior, concrete data/security risk, broken API/contract,
  serious regression, or material performance failure
- Medium: meaningful edge-case failure, incomplete requirement, dependency/compatibility
  problem, or likely maintainability/correctness problem
- Low: real but limited issue worth fixing

Do not report purely subjective style preferences as findings.

## Token efficiency

Do not include:

- a walkthrough of the implementation
- praise or summaries of correct code
- every file inspected
- implementation-agent reasoning
- large diffs or source excerpts
- speculative issues unsupported by repository evidence

Prefer references to exact files/symbols over copied code.

## Validation scope

Default to the current working-tree diff when changes are present.

Expand beyond the diff only as needed to understand dependencies, contracts, tests, or regressions. Do not perform a broad repository audit unless requested.

If there is no diff, validate the explicitly named files, commit, branch comparison, or behavior the user identifies.

## Fixes

Do not automatically fix findings. This keeps validation independent and prevents scope creep.

If the user subsequently asks to fix the findings, use a fresh implementation agent with only:

- Task Contract
- validation findings
- repository access

After fixes, recommend or perform a fresh validation pass rather than relying on the fixer to self-approve.

## Repository safety

The validation pass is read-only.

Do not:
- modify source, tests, configuration, or documentation;
- stage, commit, or push;
- create/delete branches;
- reset, stash, clean, amend, or rebase;
- overwrite unrelated user work.

If the user asks to fix findings afterward, that is a separate implementation step.

## User-facing behavior

Report findings concisely and independently. Do not imply that a passing review proves absence of all possible defects; it means no actionable issues were found within the validated scope.
