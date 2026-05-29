# Roadmap

forge-config is the connector layer for GLG agents working through Forgejo.
It is not a dashboard product, and it is not an automatic coding factory.
It exists so requests discovered in human-agent conversations can become durable,
reviewable, sortable work items.

## North Star

Give each operational domain an agent that people can talk to.
When a request, bug, missing feature, or repeated pain point appears in that
conversation, it should be captured as a Forgejo issue with enough context for
another agent — and later GLG — to reason about it.

The goal is not to make every team use a dashboard. Most people do not want one,
and many can build their own views if needed. The goal is to let them talk to the
agent responsible for their domain, while the agent network preserves the work
trail in Forgejo.

## Intended Flow

```text
Human / domain owner talks to a domain bot
  → conversation is recorded as JSONL on the server
  → bot or sweeper detects a requirement / bug / improvement
  → Forgejo issue is created with labels and source context
  → forgebot wakes up from labels / webhook events
  → forgebot asks the relevant repo/domain owner agent for first review
  → owner agent classifies scope, risk, priority, and implementation need
  → forgebot writes that review back to Forgejo
  → GLG later reviews a sorted backlog and calls owner agents for focused implementation
  → implementation, tests, and final status are recorded back on the issue
```

## Non-Goals

- Build dashboards for operational teams.
- Replace domain owners with a central UI.
- Let forgebot implement every issue immediately.
- Treat GitHub Copilot or generic hosted coding agents as GLG owner agents.
- Optimize for maximum parallel coding before the triage loop is trustworthy.

## Roles

| Role | Responsibility |
|---|---|
| **Domain bot** (`vocbot`, etc.) | Talks with the human/domain owner and surfaces requests from conversation. |
| **forgebot** | Coordinates Forgejo issues: wakes on labels, asks owner agents for first review, writes results back. |
| **Owner agent** | Understands one repo/domain, reviews issues, estimates risk/scope, and later helps with focused implementation. |
| **GLG** | Reviews the sorted backlog, chooses implementation batches, and makes final commit/merge decisions. |
| **forge-config** | Maintains the connector: CLI, label protocol, footer convention, and public operating docs. |

## Current Phase — Connector First

The basic connector now exists:

- self-hosted Forgejo profiles (`oracle`, `work`)
- `bin/forge` CLI for issue/comment/label operations
- footer identity convention for the shared `glg-bot` user
- webhook path through OpenClaw for waking forgebot
- status-label replacement via `label-set`
- multi-line review/comment upload via `comment --body-file`

This phase is about making the triage loop reliable before automating
implementation.

## Near-Term Roadmap

### 1. Issue Capture

- Ensure domain bots can create Forgejo issues from conversation context.
- Preserve source metadata when the request came from a chat/thread/session.
- Keep the issue body concise but enough for owner-agent review.

### 2. First Review Loop

- forgebot should not implement by default.
- forgebot should identify the likely owner agent and request first review.
- Owner-agent review should answer:
  - Is this real work?
  - Which repo/domain owns it?
  - What is the risk?
  - What is the likely implementation scope?
  - Does GLG need to decide first?

### 3. Backlog Sorting

- Use Forgejo labels/comments as the durable sortable surface.
- Prefer `label-set` for lifecycle status so status labels do not accumulate.
- Keep signal labels (`ci:failed`, domain tags, priority tags) separate from status labels.

### 4. Focused Implementation Batches

- GLG reviews sorted issues and calls owner agents directly.
- Owner agents may use sibling agents (`entwurf`) for analysis, tests, or review.
- Implementation should be done in focused batches, not one issue at a time by an always-on bot.

### 5. Completion Trail

- Owner agents write implementation results back to the issue.
- Use `comment --body-file` for long summaries, test output, and handoff notes.
- Use `label-set agent:done` or `label-set human:needs-review` for final state.

## Label Direction

Status labels are mutually exclusive and should be changed with `label-set`:

- `agent:ready`
- `agent:running`
- `agent:done`
- `agent:blocked` — create this label in each repo before using it
- `human:needs-review`

Signal labels such as `ci:failed`, domain names, or priorities can coexist with
one status label.

## Documentation Habit

This system evolves while being used. Periodically rewrite the README, ROADMAP,
AGENTS, and skill docs so a newly awakened agent can answer:

1. What are we building?
2. Why does Forgejo exist here?
3. What should forgebot do, and what should it not do?
4. When does GLG enter the loop?
5. Which commands are safe and canonical today?
