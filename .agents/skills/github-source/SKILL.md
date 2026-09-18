---
name: github-source
description: Fetch GitHub issues as Vibe Kanban source tickets and post blocking questions back to the issue with GitHub CLI.
workflow: github-issue-source
---

# GitHub Source

Use this plugin when a source ticket is a GitHub issue: to read the issue into a Vibe Kanban work ticket, and, after the user confirms, to post blocking questions back to the issue.

## The Steps Live In The Workflow

The readiness checks, the three request paths, and the confirmation gate before posting are the `github-issue-source` workflow, which the team edits in the Vibe Kanban UI:

```bash
pnpm vk workflow github-issue-source
```

Follow that document stage by stage; it wins over any order implied here.

## Contract

- Accepted issue references: `https://github.com/<owner>/<repo>/issues/<n>`, `<owner>/<repo>#<n>`, or `#<n>` when the current repository has a GitHub remote.
- Source context lands on the Vibe Kanban work ticket through `source_type`, `source_id`, `source_url`, `source_snapshot`, and `source_evidence`. Never create a Vibe Kanban ticket just to mirror an issue.
- Evidence keeps its provenance: label each image or link with where it came from (`issue body`, `comment by <login>`).
- Every fetch and every post is recorded with `action-log --action-type plugin:github-source`, including `--status failed` when a readiness check fails so the caller can fall back.
- Requirement and design questions are the only sections that go to an issue; technical and operations questions go to the channel the user picks.

## Do Not

- Create, close, or label GitHub issues.
- Post without the user's confirmation.
- Copy credentials or private attachment tokens into Vibe Kanban fields.
