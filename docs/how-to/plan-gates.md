# Plan 검증 게이트 통과하기

!!! info "한 줄 요약"
    `@ag-planner` 가 plan 을 저장하면 `plan-validate.py` 가 **형식·Open Questions·분량** 세 가지를 게이트로 검증합니다. 통과 못 하면 planner 가 보완 후 재검증하므로, 사용자는 주로 Open Questions 질의에 답하는 역할입니다.

## 언제 쓰나

- planner 가 **미해결 Open Questions** 를 보고하며 결정을 요청할 때
- plan 이 **형식·분량 경고** 로 반려돼 왜 그런지 이해하고 싶을 때

게이트는 planner·generator·evaluator 가 **같은 도구**(`plan-validate.py`)로 공유합니다 — 세 단계가 서로 다른 기준으로 통과/반려하지 않도록 fail-closed 로 묶여 있습니다.

## 전제

- `features/` 가 있는 프로젝트일 것 (게이트는 `features/NN-*.plan.md` 를 검증)

## 세 가지 게이트

| 게이트 | 무엇을 보는가 | 실패 시 |
|---|---|---|
| **형식** | `## 구현 계획` H2 + 모드별 필수 H3 섹션(변경 파일·구현 순서/스텝 등) | planner 가 누락 섹션 보완 후 재검증 |
| **Open Questions** | 미해결 OQ 카테고리마다 plan 에 처리 마커가 있는가 | 사용자 결정 필요 — 아래 참조 |
| **분량** | plan 이 비대해지지 않았는가 (회차 이력 누적 방지) | 이력 잔재 정리 후 1회 재검증 |

### Open Questions 게이트

feature 의 `## Open Questions` 에 미해결(`- [ ]`) 항목이 있으면, planner 는 카테고리별로 처리한 뒤 잔존 항목에 **처리 마커** 를 답니다.

| 카테고리 | planner 처리 | 잔존 시 마커 |
|---|---|---|
| (a) 같은 도메인 추가 read | scope 추가 Read 로 해결 | `범위 제외` 또는 `추정 구현` |
| (b) cross-domain 산출물 부재 | `/dp-skills:discover` 권고 → 사용자가 "그냥 진행" 명시 시에만 | `추정 구현` 또는 `범위 제외` |
| (c) 외부 시스템 spec 부재 | spec 확보 질의 → 없으면 범위 제외 | `범위 제외` (기본) |
| (d) 비즈니스 결정 | **반드시 사용자에게 질의** — 임의 결정 금지 | `범위 제외` 만 (`추정 구현` 불인정) |

- `추정 구현` — 산출물·spec 없이 추정으로 구현. **사용자 "그냥 진행" 명시가 전제** 이며, generator 가 해당 코드에 TODO 주석을 답니다.
- `범위 제외` — 해당 인터페이스·결정을 이번 구현에서 제외.

사용자 역할은 주로 **(b)·(c)·(d) 질의에 답하는 것** 입니다. 특히 (d) 비즈니스 결정은 임의로 채우지 않고 반드시 사용자 답을 받습니다 — 한 번 잘못된 전제가 들어가면 generator·evaluator 전체가 오염되기 때문입니다.

### 분량 가드

plan.md 는 항상 **"최신 확정 상태" 만** 담습니다. `N차 갱신` 헤더, `[C# 해소]` 이력 주석, 기각된 대안의 장문 사유를 본문에 누적하지 않습니다 (회차 간 이력은 git, critic 반영 disposition 은 `.plan.critic.md` 합의표). 분량 경고가 뜨면 planner 가 이력 잔재를 정리하고 1회 재검증합니다.

!!! note "왜 분량을 막나"
    이력 누적으로 plan 1개가 135k자까지 비대해진 실사례가 있습니다 — 후속 에이전트(critic·generator·evaluator) 전원이 매 라운드 plan 전문을 다시 로딩하므로, 분량은 곧 토큰 비용입니다.

## 게이트 실패가 generator·evaluator 단계에서 발견되면

generator·evaluator 가 OQ 마커 누락을 발견하는 시점엔 planner 인스턴스가 이미 종료돼 있습니다. 발견한 에이전트는 직접 고치지 않고 미해결 항목·plan 경로를 보고한 뒤 멈춥니다. 사용자는 둘 중 하나로 라우팅합니다.

1. **plan 보완** — `@ag-planner` 재호출로 마커를 채우거나 항목을 해결(`[x]`)
2. **(d) 직접 해결** — 결정을 답하면 feature 항목을 `[x] {항목} — 결정: {내용}` 으로 고친 뒤 중단된 단계를 재호출

## 흔한 실수

- **OQ 를 "합리적 추정" 으로 건너뛰지 마세요.** 게이트는 fail-closed 입니다 — 마커 없이는 generator·evaluator 가 Major 이슈로 반려합니다.
- **(d) 비즈니스 결정에 `추정 구현` 은 불인정입니다.** 사용자 결정 또는 `범위 제외` 만 통과합니다.
- **plan 에 회차 이력을 쌓지 마세요.** 분량 가드가 경고합니다 — 최신 확정 상태만 남기세요.

## 다음 단계

- How-to: [Critic 으로 계획 챌린지](critic-review.md) · [미완료 feature 순차 자동 진행](run-cycle.md)
- Explanation: [에이전트 흐름](../explanation/agent-flow.md)
- Reference: [`plan-validate.py`](../reference/tools/plan-validate.md) · [`@ag-planner`](../reference/agents/ag-planner.md)
