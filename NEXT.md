# forge-config NEXT.md

> Volatile next actions only. Durable operating model lives in [README.md](./README.md), [ROADMAP.md](./ROADMAP.md), and [AGENTS.md](./AGENTS.md).

## Current stance — 2026-06-01 AM

forge-config is the connector between OpenClaw/forgebot, owner agents, and GLG's later batch implementation flow.
The forgebot v2 triage loop is already live and documented in [ROADMAP.md](./ROADMAP.md) § "0.2.0 — forgebot triage loop v2".

New design pressure from `glg-bot/voscli#14`: `auto-fix` must mean **patch candidate + validation loop**, not “resolved.” The goal is to increase 검증 회독 count and preserve review-session traces as assets.

Remember:

- `agent:done` in the forgebot loop means **first-review triage completed**, not implementation completed.
- forgebot is dispatcher / recorder, not implementer.
- owner-agent calls are read-only first review by default.
- canonical hook sequence is `state → label-set agent:running → scenario decision → comment --body-file → final label-set`.
- OpenClaw owns transport/runtime/auth/model/gateway/lifecycle wiring up to “forgebot wakes”; forge-config owns lifecycle protocol, `auto-fix` semantics, sweeper semantics, validation loops, and follow-up issue rules.

## Next actions

### 1. Auto-fix lane protocol

Protocol is being kept in repo docs and `bin/forge`, not a detached docs page. `bin/forge auto-fix-template ISSUE` now emits the canonical report skeleton with live snapshot marker.

ClawSweeper reference narrowed: borrow safety grammar only — conservative default, review-before-mutation, durable report, marker-backed comment, snapshot drift guard, deterministic mutation gate, lane labels. Do **not** force GLG auto-fix into ClawSweeper's Plan / Review / Apply product pipeline; our core is 회독-centered validation.

Decisions still needed before automation:

- label name: `auto-fix` vs namespaced `forge:auto-fix` / `lane:auto-fix`;
- whether to add `bin/forge label-ensure` before depending on the label;
- whether `auto-fix-template` snapshot marker is enough for v0 drift guard or needs a validating apply command;
- final state convention for “patch candidate + validation loop recorded” before GLG commit;
- second-pass reviewer selection rule: different model/session, same owner agent resumed, or read-only sibling.

Concrete seed: `glg-bot/voscli#14` KST/wording/guard-class issue.

### 2. Label bootstrap decision

`agent:blocked` is prepared on current work repos (`glg-bot/{sandbox,voscli,incidentcli}`), but new repos still need labels created manually.

Decide after a few more repos:

- keep manual label creation as operational setup, or
- add a `bootstrap-labels` / `label-ensure` command to `bin/forge`.

Do not add this prematurely; it crosses from work-surface CLI into repo administration.

### 3. Profile registry / URL sanity

Current profiles are hardcoded in `bin/forge` (`oracle`, `work`). This is acceptable while there are two profiles.

Revisit when:

- a third Forgejo profile appears, or
- fork users need clean customization.

Candidate work:

- externalize profile registry instead of hardcoded `case` branches;
- validate `FORGE_URL` with an explicit `https?://` sanity check before curl;
- keep `FORGE_REPO` leak-prevention: no unprefixed fallback.

### 4. Footer identity policy

Current footer:

```text
— glg-bot [<FORGE_MODEL or unknown> / <~/.current-device>]
```

Known issue: global `FORGE_MODEL` can lie when multiple OpenClaw agents with different models share the same process environment.

Next decision point:

- if OpenClaw supports per-agent env, move `FORGE_MODEL` there;
- otherwise keep workspace-level invocation discipline;
- consider `FORGE_HOST_LABEL` only if host disclosure becomes undesirable.

### 5. HMAC verification v2

Current hook path relies on OpenClaw token/idempotency handling. Exact Forgejo HMAC verification needs raw body access.

Options still open:

- OpenClaw upstream support for raw-body transform;
- thin `forge-bus` adapter;
- Caddy module / edge verification.

Defer until the triage loop proves stable under real traffic.

### 6. Duplicate/replay guard live test

Policy is now defined in ROADMAP.md: current Forgejo state wins over webhook payload, and forgebot proceeds only when lifecycle labels are exactly `{agent:ready}`.

OpenClaw follow-up after prompt/workspace update:

- Create or reuse an issue that has final status (`agent:done` or `human:needs-review`).
- Re-add `agent:ready` without `label-set` so the lifecycle set becomes mixed.
- Confirm forgebot reads current state and skips owner review.
- Confirm no duplicate long comment and no duplicate owner-agent delegation.
- Intentional re-run path should be `label-set agent:ready` → ready-only → triage allowed.

### 7. GLG batch implementation sorting surface

After first-review issues accumulate, define the surface GLG uses to choose implementation batches.

Questions:

- Which labels represent priority vs lifecycle status?
- Should owner-agent first-review comments use a stricter template for sorting?
- Do we need a `list-ready-reviewed` helper, or is Forgejo search enough?
- How do completed implementation batches get distinguished from forgebot `agent:done` triage completion?

Do not automate implementation yet. The immediate goal is a reliable sorted backlog for GLG-driven focused implementation.

## Do not regress

- Mutating commands must print `[forge] profile=... repo=... url=...` before writes.
- Use `label-set` for lifecycle status; avoid accumulating `agent:ready,running,done`.
- Treat webhook payload as wake signal only. Current Forgejo state wins; process only ready-only (`{agent:ready}`).
- Use `comment --body-file` for long comments.
- Keep forgebot as dispatcher / recorder, not implementer.
- Keep OpenClaw as transport/runtime wiring; forge-config owns the public operating model.
- Do not let `auto-fix` collapse into single-instance quick fixes; always sweep adjacent same-shape problems and leave the validation trail.
