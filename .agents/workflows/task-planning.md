---
name: task-planning
description: Turn a Vibe Kanban group, feature, or external source ticket into approval-ready task tickets, then hand the approved task to implementation.
default: true
implementation_scope: source/**
start: intake
layout:
  intake: 0,0
  explore: 0,186
  plan: 0,372
  blocked: 0,558
  clarify: -150,744
  handoff: 0,930
  task_approval: 150,744
---

# Task planning

> Start at `intake` and follow each stage's `next`. A branch chooses the next stage from its conditions; `end` finishes the task.

Run this workflow when the user asks to implement a Vibe Kanban `group`, `feature`, `task`, or one or more external source tickets. It reads the request, explores the code it will touch, and produces a specification and an execution plan the user can approve. It writes no implementation code: that is the `task-implementation` workflow, which this one hands the ticket to.

Holds for every stage:

- Read `PROJECTS.md` in the repository and any applicable project directory, and follow it as mandatory project-specific instruction.
- Every stage runs against Vibe Kanban: read with `get <id> --json` and record what happened with `progress-log`, so the ticket stays the trace of the work. See [the vibe-kanban skill](../skills/vibe-kanban/SKILL.md) for the ticket model and CLI contract.
- Never create or maintain `documents/**`, `document.md`, product-doc `progress.md`, or `.agents/processes/**` files. The ticket is the record.
- Do not change implementation code here, not even a small fix found while exploring.

Stop and ask the user when:

- Analysis reveals materially different intents, task slices, parent links, product behaviors, data contracts, integrations, or risk profiles.
- Parent context is still missing after `smart-search` and the task cannot be safely scoped alone.
- A needed dependency, API contract, external service, or permission model is unclear.
- A purely local, reversible engineering choice is not a reason to ask: record it as an assumption in the specification and continue.

## Read the ticket and its source context

> key: `intake` · gate: `none` · next: `explore`

If the request names external source tickets instead of Vibe Kanban ids, read them through the fitting plugin (for example `github-source` for GitHub issues) or the tool the user supplies; otherwise ask the user for the content. Follow [the Source Tickets section of the vibe-kanban skill](../skills/vibe-kanban/SKILL.md) for what to store and how.

Read the requested ticket with `get <ticket-id> --json`. For a `group`, read its `feature` children; for a `feature`, read its parent `group`. Use their specifications and source snapshots as product context.

## Explore the codebase

> key: `explore` · gate: `none` · next: `plan`

Recall project memory first when a memory plugin is available, then explore the product area with CodeGraph. If the repository has no `.codegraph/` index and no exploration plugin fits, use targeted reads and record that limitation in the task `specification`.

Use `smart-search "<query>" --parent-for task --json` to find related tickets and the likely parent before deciding context is missing. Read `kind_detection` and `parent_suggestion` from its output.

Understand architecture, execution flow, dependencies, affected code, existing abstractions, side effects, and test coverage before writing any plan. A plan written without this is guesswork.

## Specification and execution plan

> key: `plan` · gate: `approval` · next: `blocked`
> Gate: stop until the user approves the execution plan; `start` refuses an unapproved task.

Read this stage before writing anything: `get <id> --json` names the workflow and the stage the ticket sits at, and `pnpm -s vk workflow task-planning` prints this document.

Rewrite the raw requirement into a Markdown specification (Objective, Context, Requirements, Acceptance Criteria, Constraints, Out of Scope). Write it before the plan: the CLI warns on a plan written against an empty specification.

Write the execution plan as top-level Markdown checklist items (`- [ ]` at column 0); indented items are supporting detail and are not tracked. Group the plan by the child projects that actually exist inside the implementation scope. Name what changes, why, the files or modules, the order, dependencies, tests to add or change, and the real risks. `approve` refuses a plan with no top-level checklist item.

For a `group`, `feature`, or source-ticket request, create or upsert the smallest independently reviewable `task` tickets by matching existing child titles or source identifiers, each with `--kind` and `--parent-id`. For a free-form task with no parent, propose the detected kind and suggested parent in one confirmation and create a top-level task only after the user accepts.

New or changed plans stay `open` and unapproved. Stop and hand the tickets back for review.

## Blocked on a decision only a human can make?

> key: `blocked` · type: `branch`
>
> - **Yes: intent, scope, UX, data contract, integration, or risk is unresolved** → `clarify`
> - **No: the plan is approved and ready to build** → `task_approval`

Decide with the stop list at the top of this workflow.

## Blocking questions

> key: `clarify` · type: `workflow` · workflow: `clarification` · next: `plan`
> Run the `clarification` workflow (`pnpm -s vk workflow clarification`) with this ticket, then come back and continue at `plan`.

Hand over this ticket id and the situation. Do not plan around an open question. When the answers arrive, update the specification or plan; a changed plan invalidates approval, which is why control returns to `plan`.

## Pending user review and approval

> key: `task_approval` · type: `branch`
>
> - **Yes** → `handoff`
> - **No** → `plan`

## Hand the approved task to implementation

> key: `handoff` · type: `workflow` · workflow: `task-implementation` · next: `end`
> Hand the ticket to the `task-implementation` workflow (`pnpm -s vk workflow task-implementation`) and follow it from its start stage. This workflow ends here.

Only an approved task crosses this line: `get <id> --json` must show `user_reviewed = true` with a non-empty execution plan. A `group` or `feature` request stops before here, because its generated tasks each need their own approval first.
