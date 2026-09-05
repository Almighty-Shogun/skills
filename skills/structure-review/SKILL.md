---
name: structure-review
description: Review a repository or scoped area for structural and architectural problems and improvement opportunities. Use repository structure, meaningful Git history, dependency boundaries, and existing architectural decisions to produce an evidence-based Markdown review. Analysis only; never implement changes.
argument-hint: "[--scope <scope>]"
---

# Structure Review

Use this skill to review the structural and architectural quality of a repository or a requested scope within it.

The skill looks for:
- architectural problems;
- structural improvement opportunities;
- coupling and change amplification;
- poor locality or cohesion;
- dependency-direction problems;
- boundary leaks;
- difficult-to-test structures;
- public-contract pressure caused by architecture.

It does not perform general linting, style review, bug hunting, or implementation.

## Invocation

```text
structure-review
structure-review --scope <scope>
```

Natural-language scope is also valid:

```text
Review the structure around authentication.
```

If no scope is supplied or clearly implied, review the repository as a whole.

`--scope` controls where recommendations are targeted, not an absolute filesystem sandbox. Inspect dependencies, consumers, or adjacent modules outside the scope when needed to prove a structural issue.

## Core workflow

```text
Repository reconnaissance
        ↓
Architecture docs / ADRs
        ↓
Meaningful Git history
        ↓
Structural hotspots
        ↓
Isolated area sub-agents
        ↓
Evidence-backed findings
        ↓
Challenge / deduplicate
        ↓
Rank findings
        ↓
STRUCTURE_REVIEW.md
```

## Repository reconnaissance

Inspect:
- repository/package/workspace structure;
- dependency boundaries;
- public APIs and cross-module contracts;
- tests and test setup;
- relevant architecture documentation;
- ADRs and design notes;
- meaningful Git history;
- repeated cross-module change patterns;
- areas with recurring structural friction.

Ignore generated/vendor/build noise unless directly relevant, including:
- `node_modules/`;
- `vendor/`;
- `dist/`;
- `build/`;
- generated source;
- lockfiles;
- generated migrations;
- third-party source;
- repository-specific generated artifacts.

## Git history

Use Git history as a prioritization signal, not proof.

Prefer areas that repeatedly:
- change together;
- require coordinated edits;
- appear in bug fixes and feature work;
- cause repeated boundary crossing;
- accumulate architectural workarounds.

Do not treat churn alone as an architectural problem.

Where practical, distinguish meaningful repeated changes from:
- renames;
- formatting-only changes;
- generated updates;
- version bumps;
- one-off migrations;
- mechanical repository moves.

Stable code should not be redesigned merely because it has not changed recently. If an area is understandable, testable, cohesive, and causes no demonstrated friction, do not invent improvements.

## Existing architectural decisions

Before challenging a structure, inspect relevant:
- ADRs;
- `ARCHITECTURE.md`;
- README files;
- CONTRIBUTING documentation;
- AGENTS / CLAUDE-style repository instructions;
- domain/design notes;
- comments that document compatibility or structural constraints.

Do not relitigate intentional architectural decisions without evidence.

An intentional decision may still be worth revisiting when the original constraints no longer apply. In that case:
- cite the existing decision;
- explain what changed;
- classify the finding conservatively, usually `Worth considering`.

## Structural evidence

Every finding requires concrete evidence of actual structural friction.

Useful evidence includes:
- repeated edits across unrelated modules for one conceptual change;
- dependency cycles;
- lower-level modules depending on higher-level concerns;
- package boundaries that are routinely bypassed;
- internal implementation details leaking through public APIs;
- duplicated domain rules across boundaries;
- difficult test setup caused by structural coupling;
- simple changes requiring knowledge of many unrelated modules;
- shallow abstractions whose interface is nearly as complex as their implementation;
- responsibilities split so widely that locality is poor;
- tightly coupled modules pretending to be independent;
- unnecessary abstractions or boundaries increasing coordination cost.

Do not infer structural problems from simplistic heuristics such as:
- large file = bad;
- many dependencies = bad;
- many classes = bad;
- few interfaces = bad;
- repository pattern/CQRS/DDD/etc. = automatically better.

Prefer demonstrated cohesion, locality, coupling, and development friction over pattern preference.

## Architectural problem vs improvement opportunity

Every finding must be classified as one of:

### Architectural problem

The current structure causes demonstrated friction, coupling, poor testability, boundary violations, change amplification, or similar structural harm.

### Improvement opportunity

The current structure is not necessarily wrong, but evidence suggests a structural change would materially improve future development.

Do not present preferences as problems.

## Confidence

Rank findings using exactly:

### Recommended

Strong evidence supports the finding and the suggested structural direction.

### Worth considering

There is meaningful evidence, but trade-offs or uncertainty remain.

### Speculation

There may be something worth revisiting, but current evidence is insufficient for a stronger recommendation.

Speculation must read as tentative. Do not present speculative findings as immediate work.

## Impact

When justified, optionally classify impact as:

```text
High
Medium
Low
```

Confidence and impact are separate.

A high-impact finding may still be speculative. A low-impact finding may still be strongly recommended.

Do not guess impact when evidence is insufficient.

## Sub-agent strategy

Use reconnaissance to identify distinct structural areas worth deeper investigation.

Then delegate independent areas to isolated sub-agents where practical:

```text
Area A → sub-agent A
Area B → sub-agent B
Area C → sub-agent C
```

Each sub-agent should receive only the context relevant to its area.

Preferred handoff:

```text
Area:
Type:
Evidence:
Impact:
Suggested direction:
Confidence:
Relevant files/modules:
Compatibility/public-contract effect:
```

Do not return:
- exploration transcripts;
- long commit histories;
- raw dependency dumps;
- generic architecture theory;
- unrelated repository observations;
- file-by-file implementation plans.

## Challenge findings before accepting them

Before including a candidate in the final report, perform a lightweight sanity check:

- Is this demonstrated structural friction or merely a preferred design?
- Is the evidence proportional and decisive?
- Is an existing architectural decision being ignored?
- Is the suggested direction based on actual need?
- Would simplification be better than adding another abstraction?
- Is the finding duplicated at a different abstraction level?

Reject or downgrade findings that fail this check.

## Deduplication

Do not pad the report with overlapping recommendations.

If several findings are symptoms of one broader structural problem, consolidate them when that produces a clearer and more accurate recommendation.

For example, do not separately report:
- authentication is too coupled;
- `AuthService` has too many responsibilities;
- session and persistence are tightly linked;

when the evidence shows these are manifestations of one underlying structural boundary problem.

## Dependency direction

Explicitly inspect for:
- circular dependencies;
- lower-level layers importing higher-level concerns;
- public modules depending on internal implementation details;
- cross-package knowledge leakage;
- boundary bypasses;
- dependency relationships that make isolated testing difficult.

Dependency count alone is not evidence of bad architecture.

## Public API and compatibility impact

If a structural recommendation would affect:
- a public API;
- a package contract;
- a serialized format;
- a database boundary;
- a plugin interface;
- an external integration;

note that impact concisely.

Example:

```text
Compatibility:
Would require a breaking public API change.
```

Do not design the migration or implementation. That belongs in `plan-implementation`.

## Suggested direction

Each finding may contain a short structural direction.

Good:

```text
Consolidate shared authorization policy behind one boundary while preserving
the existing storage boundary.
```

Too detailed:

```text
Create IAuthorizationPolicy.cs, move X/Y/Z, update DI registrations, then...
```

Do not turn the review into an implementation plan.

Prefer simplification when it solves the demonstrated problem. Removing an unnecessary abstraction or boundary is as valid as introducing one.

Do not recommend speculative abstractions for hypothetical future needs.

## Existing strengths

Do not create a generic "what is good" section.

Mention an existing strength only when it matters to a recommendation, such as a boundary that should be preserved.

## Report

Always write the final review to:

```text
STRUCTURE_REVIEW.md
```

If a matching review already exists:
- update it only when clearly appropriate and safe;
- otherwise preserve the existing file and use a unique descriptive filename.

Do not blindly overwrite unrelated review content.

If no meaningful structural problems or opportunities are found, it is valid to produce a very small report stating that no evidence-backed recommendations were identified.

Never manufacture findings to fill the report.

## Finding format

Keep findings compact and evidence-driven.

Recommended shape:

```md
## <Short structural finding>

Type: Architectural problem | Improvement opportunity
Confidence: Recommended | Worth considering | Speculation
Impact: High | Medium | Low

<One or two short paragraphs explaining the structural issue or opportunity and
the decisive evidence.>

Relevant areas:
- `path/or/module`
- `path/or/module`

Suggested direction:
<One short architectural direction.>

Compatibility:
<Only when materially relevant.>
```

Do not include:
- investigation transcripts;
- pages of Git history;
- exhaustive dependency inventories;
- generic architecture explanations;
- implementation steps;
- style-only findings;
- naming/formatting concerns;
- minor local duplication unless it creates broader structural friction.

## Repository modification

This skill is analysis-only.

Do not:
- refactor code;
- rename files;
- move modules;
- modify tests;
- change dependencies;
- create implementation branches.

The only persistent repository change is the Markdown review file.

Protect unrelated user work. Never reset, stash, clean, checkout over, or overwrite unrelated uncommitted changes.

## Token efficiency

- Use Git history and repository structure to focus investigation.
- Delegate distinct structural areas to isolated sub-agents.
- Keep sub-agent handoffs compact.
- Prefer decisive evidence over exhaustive inventories.
- Do not forward one sub-agent's full exploration to another.
- Deduplicate findings before writing the report.
- Do not generate HTML, Mermaid, diagrams, or visual reports unless explicitly requested.
- Do not load unrelated repository areas into scoped analysis.
