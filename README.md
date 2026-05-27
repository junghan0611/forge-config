# forge-config

힣 에이전트의 **공유 코드 작업면** — 셀프호스트 Forgejo 위에 얹은 운영 ownership layer.

> 봇멘트(remark42)가 정원 페이지에 분신이 코멘트를 남기는 시스템이라면,
> **forge-config는 코드 표면에서 같은 일을 하는 자리**다.
> issue / PR / label / CI / review comment 가 공유 작업면이 되고,
> 분신들이 컨텍스트를 회수하고 일을 잡고 자취를 남긴다.

## 무엇이 아닌가

- ❌ **공장식 병렬 코딩** — repo에 30개 에이전트 띄워서 worktree 분할하는 모델이 아니다
- ❌ **인프라 repo** — Docker compose / Caddy / 호스트 설정은 [`nixos-config/docker/forge/`](https://github.com/junghan0611/nixos-config/tree/main/docker/forge) 영역
- ❌ **단순 CLI binary** — bin/forge curl wrapper는 *수단*이지 정체성이 아니다

## 무엇인가

- ✅ **공유 컨텍스트의 지속성** — repo는 작업의 진입점이지 컨텍스트 경계가 아니다
- ✅ **에이전트 ownership 자리** — forge-config 담당자가 깨어나서 이슈 점검 → 다른 담당자 호출 → CICD 모니터링
- ✅ **forge 하네스의 접점 정의** — 라벨 프로토콜, footer 서명, webhook 규약
- ✅ **다중 호스트 운영 정책의 SSOT** — oracle (운영), alskdjf (예정)

## 호스트 현황

| 호스트 | 도메인 | 상태 | 역할 |
|---|---|---|---|
| Oracle | `forge.junghanacs.com` | ✅ 가동 (2026-05-27) | 가족/공개 정원 페어 repo |
| alskdjf | TBD | 📋 계획 | 개인 dev 실험, 무거운 CI |

인프라 재현 절차는 `nixos-config/docker/forge/SETUP.org` 가 SSOT.
운영 정책 (라벨/footer/봇 행동 규약) 은 이 repo 가 SSOT.

## 라벨 프로토콜 v1 — 5개로 시작

| 라벨 | 의미 |
|---|---|
| `agent:ready` | 에이전트가 잡아도 됨 |
| `agent:running` | 잡힘 — 작업 중 |
| `agent:done` | 완료 |
| `human:needs-review` | 사람 판단 필요 |
| `ci:failed` | CI 깨짐 |

운영하면서 부족하면 추가. 봇멘트도 read/reply 두 동작으로 시작했다.

## 에이전트 식별 — 단일 `glg-bot` + footer 서명

여러 에이전트가 forge에 코멘트할 때 각자 사용자 만들면 토큰/권한 관리가 폭발한다.

- **단일 `glg-bot` Forgejo 사용자**
- 코멘트 본문 마지막에 footer:
  - `— glg-bot [claude-opus-4-7 / oracle]`
  - `— glg-bot [pi-codex / nuc]`
  - `— glg-bot [claude-code / laptop]`

봇멘트의 `힣봇` 단일 신원과 일관. 미래에 분리가 필요하면 footer → user 로 승격.

## 관련 노트

- 설계: [포지 레이어 — 힣 에이전트의 공유 코드 작업면](https://notes.junghanacs.com/botlog/20260527T073823)
- 부모 패턴: [봇멘트 — 힣의 분신과 댓글로 소통하라](https://notes.junghanacs.com/botlog/20260328T112722)
- 상위 시리즈: [하네스 엔지니어링: 돌도끼에서 인공지능까지](https://notes.junghanacs.com/botlog/20260319T152938)
- 7-spike 로드맵: [agent-config issue #13](https://github.com/junghan0611/agent-config/issues/13)

## 책임 경계

| Layer | 위치 | 책임 |
|---|---|---|
| **인프라** | [`nixos-config/docker/forge/`](https://github.com/junghan0611/nixos-config/tree/main/docker/forge) | Docker compose, Caddy, 호스트별 설정 |
| **운영 ownership** | this repo | 라벨 정책, footer 규약, 봇 행동, agent skill SSOT |
| **agent surface** | `agent-config/skills/forge/` | thin pointer — this repo 가 SSOT |
| **개별 작업 repo** | nixos-config / openclaw / pi-shell-acp / ... | 각자 AGENTS.md 가진 담당자 자리 |

## Fork 친화 — 자기 운영에 맞추기

이 repo 는 **GLG 의 운영 컨벤션** 위에 박혀있다 (`oracle` + `work` 두 profile). 다른 사람이 fork 해서 자기 forge 운영면으로 쓰려면 아래 자리만 자기 환경에 맞게 수정하면 된다.

| 자리 | 내용 | 수정 방법 |
|---|---|---|
| **profile 이름** | `bin/forge` 에 `oracle`/`work` hardcode | `apply_profile()` 의 case 분기를 자기 운영 이름으로 (예: `personal`/`vps`) |
| **cwd anchor** | `bin/forge` 에 `*/repos/work/*` → work, `*/repos/gh/*` → oracle | `resolve_profile()` 의 case 분기를 자기 디렉토리 컨벤션에 맞게 |
| **환경 변수 prefix** | `WORK_FORGE_*` / `ORACLE_FORGE_*` | 동일 — profile 이름과 맞춰서 |
| **머신 정체성 SSOT** | `~/.current-forge-profile` 에 profile 이름 박음 (직접 접속 호스트만, 클라이언트는 비워둠) | 그대로 사용 가능 |
| **라벨 5종** | `agent:ready/running/done` 등 | 봇멘트 패턴이라 그대로 권장 |
| **footer 형식** | `— glg-bot [model / host]` | 자기 봇 이름으로 (`bin/forge` build_default_footer) |

운영 누적되어 3번째 profile 추가 시점이 v2 (등록제 외부화) 진짜 필요 시점. 그때까지 v1 의 hardcode 는 정직한 트레이드오프.

## 상태

✅ **v1 박힘 (2026-05-27)** — `bin/forge` 4-command minimal + multi-profile (oracle + work) + footer 자동 조립 + mutating observability. 양쪽 forge round-trip 검증 통과.

다음 단계는 [NEXT.md](./NEXT.md) 참고.
담당자 지침은 [AGENTS.md](./AGENTS.md).
