# `/dp-skills:run-cycle`

> 활성 project 의 미완료 feature 들을 순서대로 자동 순회하며 planner→critic→generator→evaluator 사이클을 진행하는 순차 오토파일럿. green 신호(critic 합의·evaluator READY)에는 자동 진행하고, plan 승인· critic blocking·NOT_READY·block 에서만 일시정지한다. 큐는 project.md 의 [ ] 미완료 feature 에서 파생하며 별도 상태 파일을 쓰지 않는다 (stateless, 재실행=재개). 병렬 아님 — 한 번에 1 feature. 수동 @mention 사이클은 불변.

활성 project 의 development phase 에서 미완료 feature 들을 순차 자동 처리하는
오케스트레이터. 메인 루프에서 `@ag-planner`·`@ag-planner-critic`·
`@ag-generator`·`@ag-evaluator` 를 순서대로 호출한다 (subagent 가 subagent 를
띄우지 않음). 기존 수동 사이클은 그대로 — 본 스킬은 선택적 순차 러너.

대상: $ARGUMENTS  (옵션: feature 부분집합 `#3-5` · `#3 #4`)

## 사전 확인

[preamble.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/preamble.md) **P-1**(TodoWrite 선로딩) · **P1**(팀) ·
**P2**(활성 project) 수행. 활성 project 없으면 P2 메시지로 종료. `.agent-state.yml`
의 `phase` 가 `qa` 면 "run-cycle 은 development phase 전용" 안내 후 종료.

## 1. 큐 구성 (project.md 파생)

1. 활성 project 의 `project.md` `## 목표` 에서 `[ ]` 미완료 feature 를 **목록
   순서대로** 수집 (`[x]` 제외). 의존성 메타데이터가 없으므로 순서를
   재계산하지 않고 목록 순서를 신뢰한다.
2. 인자로 `#3-5`·`#3 #4` 명시 시 그 부분집합으로 필터.
3. 큐가 비면 → `"신규 작업 없음 (모든 목표 완료)"` 출력 후 종료.
4. 큐 표시 + 확인: `"큐: #3 {slug} → #4 {slug} → #5 {slug} (순서대로). 진행할까요?"`
   — 사용자 승인 후 시작.

## 2. feature 1건 사이클 루프

큐의 각 feature 를 순서대로 처리한다.

### 2.1 재개 지점 판정 (stateless)

feature 산출물로 "가장 이른 미완료 단계" 를 정한다:

- `features/NN-{slug}.plan.md` 없음 → **planner 부터**
- 유효한 `.plan.md` 있음 (`tools/plan-validate.py` 통과) → **critic/generator 부터**
- `features/NN-{slug}.eval.md` 가 `NOT_READY` → **그 지점부터**

완료(`[x]`) feature 는 큐에 없으므로 도달하지 않는다.

### 2.2 planner

`@ag-planner` 호출 → `.plan.md` 산출.
**⏸ plan 승인 게이트**: 산출된 plan 을 사용자에게 제시하고 승인을 기다린다.
승인 전 다음 단계 진행 금지 (feature 마다 1회). 미승인·거부 → §3 정지.

### 2.3 critic

`@ag-planner-critic` 호출 (기본 실행) → `.plan.critic.md` 의 합의 표 생성.

판정은 합의 표를 읽어 결정한다 (열: `severity`=blocking/suggestion/nit · `status`=`open`/`accepted`/`rejected`/`deferred`. critic 이 행마다 `severity` + `status=open` 으로 emit — ag-planner-critic 절차 5):

- **blocking 미해결 행** (`severity`=blocking 이고 `status` ∈ {`open`, `deferred`}) 이 1개라도 있으면 → **⏸ §3 정지** (critic 규칙 — "blocking 미해소 시 generator 진행 금지". `deferred`=이월 도 미해결이므로 정지 — `accepted`·`rejected` 만 처리 완료로 본다).
- blocking 행이 없거나 모든 blocking 행이 `accepted`·`rejected` 로 처리됨(정지 대상 0) → **합의 표를 수정하지 않고** generator 진행. suggestion·nit 은 status 무관 비정지. 합의 표는 critic·planner 소유라 run-cycle 은 읽기만 한다 — `status` 전이는 planner (절차 10) 만 수행 (ag-planner-critic 절차 5).

### 2.4 generator

`@ag-generator` 호출 → 구현.

### 2.5 evaluator

`@ag-evaluator` 호출 → VERIFICATION REPORT.

- `READY` → `project.md` 해당 feature 체크박스 `[x]` 갱신. REPORT 의
  `metrics.domain_impact` 가 `detected` 면 항목을 **메모리에 누적**하고
  (질의·정지 없음 — green 자동 진행 유지) → **자동 다음 feature**(2.1 로).
- `NOT_READY`·`block` → **⏸ §3 정지**.

## 3. 정지 정책

문제 게이트(plan 미승인·critic blocking·NOT_READY·block) 도달 시:

- **큐 전체 정지.** 뒤 feature 로 진행하지 않는다 — N+1 이 N 에 의존할 수
  있고 의존성 데이터가 없어 알 길이 없다.
- 정지 사유·해당 REPORT 출력.
- 안내: `/dp-skills:fix-review` 로 처리 → 수정 후 **`/dp-skills:run-cycle`
  재실행(=재개)**. 부분집합으로 돌렸다면 같은 인자를 다시 넘긴다.
- 누적된 domain_impact 항목이 있으면 정지 보고에 "도메인 지식 환류 제안
  (누적)" 표를 함께 출력한다 — 처리 여부는 사용자 결정
  ([knowledge-sync.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/knowledge-sync.md) § run-cycle).

## 4. 큐 완료

모든 feature READY → 완료 요약 출력 (처리한 feature 목록·각 verdict).
누적된 domain_impact 항목이 있으면 "도메인 지식 환류 제안 (누적)" 표를
출력하고 기록 여부를 질의한다. 승인 시
[knowledge-sync.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/knowledge-sync.md) § 기록 을
수행한다 (항목 5건 이상이면 `/dp-skills:discover` 재실행 권장).

## Do-NOT

- feature 병렬 진행 금지 — 한 번에 1 feature (`.focus.md`·`STATE.md` 싱글톤·
  generator 파일 충돌 회피).
- plan 승인·NOT_READY·critic blocking 자동 통과 금지 — 반드시 정지.
- 정지 후 뒤 feature 자동 진행 금지.
- 상태 파일 생성 금지 — 큐·진행은 `project.md` + 산출물에서 파생.
- 수동 사이클·`@ag-*` 동작 변경 금지 — 본 스킬은 호출 순서만 자동화.
- feature 마다 domain_impact 질의로 정지 금지 — 누적 후 큐 종료 시 일괄 질의.
