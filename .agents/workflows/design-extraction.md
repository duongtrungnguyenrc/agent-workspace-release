---
name: design-extraction
description: "Extract or refresh a project's existing visual design language into DESIGN.md as a contract for future UI work."
default: false
implementation_scope: DESIGN.md
start: read_existing
---

# Design extraction

> Start at `read_existing` and follow each stage's `next`. A branch chooses the next stage from its conditions; `end` finishes the task.

Run this workflow when the user asks to infer, collect, extract, refresh, or document a project's visual design system in `DESIGN.md`. The output is a contract for future UI implementation describing what the project already does, not a redesign proposal. Capture a new direction only when the user explicitly asks for one.

The document shape, the token rules, and the collection guidance are in [design-md-format.md](../skills/design-collector/references/design-md-format.md). Read it before creating or substantially rewriting `DESIGN.md`.

## Read the existing contract

> key: `read_existing` · gate: `none` · next: `inspect`

Start from the nearest existing `DESIGN.md` and preserve its schema unless the user asks to replace it.

## Inspect the real visual behavior

> key: `inspect` · gate: `none` · next: `conflicting`

Inspect the sources that define how the UI actually looks: theme files, CSS variables, Tailwind config, component libraries, route and page composition, shared primitives, icons, typography imports, image assets, and screenshots when available. Use CodeGraph for structural exploration and text search for literal class names, CSS variables, asset names, and copied color values.

Prefer exact values from source over visual approximation, and label anything approximated from an image. Where there is no evidence for a token or behavior, mark it an assumption or leave it out rather than inventing it.

## Does the evidence point in several directions?

> key: `conflicting` · type: `branch`
>
> - **Yes: materially different directions, a conflict with a supplied brand reference, or a gap only new product decisions could fill** → `ask`
> - **No** → `write`

## Ask before choosing a direction

> key: `ask` · type: `workflow` · workflow: `clarification` · next: `inspect`
> Run the `clarification` workflow (`pnpm -s vk workflow clarification`) with this ticket, then come back and continue at `inspect`.

Name the directions the evidence supports and what each would mean for future UI work. Documenting existing design is in scope; choosing new product behavior is not.

## Write DESIGN.md

> key: `write` · gate: `none` · next: `end`

Apply the collection guidance in [design-md-format.md](../skills/design-collector/references/design-md-format.md), including which tokens to preserve, when to consolidate values, which components to include, and recording the source coverage that backs the document.

If the user asked only for findings, return the summary and the proposed changes without editing the file.
