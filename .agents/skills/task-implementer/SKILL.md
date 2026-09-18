---
name: task-implementer
description: Turn Vibe Kanban or external source tickets into approval-ready task tickets, then implement approved Vibe Kanban tasks.
workflow: task-planning
---

# Task Implementer

Use this when the user asks to implement a Vibe Kanban `group`, `feature`, `task`, or one or more external source tickets.

```bash
pnpm -s vk workflow task-planning
```

The work is two documents. `task-planning` reads the request, explores the code, and produces the specification and execution plan the user approves; it ends by handing the approved task to `task-implementation`, which branches, writes the code, verifies, passes the local review, commits, and opens the PR. Start at planning even when the user says "implement": the plan comes first, and the handoff stage carries the ticket across.

Read the document before writing a specification or an execution plan, and follow it stage by stage: start at its `start` stage, follow each stage's `next`, and at a branch pick the condition that matches. Gates decide where you stop, and they apply across the pair, so a task on `task-planning` still cannot be committed before the local review that lives in `task-implementation`. `base_branch` and `implementation_scope` decide where you work, and stages link to the references they need. The documents are edited by the team on the Vibe Kanban Workflows page, so they, not this file, are the authority on what happens.

This skill carries the lookup material the workflow points at: [references/git-workflow.md](references/git-workflow.md) for branch naming, commit and PR contents, the local review gate and the review stance; [references/react-ui-implementation.md](references/react-ui-implementation.md) for approved React UI work; and [references/react-reusable-components.md](references/react-reusable-components.md) when a task creates, extracts, or materially refactors reusable components. Read one only when a stage asks for it.
