---
name: task-implementation
description: "Execute an approved Vibe Kanban task: branch, implement the plan, verify, pass the local review, commit, and open the pull request."
default: false
base_branch: develop
implementation_scope: source/**
start: checkout
---

# Task implementation

> Start at `checkout` and follow each stage's `next`. A branch chooses the next stage from its conditions; `end` finishes the task.

Run this workflow on a task that `task-planning` has already taken through approval. It writes the code, so it is the half with the local review gate.

Holds for every stage:

- Keep code changes inside this workflow's implementation scope. Go outside it only when the approved task explicitly requires a workspace-level supporting change.
- Implement the approved plan and nothing else. Work that grows beyond it goes back to `task-planning` as a new or revised task.
- Keep unrelated user changes intact and commit only the files that belong to this task.
- Record progress against the plan's checklist steps as you go, so the board and the floating monitor stay truthful.

Stop and ask the user when the approved plan and the current code conflict in a way that changes product behavior or migration risk, when the base branch is unavailable or wrong for this repository, or when GitHub CLI is unavailable, unauthenticated, or has no remote once a PR is requested.

## Create the working branch

> key: `checkout` · gate: `none` · next: `implement`

Follow [git-workflow.md](../skills/task-implementer/references/git-workflow.md) for naming and Git safety. Inspect the worktree, then branch from the base branch:

```bash
git status --short --branch
git fetch
git checkout <base-branch>
git pull --ff-only
git rev-parse HEAD
git checkout -b <prefix>/vk-<task-ticket-id>-<short-title>
```

Take the prefix from the task `kind` (`feature -> feat/`, `bugfix -> fix/`, `refactor -> refactor/`, `chore -> chore/`, `docs -> docs/`, `test -> test/`); set the kind first when it is missing. With a source ticket, use `<prefix>/<source-ticket-id>-vk-<task-ticket-id>-<short-title>`.

Record the trace:

```bash
pnpm -s vk update <task-ticket-id> --branch "<branch>" --base-commit <base-head> --quiet
pnpm -s vk progress-log <task-ticket-id> --step "Checkout branch" --description "Created implementation branch" --percent 5 --quiet
```

## Implement the approved plan

> key: `implement` · gate: `none` · next: `blocked`

Re-check the scopes the plan names with CodeGraph before editing them. If exploration proves the plan is materially wrong, update it through Vibe Kanban, which invalidates approval, and let the task go back to `task-planning`.

For UI work, read the nearest `DESIGN.md` first; if the project has none and the change needs one, the `design-extraction` workflow produces it. Then read [react-ui-implementation.md](../skills/task-implementer/references/react-ui-implementation.md), and [react-reusable-components.md](../skills/task-implementer/references/react-reusable-components.md) when the task creates, extracts, or materially refactors reusable components.

Move the ticket and implement only the approved plan:

```bash
pnpm -s vk start <task-ticket-id> --workflow task-implementation --description "Approved task implementation started" --quiet
```

Treat every top-level checklist item as a monitorable step. Set it `in_progress` before executing it and `completed` after verifying it:

```bash
pnpm -s vk progress-log <task-ticket-id> --step <number> --step-status in_progress --description "<what is starting>" --quiet
pnpm -s vk progress-log <task-ticket-id> --step <number> --step-status completed --description "<what was completed and verified>" --quiet
```

Omit `--percent` on step updates; the tool derives progress from completed steps. If blocked on something the code cannot answer, `hold` the ticket with the reason.

## Blocked on a decision only a human can make?

> key: `blocked` · type: `branch`
>
> - **Yes: the answer changes behavior, data, UX, or risk** → `clarify`
> - **No** → `verify`

## Blocking questions

> key: `clarify` · type: `workflow` · workflow: `clarification` · next: `implement`
> Run the `clarification` workflow (`pnpm -s vk workflow clarification`) with this ticket, then come back and continue at `implement`.

Hand over this ticket id and the situation, and stop coding until it is answered. If the answer changes the execution plan, update it: that invalidates approval, and the task belongs back in `task-planning` before coding continues.

## Focused verification

> key: `verify` · gate: `none` · next: `local_review`

Run the project's checks for the changed scope: type check, lint, focused tests, and a build when the change can break one. Record the exact commands and their results on the verification step.

## Local user review

> key: `local_review` · gate: `local_review` · next: `reviewed`
> Gate: request the local review and stop until the user confirms; `add-commit` and `pr` refuse the task otherwise.

Review the diff yourself first, or with a review plugin the project prefers; [git-workflow.md](../skills/task-implementer/references/git-workflow.md) holds the review stance and what this gate means. Then request the review and stop:

```bash
pnpm -s vk local-review <task-ticket-id> --status requested --description @review.md --quiet
```

The description lists the changed files, the verification commands and results, and the local URL or command the user can use to try the change. Do not run `git commit`, `git push`, or `gh pr create` until `local_review` is `confirmed`.

## What did the local review say?

> key: `reviewed` · type: `branch`
>
> - **Confirmed** → `commit`
> - **Changes requested** → `implement`

## Commit

> key: `commit` · gate: `none` · next: `pr`

Stage only the files that belong to this task. Follow [git-workflow.md](../skills/task-implementer/references/git-workflow.md) for the message and commit contents, then record every commit:

```bash
pnpm -s vk add-commit <task-ticket-id> --commit-hash <hash> --url <commit-url> --branch <branch> --message "<message>" --quiet
```

## Pull request

> key: `pr` · gate: `none` · next: `close`

Push the branch and open the PR with GitHub CLI, or the delivery plugin the project prefers; [git-workflow.md](../skills/task-implementer/references/git-workflow.md) lists what the PR body must contain. Record the delivery trace and move the task to review:

```bash
pnpm -s vk pr <task-ticket-id> --pr-url <url> --pr-status open --description "PR created" --quiet
pnpm -s vk pipeline <task-ticket-id> --pipeline-status pending --pipeline-url <url> --description "Pipeline started" --quiet
pnpm -s vk review <task-ticket-id> --description "Implementation complete and PR is ready for review" --quiet
```

Offer to announce the PR through a notification plugin when the project has one, and record what ran with `action-log --action-type plugin:<name>`.

## Close

> key: `close` · gate: `none` · next: `end`

Close the task only when the PR is merged or the user explicitly accepts the implementation:

```bash
pnpm -s vk close <task-ticket-id> --description "Merged or accepted by user" --quiet
```
