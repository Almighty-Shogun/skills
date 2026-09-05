<a href="https://shogun.ms" target="_blank" rel="noopener">
	<img src="https://cdn.shogun.ms/assets/branding/app-icon-256.svg" alt="Shogun app-icon" height="62"/>
</a>

---

# Skills
A curated collection of practical AI coding-agent skills for research, implementation, review, Git workflows, releases, documentation, and repository maintenance.

A skill is a folder with a `SKILL.md` inside it: frontmatter naming the skill,
then the instructions an agent follows when the skill is invoked. Larger skills
keep supporting material in a `references/` folder that is read only when the
task needs it, so a skill costs little context until it earns it.

Everything here is Markdown. There is no code, no build step, and no runtime
dependency.

## ✨ Features

Twenty skills, grouped by what they are for.

### 🔀 Git and GitHub

| Skill | Invocation | What it does |
|---|---|---|
| **commit** | `/commit [--auto-commit] [--auto-push] [--auto-all]` | Turns a mixed working tree into small commits grouped by intent, proposes the plan, stages narrowly, never amends or force-pushes. |
| **create-pr** | `/create-pr [fix\|feat\|improvement\|chore\|docs/<slug>]` | Opens a pull request on a new or existing work branch, inferring the branch name from the actual change. |
| **review-pr** | `/review-pr [<number-or-url>] [--comment] [--no-build]` | Reviews a pull request for bugs and drift from the repository's own instructions, and reports findings without changing anything. |
| **respond-pr** | `/respond-pr [<number-or-url>]` | Surfaces unresolved reviewer feedback, lets you choose what to address, and answers it. |
| **fix-pr** | `/fix-pr [<number-or-url>]` | Diagnoses failing GitHub Actions checks, groups them by root cause, and fixes them. |
| **merge-pr** | `/merge-pr [<number-or-url>] [--no-review] [--comment] [--auto-merge]` | Merges one pull request after checking state, CI and review, then cleans up the branch. |
| **release** | `/release [project] <major\|minor\|patch\|beta\|stable\|version>` | Cuts one GitHub release from a bump keyword or an explicit version, and lets CI publish. |
| **release-notes** | `/release-notes [project] [base-ref] [--file] [--redo <tag>]` | Generates evidence-based release notes from a diff, in the standard emoji house format. |

### 🔍 Code quality

| Skill | Invocation | What it does |
|---|---|---|
| **bughunt** | `/bughunt [--report\|--github] [--scope <scope>] [--limit <n>]` | Inspects a repository for genuine, reproducible bugs and verifies each candidate before reporting it. |
| **code-validation** | `/code-validation` | Reviews the current changes against requirements and repository behavior in an isolated agent. |
| **simplify-code** | `/simplify-code [--scope <scope>] [--report]` | Simplifies existing code while preserving behavior. |
| **structure-review** | `/structure-review [--scope <scope>]` | Reviews a repository or scoped area for structural and architectural problems. |
| **api-design** | `/api-design` | Designs public and cross-boundary API contracts without implementing them. |
| **csharp-docs** | `/csharp-docs [path ...] [--verify [--fix]]` | Writes or verifies C# XML documentation against an accuracy-first standard. |

### 🧭 Research and planning

| Skill | Invocation | What it does |
|---|---|---|
| **repository-research** | `/repository-research [--save] [scope or question]` | Investigates how a repository behaves without modifying it, separating confirmed facts from inference. |
| **source-research** | `/source-research [--quick\|--deep] <question>` | Researches a question against authoritative external sources in an isolated agent. |
| **plan-implementation** | `/plan-implementation [--save] <change to plan>` | Produces a repository-grounded implementation plan without touching code. |
| **delegate-task** | `/delegate-task [--interactive\|--autonomous] <task>` | Runs research, implementation and validation as isolated agents to keep the main context small. |

### 📚 Package references

| Skill | Invocation | What it does                                                                                                             |
|---|---|--------------------------------------------------------------------------------------------------------------------------|
| **shogun-node** | `/shogun-node [package] <what you want>` | Reference for the `@almighty-shogun/*` npm packages: ownership, APIs, traps, and the installed contract.                 |
| **shogun-nuget** | `/shogun-nuget [package] <what you want>` | Reference for the `AlmightyShogun.*` NuGet packages: ownership, wiring order, configuration, and the installed contract. |

## 📦 Requirements

- **An agent that supports skills**: Claude Code, Codex, and 75 others the
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

Pick the agent and the scope. Without `-g` the skills land in the current project
rather than your home directory:

```sh
npx skills add Almighty-Shogun/skills --skill '*' -a claude-code -g
npx skills add Almighty-Shogun/skills --skill '*' -a codex -a cursor -g -y
```

The installer offers a symlink or a copy. **Take the symlink.** It keeps one
source of truth behind every agent you installed to, while copies start drifting
apart the moment anything changes.

Refresh when this repository changes:

```sh
npx skills update -g
```

Start a new session and the skills are listed.

## 💻 Usage

Invoke a skill by name: `/commit` in Claude Code, `$commit` in Codex. Arguments
follow the invocation, and each skill's argument hint is in the table above:

```text
/commit --auto-all
/review-pr 42 --comment
/shogun-nuget AspNet.Auth how do I scope tokens per host
```

Most skills here are **manual only**: they carry `disable-model-invocation: true`
and run when you ask for them, never on the agent's own judgement. The two
package references, `shogun-node` and `shogun-nuget`, are the exception. They
load automatically when work genuinely depends on one of those packages, which is
what makes them useful without being asked for by name.

## 🧹 Uninstall

Remove one skill, several, or all of them:

```sh
npx skills remove commit
npx skills remove commit shogun-node
npx skills remove --all
```

Scope it to one agent when the same skill is installed for several:

```sh
npx skills remove commit -a claude-code
```

Check what is left with `npx skills list`.

## 🔧 Adding a skill

Create a folder under `skills/` whose name matches the `name` in the frontmatter,
since the agent resolves one by the other:

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
disable-model-invocation: true
---
```

- **`description`** carries the trigger. Say when to use it and when not to.
- **`argument-hint`** is shown in the invocation list.
- **`disable-model-invocation: true`** makes the skill manual only. Leave it out
  when the agent should reach for the skill on its own.

Put anything long in `references/` and tell the skill when to read it. A skill
that loads five files for a one-line question is a skill that costs more than it
saves.

Install a skill from a local clone to try it before pushing:

```sh
npx skills add ./skills --skill my-skill -a claude-code -g
```

## 📝 Notes

- **Nothing here is built or compiled.** A change is live the moment it is saved,
  as long as the skill is symlinked rather than copied.
- **Skills are agent neutral.** They describe intent and shell commands rather
  than harness APIs, so the same file works in Claude Code and Codex.
- **Repository instructions win.** Every skill defers to the `AGENTS.md` or
  `CLAUDE.md` of the repository it runs in.
- **The workflow skills are deliberately narrow.** They stage, commit, review,
  merge and release, and refuse to amend, rebase, force-push or publish on their
  own initiative.

## 🩺 Troubleshooting

**A skill does not appear.** Confirm it installed where you expected, and for
the agent you expected:

```sh
npx skills list
```

A skill installed without `-g` lives in the current project, so it is invisible
from anywhere else. Then start a new session, since skills are listed at session
start.

**A skill behaves like an older version.** It was installed as a copy rather than
a symlink, so this repository and the installed copy have drifted:

```sh
npx skills update -g
```

Reinstalling and choosing the symlink stops it happening again.

**A skill never triggers on its own.** That is the default. Everything except
`shogun-node` and `shogun-nuget` sets `disable-model-invocation: true` and has to
be invoked by name.

**A skill triggers when you did not want it.** Narrow the `description`. It is
the only thing read before the skill loads, so an over-broad description is what
makes a skill fire on unrelated work.
