---
name: repository-research
description: Investigate how a repository currently behaves without modifying it. Traces implementations, known failures, extension points, tests, configuration, and relevant history; distinguishes confirmed facts from inference; and optionally persists findings with --save.
argument-hint: "[--save] [scope or question]"
---

# Repository Research

Understand a codebase as it exists: trace behavior, explain an implementation,
investigate a known problem, or establish why something works the way it does.
Without `--save` nothing is written; with it, the findings go to
`.repository-research/<descriptive-topic>.md`.

Neighbouring work has its own skill: proactively finding unknown bugs is
`bughunt`, external framework or specification behavior is `source-research`,
broad structural review is `structure-review`, contract design is `api-design`,
and implementation steps are `plan-implementation`. When the answer depends on
external behavior the repository cannot establish, say so and use
`source-research` rather than drifting into web research.

## Investigation

Reduce the question to a Task Contract: the question, the scope if known,
constraints, and any evidence required. For substantial investigation, hand it
to a fresh research agent with repository access so exploration stays out of
the parent context; a question answerable from one obvious place can be
answered directly. Use several agents only for genuinely independent questions,
and merge their results into one answer.

Inspect only the files, symbols, tests, configuration, docs, history, and call
paths the question needs, trace far enough to support each conclusion, and stop
once it is answered.

For a reported defect: establish or reproduce the failure, trace the behavior,
then identify the root cause, preferring an existing failing test, a minimal
reproduction, decisive CI evidence, or a deterministic bad path. If it cannot be
reproduced, do not manufacture certainty, and never call a root cause confirmed
beyond what the evidence supports.

Use Git history when it explains why code exists, when behavior changed, a
regression, or a removed guarantee; it supports a conclusion about current
behavior but does not prove one. Ignore mechanical churn.

Reuse an artifact the user references or that clearly belongs to the question
(`BUG_REPORT.md`, `.bughunt/`, `.api-design/`, `.research/`,
`STRUCTURE_REVIEW.md`, `.repository-research/`) as evidence, not authority:
confirm it against the current code.

## Output

```text
Research Findings
- Conclusion: ...
- Status: Confirmed | Inference | Unresolved
- Evidence: <important paths/symbols/tests>
- Constraints/edge cases: <only if material>
- Recommended direction: <only if useful>
- Unresolved: <only material unknowns>
```

Label conclusions individually when their certainty differs. Cite evidence as a
path and symbol, `src/Foo.cs — FooService.ExecuteAsync`, optionally with a line
number as a navigation hint. If one sentence answers the question, give one
sentence. Leave out the investigation diary, files searched, discarded
hypotheses that do not support the conclusion, long excerpts, and generic
background.

A recommended direction, when it helps, is one evidence-backed sentence, not a
redesign, API design, or implementation plan.

## Persistence and boundaries

With `--save`, update existing research on the same topic, keeping still-valid
findings, and never overwrite unrelated research; persist conclusions and
evidence, not the transcript. That file is the only write: leave source, tests,
configuration, and Git state alone, and remove any temporary instrumentation or
harness used to reproduce a failure.
