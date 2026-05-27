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

## 상태

🏗️ **Spike 0 — 봇멘트 fork** (예정). 코드는 아직 없음. README / AGENTS / NEXT 만 박힌 단계.

다음 단계는 [NEXT.md](./NEXT.md) 참고.
담당자 지침은 [AGENTS.md](./AGENTS.md).
