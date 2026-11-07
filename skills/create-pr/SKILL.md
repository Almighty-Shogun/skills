---
name: create-pr
description: >-
  Open a GitHub pull request for the current repository on a new or existing
  fix/, feat/, improvement/, chore/, or docs/ work branch. Infers the branch
  name from the actual change, states the file scope, delegates staging and
  commits to the commit skill, writes the title and body from repository
  instructions, and runs gh pr create only after approval. Never merges,
  force-pushes, tags, or publishes.
argument-hint: "[fix/<slug> | feat/<slug> | improvement/<slug> | chore/<slug> | docs/<slug>]"
disable-model-invocation: true
---

# Create PR

Create a focused pull request for the current GitHub repository, from any
directory inside it, using the `commit` skill for staging and commits.

## Contract

Two remote mutations, both gated: the commits and push made **by the `commit`
skill** under its own confirmations, and `gh pr create`, which runs only after
the user approves the title and body. Never merge, force-push, rebase, reset,
tag, create version commits, or publish.

## Harness notes

Invocation is `/create-pr` in Claude Code and `$create-pr` in Codex. Read every
`AGENTS.md` and `CLAUDE.md` that exists and follow all of them; when two
conflict, ask which wins. Also read any pull request template
(`.github/pull_request_template.md`, `.github/PULL_REQUEST_TEMPLATE/*`); its
requirements override the default body, but skip boilerplate the repository
does not require. Prefer the harness's own file and search tools over the shell
snippets, ask confirmations through its confirmation mechanism, and write the
body to its scratchpad or temp directory, never into the working tree.

Add no attribution trailers or generated-by footers unless an instruction file
asks for them. If the harness cannot invoke one skill from another, read
`../commit/SKILL.md` and follow it inline.

## 1. Resolve the repository

Use the current Git repository; outside one, inspect child directories with an
`origin` remote and ask which to use. Derive `<dir>`, `<slug>`, `<repo-name>`,
and `<default-branch>` (falling back to `main`). Run every git command as
`git -C <dir> ...` and every GitHub command with `--repo <slug>`.

Read the instruction files, preferring hints under `## Pull Request`,
`## Pull Requests`, or `## Git And PR Safety`:

```bash
find "<dir>" \( -name .git -o -name node_modules -o -name vendor \) -prune -o \
    \( -name AGENTS.md -o -name CLAUDE.md \) -print
```

Inspect the state:

```bash
git -C <dir> status --short --branch
git -C <dir> diff --stat
git -C <dir> diff --cached --stat
gh repo view <slug> --json nameWithOwner,defaultBranchRef
```

## 2. Prepare the branch

The base is `<default-branch>`. Use the branch the user gave; else reuse the
current branch when it already has an allowed prefix; else infer a concise,
accurate name from the requested work and the changes. Allowed prefixes are
`fix/`, `feat/`, `improvement/`, `chore/`, and `docs/`, unless an instruction
file allows others.

| Prefix | For |
|---|---|
| `fix/` | something was wrong: a bug, regression, broken behavior, or reliability problem |
| `feat/` | anything was added: user-facing behavior, a package, a capability |
| `improvement/` | reworking behavior that already worked, with nothing added and nothing fixed |
| `chore/` | maintenance that changes neither what a consumer can do nor runtime behavior: removals, renames, dependency bumps, upkeep |
| `docs/` | documentation-only work |

The prefix sets the title kind and is expensive to change after a push, so
choose it deliberately. `fix/` and `feat/` are the default pair. Prefer `fix/`
whenever something was actually wrong, even in a removal that corrects false
documentation, and `feat/` whenever anything was added, however small. A runtime
fix that also updates docs stays `fix/`. When two prefixes are both defensible,
or the summary cannot be inferred confidently, ask **before** creating the
branch.

Name the actual change, never something vague like `api-surface`. Reject names
containing whitespace, `..`, `@{`, `//`, a leading `-`, a trailing `/`, or
`.lock`, and protected names such as `main`, `master`, `development`, or
`release`.

When reusing a branch or a candidate exists, check its ancestry:

```bash
git -C <dir> merge-base "<default-branch>" HEAD
git -C <dir> rev-list --count "<default-branch>..HEAD"
git -C <dir> rev-list --count "HEAD..<default-branch>"
```

If it looks based on another feature branch or carries unrelated history,
report it and ask; never rebase or merge automatically.

Once the branch is known, check for an open pull request with
`gh pr view "<branch>" --repo <slug> --json number,url,state,title`. If one
exists, report it and stop; updating it belongs to `respond-pr`, `fix-pr`, or
the development workflow.

```bash
git -C <dir> rev-parse --verify "<branch>"
git -C <dir> ls-remote --heads origin "<branch>"
```

If the current branch is `<branch>`, continue. If it exists locally or remotely
and is not current, ask whether to switch to it, pick another name, or cancel.
Otherwise run `git -C <dir> switch -c "<branch>"`, keeping uncommitted changes
in place on the current `HEAD`; never stash or switch through another base.

## 3. Determine the scope

The pull request covers the whole branch delta, not just uncommitted changes:

```bash
git -C <dir> log --oneline "<default-branch>..HEAD"
git -C <dir> diff --stat "<default-branch>...HEAD"
git -C <dir> diff "<default-branch>...HEAD"
git -C <dir> diff
git -C <dir> diff --cached
```

Confirm every existing branch commit belongs to this pull request, and surface
unrelated history instead of hiding it. Read untracked files before including
them. Leave out build output, archives, logs, caches, and secrets unless
clearly intended, and scratch files, temporary migrations, and experimental
directories unless the user explicitly asks. Pull request scope and commit
scope differ: one pull request can hold several focused commits, and the
`commit` skill decides those.

State the scope before handing off, and ask when it is ambiguous:

```text
This PR should include:
- <file or group>: <why it belongs>

This PR should not include:
- <file or group>: <why it is unrelated>
```

## 4. Hand off to the commit skill

Run the `commit` skill with `<dir>` and the step 3 file scope; it owns staging,
grouping, validation, commits, and the push. Never stage ad hoc, never use
`git add .` or `git add -A`, and never push a second time yourself. Then verify:

```bash
git -C <dir> status --short --branch
git -C <dir> ls-remote --heads origin "<branch>"
git -C <dir> log --oneline origin/<default-branch>..HEAD
git -C <dir> diff --stat origin/<default-branch>...HEAD
```

Continue only once the commit skill has finished and `origin/<branch>` matches
the local `HEAD`. If the user declined the push, stop: the pull request cannot
be opened from an unpushed branch. With no commits ahead of the base, stop:
there is nothing to open.

## 5. Write the title and body

Repository style and templates take precedence. Otherwise:

- Title: `<Kind>: <short human summary>`, mapping `fix/` → `Bugfix`, `feat/` →
  `Feature`, `improvement/` → `Improvement`, `chore/` → `Chore`, `docs/` →
  `Docs`. No Conventional Commit prefixes. If the full diff clearly contradicts
  the branch prefix, flag the mismatch and ask; never rename a pushed branch
  automatically.
- Body: two or three short paragraphs. The first starts with `This PR
  resolves`, `This PR adds`, `This PR updates`, or `This PR improves`. For a
  bugfix, the second describes the old behavior with `Before this, ...` and the
  new with `This now ...`. The last covers a related change or compatibility
  note when useful.
- No validation commands or results (report those in your response), no
  headings, checklists, or test sections unless asked, and no remarks about the
  pull request being scoped or focused.

Write the body to `<scratchpad>/<repo-name>-pr-<branch-slug>.md`. Show the base,
head, commits, files, title, and exact body, without the shell command, and ask:

```text
Do you want me to create this PR?
```

Revise and show it again when edits are requested.

## 6. Create the PR

After approval, re-check that no pull request appeared meanwhile
(`gh pr view "<branch>" --repo <slug> --json url -q .url`); if one did, report
it and stop. Otherwise:

```bash
gh pr create --repo <slug> --base "<default-branch>" --head "<branch>" \
    --title "<title>" --body-file "<body-file>"
```

Report the URL. Do not merge.

## Optional overrides

A repository's `AGENTS.md` or `CLAUDE.md` may override defaults under
`## Pull Request`, `## Pull Requests`, or `## Git And PR Safety`:

| Hint | Affects |
|---|---|
| Allowed branch prefixes beyond the five defaults | step 2 |
| Default base branch, when not the GitHub-reported default | steps 1, 2, 4 |
| Repository-specific file grouping | step 3 |
| PR title and body style, required sections, or templates | step 5 |
| Whether attribution trailers are wanted | step 5 |
| Reviewers, labels, or draft-by-default policy | step 6 |
