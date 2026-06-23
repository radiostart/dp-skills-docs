# Critic 으로 계획 챌린지하기

!!! info "한 줄 요약"
    `@ag-planner` 가 세운 계획을 구현 *전에* adversarial(red-team) 시각으로 검토해, 전제·범위 오류를 사후가 아닌 사전에 잡습니다.

## 언제 쓰나

- planner 의 계획이 **복잡하거나 영향 범위가 넓을** 때
- 전제·범위·엣지케이스에 **불확실성** 이 있을 때

오타·상수 변경 같은 trivial 한 작업에서는 **스킵해도 됩니다**. critic 은 planner 와 generator 사이의 *선택적* 단계입니다.

## 전제

- `@ag-planner` 가 이미 실행돼 `features/NN-*.plan.md` 가 있을 것

## 절차

### 1. critic 호출

계획을 확인한 뒤 generator 로 넘어가기 전에 호출합니다.

```text
@ag-planner          # 계획 수립 → features/NN-*.plan.md
@ag-planner-critic   # 계획 챌린지 → .plan.critic.md
```

critic 은 `.plan.critic.md` 에 **전제 · 범위 · 엣지케이스 · 대안 · 리스크** 5축으로 챌린지를 기록합니다. plan.md 와 코드는 **수정하지 않습니다**.

각 챌린지는 `## 합의` 표에 `severity`(`blocking`·`suggestion`·`nit`) + `status: open` 행으로 적힙니다. 이 표가 다음 단계로의 인계 계약입니다 — planner 재호출 시 planner 가 각 행의 `status` 를 `accepted`(반영)·`rejected`(기각)·`deferred`(이월) 로 전이하고, `/dp-skills:run-cycle` 은 이 표를 읽어 진행/정지를 판정합니다 (`blocking` 행이 `open`·`deferred` 로 남아 있으면 정지).

### 2. blocking 결함이 있으면 planner 재호출

critic 이 blocking 을 지적했다면, 코드가 아니라 **계획** 을 고칩니다.

```text
@ag-planner          # critic 지적 반영해 plan.md 갱신
@ag-generator        # 합의된 계획으로 구현
```

## 흔한 실수

- **critic 은 코드를 보지 않습니다.** 작성된 *계획* 을 봅니다. 작성된 *코드* 검토는 [`@ag-code-reviewer`](self-code-review.md) 의 몫입니다.
- critic 의 지적을 generator 단계에서 임기응변으로 때우지 않습니다 — plan.md 자체를 planner 재호출로 갱신해야 흐름이 일관됩니다.
- 사후 검증(요구사항 충족·테스트)은 critic 영역이 아닙니다 — 그건 `@ag-evaluator` 입니다. critic 은 *사전* 검토입니다.

## 다음 단계

- How-to: [Focus 로 방향 조정](focus-direction.md) — 대화 중 결정을 계획에 반영
- Reference: [`@ag-planner-critic`](../reference/agents/ag-planner-critic.md) · [`@ag-planner`](../reference/agents/ag-planner.md)
- Explanation: [에이전트 흐름](../explanation/agent-flow.md)
