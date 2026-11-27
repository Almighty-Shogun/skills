---
name: skill-feedback
description: >-
  Log signals that a skill should change or that a new skill should exist, and
  review them with --review. Log when the user corrects what a skill did, a
  skill misses a case or its instructions are unclear, or a manual sequence
  with no skill covering it runs for the second time in a session. Not for a
  one-off preference or a skill that worked as written. Logging never edits a
  skill; a review proposes changes and applies only approved ones.
argument-hint: "[--review [<skill>...] | <signal>]"
---

# Skill Feedback

Collect evidence for improving skills while working, and turn it into skill
changes in a periodic review. Without `--review`, append one row per distinct
signal to the inbox; with it, read the inbox and propose changes. Text after
`--review` names the skills to review; an unknown flag is reported and nothing
else happens. Free text in the arguments is the signal; without it, take the
signals from the current session. With no signal and nothing in the session,
log nothing and say so.

Read every `AGENTS.md` and `CLAUDE.md` that applies and follow them; they
override this skill's defaults, and when two conflict, ask which wins.

Writing or editing the skills that come out of a review follows the skills
repository's own conventions. Committing them belongs to
`commit --dir <source repo>` with the edited skill paths.

## Inbox

The inbox is `~/.skill-feedback/inbox.md`, unless the user's instructions name
another location. It lives outside any skill folder, so reinstalling or
updating skills never touches it. Read it only to log or review. When a write
to it is blocked or fails, skip it with one line; never retry or work around
it.

When it does not exist, create it, with its directory, from this template
before logging:

```markdown
# Skill Feedback Inbox

Signals logged by the `skill-feedback` skill, which defines the row format and
the review.

## Reviewed

| Date | Skill | Type | Signal | Outcome |
|---|---|---|---|---|

## Inbox

| Date | Skill | Type | Signal |
|---|---|---|---|
```

Inbox holds open signals; Reviewed holds signals a review has handled. When a
section or its table is missing, logging restores it from the template without
rewriting other content. A review of a missing inbox reports it empty instead
of creating it.

## Logging

| Type | Log when |
|---|---|
| `correction` | The user corrects what a skill did, or how it did it. |
| `gap` | A skill did not cover a case it should have, or its instructions were unclear. |
| `new skill` | A manual sequence with no skill covering it runs for the second time in this session. |

A session cannot see earlier sessions, so the inbox carries the count: log a
`new skill` signal once per session, at the second run, and let the rows from
separate sessions add up for the review.

Infer the skill and type of a free-text signal from the session; when either
cannot be inferred, log nothing and say so in one line. A correction of a row
just logged is not a new signal: remove that row instead.

Append one row to the end of the Inbox table:

```text
| YYYY-MM-DD | <skill, or - for a new skill> | <type> | <signal> |
```

The signal is one line: what happened and what should change. No transcripts,
code excerpts, secrets, or personal data. Write any `|` in it as `\|`, so the
table keeps its columns. A repeat of an open signal gets its own row, since
repetition is the evidence a review weighs.

Logging does not interrupt the current work: no question to the user, no edit
to any skill, and at most one line in the reply naming the logged skill.

## Review

With `--review`:

1. Read the Inbox table and group the rows by skill, a renamed skill under its
   new name, and `-` rows by the sequence they describe. Count Reviewed rows
   marked `dropped: one-off` toward their group. Repeated signals outweigh
   single ones. List rows that cannot be parsed and leave them in place.
2. Resolve the skills' source repository from the user's instructions; when
   they name none, ask for it before reading (an unattended run reads the
   installed copy and says so). For each group, read the skill in its source
   and decide: change it, create a new skill, or drop the signals because they
   were one-off, are already fixed, conflict with what the skill is for, or
   concern a skill not maintained here.
3. Propose the plan and wait for approval. A change that is too vague to act on
   is reported as such, not guessed at.
4. Apply only the approved changes, in the style of the existing skills, and
   nothing beyond what the signals support. Edit and create skills in their
   source repository, never in an installed copy: install directories such as
   `~/.claude/skills` or `~/.agents/skills` are overwritten by the next update,
   and a symlink there can point to another installed copy. Without a source
   from step 2, leave the change unapplied.
5. Move each handled row to the Reviewed table with an outcome:
   `changed <skill>`, `new skill <name>`, `dropped: <reason>`, or
   `dropped: not maintained here`. Rows that were not approved, not applied, or
   too vague to act on stay in the Inbox.

An unattended run, with no one to approve, proposes only: it returns the plan
and leaves the inbox and the skills unchanged.

## Output

Logging: nothing, or one line such as `Logged a skill-feedback signal for
commit.`

Review:

```text
Skill Feedback Review
1. commit — change — <what changes, from which rows>
2. - — new skill <name> — <what it covers>
3. review-pr — drop — <reason>
4. fix-pr — unclear — <why it is too vague to act on>
<closing question, such as the source repository, when one is open>
```

or `Inbox empty. Nothing to review.` After approval, report the changed files
and the rows moved, and name the refresh step: installed copies keep the old
text until the source is published and the skills are updated. No walkthrough
of each edit.
