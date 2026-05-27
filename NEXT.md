# forge-config NEXT.md

> 끝나지 않은 일과 후속 검증 항목. 영속 baseline 은 [AGENTS.md](./AGENTS.md), 이 문서는 *지금 시점의 다음 한 걸음*.

## 지금 상태 (2026-05-27 KST 10:40)

- ✅ Oracle 인프라 가동: `https://forge.junghanacs.com` (Forgejo 15.0.2)
- ✅ admin user `junghanacs` 생성
- ✅ Caddy 리버스 프록시 + Let's Encrypt
- ✅ README / AGENTS / NEXT 초안 박힘
- ✅ 검증용 sandbox: `glg-bot/sandbox#1` 라벨 5개 + 봇 footer 코멘트 검증 완료
- ✅ `bin/forge` minimal 4-command 박힘: `list-open`, `comment`, `label-add`, `state`
- ✅ GitHub remote: `junghan0611/forge-config`
- ❌ forge-config 자체 Forgejo repo/issue — 아직 없음. sandbox 다음 단계.
- ❌ agent skill surface — 아직 없음. `bin/forge` 안정화 후 thin pointer.

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

### 5. 라벨 / 라벨 자동 생성 — 명시 bootstrap은 다음 spike로 대기

- 현재 sandbox에는 5개 라벨이 수동 검증 완료.
- 결정: minimal 4-command에는 `bootstrap-labels`를 넣지 않음. 작동면이 늘어나면 책임 경계가 흐려짐.
- → 힣 결정 대기: forge-config 자체 repo를 만들 때 `bootstrap-labels`를 CLI에 추가할지, 초기 세팅 문서로만 둘지.

### 6. 첫 issue / 첫 작업 — sandbox 완료, forge-config 자체 issue는 힣 결정 대기

- 완료: `glg-bot/sandbox#1` — "Spike 0 — 봇멘트 fork → forge API endpoint swap".
- 다음 후보: Forgejo 안에 `glg-bot/forge-config` 또는 `junghanacs/forge-config`를 만들고 동일 프로토콜로 dogfood.
- → 힣 결정 대기 (근거: GitHub 공개 repo와 Forgejo 운영 repo의 ownership/user namespace는 사람 결정이 필요).

## 미루지 말 것

- `~/.env.local` 에 `FORGE_URL` / `FORGE_TOKEN` / `FORGE_USER` 유지 (glg-bot 사용자/token)
- `bin/forge state 1` / `list-open` / `comment` round-trip 결과를 깨뜨리지 않기
- 봇 footer 자동 삽입 유지: `— glg-bot [gpt-5.5 / oracle]`

## 미루어도 되는 것

- ROADMAP.md (봇로그/issue와 중복되지 않을 때)
- docs/ 분리 (AGENTS/README가 비대해질 때)
- agent-config 표면 (CLI 최소 검증 후)
- alskdjf 인프라 (oracle 운영 안정화 후)

## 영속 사실 (NEXT 에서 빠질 것)

다음 사실들은 진척 시 NEXT 에서 빼고 AGENTS / README / docs 로 이전:

- 라벨 5개 결정 → AGENTS.md 라벨 섹션
- footer 서명 규약 → AGENTS.md footer 섹션
- 호스트 역할 분리 (oracle vs alskdjf) → README.md 호스트 표
- gotchas (INSTALL_LOCK env, inode caching) → docs/gotchas.md

지금은 영속할 자리가 정해지지 않아 NEXT 에 같이 두지만, 진척 시 옮긴다.
