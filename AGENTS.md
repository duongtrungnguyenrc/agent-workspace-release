# AGENTS.md

## Project Control Plane

Vibe Kanban is the source of truth for product documentation, implementation plans, approval, progress, questions, Git/PR/pipeline trace, and activity history. Its CLI, local server, and UI live at `vibe-kanban/`, its data at `.vibe-kanban/vibe-kanban.sqlite`. Read [the vibe-kanban skill](.agents/skills/vibe-kanban/SKILL.md) for the ticket model, the source-ticket contract, and the CLI contract.

What the agent does, in what order, comes from the **workflows** in `.agents/workflows/`. They are Markdown documents the team edits on the Vibe Kanban Workflows page, and they beat anything implied elsewhere:

```bash
pnpm vk workflows          # what exists
pnpm vk workflow <name>    # the document to follow
```

Before any project analysis or implementation, read `PROJECTS.md` when it exists in the repository or the applicable project directory, and follow it as mandatory project-specific instruction.

## Request Routing

Each skill in `.agents/skills/` is a few lines naming the workflow it runs. Use the skill to recognize the request, then read that workflow and follow it stage by stage: start at its `start` stage, follow each stage's `next`, and at a branch pick the condition that matches. A stage's gate decides where you stop.

| The user asks for                                                                                  | Skill              | Workflow                                                                |
| -------------------------------------------------------------------------------------------------- | ------------------ | ----------------------------------------------------------------------- |
| Implementing a `group`, `feature`, `task`, or an external source ticket                            | `task-implementer` | `task-planning`, which hands the approved task to `task-implementation` |
| Product documentation from notes, requirements, scenarios, or acceptance criteria they provide     | `documenter`       | `documentation`                                                         |
| Inferring stories from the existing codebase, explicitly asked for ("scan", "bootstrap", "infer")  | `doc-collector`    | `codebase-bootstrap`                                                    |
| Capturing or refreshing the project's visual design system in `DESIGN.md`                          | `design-collector` | `design-extraction`                                                     |
| An answer only a BA, designer, tech lead, or ops owner can give, while another workflow is blocked | `clarifier`        | `clarification`                                                         |
| Reading a GitHub issue onto a ticket, or posting blocking questions back to it                     | `github-source`    | `github-issue-source`                                                   |

Direct Vibe Kanban operation (CLI, API, UI) needs no workflow: use the `vibe-kanban` skill, which is the contract itself. Changing the tool itself (schema, CLI, server, UI) starts with `vibe-kanban/docs/implementation-guide.md`; the tool is workspace infrastructure, so changes there are not gated by a task ticket unless the user asks for one.

A workflow can run another one: a stage of type `workflow` either calls it and continues afterwards, or hands the ticket over and ends. Gates apply across that chain, so splitting a flow never drops one.

A skill never restates the steps. When a skill and its workflow disagree, the workflow is right.

Longer reference material lives next to the skill that owns it and is read only when a stage points at it: Git and PR mechanics and the React guidance under `task-implementer`, the ticket specification templates and use case writing standard under `documenter`, the `DESIGN.md` schema under `design-collector`.

## Mandatory Invariants

- Ticket hierarchy is `group -> feature[] -> task[]`. Story work stops at `group -> feature[]`; implementation creates or updates task tickets from the current source state.
- A Vibe Kanban task is the implementation and approval unit. Do not modify implementation code until its current checklist execution plan is explicitly approved. A plan change invalidates approval.
- Implementation stays inside the `implementation_scope` of the workflow the task runs, and branches from its `base_branch` with a conventional prefix taken from the task `kind`.
- Preserve external source context and evidence on the work ticket, including fetched images and supporting URLs. Do not mirror source tickets as separate Vibe Kanban tickets.
- Stop before changing tickets when a decision changes intent, scope, UX, data contracts, integration ownership, risk, or hierarchy. Record the questions on the ticket, move it to `hold`, and run the `clarification` workflow.
- For structural source questions, use CodeGraph first. Use text search for literals or after identifying a file. Record the fallback when CodeGraph is unavailable.
- Keep execution plans, checklist-step state, progress, decisions, verification, branch, commits, PR, pipeline, and external actions in Vibe Kanban. Do not create local process or progress Markdown files.
- Keep unrelated user changes intact and commit only files belonging to the approved work.
- Reload only on the user's UI changes. When a ticket command prints `[vibe-kanban] the user changed the workspace since VK-<id> started`, stop, run `pnpm vk reload <id>`, read what it prints, then continue from the current stage. Changes you or a `git pull` make to the same files do not interrupt a running task, so never invent a reload from your own edits.

## Plugins

`.agents/skills/` holds plugins: connectors to systems this workspace does not talk to itself, such as GitHub, Jira, Teams, Slack, or Linear. There is no registry or schema; a plugin's `SKILL.md` frontmatter `name` and `description` are the whole contract, and a plugin that has a procedure of its own names its workflow with a `workflow` key.

At a decision point (external source tickets, blocking questions for humans, notifications, code exploration, design context, review, delivery) read the installed plugins, pick the ones that fit the situation and `PROJECTS.md`, and confirm with the user in one question before triggering any that reaches a human channel or external system. Read-only fetches with an unambiguous match may run directly, saying what ran. Record what ran on the ticket with `action-log --action-type plugin:<name> --status <triggered|skipped|failed>`. When nothing fits or the user declines, fall back to the current chat and continue; never let a plugin block the ticket.

A channel plugin is any whose description says it can deliver a message to people. A good one states the system, the audiences it reaches, and what it needs (`gh`, a Teams webhook, an MCP server), reads the text to send from the caller, returns a message or thread URL, and can read replies back. `github-source` is the reference implementation, and `.agents/templates/plugin-skill/` is the starting point for a new one.
