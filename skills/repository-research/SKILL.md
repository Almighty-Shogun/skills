---
name: repository-research
description: >-
  Investigate how a repository currently behaves without lasting changes.
  Traces implementations, known failures, extension points, tests,
  configuration, and relevant history; distinguishes confirmed facts from
  inference; and optionally persists findings with --save. Not for external
  documentation or specification behavior (source-research) or finding unknown
  bugs (bughunt).
argument-hint: "[--save] [scope or question]"
---

# Repository Research

Understand a codebase as it exists: trace behavior, explain an implementation,
investigate a known problem, or establish why something works the way it does.
Without `--save` this skill persists nothing itself; with it, the findings are
also written to `.repository-research/<descriptive-topic>.md` under the
repository root, and the reply ends with that path. Work routed to
`source-research` always writes its `.research/<topic>.md`.

Read every `AGENTS.md` and `CLAUDE.md` that applies and follow them; they
override this skill's defaults, and when two conflict, ask which wins. Give each
sub-agent the rules it needs.

Neighbouring work has its own skill: proactively finding unknown bugs is
`bughunt`, external framework or specification behavior is `source-research`,
broad structural review is `structure-review`, contract design is `api-design`,
and implementation steps are `plan-implementation`. When the answer depends on
external behavior the repository cannot establish, say so and use
`source-research` rather than drifting into web research.

## Investigation

With only a scope, explain how that area works; with neither scope nor
question, ask. Reduce the question to a Task Contract: the question, the scope
if known, constraints, and any evidence required. For substantial
investigation, hand it to a fresh research agent with repository access so
exploration stays out of the parent context; a question answerable from one
obvious place can be answered directly. Use several agents only for genuinely
independent questions, and merge their results into one answer. Start each
agent with an explicit model and effort level that fits its task: lower for
simple read-only research, higher for tracing a defect to its root cause.

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
`STRUCTURE_REVIEW.md` or `STRUCTURE_REVIEW-<scope-slug>.md`,
`.repository-research/`; look for the structure reviews and `.api-design/` at
the repository root) as evidence, not authority: confirm a repository artifact
against the current code, and `.research/` against the dependency versions the
repository uses.

## Output

```text
Research Findings
- Conclusion: ...
- Status: Confirmed | Inference | Unresolved
- Evidence: <important paths/symbols/tests>
- Constraints/edge cases: <only if material>
- Recommended direction: <only if useful>
- Unresolved: <only material unknowns>
- Temporary changes: <only files left behind>
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
evidence, not the transcript. The file is a working note, committed only when
the user asks. That file and any `.research/<topic>.md` from `source-research`
are the only lasting writes: leave source, tests, configuration, and Git state
alone. To reproduce a failure, prefer commands that do not rewrite tracked files
(`npm ci`, no snapshot updates), and revert any tracked-file change the run
caused. The agent that creates temporary instrumentation or a harness removes
it before handing off and lists anything left under `Temporary changes:`;
remove only what this run created.
