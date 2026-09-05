---
name: fix-pr
description: Diagnose and fix GitHub Actions failures on a pull request. Resolve the PR, group failing checks by root cause, verify each failure, show a concise diagnosis and proposed fix for user approval, apply approved local changes through isolated sub-agents, validate independently, and never commit or push automatically.
argument-hint: "[<number-or-url>]"
disable-model-invocation: true
---

# Fix PR

Use this skill to diagnose and fix CI failures on a pull request.

This skill is PR-specific and handles GitHub Actions failures only.

It is responsible for:
- resolving the target pull request;
- inspecting relevant failing GitHub Actions checks;
- grouping failures by underlying root cause;
- distinguishing code failures from infrastructure/configuration failures;
- verifying likely causes locally where practical;
- proposing a fix;
- requiring user approval before modifying code;
- applying approved fixes with isolated sub-agents;
- independently validating the resulting diff;
- leaving commit/push work to the existing Git workflow.

It does not perform a general PR review. Use `review-pr` for that.

## Invocation

Supported forms:

```text
fix-pr
fix-pr <PR number>
fix-pr <PR URL>
```

Resolution order:

1. If the user provides a PR URL, use that PR.
2. If the user provides a PR number, use that PR in the current repository.
3. Otherwise resolve the PR associated with the current branch.
4. If no clear PR can be resolved, ask the user for a PR number or URL.

Do not guess between multiple possible PRs.

## CI provider scope

Handle GitHub Actions only.

For checks owned by external providers such as:
- Buildkite;
- CircleCI;
- Jenkins;
- GitLab CI;
- Azure DevOps;
- other third-party systems;

report the failed check and available link, but do not pretend to diagnose logs that are not accessible.

Keep external-provider support outside this skill unless explicitly added later.

## Core workflow

```text
Resolve PR
   ↓
Fetch failing checks
   ↓
Group failures by root cause
   ↓
Isolated diagnosis sub-agents
   ↓
Local reproduction where practical
   ↓
Concise diagnosis + proposed fix
   ↓
User approval
   ↓
Isolated implementation sub-agents
   ↓
Fresh validation
   ↓
Ready for commit/push
```

## Failing checks only

Focus on:
- failed checks;
- timed-out checks;
- relevant cancelled checks when cancellation itself may indicate a problem.

Do not dump successful workflow output into context.

Inspect successful checks only when they materially help compare behavior or isolate a failure.

## Group failures by root cause

Multiple failed jobs may share one underlying defect.

Example:

```text
Linux build failed
Windows build failed
Test job failed
Package job failed
```

If all four failures stem from the same compile error, treat them as one root cause.

Do not create separate implementation work merely because one defect appears in several jobs.

## Failure classification

Classify failures internally as one of:

```text
Code failure
Workflow/configuration failure
Infrastructure failure
External dependency failure
Secret/access failure
Intermittent/flaky failure
Unknown
```

Examples:

- compiler or test failure → likely `Code failure`;
- incorrect workflow syntax/path/matrix config → `Workflow/configuration failure`;
- unavailable GitHub runner → `Infrastructure failure`;
- registry/service outage → `External dependency failure`;
- missing required secret → `Secret/access failure`;
- nondeterministic timeout → possibly `Intermittent/flaky failure`.

Do not modify application code to compensate for infrastructure failures.

## Diagnosis sub-agents

Use isolated diagnosis sub-agents for independent root causes.

Example:

```text
Root cause A → diagnosis sub-agent A
Root cause B → diagnosis sub-agent B
```

Each diagnosis agent receives only:
- the relevant failing check(s);
- decisive log excerpts or references;
- relevant repository context;
- commands needed to reproduce where practical.

Do not give every diagnosis agent all CI logs.

Preferred diagnosis handoff:

```text
Check(s):
<affected checks>

Classification:
<failure category>

Cause:
<verified or best-supported root cause>

Evidence:
<decisive evidence only>

Local reproduction:
<command/result or why reproduction is not practical>

Proposed fix:
<short fix direction>

Confidence:
<high / medium / low where useful>
```

Do not return:
- full CI logs;
- investigation transcripts;
- unrelated workflow output;
- discarded hypotheses;
- generic CI advice.

## Local reproduction

Where practical, reproduce the failing CI command locally before modifying code.

Prefer:
- the exact failing test command;
- the same build/type-check/lint command;
- the same package/workspace command;
- an equivalent local environment when runner-specific behavior is not required.

If local reproduction is not practical because of:
- runner-only behavior;
- platform differences;
- secrets;
- hosted-service dependencies;
- unavailable external systems;

use the CI evidence directly and state that local reproduction was not possible.

A red workflow alone is not sufficient evidence of the root cause.

## Flaky failures

A rerun that passes does not automatically prove the problem is fixed.

For intermittent failures:
- establish repeatability where practical;
- note observed conditions/frequency;
- distinguish genuine flakiness from transient infrastructure;
- do not call the issue resolved merely because one rerun succeeded.

## Secrets and credentials

Never:
- print secret values;
- attempt to recover secret values;
- expose credentials from logs;
- copy sensitive values into reports or prompts.

If CI fails because a required secret/configuration is unavailable:
- identify only the secret/config name;
- explain the failure concisely;
- stop unless the user can resolve the configuration safely.

## Approval before implementation

Always show the user a concise diagnosis and proposed fix before modifying code.

Example:

```text
Check: test / Linux

Cause:
PaginationTests fails because page 0 is now accepted by the public boundary.

Evidence:
The failing assertion and local reproduction both confirm the regression.

Proposed fix:
Restore page validation at the public boundary.
```

Require explicit user approval before applying the fix.

Do not begin implementation merely because the diagnosis is high-confidence.

## Never weaken CI merely to make it green

Do not:
- delete failing tests;
- skip tests;
- loosen assertions without justification;
- disable linters;
- suppress compiler/type errors;
- remove matrix entries;
- add `continue-on-error`;
- lower coverage thresholds;
- bypass security checks;
- silence failures;

merely to make the workflow pass.

Any such change is acceptable only when it is the correct behavior and directly supported by the intended project requirements.

## Workflow-file changes

Workflow files may be modified when the workflow itself is genuinely incorrect.

Before changing `.github/workflows/...`, verify that:
- the workflow configuration is the actual cause;
- the application is not simply failing correctly;
- the change does not weaken required coverage or validation.

Apply extra scrutiny to changes that reduce CI coverage.

## Implementation sub-agents

After user approval, assign each independent approved root cause to an isolated implementation sub-agent where practical.

Each implementation agent receives only:
- the approved diagnosis;
- the relevant failing checks;
- the proposed fix direction;
- relevant code/workflow context.

Preferred handoff:

```text
Status:
Applied | Not applied

Files changed:
<relevant files>

What changed:
<concise summary>

Local checks:
<commands/results>

Remaining issue:
<only when applicable>
```

Do not return implementation diaries or unrelated repository changes.

## Local code changes

Approved fixes may modify the working tree.

Changes must:
- remain within the approved failure scope;
- follow repository conventions;
- avoid unrelated refactors;
- preserve intended test/CI strictness;
- protect unrelated user work.

Never:
- reset unrelated work;
- stash unrelated work;
- clean the repository destructively;
- checkout over user changes;
- overwrite unrelated uncommitted files.

## Independent validation

After implementation, use a fresh validation sub-agent.

The validator should inspect:
- the original CI failure;
- the approved diagnosis;
- the actual resulting diff;
- relevant local tests/build/lint/type-check commands;
- potential regressions;
- whether CI strictness was weakened improperly.

Do not rely on the implementation agent's explanation.

If validation identifies a concrete defect:
- fix only that defect with an isolated implementation sub-agent;
- validate again.

Avoid open-ended repair loops.

## GitHub recheck

`fix-pr` does not commit or push automatically.

Therefore, local fixes will usually not trigger new GitHub Actions runs yet.

After local validation:
- report that the changes are ready to commit/push;
- use the existing `commit` workflow when the user wants to create a commit;
- inspect new CI results later if they exist.

If a new workflow run has already been triggered independently, `fix-pr` may inspect it.

Do not promise GitHub CI is green until an actual post-fix run confirms it.

## No automatic commit or push

Do not:
- commit;
- push;
- force-push;
- amend;
- rebase;
- create or delete branches;

unless another explicit workflow handles those operations.

`fix-pr` modifies the working tree only.

## External checks

When an unsupported provider fails, report only what is known.

Example:

```text
External check: Buildkite
Status: Failed
Details: <available link>
```

Do not infer a root cause from inaccessible logs.

## Writing style

Use the same concise, human writing style as the other PR skills.

User-facing diagnoses should:
- state the failing check;
- explain the verified cause;
- show decisive evidence;
- state the proposed fix direction;
- avoid large log dumps;
- avoid implementation diaries;
- avoid unnecessary headings;
- avoid boilerplate;
- avoid generated-by/tool attribution;
- avoid narrating that the work is focused, minimal, scoped, or clean.

## Final summary

Keep the final result compact.

Useful information:

```text
Root causes found: 2
Fixes applied: 2
Local validation: passed
GitHub re-run: pending commit/push
```

Also mention:
- infrastructure/external failures that could not be fixed;
- unresolved flaky failures;
- any secret/configuration issue requiring user action.

Do not repeat full CI logs or full implementation details.

## Repository and PR conventions

Follow the same repository-resolution, PR-resolution, safety, and writing conventions used by the existing PR skills where applicable.

Keep responsibilities separate:

```text
review-pr
→ discover PR quality issues
→ read-only

respond-pr
→ address reviewer feedback
→ local changes allowed

fix-pr
→ diagnose and fix GitHub Actions failures
→ local changes allowed

merge-pr
→ verify merge readiness and merge
```

## Token efficiency

- Fetch only relevant failing-check data.
- Group jobs by root cause before delegation.
- Delegate independent root causes separately.
- Pass decisive log evidence instead of full logs.
- Prefer local reproduction over repeated prose analysis.
- Keep diagnosis and implementation handoffs compact.
- Keep validation independent.
- Do not load successful workflow output unless needed.
- Do not investigate unsupported external CI providers.
