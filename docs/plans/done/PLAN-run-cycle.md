# Plan — 순차 사이클 오토파일럿 스킬 `/dp-skills:run-cycle`

> status: approved
> created: 2026-05-29
> updated: 2026-05-29 — 설계 승인 (이름·게이트정책·큐·실패처리·plan승인 게이트 결정)
> author: jpseo@jpblog.co.kr
> 관련 메타: project lifecycle · planner→generator→evaluator 사이클 · 게이트 자동화

## 0. 운영 전제

- **순차 전용 — 병렬 아님.** feature 를 한 번에 1건씩 사이클 처리한다. 병렬을
  배제하는 것이 `.focus.md`·`STATE.md` 싱글톤 충돌과 generator 파일 충돌
  위험을 원천 제거하는 핵심 결정이다 ([PLAN 검토 근거](#9-배경-병렬을-배제한-이유)).
- **기존 수동 사이클은 그대로.** `@ag-planner`·`@ag-generator` 등 직접 멘션
  흐름은 불변. `run-cycle` 은 그 위에 얹는 **선택적 순차 러너** — 메인 루프
  오케스트레이터로 `@ag-*` 를 순서대로 호출한다 (`dp-code-review` 와 동일
  구조, subagent 가 subagent 를 띄우지 않음).
- **활성 project 1개의 development phase 전용.** QA phase·다중 project 는
  비대상 (STATE.md "진행중 1행" 싱글톤 존중).
- **plan 검토는 없애지 않는다.** "오토" 의 의미는 critic·generator·evaluator·
  다음 feature 이행을 자동화하는 것 — plan 승인 게이트는 feature 마다 유지한다.
- **상태 비저장 (stateless).** 큐·진행 위치·재개 지점을 별도 파일에 쓰지 않고
  매 호출마다 `project.md` 체크박스 + feature 산출물(`.plan.md`/`.eval.md`)에서
  파생한다. 상태 SSOT 이중화를 피한다 (doctor 철학과 일치).

## 1. 목표·범위

`project.md`·`## 목표`·의 미완료 feature 들을 순서대로 자동 순회하며,
각 feature 에 planner→critic→generator→evaluator 사이클을 돌린다. green
신호(critic 합의·evaluator READY)에는 자동 진행하고, 문제 게이트(plan 승인·
critic blocking·NOT_READY·block)에서만 일시정지한다. 수동 @mention 체이닝의
반복 수고를 제거하되 인간 판단 지점은 보존한다.

### in scope

- 새 스킬 `dp-skills/skills/run-cycle/SKILL.md`
- feature 부분집합 인자 (`#3-5`·`#3 #4`) · 정지 후 재개는 **재실행**(stateless)
- README · plugin.json 버전 bump · docs 색인

### out of scope

- feature 병렬 진행 (별도 PLAN — 정형 의존성 메타데이터·worktree 격리 선결)
- 수동 사이클 대체·폐기 (공존)
- QA phase 자동화 (development phase 만)

## 2. 큐 구성

1. `project.md`·`## 목표`·섹션에서 `[ ]` **미완료 feature 만** 목록 순서대로
   수집 (`[x]` 완료분 제외). PM·analyze 가 의도한 목록 순서를 신뢰 — 의존성
   메타데이터가 없어 시스템이 순서를 재계산하지 않는다.
2. 인자로 `#3-5`·`#3 #4` 명시 시 그 부분집합만.
3. **큐가 비면** (`[ ]` 0개) → `"신규 작업 없음 (모든 목표 완료)"` 출력 후 종료.
4. **시작 전 큐 확인**: `"큐: #3→#4→#5 (순서대로). 진행할까요?"` — 사용자
   승인 후 시작.

> **재실행 의미론**: 직전 사이클로 완료된 feature 는 evaluator READY 시
> `[x]` 가 되어 다음 `run-cycle` 에서 자동 제외된다. `create-feature`·
> `analyze` 로 신규 feature 가 추가되면 그것들이 `[ ]` 이므로 다음
> `run-cycle` 이 **신규분만** 큐에 담는다.

## 3. feature 1건 사이클 (Balanced 게이트)

각 feature 에 대해:

```
planner 호출 → plan 산출 (.plan.md)
  └ ⏸ plan 승인 게이트 (사용자 검문 — 코드 쓰기 전, feature 마다 1회)
critic 호출 (@ag-planner-critic, 기본 실행)
  └ blocking 챌린지 有 → ⏸ 정지 / nit·suggestion 만 → 자동 반영 후 진행
generator 호출 → 구현
evaluator 호출 → VERIFICATION REPORT
  └ READY     → ✅ project.md 체크박스 [x] 갱신 + 자동 다음 feature
  └ NOT_READY / block → ⏸ 큐 전체 정지 (§4)
```

- `⏸` 지점이 일시정지의 전부. critic green·evaluator READY 는 자동 진행.
- **plan 승인**: feature 마다 1회 정지 (각 feature plan 이 다르므로 큐 일괄
  승인보다 안전). 승인 전 generator 진행 금지.
- **critic 분기**: 기본 실행. critic verdict 에 미해결 blocking 챌린지가
  있으면 정지, nit/suggestion 만이면 자동 반영(`.plan.critic.md` 합의 표
  `[x]` 처리) 후 generator 진행.

> **mid-cycle 재개**: 이미 산출물이 있는 `[ ]` feature (직전 paused 등) 는
> planner 부터 재실행하지 않고 **가장 이른 미완료 단계부터** 이어간다 —
> 유효한 `.plan.md` 있으면 critic/generator 부터, NOT_READY `.eval.md` 면
> 그 지점부터. 기존 작업을 덮지 않는다.

## 4. 실패 시 — 큐 정지 정책

- 문제 게이트(plan 미승인·critic blocking·NOT_READY·block) → **큐 전체 정지
  + REPORT/사유 제시.** 뒤 feature 로 자동 진행하지 않는다 — N+1 이 N 에
  의존할 수 있고, 의존성 데이터가 없어 알 길이 없다.
- 사용자가 `/dp-skills:fix-review` → 수정 → **`/dp-skills:run-cycle` 재실행**.
  stateless 이므로 재실행이 곧 재개 — project.md + 산출물에서 첫 미완료
  feature 를 그 단계부터 다시 집어 이어간다 (§5). 부분집합으로 돌렸다면
  같은 인자(`#4-5`)를 다시 넘긴다.

## 5. 상태·재개 — stateless 파생

별도 상태 파일·스키마 변경 **없음.** 큐와 재개 지점을 매 호출마다 기존
산출물에서 파생한다 (상태 SSOT 이중화 회피):

- **큐**: `project.md`·`## 목표`·의 `[ ]` 미완료 feature (목록 순서).
- **재개 지점**: 각 feature 의 산출물로 "가장 이른 미완료 단계" 판정 —
  `.plan.md` 없음 → planner 부터 / 유효 `.plan.md` 있음 → critic/generator 부터 /
  `.eval.md` NOT_READY → 그 지점부터. `[x]` 완료분은 스킵.
- **재개 = 재실행**: 정지 후 `run-cycle` 을 다시 부르면 위 파생으로 첫 미완료
  feature 를 이어간다. paused 표식을 저장하지 않으므로 별도 `--resume` 플래그
  불필요. plan 승인은 세션 내 인터랙티브 정지일 뿐 invocation 간 영속 안 함 —
  재실행 시 해당 plan 게이트를 다시 거친다 (무해·안전).
- **순차이므로 한 번에 1 feature 만 활성** → `.focus.md` 싱글톤 충돌 없음.

## 6. TDD 모드

별도 처리 없음. TDD 프로젝트면 evaluator 의 `tdd_evidence` 게이트가 그대로
작동 — Red-Green 증거 누락 → NOT_READY → §4 정지 정책으로 흡수. autopilot 은
게이트 판정을 신뢰만 한다.

## 7. Slack

기존 `tools/slack-notify.py` 의 approval(plan 게이트 도달·정지 시)·
complete(큐 완료 시) 이벤트 발화. 설정돼 있으면 일시정지·완료 알림이 채널로.

## 8. 변경 파일

| 파일 | 작업 |
|---|---|
| `dp-skills/skills/run-cycle/SKILL.md` | **신규** — 큐 파생·사이클 루프·게이트 정책·stateless 재개 |
| `dp-skills/README.md` | 스킬 19→20종 + 표 행 + (해당 시) 흐름 안내 |
| `dp-skills/.claude-plugin/plugin.json` | version → 0.15.0 |
| docs reference 색인 | `python3 dp-skills/tools/docs_build.py` 재생성 |
| `dp-skills/docs/how-to/run-cycle.md` | 사용 레시피 (선택) |

> **상태 스키마 변경 없음** — `.agent-state.yml`·`state-schema.md` 무수정.
> 큐·진행 상태는 project.md + 산출물에서 파생 (§5).

## 9. 배경 — 병렬을 배제한 이유

`.focus.md`·`STATE.md` 가 싱글톤이고, generator 가 같은 소스 트리에 동시
Edit 시 충돌하며, feature 간 의존성을 자동 판정할 정형 메타데이터(`changed_files`·
`depends_on`)가 없다. 따라서 병렬 다중 feature 는 선결 과제(메타데이터·worktree
격리·per-feature focus·스케줄러) 없이는 위험. 순차 러너는 이 위험을 전부
회피하면서 "수동 체이닝 제거" 가치를 동일하게 제공한다.

## 10. 미해결 질문

- critic blocking 판정 기준 — verdict severity 매핑을 critic 출력 스키마와
  정확히 어떻게 연동할지 (구현 시 `ag-planner-critic.md` 출력 형식 확인).
- "유효한 `.plan.md`" 판정 — `plan-validate.py` 통과 여부로 mid-cycle 재개
  단계를 가를지 (구현 시 확정).
- 큐 확인 단계에서 feature 별 예상 의존성을 사람에게 보여줄지 — 초판은 단순
  목록만, 운영 후 검토.
