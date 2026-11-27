---
name: simplify-code
description: >-
  Simplify existing code while preserving behavior. Supports --scope and an
  analysis-only --report mode, uses isolated sub-agents for independent areas,
  respects historical intent and public contracts, validates changes
  independently, and never commits or pushes automatically. Use only when the
  user or repository instructions explicitly ask for it.
argument-hint: "[--scope <scope>] [--report]"
---

# Simplify Code

Simplify existing code without changing its behavior. Without `--report`,
invoking this skill is permission to apply high-confidence simplifications in
the resolved scope; with `--report`, nothing is changed and the findings go to
`SIMPLIFICATION_REPORT.md`.

Read every `AGENTS.md` and `CLAUDE.md` that applies and follow them; they
override this skill's defaults, and when two conflict, ask which wins. Give each
sub-agent the rules it needs.

```text
simplify-code [--scope <scope>] [--report]
```

## Scope

Use `--scope` or a clearly implied scope ("simplify the authentication code").
Otherwise prefer the uncommitted changes (`git diff HEAD` plus the untracked
files from `git status`), then the branch's changes against
`git merge-base origin/<default-branch> HEAD`; on the default branch or a
detached HEAD, or if neither gives a meaningful target, ask. Never silently
simplify the whole repository. Scope targets the changes, not the reading:
inspect callers, dependencies, tests, and history outside it when they prove a
change is safe.

## What counts

Less nesting, clearer control flow, removed redundant branches and proven dead
code, duplication consolidated where that reads better, better names, simpler
conditionals and type machinery, pointless wrappers or abstractions removed,
clever code made plain, and scattered responsibilities brought together. Fewer
lines is not the measure: dense one-liners, clever tricks, abstraction for line
count, removed useful boundaries, and patterns without demonstrated need are not
simplifications.

Make one pass over the scope; leave the remaining candidates when they are
subjective or low-value.

## Behavior preservation

Preserve inputs and outputs, side effects and their order, exceptions, public
contracts, serialization formats, externally visible timing, meaningful
performance, and framework integration. Before editing, record which checks
already fail; existing tests should still pass unchanged, judged against that
baseline. Needing to change a test is a sign behavior is changing, so never
edit tests to make a simplification pass. Test files are never simplified,
even when the scope contains them; they are the preservation evidence.

Apply only changes whose preservation is high-confidence. Leave a candidate
unchanged, and mention it, when it involves subjective trade-offs, unclear
intent, public API or compatibility changes, meaningful performance effects,
uncertain framework behavior, or architectural redesign. Changes to package
boundaries, domain ownership, or dependency direction belong to
`structure-review` or `plan-implementation`.

Understand strange code before removing it, especially workarounds,
compatibility branches, platform-specific paths, framework hooks, defensive
checks, and unusual abstractions: read callers, tests, docs, comments, and the
commits that introduced it. Code is dead only once it is shown unreachable
through reflection, DI, framework conventions, dynamic imports, plugin
registration, serialization, generated bindings, public consumers, runtime
discovery, and configuration-driven loading; absent static references alone may
not be enough.

Keep comments that explain why behavior exists, compatibility constraints,
specifications, workarounds, security, or non-obvious runtime behavior; update
or remove them only when the simplification makes them obsolete.

## Applying changes

Before editing, copy the in-scope files to a scratch directory outside the
repository; that copy is the baseline, including the user's uncommitted work.

Give each independent area its own sub-agent, with related changes kept
together, and pass the relevant code, the callers and tests needed, the history
that establishes intent, repository constraints, the behavior-preservation
rules, the high-confidence threshold, the working-tree-only rule, and the
handoff template. The sub-agents share the working tree, so run them one after
another unless their file sets are disjoint. Each records its edits as a patch
against the files as it found them. A single small area may be simplified
directly, without a sub-agent; the validation below stays independent. Start
each sub-agent with an explicit model and effort level that fits its task:
highest for implementation, fixes, and validation, lower for simple read-only
research. Each implementer hands off:

```text
Area:
What was simplified:
Why it is simpler:
Behavior-preservation evidence:
Files changed:
Checks run:
Anything intentionally left unchanged:
```

Then a fresh validation sub-agent checks the diff against the baseline, plus
any untracked files the implementers created, against the original behavior,
the existing tests, the relevant build, lint, and typecheck commands, public
contracts, side effects, exceptions, and performance-sensitive paths, without
relying on the implementer's account. The validator is read-only: it runs
non-mutating variants only (no `--fix`, no formatters, no installs), compares
`git status` before and after, and reports anything that changed. When
sub-agents cannot be started, validate inline and report that the validation
was not independent.

Attribute each finding to an area through the handoffs' `Files changed`. A
concrete regression, or another finding such as a removed why-comment or a
change that is not simpler, gets one targeted fix by a fresh fix agent that
receives the finding and that area's files, then another validation; avoid
open-ended repair loops. If that validation still fails, revert that area by
reverse-applying its own patch and count the area as left unchanged. Never use
checkout, restore, stash, or reset on a file with pre-existing changes.

Changes stay in the working tree: no commits, pushes, branches, or history
edits, and unrelated uncommitted work is protected. Committing goes through the
`commit` skill; hand it the changed files as paths.

## `--report`

Use the same area split with read-only sub-agents at lower effort. Mark each
candidate as high-confidence or needs judgment.

```text
1. Flatten nested validation flow
   Confidence: high-confidence
   Why: three nested branches obscure a single failure path.
   Area: src/...
   Direction: use early returns while preserving existing exceptions.
```

Show before and after code only when an unusual case needs it. If the report
file exists, update it only when it is clearly the same report; otherwise use a
unique descriptive name.

## Final summary

```text
Simplifications applied: 3 (2 areas)
Left unchanged:
- <candidate> — <reason>
Validation: passed | passed; <n> area(s) reverted | not independent
Changed files: <paths>
```

For `--report`, the path and a short summary.
