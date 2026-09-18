---
name: my-plugin
description: One sentence, starting with a verb, that says what this plugin reaches and through which system, so an agent can judge when it fits a decision.
---

# My Plugin

A plugin connects this workspace to a system it does not talk to itself, such as Teams, Slack, Jira, Linear, or an internal API. What the agents do, and in what order, lives in `.agents/workflows/`; a plugin only says what it can reach, what it needs, and what it guarantees.

At decision points (source tickets, blocking questions, notifications, code exploration, design context, review, delivery) the workflows read the installed plugins and propose the ones that fit, so keep the description above precise enough for that judgement.

Use this plugin when <situation, for example: the project tracks requirements in Teams and a ticket has open questions>.

## Contract

- Requires: <the CLI, webhook, MCP server, or credential this needs, and how to check it responds>.
- Reaches: <which audiences or systems, for example the #product Teams channel>.
- Writes back: results land on the Vibe Kanban ticket, for example `pnpm vk update <id> --source-snapshot @file` or `pnpm vk comment <id> --comment "<answer>" --actor user --quiet`.
- Records: every run is logged with `pnpm vk action-log <id> --action-type plugin:my-plugin --status <triggered|skipped|failed> --url <url> --description "<what happened>" --quiet`, including `failed` so the caller can fall back.

## Steps, If It Needs Them

A plugin with more than one path deserves a workflow of its own. Create it on the Workflows page, then name it here:

```yaml
workflow: my-plugin-flow
```

`github-source` is the reference: its readiness checks, its three request paths, and the confirmation gate before posting all live in the `github-issue-source` workflow.

## Do Not

- Store data outside Vibe Kanban.
- Post to a human channel without the user's confirmation.
- Copy credentials or private tokens into ticket fields.
