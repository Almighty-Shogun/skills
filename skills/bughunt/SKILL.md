---
name: bughunt
description: Proactively inspect a repository for genuine, reproducible bugs. Verify each credible candidate independently, then either write concise local bug reports or prepare GitHub issues for explicit user approval. Never fix bugs. Research fix options only with --solution, which requires --report and changes no code.
argument-hint: "[--report | --github] [--solution] [--scope <scope>] [--limit <n>]"
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
- avoid solution research and implementation, unless `--solution` is set.

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
--solution
```

Examples:

```text
bughunt --report
bughunt --github --scope src/auth
bughunt --report --scope packages/common --limit 5
bughunt --report --solution --scope packages/common
```

Natural-language scope is also valid:

```text
Use bughunt --report and focus on authentication.
```

If neither `--report` nor `--github` is supplied, determine whether the target is a Git
repository **before analyzing it**:

- when it is not a Git repository, use `--report`;
- otherwise ask the user which mode they want.

If both are supplied, ask the user to choose one before proceeding.

`--solution` adds a `## Fix options` section to each report file. It requires
`--report`, because there is no file to put that section in otherwise. When it is
supplied without `--report`, refuse and say why; do not silently report, and do
not silently fix. It never grants permission to change code.

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
VERIFIED / BY DESIGN / REJECTED / INCONCLUSIVE
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
- differs from a preferred style;
- differs from what you would have written;
- is not mentioned in the documentation;
- is not covered by a test.

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

This is a screen, not the intent check. A candidate that survives it has not been
shown to be unintended; that question belongs to the verification agent, which
has the context to answer it.

## Verification sub-agents

Assign each credible independent candidate to its own sub-agent where practical.

For example:

```text
Candidate A → verification sub-agent A
Candidate B → verification sub-agent B
Candidate C → verification sub-agent C
```

Each verification agent receives only the context necessary for its candidate.

The verification agent must determine two separate things: that the behavior
happens, and that it was not intended. Proving the first says nothing about the
second.

### Prove the behavior

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

### Establish the expectation it violates

A finding is only a bug if it breaks an expectation that something other than the
agent's own judgment sets. Cite that source, in this order of preference:

1. a documented contract, specification, or type/API guarantee;
2. a test asserting the opposite behavior;
3. a stated invariant, assertion, or schema constraint;
4. a caller in the repository that visibly breaks on the actual behavior;
5. an internal contradiction, where two code paths that must agree do not.

"A reasonable user would expect", "this is surprising", and "the obvious intent
is" are not sources. When no source can be cited, the outcome is INCONCLUSIVE,
never VERIFIED.

### Check whether it was deliberate

Before concluding, look for evidence that someone chose this behavior:
- a comment on or near the code explaining it;
- a test asserting exactly the behavior in question;
- a configuration key, flag, or option that selects it;
- the commit or pull request that introduced the exact lines, via `git blame` on
  them and `git log -S` for the relevant string or symbol, where the project is
  a Git repository;
- a changelog, ADR, or release note describing it.

Git history is usually the fastest way to settle this. A line written in a commit
whose message describes precisely this behavior is deliberate.

Some categories are deliberate often enough that they need a violated contract
before they can be reported at all:
- defensive or redundant checks;
- a deliberate fail-open or fail-closed choice;
- a swallowed exception carrying a comment;
- deliberately permissive validation;
- a tolerated race in a cache or in best-effort work;
- a parameter left unused to satisfy an interface or signature.

## Verification outcomes

Every candidate must end in exactly one internal state:

### VERIFIED

The behavior is demonstrated, the expectation it violates is cited from a source
outside the agent's own judgment, and the intent check turned up nothing showing
it was chosen. All three are required.

Only verified findings become bug reports or proposed GitHub issues.

### BY DESIGN

The behavior happens, but evidence shows it was chosen: a comment, a test
asserting it, a flag selecting it, or the commit that introduced it saying so.

Deliberate is not the same as correct, so a by-design finding may still be worth
a sentence to the user when the evidence of intent is thin or the consequence is
severe. It is never written up as a bug report or a GitHub issue.

### REJECTED

Evidence shows the suspected behavior is correct, expected, irrelevant, unreachable, already accounted for, or otherwise not a bug.

Do not include individual rejected candidates in normal output.

### INCONCLUSIVE

There is not enough evidence to prove or reject the candidate.

Failure to prove a bug does not prove that no bug exists.

Do not report inconclusive candidates as bugs.

A final compact summary may include counts such as:

```text
12 candidates investigated: 3 verified, 5 rejected, 2 by design, 2 inconclusive.
```

If an inconclusive candidate is unusually concerning, mention it briefly to the user without presenting it as a verified bug.

## Verification handoff

Verification sub-agents must return only a compact handoff.

Preferred shape:

```text
Status: VERIFIED | BY DESIGN | REJECTED | INCONCLUSIVE

Title:
<short candidate title>

Behavior:
<what happens>

Expected:
<what should happen>

Expectation source:
<which of the five sources sets it, and where it lives>

Intent check:
<what was searched for deliberateness, and what it showed>

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

Do not investigate possible solutions, unless `--solution` is set.

Bughunt stops at verified diagnosis, and with `--solution` at options a reader can
choose between.

## No solution research by default

Do not:
- design fixes;
- compare remediation strategies;
- recommend architecture changes;
- implement code changes;
- add permanent regression tests;
- create fix branches or pull requests.

`--solution` lifts the first two, and only inside the `## Fix options` section of
a report file. See **Fix options**. Everything else on that list still holds, with
or without the flag.

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
- Do not include rejected hypotheses, investigation history, raw logs, validation narration, or unrelated implementation details.
- Do not include possible fixes, except inside the `## Fix options` section that `--solution` adds.
- Do not add headings, checklists, or boilerplate sections unless they materially improve clarity.
- Do not add tool attribution or generated-by text.
- Do not narrate that the report is concise, focused, scoped, minimal, or well-structured.

## `--report`

When no verified bugs are found:
- do not create an empty report;
- tell the user that no bugs could be verified;
- include only a compact investigated / verified / rejected / by design / inconclusive count when useful.

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

With `--solution`, every report file also carries a `## Fix options` section. See
**Fix options**.

### Existing report safety

Do not blindly overwrite:
- `BUG_REPORT.md`;
- existing files in `.bughunt/`.

If a matching report already exists:
- update it only when clearly appropriate and safe;
- otherwise use a unique descriptive filename.

Preserve unrelated existing report content.

## Fix options

Only with `--solution`, and only alongside `--report`.

Append a `## Fix options` section to each report file, after the reproduction and
the severity line, before the file and line list.

Shape:

1. An opening line stating what is **not** at stake, so the reader knows the
   blast radius before reading the options.
2. Two to four options, each led by a bolded phrase naming the strategy, not the
   letter alone: `**Option A, bind the tail.**` The letter is for citing it
   later; the phrase is what makes the list skimmable.
3. Each option states what to change and roughly where, what it buys, and what it
   costs or fails to cover. The cost is not optional.
4. The do-nothing option wherever one exists, usually "keep the behavior and
   correct the documentation instead". A documentation defect wearing a bug
   costume is usually caught here.
5. A `Recommendation:` line picking one, why in a sentence, and the condition
   under which a different option is right instead.
6. A falsification paragraph naming which doc blocks and documentation pages each
   option would make untrue, with line numbers.
7. A trailing answer block for the reader: the line `Desired solution:` followed
   by an empty fenced block containing `// answer`.

The answer is freeform. A letter, or prose describing a different approach
entirely, are both valid. Do not constrain it to a letter.

### Rules

- Options are analysis. `--solution` never edits source, never creates a branch,
  and never implies permission to fix.
- Options stay scoped to the defect. Redesigning the surrounding code is still
  out of scope.
- Do not pad to a count. Two real options beat four where two are filler.
- Options must be genuinely different strategies, not one strategy at three sizes.
- Recommend, do not hedge. Listing options without picking one pushes the work
  back to the reader.
- Say when an option is a deliberate half-measure and what it leaves unfixed.

### Options are not findings

A finding is verified. An option is reasoned.

Every claim about the **current** code holds to the same standard as the finding
itself. Every claim about how a **fix** would behave is a prediction, with no fix
yet to verify it against.

Do not write predictions in the register of verified behavior, and never present
the section as a guarantee that the recommended option is safe. A chosen option
can introduce a regression the option paragraph did not anticipate.

Write the options after verification concludes, never inside a verification
sub-agent, so the verification standard stays uncontaminated by solution work.

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
- number by design;
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
- Keep rejected and by-design candidates out of the final output, beyond their counts.
- Do not research solutions.
- Do not repeat logs that can be referenced or rerun.
- Do not load unrelated repository context into candidate verification.
- Parallelize independent candidate verification when useful and supported.
