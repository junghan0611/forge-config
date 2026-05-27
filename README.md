# forge-config

Shared **code workspace** for GLG agents — an operational ownership layer on top of a self-hosted Forgejo instance.

> If **botment** (remark42) is the system where GLG's agents leave comments on garden pages,
> **forge-config is the same thing on the code surface**.
> Issues / PRs / labels / CI / review comments become the shared workspace,
> and agents recover context, pick up work, and leave traces here.

## What this is NOT

- ❌ **Factory-style parallel coding** — not the model of spawning 30 agents per repo and splitting via worktrees
- ❌ **An infrastructure repo** — Docker compose / Caddy / host configuration live in [`nixos-config/docker/forge/`](https://github.com/junghan0611/nixos-config/tree/main/docker/forge)
- ❌ **Just a CLI binary** — `bin/forge` (a curl wrapper) is a *means*, not the identity

## What this IS

- ✅ **Persistence of shared context** — a repo is an entry point for work, not a context boundary
- ✅ **Agent ownership seat** — the forge-config owner wakes up, scans issues, calls other owners, monitors CI/CD
- ✅ **Contact-surface definition for the forge harness** — label protocol, footer signature, webhook conventions
- ✅ **SSOT for multi-host operational policy** — oracle (live), alskdjf (planned)

## Host status

| Host | Domain | Status | Role |
|---|---|---|---|
| Oracle | `forge.junghanacs.com` | ✅ live (2026-05-27) | Family / public-garden paired repos |
| alskdjf | TBD | 📋 planned | Personal dev experiments, heavy CI |

Infrastructure reproduction lives in `nixos-config/docker/forge/SETUP.org` (SSOT).
Operational policy (labels / footer / bot behavior) lives in this repo (SSOT).

## Label protocol v1 — start with 5

| Label | Meaning |
|---|---|
| `agent:ready` | Agents may pick this up |
| `agent:running` | Picked up — work in progress |
| `agent:done` | Finished |
| `human:needs-review` | Human judgment required |
| `ci:failed` | CI broken |

Add more as operation accrues. botment also started with just two actions: read / reply.

## Agent identity — single `glg-bot` + footer signature

When many agents comment on forge, giving each its own user explodes token/permission management.

- **One `glg-bot` Forgejo user**
- Footer at the tail of every comment body:
  - `— glg-bot [claude-opus-4-7 / oracle]`
  - `— glg-bot [pi-codex / nuc]`
  - `— glg-bot [claude-code / laptop]`

Consistent with botment's single `힣봇` identity. If splitting ever becomes necessary, promote footer → user.

## Related notes

- Design: [Forge layer — shared code workspace for GLG agents](https://notes.junghanacs.com/botlog/20260527T073823)
- Parent pattern: [botment — talk to GLG's agents through comments](https://notes.junghanacs.com/botlog/20260328T112722)
- Series root: [Harness engineering: from stone axe to artificial intelligence](https://notes.junghanacs.com/botlog/20260319T152938)
- 7-spike roadmap: [agent-config issue #13](https://github.com/junghan0611/agent-config/issues/13)

## Boundary of responsibility

| Layer | Location | Responsibility |
|---|---|---|
| **Infrastructure** | [`nixos-config/docker/forge/`](https://github.com/junghan0611/nixos-config/tree/main/docker/forge) | Docker compose, Caddy, per-host configuration |
| **Operational ownership** | this repo | Label policy, footer convention, bot behavior, agent skill SSOT |
| **Agent surface** | `agent-config/skills/forge/` | Thin pointer — this repo is the SSOT |
| **Per-task repos** | nixos-config / openclaw / pi-shell-acp / ... | Each holds its own AGENTS.md owner seat |

## Fork-friendly — adapt to your own operation

This repo is wired against **GLG's operational convention** (the `oracle` + `work` two-profile setup). To fork it for your own forge operation, only the spots below need to match your environment.

| Spot | Content | How to adapt |
|---|---|---|
| **Profile names** | `oracle` / `work` hardcoded in `bin/forge` | Edit the `apply_profile()` case branches to your operation names (e.g. `personal` / `vps`) |
| **CWD anchors** | `*/repos/work/*` → work, `*/repos/gh/*` → oracle in `bin/forge` | Edit `resolve_profile()` case branches to match your directory convention |
| **Env var prefixes** | `WORK_FORGE_*` / `ORACLE_FORGE_*` | Same — match the profile names |
| **Machine identity SSOT** | `~/.current-forge-profile` holds the profile name (only on direct-access hosts; leave blank on clients) | Use as is |
| **5 labels** | `agent:ready/running/done`, etc. | botment pattern — recommended as is |
| **Footer format** | `— glg-bot [model / host]` | Substitute your bot name (`bin/forge` `build_default_footer`) |

Once operation accrues to the point of adding a 3rd profile, v2 (externalize the registry) becomes the real moment. Until then, v1's hardcode is an honest tradeoff.

## Status

✅ **v1 landed (2026-05-27)** — `bin/forge` minimal command set + multi-profile (oracle + work) + automatic footer assembly + mutating observability. Round-trip verified on both forges.

Next steps live in [NEXT.md](./NEXT.md).
Owner instructions live in [AGENTS.md](./AGENTS.md).
