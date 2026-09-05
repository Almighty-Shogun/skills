---
name: repository-research
description: Investigate how a repository currently behaves without modifying it. Traces implementations, known failures, extension points, tests, configuration, and relevant history; distinguishes confirmed facts from inference; and optionally persists findings with --save.
argument-hint: "[--save] [scope or question]"
disable-model-invocation: true
---

# Repository Research

Use when the goal is to understand a codebase, trace behavior, investigate a known problem, explain an implementation, or determine why something works the way it does.

This skill is research-only.

Without `--save`, it does not modify the repository.

With `--save`, its only persistent change is the research artifact under `.repository-research/`.

## Core rule

Delegate substantial investigation to a fresh research agent so exploratory repository context does not pollute the parent context.

The parent should receive conclusions and decisive evidence, not the investigation transcript.

For a tiny question that can be answered immediately from one obvious location, delegation is optional.

## Task Contract

Before investigation, reduce the request to:

- Question / goal
- Scope, if known
- Constraints
- Evidence required, if any

Do not forward unrelated conversation history.

## Boundaries with other skills

Use this skill to answer questions about the repository as it currently exists.

It may investigate a known or reported defect, including tracing and establishing its root cause where practical.

Do not turn ordinary repository investigation into:

- `bughunt` — proactive discovery and proof of unknown bugs;
- `source-research` — substantial external framework/specification research;
- `structure-review` — broad structural improvement discovery;
- `api-design` — public or cross-boundary contract design;
- `plan-implementation` — executable implementation planning.

If the answer materially depends on external framework, protocol, library, or specification behavior that cannot be established from repository-local evidence, identify that dependency and use `source-research` instead of silently expanding into broad web research.

## Research agent

Give the agent only the Task Contract and repository access.

It should:

- inspect only relevant files, symbols, tests, configuration, local docs, history, and call paths;
- trace behavior far enough to support its conclusions;
- classify material findings as `Confirmed`, `Inference`, or `Unresolved`;
- reuse repository evidence instead of generic assumptions;
- stop exploring once the question is answered with sufficient evidence;
- not modify files.

Prefer paths, symbols, diffs, tests, and direct repository inspection over pasted source or repeated descriptions.

## Known bug investigations

When investigating a reported defect, use this sequence where practical:

```text
Establish/reproduce the failure
        ↓
Trace the relevant behavior
        ↓
Identify the root cause
```

Prefer:
- an existing failing test;
- a minimal reproduction;
- decisive CI/log evidence;
- a deterministic bad path in the code.

If the failure cannot be reproduced, do not manufacture certainty.

Distinguish:
- what the repository proves;
- what is inferred;
- what remains unresolved.

Do not state a root cause as confirmed unless the evidence supports it.

This differs from `bughunt`: this skill investigates an already-known question/problem; `bughunt` proactively searches for unknown defects.

## Git history

Use Git history only when it materially helps answer the question, such as:

- why code exists;
- when behavior changed;
- regression investigation;
- removed or changed guarantees.

History is supporting evidence, not proof of current behavior.

Ignore mechanical churn such as formatting, mass renames, generated updates, or other changes that do not explain the behavior.

## Evidence locations

Identify decisive evidence with stable repository references.

Prefer:

```text
src/Foo.cs — FooService.ExecuteAsync
```

A current line number may be added as a navigation hint:

```text
src/Foo.cs line 42 — FooService.ExecuteAsync
```

The path + symbol must remain sufficient if line numbers move.

Do not dump every file inspected.

## Reuse relevant artifacts

When the user references an existing artifact, or one clearly belongs to the requested question, inspect it before repeating completed work.

Examples:

```text
BUG_REPORT.md
.bughunt/*.md
.api-design/*.md
.research/*.md
STRUCTURE_REVIEW.md
.repository-research/*.md
```

Do not blindly load every artifact in the repository.

Treat prior artifacts as evidence, not authority. Verify current repository state before relying on conclusions that may be stale.

## Recommended direction

A brief recommended direction is allowed when it materially helps answer the question.

Keep it to one concise evidence-backed direction at most.

Do not turn repository research into:
- solution research;
- architecture redesign;
- detailed API design;
- a file-by-file implementation plan.

Use the corresponding specialized skill when that work is actually requested.

## Persistence

Without `--save`, return findings in chat only.

With `--save`, persist the result under:

```text
.repository-research/<descriptive-topic>.md
```

Examples:

```text
.repository-research/altgr-euro-input.md
.repository-research/request-dispatch.md
.repository-research/cache-registration.md
```

Before writing, check whether research for the same topic already exists.

Update it when clearly appropriate and preserve still-valid findings. Do not overwrite unrelated research.

Persist only the compact conclusions and evidence, never the investigation transcript.

## Output

Return a compact handoff, normally under ~500 tokens:

```text
Research Findings
- Conclusion: ...
- Status: Confirmed | Inference | Unresolved
- Evidence: <important paths/symbols/tests>
- Constraints/edge cases: <only if material>
- Recommended direction: <only if useful>
- Unresolved: <only material unknowns>
```

Use less when possible.

If several material conclusions have different certainty, label them individually instead of assigning one status to the entire investigation.

Correctness is more important than the token ceiling.

## Repository safety

This skill is read-only except for the optional `.repository-research/` artifact created by `--save`.

Do not:

- modify source, tests, configuration, or documentation;
- leave temporary instrumentation, harnesses, or fixes behind;
- stage, commit, or push;
- create or delete branches;
- reset, stash, clean, rebase, or amend;
- checkout over or overwrite unrelated user work.

## Token efficiency

Do not return:

- investigation diaries;
- every file searched;
- discarded hypotheses unless their rejection materially supports the conclusion;
- long source excerpts;
- generic background the user did not ask for;
- repeated repository structure;
- large command output.

If one sentence answers the question, do not manufacture a full report.

## Parallel research

Do not parallelize by default.

Use multiple isolated research agents only when there are genuinely independent questions that can be investigated separately.

Synthesize them into one compact result before returning to the user. Never expose several full research transcripts.

## User-facing behavior

Return the research findings directly.

Do not imply that implementation will follow automatically.
