# forge-config NEXT.md

> 끝나지 않은 일과 후속 검증 항목. 영속 baseline 은 [AGENTS.md](./AGENTS.md), 이 문서는 *지금 시점의 다음 한 걸음*.

## 에이전트 자동화 루프 — 본 작업 진입 (2026-05-27 큰 산 넘음)

forge-config 는 그동안 **CLI/SSOT/라벨 protocol** 자리였다. 오늘 그 위에 **hook-driven auto-agent loop** 가 얹혔다. openclaw / openglg-config / forge-config / agent-config 네 자리 정합 + Phase 0~2 외부 라이브 round-trip 통과. **다음부터는 운영면 튜닝 / 확장 자리**.

### 루프 형상

```
Forgejo issue/label/comment 이벤트
  → Caddy `/openclaw/hooks/forgejo` (Authelia bypass, exact path)
    → OpenClaw `hooks.mappings[forgejo-raw]` (messageTemplate 렌더링)
      → `forge` agent turn (workspace-forge/, hook-only, gpt-5.4 Codex)
        → forge skill → `bin/forge state / label-add / comment`
          → 라벨 전이 + 코멘트 round-trip → Forgejo 로 회수
```

봇멘트(remark42 댓글 자동화)의 코드면 짝. 사람이 라벨만 토글하면 봇이 사이클을 굴리는 자리.

### 박힌 표면 — Phase 0/1/2 통과 (2026-05-27)

- ✅ **Phase 0 — OpenClaw hooks API 라이브** (`~/openclaw` `7efb9ca`, 09:08 KST) — idempotency replay dedup (`buildHookReplayCacheKey` dispatchScope 함정 박제, NEXT 보강 #10) / interactive session 격리 / hook session jsonl 분리 검증.
- ✅ **Caddy bypass 라이브** (09:10 직후) — `https://<work-host>/openclaw/hooks/forgejo` exact path bypass. `X-Forgejo-Delivery → X-OpenClaw-Idempotency-Key` 헤더 자동 복사. 외부 401 응답 확인 (Caddy 통과 → OpenClaw hook token gate 거부 = 의도 정확). openglg-config `2166fe0`.
- ✅ **Phase 1 — forge agent + workspace-forge/ + hooks.mappings** — `~/openclaw/workspace-forge/{IDENTITY,AGENTS,SOUL,USER,TOOLS}.md` 5파일 박힘 (hook-only 캐릭터, HEARTBEAT/MEMORY/DREAMS 없음). `agents.list[forge]` (openai/gpt-5.4 Codex) + `hooks.mappings[forgejo-raw]` + `allowedAgentIds: ["forge"]` 좁힘.
- ✅ **Phase 2 — Forgejo webhook 라이브 round-trip** (09:40 직후) — `glg-bot/sandbox` webhook (id=4) 박힘, `agent:ready` 라벨 토글 → 외부 trigger → forge agent 깨어남 → 5 step 사이클 → 라벨 전이 + 코멘트 round-trip 통과. **봇멘트 패턴의 코드면 짝이 처음 자율로 동작한 자리**.
- ✅ **FORGE_MODEL=gpt-5.4 박힘** — `~/openclaw/.env`. footer 자동 조립 `[gpt-5.4 / <work-host>]` 검증. 단일 정체성 함정 자리는 아래 튜닝 후속에 박힘.

### forge-config 가 받을 자리 (튜닝 후속, 운영 누적 후)

Phase 0/1/2 끝났으니 이쪽에서 다듬는 자리. 운영 한 사이클 굴려 자취 본 뒤 결정.

- **OpenClaw `agents.list[].env` 5.22 미지원 확정** (2026-05-28, openclaw 담당자 검토) — `AgentEntrySchema` `.strict()` 박혀있어 정의 안 된 키 거부. agent process 환경변수 자리 없음 (`params` 는 model params 자리, env 아님). 따라서 봇별 forge profile 주입은 **D1 정공법** (workspace 정체성 문서 한 줄) 이 영구 fallback. **5.26 신설 여부** 는 NEXT 0-upgrade 시점에 확인 자리 — 신설 시 voc/main/forge 셋 다 D1→D2 마이그레이션 가능 (`agents.list[voc].env: {FORGE_PROFILE: "work", WORK_FORGE_REPO: "glg-bot/voscli"}` 형태). 그 위에 자연 박힘.
- **`FORGE_MODEL` env 자리 — 단일 정체성 정공법 함정** (2026-05-27 발견, 운영 안정화 후 결정) — `~/.openclaw/.env` 에 `FORGE_MODEL=gpt-5.4` 박힘. OpenClaw gateway global env 라 *모든 agent process* (main/voc/forge) 가 같은 값 상속. bin/forge 는 호출자가 누구인지 신경 안 씀 (L97/L112). 영향:

  | agent | 실제 model | 박힐 footer | 정합 |
  |---|---|---|---|
  | forge (hook-only) | gpt-5.4 | `[gpt-5.4 / <work-host>]` | ✅ |
  | voc (channel) | gpt-5.4 | `[gpt-5.4 / <work-host>]` | ✅ 우연 |
  | main (channel) | claude-opus-4-7 | `[gpt-5.4 / <work-host>]` | ❌ 거짓 자취 |

  실용 위험은 낮음 — main/voc 가 채널면에서 사용자 호명으로 forge skill 부르는 일 운영 초기에는 거의 없음. 단 workspace-forge IDENTITY § "단일 정체성 정공법" 표가 보장하는 자취 (`main → [claude-opus-4-7 / <work-host>]`) 가 깨진다 — main 봇이 어쩌다 `bin/forge comment` 한 번이라도 박으면 거짓 자취.

  정공법 후보 (Phase 2 라이브 round-trip 끝낸 뒤 결정):
  - **B 안 (가장 정공법)** — OpenClaw 가 `agents.list[].env` 또는 `.environment` 같은 agent-specific env 자리를 지원하는지 확인 → 지원되면 `agents.list[forge].env: {FORGE_MODEL: "gpt-5.4"}` 로 이동. global env 제거.
  - **A 안 (fallback)** — 각 workspace AGENTS.md 에 `FORGE_MODEL=<model> bin/forge ...` 인라인 호출 룰 박음. agent 가 룰 잊으면 다시 거짓 자취 — 신뢰 못 함.
  - **C 안 (현 자리)** — global env 유지, "main/voc 가 forge skill 거의 안 부른다" 가정. 운영 누적 시 첫 거짓 자취 발견되면 B 로 즉시 이동.

  현재 결정: **C 로 라이브 검증 진행**, Phase 2 안정화 후 OpenClaw `agents.list[].env` 스키마 지원 여부 확인 → B → A 순. `~/openclaw/NEXT.md` 에도 짝 박을 자리.

- **workspace-forge/ 5파일 정체성 튜닝** (2026-05-27 검토 자리, 운영 한 사이클 후 다듬기) — `~/openclaw/workspace-forge/{IDENTITY,AGENTS,SOUL,USER,TOOLS}.md` 5파일 박힘. hook-only 캐릭터 (HEARTBEAT/MEMORY/DREAMS 없음) 정합. 매우 잘 박혔으나 미세 어긋남 4자리:
  - **IDENTITY L13** — "정체성 SSOT: 본 파일 + `agent-config/skills/forge/SKILL.md`" 표현. forge-config 기준으로는 후자가 *thin pointer*, SSOT 는 `bin/forge`. 의미 정합하지만 SSOT 중복 표기.
  - **TOOLS L9** — "v1.5 다섯 자리" 표기. 오늘 박힌 `--body-file` = v1.6. forge agent 가 issue-create 거의 안 쓸 가능성 높아 영향 미미.
  - **AGENTS** — **dedup cache key 함정** (OpenClaw NEXT 보강 #10 — `messageTemplate` 및 코멘트 본문 가변값 timestamp 박지 말 것) 이 어디에도 안 명시. agent 가 자기 코멘트 본문에 시간/모델 박는 것을 SOUL § 자기 표현 자제 가 막지만, dedup 위험 자리는 별도 한 줄 박을 자리.
  - **AGENTS L43** — `host` fallback "`~/.current-device` 또는 work hostname literal" — `~/.current-device` 가 SSOT. envless hook session 대비 fallback 박은 건 안전 선택, 그대로 둬도 무방.
- **봇 행동 규약 정합** — workspace-forge AGENTS.md vs 본 repo AGENTS.md "워크플로 — 깨어났을 때" 가 같은 그림인지. hook-driven turn 의 6단계 (정체 확인 → 환경 → 점검 → 분류/위임 → 회수 → 보고) 가 자연 발화되는지 라이브 검증.
- **messageTemplate 튜닝** — Forgejo payload → 봇 첫 메시지 사이의 verbosity 자리. `label.name == "agent:ready"` 가 아닐 때 `NO_REPLY` 가 정합인지 / 노이즈 라벨 (`ci:failed`/`human:needs-review`) 도 깨워야 하는지.
- **라벨 전이 자동화 검증** — `agent:ready` 라벨 토글 → 봇이 `agent:running` 박고 → 작업 → `agent:done`/`agent:blocked` 까지 한 사이클. `label-set` 동사 v2 후보가 이 시점에 필요해질 가능성 ↑ (`label-add` 만으로 누적되면 라벨 묘지).
- **footer hostname 노출 정책** — work forge 의 work-host 봇이 코멘트 시 footer 의 hostname 이 어떻게 박히는지. `FORGE_MODEL` / `~/.current-device` 가 OpenClaw workspace 안에서 어떤 값이 되는지 라이브 확인 후 AGENTS.md 정합화.
- **forge agent OAuth 분리** — Codex (`openai/gpt-5.4`) 인증이 main/voc 와 공유. work forge round-trip 운영 누적되면 회사 계정 분리 시점. NEXT 4 (`~/openclaw/NEXT.md`) 와 묶음.
- **dedup cache key 함정 가드** — `messageTemplate` 에 `now()` / timestamp 박지 말 것. `buildHookReplayCacheKey` 의 `dispatchScope` 가 message 까지 포함해 dedup 함 (openclaw NEXT 보강 #10). forge-config 의 SKILL.md / AGENTS.md 에 운영 룰로 박을 자리.
- **HMAC 검증 v2** — Phase 5 후속. 현재 D안 (`hooks.mappings.transform`) 에 raw body 안 들어와 HMAC 정확 검증 불가. 운영 누적 후 (a) OpenClaw upstream 기여 / (b) 얇은 forge-bus adapter 부활 / (c) Caddy 확장 모듈 셋 중 결정.

### 다음 세션 cold pickup 자리

| 자리 | 소요 | 우선순위 | 시점 신중도 |
|---|---|---|---|
| **A. glg-bot/voscli 에 같은 webhook 적용** (운영 확장) | 10분 | 운영 가치 ↑ | ⚠️ 회사 동료 (원민재님) 가 보는 자리. sandbox#2 사이클 자취 한 번 검토 후. forge agent 첫 코멘트 톤 사람이 보고 가는 자리 |
| **B. label-remove / label-set 동사 (v2)** | 30분~1시간 | 미관 자리 | 운영 누적 후 라벨 누적 (`ready,running,done`) 이 신호 노이즈로 발현될 때. 미리 박지 말 것 |
| **C. workspace-forge/ 5파일 정체성 미세 어긋남** (아래 튜닝 후속 #2) | 15분 | 낮음 | 운영 한 사이클 라이브 자취 본 뒤 |
| **D. FORGE_MODEL agent-specific env 정공법** (아래 튜닝 후속 #1) | 30분 | 잠재 위험 | main 봇이 어쩌다 forge skill 부르는 자리 발견되면 즉시 |
| ~~**E. forge repo 발견성 — `glg-bot/forge-config#1`**~~ | ✅ 해결 (2026-05-28) | — | `bin/forge repos` 동사 + SKILL.md "발견성" 단락 박힘. D1 (workspace 정체성 한 줄) 은 openclaw 담당자 자리. 라이브 검증은 vocbot 다음 turn |

추천 순서: **A 먼저 (운영 실 가치) → E 는 신뢰 자취 회복 자리 (운영 봇이 봇멘트면처럼 발화하려면 발견성 보장 필수) → C/D 는 라이브 자취 본 뒤**. A 진입 전 sandbox 한 사이클 자취 확인 자리.

### Webhook 자동 등록 — 별도 도구 X (2026-05-27 결정)

질문: forge-config 에 `bin/forge webhook-register` 같은 동사 박아야 하나?

답: **박지 않는다.** 운영 사실:

- Forgejo webhook 등록은 repo 당 한 번 curl POST (혹은 Settings UI 클릭) 로 끝남
- payload: `{type:"forgejo", config:{url, content_type, secret, authorization_header}, events:[...], active:true}`
- 한 사이클 운영 → repo 가 늘어도 매번 한 번씩만 박으면 되는 자리
- bin/forge SSOT 의 5동사 minimal 원칙 깸 (책임 경계: webhook 등록은 *Forgejo 운영자*, bin/forge 는 *에이전트 작업면*)
- 운영 누적 → repo 5~10개 넘어 매번 박는 부담 명백해지면 그때 동사 추가

라이브 절차는 `~/openclaw/NEXT.md` § Phase 2 박힌 curl 한 줄 그대로. 운영 사실 누적은 `~/openclaw` 쪽에서 박고, forge-config 는 *받을 일 없는* 자리.

### 정합 자리 — 흩어지지 않게

| repo | 자리 |
|---|---|
| **`~/openclaw`** | `hooks.mappings` / `agents.list[forge]` / `workspace-forge/` (실행 표면) |
| **`forge-config`** (이 repo) | `bin/forge` SSOT, 라벨 protocol, 봇 행동 규약, footer 규약 (정책 표면) |
| **`agent-config`** | `skills/forge/SKILL.md` (thin pointer, openclaw workspace 가 symlink 로 소비) |
| **`openglg-config`** | Caddy bypass + idempotency header (transport 표면) |

네 자리가 **계속 정합**되어야 루프가 깨지지 않는다. AGENTS.md / SKILL.md 갱신 시 항상 두 짝 같이.

---

## 지금 상태 (2026-05-27 KST 큰 산 넘음)

오늘 박힌 자취는 위 § "에이전트 자동화 루프" 에 영속화. 본 섹션은 그 이전부터의 누적 자리.

- ✅ Oracle 인프라 가동: `https://forge.junghanacs.com` (Forgejo 15.0.2)
- ✅ admin user `junghanacs` 생성
- ✅ Caddy 리버스 프록시 + Let's Encrypt
- ✅ README / AGENTS / NEXT 초안 박힘
- ✅ 검증용 sandbox: `glg-bot/sandbox#1, #2` (state/list/comment + label-add round-trip 검증)
- ✅ `bin/forge` minimal 4-command 박힘: `list-open`, `comment`, `label-add`, `state`
- ✅ GitHub remote: `junghan0611/forge-config` (공개면)
- ✅ Forgejo repo: `glg-bot/forge-config` (운영면 — 라벨 5개 박힘). GitHub의 짝
- ✅ AGENTS.md 운영 사실 영속화 (라벨/forge cmd/footer/secret)
- ✅ agent skill thin pointer 박힘: `agent-config/skills/forge/SKILL.md` 가 `~/repos/gh/forge-config/bin/forge` SSOT 가리킴. `ghcli` 패턴 (scripts/ 폴더 제거, SKILL.md 단일). 동사 4개 / footer 자동 조립 / 라벨 5개 — bin/forge 와 정합 (2026-05-27).
- ✅ `bin/forge` multi-profile (oracle + work) — `--forge` 플래그 / `FORGE_PROFILE` env / cwd 패턴 자동 결정. thinkpad 에서 양쪽 forge round-trip 검증 통과 (2026-05-27).
- ✅ 머신 정체성 SSOT 분리 — `~/.current-forge-profile` 도입. hostname SSOT (`~/.current-device`) 와 forge profile 결정 SSOT 분리. 직접 접속 호스트 (oracle/work) 만 박고 클라이언트 머신 (thinkpad/laptop/nuc) 은 비워둠. `.env.local` 의 case 분기 입력 갱신, AGENTS.md / SKILL.md 예시 정합 (2026-05-27).
- ✅ `issue-create` 동사 추가 — 5번째 동사. v1.5 박힘. sweeper 의 일차 입력 자리. atomic 라벨 옵션 (`--labels`), footer 자동 부착, default repo fallback. work forge 이슈 요청 응답 + oracle sandbox 5단계 round-trip 검증 통과. (2026-05-27)
- ✅ `issue-create --body-file PATH` (또는 `-` = stdin) 추가 — v1.6. multi-line BODY 지원. inline BODY 인자에 `\n` 들어가면 positional 파서가 split 해 `pos_count > 3` guard 발동하던 v1.5 한계 해소. work host 의 운영 봇이 multi-line 이슈 생성 실패 → memory 파일 우회 기록만 남기는 자리로 신호 노출. TITLE 은 여전히 single-line. SKILL.md / AGENTS.md 짝으로 갱신. (2026-05-27)
- ✅ `repos [OWNER]` 동사 추가 — 6번째 동사. discovery primitive. `GET /api/v1/users/{owner}/repos?limit=50` 회수, sort + open_issues_count + description 표시. OWNER 생략 시 `$FORGE_USER` (= `glg-bot`). vocbot `glg-bot/forge-config#1` 자취 (`teamgoqual/hej-nixos-cluster` 추측 → 404 → admin escalation) 의 namespace 막힘 자리 닫음. oracle/work 양쪽 round-trip 통과 — work forge 가 정확히 3개 (`incidentcli`/`sandbox`/`voscli`) 회수 확인. SKILL.md "발견성" 단락 + 사용법 예제 짝으로 갱신. (2026-05-28)
- ✅ `log_context` 호출 자리 fix — dispatch 단계에서 호출하던 자리 cmd_*안으로 이동, 실 target repo 정확 노출. forge bot (Mattermost) 의 첫 라이브 검증에서 work profile + explicit repo 인자 조합 시 stderr `repo=<unset>` 자취 노출 → 진단: explicit repo 인자 파싱 *전에* `DEFAULT_REPO` (work 는 빈 값) 만 비춤. oracle 은 default `glg-bot/sandbox` 가 우연히 맞아 가려져 있던 잠재 자리. `cmd_comment`/`cmd_label_add`/`cmd_issue_create` 각각 실 repo 결정 직후 `log_context "$repo"` 호출, `log_context` 시그니처에 인자 추가. work/oracle 양쪽 sandbox round-trip 재검증 통과 — `[forge] profile=<p> repo=glg-bot/sandbox url=<u>` 정확 노출. (2026-05-28 forge bot 보고 → 담당자 fix)
- 🔄 alskdjf 인프라 — 힣이 직접 구축 중 (2026-05-27). work forge 가동 검증 통과 (v15.0.2). 운영 양식: **mirror (M)** — 코드 SSOT 외부 GitHub, Forgejo 는 봇 운영면(이슈/라벨/PR 자동화). oracle 의 *짝(paired)* 패턴과 분기된 자리. work 토큰 이중화(pass + ~/.env.local prefix 변수). multi-host 처리는 `~/.env.local` host-scoped case + `~/.current-device` 로 해결 (아래 "미루어도 되는 것" 참조)

## 다음 한 걸음 — 결정/대기 항목

forge-config 구성에 대해 담당자가 박은 현재 결정. 기준: 봇멘트 패턴 일관성 / 봇로그 SSOT / 닫힌 계 / 최소 권한.

### 1. 디렉토리 구조 — minimal-first로 결정

```
forge-config/
├── README.md             ✓ 박힘
├── AGENTS.md             ✓ 박힘
├── NEXT.md               ✓ 박힘
├── bin/forge             ✓ sh + curl + jq minimal 4-command
├── docs/                 → 힣 결정 대기 (근거: 영속 문서 분리는 필요하지만 지금은 봇로그+AGENTS가 SSOT)
├── .claude/skills/forge/ → 다음 spike: thin pointer 표면
└── examples/             → 실제 round-trip 누적 후 필요 시
```

- `ROADMAP.md`는 아직 만들지 않음. 7-spike SSOT는 이미 agent-config #13 + 봇로그에 있으므로 중복 방지.
- `lib/`는 아직 만들지 않음. 4-command가 한 파일로 읽히는 동안은 봇멘트 패턴상 단일 스크립트 유지.

### 2. CLI 언어 — sh + curl + jq 로 결정

- 결정: **POSIX sh + curl + jq**.
- 근거: 봇멘트 패턴 일관성, OpenClaw/서버 환경 호환성, 닫힌 계에서 의존성 최소화.
- 보류: sibling 호출/웹훅/복잡한 상태 전이가 들어와 단일 sh가 불투명해질 때만 deno/go 재검토.

### 3. agent-config 와의 표면 분리 방식 — thin pointer 로 결정

- 결정: **(D) thin pointer**.
- agent-config skill은 “`~/repos/gh/forge-config/bin/forge`를 사용하라”는 표면만 제공.
- SSOT는 forge-config에 둔다. symlink/submodule/복사는 동기화 실패면이 늘어나므로 v1에서 제외.

### 4. GitHub repo 공개 시점 — round-trip 후 공개/푸시 유지로 결정

- 현재 remote는 `https://github.com/junghan0611/forge-config.git`.
- 결정: README/AGENTS만 있는 상태가 아니라, `bin/forge` 첫 round-trip까지 포함한 뒤 공개 이력으로 둔다.
- 주의: 시크릿은 계속 `~/.env.local`/`pass`에만. 이 repo에는 변수명/절차만 둔다.

### 5. 라벨 자동 bootstrap CLI — 다음 spike로 대기

- 현재 sandbox + forge-config repo 둘 다 5개 라벨 박힘 (수동 curl로 5회 반복).
- 결정: minimal 4-command에는 `bootstrap-labels` 넣지 않음 (책임 경계 흐려짐 회피).
- 운영 누적이 늘어 매번 5개 박는 부담이 명백해지면 그때 CLI 추가. 지금은 절차 문서로 충분.

### 6. forge-config 자체 Forgejo repo — `glg-bot/forge-config` 로 결정 (2026-05-27)

- 결정: **`glg-bot/forge-config`** (Forgejo).
- 의미: GitHub `junghan0611/forge-config`는 *공개면*(코드/문서), Forgejo `glg-bot/forge-config`는 *그 뒤의 힣 에이전트 공간*(이슈/라벨/봇 활동).
- 봇멘트 패턴 그대로: 정원(공개 markdown) ↔ 댓글(remark42 인프라)와 동형. 짝이지만 독립 자리.
- 박힌 자리: https://forge.junghanacs.com/glg-bot/forge-config (라벨 5개 완비, 이슈 0)
- 다음: 첫 이슈는 담당자가 자기 작업을 dogfood로 박는 자리 (예: agent-config thin pointer 진행 추적)

## 미루지 말 것

- `~/.env.local` 에 `ORACLE_FORGE_*` / `WORK_FORGE_*` 두 세트 유지 (glg-bot 사용자/token)
- `bin/forge state 1` / `list-open` / `comment` round-trip 결과를 깨뜨리지 않기
- 봇 footer 자동 조립 유지: `— glg-bot [<FORGE_MODEL> / <~/.current-device>]`
- profile 자동 결정 (cwd `*/repos/work/*` → work, 그 외 → oracle) 회귀 금지

## v2 후보 — 운영 누적 후 다시 보기

- **`label-remove` / `label-set` 동사** — 현재 `label-add` 만 있어 라벨 누적 (`agent:ready,agent:running,agent:done`). 상태 라벨 (agent:*) 은 mutual exclusive 가 정합 — `label-set agent:done` 이 `agent:running` 같은 이전 상태 라벨 자동 제거하는 동사 필요.
- ~~**Forge webhook → auto-agent 루프**~~ — **진입 (2026-05-27)**. OpenClaw `hooks.mappings` 네이티브 경로로 D안 정공법 박힘. work forge (`<work-host>/forge`, `glg-bot/voscli`) 가 첫 검증 자리. 진행 위치는 `~/openclaw/NEXT.md` § 0 + 본 문서 상단 "에이전트 자동화 루프" 섹션. **회사일 우선** 원칙 그대로 — oracle 자동화는 work 운영 안정화 후.
- **Profile 등록 외부화** — 현재 `bin/forge` 는 `oracle` / `work` 두 profile 만 hardcode. fork 사용자가 자기 운영 (`personal`/`vps`/`homelab` 등) 으로 자유롭게 추가하려면 `FORGE_PROFILES` env 등록제로 외부화 필요. POSIX sh 의 indirect 변수 expansion 한계로 `eval` 의존 — v1 운영 안정 후 검토.
- **URL sanity 검증** — `bin/forge` 에 `https?://` 형식 검사 박기. 현재는 깨진 unprefixed 잔재가 들어와도 `curl: Bad hostname` 으로 실패하지만 명시적 검증이 운영 안전성 ↑.
- **Footer hostname 노출 정책** — 회사 머신에서 oracle forge 에 코멘트 시 footer 에 회사 hostname 박힘. `FORGE_HOST_LABEL` override env 도입, 또는 운영 룰 (각 머신은 자기 host 의 forge 만) 로 처리.

## 미루어도 되는 것

- ROADMAP.md (봇로그/issue와 중복되지 않을 때)
- docs/ 분리 (AGENTS/README가 비대해질 때)
- agent skill farm 배포 (`./run.sh setup` — agent-config 담당자 결정. thin pointer 박혔으므로 진행 가능, 단 다중 호스트에 `~/repos/gh/forge-config` 클론 사전 확인 필요)
- ~~**bin/forge multi-host 지원**~~ — **해결됨 (2026-05-27 1차)**. `~/.env.local` 끝에 host-scoped `case` 문 + `~/.current-device` 값으로 prefix 변수(`<HOST>_FORGE_*`)를 unprefixed `FORGE_*` 로 export. 단일-host 인터페이스 — host 에 직접 ssh 들어간 경우에 적용.
- ~~**bin/forge profile 자동 결정 (multi-instance from one host)**~~ — **해결됨 (2026-05-27 2차, gpt-5.5 자문 반영)**. thinkpad 에서 양쪽 forge 다 굴리는 자리. profile 우선순위:
  1. `--forge` 플래그 → `FORGE_PROFILE` env → cwd 패턴(`*/repos/work/*` → work, `*/repos/gh/*` → oracle, **그 외 → 에러**) → legacy URL/TOKEN fallback.
  2. footer 자동 조립: `[$FORGE_MODEL / $(cat ~/.current-device)]`. `FORGE_BOT_FOOTER` env 무시.
  3. **`FORGE_REPO` 는 profile-specific 만** (unprefixed fallback 제거) — 인스턴스 간 leak 방지.
  4. **Mutating 명령 stderr 로그** — `comment`/`label-add` 진입 시 `[forge] profile=... repo=... url=...` 한 줄로 잘못된 write 사전 차단.
  5. 검증: thinkpad 에서 oracle/work 양쪽 round-trip + 중립 cwd 에러 + override 5/5 통과.

## 영속 사실 — 이전 완료 / 대기

다음 사실들의 영속 자리:

- ✅ 라벨 5개 정의 → AGENTS.md "라벨 protocol v1" 섹션 (Turn 2 영속화)
- ✅ footer 서명 규약 → AGENTS.md "봇 footer 서명" 섹션 (Turn 2 영속화)
- ✅ 호스트 역할 분리 (oracle vs alskdjf) → README.md 호스트 표 (초안에 이미 있음)
- ⏳ gotchas (INSTALL_LOCK env, inode caching, write:user scope) → 봇로그 20260527T073823 LLMLOG 섹션에 박힘. 운영이 누적되면 `docs/gotchas.md`로 별도 분리 검토
- ⏳ multi-host 처리 방식 (`~/.env.local` host-scoped case + `~/.current-device`) — 2026-05-27 결정. 운영 안정화 후 AGENTS.md "다중 호스트 정책" 섹션에 박을 자리
