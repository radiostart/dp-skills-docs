# Focus 로 방향 조정

!!! info "한 줄 요약"
    메인 대화에서 내린 결정("소프트 딜리트 빼자")을 `.focus.md` 에 기록해, 다음 에이전트 호출에 자동 반영합니다.

## 언제 쓰나

- planner 의 계획을 받고 **방향을 바꾸기로** 결정했을 때
- 에이전트에게 **추가 지시·제약** 을 전달하고 싶을 때

메인 대화에서 내린 결정은 서브에이전트에게 보이지 않습니다 — 에이전트는 별도 인스턴스로 실행되기 때문입니다. focus 가 그 간극을 메웁니다.

## 전제

- 활성 프로젝트가 있을 것

## 절차

### 1. 결정을 기록

```text
/dp-skills:focus "소프트 딜리트는 archived_at 타임스탬프로 대체"
```

`.focus.md` 에 지시가 기록됩니다.

### 2. 에이전트 재호출

```text
@ag-planner          # .focus.md 반영된 새 계획
@ag-planner-critic   # 챌린지도 .focus.md 반영
@ag-generator
@ag-evaluator
```

`@ag-planner`·`@ag-planner-critic`·`@ag-generator`·`@ag-evaluator` 가 호출 시 `.focus.md` 를 자동으로 읽습니다.

### 3. 지시가 끝나면 삭제

```text
/dp-skills:focus --clear
```

## 흔한 실수

- **메인 대화에 결정을 적는 것만으로는 에이전트에 전달되지 않습니다.** 반드시 `/dp-skills:focus` 로 `.focus.md` 에 기록해야 다음 호출에 반영됩니다.
- 더 이상 유효하지 않은 지시를 `--clear` 하지 않고 두면 이후 사이클에 계속 끼어듭니다. 반영이 끝나면 정리하세요.

## 다음 단계

- How-to: [Critic 으로 계획 챌린지](critic-review.md)
- Reference: [`/dp-skills:focus`](../reference/skills/focus.md)
