---
name: github-issue-source
description: Read a GitHub issue into a Vibe Kanban work ticket, and post blocking questions back to that issue once the user confirms.
default: false
start: readiness
---

# GitHub issue source

> Start at `readiness` and follow each stage's `next`. A branch chooses the next stage from its conditions; `end` finishes the task.

Owned by the `github-source` skill. A core workflow calls this one at a decision point; it never creates, closes, or labels issues, and it never posts without the user's confirmation.

## Check readiness

> key: `readiness` · gate: `none` · next: `ready`

```bash
gh auth status
git remote -v
```

## Is GitHub reachable?

> key: `ready` · type: `branch`
>
> - **Yes: authenticated with a GitHub remote** → `request`
> - **No** → `report_failed`

## What is being asked?

> key: `request` · type: `branch`
>
> - **Read an issue into a work ticket** → `read_issue`
> - **Post blocking questions to the issue** → `confirm_post`
> - **Check whether the questions were answered** → `read_answers`

## Report the failure and hand back

> key: `report_failed` · gate: `none` · next: `end`

Tell the caller that GitHub CLI is unavailable or unauthenticated so it can fall back to asking the user for the content. Record it:

```bash
pnpm vk action-log <id> --action-type plugin:github-source --status failed --description "<which check failed>" --quiet
```

## Read the issue

> key: `read_issue` · gate: `none` · next: `store_source`

Accepted references: `https://github.com/<owner>/<repo>/issues/<n>`, `<owner>/<repo>#<n>`, or `#<n>` when the current repository has a GitHub remote.

```bash
gh issue view <n> --repo <owner>/<repo> --json number,title,body,state,labels,assignees,url,comments,createdAt,updatedAt
```

## Confirm before posting

> key: `confirm_post` · gate: `ask_user` · next: `post_questions`
> Gate: stop and ask the user before continuing past this stage.

Name the issue, the audience, and which question sections would be posted. Post only the `requirement` and `design` sections, so product reviewers see them where the requirement lives; `technical` and `operations` sections go to the developer or operations channel the user chooses.

## Read the answers back

> key: `read_answers` · gate: `none` · next: `end`

```bash
gh issue view <n> --repo <owner>/<repo> --json comments
```

Store each answer below the matching ticket question and let the calling workflow decide whether the questions are resolved:

```bash
pnpm vk questions <id> --category <category> --question <number> --answer "<answer>" --actor "<login>" --quiet
```

## Store it on the work ticket

> key: `store_source` · gate: `none` · next: `end`

Write the result onto the Vibe Kanban work ticket, never into a mirrored ticket:

- `--source-type github --source-id <owner>/<repo>#<n> --source-url <url>`
- `--source-snapshot @file`: title, state, labels, body, and the comment thread condensed to what changes requirements or acceptance.
- `--source-evidence @file`: every image URL found in the body or comments as `{ "type": "image", ... }` and every non-image link as `{ "type": "link", ... }`, labelled with where it came from (`issue body`, `comment by <login>`).

Record the fetch:

```bash
pnpm vk action-log <id> --action-type plugin:github-source --status triggered --url <issue-url> --description "Fetched issue <owner>/<repo>#<n>" --quiet
```

## Post the questions

> key: `post_questions` · gate: `none` · next: `end`

```bash
gh issue comment <n> --repo <owner>/<repo> --body-file <questions.md>
```

The body starts with `Questions from Vibe Kanban VK-<id>` and lists the questions verbatim. Record the comment URL and keep the ticket on `hold`:

```bash
pnpm vk action-log <id> --action-type plugin:github-source --status triggered --url <comment-url> --description "Posted requirement questions to <owner>/<repo>#<n>" --quiet
```
