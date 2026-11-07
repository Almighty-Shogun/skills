---
name: source-research
description: Research technical questions against authoritative external sources using an isolated research sub-agent. Supports --quick and --deep modes, infers the mode when omitted, persists concise source-backed findings to .research/<topic>.md, and never implements changes.
argument-hint: "[--quick | --deep] <research question>"
---

# Source Research

Answer technical questions from external documentation, specifications,
first-party source, changelogs, issue trackers, and maintainer statements: what
the docs say, how an API actually behaves, what changed between versions, what a
specification requires. Questions about the current codebase belong to
`repository-research`.

## Modes

- `--quick`: a narrow, factual question that one or two authoritative sources
  settle. Stop once it is answered confidently. Quick does not relax source
  quality, version correctness, or contradiction handling.
- `--deep`: a broad, ambiguous, or comparative question, where sources need
  cross-checking, implementation semantics or version differences matter,
  docs may conflict with source or changelogs, or edge cases are important.
  Investigate contradictions, version-specific behavior, and limitations, and
  stop when another source would not change the conclusion.

Without a flag, infer the mode from the question; ask only when it is genuinely
unclear. Source restrictions come in natural language ("only Microsoft
sources"); there are no scope flags.

## Research agent

Run the research in an isolated sub-agent: it may read a lot, but its handoff
stays compact.

```text
Conclusion: <concise answer>
Important findings: <decisive findings only>
Contradictions: <meaningful conflicts only>
Unresolved: <remaining uncertainty>
Report: <path to the persisted Markdown>
```

## Sources

Prefer, in roughly this order: standards and specifications, official
documentation, first-party source code, official changelogs, release notes,
issues, and API references, maintainer statements, then high-quality secondary
or community sources. Use secondary sources where primary documentation is
incomplete or real-world behavior is undocumented, and never let them silently
override stronger primary evidence.

Check that each important source applies to the relevant framework, language,
runtime, OS, library, or API version, and never blend evidence from materially
different versions. Check publication dates for changing technologies: an old
source can still be authoritative for its version, and a newer one is not
automatically right, but stale documentation is never presented as current.

When sources disagree, name the conflict, compare authority, version, and
freshness, say which is better supported and why, and say when it stays
unresolved. Distinguish what a source states directly (confirmed), what follows
from several pieces of evidence (inference), and what the evidence cannot settle
(unresolved), labelling it where the distinction matters to the reader.

When strong evidence is unavailable, say so and record the uncertainty rather
than padding with weak sources. A smaller honest report beats a confident
unsupported one.

## Report

Every run, quick or deep, persists to `.research/<descriptive-topic>.md`, such
as `.research/aspnet-idempotency.md`. If research on the topic exists, update
it: keep still-valid conclusions, replace contradicted claims, keep useful
version history, and never overwrite unrelated research.

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
alone, and protect unrelated uncommitted work.
