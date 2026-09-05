---
name: review-pr
description: >-
  Review a GitHub pull request for bugs, mistakes, inconsistencies, and drift
  from the repository's own instructions, and report findings without changing
  anything. Use when the user asks to review, check, audit, inspect, or sanity
  check a PR or pull request, asks whether a PR has blocking code-review
  findings, or passes a PR number or URL. Reads the PR diff, surrounding code,
  review-relevant check state, and the merge result when local validation is
  needed. Never edits code, never merges, never approves or requests changes
  through GitHub, and posts a comment only when asked. Detailed CI diagnosis
  and fixing belongs to fix-pr.
argument-hint: "[<number-or-url>] [--comment] [--no-build]"
disable-model-invocation: true
---

# Review PR

Review one pull request and report what is wrong with it. The skill produces
findings only. It changes no files, merges nothing, and leaves the working copy
exactly as it found it.

## The contract

This skill is **read-only for code**. It never edits, stages, commits, or reverts
anything, and it never merges, closes, approves, or requests changes on a pull
request.

Its only remote mutation is `gh pr comment`, reachable **only** via `--comment`,
and **only** after the user approves the comment text in step 8. Every other
invocation ends by writing findings to your response.

It may perform a local trial merge to build the pull request, but only with a
clean working tree, and it always aborts that merge before finishing, on every
exit path including failure.

## Invocation

Argument forms, shown without the harness invocation prefix:

```text
<no argument>
12
https://github.com/owner/repo/pull/12
12 --comment
12 --no-build
--no-build
```

- No PR argument: review the pull request for the current branch. When the
  current branch has none, list the open pull requests and ask which to review.
- A number: review that pull request in the current repository.
- A PR URL: review that exact pull request and resolve its repository from the
  URL. Do not guess from an unrelated current repository.
- `--comment` offers to post the findings to the pull request as a comment, after
  showing the exact text and asking.
- `--no-build` skips the local build in step 5 unconditionally.

## Harness notes

This skill runs under any agent harness. Invocation is `/review-pr` in Claude Code
and `$review-pr` in Codex.

- Repository instructions live in `AGENTS.md`, `CLAUDE.md`, or both. Read every
  one that exists and follow all of them. If two files conflict, ask the user
  which wins.
- Shell snippets show intent. When your harness has a dedicated file-reading or
  search tool, use it instead of `sed`, `cat`, or `find`.
- Confirmation prompts show required substance, not required formatting. Ask
  through whatever confirmation mechanism your harness provides.
- Write any comment body to your harness's scratchpad or temp directory, never
  inside the repository working tree.
- **Do not add tool attribution trailers or generated-by footers** to anything
  this skill writes.
- The `merge-pr` skill invokes this one for the pull request it is about to
  merge. Keep the report format in step 7 stable, because that skill gates on the
  blocking findings.

## 1. Resolve the repository

Use the current Git repository. If the current directory is not inside one,
inspect direct child directories that have an `origin` remote and ask the user
which to use.

Derive:

- `<dir>`: repository root.
- `<slug>`: GitHub `owner/name` from `origin`.
- `<repo-name>`: the `name` half of `<slug>`.
- `<default-branch>`: GitHub default branch, falling back to `main` when it
  cannot be resolved.

Run **every** git command with `git -C <dir> ...` and **every** `gh pr` command
with `--repo <slug>`. `gh repo view` takes the repository positionally and
rejects `--repo`.

Read repository-specific instructions:

```bash
find "<dir>" \
    \( -name .git -o -name node_modules -o -name vendor \) -prune -o \
    \( -name AGENTS.md -o -name CLAUDE.md \) -print
```

Read every file found, in full. They are the standard the change is measured
against in step 6, not background reading.

## 2. Resolve the pull request

With a PR URL, resolve that exact repository and pull request.

With a number, use it in the current repository.

Without either, resolve the pull request for the current branch:

```bash
gh pr view --repo <slug> --json number,title,url
```

When the current branch has no pull request, list the open ones:

```bash
gh pr list --repo <slug> --state open \
    --json number,title,isDraft,headRefName
```

If there are none, say so and stop. Otherwise ask **Which PR do you want to
review?** and present one entry per pull request in the form `<number>: <title>`,
as a single choice.

Harness limit: a structured question commonly supports at most four options. When
more pull requests are open than the harness can present, do not truncate the
list. Print all of them as numbered text and ask the user to reply with the
number they want.

Review one pull request per run. To review several, run the skill once for each.

Read it in full:

```bash
gh pr view <number> --repo <slug> --json \
    number,title,body,state,isDraft,author,baseRefName,headRefName,url,\
additions,deletions,changedFiles,statusCheckRollup,comments,reviews
```

Report plainly when the pull request is already merged or closed; reviewing it is
still valid, but the findings can no longer gate anything.

## 3. Read the checks

Record what the host actually ran:

```bash
gh pr checks <number> --repo <slug>
```

Four outcomes, and they must not be conflated:

| Rollup | Meaning | Effect |
|---|---|---|
| entries, all successful | the host verified those checks | skip the local build in step 5 |
| an entry failing/timed out/cancelled | the host found or recorded a problem | blocking review evidence; inspect the decisive failure context |
| an entry queued/in progress/pending/expected | hosted validation is incomplete | report checks as pending; never describe the PR as merge-ready |
| empty | **nothing ran** | build locally in step 5 |

An empty rollup is not a pass. Pending checks are not a pass either. Passing
checks are evidence, not proof of correctness; continue the code review even
when every hosted check is green.

For a failing check, inspect enough failure context to identify what the host
actually observed:

```bash
gh run view <run-id> --repo <slug> --log-failed
```

Quote or summarize only the decisive part that shows the actual failure, and
say which job and step produced it.

Do not turn this into root-cause/fix investigation. When the user wants to know
why CI failed or wants it fixed, hand off to `fix-pr`.

## 4. Read the change

Read the diff:

```bash
gh pr diff <number> --repo <slug>
```

A diff alone is not enough to review correctness, because it hides the code
around each hunk. For every non-trivial change, read the whole file it lands in,
and read the callers of anything whose signature, name, or behaviour changed.

Pay attention to what the diff **removes**. A deleted export, renamed symbol,
moved file, or dropped branch of a conditional is where breakage hides, and none
of it is visible from the added lines.

## 5. Build the merge result

Skip this step when `--no-build` was passed, or when step 3 found checks that
all succeeded. Say in the report which reason applied.

When hosted checks are absent, failed, or still incomplete, a local build may
provide additional evidence when safe. Build the **merge result** rather than
the branch alone, because that is what the base branch would receive.

This requires a clean working tree. If `git status --short` reports anything,
skip the build, say so in the report, and continue with the rest of the review
rather than stopping:

```bash
git -C <dir> status --short
git -C <dir> fetch origin pull/<number>/head:review-pr-<number>
git -C <dir> merge --no-commit --no-ff review-pr-<number>
```

If the merge conflicts, that is a blocking finding. Abort and clean up.

With the merge staged but uncommitted, run the repository's own checks. Discover
them from `AGENTS.md`, `CLAUDE.md`, `package.json` scripts, `Makefile`, task
files, or the CI workflow definitions. Never hardcode a command. Prefer the
verification surface the repository names for release preparation, and note in
the report exactly which commands ran.

Always undo the trial merge, whether the build passed, failed, or errored:

```bash
git -C <dir> merge --abort
git -C <dir> branch -D review-pr-<number>
git -C <dir> status --short
```

Confirm the working tree is clean again before reporting. If it is not, say so
prominently; a half-applied merge left behind is worse than any finding.

## 6. Review

Look for, in roughly this order of value:

- **Correctness.** Logic that does not do what its name or its documentation
  says. Inverted conditions, wrong operator, off-by-one, unhandled `null` or
  `undefined`, an error path that swallows or rethrows the wrong thing, an async
  call that is not awaited.
- **Breakage.** A removed or renamed export, changed signature, or moved file
  that something else still references. Search for the old name across the whole
  repository rather than trusting the diff.
- **Contract drift.** Behaviour that no longer matches the documentation,
  comments, or type signatures describing it, in either direction. A renamed
  export whose page still uses the old name. A widened parameter whose docs still
  say the old type.
- **Security.** New or changed trust boundaries, authorization/authentication,
  secret handling, input validation, filesystem/network access, deserialization,
  command execution, or dependency use that creates a concrete security risk. Do
  not report generic security possibilities without a reachable condition.
- **Performance.** Material regressions on relevant paths: unnecessary repeated
  work, accidental quadratic behaviour, blocking work on hot paths, unbounded
  allocations/queries, or loss of caching/batching. Only report performance when
  the impact is plausible for the repository's actual usage; do not nitpick
  micro-optimizations.
- **Dependencies.** For added or upgraded dependencies, check whether the existing
  stack already provides the capability, whether the dependency is actually used,
  whether the selected version is compatible with the repository, and whether
  release/migration notes reveal breaking changes. When maintenance state, license,
  or transitive impact materially affects the project, verify those rather than
  guessing.
- **Repository rules.** Anything contradicting the instruction files read in step
  1: naming, structure, formatting, export conventions, documentation
  requirements, commit or dependency policy.
- **Leftovers.** Debug output, commented-out code, scratch or temporary files,
  build artefacts, dependencies added but unused, secrets or credentials.
- **Coverage.** New behaviour with no test, where the repository has tests.
  Passing tests are necessary evidence, not sufficient evidence that the change is
  correct; inspect whether the tests exercise the changed contract and important
  failure paths.
- **Complexity movement.** A refactor that claims to simplify code but merely moves
  complexity into another module, wrapper, generic abstraction, or configuration
  surface. Judge the resulting system, not line-count reduction.
- **Description accuracy.** Claims in the pull request body that the diff does not
  support, and significant changes the body never mentions.

### Verifying findings

Every finding must be something observed, not something suspected.

Before reporting a bug, trace it to the specific line and state the input or
condition that triggers it. Where the behaviour can be executed, execute it:
write a scratch file outside the repository, run it, and delete it afterwards.

Watch how results are printed, so a correct implementation is not reported as
broken by a misleading representation.

Separate what was observed from what is inferred, in the wording of the finding
itself. A finding you could not confirm is reported as a question, not as a
defect. **Do not pad the report.** A short list of real findings is worth more
than a long list padded with speculation, and a review that finds nothing should
say so plainly.

## 7. Classify and report

One rule decides severity:

> **Blocking** if merging would leave the default branch wrong or broken.
> Everything else is non-blocking.

| Blocking | Non-blocking |
|---|---|
| a correctness defect | naming and style |
| a failing check | wording and documentation polish |
| a failing local build, or a conflicting merge | a missing test unless repository requirements make that omission blocking |
| a reference to something the change removed | duplication or structure |
| a concrete security vulnerability or leaked secret/credential | a body that undersells the change |
| a material performance regression that would make the merged behavior unacceptable | a minor efficiency concern with no meaningful user/system impact |
| an incompatible dependency/API change that breaks consumers or required builds | a dependency concern without demonstrated breakage |

Report in this shape, and keep it stable because `merge-pr` gates on it:

```text
Review of #<number>: <title>

Checks: <all passed | N failing, named | none ran>
Build:  <commands run and result | skipped, and why>

Blocking findings

1. <file>:<line> <what is wrong, and the condition that triggers it>
   <what was observed, and how>

Non-blocking findings

1. <file>:<line> <what is wrong>

Verdict: <no blocking findings | N blocking findings>
```

When there are no blocking findings, say **no blocking findings** in those words,
so the calling skill can rely on it.

Do not use `safe to merge` as the review verdict. Merge readiness also depends on
current CI, branch protection, draft state, approvals, and other gates owned by
`merge-pr`.

When checks are pending, make that visible in the `Checks:` line even if the code
review has no blocking findings.

Omit a section that has no entries rather than printing an empty heading.

## 8. Comment, only when asked

Without `--comment`, stop here. The findings live in your response.

With `--comment`, write the comment body to your harness's scratchpad or temp
directory, never inside the repository working tree:

```text
<scratchpad>/<repo-name>-review-<number>.md
```

Show the user the exact text, and ask:

```text
Do you want me to post this review as a comment on #<number>?
```

The comment is **not** the step 7 report pasted onto GitHub. Step 7 is a working
report for the user and for a calling skill; the comment is prose that lands on
the pull request, so it follows the same writing style the repository uses for
pull request bodies.

Use the pull request writing style from the repository instruction file when
present. Otherwise use this default, which mirrors the body style of the
`create-pr` skill:

- Two or three short paragraphs. Add one more only when a finding genuinely needs
  its own, and prefer merging small related findings into a single paragraph.
- The first paragraph states the outcome, opening with `This review found`,
  `This review raises`, or `This review found nothing blocking`.
- For a defect, describe the current behaviour with `As written, ...` and its
  consequence with `This means ...`, the same way a bugfix body uses
  `Before this, ...` and `This now ...`.
- The final paragraph covers non-blocking findings, or a compatibility note, when
  either is worth saying.
- **Do not** add markdown headings, checklists, or tables.
- **Do not** paste build or check output, and do not list commands as a
  transcript. Naming in prose what was run is fine, and is how the closing
  paragraph states what backs the review.
- **Do not** add meta statements about the review being thorough, limited, or
  automated.

Link every code reference so it is clickable from the pull request. Use a
permalink to the blob at the head commit, with the line anchor, and use the file
path as the link text:

```text
[`src/routing/compileRoutes.ts`](https://github.com/<slug>/blob/<sha>/src/routing/compileRoutes.ts#L47)
```

Resolve `<sha>` once from the pull request head:

```bash
gh pr view <number> --repo <slug> --json headRefOid -q .headRefOid
```

Link a range with `#L47-L52` when the defect spans lines. Keep the line number in
the anchor rather than in the prose, so the sentence stays readable. Use a fenced
block only when a short excerpt is the clearest way to show a defect, never to
reproduce a whole function.

State what backs the review. When no checks ran, or the build was skipped, say so
in the closing paragraph rather than leaving a reader to assume the findings came
from a full build.

Do not add reviewer/tool/model attribution, generated-by text, or automation footers.

Long reviews are the exception the paragraph limit does not cover. When there are
more than three or four findings, keep the opening and closing paragraphs and put
the findings in a numbered list between them, one short paragraph each, still in
the same voice. Never fall back to the step 7 layout.

Rules for the comment specifically:

- Write no preamble, greeting, sign-off, reviewer attribution, model name, or
  generated-by footer.
- Do not approve, request changes, or tell the author what to do next. The
  comment states findings; the decision belongs to whoever reads it.
- Link nothing that only exists locally, such as a scratch file path.
- Follow the repository's own writing conventions, including any rules its
  instruction files set about punctuation and phrasing.

Post only after approval:

```bash
gh pr comment <number> --repo <slug> --body-file "<body-file>"
```

Posting is visible to everyone with access to the repository, so never post
without approval, even when the user passed `--comment`. That flag opts into
being asked, not into posting.

This holds when another skill invokes this one. A calling skill can forward
`--comment`, and that still only reaches the approval prompt above. No caller can
cause a comment to be posted without the user seeing its text first.

Remove the temporary file after posting succeeds.

Never use `gh pr review`. This skill does not approve pull requests or request
changes through GitHub; it reports findings and leaves the decision to the user.

## Safety

- Never edit, stage, commit, or revert anything in the repository.
- Never merge, close, approve, or request changes on a pull request.
- Never leave a trial merge in progress; `git merge --abort` on every exit path.
- Never run `git branch -D` on anything except the `review-pr-<number>` scratch
  branch this skill created.
- Never stash, discard, or commit the user's uncommitted work; skip the build
  instead.
- Never post a comment without explicit approval.
- Never report a finding that was not verified, and never describe an inference
  as an observation.
- Never describe a pull request as verified when no checks ran and no build was
  performed.
- If the environment blocks a git or `gh` command, show the blocked command and
  ask the user to approve or run it. Never work around it by editing `.git`
  directly.

## Optional overrides

A repository's `AGENTS.md` or `CLAUDE.md` may override defaults. Recognised hints
live under `## Pull Request`, `## Pull Requests`, `## Review`, or
`## Build And Verification`:

| Hint | Affects |
|---|---|
| Required verification command per change scope | step 5 |
| Default branch, when not the GitHub-reported default | steps 1, 5 |
| Repository-specific review checklist or forbidden patterns | step 6 |
| What counts as blocking for this repository | step 7 |
| Pull request body style, which the posted comment follows | step 8 |
| Whether commenting on pull requests is permitted at all | step 8 |
