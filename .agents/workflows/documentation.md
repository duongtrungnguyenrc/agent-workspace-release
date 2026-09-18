---
name: documentation
description: Turn user-provided product notes into a Vibe Kanban group with its feature children, stopping at product documentation.
default: false
start: read_source
---

# Product documentation

> Start at `read_source` and follow each stage's `next`. A branch chooses the next stage from its conditions; `end` finishes the task.

Run this workflow when the user asks to create, organize, or update product documentation, user stories, use cases, scenarios, requirements, or acceptance criteria from material they provide. To infer stories from existing code instead, run `codebase-bootstrap`.

Holds for every stage:

- All product documentation lives in Vibe Kanban ticket `specification` fields. Never create or maintain `documents/**`, `document.md`, product-doc `progress.md`, or `.agents/processes/**` files.
- This workflow stops at `group -> feature[]`. It never creates task tickets or execution plans, and never changes implementation code. When the user asks to implement one of these tickets, that is the `implementation` workflow, which rescans the code first.
- Source fields carry external references or imported raw input, never local document paths.
- See [the vibe-kanban skill](../skills/vibe-kanban/SKILL.md) for the ticket model and [ticket-templates.md](../skills/documenter/references/ticket-templates.md) for specification shapes and the use case writing standard.

## Read the product source

> key: `read_source` · gate: `none` · next: `shape`

Read `PROJECTS.md` when it exists and follow it.

If the user references external source tickets, read them through the fitting plugin (for example `github-source`) or the tool the user supplies; if none fits, ask for the content. Follow [the Source Tickets section of the vibe-kanban skill](../skills/vibe-kanban/SKILL.md) for what to store and how.

Use `smart-search` to find groups, features, and source ids that already cover this material before creating anything new.

## Shape the story

> key: `shape` · gate: `none` · next: `ambiguous`

Normalize the raw notes into a concise product story, then identify the distinct features inside it. One `group` commonly has many `feature` children; create as many as the story needs for clear review.

Not every request is a user story. For UAT feedback, QC feedback, maintenance, or standalone source-ticket work, ask before forcing a `group -> feature` hierarchy; a feedback-themed `group` or top-level `task` tickets may represent the material better. Before recording a task-shaped item as top-level, run `smart-search "<title>" --parent-for task --json` and confirm the suggested parent and detected kind with the user.

## Does the material leave product meaning open?

> key: `ambiguous` · type: `branch`
>
> - **Yes: competing intents, conflicting actors or goals, unclear hierarchy, or acceptance that changes behavior** → `clarify`
> - **No** → `write`

A minor uncertainty that does not change product meaning is not a blocker: record it as an assumption in the ticket `specification` and continue.

## Ask the product owner

> key: `clarify` · type: `workflow` · workflow: `clarification` · next: `shape`
> Run the `clarification` workflow (`pnpm vk workflow clarification`) with this ticket, then come back and continue at `shape`.

Story work mostly produces `requirement` questions for the BA or Product Owner and `design` questions for the Designer, written in product language with no code evidence. The ticket stays on `hold` until they are answered, then shaping resumes with the answers.

## Write the tickets

> key: `write` · gate: `none` · next: `handback`

Create the `group` ticket first, then every `feature` under it:

```bash
pnpm vk create --type group --title "<story title>" --specification "<story markdown>"
pnpm vk create --type feature --parent-id <group-ticket-id> --title "<Verb Noun>" --specification "<feature markdown>"
```

Follow the use case writing standard and the specification shapes in [ticket-templates.md](../skills/documenter/references/ticket-templates.md).

When updating existing documentation, read the ticket tree with `get <id> --json` and update through Vibe Kanban, preserving parent-child links unless the user asks to reorganize.

## Hand back for review

> key: `handback` · gate: `none` · next: `end`

Return the created or updated hierarchy with ticket ids, grouped as `group -> feature[]`. Do not list task tickets unless they already existed. Tell the user that implementation planning is a separate request, and that an approved task whose plan changed must be reviewed again.
