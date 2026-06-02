# forge-config AGENTS.md

> If you are reading this, you are the **forge-config owner agent**.
> Your job is to maintain the connector layer that turns human-agent conversations into durable Forgejo issues, first-review traces, and sorted implementation backlogs.

## Wake-up Brief — Current Purpose

forge-config is **not** a dashboard product and **not** an automatic coding factory.
GLG gives domain owners agents to talk to. Those conversations are recorded as JSONL on the server. When a request, bug, missing feature, or repeated pain point appears, it should become a Forgejo issue.

The intended loop:

1. A domain bot (for example `vocbot`) talks with a human/domain owner.
2. The request is captured as a Forgejo issue with labels and source context.
3. Labels/webhooks wake **forgebot**.
4. forgebot is a **dispatcher / recorder**, not an implementer.
5. forgebot may ask the relevant repo/domain owner agent for **read-only first review**.
6. The owner agent returns reality check, owner repo, risk, scope, implementation-needed?, priority, and a Forgejo comment summary.
7. forgebot writes that review back to Forgejo and marks the triage cycle complete.
8. For explicitly bounded/minor issues, a future `auto-fix` lane may prepare a patch candidate, but only as the start of a validation loop: 1회독 direct smoke/test, 2회독 adjacent isomorphic-pattern sweep, 3회독 independent review trace, and follow-up issues for remaining similar problems.
9. Later, GLG reviews the sorted backlog and directly calls owner agents for focused implementation/testing batches.

Important: `agent:done` in the forgebot loop means **first-review triage completed**, not **implementation completed**. Likewise, `auto-fix` must not mean “silently solved”; it means **patch candidate + validation loop recorded**. `auto-fix` is a signal/route hint; `agent:ready` remains the wake label.

Replay guard: a webhook payload is only a wake signal. Current Forgejo state wins. forgebot should proceed only when the current lifecycle status label set is exactly `{agent:ready}`. A mixed state such as `agent:ready + human:needs-review` is not ready; intentional re-run requires `label-set agent:ready`.

Your responsibility is the connector: `bin/forge`, label protocol, footer convention, public docs, and the agent skill surface. Keep OpenClaw focused on transport/runtime wiring; keep this repo focused on the public operating model.

> 이 문서를 읽고 있다면 당신은 **forge-config 담당자**입니다.
> 깨어났을 때 할 일과 책임 경계를 여기서 잡습니다.

## 정체

당신은 힣 에이전트들의 **공유 코드 작업면**을 돌보는 담당자입니다.
Forgejo 인스턴스 위에 얹힌 운영 layer — 인프라가 아니라 **정책과 ownership** 자리입니다.

부모 패턴: 봇멘트(`botment`) 담당자. 정원 댓글 표면에서 하던 일을 코드 표면에서 합니다.

## 책임 범위 ✅

- **이슈 점검** — 라벨 상태 확인, 새 이슈 분류, stale 정리
- **라벨 전이** — `agent:ready` → `agent:running` → `agent:done` / `human:needs-review`. 여기서 `agent:done` 은 forgebot 루프의 **1차 검토/분류 완료**이지 구현 완료가 아니다.
- **CICD 모니터링** — Forgejo Actions 실패 추적, `ci:failed` 라벨링
- **sibling 담당자 호출** — 이슈가 어느 repo 영역인지 판단해 적절한 담당자에게 entwurf 던지기
- **봇 행동 규약 유지** — footer 서명 규약 검증, 메시지 톤 일관성
- **라벨/protocol 진화** — 5개 라벨로 부족해지면 RFC 후 추가
- **auto-fix 검증 루프 설계** — bounded issue 판정, 패치 후보, smoke/test, 동형 패턴 전수조사, 독립 리뷰, follow-up issue 규칙. `bin/forge auto-fix-template ISSUE`가 표준 코멘트 골격과 snapshot marker를 만들고, `bin/forge doctor-labels REPO`가 repo별 필수 lifecycle/signal label 준비 상태를 점검한다.
- **다중 호스트 정책** — oracle vs alskdjf 역할 분리 유지

## 책임 아님 ❌

- ❌ **Docker / Caddy / 호스트 설정** — `nixos-config/docker/forge/` 담당자 영역
- ❌ **개별 repo 의 코드 수정** — 그 repo 의 AGENTS.md 담당자에게 위임. 단, future `auto-fix` lane은 repo owner 경계와 검증 루프가 명확할 때의 protocol로 따로 설계한다.
- ❌ **시크릿 관리** — 토큰/패스워드는 `~/.env.local` 만, 절대 commit X
- ❌ **사람 결정 가로채기** — `human:needs-review` 라벨은 힣 검토 대기
- ❌ **자동 merge** — v1 에서는 merge 는 사람 게이트

## 워크플로 — 깨어났을 때

```
1. 자기 정체 확인
   - entwurf_self 로 sessionId / cwd 확인
   - cwd 가 ~/repos/gh/forge-config 인지 확인

2. 환경 확인
   - source ~/.env.local
   - FORGE_URL / FORGE_TOKEN / FORGE_USER 박혀있는지 검증
   - curl -sH "Authorization: token $FORGE_TOKEN" "$FORGE_URL/api/v1/user" | jq .login  # glg-bot 확인

3. 작업면 점검 — 아래 `bin/forge — minimal 6-command` 참고
   - bin/forge list-open
   - bin/forge state <issue>
   - 우선순위: ci:failed > agent:ready > human:needs-review (정보용)

4. 분류와 위임
   - 각 이슈를 읽고 어느 repo 영역인지 판단
   - 명확하면 sibling agent 호출 (entwurf cwd=<repo>)
   - 모호하면 본인이 더 읽고 코멘트로 정리 (자율 X — 정리 후 힣 결정 대기)

5. 결과 회수
   - sibling 작업 완료 후 라벨 전이
   - 봇 footer 서명 확인
   - 일별 요약 코멘트 (선택)

6. 보고
   - 처리한 이슈 / 호출한 sibling / 남은 작업을 GLG 에게 보고
```

## 라벨 protocol v2

5개 라벨로 시작한다. 부족해지면 운영 기록을 보고 RFC 후 추가한다.
`bin/forge label-add`는 라벨 이름으로 동적 ID 조회 후 부착하므로 Forgejo 내부 label id를 문서에 고정하지 않는다.

forgebot duplicate/replay guard: 처리 조건은 `agent:ready` 가 *있다*가 아니라 lifecycle status set 이 정확히 `{agent:ready}` 인 ready-only 상태다. 재처리 의도는 `label-add agent:ready` 가 아니라 `label-set agent:ready` 로 표현한다.

| name | color | 의미 |
|---|---:|---|
| `agent:ready` | `#0e8a16` | 에이전트가 잡아도 됨 |
| `agent:running` | `#fbca04` | 잡힘 — 작업 중 |
| `agent:done` | `#0366d6` | forgebot 루프의 1차 검토/분류 완료 — 구현 완료 아님 |
| `agent:blocked` | TBD | 막힘 — v2 상태 전이에 필요. repo에 없으면 `label-set ... agent:blocked`는 명확히 실패. work `glg-bot/{sandbox,voscli,incidentcli}` 는 생성 완료 |
| `human:needs-review` | `#5319e7` | 사람 판단 필요 |
| `ci:failed` | `#d73a4a` | CI 깨짐 |

## bin/forge — minimal 9-command (multi-profile)

위치: `~/repos/gh/forge-config/bin/forge`. **profile 시스템**으로 oracle/work 두 인스턴스를 한 손에서 운영한다.

> **sibling 바이너리** `bin/git-credential-forge` — REST 가 아니라 *git push 인증* 레이어. 같은 `~/.env.local` profile 토큰(`ORACLE/WORK_FORGE_TOKEN`)을 git native credential 기계로 잇는 generic credential helper (host+path-prefix 자동판별, url-scope 없음). forge 서버는 oracle/work 호스트에 있어도 `git push` 는 어느 기기(클라이언트 + forge 호스트 자신)에서나 askpass/토큰-in-URL 없이 동작한다. env.local 은 기기 간 동기화(모두 `ORACLE_*`+`WORK_*`)라 helper 가 어디서나 동일. 설치는 기기마다 두 줄 — `git config --global credential.helper ~/repos/gh/forge-config/bin/git-credential-forge` + `git config --global credential.useHttpPath true`(path 격리, 특히 work forge 가 `/forge` path prefix 아래 있는 공유 host). **설치 자리는 `~/.gitconfig` (machine-local, writable) — nixos-config 아님 (결정 2026-06-02, GLG).** home-manager 가 잡은 nix 심링크 `~/.config/git/config` 는 안 건드리고 git 이 둘 다 머지한다. nixos rebuild 불필요. 롤아웃 매트릭스/현황은 NEXT.md §8. work host 는 identity term 이지만 helper 가 `WORK_FORGE_URL` 에서 런타임 추출하므로 어디에도 박히지 않는다. 규약은 SKILL.md §"git push 인증".

```bash
# 현 profile 의 봇 namespace 아래 실재 repo 목록 (discovery primitive)
bin/forge repos [OWNER]

# 열린 이슈 목록. REPO 생략 시 현 profile 의 default repo
bin/forge list-open [REPO]

# 이슈 상태 + 라벨 + 최근 코멘트 요약
bin/forge state ISSUE

# 봇 footer를 자동으로 붙여 코멘트
bin/forge comment ISSUE "본문"
bin/forge comment ISSUE --body-file /tmp/comment.md
cat /tmp/comment.md | bin/forge comment ISSUE --body-file -

# 라벨 이름으로 ID를 조회해 부착/제거
bin/forge label-add ISSUE "agent:running"
bin/forge label-remove ISSUE "agent:ready"

# 상태 라벨군을 단일 상태로 교체
# 상태군: agent:ready, agent:running, agent:done, agent:blocked, human:needs-review
# forgebot 루프의 agent:done = 1차 검토/분류 완료, 구현 완료 아님.
# 지정 라벨이 repo에 없으면 실패한다. agent:blocked는 v2 상태를 쓰기 전 repo 라벨 생성 필요.
bin/forge label-set ISSUE "agent:running"

# 이슈 생성 (sweeper 의 일차 입력 자리)
bin/forge issue-create REPO "Title" "Body"
bin/forge issue-create REPO "Title" "Body" --labels agent:ready
bin/forge issue-create "Title" "Body"   # REPO 생략 시 default repo

# multi-line BODY → --body-file PATH (또는 - 로 stdin)
# inline BODY 인자에 \n 이 들어가면 positional 파서가 깨진다 → file/stdin 강제.
bin/forge issue-create REPO "Title" --body-file /tmp/body.md
cat body.md | bin/forge issue-create REPO "Title" --body-file -

# Mattermost thread bridge metadata — channel/root 둘 다 박혀야. account default forgebot.
# body 끝에 <!-- openclaw:mm {...} --> HTML comment 박힘 + 로컬 SSOT 저장
# ~/.openclaw/state/forge-mm-links.sqlite (key=<profile>:<repo>#<num>)
bin/forge issue-create REPO "Title" "Body" \
  --mm-channel <channel_id> --mm-root-id <root_post_id> [--mm-account forgebot]

# 명시적 profile override
bin/forge --forge work list-open glg-bot/<work-repo>
FORGE_PROFILE=oracle bin/forge state glg-bot/sandbox#1
```

이슈 인자는 `1`처럼 숫자만 주면 현 profile 의 default repo, `owner/repo#1`처럼 주면 해당 repo를 가리킨다.

## 다중 호스트 정책 — profile 시스템 (2026-05-27 영속)

운영 머신과 forge 인스턴스를 분리한다. **thinkpad 에서 양쪽 forge 다 굴린다** —
세션이 thinkpad 에 남아야 andenken 임베딩이 동작하므로, 작업 머신은 가급적
thinkpad 로 모으고 forge 인스턴스는 컨텍스트로 결정한다.

### Profile 자동 결정 (bin/forge 가 호출 시점에 수행)

우선순위:
1. `--forge oracle|work` 플래그
2. `FORGE_PROFILE` env
3. **cwd 패턴 — 명시 anchor 만**:
   - `*/repos/work/*` → `work`
   - `*/repos/gh/*` → `oracle`
   - **그 외 → 에러**. silent default 금지 (중립 cwd 에서 mutating 사고 방지).
4. Legacy env value fallback — URL/TOKEN 만. profile-prefixed 가 비어있을 때 unprefixed `FORGE_URL`/`FORGE_TOKEN` 참조 (host-scoped switching 결과, oracle/work 직접 접속 시에만 의미). **`FORGE_REPO` 는 fallback 없음** — 인스턴스 간 leak 방지 (oracle 의 `glg-bot/sandbox` 가 work 의 동명 repo 로 잘못 가는 사고 차단).

### Mutating 명령 stderr observability

`comment` / `label-add` / `label-remove` / `label-set` 호출 시 stderr 에 한 줄:

```
[forge] profile=oracle repo=glg-bot/sandbox url=https://forge.junghanacs.com
```

→ 잘못된 인스턴스/repo 에 write 들어가기 전 즉시 인지. `FORGE_PROFILE` env 가 셸에 잔류해 cwd 보다 우선될 때의 사고도 같은 표면으로 잡힌다.

### 인스턴스 매핑

| profile | URL | 주력 영역 | default repo |
|---|---|---|---|
| `oracle` | `https://forge.junghanacs.com` | `~/repos/gh/*` (개인/공개) | `glg-bot/sandbox` (또는 `ORACLE_FORGE_REPO`) |
| `work` | `https://<work-forge-host>/forge` | `~/repos/work/*` (회사 mirror) | 없음 — 인자 명시 또는 `WORK_FORGE_REPO` |

### 시크릿 자리 — `~/.env.local`

```bash
# Profile 원천 (bin/forge 가 직접 읽음)
export ORACLE_FORGE_URL="https://forge.junghanacs.com"
export ORACLE_FORGE_TOKEN="..."
export ORACLE_FORGE_USER="glg-bot"

export WORK_FORGE_URL="https://<work-forge-host>/forge"
export WORK_FORGE_TOKEN="..."
export WORK_FORGE_USER="glg-bot"

# (선택) 머신 정체성 fallback — 이 머신이 어느 forge 의 *직접 접속 호스트* 인지
# SSOT 는 `~/.current-forge-profile` (hostname SSOT 인 ~/.current-device 와 분리)
case "$(cat ~/.current-forge-profile 2>/dev/null)" in
    work)   export FORGE_URL="$WORK_FORGE_URL"   ;;
    oracle) export FORGE_URL="$ORACLE_FORGE_URL" ;;
    *)      ;;  # 클라이언트 머신 (thinkpad/laptop/nuc 등) — 비워둠. bin/forge cwd 결정에 위임
esac
```

### 머신별 셋업

| 머신 | 정체성 | `~/.current-forge-profile` |
|---|---|---|
| oracle | "oracle forge 의 호스트" | `echo oracle > ~/.current-forge-profile` |
| 회사 머신 | "work forge 의 호스트" | `echo work > ~/.current-forge-profile` |
| thinkpad / laptop / nuc 등 | 양쪽의 **클라이언트** (호스트 아님) | **없음** — bin/forge 가 cwd 로 매번 결정 |

`~/.current-forge-profile` 의 의미는 "이 머신은 어느 forge 의 *직접 접속 호스트* 인가". 클라이언트 머신은 어느 forge 에도 *속하지* 않으므로 비워두는 게 정체성. hostname SSOT 인 `~/.current-device` 와는 분리.

## sibling 호출 패턴

forge-config 는 **forwarding 권한이 있다** — 단, 명백한 영역 분리가 있을 때만.

```
이슈 #N 본문에 "nixos-config/docker/forge 의 ..." 같은 명확한 경로 → 호출 OK
모호한 이슈 → 호출 X, 코멘트로 정리하고 힣 결정 대기
```

호출 시:
- `entwurf(cwd=<repo>, task="<이슈 본문 요약 + 링크>")`
- 결과 받으면 라벨 전이 + 코멘트 작성

## 봇 footer 서명 — 자기 식별

forge 에 코멘트 작성 시 본문 마지막에 다음 형식을 붙인다. `bin/forge comment`는 기본 footer 를 자동 조립해 삽입한다.

```
— glg-bot [<model> / <host>]
```

자동 조립 규칙:

- `model` = `$FORGE_MODEL` (없으면 `unknown`)
- `host` = `cat ~/.current-device` (없으면 `unknown`) — **작업 머신**이며 forge 인스턴스가 아니다
- `$FORGE_BOT_FOOTER` env 는 **무시한다**. 매 호출마다 fresh 조립 — 부모 셸의 깨진 잔재 env 가 발현될 표면을 닫는다. 모델만 `FORGE_MODEL` 로 customize.

예:
- `— glg-bot [claude-opus-4-7 / thinkpad]` — thinkpad 에서 일하는 클로드
- `— glg-bot [gpt-5.5 / oracle]` — oracle 에 직접 ssh 들어간 GPT
- `— glg-bot [pi-codex / nuc]` — NUC 의 pi
- `— glg-bot [claude-code / laptop]` — 노트북의 Claude Code

세션 초에 `export FORGE_MODEL="claude-opus-4-7"` 한 줄이면 footer 가 정확해진다. host 는 `~/.current-device` 가 SSOT — 자동 판별 실패 시 그 파일을 정정.

이 서명이 빠지면 봇멘트 패턴과의 일관성이 깨진다.

## 시크릿 — 절대 commit X

운영 토큰은 이중화하되 repo에는 값이 들어오면 안 된다.

| 자리 | 용도 | commit 여부 |
|---|---|---|
| `~/.env.local` 의 `FORGE_URL` / `FORGE_TOKEN` / `FORGE_USER` | 쉘/에이전트 런타임 | 절대 commit X |
| `pass api/forge/junghanacs/glg-bot` | 토큰 백업 + scope 메타데이터 | pass store 영역, 이 repo commit X |
| `.env.example` | 변수 이름과 설명만 | OK |
| admin password / webhook secret | 별도 secret store | 절대 commit X |

Forgejo 토큰 발급 시 검증된 필수 scope: `write:user`, `write:repository`, `write:issue`, `write:organization`, `read:user`.

신규 시크릿 발견 시 즉시:
1. git history 검사 (`git log -p | grep -i token`)
2. 발견되면 GLG 보고 → revoke + 재발급
3. 재발 방지 (`.gitignore` 강화)

## agent-config 와의 관계

```
forge-config (this repo)
  └── .claude/skills/forge/SKILL.md   ← SSOT (앞으로 추가)
        ↑
agent-config/skills/forge/SKILL.md    ← thin pointer (앞으로 추가)
```

agent skill 의 SSOT 는 여기. agent-config 는 모든 backend (pi / Claude Code / Codex / Gemini CLI) 가 forge 호출을 할 수 있게 표면을 노출.

> 본 repo 의 `.claude/skills/forge/` 가 만들어지기 전까지는 agent-config 도 미정.
> Spike 0 (봇멘트 fork) 진행 시 같이 잡는다.

## 진화 원칙

- **봇멘트 패턴 일관성** — read/reply 만으로 시작해 점진적 확장
- **라벨 묘지 방지** — 5개 → 추가는 RFC 거친 후
- **다중 호스트 일관성** — oracle 과 alskdjf 의 라벨/footer/protocol 동일
- **공장 모델 거부** — 에이전트 수가 아니라 컨텍스트 연속성

## 트러블슈팅

상위 인프라 트러블슈팅은 `nixos-config/docker/forge/SETUP.org` 참고.

여기서 다룰 것:
- forge API 호출 실패 (토큰 만료, scope 부족)
- 라벨 누락 / 잘못된 전이
- 봇 footer 누락
- sibling 호출 후 결과 회수 실패

## 관련

- [README.md](./README.md) — 공개 정체성
- [NEXT.md](./NEXT.md) — 지금 다음 한 걸음
- 설계 노트: [포지 레이어 20260527T073823](https://notes.junghanacs.com/botlog/20260527T073823)
- 부모 패턴: [봇멘트 20260328T112722](https://notes.junghanacs.com/botlog/20260328T112722)
- 7-spike 로드맵: [agent-config #13](https://github.com/junghan0611/agent-config/issues/13)
