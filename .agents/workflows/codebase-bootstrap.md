---
name: codebase-bootstrap
description: Scan an existing codebase with CodeGraph and create the Vibe Kanban group and feature tickets that describe the product it already implements.
default: false
start: confirm_index
---

# Codebase bootstrap

> Start at `confirm_index` and follow each stage's `next`. A branch chooses the next stage from its conditions; `end` finishes the task.

Run this workflow only when the user explicitly asks to auto-generate, infer, scan, or bootstrap user stories or use cases from the current project. Never run it implicitly during implementation, bug fixing, or story-writing from material the user provides; that material goes to the `documentation` workflow.

Holds for every stage:

- The output is a product map inferred from code: `group -> feature[]` and never `task` tickets. This is documentation discovery, not implementation planning.
- Evidence and assumptions for each cluster live in the ticket `specification`, and the origin is marked with source fields such as `source_type=codegraph`.
- See [the vibe-kanban skill](../skills/vibe-kanban/SKILL.md) for the ticket model and [ticket-templates.md](../skills/documenter/references/ticket-templates.md) for specification shapes and the use case writing standard.

## Confirm the index

> key: `confirm_index` · gate: `none` · next: `scan`

Confirm that setup has run and the repository has a `.codegraph/` index, or that another code-exploration plugin the project prefers is available. If neither exists, stop and tell the user to run `./scripts/setup.sh` or `codegraph init`. Do not infer a whole feature map from ad hoc file search.

## Scan the product surface

> key: `scan` · gate: `none` · next: `cluster`

Explore with CodeGraph first, focusing on the entry points that reveal product features: app routes, screens, pages, controllers, handlers, and navigation; sidebar, menu, tab, route registry, layout shell, and permission-gated modules; API route groups, service boundaries, GraphQL resolvers, background jobs, and mobile screens; and existing tests or fixtures that describe workflows.

Use text search only for literal labels, menu text, route strings, or after CodeGraph has identified a file.

## Cluster into features

> key: `cluster` · gate: `none` · next: `ambiguous`

Group the findings into coherent product capabilities, each with the code evidence behind it. Before creating anything, list existing tickets with `list --json` and prefer updating an inferred `group` or `feature` whose title or source evidence clearly matches.

## Is the product map ambiguous?

> key: `ambiguous` · type: `branch`
>
> - **Yes: several reasonable maps, unclear ownership, overlapping clusters, or a non-user-facing area** → `ask`
> - **No** → `create`

Uncertainty only about labels or minor grouping is not a blocker: record the assumption in the ticket `specification` and continue.

## Ask before deciding the map

> key: `ask` · type: `workflow` · workflow: `clarification` · next: `cluster`
> Run the `clarification` workflow (`pnpm vk workflow clarification`) with this ticket, then come back and continue at `cluster`.

Ask one concise question naming the discovered options. When the question belongs to an existing ticket and needs a product owner or designer rather than the developer in the chat, the clarification flow records and routes it; clustering resumes with the answer.

## Create the ticket tree

> key: `create` · gate: `none` · next: `handback`

Create a `group` per capability and `feature` children for distinct actor goals, scenarios, or acceptance areas. Record the evidence and the assumptions in `specification`, and mark the origin with source fields:

```bash
pnpm vk create --type group --title "<feature>" --specification "<markdown>" --source-type codegraph --source-snapshot "<evidence>"
pnpm vk create --type feature --parent-id <group-id> --title "<Verb Noun>" --specification "<markdown>" --source-type codegraph --source-snapshot "<evidence>"
```

Do not create `task` tickets here. If an inferred area is not story-shaped, ask before representing it as a top-level task.

## Hand back for review

> key: `handback` · gate: `none` · next: `end`

Return the created or updated hierarchy with ticket ids and the main code evidence per cluster, then stop for user review. Implementation planning is a separate request that rescans the code.
