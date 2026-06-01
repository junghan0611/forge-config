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
- Treat `auto-fix` as “solved” without a validation trail.
- Let forgebot implement every issue immediately.
- Treat GitHub Copilot or generic hosted coding agents as GLG owner agents.
- Optimize for maximum parallel coding before the triage loop is trustworthy.

## Roles

| Role | Responsibility |
|---|---|
| **Domain bot** (`vocbot`, etc.) | Talks with the human/domain owner and surfaces requests from conversation. |
| **forgebot** | Dispatcher / recorder: wakes on labels, runs the live hook sequence, asks owner agents for read-only first review when needed, writes results back. For explicitly bounded/minor lanes it may coordinate an `auto-fix` validation loop, but the product semantics live in forge-config, not OpenClaw. |
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

## Completed Milestones

### 0.1.0 — Forge connector baseline (2026-05-27/28)

- Self-hosted Forgejo profiles are operational (`oracle`, `work`).
- `bin/forge` provides the agent-facing issue/comment/label surface.
- Multi-profile routing, mutating-command observability, and footer identity are in place.
- Mattermost thread bridge metadata can be written on issue creation via `--mm-channel` / `--mm-root-id`.
- `repos [OWNER]` exists as the discovery primitive so agents do not guess Forgejo namespaces.

### 0.2.0 — forgebot triage loop v2 (2026-05-29)

- The live OpenClaw hook prompt now follows:
  `state` → `label-set agent:running` → scenario decision → `comment --body-file` → final `label-set`.
- Long Forgejo comments are written through files; no long inline shell strings.
- Status labels are kept singleton with `label-set`.
- `agent:done` in the forgebot loop means **first-review triage completed**, not implementation completed.
- forgebot is documented as dispatcher / recorder, not implementer.
- Owner-agent calls are read-only first review by default.
- Owner-agent review returns: reality check, owner repo/domain, risk, scope, implementation-needed?, priority, and Forgejo comment summary.
- Live checks passed:
  - `glg-bot/sandbox#11`: classification-only path → `human:needs-review`, GREEN.
  - `glg-bot/voscli#10`: owner-agent read-only review → `agent:done`, GREEN.
- Both checks kept exactly one lifecycle status label; no status-label accumulation.
- Work Forge `glg-bot/{sandbox,voscli,incidentcli}` now has the `agent:blocked` label prepared.

### 0.2.1 — Mattermost thread bridge E2E (2026-05-29)

- Synthetic `vocbot` root post was created in `pjt_voc_report`.
- `glg-bot/voscli#11` was created with `--mm-channel`, `--mm-root-id`, and `--labels agent:ready`.
- forgebot webhook turn succeeded:
  - recovered the SQLite Mattermost link row;
  - called owner-agent read-only review;
  - wrote the Forgejo comment through `comment --body-file`;
  - moved lifecycle status to singleton `human:needs-review`;
  - replied to the original Mattermost thread.
- Mattermost thread reply message id: `4chwg7h1mp8ebeufj7konio5uy`.
- `#forge-events` broadcast also went out as a separate root post and did not collide with the original thread reply.
- Schema correction: OpenClaw message tool uses `target: "channel:<channel_id>"` plus `replyTo: "<root_id>"`; older docs saying `replyToId` were wrong.

## Auto-fix Direction — Validation Loop, Not Completion

`auto-fix` should be designed as a lane/signal for **bounded patch candidate +
extra verification**, not as a lifecycle status that means the issue is solved.
The key product goal is to increase review passes — 1회독, 2회독, 3회독 — while
keeping the session traces as reusable assets.

Expected routine:

1. Intake / Gate — classify the issue as minor/bounded enough for `auto-fix` candidacy and keep lifecycle labels singleton.
2. 1회독 direct fix pass — prepare a patch candidate and run directly related smoke/tests.
3. 2회독 adjacent same-shape pass — sweep similar commands, guards, wording, time surfaces, schema keys, and tests.
4. 3회독 independent review pass — have a different model/session or read-only reviewer inspect diff, tests, and sweep result.
5. Publish / Gate — re-fetch live issue state, skip on snapshot drift, write a structured report, create follow-up issues if needed.
6. End with a label/comment meaning “patch candidate + validation loop recorded,” not silent implementation completion.

This lane belongs to forge-config / forge skill / forgebot design. OpenClaw owns
transport, webhook/channel wiring, auth/model profiles, backend/gateway stability,
heartbeat, and session isolation — up to “the issue wakes forgebot.”

### Recent trigger — `glg-bot/voscli#14` (2026-06-01)

`agent:ready` woke forgebot, but the `openai-codex` auth profile was empty, so
forgebot died quickly. After restoring GPT auth profile and restarting the
gateway, both `openai/gpt-5.4` and `anthropic/claude-sonnet-4-6` forgebot runs
were GREEN. Replaying `#14` produced a first-review comment and ended at
`human:needs-review`.

GLG's design read: KST surface / wording / guard-type issues like this are likely
minor enough for an `auto-fix` lane, but only if the lane includes adjacent-pattern
sweep and second-pass validation rather than fixing one visible instance.

The repo-owned surface is `bin/forge auto-fix-template ISSUE` plus this operating model. It intentionally borrows ClawSweeper's safety grammar — conservative default, review-before-mutation, durable report, marker-backed comment, snapshot drift guard, and deterministic mutation gate — without copying its Plan / Review / Apply product pipeline or GitHub-scale machinery. GLG auto-fix is 회독-centered validation.

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
- Duplicate/replay guard policy: webhook payload is only a wake signal; current Forgejo state wins. Proceed with forgebot triage only when the current lifecycle status label set is exactly `{agent:ready}`. If `agent:ready` is mixed with `agent:done`, `agent:running`, `agent:blocked`, or `human:needs-review`, do not run owner review again. Intentional re-run requires `label-set agent:ready` to make the lifecycle status ready-only.

### 4. Bounded Auto-fix Lane

- Define when `auto-fix` is allowed: minor, bounded, reversible, testable, and with clear owner repo.
- Keep `auto-fix` as a lane/signal, not a replacement for lifecycle status.
- Require `bin/forge auto-fix-template ISSUE` or an equivalent structured report body for comment output.
- Require direct smoke/test output in the comment trail.
- Require adjacent isomorphic-pattern sweep after the patch candidate.
- Require independent third-pass review and follow-up issue creation when remaining similar problems are found.
- Prefer “patch candidate + validation loop recorded” wording over “done/solved.”

### 5. Focused Implementation Batches

- GLG reviews sorted issues and calls owner agents directly.
- Owner agents may use sibling agents (`entwurf`) for analysis, tests, or review.
- Implementation should be done in focused batches, not one issue at a time by an always-on bot.

### 6. Completion Trail

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

For forgebot replay safety, **ready-only** is the only processing state:

```text
lifecycle labels == {agent:ready}
```

A webhook event that merely contains `agent:ready` is not enough. The bot must read current state first and skip triage if any other lifecycle status is present. A human who wants to re-run triage should use `label-set agent:ready`, not `label-add agent:ready`.

## Documentation Habit

This system evolves while being used. Periodically rewrite the README, ROADMAP,
AGENTS, and skill docs so a newly awakened agent can answer:

1. What are we building?
2. Why does Forgejo exist here?
3. What should forgebot do, and what should it not do?
4. When does GLG enter the loop?
5. Which commands are safe and canonical today?
