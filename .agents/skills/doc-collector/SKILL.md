---
name: doc-collector
description: Explicitly scan a project with CodeGraph to infer feature clusters and create Vibe Kanban group/feature tickets on request.
workflow: codebase-bootstrap
---

# Document Collector

Use this only when the user explicitly asks to auto-generate, infer, scan, or bootstrap stories from the current project. Never run it implicitly during implementation, bug fixing, or story-writing from material the user provides.

```bash
pnpm -s vk workflow codebase-bootstrap
```

Read that document and follow it stage by stage. It is edited by the team on the Vibe Kanban Workflows page, so it, not this file, is the authority on what happens.
