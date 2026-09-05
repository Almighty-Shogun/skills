---
name: create-pr
description: >-
  Open a GitHub pull request for the current repository, on a new or existing
  work branch. Use when the user asks to create, open, or raise a PR or pull
  request, asks to push work up for review, optionally passing a fix/*, feat/*,
  improvement/*, chore/*, or docs/* branch name, or asks to reuse the current
  work branch. Resolves the
  repository and default branch, validates or infers the branch name from the
  actual change, states the intended file scope, delegates staging and commits to
  the commit skill, writes a repository-appropriate title and body from
  repository instructions, and runs gh pr create only after explicit approval. It
  never merges, force-pushes, creates release tags or version commits, or runs
  publish commands.
argument-hint: "[fix/<slug> | feat/<slug> | improvement/<slug> | chore/<slug> | docs/<slug>]"
disable-model-invocation: true
---

# Create PR

Create a focused pull request for the current GitHub repository. The skill works
from any directory inside the repository and relies on the `commit` skill for
staging and commit creation.

## The contract

This skill's remote mutations are exactly two, both gated:

1. The commits and push performed **by the `commit` skill** in step 4, under that
   skill's own confirmation gates.
2. `gh pr create` in step 6, which must not run before the user approves the
   proposed title and body in step 5.

It never merges a PR, force-pushes, rebases, resets, creates release tags or
version commits, or runs publish commands.

## Invocation

Argument forms, shown without the harness invocation prefix:

```text
<no argument>
fix/example-bug
feat/example-feature
```

## Harness notes

This skill runs under any agent harness. Invocation is `/create-pr` in Claude
Code and `$create-pr` in Codex.

- Repository instructions live in `AGENTS.md`, `CLAUDE.md`, or both. Read every
  one that exists and follow all of them. If two files conflict, ask the user
  which wins.
- Inspect repository PR templates when present, including `.github/pull_request_template.md` and `.github/PULL_REQUEST_TEMPLATE/*`. Repository requirements/templates override the default body shape where applicable.
- Shell snippets show intent. When your harness has a dedicated file-reading or
  search tool, use it instead of `sed`, `cat`, or `find`.
- Confirmation prompts show required substance, not required formatting. Ask
  through whatever confirmation mechanism your harness provides.
- Write the PR body to your harness's scratchpad or temp directory, never inside
  the repository working tree.
- **Do not add tool attribution trailers or generated-by footers** to commits or
  PR bodies unless the repository instruction file explicitly asks for them.
- This skill depends on the `commit` skill (step 4). If your harness cannot
  invoke one skill from another, read `../commit/SKILL.md` and follow its rules
  inline.

## 1. Resolve the repository

Use the current Git repository. If the current directory is not inside a Git
repository, inspect direct child directories that have an `origin` remote and ask
the user which one to use.

Derive:

- `<dir>`: repository root.
- `<slug>`: GitHub `owner/name` from `origin`.
- `<repo-name>`: the `name` half of `<slug>`.
- `<default-branch>`: GitHub default branch, falling back to `main` when it
  cannot be resolved.

Run **every** git command with `git -C <dir> ...` and **every** GitHub command
with `--repo <slug>`, so the skill works from any current directory.

Read repository-specific instructions:

```bash
find "<dir>" \
    \( -name .git -o -name node_modules -o -name vendor \) -prune -o \
    \( -name AGENTS.md -o -name CLAUDE.md \) -print
```

Read each file found. Prefer pull-request hints inside `## Pull Request`,
`## Pull Requests`, or `## Git And PR Safety` when present; see the overrides
index at the end of this file.

Also inspect any applicable pull-request template before generating the body. Do not copy irrelevant template boilerplate when the repository does not require it.

Inspect the repository:

```bash
git -C <dir> status --short --branch
git -C <dir> branch --show-current
git -C <dir> diff --stat
git -C <dir> diff --name-status
git -C <dir> diff --cached --stat
git -C <dir> diff --cached --name-status
gh repo view --repo <slug> --json nameWithOwner,defaultBranchRef
```

## 2. Prepare the branch

Base branch is `<default-branch>`.

Resolve and validate the target branch:

- Use the provided branch when present and valid.
- Reuse the current work branch when no branch is provided and it already matches
  an allowed work-branch prefix.
- If no branch is provided and the current branch does not match an allowed
  prefix, infer a concise, accurate name from the requested work and the local
  changes.
- Accept only `fix/*`, `feat/*`, `improvement/*`, `chore/*`, or `docs/*` unless the repository instruction file explicitly allows another prefix.
- Infer `fix/<slug>` for bug fixes, regressions, broken behavior, or reliability fixes.
- Infer `feat/<slug>` for new user-facing behavior, new packages, or new product capabilities.
- Infer `improvement/<slug>` for reworking behavior that already worked, such as
  a clearer API shape, better defaults, or a rewritten implementation with the
  same surface.
- Infer `chore/<slug>` for maintenance that adds no behavior and fixes nothing, such as removing an API, renaming, dependency bumps, or repository upkeep.
- Infer `docs/<slug>` for documentation-only work that is neither correcting broken runtime behavior nor introducing a product capability.

**The prefix sets the title kind and cannot be changed cheaply after a push, so
choose it deliberately.** `fix/` and `feat/` are the default pair and stay the right answer for most runtime/product work. Reach for `improvement/`, `chore/`, or `docs/` only when those are more honest:

- Prefer `fix/` over `chore/` whenever something was actually wrong. A removal
  that also corrects false documentation is still a fix.
- Prefer `feat/` over `improvement/` whenever anything was added. A rework that
  introduces one new method is a feature, however small that method is.
- `chore/` is narrow. If the change alters what a consumer can do or what the code does at runtime, it is not a chore.
- `docs/` is for documentation-only work. A runtime bug fix that also updates docs remains `fix/`; a feature that includes docs remains `feat/`.

- **Do not use vague names** such as `api-surface`. The name should describe the
  actual change.
- Ask the user when two prefixes both look defensible, or when the summary
  cannot be inferred confidently. Ask **before** creating the branch, never
  after.

Reject names with whitespace, `..`, `@{`, `//`, a leading `-`, a trailing `/`, or
`.lock`. Reject protected names such as `main`, `master`, `development`, or
`release`.

Before creating or switching branches, verify the branch/base relationship when a candidate branch exists or the current branch is being reused:

```bash
git -C <dir> merge-base "<default-branch>" HEAD
git -C <dir> rev-list --count "<default-branch>..HEAD"
git -C <dir> rev-list --count "HEAD..<default-branch>"
```

If ancestry suggests the work branch was based on another feature branch or otherwise contains unrelated history, report it and ask before continuing. Do not automatically rebase or merge.

As soon as a concrete branch is known, check for an existing open PR:

```bash
gh pr view "<branch>" --repo <slug> --json number,url,state,title
```

If an open PR already exists, report it and stop instead of preparing a duplicate. Use `respond-pr`, `fix-pr`, or the relevant development workflow when the goal is to update that PR.

Check whether the branch already exists:

```bash
git -C <dir> rev-parse --verify "<branch>"
git -C <dir> ls-remote --heads origin "<branch>"
```

If the current branch already matches `<branch>`, continue. If the branch exists
locally or remotely and the current branch does not match it, stop and ask
whether to switch to it, use another name, or cancel. Otherwise create it:

```bash
git -C <dir> switch -c "<branch>"
```

If the working tree already has uncommitted changes, keep them in place and
create the branch from the current `HEAD`. Do not stash or switch through another
base branch unless the user explicitly asks.

## 3. Determine the PR and commit scope

Determine PR scope from the complete branch delta, not merely uncommitted changes. Inspect:

```bash
git -C <dir> log --oneline "<default-branch>..HEAD"
git -C <dir> diff --stat "<default-branch>...HEAD"
git -C <dir> diff --name-status "<default-branch>...HEAD"
git -C <dir> diff "<default-branch>...HEAD"
git -C <dir> diff
git -C <dir> diff --cached
```

Review every existing branch commit in `<default-branch>..HEAD` and make sure it actually belongs to this PR. If committed history contains clearly unrelated work, surface it instead of hiding it inside the PR. Do not rewrite/rebase history automatically.

For untracked files, inspect their contents before including them. Exclude build
outputs, archives, logs, caches, secrets, and unrelated local files unless they
are clearly intended. Do not include scratch files, temporary migration files, or
experimental directories unless the user explicitly asks.

Use the repository instruction file to identify repository-specific file groups.
If no grouping guidance exists, group by intent and ownership rather than by
minimizing commit count. Prefer more focused commits over fewer broad ones.

PR scope and commit scope are different. A PR may legitimately contain several focused commits. Do not collapse independent/reviewable changes into one giant commit merely because they belong to the same PR. Let the `commit` skill apply its atomicity, reversibility, and reviewability rules.

State the intended scope before handing off:

```text
This PR should include:
- <file or group>: <why it belongs>

This PR should not include:
- <file or group>: <why it is unrelated>
```

If the scope is ambiguous, ask which files should be included. If it is clear,
continue.

## 4. Hand off to the commit skill

Run the `commit` skill through your harness's skill mechanism, handing it `<dir>`
as the repository root and the file scope from step 3. Let that skill handle
staging, commit grouping, verification, commit creation, and push confirmation.

If your harness cannot invoke one skill from another, read `../commit/SKILL.md`
and follow its rules inline for the same file scope. **Do not substitute ad-hoc
staging**, and never use `git add .` or `git add -A`.

Do not create the PR until the commit skill has completed and the branch has been pushed to `origin`. The `commit` workflow owns committing and pushing local work; `create-pr` must not perform a second ad-hoc push. Then verify:

```bash
git -C <dir> status --short --branch
git -C <dir> log --oneline origin/<default-branch>..HEAD
git -C <dir> diff --stat origin/<default-branch>...HEAD
git -C <dir> diff --name-status origin/<default-branch>...HEAD
```

If there are no commits ahead of the base branch, **stop**; there is nothing to
open a PR for.

## 5. Generate the PR text and review gate

Use the PR writing style and required sections from repository instructions and applicable PR templates when present. Those requirements take precedence. Otherwise use this default style:

- Title format: `<Kind>: <short human summary>`. Normally map `fix/` → `Bugfix`, `feat/` → `Feature`, `improvement/` → `Improvement`, `chore/` → `Chore`, and `docs/` → `Docs`. If the actual complete PR diff clearly contradicts the branch prefix, do not write a misleading title just to match it. Flag the mismatch and ask how to proceed. Never rename an already-pushed branch automatically.
- Keep the title direct and short.
- **Do not** use Conventional Commit prefixes such as `fix:` in the PR title.
- Body is two or three short paragraphs.
- First paragraph starts with `This PR resolves`, `This PR adds`,
  `This PR updates`, or `This PR improves`.
- For a bugfix, the second paragraph explains the previous behavior with
  `Before this, ...` and the new behavior with `This now ...`.
- The final paragraph describes a concrete related change or compatibility note
  when useful.
- **Do not** include validation commands or results in the body unless the user
  explicitly asks. Report validation separately in your response.
- **Do not** add meta statements about the PR being scoped, limited, focused, or
  only containing certain files.
- **Do not** add markdown headings, checklists, or test sections unless the user
  asks.
- **Do not** append tool attribution trailers or generated-by footers unless the
  repository instruction file explicitly asks for them.

Write the proposed body to your harness's scratchpad or temp directory, never
inside the repository working tree:

```text
<scratchpad>/<repo-name>-pr-<branch-slug>.md
```

Show the user the consequential PR content: base branch, head branch, commits included, files included, proposed title, and the exact proposed body. Do not clutter the approval preview with the shell command. Then ask:

```text
Do you want me to create this PR?
```

Do **not** run `gh pr create` until the user clearly approves. If edits are
requested, revise the body file and show it again.

## 6. Create the PR

After approval, re-check that no PR was created for the branch while this workflow was preparing the request:

```bash
gh pr view "<branch>" --repo <slug> --json url -q .url
```

If one exists, report the URL and stop. Otherwise create it:

```bash
gh pr create \
    --repo <slug> \
    --base "<default-branch>" \
    --head "<branch>" \
    --title "<title>" \
    --body-file "<body-file>"
```

Report the PR URL from the command output. **Do not merge the PR.**

## Optional overrides

A repository's `AGENTS.md` or `CLAUDE.md` may override defaults. Recognised hints
live under `## Pull Request`, `## Pull Requests`, or `## Git And PR Safety`:

| Hint | Affects |
|---|---|
| Allowed branch prefixes beyond `fix/*`, `feat/*`, `improvement/*`, `chore/*`, and `docs/*` | step 2 |
| Default base branch, when not the GitHub-reported default | steps 1, 2, 4 |
| Repository-specific file grouping | step 3 |
| PR title and body style, required sections, or templates | step 5 |
| Whether attribution trailers are wanted | step 5 |
| Reviewers, labels, or draft-by-default policy | step 6 |
