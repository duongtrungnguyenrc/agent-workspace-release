# Git Workflow

Git and GitHub mechanics for implementation work: how branches are named, what a commit and a PR must contain, and what the local review gate means.

This file does not define the order of the work. The stages, the gates, and where the work stops for review come from the workflow the task runs (`pnpm vk workflow`). When a stage's instructions and this file disagree about sequence, the workflow wins; this file still governs naming, contents, and Git safety.

## Core Rules

- Do not modify implementation code on the workflow's base branch, `main`, or any other protected branch.
- Do not overwrite, reset, or revert user changes unless the user explicitly asks for that destructive operation.
- Inspect the worktree with `git status --short --branch` before changing branches.
- Keep unrelated local changes intact. Ask the user only when those changes block checkout, verification, or committing the task.
- Take the base branch from the workflow's `base_branch`. If that branch is missing, or the repository clearly integrates somewhere else, stop and ask before choosing another base.
- Commit only files that belong to the task. Avoid a broad `git add .` unless the task intentionally changed everything visible.
- Use GitHub CLI (`gh`) for PR creation when it is authenticated and a GitHub remote is configured.

## Branch Naming

Husky/conventional-compatible prefixes, taken from the Vibe Kanban task `kind`:

| kind       | prefix      |
| ---------- | ----------- |
| `feature`  | `feat/`     |
| `bugfix`   | `fix/`      |
| `refactor` | `refactor/` |
| `chore`    | `chore/`    |
| `docs`     | `docs/`     |
| `test`     | `test/`     |

Set the kind on the ticket first when it is missing (`update <id> --kind <kind>`).

- Without a source ticket: `<prefix>/vk-<task-ticket-id>-<short-title>`
- With a source ticket: `<prefix>/<source-ticket-id>-vk-<task-ticket-id>-<short-title>`, for example `fix/JIRA-123-vk-42-login-timeout`. With several source tickets, use the primary id in the branch and keep the full set in the PR body and the Vibe Kanban source snapshot.

If the branch already exists, inspect it rather than recreating it. Confirm it is based on the expected base, or ask whether rebasing, recreating, or continuing from it is safer.

## Local Review Gate

The local review happens before the first commit, and Vibe Kanban enforces it: `add-commit` and `pr` refuse a task whose `local_review` is not `confirmed` or `skipped`.

- The review request description must list the changed files, the verification commands and their results, and the local URL or command the user can use to try the change.
- `requested` means the agent has stopped. Do not run `git commit`, `git push`, or `gh pr create` while a task sits there.
- The user answers in the UI or with `local-review <id> --status confirmed --actor user`. On `changes_requested`, address the notes, verify again, and request the review again.
- A specification or execution-plan change re-arms the gate: `local_review` returns to `pending` so the work that follows gets its own review, even when the task already has commits. Only a `closed` or `cancelled` ticket is left alone.
- Skipping is not a workflow option. `--skip-local-review "<reason>"` exists only for the case where the user explicitly said in the current conversation that this task may be committed without a local pass; the reason is recorded on the ticket.
- A rejected `add-commit` means the gate was bypassed: stop and request the review instead of pushing.

## Commit Contents

```bash
git status --short
git diff -- <relevant-paths>
git add <relevant-paths>
git commit -m "<type>: <summary>"
git rev-parse HEAD
```

- One concise conventional message that references the ticket when there is one.
- Record every commit on the task ticket with `add-commit --commit-hash <hash> --url <commit-url> --branch <branch> --message "<message>"`. Pass the canonical web URL when a remote exists so ticket detail links straight to it; the UI can derive GitHub and GitLab links from the PR URL as a fallback.
- Never create an empty commit after verification finds nothing to change, unless the user asked for one.

## Pull Request Contents

Verify readiness, then push and open the PR against the workflow's base branch:

```bash
gh auth status
git remote -v
git push -u origin <branch>
gh pr create --base <base-branch> --head <branch> --title "<title>" --body "<body>"
```

The PR body includes:

- The Vibe Kanban ticket code and title, when there is one.
- The source ticket system, id, and URL when the work came from Jira, another Kanban board, Linear, GitHub Issues, or a similar system.
- A summary of the changes.
- The verification commands and their results.
- Known risks, migrations, rollout notes, and follow-up work.

If `gh` is missing, unauthenticated, or there is no GitHub remote, stop after the commit and give the user the exact branch, commit, and command to run next.

## PR Review

When asked to review a PR, or the agent's own changes before a PR:

- Take a code-review stance: findings first, ordered by severity, with file and line references.
- Inspect only the relevant diff unless the review needs broader context.
- Check correctness, behavior regressions, security and privacy risk, data migrations, missing tests, and user-facing UX issues.
- If nothing is wrong, say so plainly and name the verification gaps that remain.

Do not close a Vibe Kanban task because a PR exists. Close it only after merge or explicit user acceptance.
