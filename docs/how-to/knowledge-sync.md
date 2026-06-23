# 도메인 지식 환류 (knowledge-sync)

!!! info "한 줄 요약"
    evaluator 가 사이클 끝에서 "이번 변경이 팀 도메인 지식 문서에 남길 게 있나" 를 판정해, 있으면 **환류 제안** 을 띄웁니다. 승인하면 `workspace/{TEAM}/context/` 의 scope 문서에 반영됩니다.

## 언제 쓰나

- `@ag-evaluator` 의 VERIFICATION REPORT 끝에 **"도메인 지식 환류 제안"** 표가 떴을 때
- 이번 사이클이 라우트·모델·enum·외부 의존·도메인 용어를 건드렸고, 그 사실을 다음 사이클의 planner 도 알아야 할 때

코드와 문서가 어긋난 걸 *우연히* 발견하는 drift 대응이 "기존 문서가 코드와 다름" 을 다룬다면, 환류는 "이번 변경이 만든 *새* 지식을 사이클 종료 시 체계적으로 점검" 하는 반대 방향입니다.

## 전제

- `@ag-evaluator` 사이클이 끝났을 것 (`status: READY`)
- 도메인이 판정됐을 것 (`domain: null` 이면 환류 판정을 건너뜁니다)

## evaluator 의 판정 — detected / none / skip

evaluator 는 심사를 마친 뒤 이번 변경 diff 를 셋 중 하나로 분류합니다.

| 판정 | 기준 |
|---|---|
| **detected** | 라우트/엔드포인트, 도메인 모델·컬럼, enum·상태값·상수, 외부 의존(API·MQ·타 도메인), 비즈니스 용어 신설, 또는 scope 에 이미 기록된 항목의 동작 변경 |
| **none** | 내부 리팩터(공개 표면 불변)·버그 수정·테스트만 변경·스타일/주석 |
| **skip** | `domain: null` (판정 기준 도메인 없음) |

`none` 판정 휴리스틱은 **"이 변경을 모르는 다음 planner 가 잘못된 계획을 세울 수 있는가"** — 아니면 none 입니다.

!!! warning "gate 가 아닙니다"
    환류는 통과/반려 게이트가 **아닙니다** — `status: READY` 와 `detected` 는 정상 공존합니다. REPORT 의 `metrics.domain_impact` 에 기록될 뿐, 통과 판정에 영향을 주지 않습니다.

## 절차

### 1. 제안 표 확인

`status: READY` + `detected` 면 evaluator 가 REPORT 직후 표를 띄웁니다.

```text
## 도메인 지식 환류 제안
| # | 유형 | 내용 | 대상 문서 |
|---|---|---|---|
| 1 | enum | 발송상태 CANCELED 추가 | scope/retail.md |
기록할까요?
```

### 2. 승인 → 반영

승인하면 메인 루프가 항목별 before/after 미리보기를 제시하고, 최종 확인 후 `context/` 문서에 반영합니다 (scope 200줄 제약·MANIFEST 갱신 규약을 따름). **evaluator 가 직접 쓰지 않습니다** — 기록 주체는 사용자 승인을 받은 메인 루프입니다.

### 3. 거부 → 미기록

거부하면 기록하지 않습니다. 미기록 지식은 이후 사이클에서 drift 로 발견될 수 있습니다 (의도된 안전망).

## run-cycle 에서는

[`/dp-skills:run-cycle`](run-cycle.md) 오토파일럿은 feature 마다 멈추지 않습니다 — detected 항목을 메모리에 누적했다가 **큐 종료 시점**(완료 요약 또는 정지 보고)에 누적 표로 한 번에 묻습니다. `NOT_READY` 인 동안에는 질의하지 않고 REPORT 에 기록만 해 두었다가, 재작업으로 `READY` 가 되는 재평가에서 묻습니다.

## 흔한 실수

- **환류를 반려로 오해하지 마세요.** detected 가 떠도 사이클은 이미 READY 입니다 — 기록은 선택입니다.
- **`rules/` 는 환류 대상이 아닙니다.** 정책(`rules/{domain}.md`)은 코드에서 추론할 수 없습니다. 구현과 rules 의 상충은 drift 대응 경로로 처리합니다.
- **항목이 5건 이상이거나 구조를 개편해야 하면** 개별 기록 대신 [`/dp-skills:discover`](domain-discover.md) 재실행이 더 안전합니다 (scope·MANIFEST 단순 덮어쓰기).

## 다음 단계

- How-to: [도메인 스코프 추출](domain-discover.md) · [미완료 feature 순차 자동 진행](run-cycle.md)
- Explanation: [에이전트 흐름](../explanation/agent-flow.md)
- Reference: [`@ag-evaluator`](../reference/agents/ag-evaluator.md)
