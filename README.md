# forge-config

Shared connector for GLG agent workshops.

The workshop does not have to live only at home. The same identity can stand
up a workshop **inside the company** (GitHub Copilot) and **at home**
(self-hosted Forgejo). Trust is **identity plus cooperation**, not which host
runs the model.

forge-config is the public operating policy and CLI layer for that identity.
It is the code-surface sibling of **botment**: botment leaves traces on garden
comments; forge-config leaves traces on code work items, wherever the ledger
lives.

## Why the workshop exists

GLG is giving agents to domain owners. A domain owner should not need a
dashboard just to ask for help. They talk to the responsible bot (for example
`vocbot`). When a requirement, bug, missing feature, or repeated pain point
appears, it should become a durable, reviewable work item.

The original question was: *can we trust GitHub Copilot agents that are not
힣's agents?* Forgejo was the probe — own host, own bot user, own labels, own
footer. The answer now is: if identity is thick enough and agents cooperate,
the host is secondary.

Generic hosted agents **without** that identity remain out of scope. The point
is not “AI writes code somewhere.” The point is a GLG-owned agent network with
durable context, explicit ownership, and reviewable traces — on Forgejo **or**
on GitHub.

## Two workshops, one identity

| Workshop | Host | Ledger | Agent surface |
|---|---|---|---|
| Home | Forgejo `oracle` | Forgejo issues | `bin/forge`, forgebot, owner agents, footer |
| Company | GitHub | GitHub issues / PRs | Copilot custom agents in `.github/agents/*.agent.md` |
| Company mirror (optional) | Forgejo `work` | work Forgejo | same Forgejo protocol as home |

`AGENTS.md` is the shared identity. `.github/agents/*.agent.md` is the
selectable Copilot persona. `bin/forge` is the Forgejo hand. Do not collapse
these three into one file.

## What this is NOT

- ❌ A dashboard product for operational teams
- ❌ A factory-style automatic coding system
- ❌ A replacement for GLG deciding what to implement
- ❌ An infrastructure repo — Docker/Caddy/host config lives elsewhere
- ❌ Just a CLI binary — `bin/forge` is the Forgejo hand, not the whole body
- ❌ “Copilot replaces 힣 agents” — Copilot is a host; identity still lives here

## What this IS

- ✅ A connector between conversations, work ledgers, and owner agents
- ✅ The SSOT for identity: ownership, traces, label protocol, footer, review loops
- ✅ The Forgejo CLI surface (`bin/forge`) and the GitHub Copilot guidance surface
- ✅ A public record of the operating model so a workshop can be stood up at home or at work

## GitHub Copilot — company workshop

GitHub Copilot Max credits (chat, CLI, cloud agent) are one pool. Use them on
a custom agent that carries this repo's identity, not on the generic default.

How to set a custom agent:

1. Add `.github/agents/<name>.agent.md` on the GitHub repo (this repo's
   profile is [`.github/agents/forge-config.agent.md`](./.github/agents/forge-config.agent.md)).
2. YAML frontmatter needs at least `description`. Optional:
   `name`, `tools`, `model`, `disable-model-invocation: true` (manual pick only).
3. Markdown below the frontmatter is the prompt (max 30k characters). Point it
   at `AGENTS.md`; do not fork a second philosophy.
4. **Merge to the default branch.** Until then it will not appear in the
   Assign dropdown or [github.com/copilot/agents](https://github.com/copilot/agents).
5. Use it from: issue Assign → Copilot → custom agent; the Agents tab;
   Copilot CLI `/agent`.

`AGENTS.md` is instructions every agent should read. `.agent.md` is the
persona you pick instead of the default cloud agent. Cloud runtime extras
(`copilot-setup-steps.yml`, repo MCP) are optional and per-repo.

Docs: [custom agents](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/create-custom-agents),
[configuration reference](https://docs.github.com/en/copilot/reference/custom-agents-configuration).

## Operating Loop (home / Forgejo)

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
review, then follow-up issues for remaining similar problems. Same-shape sweep
uses exact existing paths; a missing optional sweep path or zero-match `rg`
result is a report finding (`path missing` / `no matches`), not a hook failure.
`bin/forge auto-fix-template ISSUE` emits the canonical report
skeleton with a live issue snapshot marker (`schema`, `report_id`, `session_key`,
`issue_updated_at`, lifecycle labels, sorted labels, provider/model, and
forge-config commit).

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
bin/forge close ISSUE
bin/forge reopen ISSUE
bin/forge issue-create [REPO] TITLE BODY [OPTIONS]
bin/forge issue-create [REPO] TITLE --body-file PATH [OPTIONS]
bin/forge auto-fix-template ISSUE
bin/forge doctor-labels [REPO]
```

Use `comment --body-file` for long reviews, test output, or handoff notes.
Use `label-set` for lifecycle status so status labels do not accumulate. For replay safety, forgebot should triage only when current lifecycle labels are exactly `{agent:ready}`; webhook payload is only a wake signal. Use `doctor-labels` before putting a repo on the auto-fix lane; it fails if required lifecycle/signal labels are missing.

Use `close` / `reopen` for Forgejo's open/closed state. This axis is **orthogonal to the lifecycle labels**: `agent:done` means first-review triage completed, while `closed` means the issue is resolved or withdrawn and tracking stops. Record the reason with `comment` first, then `close` — the verbs themselves stay thin (a single PATCH of issue state).

The verb is **unguarded** (same surface as `comment` / `label-set`); who may close is a **convention, not a code gate** — the same grain as the auto-fix lane, where validation lives in the loop, not a guard. Forge is an internal work surface (`glg-bot/*`, not operator-visible), so close carries little external-exposure cost, unlike a push to an operator-visible `main`. The convention splits by *why* you close:

- **Resolution close** — fixed in a shipped tag/commit and confirmed non-reproducing: the **owner agent may close autonomously** (reason comment first, then `close`).
- **Judgment close** — won't-fix, duplicate, invalid-by-design, deprioritized/withdrawn: this is a value call, so route through GLG or `human:needs-review`, not a solo owner decision.

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
labels**, not as lifecycle completion states. `agent:ready` remains the wake
label; `auto-fix` is only a route hint. `label-set` preserves non-lifecycle
signal labels while replacing the singleton lifecycle status. `auto-fix` means a
bounded patch candidate may be prepared and verified, followed by
adjacent-pattern sweep and a recorded review trail. More domain/priority labels
should be added only after operational need is clear.

## Agent Identity

Identity is the same on both hosts. On Forgejo, one bot user `glg-bot` writes
comments with a footer:

```text
— glg-bot [<model> / <host>]
```

Examples:

- `— glg-bot [gpt-5.4 / work-host]`
- `— glg-bot [claude-opus-4-7 / thinkpad]`
- `— glg-bot [pi-codex / nuc]`
- `— glg-bot [copilot-cli / thinkpad]`

This keeps token and permission management small while preserving which agent
and machine left the trace.

On GitHub, Copilot uses the GitHub identity that opened the session or PR.
The custom agent file plus `AGENTS.md` are what make that session a 힣
workshop, not a generic coding agent. Prefer the same footer in GitHub
comments when a 힣 agent is writing through Copilot CLI.

## Boundary of Responsibility

| Layer | Location | Responsibility |
|---|---|---|
| Infrastructure | `nixos-config/docker/forge/` | Forgejo, Caddy, host deployment |
| Connector / policy | this repo | CLI, label protocol, footer, GitHub Copilot guidance, bot behavior |
| GitHub custom agent | `.github/agents/*.agent.md` | Selectable Copilot persona; must be on default branch |
| Agent skill surface | `agent-config/skills/forge/` | Thin pointer consumed by agent harnesses |
| OpenClaw | `openclaw` | Chat/session/webhook transport, auth/model profiles, backend/gateway stability, forgebot runtime wiring |
| forgebot | OpenClaw workspace | Dispatcher / recorder on Forgejo: state → `label-set agent:running` → scenario decision → read-only owner review or bounded `auto-fix` → `comment --body-file` → final `label-set` |
| Owner repos | `voscli`, `nixos-config`, `openclaw`, `entwurf`, ... | Domain first review; GitHub Copilot custom agents live in *that* repo's `.github/agents/` |

OpenClaw does not need to remember the entire philosophy. It should provide the
transport and runtime connection. forge-config records the public operating model.

## Roadmap

See [ROADMAP.md](./ROADMAP.md).

## Related Notes

- Design: [Forge layer — shared code workspace for GLG agents](https://notes.junghanacs.com/botlog/20260527T073823)
- Parent pattern: [botment — talk to GLG's agents through comments](https://notes.junghanacs.com/botlog/20260328T112722)
- Series root: [Harness engineering: from stone axe to artificial intelligence](https://notes.junghanacs.com/botlog/20260319T152938)

## Status

✅ Connector v2 is active on Forgejo: multi-profile CLI, mutating-command
observability, `comment --body-file`, `label-remove`, status-safe `label-set`,
and `close` / `reopen`.

✅ 2026-08-20: GitHub Copilot is a second workshop host, not a foreign agent.
Identity stays here. Custom agent profile lives in `.github/agents/`.

Next steps live in [NEXT.md](./NEXT.md). Owner instructions live in
[AGENTS.md](./AGENTS.md).
