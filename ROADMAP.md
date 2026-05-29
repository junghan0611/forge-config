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
  → forgebot asks the relevant repo/domain owner agent for read-only first review
  → owner agent returns reality check, owner repo, risk, scope,
    implementation-needed?, priority, and a Forgejo comment summary
  → forgebot writes that review back to Forgejo
  → forgebot marks the first-review triage cycle complete
  → GLG later reviews a sorted backlog and calls owner agents for focused implementation
  → implementation and tests are recorded in a later GLG-driven batch step
```

## Non-Goals

- Build dashboards for operational teams.
- Replace domain owners with a central UI.
- Treat `agent:done` in the forgebot loop as implementation completed.
- Let forgebot implement every issue immediately.
- Treat GitHub Copilot or generic hosted coding agents as GLG owner agents.
- Optimize for maximum parallel coding before the triage loop is trustworthy.

## Roles

| Role | Responsibility |
|---|---|
| **Domain bot** (`vocbot`, etc.) | Talks with the human/domain owner and surfaces requests from conversation. |
| **forgebot** | Dispatcher / recorder: wakes on labels, runs the live hook sequence, asks owner agents for read-only first review when needed, writes results back. |
| **Owner agent** | Understands one repo/domain, returns reality check / owner repo / risk / scope / implementation-needed? / priority / comment summary, and later helps with focused implementation. |
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
implementation. In this phase, `agent:done` means **first-review triage
completed**, not **implementation completed**.

## Near-Term Roadmap

### 1. Issue Capture

- Ensure domain bots can create Forgejo issues from conversation context.
- Preserve source metadata when the request came from a chat/thread/session.
- Keep the issue body concise but enough for owner-agent review.

### 2. First Review Loop

- forgebot should not implement by default.
- forgebot should identify the likely owner agent and request **read-only** first review.
- Live hook sequence: `state` → `label-set agent:running` → scenario decision → `comment --body-file` → final `label-set`.
- Long comments must go through a file; no long inline shell strings.
- Owner-agent review should return:
  - reality check
  - owner repo/domain
  - risk
  - scope
  - implementation needed?
  - priority
  - Forgejo comment summary

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
- Use `label-set agent:done` for first-review triage completion, or `label-set human:needs-review` when GLG judgment is required. Do not use `agent:done` to imply implementation completion in the forgebot loop.

## Label Direction

Status labels are mutually exclusive and should be changed with `label-set`:

- `agent:ready`
- `agent:running`
- `agent:done` — first-review triage completed in the forgebot loop
- `agent:blocked` — create this label in each repo before using it; work forge `glg-bot/{sandbox,voscli,incidentcli}` is already prepared
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
