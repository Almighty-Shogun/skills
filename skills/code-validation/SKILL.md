---
name: code-validation
description: Token-efficient independent code review using an isolated validation agent. Validates current changes against requirements and repository behavior without rewriting correct code.
disable-model-invocation: true
---

# Code Validation

Review local changes for correctness, regressions, and missed requirements. A
GitHub pull request belongs to `review-pr`, and CI failures to `fix-pr`.

Validation is read-only. Fixes happen only when the user asks for them after
seeing the findings.

## Independence

Run the review in a fresh validation agent, so it judges the repository state
and the diff rather than an implementation summary. Give it a Task Contract
(intended behavior, scope, constraints, acceptance criteria, drawn from the
request, tests, issue context, or an approved plan), repository access, the
current diff and status, the relevant tests, and any `AGENTS.md` or `CLAUDE.md`.
Leave out implementation reasoning and self-assessment unless a needed fact
cannot be recovered from the repository.

## Scope

Default to the working-tree diff. Without one, validate the files, commit,
branch comparison, or behavior the user names. Read beyond the diff only as far
as dependencies, contracts, tests, and regressions require; no broad audit
unless asked.

## What it checks

Correctness, missed requirements, regressions, edge cases, unintended behavior
or API changes, and error handling. Security where the change touches trust
boundaries, auth, validation, secrets, deserialization, filesystem, network, or
command execution. Performance where the path is performance-sensitive.
Conventions, and complexity that creates real maintenance or correctness risk,
including refactors that move complexity instead of removing it. For added or
upgraded dependencies: necessity, actual use, compatibility, and breaking
release notes, with maintenance, license, or transitive concerns only when they
matter. Missing or insufficient tests.

It runs the narrowest useful tests, build, lint, or typecheck. Passing checks
are evidence, not proof; it checks whether they exercise the changed contract
and its failure paths. Repository contracts and technical evidence outrank
reviewer preference, so correct code is not rewritten for style.

## Output

```text
Validation
1. [High] <problem> — <file/symbol> — <impact and required correction>
2. [Medium] ...
3. [Low] ...
```

or `Validation passed. No actionable issues found.`, optionally with one line on
the checks run.

- **High**: incorrect behavior, a concrete data or security risk, a broken
  contract, a serious regression, or a material performance failure.
- **Medium**: a meaningful edge-case failure, an incomplete requirement, a
  dependency or compatibility problem, or a likely correctness or maintenance
  problem.
- **Low**: a real but limited issue worth fixing.

Report findings only, with file and symbol references instead of copied code;
no walkthrough, praise, inspected-file lists, or issues without evidence. A pass
means no actionable issues were found within the validated scope, not that none
exist.

## Fixes

When the user asks for fixes, use a fresh implementation agent with only the
Task Contract, the findings, and repository access, then run a fresh validation
rather than letting the fixer approve its own work.
