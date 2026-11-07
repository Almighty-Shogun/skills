---
name: simplify-code
description: Simplify existing code while preserving behavior. Supports --scope and an analysis-only --report mode, uses isolated sub-agents for independent areas, respects historical intent and public contracts, validates changes independently, and never commits or pushes automatically.
argument-hint: "[--scope <scope>] [--report]"
disable-model-invocation: true
---

# Simplify Code

Simplify existing code without changing its behavior. Without `--report`,
invoking this skill is permission to apply high-confidence simplifications in
the resolved scope; with `--report`, nothing is changed and the findings go to
`SIMPLIFICATION_REPORT.md`.

```text
simplify-code [--scope <scope>] [--report]
```

## Scope

Use `--scope` or a clearly implied scope ("simplify the authentication code").
Otherwise prefer the uncommitted diff, then the branch's changes against its
base; if neither gives a meaningful target, ask. Never silently simplify the
whole repository. Scope targets the changes, not the reading: inspect callers,
dependencies, tests, and history outside it when they prove a change is safe.

## What counts

Less nesting, clearer control flow, removed redundant branches and proven dead
code, duplication consolidated where that reads better, better names, simpler
conditionals and type machinery, pointless wrappers or abstractions removed,
clever code made plain, and scattered responsibilities brought together. Fewer
lines is not the measure: dense one-liners, clever tricks, abstraction for line
count, removed useful boundaries, and patterns without demonstrated need are not
simplifications.

## Behavior preservation

Preserve inputs and outputs, side effects and their order, exceptions, public
contracts, serialization formats, externally visible timing, meaningful
performance, and framework integration. Existing tests should still pass
unchanged; needing to change a test is a sign behavior is changing, so never
edit tests to make a simplification pass.

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

Give each independent area its own sub-agent, with related changes kept
together, and pass only the relevant code, the callers and tests needed, the
history that establishes intent, and repository constraints:

```text
Area:
What was simplified:
Why it is simpler:
Behavior-preservation evidence:
Files changed:
Checks run:
Anything intentionally left unchanged:
```

Then a fresh validation sub-agent checks the actual diff against the original
behavior, the existing tests, the relevant build, lint, and typecheck commands,
public contracts, side effects, exceptions, and performance-sensitive paths,
without relying on the implementer's account. A concrete regression gets one
targeted fix and another validation; avoid open-ended repair loops.

Changes stay in the working tree: no commits, pushes, branches, or history
edits, and unrelated uncommitted work is protected. Committing goes through the
`commit` skill.

## `--report`

```text
1. Flatten nested validation flow
   Why: three nested branches obscure a single failure path.
   Area: src/...
   Direction: use early returns while preserving existing exceptions.
```

Show before and after code only when an unusual case needs it. If the report
file exists, update it only when it is clearly the same report; otherwise use a
unique descriptive name.

## Final summary

```text
Simplifications applied: 3
Areas left unchanged due to uncertainty: 1
Validation: passed
```

For `--report`, the path and a short summary. Stop when the remaining candidates
are subjective or low-value.
