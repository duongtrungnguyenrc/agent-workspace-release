---
name: design-collector
description: Extract or refresh a project's design language from existing UI source, styles, screenshots, or brand assets and write it in the repository's DESIGN.md format.
workflow: design-extraction
---

# Design Collector

Use this when the user asks to infer, collect, extract, refresh, or document a project's visual design system in `DESIGN.md`. The output is a contract describing what the project already does, not a redesign proposal.

```bash
pnpm -s vk workflow design-extraction
```

Read that document and follow it stage by stage. It is edited by the team on the Vibe Kanban Workflows page, so it, not this file, is the authority on what happens.

This skill carries [references/design-md-format.md](references/design-md-format.md): the `DESIGN.md` frontmatter and body schema, the token rules, and the collection guidance.
