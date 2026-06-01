# forge-config

Shared Forgejo connector for GLG agents.

forge-config is the public operational policy and CLI layer that lets GLG agents
turn conversations into durable, reviewable Forgejo issues. It is the code-surface
sibling of **botment**: botment lets agents leave traces on garden comments;
forge-config lets agents leave traces on code work items.

## Why Forge Exists

GLG is now giving agents to domain owners.

A domain owner should not need a dashboard just to ask for help. Operational
teams, support teams, and internal users can talk to the responsible bot directly
(for example `vocbot`). Those conversations are recorded on the server as JSONL.
When a requirement, bug, missing feature, or repeated pain point appears, it
should be captured as a Forgejo issue.

Forgejo becomes the shared work ledger:

- requests are captured from human-agent conversations;
- issues preserve enough context for review;
- labels wake the coordination bot;
- owner agents classify and review the work;
- GLG later sorts the backlog and runs focused implementation batches.

This is why GitHub Copilot or generic hosted coding agents are not the target.
The point is not “AI writes code somewhere.” The point is a GLG-owned agent
network with durable context, explicit ownership, and reviewable traces.

## What this is NOT

- ❌ A dashboard product for operational teams
- ❌ A factory-style automatic coding system
- ❌ A replacement for GLG deciding what to implement
- ❌ An infrastructure repo — Docker/Caddy/host config lives elsewhere
- ❌ Just a CLI binary — `bin/forge` is the hand, not the whole body

## What this IS

- ✅ A connector between conversations, Forgejo issues, and owner agents
- ✅ The SSOT for label policy, footer identity, and bot behavior on Forgejo
- ✅ The CLI surface used by agents to read/write the shared issue ledger
- ✅ A public record of the evolving operating model for GLG agent ownership

## Operating Loop

```text
Human / domain owner talks to a domain bot
  → conversation is recorded as JSONL
  → request / bug / improvement is detected
  → Forgejo issue is created with labels and source context
  → forgebot wakes up
  → forgebot asks the relevant owner agent for read-only first review
  → owner agent returns reality check, owner repo, risk, scope,
    implementation-needed?, priority, and a Forgejo comment summary
  → forgebot writes the review back to Forgejo
  → forgebot marks the triage cycle complete
  → GLG later reviews the sorted backlog
  → GLG calls owner agents for focused implementation and tests
  → implementation completion is recorded in a later GLG-driven batch step
```

The early automation target is **capture + first review + sorting**, not automatic
implementation. In the forgebot loop, `agent:done` means **first-review triage
completed**, not **implementation completed**. Implementation happens later in
focused GLG-driven batches so quality stays high.

When a bounded/minor issue is safe enough for an `auto-fix` lane, `auto-fix`
still must not mean “silently solved.” It means **patch candidate + validation
loop**: 1회독 direct fix/test, 2회독 adjacent same-shape sweep, 3회독 independent
review, then follow-up issues for remaining similar problems. `bin/forge
auto-fix-template ISSUE` emits the canonical report skeleton with a live issue
snapshot marker.

## `bin/forge`

`bin/forge` is a small POSIX shell + curl + jq CLI used by agents.
It supports two profiles today: `oracle` and `work`.

Core operations:

```bash
bin/forge repos [OWNER]
bin/forge list-open [REPO]
bin/forge state ISSUE
bin/forge comment ISSUE "short body"
bin/forge comment ISSUE --body-file /tmp/review.md
bin/forge label-add ISSUE LABEL
bin/forge label-remove ISSUE LABEL
bin/forge label-set ISSUE STATUS_LABEL
bin/forge issue-create [REPO] TITLE BODY [OPTIONS]
bin/forge issue-create [REPO] TITLE --body-file PATH [OPTIONS]
bin/forge auto-fix-template ISSUE
```

Use `comment --body-file` for long reviews, test output, or handoff notes.
Use `label-set` for lifecycle status so status labels do not accumulate. For replay safety, forgebot should triage only when current lifecycle labels are exactly `{agent:ready}`; webhook payload is only a wake signal.

## Label Protocol v2

Status labels are mutually exclusive and should be changed with `label-set`:

| Label | Meaning |
|---|---|
| `agent:ready` | Ready for agent review |
| `agent:running` | Picked up / under review |
| `agent:done` | First-review triage completed in the forgebot loop; not implementation completed |
| `agent:blocked` | Blocked — create this label in each repo before using it; current work repos (`sandbox`, `voscli`, `incidentcli`) are prepared |
| `human:needs-review` | GLG / human decision required |

Signal labels can coexist with one status label:

| Label | Meaning |
|---|---|
| `ci:failed` | CI is broken |

Future automation labels such as `auto-fix` should be introduced as **lane/signal
labels**, not as lifecycle completion states. `auto-fix` means a bounded patch
candidate may be prepared and verified, followed by adjacent-pattern sweep and a
recorded review trail. More domain/priority labels should be added only after
operational need is clear.

## Agent Identity

Forgejo uses one bot user, `glg-bot`. Each comment ends with a footer:

```text
— glg-bot [<model> / <host>]
```

Examples:

- `— glg-bot [gpt-5.4 / work-host]`
- `— glg-bot [claude-opus-4-7 / thinkpad]`
- `— glg-bot [pi-codex / nuc]`

This keeps token and permission management small while preserving which agent and
machine left the trace.

## Boundary of Responsibility

| Layer | Location | Responsibility |
|---|---|---|
| Infrastructure | `nixos-config/docker/forge/` | Forgejo, Caddy, host deployment |
| Connector / policy | this repo | CLI, label protocol, footer convention, bot behavior |
| Agent skill surface | `agent-config/skills/forge/` | Thin pointer consumed by agent harnesses |
| OpenClaw | `openclaw` | Chat/session/webhook transport, auth/model profiles, backend/gateway stability, forgebot runtime wiring |
| forgebot | OpenClaw workspace | Dispatcher / recorder: state → `label-set agent:running` → scenario decision → read-only owner review or bounded `auto-fix` validation lane when explicitly supported → `comment --body-file` → final `label-set` |
| Owner repos | `voscli`, `nixos-config`, `openclaw`, ... | Domain-specific first review now; focused implementation later when GLG calls the owner agent |

OpenClaw does not need to remember the entire philosophy. It should provide the
transport and runtime connection. forge-config records the public operating model.

## Roadmap

See [ROADMAP.md](./ROADMAP.md).

## Related Notes

- Design: [Forge layer — shared code workspace for GLG agents](https://notes.junghanacs.com/botlog/20260527T073823)
- Parent pattern: [botment — talk to GLG's agents through comments](https://notes.junghanacs.com/botlog/20260328T112722)
- Series root: [Harness engineering: from stone axe to artificial intelligence](https://notes.junghanacs.com/botlog/20260319T152938)

## Status

✅ Connector v2 is active: multi-profile Forgejo CLI, mutating-command
observability, `comment --body-file`, `label-remove`, and status-safe `label-set`.

Next steps live in [NEXT.md](./NEXT.md). Owner instructions live in
[AGENTS.md](./AGENTS.md).
