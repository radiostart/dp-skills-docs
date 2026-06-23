# Characterize 모드

!!! info "한 줄 요약"
    레거시 코드의 *현재 동작* 을 spec 으로 포착하는 characterization test 사이클로 전환합니다. 구현은 건드리지 않습니다.

## 언제 쓰나

- 테스트가 없는 **레거시 코드** 를 안전하게 만들고 싶을 때
- 리팩터·수정에 앞서 **현재 동작을 먼저 고정** 하고 싶을 때

새 동작을 테스트 우선으로 만드는 거라면 [TDD 모드](tdd-mode.md)입니다. characterize 는 *바뀌어야 할 동작* 이 아니라 *지금 그대로의 동작* 을 잡습니다.

## 전제

- 활성 프로젝트가 있을 것
- 동작을 포착할 대상 코드가 이미 존재할 것

## 절차

### 1. 모드 켜기

```text
/dp-skills:characterize on
```

`.agent-state.yml` 의 `mode: characterize` 플래그를 토글하고, 구현 루트(`{source_root}` — 예: `app/`·`src/main/`) 수정을 잠급니다.

### 2. 사이클 진행

평소처럼 `@ag-planner → @ag-generator → @ag-evaluator` 를 호출하되, 각 에이전트 동작이 바뀝니다.

| 에이전트 | 기본/TDD | Characterize 모드 |
|---|---|---|
| `@ag-planner` | Red 계약 (바뀌어야 할 동작) | **Characterization Contract** (현재 관찰된 입출력) |
| `@ag-generator` | 구현 수정 | **`{source_root}` 잠금** — spec 만 추가, 구현 변경 금지 |
| `@ag-evaluator` | 변경 관련 테스트 실행 | spec 이 현재 동작을 정확히 포착하는지 검증, 구현 수정 감지 시 반려 |

### 3. 모드 끄기

포착이 끝나면 복귀합니다.

```text
/dp-skills:characterize off
```

리팩터는 모드를 끈 **별도 사이클** 에서 진행합니다.

## 흔한 실수

- **characterize 사이클에서 구현을 고치지 않습니다.** generator 가 `{source_root}` 를 수정하면 evaluator 가 반려합니다. spec 추가와 리팩터는 분리된 사이클입니다.
- `tdd: true` 와 동시에 설정되면 characterize 가 우선합니다 — Red 계약 대신 Characterization Contract 가 적용됩니다.

## 다음 단계

- How-to: [TDD 모드 활성화](tdd-mode.md) — 포착 후 리팩터 사이클
- Reference: [`/dp-skills:characterize`](../reference/skills/characterize.md)
