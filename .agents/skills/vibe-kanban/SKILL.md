---
name: vibe-kanban
description: Operate the local Vibe Kanban tool for agent work tickets, approvals, checklist progress, evidence, and Git/PR trace.
metadata:
  short-description: Manage agent implementation tickets
---

# Vibe Kanban

Use this for direct ticket work: reading, creating, updating, approving, recording progress, questions, commits, PRs, and activity. It is the control plane every workflow writes to, which is why it has no workflow of its own: what to do in a given flow comes from `.agents/workflows/`, and this file says what the tool guarantees.

The workspace ticket tool: CLI in `vibe-kanban/scripts/`, a local server and React UI, data in `.vibe-kanban/vibe-kanban.sqlite`. It is the source of truth for product documentation, implementation plans, approval, progress, questions, Git/PR/pipeline trace, and activity history.

Run it from anywhere in the repository:

```bash
pnpm vk <command>
```

Related scripts: `pnpm vk:serve` (local server), `pnpm vk:dev`, `pnpm vk:build`, `pnpm vk:check`. Run `pnpm vk --help` for the full command list.

Pass no flags to pnpm itself: `--json` and `--quiet` belong to this CLI and pnpm writes its own output to stderr, so stdout stays parseable. Any directory inside the repository works, because the CLI finds the workspace by walking up. Where pnpm is unavailable or refuses a flag, `node vibe-kanban/cli/index.mjs <command>` is the same program.

## Ticket Model

```text
group ticket            # user story, product capability, or a feedback/maintenance theme
  feature ticket[]      # one use case: actor-system behavior, flows, rules, acceptance
    task ticket[]       # implementation unit, created when implementation is requested

task ticket             # may be top-level for standalone or source-ticket-driven work
```

- `group` and `feature` `specification`: product documentation.
- `task` `specification`: implementation context written after exploring the current code.
- `task` `execution_plan`: the plan that must be approved before coding.
- `task` status, progress, events, commits, PR, pipeline, and action fields: the implementation trace.
- `source_*` fields: the external ticket this work came from, never a parent-child link.

Standalone tasks are allowed when the running workflow chooses them. Story work stops at `group -> feature[]`.

## Contract

- Task execution plans use top-level Markdown checklist items (`- [ ]` at column 0); indented items are supporting detail, not steps. The tool derives ordered plan steps and keeps their state when an unchanged label survives a revision.
- `approve` rejects a task whose plan has no top-level checklist item. Rewrite prose plans as checklists before asking for approval.
- Update a step with `progress-log <id> --step <number|exact label> --step-status <pending|in_progress|completed|blocked> --description <text> --quiet`. Omit `--percent` on step updates; the tool derives `progress_percent` from completed steps. Use `--percent` only for lifecycle milestones before the first step starts, and never lower a percent the checklist already reached.
- Open questions are classified per audience: `requirement` (BA / Product Owner), `design` (Designer / UX), `technical` (Tech Lead / Developers), `operations` (DevOps / Admin / PM). `questions <id> --category <c> --questions <text|@file>` replaces that category's section, keeps the others, and moves the ticket to `hold`. Answer with `questions <id> --category <c> --question <number|exact text> --answer <text|@file> --actor <name>`; the answer is kept inline below its question. The command rejects requirement and design questions containing code evidence, rejects secrets anywhere, and warns when a technical question cites nothing. Use `--dry-run` to validate, and `--clear [--category <c>]` only to remove a section.
- Task tickets carry `local_review` (`pending`, `requested`, `changes_requested`, `confirmed`, `skipped`). `add-commit` and `pr` refuse a task that is not `confirmed` or `skipped`. A specification or execution-plan change re-arms the gate: `local_review` returns to `pending` so the work that follows gets its own review, even when the task already has commits. Only a `closed` or `cancelled` ticket is left alone.
- `source_evidence` is a JSON array of `{ "type": "image" | "link", "url": "https://...", "label": "...", "description": "..." }`. Use `--source-evidence @file.json` for anything non-trivial.
- `delete` removes a ticket with its revisions, commits, and plan steps, refuses when children exist unless `--cascade` is passed, and keeps a `ticket.deleted` audit event. Ask the user before deleting tickets the agent did not create in this conversation.
- Every mutation records an event, and the local server emits `tickets:changed` for API and CLI writes so the UI refreshes in real time.
- Pass `--description` on status, progress, PR, pipeline, and action changes so activity and realtime messages stay useful.

## Source Tickets

A Jira, external Kanban, Linear, GitHub Issue, or similar human-managed ticket is the **source requirement**. A Vibe Kanban `group`, `feature`, or `task` is the **agent work ticket**. They are not the same thing, and a source ticket never becomes a mirrored Vibe Kanban ticket of its own.

### Reading One

Read the source through the plugin skill that fits it (for example `github-source` for GitHub issues) or the tool the user supplies. When nothing fits, ask the user for the content rather than guessing.

### Storing It On The Work Ticket

```text
--source-type <system> --source-id <id> --source-url <url>
--source-snapshot @file
--source-evidence @file
```

- The snapshot holds the title, state, labels, body, and the comment thread condensed to what changes requirements or acceptance.
- Evidence is a JSON array of `{ "type": "image" | "link", "url": "https://...", "label": "...", "description": "..." }`. Capture screenshots, attached images, design references, logs, documents, and supporting URLs when they materially affect implementation or acceptance.
- Keep provenance in the label or description, for example `issue body` or `comment by <login>`.
- Fetch image or attachment content when the source tool supports it. Never invent inaccessible evidence, and never copy credentials or private attachment tokens into ticket fields.
- Do not reduce useful visual evidence to a text-only summary.

Prefer `@file` for non-trivial JSON so shell quoting cannot corrupt URLs or descriptions.

## Workflows And Reload

- `get <id>` names the workflow the ticket follows and the stage it sits at, with that stage's instructions, so reading a ticket is also reading what to do next. A task is stamped with the default workflow when it is created; `start --workflow <name>` changes it.
- Writing an execution plan on a task warns on stderr when it has no top-level checklist item or the task has no specification, because the plan stage asks for both and `approve` refuses the first.
- `workflow [<name>]` prints the document to follow; `workflows` lists them; `--skill <name>` resolves the workflow a plugin skill names in its frontmatter; `--json` returns the structure.
- The `approval` and `local_review` gates are enforced only while a reachable stage of the ticket's workflow carries them, so `start --workflow <name>` changes what a task must satisfy.
- `start` stamps the task with its workflow and the latest workspace revision. When the user edits a workflow or a skill in the UI, every ticket-scoped command prints a stderr warning; run `reload <task-id>`, read what it prints, then continue. Edits by an agent, an editor, or `git` do not raise the warning and must not trigger a reload.

## Free-Form Task Intake

Use `smart-search` before asking the user for ticket or parent ids:

```bash
pnpm vk smart-search "<query>" --parent-for task --json
```

The JSON includes `kind_detection` and a `parent_suggestion` with `confidence` and up to three candidates. For a task request with no id and no explicit parent, confirm the proposed kind and parent in one concise question. A `high` confidence suggestion is the default answer; for `medium`, `low`, or none, list the candidates and offer a top-level task. Create with `--kind <kind> --parent-id <id>`, and omit `--parent-id` only after the user accepts a standalone task.

## Common Commands

```bash
pnpm vk get <id> --json
pnpm vk workflow
pnpm vk reload <id>
pnpm vk activity --limit 50 --json
pnpm vk progress-log <id> --step 1 --step-status in_progress --description "Started step" --quiet
pnpm vk questions <id> --category requirement --questions @req.md --description "Blocked pending BA answer" --quiet
pnpm vk questions <id> --category requirement --question 1 --answer "Use the existing approval policy" --actor "Product Owner" --quiet
pnpm vk local-review <id> --status requested --description @review.md --quiet
pnpm vk add-commit <id> --commit-hash <hash> --url <commit-url> --branch <branch> --message <message> --quiet
pnpm vk pr <id> --pr-url <url> --pr-status open --description "PR created" --quiet
```

## Extending The Tool

Read [vibe-kanban/docs/implementation-guide.md](../../../vibe-kanban/docs/implementation-guide.md) before changing schema, lifecycle, CLI/API behavior, workflow documents, realtime events, or UI. Keep it local-first: SQLite, Node.js, React, Vite, Tailwind v4 with shadcn/ui and theme tokens, Socket.IO. Do not add external databases, authentication, queues, or source-system fetching. The UI lives in `vibe-kanban/ui/` and builds to `vibe-kanban/assets/dist/`; use the existing `components/ui/*` primitives and `tone()` helpers before adding styling.

```bash
pnpm vk:check
pnpm vk:build
```
