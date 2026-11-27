---
name: structure-review
description: >-
  Review a repository or scoped area for structural and architectural problems
  and improvement opportunities. Use repository structure, meaningful Git
  history, dependency boundaries, and existing architectural decisions to
  produce an evidence-based Markdown review. Analysis only; never implement
  changes. Not for bugs (bughunt), PRs (review-pr), or local simplification
  (simplify-code).
argument-hint: "[--scope <scope>]"
---

# Structure Review

Review the structural quality of a repository or a scope within it: coupling
and change amplification, poor locality or cohesion, dependency direction,
boundary leaks, hard-to-test structures, and architecture that pressures public
contracts. Not linting, style review, bug hunting, or implementation. Route a
finding to implement to `plan-implementation` or `delegate-task`, a
public-contract change it flags to `api-design`, and a bug it notices to
`bughunt`.

```text
structure-review [--scope <scope>]
```

Scope may also be natural language ("the structure around authentication");
without one, review the whole repository. Scope targets the recommendations, not
the reading: inspect dependencies, consumers, and adjacent modules outside it
when they prove a structural issue. If the scope resolves to no files or
modules, stop and ask, and write nothing.

## Reconnaissance

Read the repository and package layout, dependency boundaries, public and
cross-module contracts, test setup, and the architecture record: ADRs,
`ARCHITECTURE.md`, READMEs, contributing docs, `AGENTS.md` and `CLAUDE.md`, and
comments documenting structural constraints. Ignore `node_modules`, `vendor`,
build output, generated code, generated migrations, lockfiles, and third-party
source.

Use Git history to prioritize: areas that repeatedly change together, need
coordinated edits, recur in fixes and features, or accumulate workarounds. Churn
alone is not a problem, so discount renames, formatting, generated updates,
version bumps, one-off migrations, and mechanical moves. Stable code that is
understandable, testable, and cohesive gets no invented improvements. Without
Git, or with shallow history, the report says so, skips history-based
prioritization, and never cites stability as evidence.

Do not relitigate an intentional architectural decision without evidence. When
its original constraints no longer hold, cite the decision, say what changed,
and classify the finding conservatively, usually `Worth considering`.

## Evidence

Every finding needs concrete evidence of friction: one conceptual change
requiring edits across unrelated modules, dependency cycles, lower layers
importing higher-level concerns, routinely bypassed package boundaries,
implementation detail leaking through public APIs, domain rules duplicated
across boundaries, test setup made hard by coupling, shallow abstractions as
complex as what they wrap, responsibilities scattered so widely that locality
suffers, or unnecessary boundaries raising coordination cost. Look explicitly
for cycles, inverted dependencies, and public modules depending on internals.

Size heuristics are not evidence: large files, many dependencies or classes, few
interfaces, or the absence of a favourite pattern. Judge cohesion, locality,
coupling, and demonstrated friction, not pattern preference.

## Investigation

Use reconnaissance to find distinct structural areas, then give each
independent area its own sub-agent with only the context it needs; a single
small area may be reviewed directly. Start each with an explicit model and
effort level that fits its task: higher for areas needing deep tracing, lower
for shallow ones. Each hands off:

```text
Area:
Type:
Evidence:
Impact: (only when justified)
Suggested direction: (only when justified)
Confidence:
Relevant files/modules:
Compatibility/public-contract effect:
```

Handoffs carry evidence, not exploration transcripts, commit logs, dependency
dumps, architecture theory, or implementation plans.

Before accepting a finding, ask whether it is demonstrated friction or a
preferred design, whether the evidence is decisive, whether it ignores an
existing decision, and whether simplifying would beat adding an abstraction.
Reject or downgrade what fails. Consolidate findings that are symptoms of one
structural problem: "auth is coupled", "`AuthService` does too much", and
"session and persistence are tightly linked" may be one boundary finding.

## Classification

- **Type**: `Architectural problem` when the structure causes demonstrated
  harm; `Improvement opportunity` when it is not wrong but a change would
  materially help future development. Never present a preference as a problem.
- **Confidence**, exactly one of: `Recommended` (strong evidence for the
  finding and the direction), `Worth considering` (meaningful evidence,
  trade-offs remain), `Speculation` (possibly worth revisiting, evidence
  insufficient; must read as tentative).
- **Impact**, when justified: High, Medium, or Low. Confidence and impact are
  independent; omit impact rather than guess.

Each finding may carry one short structural direction, such as "Consolidate
shared authorization policy behind one boundary while preserving the existing
storage boundary.", never a file-by-file plan. Removing an unnecessary
abstraction is as valid as adding one, and no abstraction is recommended for
hypothetical needs. When a direction would affect a public API, package
contract, serialized format, database boundary, plugin interface, or external
integration, note it (`Would require a breaking public API change.`) without
designing the migration. Mention an existing strength only when a
recommendation should preserve it.

## Report

Write `STRUCTURE_REVIEW.md` at the repository root
(`git rev-parse --show-toplevel`), whatever the scope. It is a working note,
committed only when the user asks. If one exists, update it only when it is
clearly the same review: keep still-valid findings, drop resolved ones, and
preserve unrelated content. Otherwise keep it and write
`STRUCTURE_REVIEW-<scope-slug>.md` at the same root, such as
`STRUCTURE_REVIEW-packages-api.md`. A review that finds nothing evidence-backed
is a short report saying so; never manufacture findings.

```md
Scope: <scope, or whole repository>
Date: YYYY-MM-DD
Commit: <reviewed commit>

## <Short structural finding>

Type: Architectural problem | Improvement opportunity
Confidence: Recommended | Worth considering | Speculation
Impact: High | Medium | Low (only when justified)

<One or two short paragraphs explaining the issue and the decisive evidence.>

Relevant areas:
- `path/or/module`

Suggested direction: (only when justified)
<One short architectural direction.>

Compatibility:
<Only when materially relevant.>
```

No pages of history, dependency inventories, implementation steps, naming or
formatting concerns, or minor local duplication without broader friction. No
HTML, Mermaid, or diagrams unless asked.

## Boundaries

Analysis only: the review file is the only change. No refactors, renames, moved
modules, test or dependency changes, or branches; protect unrelated
uncommitted work.

## Final summary

The finding count per confidence and the report path, without repeating the
report.
