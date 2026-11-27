<a href="https://shogun.ms" target="_blank" rel="noopener">
	<img src="https://cdn.shogun.ms/assets/branding/app-icon-256.svg" alt="Shogun app-icon" height="62"/>
</a>

---

# Skills
A curated collection of practical AI coding-agent skills for research,
implementation, review, Git workflows, releases, documentation, and repository
maintenance.

A skill is a folder with a `SKILL.md` inside it: frontmatter naming the skill,
then the instructions an agent follows when the skill is invoked. Larger skills
keep supporting material in a `references/` folder that is read only when the
task needs it, so a skill costs little context until it earns it.

Everything here is Markdown. There is no code, no build step, and no runtime
dependency.

## ✨ Features

Twenty-one skills, grouped by what they are for.

### 🔀 Git and GitHub

| Skill | Invocation | What it does |
|---|---|---|
| [**commit**](skills/commit/SKILL.md) | `/commit [--auto-commit] [--auto-push] [--auto-all] [--dir <dir>] [--message <msg>] [<path>...]` | Turns a mixed working tree into small commits grouped by intent, proposes the plan, stages narrowly, never amends or force-pushes. |
| [**create-pr**](skills/create-pr/SKILL.md) | `/create-pr [<fix\|feat\|improvement\|chore\|docs>/<slug>]` | Opens a pull request on a new or existing work branch, inferring the branch name from the actual change. |
| [**review-pr**](skills/review-pr/SKILL.md) | `/review-pr [<number-or-url>] [--comment] [--no-build]` | Reviews a pull request for bugs and drift from the repository's own instructions, and reports findings without changing anything, except a PR comment with `--comment` after your approval. |
| [**respond-pr**](skills/respond-pr/SKILL.md) | `/respond-pr [<number-or-url>]` | Surfaces unresolved reviewer feedback, lets you choose what to address, and answers it. |
| [**fix-pr**](skills/fix-pr/SKILL.md) | `/fix-pr [<number-or-url>]` | Diagnoses failing GitHub Actions checks, groups them by root cause, and fixes them. |
| [**merge-pr**](skills/merge-pr/SKILL.md) | `/merge-pr [<number-or-url>] [--no-review] [--comment] [--auto-merge]` | Merges one pull request after checking state, CI and review, then cleans up the branch. |
| [**release**](skills/release/SKILL.md) | `/release [project] <major\|minor\|patch [beta]\|stable\|version>` | Cuts one GitHub release from a bump keyword or an explicit version, and lets CI publish. |
| [**release-notes**](skills/release-notes/SKILL.md) | `/release-notes [project] [base-ref \| --redo <tag>] [--head <sha> --tag <id> [--prerelease]] [--file]` | Generates evidence-based release notes from a diff, in the standard emoji house format. |

### 🔍 Code quality

| Skill | Invocation | What it does |
|---|---|---|
| [**bughunt**](skills/bughunt/SKILL.md) | `/bughunt [--report\|--github] [--solution] [--scope <scope>] [--limit <n>]` | Inspects a repository for genuine, reproducible bugs and verifies each candidate before reporting it. |
| [**code-validation**](skills/code-validation/SKILL.md) | `/code-validation [<paths> \| <commit> \| <base>...<head>]` | Reviews the current changes against requirements and repository behavior in an isolated agent. |
| [**simplify-code**](skills/simplify-code/SKILL.md) | `/simplify-code [--scope <scope>] [--report]` | Simplifies existing code while preserving behavior. |
| [**structure-review**](skills/structure-review/SKILL.md) | `/structure-review [--scope <scope>]` | Reviews a repository or scoped area for structural and architectural problems. |
| [**api-design**](skills/api-design/SKILL.md) | `/api-design <contract to design or review>` | Designs public and cross-boundary API contracts without implementing them. |
| [**csharp-docs**](skills/csharp-docs/SKILL.md) | `/csharp-docs [path ...] [--verify [--fix]]` | Writes or verifies C# XML documentation in a compact, accuracy-first style. |

### 🧭 Research and planning

| Skill | Invocation | What it does |
|---|---|---|
| [**repository-research**](skills/repository-research/SKILL.md) | `/repository-research [--save] [scope or question]` | Investigates how a repository behaves without lasting changes, separating confirmed facts from inference. |
| [**source-research**](skills/source-research/SKILL.md) | `/source-research [--quick\|--deep] <research question>` | Researches a question against authoritative external sources in an isolated agent. |
| [**plan-implementation**](skills/plan-implementation/SKILL.md) | `/plan-implementation [--save] <change to plan>` | Produces a repository-grounded implementation plan without lasting code changes. |
| [**delegate-task**](skills/delegate-task/SKILL.md) | `/delegate-task [--interactive\|--autonomous] <task>` | Runs research, implementation and validation as isolated agents to keep the main context small. |

### 🧰 Skill maintenance

| Skill | Invocation | What it does |
|---|---|---|
| [**skill-feedback**](skills/skill-feedback/SKILL.md) | `/skill-feedback [--review [<skill>...] \| <signal>]` | Logs signals that a skill should change or that a new skill should exist, and reviews them into approved skill changes. |

### 📚 Package references

| Skill | Invocation | What it does |
|---|---|---|
| [**shogun-node**](skills/shogun-node/SKILL.md) | `/shogun-node [package] <what you want>` | Reference for the `@almighty-shogun/*` npm packages: ownership, APIs, traps, and the installed contract. |
| [**shogun-nuget**](skills/shogun-nuget/SKILL.md) | `/shogun-nuget [package] <what you want>` | Reference for the `AlmightyShogun.*` NuGet packages: ownership, wiring order, configuration, and the installed contract. |

## 📦 Requirements

- **An agent that supports skills**: Claude Code, Codex, or any other agent the
  installer knows about.
- **Node.js**, to run `npx skills`.
- **Git**, for every skill that reads a working tree.
- **GitHub CLI (`gh`), authenticated**, for the pull request and release skills.

Individual skills assume the toolchain of the project they run in, such as
`dotnet` for `csharp-docs` or `bun` for a Node project. Nothing needs installing
for the skills themselves.

## 🚀 Installation

Install with [`skills`](https://github.com/vercel-labs/skills), which reads this
repository straight from GitHub and writes into whichever agent directories you
pick.

Install every skill:

```sh
npx skills add Almighty-Shogun/skills --skill '*'
```

Or see what is here first, and choose:

```sh
npx skills add Almighty-Shogun/skills --list
npx skills add Almighty-Shogun/skills
```

Install one skill, or a few:

```sh
npx skills add Almighty-Shogun/skills --skill commit --skill shogun-nuget
```

Some skills hand work to others, so install those with them: `create-pr` needs
`commit`; `release` needs `commit` and `release-notes`; `merge-pr` needs
`review-pr`; `respond-pr` needs `commit` and `review-pr`; `fix-pr` follows the
resolution rules of `review-pr`.

Pick the agent and the scope. Without `-g` the skills land in the current
project rather than your home directory:

```sh
npx skills add Almighty-Shogun/skills --skill '*' -a claude-code -g
npx skills add Almighty-Shogun/skills --skill '*' -a codex -a cursor -g -y
```

When the agents you pick use more than one skills directory and neither
`--copy` nor `-y` is given, the installer offers a symlink or a copy. **Take the
symlink.** It keeps one installed copy under `.agents/skills/` behind every
agent, so the agents stay consistent with each other. Neither choice tracks
this repository: an install is a snapshot.

Refresh when this repository changes, in the scope you installed to:

```sh
npx skills update -g   # global
npx skills update -p   # current project
```

A local-path install is never updated; re-run its `npx skills add` instead.

Start a new session and the skills are listed.

## 💻 Usage

Invoke a skill by name: `/commit` in Claude Code, `$commit` in Codex. Arguments
follow the invocation, and each skill's argument hint is in the table above:

```text
/commit --auto-all
/review-pr 42 --comment
/shogun-nuget AspNet.Auth how do I scope tokens per host
```

Two skills are **manual only**: `merge-pr` and `release` carry Claude Code's
`disable-model-invocation: true`, and for agents that ignore that key their
`description` says to use them only when you invoke them by name.
`create-pr`, `code-validation` and `simplify-code` can be started by the agent,
but their `description` limits them to when you or the repository instructions
explicitly ask for them. The others can also load on the agent's own judgement
when a task matches their `description`. That is what lets the two
package references, `shogun-node` and `shogun-nuget`, load when work genuinely
depends on one of those packages, and `skill-feedback` log a signal the moment a
skill is corrected or falls short.

## 🧹 Uninstall

Remove one skill or several, repeating the scope used at install (`-g` for the
global install above):

```sh
npx skills remove commit -g
npx skills remove commit shogun-node -g
```

`npx skills remove --all -g` removes every globally installed skill, including
those from other repositories, without asking.

Scope it to one agent when the same skill is installed for several:

```sh
npx skills remove commit -a claude-code -g
```

Check what is left with `npx skills list` and `npx skills list -g`.

## 🔧 Adding a skill

Create a folder under `skills/` whose name matches the `name` in the
frontmatter, since the agent resolves one by the other:

```text
skills/<name>/
├── SKILL.md
└── references/        (optional, read on demand)
```

`SKILL.md` starts with frontmatter:

```yaml
---
name: my-skill
description: >-
  What the skill does and when to use it. This is the only part the agent reads
  before deciding to load the skill, so it decides whether it ever runs.
argument-hint: "[--flag] <argument>"
---
```

- **`description`** carries the trigger. Say when to use it and when not to.
- **`argument-hint`** is shown in Claude Code's autocomplete.
- **`disable-model-invocation: true`** makes the skill manual only in Claude
  Code, which then keeps the `description` out of the model's context. Other
  agents ignore the key, so also end the `description` with "Use only when the
  user invokes it by name." Leave both out when the agent should reach for the
  skill on its own.

Put anything long in `references/` and tell the skill when to read it. A skill
that loads five files for a one-line question is a skill that costs more than it
saves.

Prose wraps at 80 columns; tables, code blocks, URLs, and frontmatter values
other than `description` (such as `argument-hint` and `license`) are exempt.
Write the `description` in the `>-` folded style shown above. Emojis belong in
this README, and in a skill only as part of an output format it produces (such
as the `release-notes` headings).

Install a skill from a local clone to try it before pushing:

```sh
npx skills add ./skills --skill my-skill -a claude-code -g
```

## 📝 Notes

- **Nothing here is built or compiled.** A change reaches agents once it is
  pushed and `npx skills update` runs, or once the local `npx skills add` runs
  again; editing the clone alone changes nothing an agent loads.
- **Skills are agent neutral.** They describe intent and shell commands rather
  than harness APIs, so the same file works in Claude Code and Codex.
- **Repository instructions win.** Every skill reads both the `AGENTS.md` and
  the `CLAUDE.md` of the repository it runs in, not only the one the agent
  loads itself, and defers to them.
- **The workflow skills are deliberately narrow.** They stage, commit, review,
  merge and release, and refuse to amend, rebase, force-push or publish on their
  own initiative.

## 🩺 Troubleshooting

**A skill does not appear.** Confirm it installed where you expected, and for
the agent you expected:

```sh
npx skills list
npx skills list -g
```

A skill installed without `-g` lives in the current project, so it is invisible
from anywhere else. Then start a new session. Claude Code picks up added or
edited skills within the session; run `/reload-skills` there if one still does
not appear.

**A skill behaves like an older version.** The installed snapshot is older than
this repository. Update the scope it was installed in, or re-run the
`npx skills add` of a local-path install:

```sh
npx skills update -g
npx skills update -p
```

**A skill never triggers on its own.** Check its frontmatter. `merge-pr` and
`release` set `disable-model-invocation: true` for Claude Code, and their
`description` limits them to an invocation by name elsewhere.
`create-pr`, `code-validation` and `simplify-code` trigger only on an explicit
request from you or the repository instructions. Any other skill triggers only
when the task matches its `description`.

**A skill triggers when you did not want it.** Narrow the `description`. It is
the only thing read before the skill loads, so an over-broad description is what
makes a skill fire on unrelated work.
