# Vibe Kanban

A local control plane for building software with coding agents.

Vibe Kanban gives an AI coding agent the one thing it normally lacks: a durable record of what it was asked to do, what it decided, what you approved, and what it changed. Tickets, plans, approvals, questions, progress, commits, and pull requests live in a single SQLite file next to your code. The agent reads and writes it through a CLI; you read and steer it through a local web UI.

It runs entirely on your machine. Nothing is sent anywhere except to the model provider you configure yourself.

```
                      ┌──────────────────────────────┐
  you ───────────────▶│  Web UI at 127.0.0.1:8765    │
                      │  board · agent panel · ops   │
                      └──────────────┬───────────────┘
                                     │
                      ┌──────────────▼───────────────┐
  Claude Code ───────▶│  Vibe Kanban                 │──▶ .vibe-kanban/vibe-kanban.sqlite
  Codex       ───────▶│  CLI · server · gateway      │──▶ .agents/ (skills, workflows)
                      └──────────────┬───────────────┘
                                     │
                            your model provider
```

---

## Table of contents

- [What this repository contains](#what-this-repository-contains)
- [Requirements](#requirements)
- [Install from scratch](#install-from-scratch)
- [Where your work lives](#where-your-work-lives)
- [Connect a model provider](#connect-a-model-provider)
- [Connect a coding agent](#connect-a-coding-agent)
- [The working loop](#the-working-loop)
- [Workflows](#workflows)
- [Skills](#skills)
- [Instructions: AGENTS.md and PROJECTS.md](#instructions-agentsmd-and-projectsmd)
- [Operations](#operations)
- [Security and your data](#security-and-your-data)
- [Updating](#updating)
- [CLI reference](#cli-reference)
- [Environment variables](#environment-variables)
- [Troubleshooting](#troubleshooting)

---

## What this repository contains

This is the **compiled distribution**. The TypeScript and React sources are maintained separately and are not published here; what you get is everything needed to run it:

| Path                           | What it is                                                                                        |
| ------------------------------ | ------------------------------------------------------------------------------------------------- |
| `cli/vibe-kanban.mjs`          | The whole application — CLI, HTTP server, Socket.IO, and the model gateway, bundled into one file |
| `assets/dist/`                 | The compiled web UI                                                                               |
| `.agents/workflows/`           | What the agents do, stage by stage. Plain Markdown you can edit                                   |
| `.agents/skills/`              | How an agent discovers what this workspace can do                                                 |
| `.agents/templates/`           | Starting point for a skill of your own                                                            |
| `AGENTS.md`                    | The rules every agent reads before it starts                                                      |
| `scripts/update-workspace.mjs` | Pulls a newer release into your workspace without losing your edits                               |

Everything under `.agents/` is yours to change. That is the point: behaviour lives in Markdown you own, not in the binary.

---

## Requirements

**Required**

- **Node.js 22.5 or newer.** The database layer uses Node's built-in `node:sqlite`, which does not exist in earlier versions. Check with `node --version`.
- **pnpm** (recommended) or npm. Install pnpm with `npm install -g pnpm` or `corepack enable`.
- **git**, for updates and for anything the agent does with branches and commits.

**Optional, but this is most of the value**

- **A coding agent CLI** on your `PATH` — [Claude Code](https://claude.com/claude-code) (`claude`) or [Codex](https://developers.openai.com/codex/cli) (`codex`). Without one you still get the board, the tickets, and the workflows; with one you get agent sessions driven from the UI.
- **A model provider.** Either an Anthropic account you sign into with `claude`, or an API key for any OpenAI- or Anthropic-compatible endpoint.
- **GitHub CLI** (`gh`), if you want the agent to open pull requests.

Vibe Kanban runs on macOS, Linux, and Windows (WSL recommended on Windows).

---

## Install from scratch

### 1. Get the repository

```bash
git clone <this-repository-url> vibe-kanban
cd vibe-kanban
```

Keep the `.git` directory. It is how updates are delivered — there is no separate updater channel or version file.

### 2. Install dependencies

```bash
pnpm install --prod
```

or, with npm:

```bash
npm install --omit=dev
```

Nothing is compiled at install time; this only fetches the runtime packages the bundled server needs.

### 3. Start it

```bash
pnpm start
```

You should see:

```
Vibe Kanban running at http://127.0.0.1:8765
SQLite database: /path/to/vibe-kanban/.vibe-kanban/vibe-kanban.sqlite
```

Open <http://127.0.0.1:8765>. The database file is created on first run.

To use a different port or interface:

```bash
pnpm vk serve --port 9000
```

Binding to anything other than loopback is refused until you turn on LAN mode and set a password — see [Security and your data](#security-and-your-data).

### 4. Put something on the board

An empty board is hard to judge. This creates a small set of example tickets so you can see the shape of things:

```bash
pnpm vk seed
```

Refresh the browser. Delete them whenever you like with `pnpm vk delete <ticket-id> --cascade`.

### 5. Stop it

`Ctrl+C`. Everything is in `.vibe-kanban/`; nothing else on your machine was touched except the encryption key described under [Security and your data](#security-and-your-data).

---

## Where your work lives

Vibe Kanban needs to know which directory is "the workspace" — where `.agents/` is read from and where `.vibe-kanban/` is written. It works this out by walking up from the current directory, looking for `.vibe-kanban/vibe-kanban.sqlite`, then `.vibe-kanban`, then `.git`. So commands work from any subdirectory of your project, not just its root.

There are two sensible layouts.

### Layout A — this repository is the workspace

Good for a new project, or for trying it out. Put your code under `source/`, or anywhere inside the clone.

```
vibe-kanban/            ← you started the server here
├── .agents/            ← workflows and skills
├── .vibe-kanban/       ← the database
├── source/             ← your project
└── cli/ assets/ ...
```

Nothing more to do. This is what you already have after `pnpm start`.

### Layout B — your existing repository is the workspace

Better when you already have a project with its own git history. Copy the agent configuration into it and run the bundled CLI from inside it:

```bash
cd /path/to/your-project
cp -R /path/to/vibe-kanban/.agents .
cp /path/to/vibe-kanban/AGENTS.md .

node /path/to/vibe-kanban/cli/vibe-kanban.mjs serve
```

The database lands at `/path/to/your-project/.vibe-kanban/`, the workflows are the copies you now own, and the agent sees your code. Add `.vibe-kanban/` to your project's `.gitignore` unless you want the ticket history committed — it is a legitimate choice either way, and a team that wants shared history commits it.

A shell alias makes this comfortable:

```bash
alias vk='node /path/to/vibe-kanban/cli/vibe-kanban.mjs'
```

If you would rather not copy anything, point at the workspace and the database explicitly instead — they are separate settings, and both are needed:

```bash
VIBE_KANBAN_WORKSPACE=/path/to/your-project \
VIBE_KANBAN_DB=/path/to/your-project/.vibe-kanban/vibe-kanban.sqlite \
  pnpm start
```

---

## Connect a model provider

Vibe Kanban talks to models through its own small gateway: you register providers and models once, and everything else — the agent panel, the assistant that reviews plans and diffs, the OpenAI-compatible proxy — goes through them. It is vendor-neutral; there is no built-in account and no telemetry.

Open **Settings → Providers** and choose **Add provider**.

### Option 1 — sign in with your Claude account

If you already use Claude Code, you do not need an API key at all.

1. In a terminal, run `claude auth login` and complete the sign-in.
2. In **Settings → Providers**, add a provider with:
   - **Protocol**: `anthropic-compatible`
   - **Authentication**: _Claude account on this machine_
3. Save, then press **Probe**. A healthy provider reports the account it found.

Vibe Kanban shells out to `claude auth status` to check this and never reads or stores your credentials. If the probe says you are not logged in, run `claude auth login` again.

### Option 2 — an API key, stored encrypted

Works with Anthropic, OpenAI, OpenRouter, a company gateway, a local llama.cpp or Ollama endpoint — anything that speaks one of the two wire formats.

1. Go to **Settings → Credentials** and add the key. It is encrypted with AES-GCM before it touches the database, and no read API ever returns it.
2. Go to **Settings → Providers → Add provider** and fill in:
   - **Protocol**: `openai-compatible` or `anthropic-compatible`
   - **Base URL**: for example `https://api.openai.com/v1` or your gateway's URL
   - **Authentication**: _API key from the vault_, then pick the credential you just created
   - **API surface**: `responses` (the default) or `chat`. If requests fail with a 404 on the path, this is usually the setting to change
   - **Timeout**: milliseconds, between 1,000 and 300,000
3. Save and press **Probe**.

> Secrets belong in the vault. The extra environment variables a provider can set are for things like region or project id — the form refuses anything that looks like a key.

### Add models

With a healthy provider, open it and choose **Load models**. Vibe Kanban asks the provider what it serves and lists them; tick the ones you want. If a provider does not expose a model list, add them by hand — you supply the upstream id, a display name, the context window, and what the model can do (streaming, tools, vision, reasoning). Pricing is optional and only feeds the cost estimates on the Operations page.

### Routes

A route is a stable alias — `default`, `fast`, `reviewer` — pointing at a model, so you can swap the model underneath without touching anything else. Routing policies decide how a route resolves: a fixed model, by capability, a fallback chain, weighted, or learned from past runs.

### Use it as an OpenAI-compatible endpoint

Any tool that speaks the OpenAI API can point at Vibe Kanban:

```bash
curl http://127.0.0.1:8765/v1/models
curl http://127.0.0.1:8765/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"default","messages":[{"role":"user","content":"hello"}]}'
```

Usage, cost, and errors for these calls show up on the Operations page alongside everything else.

---

## Connect a coding agent

The agent panel and ticket dispatch drive a real coding agent on your machine.

1. **Install one.** `claude` or `codex` must be on your `PATH`. Check with `which claude` or `which codex`.
2. **Create the folder it reads.** A skill becomes visible to an agent through a symlink from that agent's own skills directory. Create the folder once in your workspace root and the link toggles turn on:

   ```bash
   mkdir -p .claude/skills    # for Claude Code
   mkdir -p .codex/skills     # for Codex
   ```

3. **Link the skills.** Open the **Skills** page and switch on _Claude Code_ or _Codex_ for each skill you want that agent to see. The symlink points into `.agents/skills/`, so editing a skill changes it everywhere at once.

4. **Open a session.** With at least one healthy provider configured, an agent button appears in the app bar. The panel is docked, not modal — drag its edge to resize, keep working on the board beside it. On a ticket, **Agent dispatch** starts a session already carrying that ticket's plan and context; it stays hidden while no provider can take the work.

Conversations survive a reload: reopening a session resumes the thread the agent was already in rather than starting a blank one.

---

## The working loop

This is the shape of a day's work. Every step is visible in the UI, and every step has a CLI equivalent the agent uses.

**1. Ask for something.** In an agent session: _"read this issue and plan it"_, _"implement PSS-1421"_, _"write the product docs from these notes"_, _"tell me what this codebase already does"_. The request routes to the workflow that matches.

**2. The agent explores and writes a plan.** It reads the code, asks you anything genuinely ambiguous, and writes an execution plan onto the ticket. It does not write code yet.

**3. You approve.** On the ticket, or:

```bash
pnpm vk approve <ticket-id> --actor "you"
```

Nothing is implemented before this. Approval applies to **task** tickets — the leaves that get implemented — not to the groups and features above them.

**4. It implements.** Branch, change, verify, with progress written back to the ticket as it goes. The plan stepper follows along.

**5. You review locally.** The agent asks for a local review before it commits anything. Confirm it, or ask for changes — either answer is delivered straight into the agent's open session, so it picks the work back up without you retyping anything.

```bash
pnpm vk local-review <ticket-id> --status confirmed
pnpm vk local-review <ticket-id> --status changes_requested --description "the empty state still shows a spinner"
```

**6. It commits and opens the PR.** Commit hashes, branch, PR number, and pipeline status are all recorded on the ticket, so the ticket is the record of what happened.

**When the agent needs a human decision** it does not guess. It parks the ticket on `hold` and asks, tagged by who can answer — `requirement` (BA, product), `design` (UX), `technical` (tech lead), `operations` (DevOps, PM). Each question is answered where it is asked on the ticket page, and where the question already names its options, they appear as buttons.

```bash
pnpm vk questions <ticket-id> --category requirement --question 1 --answer "Soft delete, keep 30 days"
```

---

## Workflows

What the agents do is not in the binary. It is seven Markdown documents in `.agents/workflows/`, and you can change every one of them.

| Workflow              | Runs when you ask for                                                                                               |
| --------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `task-planning`       | Implementing a group, feature, task, or an external issue: reads it, explores the code, writes the plan you approve |
| `task-implementation` | The half that writes code: branch, implement, verify, local review, commit, PR                                      |
| `documentation`       | Product documentation from notes you provide                                                                        |
| `codebase-bootstrap`  | Inferring stories from code that already exists                                                                     |
| `design-extraction`   | Capturing the project's visual system into `DESIGN.md`                                                              |
| `clarification`       | Getting an answer only a person can give                                                                            |
| `github-issue-source` | Reading a GitHub issue onto a ticket, and posting questions back to it                                              |

Each document names its stages, the branches between them, the gate where work stops for you (plan approval, local review, ask the user), the base branch, the scope, and the instructions the agent follows in that stage.

Edit them on the **Workflows** page. The **Graph** tab is a node canvas — drag cards to arrange, drag from a port onto another card to connect, fork with branch nodes, call another workflow with workflow nodes. **Stages** is the same thing as a list, and **Document** is the file the agent actually reads. Or just edit the Markdown in your editor; both are the same file.

```bash
pnpm vk workflows              # every workflow
pnpm vk workflow task-planning # the document an agent follows
```

A task that is already running keeps following the version it started with. When you change a workflow **in the UI**, the agent is told on its next command for that task to run `pnpm vk reload <task-id>`, which prints what changed. Editing the same file in your editor deliberately does not interrupt running work.

---

## Skills

A skill is how an agent discovers what this workspace can do. Each one is a few lines — a name, a description the runtime surfaces, and the workflow it runs:

```markdown
---
name: task-implementer
description: Turn Vibe Kanban or external source tickets into approval-ready task tickets, then implement approved tasks.
workflow: task-implementation
---

Run `pnpm vk workflow task-implementation` and follow it stage by stage.
```

The steps are not in the skill, so a skill cannot drift from the behaviour you edit on the Workflows page. Seven ship with this release: `task-implementer`, `documenter`, `doc-collector`, `design-collector`, `clarifier`, `github-source`, and `vibe-kanban`.

The same shape covers connectors to systems this workspace does not talk to itself — Jira, Teams, Slack, Linear. `github-source` is the worked example, and `.agents/templates/plugin-skill/` is the starting point for your own.

### Skill marketplace

Skills other people wrote install from any git repository that holds directories with a `SKILL.md`. On the **Skills** page, press **Marketplace**.

Paste either a clone URL or **the link you copied from a folder on GitHub, GitLab, or Bitbucket** — a browse link such as `https://github.com/anthropics/skills/tree/main/skills/frontend-design` is split back into the repository, the branch, and the folder, and the folder is searched instead of the whole repository. That matters when a repository holds a hundred skills and you want one.

The same from the CLI:

```bash
pnpm vk marketplace add --url https://github.com/anthropics/skills/tree/main/skills/frontend-design
pnpm vk marketplace browse
pnpm vk marketplace install --entry skills/frontend-design
pnpm vk marketplace update --name frontend-design
```

Installing copies the directory into `.agents/skills/` and records where it came from in a `.marketplace.json` beside the `SKILL.md`, so the skill shows its origin on the Skills page and can be updated to the repository's newest commit later.

A checkout is not trusted. An entry path cannot climb out of the repository, symlinks are refused, a skill may bring at most 200 files or 2 MB, nothing is written until the whole tree passes those checks, and a skill you wrote yourself is never replaced without being asked. Cloning runs without a credential prompt, so a private repository needs its credentials already configured for git.

> A new or removed skill is discovered when an agent session **starts**. Restart the agent after installing one. Editing the content of a skill does not need a restart.

---

## Instructions: AGENTS.md and PROJECTS.md

`AGENTS.md` at the workspace root is the standing brief every agent reads: how this workspace works and what the rules are. `PROJECTS.md` is the per-project half — stack, conventions, deployment, anything specific to the thing you are building.

The **Instructions** page shows both side by side as editable Markdown with a live preview, creates `PROJECTS.md` when it is missing, and refuses to overwrite a document that changed on disk after the page loaded.

---

## Operations

The **Operations** page is the live view of everything the gateway is doing: token usage and estimated cost, provider health, route decisions, errors, active agent sessions, and a cancel button for each.

Spending and concurrency caps are set under **Settings → Limits & retention**, either globally or scoped to one provider, model, runtime, or routing policy. A request over its limit is rejected _before_ the provider is called and writes a redacted audit record. History retention is configured in the same place.

Dispatch runs hold a lease that heartbeats while the child process is alive, so a worker that dies is cancelled rather than left running forever.

---

## Security and your data

**It listens on loopback only.** `127.0.0.1` is the default, and binding to anything else is refused outright unless you enable LAN mode under **Settings → Network & access** _and_ set a password of at least twelve characters. With that on, authentication covers the API, the proxy, agent controls, and the realtime connection.

**Credentials are encrypted.** API keys are sealed with AES-GCM under a 32-byte master key before they reach the database, and no read API returns a secret.

**The master key is stored outside the workspace**, at `~/.config/vibe-kanban/master.key` in a directory created with `0700`. It is generated on first use.

- To manage it yourself, set `VIBE_KANBAN_MASTER_KEY` (32 bytes, base64 or 64 hex characters) or point `VIBE_KANBAN_MASTER_KEY_FILE` at a file.
- **Back it up separately from the database.** The database alone cannot decrypt anything.
- Losing it means re-entering your API keys; nothing else is lost.

**Everything else is one file.** `.vibe-kanban/vibe-kanban.sqlite` holds tickets, history, settings, and the encrypted credentials. To back up, stop the server and copy that file and the master key. To move machines, copy both.

---

## Updating

A newer release is delivered as commits on this repository. From your workspace root:

```bash
pnpm workspace:update --dry-run   # show exactly what would change
pnpm workspace:update             # apply it
```

The updater fetches `origin/main`, checks that it merges cleanly, and merges it into the branch you are on. It then verifies the dependencies and the compiled artifacts.

- **Your edits survive.** Committed changes stay as branch history; uncommitted tracked and untracked files, including your own skills under `.agents/skills/`, are preserved.
- **Your data is untouched.** The database, `node_modules`, and your project source are never modified.
- **Conflicts stop it.** If an incoming change conflicts with a customization of yours, the updater stops _before_ changing anything and tells you which files. `--force` accepts the new version for those files and makes you type `APPLY` first.
- **`.git` is required.** Git history and the remote are the entire update state. If `.git` is missing, clone this repository again.

To change which repository or branch you track:

```bash
pnpm workspace:update --remote upstream --branch stable
pnpm workspace:update --remote-url https://github.com/owner/repository.git
```

---

## CLI reference

Run any command with `pnpm vk <command>`. Pass options straight through — do **not** put `--` in front of them.

```bash
pnpm vk --help              # every command and option
pnpm vk serve --port 8765   # start the server
```

Reading:

```bash
pnpm vk list                                          # the board
pnpm vk get <ticket-id> --json                        # one ticket in full
pnpm vk events <ticket-id>                            # its history
pnpm vk activity --limit 50                           # everything, newest first
pnpm vk smart-search "<query>" --parent-for task --json
pnpm vk detect-kind "<task request>" --json
```

Moving work along:

```bash
pnpm vk create --title "<title>" --raw-requirement @notes.md
pnpm vk approve <ticket-id> --actor "you"
pnpm vk start <ticket-id> --workflow task-implementation
pnpm vk progress <ticket-id> --percent 60 --note "API done, UI left"
pnpm vk local-review <ticket-id> --status confirmed
pnpm vk review <ticket-id>
pnpm vk close <ticket-id>
pnpm vk delete <ticket-id> --cascade
```

Talking to the agent:

```bash
pnpm vk comment <ticket-id> --comment "use the existing retry helper" --actor user
pnpm vk questions <ticket-id> --category requirement --questions @questions.md
pnpm vk questions <ticket-id> --category requirement --question 1 --answer "Soft delete"
pnpm vk reload <ticket-id>
```

Workspace:

```bash
pnpm vk skills
pnpm vk workflows
pnpm vk workflow <name>
pnpm vk marketplace browse
```

Any option taking text also takes `@file` to read it from a file, and `--json` is available almost everywhere for machine-readable output.

---

## Environment variables

| Variable                      | Effect                                                                                                  |
| ----------------------------- | ------------------------------------------------------------------------------------------------------- |
| `VIBE_KANBAN_DB`              | Path to the SQLite file. Default: `<workspace>/.vibe-kanban/vibe-kanban.sqlite`                         |
| `VIBE_KANBAN_WORKSPACE`       | Which directory `.agents/` is read from. Default: the nearest ancestor holding `.vibe-kanban` or `.git` |
| `VIBE_KANBAN_MASTER_KEY`      | Supply the credential encryption key yourself. 32 bytes, base64 or 64 hex characters                    |
| `VIBE_KANBAN_MASTER_KEY_FILE` | Read the key from this file instead of `~/.config/vibe-kanban/master.key`                               |

Note that `VIBE_KANBAN_DB` and `VIBE_KANBAN_WORKSPACE` are independent: setting the workspace does not move the database. Set both, or simply run the CLI from inside the project.

---

## Troubleshooting

**`SyntaxError` or "Cannot find module 'node:sqlite'" on startup**
Node is older than 22.5. Check `node --version` and upgrade.

**`pnpm vk -- list` prints the help text**
Drop the `--`. pnpm passes arguments through on its own; the `--` arrives as an argument and is not a command. Use `pnpm vk list`.

**`EADDRINUSE` on 8765**
Something else has the port. `pnpm vk serve --port 9000`, or stop the other process.

**The board is empty but I know I have tickets**
You are in a different workspace than you think. The CLI prints the database path on `serve`; for other commands, check that `.vibe-kanban/` or `.git` sits where you expect. `VIBE_KANBAN_DB` overrides it outright.

**"non-loopback binding requires LAN mode and a configured password"**
You asked it to bind to a public interface. Enable LAN mode and set a password under **Settings → Network & access** first, or bind to `127.0.0.1`.

**A provider probe fails with "Missing environment variable: `SOMETHING_API_KEY`"**
The provider names a credential variable, but no credential from the vault is attached to it. Open the provider, pick a credential under **Authentication**, and save. This message comes from the agent runtime, which is told the variable name but never the secret.

**A provider returns 404 on every request**
The wrong wire format. Switch **API surface** between `responses` and `chat` on the provider and probe again.

**The agent button or Agent dispatch is missing**
Both appear only when a provider exists that is enabled and not unavailable. Check **Settings → Providers** and press **Probe**.

**The Skills page shows the agent toggles greyed out**
The `.claude` or `.codex` folder does not exist in the workspace root. `mkdir -p .claude/skills` (or `.codex/skills`) and reload the page.

**The agent does not see a skill I just installed**
An agent lists its skills when its session starts. Restart the agent session. Editing an existing skill's content does not need this.

**Codex prints "failed to clean up stale arg0 temp dirs: Permission denied"**
Harmless, and from Codex itself rather than Vibe Kanban. It is a leftover temp directory owned by another user.

**Codex prints "Ignoring malformed agent role definition: agent role `subagent` must define a description"**
An agent role in your Codex configuration is missing its `description` key. Fix it in `~/.codex/config.toml`; Codex continues without that role in the meantime.

**A run is stuck in progress after the process died**
Dispatch leases expire. The run is cancelled once its lease times out, or you can cancel it yourself on the Operations page.

---

## Getting help

Open an issue on this repository. Include what you ran, what you expected, what happened, your Node version, and — if the gateway is involved — the provider protocol and API surface, never the key itself.
