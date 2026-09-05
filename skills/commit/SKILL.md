---
name: commit
description: >-
  Create clean, focused commits from the current repository state. Inspects the
  working tree, groups changes by atomic intent and reviewability, proposes a
  commit plan, stages narrowly, validates each staged group before committing,
  and asks before pushing. Uses only feat, fix, chore, and docs commit types by
  default. Never creates release/version commits or performs destructive Git
  operations.
argument-hint: "[--auto-commit] [--auto-push] [--auto-all]"
---

# Commit

Create focused commits from the current repository.

The goal is not to minimize commit count. The goal is to produce commits that are:
- atomic;
- easy to review;
- easy to revert;
- easy to cherry-pick;
- small enough to understand without hiding several subchanges together.

## Contract

Allowed mutations:
- narrow `git add` / index staging;
- `git commit` for approved groups;
- `git push` only after separate approval or an explicit auto-push option.

Do not:
- amend;
- rebase;
- reset;
- force-push;
- revert user work;
- create/delete branches;
- create release/version commits;
- broadly stage with `git add .` or `git add -A`;
- bypass hooks with `--no-verify`;
- edit `.git` metadata directly.

The `--auto-*` options skip confirmation only. They never skip inspection, grouping, safety checks, or validation.

## Invocation

```text
commit
commit --auto-commit
commit --auto-push
commit --auto-all
```

- `--auto-commit` skips confirmation before creating commits.
- `--auto-push` skips confirmation before pushing.
- `--auto-all` skips both confirmations.

If grouping is genuinely ambiguous, ask even when an auto option is present.

## Repository instructions

Read applicable `AGENTS.md` and `CLAUDE.md` files before grouping or committing.

Precedence for commit conventions:

1. explicit repository instructions;
2. established recent repository history;
3. this skill's defaults.

If repository instructions conflict materially, ask which should win.

Do not add tool-attribution, generated-by, or AI co-author trailers unless repository instructions explicitly require them.

## 1. Inspect

Move to the repository root and inspect:

```bash
git status --short
git branch --show-current
git diff --stat
git diff --name-status
git diff --cached --stat
git diff --cached --name-status
```

Read relevant unstaged and staged diffs.

Inspect untracked files before grouping them.

Do not infer intent from filenames alone.

Avoid committing:
- secrets;
- `.env` values;
- private keys;
- logs;
- caches;
- build output;
- large generated artifacts;
- archives/binaries;

unless they are clearly intentional repository content.

## 2. Protect existing staged work

Treat the user's existing index as protected state.

Never:
- unstage user-staged content without approval;
- absorb unrelated staged changes into a new commit;
- destroy partial staging;
- replace the index wholesale.

If existing staged work already forms a coherent intended commit, preserve it.

If staged and unstaged hunks of the same file cannot be separated safely, stop and ask rather than rewriting the user's staging.

## 3. Group by atomic intent

Group changes by what they accomplish, not by directory or file type.

Main test:

> Could this part reasonably be reviewed, reverted, or cherry-picked without the other part?

If yes, it probably belongs in a separate commit.

Examples that normally belong together:
- a bug fix and the regression test proving it;
- a feature implementation and tests for that exact behavior;
- a dependency manifest change and the lockfile update caused by it;
- a rename/move and the references required to keep the repository working;
- source plus documentation that is genuinely part of introducing the same public behavior.

Examples that normally split:
- independent bug fixes;
- unrelated features;
- standalone documentation cleanup;
- unrelated formatting changes;
- independent workflow/configuration work;
- a lockfile refresh unrelated to a dependency declaration;
- repository-skill/instruction changes unrelated to the runtime change.

File type alone is never a reason to split.

Documentation may travel with source when both are one atomic change. Tests should normally travel with the behavior they verify.

### Keep commits reviewable

Atomic does not mean "put the whole feature branch in one commit."

A group is too broad when:
- its subject cannot describe every file precisely;
- it contains several independently understandable subchanges;
- a reviewer would need to mentally separate different concerns;
- meaningful parts could be reverted/cherry-picked independently;
- the group grew large merely because everything belongs to the same feature/package/PR.

For every large group, perform a second decomposition pass before presenting the plan.

Look for clean seams such as:
- independent behavior changes;
- separate bug fixes;
- separate public contracts;
- independent migration/configuration work;
- standalone docs cleanup;
- unrelated maintenance.

Split at those seams when doing so leaves each commit valid and understandable.

There is no arbitrary numeric file cap: a mechanical rename may legitimately touch many files, while three unrelated files may already require three commits. But a large file count is a warning that must trigger another grouping review, not a reason to accept a giant commit automatically.

When in doubt, split.

### Partial-file changes

If one file contains unrelated hunks, stage only the hunks belonging to the current commit.

Use a reliable patch/index mechanism supported by the harness.

Do not invoke an interactive patch command the harness cannot control reliably. If safe selective staging is not possible, ask the user.

## 4. Commit messages

Use only these default types:

```text
feat
fix
chore
docs
```

Meaning:

- `feat:` new behavior, public capability, export, or feature.
- `fix:` correction of broken or incorrect behavior/configuration/workflow.
- `docs:` documentation-only changes.
- `chore:` maintenance, dependencies, lockfiles, formatting, internal cleanup, instructions, skills, or other non-feature/non-fix work.

Do not use other Conventional Commit types unless explicit repository instructions override this skill.

Use lowercase type prefixes.

Examples:

```text
feat: add request validation
fix: preserve null serialization
docs: document cache configuration
chore: update repository skills
```

### Match repository history

Inspect recent history when needed:

```bash
git log --no-merges -20 --pretty=format:"%h %s%n%b"
```

If the repository consistently uses scopes, match them:

```text
fix(api): handle duplicate requests
```

If it consistently uses another subject convention permitted by repository instructions, follow that convention.

### Subject-first

Prefer one-line subject-only commits.

Do not add a body merely to narrate:
- implementation details;
- validation;
- reasoning already obvious from the diff;
- AI/tool activity.

Use a body only when:
- the user explicitly requests one;
- repository instructions require one;
- recent history clearly treats bodies as normal;
- important non-obvious context genuinely belongs in Git history.

Do not include package versions in commit subjects unless explicitly requested.

Do not create release/version commits.

## 5. Propose the plan

Show the intended commits before mutation unless `--auto-commit` / `--auto-all` applies.

Example:

```text
Commit 1: fix: preserve null serialization
Files:
- src/serialization/deserialize.ts
- tests/serialization/deserialize.test.ts

Commit 2: docs: document serializer behavior
Files:
- docs/serialization.md
```

For each group, ensure the subject accurately describes all included changes.

If a group is large, verify the second decomposition pass happened before presenting it.

Without auto-commit, ask approval for the proposed commit(s).

If the user changes grouping or messages, revise the plan before staging.

## 6. Stage one group

Stage only the current group.

Before staging:

```bash
git diff --cached --name-status
```

After staging:

```bash
git diff --cached --stat
git diff --cached --name-status
git diff --cached
```

Confirm the index contains exactly the current intended commit plus any explicitly preserved staged state that belongs to it.

Never broadly stage the working tree.

## 7. Validate before committing

Validation happens **after staging the current group and before creating its commit**.

Always run when applicable:

```bash
git diff --cached --check
```

Choose additional checks from repository instructions and the staged scope.

Examples:
- application/package code → relevant test/build/typecheck;
- bug fix → regression test where available;
- docs → docs validation/build when useful;
- dependency/config/workflow → relevant targeted verification.

Validation should match the staged commit, not unrelated working-tree changes.

Passing checks are necessary evidence, not a reason to ignore an obviously incorrect diff.

If a required check fails:
- do not create the commit;
- report the failure;
- only modify code if the user asked this workflow to fix it or another mutation workflow is invoked.

If checks are impractical or intentionally skipped, report that clearly.

## 8. Commit

After the staged group passes its relevant validation:

```bash
git commit -m "<message>"
```

Then re-check:

```bash
git status --short
git diff --cached --name-status
```

Repeat stage → validate → commit for each approved group.

If the working tree changes unexpectedly during the workflow, stop and re-inspect before continuing.

## 9. Report

Keep the result compact.

Example:

```text
Created:
- a12bc34 fix: preserve null serialization
- d45ef67 docs: document serializer behavior

Validation:
- bun test ✓
- bun run typecheck ✓

Branch: feature/serialization
```

Mention skipped/failed validation where relevant.

Do not narrate every Git command or staging step.

## 10. Push

Push only after all approved commits are created and reported.

Unless `--auto-push` or `--auto-all` applies, ask explicit approval.

Before pushing:
- inspect the current branch;
- inspect its configured upstream;
- inspect plausible remotes when needed.

Push only the branch containing the new commits.

Never:
- `git push --all`;
- push tags as part of this skill;
- force-push;
- silently choose an unusual remote when several plausible remotes exist.

If no safe target is clear, ask.

## Safety

- Protect all unrelated working-tree and staged changes.
- Never unstage user work without approval.
- Never reset, stash, clean, rebase, amend, or force-push.
- Never use broad staging.
- Never bypass Git hooks.
- Never create release/version commits.
- Never publish packages or release tags.
- Never commit secrets or generated noise unless clearly intentional.
- If Git write access is blocked, report the blocked operation rather than working around repository metadata.

## Token efficiency

- Inspect only diffs needed to group and validate the changes.
- Do not produce an essay explaining every group.
- Use repository history only to establish actual commit conventions.
- Keep plan and final report concise.
- Do not repeat source diffs in the response.
