---
name: review-pr
description: >-
  Review a GitHub pull request for bugs, mistakes, inconsistencies, and drift
  from the repository's own instructions, and report findings without changing
  anything. Use when asked to review, check, audit, inspect, or sanity check a
  PR, asked whether a PR has blocking findings, or given a PR number or URL to
  review. Never edits code, merges, or approves; posts a comment only with
  --comment and approval. CI diagnosis and fixing belongs to fix-pr, reviewer
  feedback to respond-pr.
argument-hint: "[<number-or-url>] [--comment] [--no-build]"
---

# Review PR

Review one pull request and report what is wrong with it.

## Contract

Read-only for code: never edit, stage, commit, or revert, and never merge,
close, approve, or request changes. The only remote mutation is `gh pr comment`,
reached only through `--comment` and only after the user approves the exact
text. A trial merge for building runs only in a temporary detached worktree,
never in the user's checkout, and the worktree is always removed before
finishing, on every exit path.

## Invocation

```text
review-pr [<number> | <PR URL>] [--comment] [--no-build]
```

- No PR: review the current branch's pull request; if it has none, list the
  open ones and ask.
- A number: that pull request in the current repository, or in its parent
  when `origin` is a fork. A pull request inside the fork itself needs its URL.
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
`origin` remote: with a URL, use the child whose `origin` names the URL's
repository when exactly one does; otherwise ask which to use. Derive `<dir>`
(root), `<origin-slug>` (`owner/name` from the `origin` URL, HTTPS or SSH,
without `.git`), `<repo-name>`, and `<default-branch>` (falling back to
`main`). Compare slugs case-insensitively.

With a URL and no usable clone (no current repository and no child matching
the URL, or no `origin` remote), continue without `<dir>`: read files through
the API (step 4), skip the build, and say so. Without a URL and without
`origin`, nothing resolves a number or branch: ask for the PR URL.

`<slug>` is the repository the pull request lives in. With `origin`, run
`gh repo view <origin-slug> --json isFork,parent`. With a URL, `<slug>` is the
URL's `owner/name` (`https://github.com/<owner>/<name>/pull/<number>`).
Otherwise, when `origin` is a fork, a number or branch resolves in the fork's
parent, not the fork, so `<slug>` is the parent's `owner/name`; else `<slug>`
is `<origin-slug>`.

Run every git command as `git -C <dir> ...` and every `gh pr` command with
`--repo <slug>`; `gh repo view` takes the repository positionally and rejects
`--repo`.

Find the instruction files and read each in full. They are the standard the
change is measured against in step 6:

```bash
find "<dir>" \( -name .git -o -name node_modules -o -name vendor \) -prune -o \
    \( -name AGENTS.md -o -name CLAUDE.md \) -print
```

When there is no `<dir>`, or `<slug>` is neither `<origin-slug>` nor its
parent, these files are not the pull request repository's: skip the Repository
rules category in step 6 and say so.

## 2. Resolve the pull request

Without a number or URL, look up the current branch's pull request and use it
when one exists. When there is none, or `git -C <dir> branch --show-current` is
empty (detached `HEAD`), list the open pull requests instead:

```bash
gh pr view "$(git -C <dir> branch --show-current)" --repo <slug> \
    --json number,title,url
gh pr list --repo <slug> --state open --limit 100 \
    --json number,title,isDraft,headRefName
```

When `<slug>` is the fork's parent, pass the branch as `<owner>:<branch>`, with
`<owner>` from `<origin-slug>`: a fork pull request matches only in that form,
and a bare name could pick a same-named branch in the parent.

With no open pull requests, say so and stop. Otherwise ask **Which PR do you
want to review?** as a single choice of `<number>: <title>`. A structured
question usually holds at most four options; when more are open, print them all
as a numbered list and ask for the number rather than truncating. When the list
returns 100 entries, say it may be capped.

One pull request per run. Read it:

```bash
gh pr view <number> --repo <slug> --json \
    number,title,body,state,isDraft,author,baseRefName,headRefName,headRefOid,\
isCrossRepository,url,additions,deletions,changedFiles,statusCheckRollup,\
comments,reviews
```

A merged or closed pull request can still be reviewed; say plainly that the
findings can no longer gate anything.

## 3. Read the checks

```bash
gh pr checks <number> --repo <slug>
```

It exits 1 when a check failed or none were reported, and 8 when one is
pending; that exit is data, so read the output. Exit 1 with any other error
message is a command failure. Exit 0 can still list cancelled or skipped
entries.

| Rollup | Meaning | Effect |
|---|---|---|
| entries, all successful | the host verified those checks | skip the local build |
| an entry failing, timed out, or cancelled | the host recorded a problem | blocking evidence; inspect the decisive failure |
| an entry queued, in progress, pending, or expected | hosted validation is incomplete | report checks as pending; never call the PR merge-ready |
| an entry skipped or neutral | the host gave it no verdict | non-failing; name it, and judge the rest by the other rows (all skipped counts as empty) |
| empty | **nothing ran** | build locally in step 5 |

An empty or pending rollup is not a pass, and passing checks are evidence, not
proof: review the code regardless. For a failing Actions check, take
`<run-id>` and `<job-id>` from its link
(`.../actions/runs/<run-id>/job/<job-id>`) and read enough of
`gh run view <run-id> --repo <slug> --log-failed` to quote the decisive part
and name the job and step. That command refuses while another job of the run
is still running; then read the finished job's log with
`gh api repos/<slug>/actions/jobs/<job-id>/logs`. Report a check from an
external provider by its link. Root-cause investigation and fixing belong to
`fix-pr`.

## 4. Read the change

```bash
gh pr diff <number> --repo <slug>
```

Beyond GitHub's diff limits (such as more than 300 files) it fails with HTTP
406; then list the files with
`gh api --paginate repos/<slug>/pulls/<number>/files` and read each at the head.

A diff hides the code around each hunk. For every non-trivial change, read the
whole file and the callers of anything whose signature, name, or behavior
changed. Watch what the diff **removes**: a deleted export, renamed symbol,
moved file, or dropped branch is where breakage hides.

Read and search at the step 2 `headRefOid`, never the checkout's own branch.
When `<slug>` is `<origin-slug>`, use the local object store; otherwise read
files through the API, and say that searches covered only the files read:

```bash
git -C <dir> fetch origin pull/<number>/head
git -C <dir> show <headRefOid>:<path>
git -C <dir> grep <pattern> <headRefOid>
gh api "repos/<slug>/contents/<path>?ref=<headRefOid>" \
    -H "Accept: application/vnd.github.raw+json"
```

Then re-read the head with
`gh pr view <number> --repo <slug> --json headRefOid -q .headRefOid`. If it
differs from step 2's, the head moved during the review: say so in the report
and that newer commits are not covered.

## 5. Build the merge result

Skip with `--no-build`, or when every check succeeded (skipped or neutral ones
aside); say which in the report. Also skip, and say so, when `<slug>` is not
`<origin-slug>`, since this checkout cannot build that pull request, and for a
merged one, since the base already contains it. When `isCrossRepository` is
true, the head comes from a fork whose author may lack write access, and
building runs its code: ask first, and skip without approval. The same approval
covers any execution in step 6. Otherwise build the **merge result**, since
that is what the base branch would receive. Build it in a temporary detached
worktree at a new path `<worktree>` in the scratchpad or temp directory, so the
user's checkout is never touched:

```bash
git -C <dir> fetch origin <baseRefName>
git -C <dir> worktree add --detach "<worktree>" origin/<baseRefName>
git -C "<worktree>" fetch origin pull/<number>/head
git -C "<worktree>" rev-parse FETCH_HEAD
git -C "<worktree>" merge --no-commit --no-ff FETCH_HEAD
```

The head fetch writes only `FETCH_HEAD`, never a named ref. If `FETCH_HEAD` is
not the step 2 `headRefOid`, the head moved during the review: skip the merge
and report the move as in step 4. A content conflict is a blocking finding; any
other merge or worktree failure, such as a missing `origin/<baseRefName>` or no
merge base in a shallow clone, is a setup failure. With the merge staged,
prepare `<worktree>` as the repository instructs, such as installing
dependencies, since a fresh worktree has none and lacks local untracked config.
A setup failure makes the build errored or skipped, not a blocking finding.
Then run
the repository's own checks in `<worktree>`, discovered from instruction files,
`package.json` scripts, a `Makefile`, task files, or CI workflows; prefer the
release-preparation verification surface, and record the commands that ran.
Never hardcode one.

Keep `<worktree>` until any step 6 execution that needs it has run, then remove
it whether the build passed, failed, or errored, or the merge never started,
and confirm it is gone:

```bash
git -C <dir> worktree remove --force "<worktree>"
git -C <dir> worktree list
```

If it is still listed, say so prominently: a leftover worktree is worse than
any finding.

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
file outside the repository, against `<worktree>` before its removal when it
needs the PR's code, and delete the file afterwards. Watch how results are
printed so a misleading representation is not reported as a bug. Phrase what
you could not confirm as a question. A short list of real findings beats a long
padded one, and a review that finds nothing says so plainly.

## 7. Classify and report

> **Blocking** if merging would leave the base branch (`baseRefName`) wrong or
> broken.
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

Head:   <step 2 headRefOid, and whether it moved>
Checks: <all passed | none ran | N failing (named), M pending, K skipped (named)>
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
owns. Omit empty sections, and in `Checks:` combine only the counts that apply.

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

`<sha>` is the step 2 `headRefOid`, the commit reviewed; use `#L47-L52` for a
range. Before posting, re-read the head as in step 4; if it moved, say in the
comment which commit was reviewed. Use a fenced excerpt only when it is the
clearest way to show a defect. Link nothing that exists only locally.

After approval, post with `gh pr comment <number> --repo <slug> --body-file
"<body-file>"` and remove the file. Never use `gh pr review`.

## Safety

- Never merge, build, or create a branch in the user's checkout, and never
  leave the temporary worktree behind; run `git worktree remove` only on the
  worktree this skill created.
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
| Default branch, when not the GitHub-reported default | step 1 |
| Repository-specific review checklist or forbidden patterns | step 6 |
| What counts as blocking for this repository | step 7 |
| Pull request body style, which the posted comment follows | step 8 |
| Whether commenting on pull requests is permitted at all | step 8 |
