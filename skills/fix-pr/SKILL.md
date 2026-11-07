---
name: fix-pr
description: Diagnose and fix GitHub Actions failures on a pull request. Resolve the PR, group failing checks by root cause, verify each failure, show a concise diagnosis and proposed fix for user approval, apply approved local changes through isolated sub-agents, validate independently, and never commit or push automatically.
argument-hint: "[<number-or-url>]"
---

# Fix PR

Diagnose and fix GitHub Actions failures on one pull request. General review
belongs to `review-pr`, reviewer feedback to `respond-pr`, and merging to
`merge-pr`.

## Contract

Approved fixes change the working tree only. Commits and pushes go through the
`commit` skill when the user wants them; this skill never commits, pushes,
amends, rebases, or creates or deletes branches. Stay within the approved failure scope,
follow repository conventions, and protect unrelated uncommitted work.

## Resolve the PR

Use the PR URL when given, else the number in the current repository, else the
pull request for the current branch. If none resolves, or several plausibly
could, ask for a number or URL; never guess. Follow the repository and PR
resolution conventions of `review-pr`.

## Failing checks

Read failed, timed-out, and meaningfully cancelled checks. Leave successful
output out of context unless comparing against it isolates a failure.

GitHub Actions only. For an external provider (Buildkite, CircleCI, Jenkins,
and the like), report the check, its status, and its link, and never infer a
cause from logs you cannot read.

Group failures by root cause: four jobs failing on one compile error are one
root cause and one piece of work. Classify each as a code failure, workflow or
configuration failure, infrastructure failure, external dependency failure,
secret or access failure, flaky failure, or unknown. Never change application
code to compensate for an infrastructure failure.

## Diagnose

Give each independent root cause its own diagnosis sub-agent, with only its
failing checks, the decisive log excerpt, the relevant repository context, and
the commands to reproduce. Reproduce locally where practical, using the same
test, build, typecheck, lint, or workspace command. When runner-only behavior,
platform differences, secrets, or hosted services make that impossible, work
from the CI evidence and say so. A red workflow alone does not establish a
cause.

```text
Check(s): <affected checks>
Classification: <category>
Cause: <verified or best-supported root cause>
Evidence: <decisive evidence only>
Local reproduction: <command and result, or why not practical>
Proposed fix: <short direction>
Confidence: <high | medium | low, where useful>
```

A rerun that passes does not prove a flaky failure fixed: establish
repeatability where practical, note the conditions and frequency, and tell real
flakiness from transient infrastructure.

Never print, recover, or copy secret values. When a required secret or setting
is unavailable, name it, explain the failure, and stop unless the user can
resolve it safely.

## Approval

Before changing anything, show the diagnosis and proposed fix, however
confident it is, and wait for explicit approval:

```text
Check: test / Linux

Cause:
PaginationTests fails because page 0 is now accepted by the public boundary.

Evidence:
The failing assertion and local reproduction both confirm the regression.

Proposed fix:
Restore page validation at the public boundary.
```

## Never weaken CI to make it green

Deleting or skipping tests, loosening assertions, disabling linters, suppressing
compiler or type errors, removing matrix entries, adding `continue-on-error`,
lowering coverage thresholds, or bypassing security checks is acceptable only
when it is the correct behavior by the project's actual requirements. A
workflow file may change when the workflow itself is wrong: confirm it is the
real cause, that the application is not simply failing correctly, and that
required coverage is not reduced.

## Implement and validate

Assign each approved root cause to an isolated implementation sub-agent with
the approved diagnosis, the failing checks, the fix direction, and the relevant
code. It returns:

```text
Status: Applied | Not applied
Files changed: <files>
What changed: <summary>
Local checks: <commands and results>
Remaining issue: <only when applicable>
```

Then a fresh validation sub-agent checks the original failure, the approved
diagnosis, the actual diff, the relevant local checks, regressions, and whether
CI strictness was weakened, without relying on the implementer's account. A
concrete defect it finds gets one targeted fix and another validation; avoid
open-ended repair loops.

## Report

Local fixes trigger no new CI run until they are committed and pushed, so say
the changes are ready for the `commit` skill and never promise green CI before a
post-fix run confirms it. A run triggered independently may be inspected.

```text
Root causes found: 2
Fixes applied: 2
Local validation: passed
GitHub re-run: pending commit/push
```

Also name infrastructure or external failures that could not be fixed,
unresolved flaky failures, and secret or configuration issues needing the user.
Keep diagnoses short and human: the check, the cause, the decisive evidence, the
fix direction, with no log dumps, implementation diaries, attribution, or
remarks about the work being focused or minimal.
