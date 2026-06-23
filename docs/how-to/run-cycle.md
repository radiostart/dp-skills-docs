# 미완료 feature 순차 사이클 자동 진행

!!! info "한 줄 요약"
    활성 project 의 미완료 feature 들을 `/dp-skills:run-cycle` 로 순서대로 자동 순회하며 planner→critic→generator→evaluator 사이클을 돌립니다. green 신호엔 자동 진행, 문제 게이트에선 멈춥니다.

## 언제 쓰나

- features 가 여러 개 쌓여 있고, 각각 `@ag-planner`→`@ag-generator`→`@ag-evaluator` 를 일일이 멘션하기 번거로울 때
- 미완료 feature 를 **순서대로 한 번에** 진행하되, 중요한 판단 지점(plan 승인·검증 실패)에서는 멈추고 개입하고 싶을 때

`/dp-skills:run-cycle` 은 기존 수동 사이클을 **대체하지 않습니다** — 그 위에 얹는 선택적 순차 러너입니다. 단건만 돌리거나 흐름을 세밀히 제어하고 싶으면 종전대로 `@ag-*` 를 직접 멘션하면 됩니다.

## 전제

- 활성 project 가 있을 것 (`/dp-skills:project {NAME}` 로 활성화). 없으면 안내 후 종료합니다.
- development phase 일 것 — QA phase 에서는 동작하지 않습니다 (`/dp-skills:qa` 사용).
- `project.md` 의 `## 목표` 에 `[ ]` 미완료 feature 가 있을 것. 모두 `[x]` 면 "신규 작업 없음" 으로 종료합니다.

## 절차

### 1. 실행

```text
/dp-skills:run-cycle
```

특정 feature 만 돌리려면 부분집합을 인자로 줍니다.

```text
/dp-skills:run-cycle #3-5      # 3~5번
/dp-skills:run-cycle #3 #4     # 3번, 4번
```

### 2. 큐 확인

`project.md` 의 `[ ]` 미완료 feature 를 **목록 순서대로** 큐에 담아 보여주고 승인을 받습니다.

```text
큐: #3 order-cancel → #4 refund → #5 history (순서대로). 진행할까요?
```

> 순서는 `project.md` 목록 순서를 그대로 따릅니다 (PM·analyze 가 의도한 순서). feature 간 의존성을 자동 계산하지 않으므로, 순서가 중요하면 목록을 먼저 정리하세요.

### 3. feature 별 사이클 자동 진행

각 feature 마다 다음을 자동으로 돕니다.

| 단계 | 동작 | 멈춤? |
|---|---|---|
| planner | `.plan.md` 산출 | **⏸ plan 승인** (feature 마다 1회 — 코드 쓰기 전 검문) |
| critic | `.plan.critic.md` 합의표 | blocking 행이 미해결(`open`·`deferred`)이면 ⏸ / 모두 `accepted`·`rejected` 거나 suggestion·nit 만이면 자동 진행 |
| generator | 구현 | — |
| evaluator | VERIFICATION REPORT | `READY` → `[x]` 갱신 후 다음 feature (도메인 지식 환류 `detected` 항목은 누적 — 멈추지 않음) / `NOT_READY`·`block` → ⏸ |

green 신호(critic 합의·evaluator `READY`)에는 멈추지 않고 다음으로 넘어갑니다.

### 4. 정지 시 대응

문제 게이트(plan 미승인·critic blocking·`NOT_READY`·`block`)에 걸리면 **큐 전체가 멈추고** 사유·REPORT 를 보여줍니다. 뒤 feature 로 자동 진행하지 않습니다 (다음 feature 가 현재 feature 에 의존할 수 있기 때문).

```text
/dp-skills:fix-review     # finding 별 처리 경로 추천
# … 수정 …
/dp-skills:run-cycle      # 재실행 = 재개 (정지 지점부터 이어감)
```

재개를 위한 별도 플래그는 없습니다. `run-cycle` 은 매번 `project.md` + 산출물(`.plan.md`/`.eval.md`)에서 진행 상태를 다시 파악하므로 **재실행이 곧 재개**입니다. 부분집합으로 돌렸다면 같은 인자를 다시 넘기세요.

### 5. 완료

모든 feature 가 `READY` 가 되면 처리한 feature 목록과 각 verdict 요약을 출력합니다.
사이클 중 누적된 도메인 지식 환류 제안이 있으면 표로 보여주고 기록 여부를 물어봅니다
(정지 시에도 그때까지의 누적분을 함께 출력합니다).

## 흔한 실수

- **병렬을 기대하지 마세요.** run-cycle 은 한 번에 1 feature 만 처리합니다 — `.focus.md`·`STATE.md` 싱글톤 충돌과 generator 파일 충돌을 피하기 위한 의도된 설계입니다.
- **plan 승인은 매 feature 마다 멈춥니다.** "자동" 의 의미는 critic·generator·evaluator·다음 feature 이행을 자동화하는 것이지, plan 검토를 없애는 게 아닙니다.
- **상태 파일을 찾지 마세요.** run-cycle 은 상태를 저장하지 않습니다. 진행 상황의 SSOT 는 `project.md` 체크박스와 feature 산출물입니다.
- 정지 후 곧바로 다음 feature 가 진행되길 기대하지 마세요 — 의존성 안전을 위해 큐 전체가 멈춥니다.

## 다음 단계

- Explanation: [에이전트 흐름](../explanation/agent-flow.md) — 수동 사이클의 단계별 책임
- How-to: [Critic 으로 계획 챌린지](critic-review.md) · [PR 직전 셀프 코드 리뷰](self-code-review.md)
- Reference: [`/dp-skills:run-cycle`](../reference/skills/run-cycle.md)
