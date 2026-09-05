---
name: simplify-code
description: Simplify existing code while preserving behavior. Supports --scope and an analysis-only --report mode, uses isolated sub-agents for independent areas, respects historical intent and public contracts, validates changes independently, and never commits or pushes automatically.
argument-hint: "[--scope <scope>] [--report]"
disable-model-invocation: true
---

# Simplify Code

Use this skill to simplify existing code without changing its intended behavior.

The skill may:
- discover worthwhile simplifications;
- reduce unnecessary complexity;
- improve readability and locality;
- remove redundant or dead code when proven safe;
- simplify control flow, types, and abstractions;
- apply safe changes directly unless `--report` is used.

It must preserve behavior.

## Invocation

Supported forms:

```text
simplify-code
simplify-code --scope <scope>
simplify-code --report
simplify-code --report --scope <scope>
```

Natural-language scope is also valid:

```text
Simplify the authentication code.
```

## Scope resolution

If `--scope` is supplied or clearly implied, use that scope.

If no scope is supplied:

1. Prefer the current uncommitted diff when meaningful.
2. Otherwise prefer current branch changes against the base branch.
3. If neither provides a meaningful target, ask the user which area should be simplified.

Do not silently simplify the entire repository.

`--scope` controls where simplifications are targeted. Inspect adjacent callers, dependencies, tests, or history when necessary to prove a simplification is safe.

## `--report`

`--report` is analysis-only.

In report mode:
- identify worthwhile simplifications;
- explain why they help;
- provide a concise direction;
- do not modify code.

Write the report to:

```text
SIMPLIFICATION_REPORT.md
```

If a matching report already exists:
- update it only when clearly appropriate and safe;
- otherwise preserve it and use a unique descriptive filename.

Do not blindly overwrite unrelated report content.

Without `--report`, invoking this skill is permission to apply high-confidence simplifications within the resolved scope.

## Core workflow

Without `--report`:

```text
Resolve scope
   ↓
Reconnaissance
   ↓
Identify worthwhile simplifications
   ↓
Check historical intent / safety
   ↓
Delegate independent areas
   ↓
Apply high-confidence changes
   ↓
Fresh validation
```

With `--report`:

```text
Resolve scope
   ↓
Reconnaissance
   ↓
Identify worthwhile simplifications
   ↓
Check historical intent / safety
   ↓
SIMPLIFICATION_REPORT.md
```

## What counts as simplification

Valid simplifications include:
- reducing unnecessary nesting;
- using clearer control flow;
- removing redundant branches;
- removing proven dead code;
- consolidating duplication when it improves clarity;
- improving naming;
- simplifying overly complex conditionals;
- simplifying unnecessary type machinery;
- removing pointless wrappers or abstractions;
- replacing clever code with clearer code;
- improving locality when responsibilities are needlessly scattered.

Do not assume fewer lines means better code.

Do not simplify by:
- compressing readable code into dense expressions;
- replacing explicit logic with clever tricks;
- hiding behavior behind abstraction solely to reduce line count;
- removing useful boundaries;
- introducing a design pattern without demonstrated need.

## Behavior preservation

Behavior preservation is mandatory.

Preserve:
- inputs and outputs;
- side effects;
- side-effect ordering;
- exceptions and error behavior;
- public contracts;
- serialization formats;
- externally visible timing/order where relevant;
- meaningful performance characteristics;
- framework/runtime integration behavior.

Existing tests should normally remain valid unchanged.

If simplifying code appears to require changing tests, treat that as a warning that behavior may be changing.

Do not change tests merely to make a simplification pass.

## Confidence threshold

Without `--report`, apply only simplifications where behavior preservation is high-confidence.

If a candidate involves:
- subjective trade-offs;
- unclear historical intent;
- public API changes;
- architectural redesign;
- compatibility changes;
- meaningful performance changes;
- uncertain framework behavior;

leave it unchanged and mention it rather than guessing.

Such candidates may belong in `structure-review`, `plan-implementation`, or another workflow.

## Architecture boundary

This skill handles local or contained simplification.

It may remove an obviously unnecessary local abstraction when safety is clear.

It must not turn into broad architectural redesign.

If simplification requires changing package boundaries, public architecture, domain ownership, or major dependency direction, stop and surface that separately.

## Chesterton's Fence

Before removing or collapsing code whose purpose is unclear, understand why it exists.

Inspect where useful:
- surrounding callers;
- tests;
- repository documentation;
- comments;
- Git history;
- related commits;
- compatibility notes.

Do not delete strange-looking code merely because its purpose is not immediately obvious.

Historical context is especially important for:
- workarounds;
- compatibility branches;
- platform-specific behavior;
- framework hooks;
- defensive checks;
- unusual abstractions.

## Dead code requires proof

Before removing code as unused, verify it is not reached through:
- reflection;
- dependency injection;
- framework conventions;
- dynamic imports;
- plugin registration;
- serialization/deserialization;
- generated bindings;
- external/public package consumers;
- runtime discovery;
- configuration-driven loading.

Static reference absence alone may be insufficient.

## Comments and documentation

Preserve comments that explain:
- why behavior exists;
- compatibility constraints;
- external specifications;
- workarounds;
- security considerations;
- non-obvious runtime behavior.

Remove or update comments only when the simplification makes them obsolete.

Do not simplify code by deleting useful rationale.

## Sub-agent strategy

Use reconnaissance to identify independent simplification areas.

Delegate unrelated areas to isolated sub-agents where practical:

```text
Area A → sub-agent A
Area B → sub-agent B
Area C → sub-agent C
```

Related changes should stay together.

Each sub-agent receives only:
- the relevant code;
- necessary callers/tests;
- historical context needed for safety;
- repository constraints.

Preferred handoff:

```text
Area:
What was simplified:
Why it is simpler:
Behavior-preservation evidence:
Files changed:
Checks run:
Anything intentionally left unchanged:
```

Do not return:
- exploration transcripts;
- unrelated repository observations;
- giant before/after code dumps;
- speculative architecture proposals.

## Report findings

In `--report` mode, keep findings concise.

Preferred shape:

```text
1. Flatten nested validation flow
   Why: three nested branches obscure a single failure path.
   Area: src/...
   Direction: use early returns while preserving existing exceptions.
```

Do not include large before/after blocks unless they are genuinely needed to explain an unusual case.

## Validation

After modifications, use a fresh validation sub-agent.

Validate against:
- original behavior;
- actual resulting diff;
- existing tests;
- relevant build/lint/type-check commands;
- public contracts;
- side effects and exceptions;
- performance-sensitive behavior where applicable.

Do not rely on the implementation agent's explanation.

If validation finds a concrete regression:
- fix only that regression;
- validate again;
- avoid open-ended repair loops.

## Repository modification

Without `--report`, this skill may modify the working tree.

It must not:
- commit;
- push;
- amend;
- rebase;
- create or delete branches;
- reset unrelated work;
- stash unrelated work;
- clean destructively;
- checkout over user changes.

Protect unrelated uncommitted work.

Use the existing `commit` workflow when the user wants changes committed.

## Writing style

Follow the same concise, human style used by the other skills.

Avoid:
- implementation diaries;
- generic praise;
- unnecessary headings;
- boilerplate;
- generated-by/tool attribution;
- narrating that changes are focused, minimal, clean, or elegant.

## Final summary

Keep the final result compact.

Useful information:

```text
Simplifications applied: 3
Areas left unchanged due to uncertainty: 1
Validation: passed
```

For `--report`, return the report path and only a short summary.

Do not repeat full report content in the final response.

## Token efficiency

- Prefer current diff/branch scope over repository-wide scanning.
- Delegate independent areas separately.
- Keep handoffs compact.
- Use Git history only where it materially helps establish intent.
- Prefer actual tests/repository state over prose explanations.
- Do not forward one sub-agent's reasoning to another.
- Avoid exhaustive before/after narration.
- Stop when remaining candidates are subjective or low-value.
