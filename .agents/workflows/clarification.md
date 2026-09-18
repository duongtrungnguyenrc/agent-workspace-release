---
name: clarification
description: Turn a blocking uncertainty into questions written for the people who can answer them, record them on the ticket, and deliver them through a channel the developer confirms.
default: false
start: reason
---

# Clarification

> Start at `reason` and follow each stage's `next`. A branch chooses the next stage from its conditions; `end` finishes the task.

Run this workflow whenever another workflow hits a decision it cannot make alone: competing intents, unclear scope or acceptance, a UX or data-contract choice, an integration or permission unknown, or an operational blocker. The caller hands over the ticket id and the situation, and does not code, create tickets, or change plans while the question is open.

Holds for every stage:

- Ask only what you cannot find out yourself. One decision per question, each ending with `?`, with the options you already know stated inline.
- Write every question in the language of the audience who must answer it. A `requirement` or `design` question carries no code evidence; a `technical` question carries the evidence that makes it answerable; an `operations` question names systems and owners but never secret values.
- The `questions` command enforces those rules. When it rejects text, fix the wording rather than changing the category to get it through.
- Never send anything to a human channel without the developer's confirmation in the current chat, and never post technical questions to a product audience.
- The ticket stays on `hold` until answers arrive. Never invent an answer to unblock yourself.

## Reason before asking

> key: `reason` · gate: `none` · next: `answerable`

Write the question only once you can state all four of these. If you cannot, investigate first (ticket tree, source snapshot, CodeGraph, memory) instead of asking.

- **Decision**: what exactly must be decided, in one sentence.
- **Why it blocks**: which step cannot proceed, and what would be wrong if you guessed.
- **Options**: the two or three concrete answers you can see and what each implies, including your recommended default.
- **Owner**: who can actually answer, which picks the category.

Do not ask what you can find out yourself, do not bundle unrelated decisions, and do not ask the user to choose between options you have not described.

## Can you answer it yourself?

> key: `answerable` · type: `branch`
>
> - **Yes: the evidence is available to you** → `end`
> - **No: it needs a human decision** → `classify`

## Classify and write for that audience

> key: `classify` · gate: `none` · next: `record`

One situation often produces two questions for two audiences; never send the technical one to the BA.

| Category      | Audience               | Writing rule                                                                                                                                                                                  |
| ------------- | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `requirement` | BA / Product Owner     | Product language only. Describe the observable situation and the outcomes to choose between. No file paths, identifiers, method calls, stack traces, SQL, endpoints, or implementation terms. |
| `design`      | Designer / UX          | Same rule; name screens and states, never components or props.                                                                                                                                |
| `technical`   | Tech Lead / Developers | Cite the evidence: path, symbol, log line, endpoint, data shape, and the options with their cost.                                                                                             |
| `operations`  | DevOps / Admin / PM    | Name systems and owners; never paste secret values.                                                                                                                                           |

Write each item as a single question ending with `?`, with the options inline when they exist.

## Record on the ticket

> key: `record` · gate: `none` · next: `channel`

One file per category, validated before writing:

```bash
pnpm -s vk questions <ticket-id> --category requirement --questions @requirement.md --dry-run --json
pnpm -s vk questions <ticket-id> --category requirement --questions @requirement.md --description "Blocked pending BA decision on <topic>" --quiet
```

The command rejects code evidence in `requirement` and `design`, rejects secrets anywhere, and warns when a `technical` question cites nothing. Fix the wording; never force it through by changing the category. The ticket moves to `hold`.

## Find a channel and ask the developer

> key: `channel` · gate: `ask_user` · next: `sent`
> Gate: stop and ask the user before continuing past this stage.

Read the installed skills and keep the ones that can post to a human channel (Teams, Slack, Jira comment, Linear, email, or `github-source` for the source issue). Check `PROJECTS.md` for channel rules. Map each category to where its audience reads, and send separate messages so each audience sees only its own questions.

Ask the developer in the current chat, one question per category, before sending anything: name the skill, the channel, and the audience, and offer the alternatives (send through that skill, answer here now, or hold without sending).

## Was it delivered?

> key: `sent` · type: `branch`
>
> - **Sent through a channel skill** → `await`
> - **Answered in chat, or no channel fits** → `store`

Record the outcome either way so the ticket shows where the question went:

```bash
pnpm -s vk action-log <ticket-id> --action-type plugin:<skill-name> --status <triggered|skipped|failed> --url <message-or-thread-url> --description "<what happened>" --quiet
```

## Wait for the answer

> key: `await` · gate: `ask_user` · next: `store`
> Gate: stop and ask the user before continuing past this stage.

Keep the ticket on `hold`. Read replies back through the channel skill when it supports that; otherwise ask the developer to forward them. Never invent an answer to unblock yourself.

## Store the answers and hand back

> key: `store` · gate: `none` · next: `end`

Store each answer directly below the matching question. Select it by its 1-based position within the category or by its exact text:

```bash
pnpm -s vk questions <ticket-id> --category <category> --question <number> --answer "<answer>" --actor "<who>" --description "Answered by <who>" --quiet
```

The command keeps the question and inserts the attributed answer inline so the decision remains readable in context. Use `--clear --category <category>` only when the whole question-and-answer section should be removed from the ticket.

Hand control back to the caller with the decisions summarized. The caller moves the ticket out of `hold` and updates the specification or plan; a changed plan invalidates approval and must be reviewed again.
