---
name: clarifier
description: Turn a blocking uncertainty into well-formed questions, classify each by who must answer it (BA, designer, tech lead, ops), record them on the Vibe Kanban ticket, then find channel skills (Teams, Slack, Jira, GitHub) and ask the developer before sending.
workflow: clarification
---

# Clarifier

Use this whenever another flow hits a decision it cannot make alone: competing intents, unclear scope or acceptance, a UX or data-contract choice, an integration or permission unknown, or an operational blocker. The caller hands over the ticket id and the situation, and does not code or change plans while the question is open.

```bash
pnpm -s vk workflow clarification
```

Read that document and follow it stage by stage. It is edited by the team on the Vibe Kanban Workflows page, so it, not this file, is the authority on what happens.
