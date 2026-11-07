---
name: commit
description: >-
  Create focused commits from the current working tree: group changes by
  atomic intent, propose the plan, stage narrowly, validate each group before
  committing, and ask before pushing. Uses feat, fix, improvement, chore, and
  docs types by default. Never creates release commits or rewrites history.
argument-hint: "[--auto-commit] [--auto-push] [--auto-all]"
---

# Commit

Turn the current working tree into commits that are each atomic and easy to
review, revert, and cherry-pick. The goal is not the fewest commits.

## Contract

Allowed: narrow staging, `git commit` for approved groups, and `git push` after
separate approval or an auto-push flag. Never amend, rebase, reset,
force-push, revert user work, create or delete branches, create release or
version commits, stage broadly with `git add .` or `git add -A`, bypass hooks
with `--no-verify`, or edit `.git` directly. If Git write access is blocked,
report the operation instead of working around it.

## Invocation

- `--auto-commit` skips confirmation before committing.
- `--auto-push` skips confirmation before pushing.
- `--auto-all` skips both.

The flags skip confirmation only, never inspection, grouping, safety checks, or
validation.
Genuinely ambiguous grouping is still asked about.

## Conventions

Read `AGENTS.md` and `CLAUDE.md` first. Precedence: explicit repository
instructions, then established recent history, then this skill. Ask when
instructions conflict materially. Add no tool-attribution, generated-by, or AI
co-author trailers unless the repository requires them.

## 1. Inspect

From the repository root:

```bash
git status --short
git branch --show-current
git diff --stat
git diff --cached --stat
```

Read the relevant diffs and every untracked file before grouping; never infer
intent from filenames. Do not commit secrets, `.env` values, keys, logs, caches,
build output, large generated artifacts, archives, or binaries unless they are
clearly intended repository content.

## 2. Protect existing staged work

The user's index is protected. Never unstage user-staged content without
approval, absorb unrelated staged changes into a new commit, or replace the
index wholesale. Preserve staged work that already forms a coherent commit. If
staged and unstaged hunks of one file cannot be separated safely, ask.

## 3. Group by atomic intent

Group by what changes accomplish, not by directory or file type. The test:

> Could this part reasonably be reviewed, reverted, or cherry-picked without the other part?

If yes, it is probably a separate commit. A fix travels with its regression
test, a feature with its tests, a manifest change with the lockfile update it
caused, a rename with the references that keep the repository working, and
source with documentation that is genuinely part of the same public change.
Independent fixes, unrelated features, standalone doc or formatting cleanup,
unrelated workflow or config work, an unrelated lockfile refresh, and skill or
instruction edits unrelated to the runtime change split apart. File type alone
is never a reason to split.

Atomic does not mean one commit per feature branch. A group is too broad when
its subject cannot describe every file precisely, or it holds several
independently understandable changes a reviewer would have to separate
mentally. There is no file-count cap, since a mechanical rename may touch many
files, but a large group always gets a second decomposition pass looking for
clean seams: independent behavior, separate fixes, separate public contracts,
migration or config work, standalone docs, unrelated maintenance. When in
doubt, split.

When one file holds hunks for different commits, stage only the current
commit's hunks through a patch mechanism the harness controls reliably, never
an interactive one. If selective staging is not safe, ask.

## 4. Commit messages

| Type | For |
|---|---|
| `feat:` | new behavior, capability, or export |
| `fix:` | broken or incorrect behavior, configuration, or workflow corrected |
| `improvement:` | reworking behavior that already worked, with nothing added or fixed |
| `docs:` | documentation only |
| `chore:` | maintenance, dependencies, lockfiles, formatting, internal cleanup, instructions, skills |

`feat` and `fix` are the default pair. Prefer `fix` whenever something was
actually wrong and `feat` whenever anything was added, however small. `chore` is
narrow: a change to what a consumer can do or what runs is not a chore. These
match the `create-pr` branch prefixes, so a pull request's commits can share
its kind. Use other types only when repository instructions say so.

Lowercase types, matching recent history
(`git log --no-merges -20 --pretty=format:"%h %s%n%b"`): use scopes such as
`fix(api): handle duplicate requests` when the repository consistently does.

```text
feat: add request validation
fix: preserve null serialization
improvement: resolve the audience check during token validation
docs: document cache configuration
chore: update repository skills
```

Subject-only by default. Add a body only when the user or repository asks for
one, recent history treats bodies as normal, or non-obvious context genuinely
belongs in history; never to narrate implementation, validation, or tool
activity. No package versions in subjects unless requested.

## 5. Propose the plan

Unless auto-commit applies, show the plan and ask for approval; revise it before
staging when the user changes groups or messages.

```text
Commit 1: fix: preserve null serialization
Files:
- src/serialization/deserialize.ts
- tests/serialization/deserialize.test.ts

Commit 2: docs: document serializer behavior
Files:
- docs/serialization.md
```

## 6. Stage, validate, commit

For each group in turn:

1. Stage only that group, then confirm with `git diff --cached --stat` and
   `git diff --cached` that the index holds exactly the intended commit plus any
   preserved staged state that belongs to it.
2. Validate the staged group: always `git diff --cached --check`, plus checks
   matching the scope and repository instructions (tests, build, or typecheck
   for code; the regression test for a fix; a docs build for docs). Passing
   checks do not excuse an obviously wrong diff. If a required check fails,
   do not commit; report it, and change code only when the user asked this
   workflow to fix it. Report checks that were skipped or impractical.
3. `git commit -m "<message>"`, then `git status --short`.

If the working tree changes unexpectedly mid-workflow, stop and re-inspect.

## 7. Report

```text
Created:
- a12bc34 fix: preserve null serialization
- d45ef67 docs: document serializer behavior

Validation:
- bun test ✓
- bun run typecheck ✓

Branch: feature/serialization
```

Mention skipped or failed validation; do not narrate each Git command.

## 8. Push

Push after all commits are created and reported, asking first unless
`--auto-push` or `--auto-all` applies. Check the current branch, its upstream,
and the plausible remotes, and push only the branch with the new commits. Never
`git push --all`, push tags, force-push, or silently pick an unusual remote when
several are plausible; ask when no safe target is clear.
