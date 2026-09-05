---
name: source-research
description: Research technical questions against authoritative external sources using an isolated research sub-agent. Supports --quick and --deep modes, infers the mode when omitted, persists concise source-backed findings to .research/<topic>.md, and never implements changes.
argument-hint: "[--quick | --deep] <research question>"
disable-model-invocation: true
---

# Source Research

Use this skill for technical research that depends on external documentation, specifications, source code, changelogs, issue trackers, maintainer statements, or other authoritative sources.

This skill is for answering questions such as:
- What does the official documentation say?
- How does this API or framework actually behave?
- What changed between versions?
- What does the specification require?
- How does the first-party implementation handle this?
- Which interpretation is best supported by authoritative evidence?

It is not for repository-local investigation. Use `repository-research` when the answer should come primarily from the current codebase.

## Invocation

Supported modes:

```text
source-research --quick
source-research --deep
```

If neither mode is supplied, infer the appropriate mode from the research question.

Use `--quick` when:
- the question is narrow;
- the expected answer is factual;
- one or two authoritative sources are likely sufficient;
- version applicability is straightforward.

Use `--deep` when:
- the question is broad, ambiguous, or comparative;
- multiple sources need cross-checking;
- implementation semantics matter;
- version differences may matter;
- documentation may conflict with source code or changelogs;
- edge cases, limitations, or unresolved behavior are important.

If the correct mode cannot be determined confidently from context, ask the user which mode they want before researching.

Do not invent additional scope flags. Source restrictions should be expressed naturally by the user, for example:

```text
Use only Microsoft sources.
Only use official documentation and source code.
Include community reports as supporting evidence.
```

## Core workflow

```text
Research question
      ↓
Infer or read --quick / --deep
      ↓
Isolated research sub-agent
      ↓
Authoritative source collection
      ↓
Version/freshness validation
      ↓
Cross-check / contradiction handling
      ↓
Concise research artifact
      ↓
Compact parent handoff
```

## Source hierarchy

Prefer sources in roughly this order:

1. Standards and specifications
2. Official documentation
3. First-party source code
4. Official changelogs, issue trackers, release notes, and API references
5. Maintainer statements
6. High-quality secondary or community sources

Prefer primary evidence for factual claims.

Secondary or community sources may be used when:
- primary documentation is incomplete;
- real-world behavior is undocumented;
- they provide material evidence not available elsewhere.

Do not let weak secondary sources silently override stronger primary evidence.

## Quick mode

`--quick` means answer the question with the minimum strong evidence required.

The research sub-agent should:
- prefer one or two authoritative sources when sufficient;
- verify version relevance;
- stop once the question is answered confidently;
- avoid unnecessary source collection;
- keep the persisted report compact;
- avoid exploring edge cases that do not affect the requested answer.

Quick does not mean careless.

Do not sacrifice source quality, version correctness, or contradiction handling merely to be fast.

## Deep mode

`--deep` means investigate the topic thoroughly enough to resolve ambiguity and important edge cases.

The research sub-agent should:
- consult multiple primary sources where useful;
- cross-check documentation against first-party source code, specifications, changelogs, or issues when relevant;
- investigate contradictions;
- capture meaningful edge cases and limitations;
- distinguish version-specific behavior;
- identify unresolved questions;
- stop when additional sources no longer materially improve the answer.

Deep research must still be selective. Do not read or summarize sources that do not improve the conclusion.

## Version awareness

For versioned or evolving technologies, verify that each important source applies to the relevant version.

Examples include:
- framework versions;
- language versions;
- runtime versions;
- operating system versions;
- library/package versions;
- API versions.

Do not silently combine evidence from materially different versions.

When historical evidence is useful:
- identify its version/date;
- explain why it remains relevant;
- do not present obsolete behavior as current.

## Freshness

For changing technologies, check publication/update dates and release relevance.

Old sources are acceptable when they remain authoritative for the requested version or behavior.

Do not assume newer always means more correct, but do not present stale documentation as current without verification.

## Contradicting sources

Never silently choose one conflicting source.

When sources disagree:
- identify the conflict;
- compare authority, version, and freshness;
- explain which source is better supported and why;
- state when the contradiction remains unresolved.

Do not manufacture certainty when authoritative evidence is insufficient.

## Confirmed, inference, unresolved

Distinguish between:

### Confirmed

Directly supported by authoritative evidence.

### Inference

A conclusion derived from multiple pieces of evidence but not stated directly by a source.

### Unresolved

The available evidence is insufficient, ambiguous, contradictory, or version-dependent in a way that cannot be resolved confidently.

These do not need to become labels on every sentence. Use them where the distinction materially affects understanding.

## Research sub-agent

Delegate research to an isolated sub-agent.

The sub-agent may consume substantial source context, but its parent-facing handoff must remain compact.

Preferred handoff:

```text
Conclusion:
<concise answer>

Important findings:
<only decisive findings>

Contradictions:
<only meaningful conflicts>

Unresolved:
<only remaining uncertainty>

Report:
<path to persisted Markdown>
```

Do not return:
- browsing transcripts;
- page-by-page summaries;
- full source excerpts;
- discarded search paths;
- repeated citations;
- implementation details unrelated to the research question.

## Persistence

Persist every research result under:

```text
.research/<descriptive-topic>.md
```

Use a concise descriptive slug.

Examples:

```text
.research/aspnet-idempotency.md
.research/nuxt-server-route-caching.md
.research/xkb-altgr-behavior.md
```

Both `--quick` and `--deep` persist results. The difference is research depth, not persistence.

## Existing research

Before creating a new research file, check for an existing file covering the same topic.

If matching research exists:
- update it when clearly appropriate;
- preserve still-valid prior conclusions;
- replace outdated or contradicted claims;
- retain useful historical/version context when relevant.

Do not create duplicate research files unnecessarily.

Do not overwrite unrelated research.

## Report structure

Keep the Markdown artifact concise and source-backed.

Preferred structure:

```md
# <Research topic>

## Conclusion
...

## Findings
...

## Version / Compatibility
...

## Unresolved
...

## Sources
...
```

Omit sections that add no value.

Do not add headings merely to fill a template.

## Citations

Cite factual claims to the sources that support them.

Prefer direct links/references to:
- official documentation;
- specifications;
- first-party source code;
- changelogs;
- official issues/releases.

Avoid citation clutter. Cite enough to make important claims auditable without attaching a citation to every trivial sentence.

## No implementation creep

This skill is research-only.

It may state a concise evidence-backed direction when directly supported by the research.

It must not:
- modify application/source code;
- produce file-by-file implementation steps;
- create a migration plan;
- implement a fix;
- refactor the repository;
- turn the research artifact into an implementation plan.

Use `plan-implementation` or `delegate-task` for follow-up implementation work.

## Failure behavior

If strong evidence is unavailable:
- say so;
- record the uncertainty;
- do not pad the report with weak sources merely to produce an answer;
- do not convert speculation into fact.

A smaller honest report is better than a confident unsupported one.

## Repository modification

The only persistent repository changes made by this skill are files under:

```text
.research/
```

Protect unrelated user work.

Do not commit, push, create/delete branches, amend, rebase, or perform other Git-history mutations.

Never reset, stash, clean, checkout over, or overwrite unrelated uncommitted changes.

## Token efficiency

- Use an isolated research sub-agent.
- Keep parent handoffs compact.
- Prefer high-authority sources over broad source collection.
- Stop when additional sources no longer materially improve confidence.
- Do not forward full source context to the parent.
- Do not include investigation history in the final report.
- Reuse existing research when appropriate.
- Do not research implementation details unless required to answer the actual question.
