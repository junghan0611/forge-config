# forge-config NEXT.md

> Volatile next actions only. Durable operating model lives in [README.md](./README.md), [ROADMAP.md](./ROADMAP.md), and [AGENTS.md](./AGENTS.md).

## Current stance — 2026-09-04

sorge 순회가 호출했다. 갱신 빚은 5커밋으로 작았고, 이 사이클의 결과는 두 가지다.

**1. 담당자 문서를 id 로 기억한다.** `AGENTS.md` 최상단에
`## 담당자 문서 — denote id 20260527T073823` 절을 세웠다. 제목·슬러그·파일명은 움직이고
`#+identifier:` 는 안 움직인다. 읽는 자리는 `~/org/botlog/` org 원본이고
`notes/content/` 의 export md 는 한 주기 늦으므로 판정 근거로 쓰지 않는다.

**2. `forge` 스킬 실물이 이 리포로 왔다 — 이관 완료.**
GLG 판정으로 진행, agent-config 담당자와 자문해 두 커밋으로 옮겼다.
`.claude/skills/forge/SKILL.md` (361줄, 내용 무변경) ← `dfe25c8` 푸시됨.
agent-config 는 `skills/forge → ../../forge-config/.claude/skills/forge` 상대 심링크
← `3b9f72e` **로컬 커밋, 푸시는 GLG 자리**. 도달 6/6 실측 (claude / pi /
claude-plugin / codex / copilot / kiro, 전부 md5 `3bd833c3…`).

**이제부터 이 집의 의무**: `bin/forge` 를 고치는 커밋이 `SKILL.md` 도 같이 들고 간다.
두 파일이 한 커밋에 없으면 그때부터 다시 갈린다. 근거와 이력은 `AGENTS.md`
§agent-config 와의 관계.

agent-config 담당자가 넘긴 것 하나 (전달받음, 이 집에서 재보지 않음): `run.sh` 의
`LINKED_SKILL_REPOS` 가 리포 이름과 스킬 이름을 이중 사용해서, `forge-config` 가
`forge` 를 소유하는 첫 사례에 `skills/forge-config` 를 찾다 실패하는 거짓 경고가
났다. 그 집이 `LINKED_SKILL_NAMES[forge-config]=forge` 매핑으로 고쳤고 클론 URL도
등록했다 — 새 기계에서 이 리포가 자동으로 pull 된다.

## Current stance — 2026-08-20

The workshop does not have to live only at home. Company GitHub can stand up
the same identity with Copilot custom agents. Trust is identity plus
cooperation, not host. Generic Copilot without a persona is still out of
scope. Forgejo ledger stays; do not machine-copy GitHub issues into Forgejo.

Docs rewritten for both forks (README / AGENTS / ROADMAP). This repo's
GitHub persona is `.github/agents/forge-config.agent.md` — visible in Assign
only after default-branch merge. Other GitHub repos (entwurf, company) own
their own `.github/agents/` files; forge-config only guides the shape.

Next caller may be pi, not Copilot. Do not drop the dual-workshop core.

## Current stance — 2026-06-04

`bin/forge` had no open/closed verb. A voscli-owner session hit the wall: #20/#22
were verifiably fixed in shipped versions and ready to close, but the only
sanctioned surface was comment/label — closing forced the web UI, so it was
punted to GLG. The REST layer always supported it (`PATCH /repos/{o}/{r}/issues/{n}`
`{"state":"closed"}`; the `api` helper is method-agnostic), only the `bin/forge`
surface was missing.

Added thin `close` / `reopen` verbs (`bin/forge`), documented the axis in
README + skill SKILL.md, live round-trip smoke on oracle `sandbox#14`
(open→closed→open→closed). Working-tree change, **not committed** — GLG commits.

Design note recorded in README/SKILL: open/closed is **orthogonal** to lifecycle
labels — `agent:done` = triage done, `closed` = resolved/withdrawn, tracking
stops. Verbs stay thin (single state PATCH); reason goes in a `comment` first.

Close policy — **resolved** (proposed by the voscli owner, both owners agree;
encoded in README + skill SKILL.md; GLG's go finalizes adoption). Keep the verb
**unguarded**; split by intent as a **convention, not a code gate** (same grain
as the auto-fix lane — validation by loop, not guard):

- **resolution close** — fixed in a shipped tag/commit + confirmed
  non-reproducing → owner agent closes autonomously, reason `comment` first.
- **judgment close** — won't-fix / duplicate / invalid-by-design / deprioritized
  → route via GLG or `human:needs-review`; not a solo owner call.

Rationale: forge is an internal surface (`glg-bot/*`, operator-invisible) so
close has low external-exposure cost; human-only would make GLG a bottleneck
(16 open voscli issues, several already fixed but unclosed is the evidence).

Pending GLG go: the voscli owner closes #20 (v2026.6.1 AND intersection) and
#22 (v0.7.2 cwd footgun, vocbot-verified) as resolution closes (comment +
close). #19/#21/#18 stay open (unresolved body bugs); forge-infra label tangle
on #4/#5/#7/#9 is the separate `label-set` cleanup track.

## Current stance — 2026-06-02

forge becomes *usable* only if pushing code to a forge remote is a sanctioned
path, not per-agent improvisation. `bin/forge` covers the REST layer
(issue/label/comment); the git push layer was missing. A 2026-06-02 session
(memex-kb/scanpdf → work forge) exposed it: no credential helper, askpass
blocked, agent fell back to token-in-URL by hand-sourcing `~/.env.local`.

Fixed by `bin/git-credential-forge` — a generic, host-autodetecting git
credential helper that bridges the same env.local profile token
(`ORACLE_FORGE_TOKEN` / `WORK_FORGE_TOKEN`) into git's native machinery. Servers
live on oracle/work; clients (thinkpad/nuc/laptop) push without askpass or
token-in-URL. Install is one line per machine, no nixos rebuild:
`git config --global credential.helper ~/repos/gh/forge-config/bin/git-credential-forge`.
See §8 for remaining hygiene.

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

### 0. Company GitHub workshop — other repos

This repo now has `.github/agents/forge-config.agent.md`. After GLG merge/push,
it appears in Assign. entwurf and company GitHub repos still need their own
personas; forge-config only guides the shape. Do not invent those files here.

### 1. Auto-fix v1 runner policy — freeze OpenClaw, move semantics here

v0 is live GREEN: sandbox#13 and voscli#15 both passed `agent:ready + auto-fix` → report skeleton → `agent:done + auto-fix`, and replay/idempotency smoke did not duplicate reports. Details moved to [ROADMAP.md](./ROADMAP.md) § "0.2.2 — auto-fix v0 signal routing E2E".

v1 seed is also live GREEN/YELLOW→GREEN:

- `glg-bot/voscli#14`: forgebot performed bounded workspace guard patch (`workspace-voc/SOUL.md`, `workspace-voc/TOOLS.md`) and recorded filled report; final `agent:done + auto-fix`.
- `glg-bot/voscli#16`: exposed `rg` no-match as another nonfatal sweep case after missing-path fix.
- `glg-bot/voscli#17`: post-fix regression passed (`agent:done + auto-fix`, comment 1, no new outcome=error observed).

OpenClaw side should now freeze at: wake/routing, bounded workspace/doc/config patch allowance, no owner entwurf, no broad source implementation, missing optional path / `rg` no-match as report findings.

Protocol is kept in repo docs and `bin/forge`, not a detached docs page. `bin/forge auto-fix-template ISSUE` emits the canonical report skeleton with live snapshot marker. `bin/forge doctor-labels REPO` checks required lifecycle/signal labels before a repo is onboarded to the lane.

Next forge-config work before expanding v1:

- Write runner policy into README/ROADMAP/AGENTS as stable rule, not only OpenClaw prompt:
  - allowed in hook: docs, prompts, workspace guards, small config surfaces;
  - forbidden in hook: broad product source implementation, long builds/tests, dependency installs, owner entwurf;
  - report required: exact files, commands, diff stat, same-shape sweep, independent review status, final meaning;
  - sweep hygiene: exact existing paths only; missing optional path or zero-match `rg` = report finding, not hook failure.
- Decide whether `doctor-labels` + manual label creation stays the safe boundary, or add mutating `label-ensure` later.
- Decide whether a validating apply command is needed before v1 writes, or template marker + current Forgejo state is enough for now.
- Define independent reviewer selection for later batch mode: different model/session, same owner agent resumed, or read-only sibling.

### 2. Label bootstrap decision

`agent:blocked` and `auto-fix` are prepared on current tested work repos (`glg-bot/{sandbox,voscli}`). `incidentcli` and future repos still need label checks before onboarding.

Current safe boundary:

- `bin/forge doctor-labels REPO` checks required labels and exits non-zero when missing.
- Label creation remains manual for now.

Revisit mutating `bootstrap-labels` / `label-ensure` only after another repo needs onboarding. Do not add this prematurely; it crosses from work-surface CLI into repo administration.

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

Basic replay/idempotency smoke passed on `glg-bot/voscli#15`: a manual ping after final `agent:done + auto-fix` did not create another auto-fix report.

Stronger mixed-lifecycle test remains optional:

- Re-add `agent:ready` without `label-set` so lifecycle becomes mixed.
- Confirm forgebot reads current state and skips owner review.
- Intentional re-run path should be `label-set agent:ready` → ready-only → triage allowed.

### 7. GLG batch implementation sorting surface

After first-review issues accumulate, define the surface GLG uses to choose implementation batches.

Questions:

- Which labels represent priority vs lifecycle status?
- Should owner-agent first-review comments use a stricter template for sorting?
- Do we need a `list-ready-reviewed` helper, or is Forgejo search enough?
- How do completed implementation batches get distinguished from forgebot `agent:done` triage completion?

Do not automate implementation yet. The immediate goal is a reliable sorted backlog for GLG-driven focused implementation.

### 8. git push auth — `git-credential-forge`

Helper lands as `bin/git-credential-forge` (POSIX sh, generic /
host-autodetecting: no profile arg, no url scope). git calls `helper get`, it
matches the request host — and path-prefix when git provides it — against
`ORACLE/WORK_FORGE_URL` in `~/.env.local`, emits the matching token, else stays
silent so git falls through (github, etc.). Token never written to disk; SSOT
stays `~/.env.local`. No forge host literal in any committed file (host derived
at runtime).

Tested 7/7 + real `git credential fill`: oracle host-only emit, work
`/forge`-path emit, **work non-forge path declined (leak prevented)**, host-only
fallback when useHttpPath off, `forgeXYZ`≠`forge` boundary, github + non-https +
store/erase decline.

**Path isolation** matters because the work forge lives under a path prefix
(`<work-host>/forge`) on a host that may serve other https git. With
`credential.useHttpPath true` the helper requires the forge path prefix, so a
non-forge service on the same host never gets the forge token. Oracle forge is a
dedicated host (no prefix) → host-only is correct there.

Install — per machine, machine-local `~/.gitconfig`, **NOT nixos-config**
(decided 2026-06-02). Lands in writable `~/.gitconfig`, not the home-manager nix
symlink `~/.config/git/config`; git merges both:

```bash
git config --global credential.helper ~/repos/gh/forge-config/bin/git-credential-forge
git config --global credential.useHttpPath true   # engages path isolation (esp. work host)
```

Fleet readiness (verified 2026-06-02): env.local is **synced** across machines —
thinkpad / oracle-host / work-host all carry `ORACLE_*` + `WORK_*`, forge-config
is cloned on all three, `~/.gitconfig` is writable on all three.

| machine | helper installed | note |
|---|---|---|
| thinkpad | ✅ helper + useHttpPath | client; pushes to both forges |
| oracle host | ⏳ after commit+pull | forge-config clone present, file pending push |
| work host | ⏳ after commit+pull | forge-config clone present; **needs useHttpPath** (work forge is path-prefixed) |
| nuc / laptop | ⏳ on next forge push | currently offline |

Rollout is gated on **GLG committing + pushing** the helper (file is uncommitted
on thinkpad). After push: `git -C ~/repos/gh/forge-config pull` then the two
install lines on the oracle host + work host.

Remaining hygiene:

- **migrate existing forge remotes**: any repo whose forge remote URL embeds a
  token (`https://glg-bot:<token>@host/...`) → rewrite to clean
  `https://<host>/...` so the helper (not the URL) carries auth. Audit:
  `git -C <repo> remote -v | grep '@'`.
- **optional**: a `bin/forge doctor-git` read-only check (helper reachable +
  each profile resolves a host + useHttpPath state), mirroring `doctor-labels`.

Boundary: forge-config owns the helper binary + protocol docs (SKILL.md §"git
push 인증"); individual repos own their remote URL hygiene.

## Do not regress

- Mutating commands must print `[forge] profile=... repo=... url=...` before writes.
- Use `label-set` for lifecycle status; avoid accumulating `agent:ready,running,done`.
- Treat webhook payload as wake signal only. Current Forgejo state wins; process only ready-only (`{agent:ready}`).
- Use `comment --body-file` for long comments.
- Keep forgebot as dispatcher / recorder, not implementer.
- Keep OpenClaw as transport/runtime wiring; forge-config owns the public operating model.
- Do not let `auto-fix` collapse into single-instance quick fixes; always sweep adjacent same-shape problems and leave the validation trail.
- Never embed a forge token in a remote URL or a committed file. git push auth goes through `git-credential-forge` reading `~/.env.local`; the work forge host stays out of public repos (identity term).
