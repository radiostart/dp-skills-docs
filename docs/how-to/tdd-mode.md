# TDD 모드 활성화

!!! info "한 줄 요약"
    프로젝트에 Red→Green→Refactor 사이클을 적용합니다. 각 에이전트의 책임이 테스트 우선으로 확장됩니다.

## 언제 쓰나

- 신규 기능을 **테스트 우선**으로 구현하고 싶을 때
- 이미 진행 중인 프로젝트에 **TDD 체계를 사후 도입**할 때

레거시 코드의 *현재 동작* 을 포착하려는 거라면 TDD 가 아니라 [Characterize 모드](characterize-mode.md)입니다.

## 전제

- `workspace/_common/config.md` 의 `## 언어·도구 기본값` 에 `test_command` 가 채워져 있을 것 — 비어 있으면 TDD 사이클이 시작되지 않고 등록 안내 후 중단됩니다.

## 절차

### 신규 프로젝트로 시작하는 경우

```text
/dp-skills:project MyFeature --tdd
```

### 이미 있는 프로젝트에 사후 적용하는 경우

```text
/dp-skills:tdd
```

`project.md` 의 제한사항·에이전트 호출 흐름과 `agents/*.md`(planner/generator/evaluator), `.agent-state.yml` 의 `tdd: true` 플래그를 갱신합니다. idempotent 입니다.

### 에이전트별 동작 변화

`tdd: true` 가 설정되면 각 에이전트 책임이 확장됩니다.

| 에이전트 | 기본 | TDD 모드 추가 |
|---|---|---|
| `@ag-planner` | 구현 계획 | + 스텝 분할 + 실패 테스트 (Red) |
| `@ag-generator` | 코드 구현 | + 실패 테스트를 통과시키는 최소 구현 (Green) |
| `@ag-evaluator` | 체크리스트 | + **변경 관련 테스트만** `{test_command} {paths}` 실행 |

## 흔한 실수

- **evaluator 가 전체 테스트 스위트를 돌리지 않습니다.** 인자 없는 전체 실행은 금지 — 변경 관련 테스트만 경로를 지정해 실행합니다.
- `test_command` 가 비어 있으면 사이클이 아예 시작되지 않습니다. `config.md` 에 `bundle exec rspec` / `./gradlew test --tests` / `pnpm test` 같은 값을 먼저 등록하세요.
- `tdd: true` 와 `mode: characterize` 가 동시에 설정되면 **characterize 가 우선**합니다.

## 다음 단계

- How-to: [Characterize 모드](characterize-mode.md) — 레거시 동작 포착 사이클
- Reference: [`/dp-skills:tdd`](../reference/skills/tdd.md) · [`/dp-skills:project`](../reference/skills/project.md)
