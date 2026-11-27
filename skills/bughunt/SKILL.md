---
name: bughunt
description: >-
  Proactively inspect a repository for genuine, reproducible bugs. Verify each
  credible candidate independently, then either write concise local bug reports
  or prepare GitHub issues for explicit user approval. Never fix bugs. Research
  fix options only with --solution, which implies --report and changes no code.
argument-hint: "[--report | --github] [--solution] [--scope <scope>] [--limit <n>]"
---

# Bughunt

Search a repository, or a scope within it, for real bugs; verify them; report
only the verified ones. Bughunt discovers and verifies. It never fixes.

Read every `AGENTS.md` and `CLAUDE.md` that applies and follow them; they
override this skill's defaults, and when two conflict, ask which wins. Give each
sub-agent the rules it needs.

## Invocation

```text
bughunt [--report | --github] [--solution] [--scope <scope>] [--limit <n>]
```

- Scope may also be given in natural language ("focus on authentication").
  Without one, inspect the whole repository.
- Inspect the working tree. Uncommitted lines are the user's work in progress
  and have no history for the intent check: skip candidates that live only in
  them, and say so in the summary.
- `--github` needs a Git repository with a GitHub remote that an authenticated
  `gh` can reach (see `--github`). Without one, refuse an explicit `--github` up
  front with the reason; with none of `--report`, `--github`, or `--solution`,
  use `--report`. Otherwise, with none of the three, ask which mode. Both
  `--report` and `--github` supplied: ask which one.
- `--limit <n>` caps the number of **verified findings returned**, not the
  number of candidates investigated; `n` must be an integer of at least 1.
  Verify every credible candidate, keep the `n` highest-severity findings
  (unrated ones last, ties broken by reach, then by reproduction strength), and
  report how many were omitted. Only kept findings get report files or drafts.
  Under `--github`, open duplicates are reported, not drafted, and do not count
  toward `n`.
- `--solution` adds a `## Fix options` section to each report file, so it
  implies `--report`. This rule takes precedence: with `--github`, ask whether
  to switch to `--report`; otherwise run `--github` without fix options. It
  never grants permission to change code.

## Candidates

Look through source, tests, configuration, error handling, boundaries, state
transitions, concurrency, parsing and serialization, validation, lifecycle and
platform-specific code, API contracts, TODOs, and, with GitHub access, existing
issues.

A suspicious line is only a candidate. Code that looks unsafe, inconsistent,
racy, unusual, nullable, undocumented, untested, or unlike what you would have
written is not a bug on that basis.

Screen each candidate cheaply before deeper work. Drop it when the behavior is
expected, a test already proves it correct, the path is unreachable, or it is a
documented limitation. Merge duplicate symptoms of one defect. A defect an open
issue or TODO already records is known, not new; for a closed issue, check
whether this is a regression. The screen is not the intent check: a surviving
candidate has not yet been shown to be unintended.

## Verification

Give each credible, independent candidate its own verification sub-agent.
Brief it with the candidate, the scope, the verification standard below, the
outcomes, the Handoff template, and the temporary-change and no-fix rules. Run
in parallel, at most four at once, only agents that change no repository file;
run the others one after another. Start each with an explicit model and effort
level that fits its task: highest for verification, lower for simple read-only
research. When sub-agents cannot be started, verify inline, settle every
verdict before writing any fix options, and say in the summary that
verification was not independent.

The agent establishes three things: that the behavior happens, that it violates
a cited expectation, and that nobody intended it.

**Prove the behavior** with a repeatable failing test, a minimal reproduction, a
deterministic command or request, or a code path that provably yields the wrong
result. A flaky bug still counts: record the conditions and the observed
reproduction rate. Capture environment details only when they affect
reproduction.

**Cite the expectation it violates**, from a source other than the agent's own
judgment, in this order of preference: a documented contract, specification,
or type guarantee; a test asserting the opposite; a stated invariant, assertion,
or schema constraint; a caller in the repository that visibly breaks; two code
paths that must agree and do not. "A reasonable user would expect" is not a
source. With no source, the outcome is INCONCLUSIVE.

**Check for deliberateness**: a comment explaining the behavior, a test
asserting exactly it, a flag or option selecting it, a changelog or ADR, or the
commit that introduced the lines (`git blame`, `git log -S`). A commit whose
message describes precisely this behavior makes it deliberate. Defensive checks,
deliberate fail-open or fail-closed choices, commented swallowed exceptions,
permissive validation, tolerated races in caches or best-effort work, and
parameters unused to satisfy a signature need a violated contract before they
can be reported at all.

### Outcomes

- **VERIFIED**: demonstrated, a cited expectation is violated, and the intent
  check found nothing. All three. Only these become reports or issues.
- **BY DESIGN**: it happens, but evidence shows it was chosen. Never written up;
  worth a sentence to the user only when the evidence of intent is thin or the
  consequence severe. When it violates a documented contract, mention it to the
  user as a documentation mismatch.
- **REJECTED**: correct, expected, unreachable, or otherwise not a bug.
- **INCONCLUSIVE**: not enough evidence either way. Never reported as a bug;
  mention it briefly if it is unusually concerning.

### Handoff

```text
Status: VERIFIED | BY DESIGN | REJECTED | INCONCLUSIVE
Title: <short title>
Behavior: <what happens>
Expected: <what should happen>
Expectation source: <which source, and where it lives>
Intent check: <what was searched, and what it showed>
Reproduction: <minimal reproduction>
Evidence: <decisive evidence only>
Impact: <observed consequence; who is affected and how often>
Relevant code: <files, functions>
Root cause: <only when found while proving the bug>
Temporary changes: removed | left: <paths>
```

Record a root cause only when it surfaced naturally; do not investigate further
to find one. Handoffs carry conclusions, not transcripts, failed hypotheses,
logs, long excerpts, or fixes.

### Temporary changes

Verification may add temporary tests, scripts, instrumentation, config, or
harnesses. Keep scratch files outside the repository where possible. Each
verification agent reverts only its own hunks before handing off, and never
uses checkout, restore, stash, or reset on a file with pre-existing changes.
The parent removes anything a handoff reports as left. The only persistent
change is the report files from `--report`; `--github` leaves the repository
untouched. Protect unrelated uncommitted work.

## Findings

Merge verified findings that are symptoms of one defect, including ones found
independently by separate agents. Assign severity (Low, Medium, High, Critical)
from demonstrated impact and realistic reach, never from a theoretical worst
case; omit it when the evidence cannot support one.

Write for humans. Each report has, in order: a short title naming the broken
behavior; a `Severity:` line when one was assigned; two or three short
paragraphs on what happens, when, and what should happen instead; the minimum
reproduction (a short numbered list when clearer); the expectation source; the
relevant files and symbols; then `## Fix options` under `--solution`. Leave out
hypotheses, investigation history, logs, unrelated implementation detail, and
fixes (except under `## Fix options`). Add no boilerplate headings,
attribution, or commentary on the report's own qualities.

Bughunt does not design fixes, recommend architecture changes, implement code,
add permanent tests, or open branches or pull requests. `--solution` lifts only
the first, and only inside `## Fix options`. A later fix belongs to another
workflow such as `delegate-task`.

## `--report`

Report files go at the repository root. They are working notes, committed only
when the user asks.

- No verified bugs: create no file; say so, with a compact count of
  investigated, verified, rejected, by-design, and inconclusive candidates.
- One: `BUG_REPORT.md`.
- Several: `.bughunt/<descriptive-slug>.md` each, such as
  `.bughunt/session-token-not-invalidated.md`. Every file in `.bughunt/` is a
  report; add no index.

Never blindly overwrite an existing `BUG_REPORT.md` or `.bughunt/` file: update
it only when it is clearly the same report, otherwise use a unique name, and
preserve unrelated report content. When `BUG_REPORT.md` holds an unrelated
report, a single report goes to `.bughunt/<descriptive-slug>.md`. Never
rewrite an answered `Desired solution:` block: when the fix options would
change, leave the options and the answer as they are and tell the user the
report needs a new answer, and why.

## Fix options

Only with `--solution`. Write them after verification concludes, never inside a
verification agent, so solution work cannot contaminate the verdict. Append
`## Fix options` at the end of each report:

1. An opening line stating what is **not** at stake, so the reader knows the
   blast radius first.
2. One to four genuinely different strategies, not counting the do-nothing
   option, each led by a bolded phrase naming it:
   `**Option A, bind the tail.**`
3. For each: what to change and roughly where, what it buys, and what it costs
   or leaves uncovered. The cost is required; say when an option is a
   deliberate half-measure.
4. The do-nothing option where one exists, usually "keep the behavior and
   correct the documentation". A documentation defect posing as a bug is
   usually caught here.
5. `Recommendation:` picking one, why in a sentence, and when a different
   option would be right instead.
6. When any option would make documentation untrue, a paragraph naming the doc
   blocks and documentation pages each such option would make untrue, with
   line numbers.
7. The line `Desired solution:` followed by a fenced block containing only the
   placeholder `// answer`. The reader's answer is freeform: a letter or a
   different approach entirely.

Stay scoped to the defect and do not pad to a count. Claims about the current
code meet the finding's standard; claims about how a fix would behave are
predictions, so never write them as verified behavior or present the
recommendation as guaranteed safe.

## `--github`

Resolve the target repository first. `<origin-slug>` is the `owner/name` of
`origin`. Run
`gh repo view <origin-slug> --json isFork,parent,hasIssuesEnabled`: when
`origin` is a fork, `<slug>` is the parent's `owner/name`; else it is
`<origin-slug>`. Pass `--repo <slug>` to every `gh issue` and `gh label` call.
When `gh` is not authenticated, or issues cannot be read or created in
`<slug>`, stop and report why.

For each verified finding, check open and closed issues for duplicates. An open
duplicate is reported instead of drafted; a matching closed issue is regression
context. Draft the exact title and body, pick applicable existing labels, and
show every proposed issue before creating any:

```text
Proposed GitHub issue

Title:
<exact title>

Body:
<exact body>

Labels:
<existing labels, if any>
```

The user may approve all, approve individually, edit, or skip. Post exactly the
approved text; regenerate it only when an edit is requested. Creation is always
approval-gated, and verification agents never create issues.

A verified security vulnerability is never drafted as a public issue. Flag it,
propose a private vulnerability report or security advisory instead, and
exclude it from "approve all".

Reuse existing labels and never create one unasked. When the same missing label
would apply across several findings, ask once whether to create it.

## Final summary

Counts per outcome and, when `--limit` is given, the count it omitted, report
paths or approved issue links, and a brief note of any known duplicate or
concerning inconclusive candidate. Rejected and by-design candidates appear only
as counts, except a by-design candidate with thin evidence of intent, a severe
consequence, or a documentation mismatch, which gets one sentence. Do not repeat
report bodies that already exist in files or issues.
