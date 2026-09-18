---
name: documenter
description: Create and update product group (user story) and feature (use case) tickets and work-ticket documentation directly in Vibe Kanban.
workflow: documentation
---

# Documenter

Use this when the user asks to create, organize, or update product documentation, user stories, use cases, scenarios, requirements, or acceptance criteria from material they provide. To infer stories from code that already exists, use `doc-collector` instead.

```bash
pnpm -s vk workflow documentation
```

Read that document and follow it stage by stage. It is edited by the team on the Vibe Kanban Workflows page, so it, not this file, is the authority on what happens.

This skill carries [references/ticket-templates.md](references/ticket-templates.md): the `group`, `feature`, and `task` specification shapes, the execution plan shape, and the use case writing standard. The `codebase-bootstrap` workflow uses the same file.
