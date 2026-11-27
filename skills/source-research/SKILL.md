---
name: source-research
description: >-
  Research technical questions against authoritative external sources using an
  isolated research sub-agent. Use when the user explicitly asks for external
  research or another skill routes a question here. Supports --quick and --deep
  modes, infers the mode when omitted, persists concise source-backed findings
  to .research/<topic>.md, and never implements changes. Not for how the
  current repository behaves (repository-research).
argument-hint: "[--quick | --deep] <research question>"
---

# Source Research

Answer technical questions from external documentation, specifications,
first-party source, changelogs, issue trackers, and maintainer statements: what
the docs say, how an API actually behaves, what changed between versions, what a
specification requires. Questions about the current codebase belong to
`repository-research`.

Read every `AGENTS.md` and `CLAUDE.md` that applies and follow them; they
override this skill's defaults, and when two conflict, ask which wins. Give
each sub-agent the rules it needs.

## Modes

- `--quick`: a narrow, factual question that one or two authoritative sources
  settle. Stop once it is answered confidently. Quick does not relax source
  quality, version correctness, or contradiction handling. When one or two
  sources do not settle it, stop with the gap under Unresolved and suggest
  `--deep`.
- `--deep`: a broad, ambiguous, or comparative question, where sources need
  cross-checking, implementation semantics or version differences matter,
  docs may conflict with source or changelogs, or edge cases are important.
  Investigate contradictions, version-specific behavior, and limitations, and
  stop when another source would not change the conclusion.

Without a flag, infer the mode from the question; ask only when it is genuinely
unclear. With both flags, ask which one. Source restrictions come in natural
language ("only Microsoft sources"); there are no scope flags.

## Research agent

Run the research in an isolated sub-agent: it may read a lot, but its handoff
stays compact. Start it with an explicit model and effort level that fits the
mode: lower for `--quick`, higher for `--deep`. Give it the target report path,
the existing-file rule under Report, and the Boundaries. Where a sub-agent
cannot be started, such as inside another sub-agent, research inline with the
same handoff and say so.

```text
Conclusion: <concise answer>
Important findings: <decisive findings only>
Contradictions: <meaningful conflicts only>
Unresolved: <remaining uncertainty>
Report: <path to the persisted Markdown>
```

## Sources

Prefer, in roughly this order: standards and specifications, official
documentation and API references, first-party source code, official
changelogs, release notes, issues, maintainer statements, then high-quality
secondary or community sources. Use secondary sources where primary
documentation is incomplete or real-world behavior is undocumented, and never
let them silently override stronger primary evidence.

Take the relevant framework, language, runtime, OS, library, or API version
from the question, else from the manifest or lockfile of the package in
question, else state the assumed version in the Conclusion. Check that each
important source applies to that version, and never blend evidence from
materially different versions. Check publication dates for changing
technologies: an old source can still be authoritative for its version, and a
newer one is not automatically right, but stale documentation is never
presented as current.

When sources disagree, name the conflict, compare authority, version, and
freshness, say which is better supported and why, and say when it stays
unresolved. Distinguish what a source states directly (confirmed), what follows
from several pieces of evidence (inference), and what the evidence cannot settle
(unresolved), labelling it where the distinction matters to the reader.

When strong evidence is unavailable, say so and record the uncertainty rather
than padding with weak sources. A smaller honest report beats a confident
unsupported one.

## Report

Every run, quick or deep, persists to `.research/<topic>.md` under the
repository root, such as `.research/aspnet-idempotency.md`; outside a
repository, answer in chat only. The file is a working note, committed only
when the user asks. If no source can be reached, say so and persist nothing. If
research on the topic exists, update it: keep still-valid conclusions, replace
contradicted claims, keep useful version history, and never overwrite
unrelated research.

Typical sections are Conclusion, Findings, Version / Compatibility,
Unresolved, and Sources, omitting any that add nothing. Cite important factual
claims with direct links to the documentation, specification, source, or
release, enough to make them auditable without a citation on every trivial
sentence. Leave out browsing history, page-by-page summaries, and long
excerpts.

A concise, evidence-backed direction is allowed. File-by-file steps, migration
plans, and fixes are not; those belong to `plan-implementation` or
`delegate-task`.

## Boundaries

The only persistent change is the `.research/` file. Leave source and Git state
alone, and protect unrelated uncommitted work. Clone or download first-party
source into a temporary directory outside the repository, and remove it
afterwards.

## Final summary

The conclusion, the unresolved points, and the `.research/` path, without
repeating the report.
