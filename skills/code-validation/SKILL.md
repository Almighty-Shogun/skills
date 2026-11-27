---
name: code-validation
description: >-
  Token-efficient independent code review using an isolated validation agent.
  Validates current changes against requirements and repository behavior without
  rewriting correct code. Use only when the user or repository instructions
  explicitly ask for it.
argument-hint: "[<paths> | <commit> | <base>...<head>]"
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
diff of the resolved scope (`git show <commit>`, `git diff <base>...<head>`, the
named paths, or the working-tree diff), the relevant tests, and any `AGENTS.md`
or `CLAUDE.md`. When nothing states the intent, infer it from the diff and
commit messages, label it inferred, and skip missed-requirement findings.
Leave out implementation reasoning and self-assessment unless a needed fact
cannot be recovered from the repository. Start each agent with an explicit model
and effort level that fits its task: highest for validation and fixes, lower for
simple read-only research. When a fresh agent cannot be started, validate
inline and say that the result is not independent.

## Scope

Validate the files, commit, branch comparison, or behavior the user names.
When nothing is named, default to the working-tree changes: `git diff HEAD`
plus the untracked files from `git ls-files --others --exclude-standard`, or
`git diff --cached` plus untracked files when HEAD does not exist. Leave out
known skill artifacts (`BUG_REPORT.md`, `.bughunt/`, `.plan-implementation/`,
`.research/`, `.repository-research/`, `.api-design/`, `STRUCTURE_REVIEW.md`,
`STRUCTURE_REVIEW-<scope-slug>.md`, `SIMPLIFICATION_REPORT.md`,
`.release-notes/`) unless they are the subject. With a clean tree,
validate the branch against its merge-base with the default branch; on the
default branch with nothing to compare, ask. Outside a Git repository, ask for
the files. Read beyond the diff only as far as dependencies, contracts, tests,
and regressions require; no broad audit unless asked.

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

It runs the narrowest useful tests, build, lint, or typecheck, in non-mutating
variants only: no `--fix`, CI mode for snapshot tests, no installs. It compares
`git status` before and after and reports anything that changed. When a check
fails, it checks whether it also fails on the base of the change (HEAD for
working-tree changes) in a temporary worktree, never by stashing, and reports
pre-existing failures separately from regressions. Passing checks are
evidence, not proof; it checks whether they exercise the changed contract and
its failure paths. Repository contracts and technical evidence outrank
reviewer preference, so correct code is not rewritten for style.

## Output

```text
Validation
1. [High] <problem> — <file/symbol> — <impact and required correction>
2. [Medium] ...
3. [Low] ...
```

or `Validation passed. No actionable issues found.` Either way, add one line on
the checks that ran and those that could not.

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
Task Contract, the findings the user selected, and repository access. Changes
stay in the working tree: no commits, pushes, branches, or history edits, and
unrelated uncommitted work is protected. It hands off:

```text
Finding: <number>
Status: Fixed | Not fixed: <reason>
Files changed:
Checks run:
```

Then run a fresh validation of the original scope plus the fix's working-tree
changes, against the same Task Contract, rather than letting the fixer approve
its own work. Run one fix round only, then report any remaining findings to
the user.
