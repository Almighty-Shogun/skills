---
name: bughunt
description: Proactively inspect a repository for genuine, reproducible bugs. Verify each credible candidate independently, then either write concise local bug reports or prepare GitHub issues for explicit user approval. Never fix bugs or research solutions.
argument-hint: "[--report | --github] [--scope <scope>] [--limit <n>]"
---

# Bughunt

Use this skill to proactively search a repository, or a requested scope within it, for real bugs.

Bughunt is discovery and verification only.

It must:
- inspect the repository for plausible bug candidates;
- cheaply reject obvious false positives before deeper work;
- verify credible candidates with reproducible evidence;
- isolate verification work across sub-agents when multiple candidates exist;
- report only verified bugs;
- avoid solution research and implementation.

It must not fix bugs.

## Invocation

Supported modes:

```text
bughunt --report
bughunt --github
```

Optional:

```text
--scope <scope>
--limit <n>
```

Examples:

```text
bughunt --report
bughunt --github --scope src/auth
bughunt --report --scope packages/common --limit 5
```

Natural-language scope is also valid:

```text
Use bughunt --report and focus on authentication.
```

If neither `--report` nor `--github` is supplied, ask the user which mode they want **before analyzing the repository**.

If both are supplied, ask the user to choose one before proceeding.

`--limit <n>` means the maximum number of **verified findings returned**. It must not mean "stop after investigating N candidates."

If no scope is supplied or clearly implied, inspect the repository as a whole.

## Core workflow

```text
Repository reconnaissance
        ↓
Plausible bug candidates
        ↓
Cheap filtering / known-bug checks
        ↓
One isolated verification sub-agent per credible candidate
        ↓
VERIFIED / REJECTED / INCONCLUSIVE
        ↓
Deduplicate related verified findings
        ↓
--report or --github output
```

Do not use one large verification context for several unrelated candidates when they can be isolated.

## Repository reconnaissance

Inspect the relevant repository or requested scope for plausible defects.

Useful sources include:
- source code;
- tests;
- configuration;
- error handling;
- boundary conditions;
- state transitions;
- concurrency-sensitive code;
- parsing and serialization;
- validation;
- lifecycle behavior;
- platform-specific code;
- API contracts;
- TODOs and known limitations;
- existing GitHub issues when `--github` is used or GitHub access is otherwise appropriate.

A suspicious-looking implementation is only a candidate.

Do not report a bug merely because code:
- looks unsafe;
- seems inconsistent;
- might race;
- could return null;
- appears unusual;
- differs from a preferred style.

## Cheap candidate filtering

Before assigning a full verification sub-agent, cheaply determine whether the candidate is worth deeper investigation.

Check whether:
- the suspected behavior is actually expected;
- an existing test already proves the behavior correct;
- the code path is unreachable;
- the behavior is explicitly documented as a supported limitation;
- an existing GitHub issue or TODO already documents the same defect;
- several candidates are clearly duplicate symptoms of the same underlying defect.

Known defects may still be real bugs, but do not present them as newly discovered.

When an existing issue describes the same bug:
- open issue: treat it as already known;
- closed issue: determine whether the current behavior appears to be a regression before deciding how to report it.

Avoid spending a full verification sub-agent on a candidate already proven irrelevant.

## Verification sub-agents

Assign each credible independent candidate to its own sub-agent where practical.

For example:

```text
Candidate A → verification sub-agent A
Candidate B → verification sub-agent B
Candidate C → verification sub-agent C
```

Each verification agent receives only the context necessary for its candidate.

The verification agent must determine whether the candidate is genuinely incorrect.

Prefer evidence such as:
- a repeatable failing test;
- a minimal reproduction;
- a deterministic command or request;
- a clear expected-vs-observed behavior mismatch;
- a violated documented contract, specification, or runtime invariant;
- a deterministic code path proving an incorrect result.

Where useful, minimize the reproduction.

Intermittent bugs do not need to reproduce 100% of the time. If behavior is repeatedly demonstrable but flaky, record the conditions and observed reproduction rate rather than rejecting it solely for being intermittent.

Environment details should be captured only when they materially affect reproduction.

## Verification outcomes

Every candidate must end in exactly one internal state:

### VERIFIED

There is sufficient evidence that the behavior is genuinely incorrect.

Only verified findings become bug reports or proposed GitHub issues.

### REJECTED

Evidence shows the suspected behavior is correct, expected, irrelevant, unreachable, already accounted for, or otherwise not a bug.

Do not include individual rejected candidates in normal output.

### INCONCLUSIVE

There is not enough evidence to prove or reject the candidate.

Failure to prove a bug does not prove that no bug exists.

Do not report inconclusive candidates as bugs.

A final compact summary may include counts such as:

```text
12 candidates investigated: 4 verified, 6 rejected, 2 inconclusive.
```

If an inconclusive candidate is unusually concerning, mention it briefly to the user without presenting it as a verified bug.

## Verification handoff

Verification sub-agents must return only a compact handoff.

Preferred shape:

```text
Status: VERIFIED | REJECTED | INCONCLUSIVE

Title:
<short candidate title>

Behavior:
<what happens>

Expected:
<what should happen>

Reproduction:
<minimal reproduction>

Evidence:
<only decisive evidence>

Relevant code:
<files/functions/locations>

Root cause:
<only when discovered naturally>
```

Do not return:
- investigation transcripts;
- failed hypotheses;
- raw log dumps;
- lengthy code excerpts;
- speculative fixes;
- solution comparisons;
- unrelated repository observations.

## Root cause rule

If the root cause becomes apparent naturally while proving the bug, record it concisely.

Do not perform additional investigation solely to discover the root cause.

Do not investigate possible solutions.

Bughunt stops at verified diagnosis.

## No solution research

Do not:
- design fixes;
- compare remediation strategies;
- recommend architecture changes;
- implement code changes;
- add permanent regression tests;
- create fix branches or pull requests.

If the user later wants a fix, another workflow such as `delegate-task` should handle it.

## Temporary verification changes

Verification may require temporary:
- tests;
- scripts;
- instrumentation;
- local config changes;
- reproduction harnesses.

These changes must be cleaned up afterward.

Never:
- reset unrelated work;
- stash unrelated work;
- checkout over user changes;
- run destructive clean operations against unrelated files;
- overwrite existing uncommitted changes.

Track only the temporary changes created by bughunt and remove only those exact changes.

The only persistent repository changes allowed are the report files created by `--report`.

`--github` should leave repository contents unchanged.

## Deduplicate verified findings

Before producing final output, compare verified findings for overlap.

If multiple candidates are merely different symptoms of the same underlying defect, merge them into one finding when that is clearer and more accurate.

Do not create multiple reports or issues for the same defect merely because separate sub-agents discovered it independently.

## Severity

Assign severity only when the evidence supports it.

Prefer:

```text
Low
Medium
High
Critical
```

Base severity on demonstrated impact and realistic reach.

Do not inflate severity based only on theoretical worst cases.

If there is not enough evidence to classify severity confidently, omit it instead of guessing.

## Writing style

Write findings for humans, not as investigation transcripts.

Follow these rules for both local reports and GitHub issues:

- Keep titles short, specific, and centered on the broken behavior.
- Prefer two or three short paragraphs for the body.
- Explain what happens, when it happens, and what should happen instead.
- Include only the minimum reproduction needed to demonstrate the bug.
- Use a short numbered reproduction list when it is clearer than prose.
- Include relevant environment or compatibility details only when they affect reproduction.
- Include the root cause only when it became apparent naturally during verification.
- Do not include rejected hypotheses, investigation history, raw logs, possible fixes, validation narration, or unrelated implementation details.
- Do not add headings, checklists, or boilerplate sections unless they materially improve clarity.
- Do not add tool attribution or generated-by text.
- Do not narrate that the report is concise, focused, scoped, minimal, or well-structured.

## `--report`

When no verified bugs are found:
- do not create an empty report;
- tell the user that no bugs could be verified;
- include only a compact investigated / verified / rejected / inconclusive count when useful.

When exactly one verified bug is found:

```text
BUG_REPORT.md
```

When two or more verified bugs are found:

```text
.bughunt/
├── <descriptive-bug-slug>.md
├── <descriptive-bug-slug>.md
└── ...
```

Use descriptive slugs derived from each bug title.

Example:

```text
.bughunt/session-token-not-invalidated.md
```

Do not create an index file unless the number of reports is large enough that navigation genuinely benefits from one.

### Existing report safety

Do not blindly overwrite:
- `BUG_REPORT.md`;
- existing files in `.bughunt/`.

If a matching report already exists:
- update it only when clearly appropriate and safe;
- otherwise use a unique descriptive filename.

Preserve unrelated existing report content.

## `--github`

For each verified finding:

1. Check existing open and closed GitHub issues for duplicates when GitHub access is available.
2. If an open issue already describes the defect, report that instead of drafting a duplicate.
3. If a closed issue matches and the defect now reproduces again, treat the prior issue as useful regression context.
4. Draft the exact GitHub issue title and body.
5. Determine applicable existing repository labels.
6. Show the exact proposed issue to the user.
7. Require explicit approval before creation.

Verification sub-agents must never create GitHub issues themselves.

### Issue approval

Show the exact content that would be submitted:

```text
Proposed GitHub issue

Title:
<exact title>

Body:
<exact body>

Labels:
<existing labels, if any>
```

For multiple findings, show all proposed issues before posting.

Allow the user to:
- approve all;
- approve individually;
- edit;
- skip.

Do not regenerate approved issue text after approval unless the user requested an edit.

Post exactly the approved title, body, and labels.

Every GitHub issue creation is approval-gated.

## GitHub labels

Reuse existing repository labels when clearly applicable.

Do not invent or automatically create labels.

If the same useful missing label would apply repeatedly across several findings, ask the user once whether they want that label created.

Example:

```text
Suggested missing label: input
Would apply to 3 proposed issues.

Create and apply this label?
```

Creating a label requires explicit user approval.

Do not repeatedly ask about the same missing label.

## Final summary

Keep the parent-facing result compact.

Useful information:
- number of candidates investigated;
- number verified;
- number rejected;
- number inconclusive;
- paths of created reports, or links/numbers of approved GitHub issues;
- brief mention of any important known duplicate or concerning inconclusive finding.

Do not repeat the full bug bodies in the final summary when they already exist in report files or GitHub issues.

## Token efficiency

Treat sub-agents as disposable workers.

- Do not forward one verification agent's full investigation to another.
- Give each verification agent only the candidate-specific context it needs.
- Prefer repository state, tests, commands, and diffs over repeated prose descriptions.
- Keep sub-agent handoffs compact.
- Keep rejected candidates out of the final output.
- Do not research solutions.
- Do not repeat logs that can be referenced or rerun.
- Do not load unrelated repository context into candidate verification.
- Parallelize independent candidate verification when useful and supported.
