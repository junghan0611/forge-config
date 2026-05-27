# forge-config NEXT.md

> 끝나지 않은 일과 후속 검증 항목. 영속 baseline 은 [AGENTS.md](./AGENTS.md), 이 문서는 *지금 시점의 다음 한 걸음*.

## 지금 상태 (2026-05-27 KST 09:30)

- ✅ Oracle 인프라 가동: `https://forge.junghanacs.com` (Forgejo 15.0.2)
- ✅ admin user `junghanacs` 생성
- ✅ Caddy 리버스 프록시 + Let's Encrypt
- ✅ README / AGENTS / NEXT 초안 박힘
- ❌ 코드 (`bin/forge`, skill) — **아직 없음**. 같이 논의 후 결정.
- ❌ GitHub repo 생성 — **아직 안 함**. 로컬 git init 만 됨.

## 다음 한 걸음 — 같이 논의할 항목

힣 에이전트들이랑 forge-config 를 어떻게 구성할지 같이 잡아야 함:

### 1. 디렉토리 구조 — voscli 패턴 차용?

```
forge-config/
├── README.md             ✓ 박힘
├── AGENTS.md             ✓ 박힘
├── NEXT.md               ✓ 박힘
├── ROADMAP.md            ? — voscli 처럼 7-spike SSOT 만들까
├── docs/
│   ├── label-protocol.md ? — 5개 라벨 의미
│   ├── footer-signing.md ? — "— glg-bot [model / host]" 규약
│   ├── host-setup.md     ? — oracle / alskdjf 재현 가이드
│   └── gotchas.md        ? — INSTALL_LOCK env, inode caching 등
├── bin/forge             ? — curl wrapper CLI (sh 또는 deno?)
├── lib/                  ? — 공용 함수
├── .claude/skills/forge/
│   └── SKILL.md          ? — agent surface (cwd 기반 auto-load)
└── examples/             ? — list / comment / label-add 실예
```

### 2. CLI 언어 — sh / bash vs deno vs go?

- **sh + curl + jq** — 봇멘트 패턴 일관성, OpenClaw 컨테이너 호환
- **deno** — 타입 안전, 의존성 0 (단일 binary), 사용자 다른 곳에서도 활용
- **go (native)** — voscli 의 GraalVM 처럼 binary 배포, 더 무거움

봇멘트가 sh 인 것 보면 일관성 위해 **sh + curl + jq** 가 자연스러움.
근데 라벨 protocol / sibling 호출 같은 게 복잡해지면 deno 도 후보.

### 3. agent-config 와의 표면 분리 방식

- (A) symlink — `agent-config/skills/forge/` → `~/repos/gh/forge-config/.claude/skills/forge/`
- (B) git submodule — agent-config 안에 forge-config 박기
- (C) duplicate — forge-config SKILL.md 를 agent-config 로 복사 (lockfile 동기화)
- (D) thin pointer — agent-config skill 은 단 몇 줄로 "forge-config repo 의 bin/forge 실행해" 만 안내

voscli 는 (D) 패턴 (`.pi/settings.json` 으로 pi 공유, `.claude/skills/voscli/` cwd 기반 auto-load).
**(D) 가 가장 단순**. 변경 추적도 forge-config 한 자리에서.

### 4. GitHub repo 공개 시점

- 지금 push? — README/AGENTS 만 박힌 상태로 noisy
- spike 0 (봇멘트 fork) 완료 후 push? — 첫 round-trip 검증 후 공개가 깔끔
- private 으로 먼저 push → 어느 정도 진행 후 public 전환?

봇멘트도 별도 공개 repo 없이 agent-config 안에 살았다. forge-config 는 더 큰 ownership 이라 별도가 맞는데, **언제 public 으로 가나** 는 결정 필요.

### 5. 라벨 / 라벨 자동 생성

- Forgejo 에 5개 라벨 미리 박을까? — bin/forge bootstrap-labels
- 첫 issue 만들 때 만들까? — 게으른 초기화

### 6. 첫 issue / 첫 작업

- `forge-config` 자체의 이슈를 forge.junghanacs.com 에 박는다 (eat your own dog food)
- 첫 이슈: "Spike 0 — 봇멘트 fork → forge API endpoint swap"
- 라벨 protocol 검증의 첫 케이스

## 미루지 말 것

- `~/.env.local` 에 `FORGE_URL` / `FORGE_TOKEN` / `FORGE_USER` 박기 (glg-bot 사용자 생성 + token 발급 후)
- 그게 안 박혀있으면 어떤 cli 도 동작 X

## 미루어도 되는 것

- ROADMAP.md (논의 후)
- bin/forge 코드 (논의 후)
- agent-config 표면 (CLI 결정 후)
- alskdjf 인프라 (oracle 운영 안정화 후)

## 영속 사실 (NEXT 에서 빠질 것)

다음 사실들은 진척 시 NEXT 에서 빼고 AGENTS / README / docs 로 이전:

- 라벨 5개 결정 → AGENTS.md 라벨 섹션
- footer 서명 규약 → AGENTS.md footer 섹션
- 호스트 역할 분리 (oracle vs alskdjf) → README.md 호스트 표
- gotchas (INSTALL_LOCK env, inode caching) → docs/gotchas.md

지금은 영속할 자리가 정해지지 않아 NEXT 에 같이 두지만, 진척 시 옮긴다.
