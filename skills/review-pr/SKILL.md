---
name: review-pr
description: >-
  Review a GitHub pull request for bugs, mistakes, inconsistencies, and drift
  from the repository's own instructions, and report findings without changing
  anything. Use when asked to review, check, audit, inspect, or sanity check a
  PR, asked whether a PR has blocking findings, or given a PR number or URL. Never edits code, merges, or approves; posts a comment only
  with --comment and approval. CI diagnosis and fixing belongs to fix-pr.
argument-hint: "[<number-or-url>] [--comment] [--no-build]"
---

# Review PR

Review one pull request and report what is wrong with it.

## Contract

Read-only for code: never edit, stage, commit, or revert, and never merge,
close, approve, or request changes. The only remote mutation is `gh pr comment`,
reached only through `--comment` and only after the user approves the exact
text. A trial merge for building is allowed with a clean working tree and is
always aborted before finishing, on every exit path.

## Invocation

```text
review-pr [<number> | <PR URL>] [--comment] [--no-build]
```

- No PR: review the current branch's pull request; if it has none, list the
  open ones and ask.
- A number: that pull request in the current repository.
- A URL: that exact pull request, with the repository resolved from the URL.
- `--comment` offers to post the findings as a comment.
- `--no-build` skips the local build in step 5.

## Harness notes

Invocation is `/review-pr` in Claude Code and `$review-pr` in Codex. Read every
`AGENTS.md` and `CLAUDE.md` that exists and follow all of them; when two
conflict, ask which wins. Shell snippets show intent, so prefer the harness's
own file and search tools. Ask confirmations through the harness's confirmation
mechanism, and write any comment body to its scratchpad or temp directory,
never into the working tree. Add no attribution or generated-by text to
anything this skill writes.

`merge-pr` invokes this skill and gates on the step 7 report, so keep that
format stable.

## 1. Resolve the repository

Use the current Git repository. Outside one, inspect child directories with an
`origin` remote and ask which to use. Derive `<dir>` (root), `<slug>` (`owner/name`
from `origin`), `<repo-name>`, and `<default-branch>` (falling back to `main`).

Run every git command as `git -C <dir> ...` and every `gh pr` command with
`--repo <slug>`; `gh repo view` takes the repository positionally and rejects
`--repo`.

Find the instruction files and read each in full. They are the standard the
change is measured against in step 6:

```bash
find "<dir>" \( -name .git -o -name node_modules -o -name vendor \) -prune -o \
    \( -name AGENTS.md -o -name CLAUDE.md \) -print
```

## 2. Resolve the pull request

Without a number or URL:

```bash
gh pr view --repo <slug> --json number,title,url
gh pr list --repo <slug> --state open --json number,title,isDraft,headRefName
```

With no open pull requests, say so and stop. Otherwise ask **Which PR do you
want to review?** as a single choice of `<number>: <title>`. A structured
question usually holds at most four options; when more are open, print them all
as a numbered list and ask for the number rather than truncating.

One pull request per run. Read it:

```bash
gh pr view <number> --repo <slug> --json \
    number,title,body,state,isDraft,author,baseRefName,headRefName,url,\
additions,deletions,changedFiles,statusCheckRollup,comments,reviews
```

A merged or closed pull request can still be reviewed; say plainly that the
findings can no longer gate anything.

## 3. Read the checks

```bash
gh pr checks <number> --repo <slug>
```

| Rollup | Meaning | Effect |
|---|---|---|
| entries, all successful | the host verified those checks | skip the local build |
| an entry failing, timed out, or cancelled | the host recorded a problem | blocking evidence; inspect the decisive failure |
| an entry queued, in progress, pending, or expected | hosted validation is incomplete | report checks as pending; never call the PR merge-ready |
| empty | **nothing ran** | build locally in step 5 |

An empty or pending rollup is not a pass, and passing checks are evidence, not
proof: review the code regardless. For a failure, read enough of
`gh run view <run-id> --repo <slug> --log-failed` to quote the decisive part and
name the job and step. Root-cause investigation and fixing belong to `fix-pr`.

## 4. Read the change

```bash
gh pr diff <number> --repo <slug>
```

A diff hides the code around each hunk. For every non-trivial change, read the
whole file and the callers of anything whose signature, name, or behavior
changed. Watch what the diff **removes**: a deleted export, renamed symbol,
moved file, or dropped branch is where breakage hides.

## 5. Build the merge result

Skip with `--no-build`, or when every check succeeded; say which in the report.
Otherwise build the **merge result**, since that is what the base branch would
receive. This needs a clean working tree: if `git status --short` shows
anything, skip the build, say so, and continue the review.

```bash
git -C <dir> fetch origin pull/<number>/head:review-pr-<number>
git -C <dir> merge --no-commit --no-ff review-pr-<number>
```

A conflict is a blocking finding. With the merge staged, run the repository's
own checks, discovered from instruction files, `package.json` scripts, a
`Makefile`, task files, or CI workflows; prefer the release-preparation
verification surface, and record the commands that ran. Never hardcode one.

Undo the trial merge whether the build passed, failed, or errored, and confirm
the tree is clean again:

```bash
git -C <dir> merge --abort
git -C <dir> branch -D review-pr-<number>
git -C <dir> status --short
```

If it is not clean, say so prominently: a half-applied merge left behind is
worse than any finding.

## 6. Review

In rough order of value:

- **Correctness**: logic that does not do what its name or docs say: inverted
  conditions, wrong operators, off-by-one, unhandled null, swallowed or wrong
  errors, missing awaits.
- **Breakage**: a removed or renamed export, changed signature, or moved file
  still referenced elsewhere. Search the whole repository for the old name.
- **Contract drift**: behavior that no longer matches its docs, comments, or
  types, in either direction.
- **Security**: changed trust boundaries, auth, secrets, validation, filesystem
  or network access, deserialization, command execution, or dependencies, with a
  reachable condition, not a generic possibility.
- **Performance**: material regressions on relevant paths, plausible for the
  repository's actual usage; no micro-optimizations.
- **Dependencies**: whether an addition is needed, used, version-compatible, and
  free of breaking release notes; check maintenance, license, or transitive
  impact only when it materially matters.
- **Repository rules**: anything contradicting the step 1 instruction files:
  naming, structure, formatting, export conventions, documentation
  requirements, commit or dependency policy.
- **Leftovers**: debug output, commented-out code, scratch files, build
  artefacts, unused dependencies, secrets.
- **Coverage**: new behavior without a test where the repository has tests, and
  tests that do not exercise the changed contract or its failure paths.
- **Complexity movement**: a "simplification" that only moves complexity
  elsewhere. Judge the resulting system, not the line count.
- **Description accuracy**: claims the diff does not support, and significant
  changes the body never mentions.

Every finding is observed, not suspected. Trace it to a line and state the
triggering input or condition; where the behavior can be executed, run a scratch
file outside the repository and delete it afterwards. Watch how results are
printed so a misleading representation is not reported as a bug. Phrase what
you could not confirm as a question. A short list of real findings beats a long
padded one, and a review that finds nothing says so plainly.

## 7. Classify and report

> **Blocking** if merging would leave the default branch wrong or broken.
> Everything else is non-blocking.

| Blocking | Non-blocking |
|---|---|
| a correctness defect | naming and style |
| a failing check | wording and documentation polish |
| a failing local build, or a conflicting merge | a missing test, unless repository rules make it blocking |
| a reference to something the change removed | duplication or structure |
| a concrete security vulnerability or leaked secret | a body that undersells the change |
| a material performance regression | a minor efficiency concern |
| an incompatible dependency or API change that breaks consumers or builds | a dependency concern without demonstrated breakage |

Report in this shape:

```text
Review of #<number>: <title>

Checks: <all passed | N failing, named | N pending | none ran>
Build:  <commands run and result | skipped, and why>

Blocking findings

1. <file>:<line> <what is wrong, and the condition that triggers it>
   <what was observed, and how>

Non-blocking findings

1. <file>:<line> <what is wrong>

Verdict: <no blocking findings | N blocking findings>
```

With no blocking findings, say **no blocking findings** in those words so a
calling skill can rely on it. Never use "safe to merge": merge readiness also
depends on CI, branch protection, draft state, and approvals, which `merge-pr`
owns. Omit empty sections.

## 8. Comment, only when asked

Without `--comment`, stop here.

With it, write the body to `<scratchpad>/<repo-name>-review-<number>.md`, show
the exact text, and ask:

```text
Do you want me to post this review as a comment on #<number>?
```

Posting is visible to everyone with access to the repository, so the flag opts
into being asked, never into posting. This holds when another skill forwards
`--comment`.

The comment is prose for the pull request, not the step 7 report. Follow the
repository's pull request writing style when it has one; otherwise:

- Two or three short paragraphs. The first states the outcome, opening with
  `This review found`, `This review raises`, or `This review found nothing
  blocking`. Describe a defect with `As written, ...` and its consequence with
  `This means ...`. The last covers non-blocking findings or a compatibility
  note when either is worth saying, and states what backs the review, naming a
  skipped build or absent checks.
- With more than three or four findings, keep the opening and closing
  paragraphs and put the findings in a numbered list between them, one short
  paragraph each.
- No headings, checklists, tables, pasted output, command transcripts,
  greetings, sign-offs, or remarks about the review being thorough or
  automated. Do not approve or tell the author what to do next.

Link every code reference to the blob at the head commit, with the file path as
link text and the line anchor in the URL:

```text
[`src/routing/compileRoutes.ts`](https://github.com/<slug>/blob/<sha>/src/routing/compileRoutes.ts#L47)
```

Resolve `<sha>` once with `gh pr view <number> --repo <slug> --json headRefOid
-q .headRefOid`; use `#L47-L52` for a range. Use a fenced excerpt only when it
is the clearest way to show a defect. Link nothing that exists only locally.

After approval, post with `gh pr comment <number> --repo <slug> --body-file
"<body-file>"` and remove the file. Never use `gh pr review`.

## Safety

- Never leave a trial merge in progress, and run `git branch -D` only on the
  `review-pr-<number>` branch this skill created.
- Never stash, discard, or commit the user's work to make a build possible;
  skip the build instead.
- Never describe an inference as an observation, or a pull request as verified
  when no checks ran and no build was performed.
- If the environment blocks a git or `gh` command, show it and ask the user to
  approve or run it; never work around it by editing `.git`.

## Optional overrides

A repository's `AGENTS.md` or `CLAUDE.md` may override defaults under
`## Pull Request`, `## Pull Requests`, `## Review`, or `## Build And
Verification`:

| Hint | Affects |
|---|---|
| Required verification command per change scope | step 5 |
| Default branch, when not the GitHub-reported default | steps 1, 5 |
| Repository-specific review checklist or forbidden patterns | step 6 |
| What counts as blocking for this repository | step 7 |
| Pull request body style, which the posted comment follows | step 8 |
| Whether commenting on pull requests is permitted at all | step 8 |
