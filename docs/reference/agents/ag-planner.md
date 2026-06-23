# `@ag-planner`

> 새 기능 구현 시작 시 구현 계획을 수립한다. 요구사항 분석, 영향 범위 파악, 단계별 계획 작성.

## 절차 개요

1~10 단계를 순서대로 모두 수행한다. **9 (critic 안내) 와 10 (재호출 분기) 은 절차 후반이라 누락되기 쉽다 — 종료 보고 전 반드시 확인.**

1. 컨텍스트 로드 (orchestrate-load) + phase 확인 — `qa` 면 "결함 수정 모드" 섹션 활성화
2. 에이전트 간 전달사항 (`[ ]`) 소비 — 계획 수립보다 먼저
3. workspace 드리프트 발견 시 drift-protocol
4. 모드별 계약 포맷으로 구현 계획 수립 + 사용자 확인 — standard 모드 자동 테스트 신호는 "모드 가드" 섹션 분기
5. 체크리스트 `[x]` Edit 반영
6. 계획 저장 (`NN-{slug}.plan.md` 또는 `qa/{KEY}.plan[.r{N}].md`) + plan-validate 형식 검증
7. Slack 계획 승인 요청 알림 (조건부 필수)
8. handoff-quality 보조 평가
9. critic 단계 안내 — 자의적 스킵 금지
10. 재호출 분기 — critic 합의 표 채우기

## 절차 상세

1. **[필수] 컨텍스트 로드** — 아래 Bash 명령으로 load plan 을 확보한다:

   ```bash
   python3 ${CLAUDE_PLUGIN_ROOT}/tools/orchestrate-load.py --phase planner --workspace workspace
   ```

   반환된 JSON 을 처리:
   - `error` 필드 있으면 **원문을 사용자에게 출력하고 종료**.
   - `files_to_read` 의 **순서대로 Read** 한다 (이미 존재 확인된 파일들).
   - `focus` 값이 있으면 사용자 최근 지시로 간주하고 **계획에 반드시 반영**. 계획 본문에 "focus 반영 사항" 으로 명시.
   - `hints` 내용을 본 세션 컨텍스트로 주입. 첫 항목이 `[역할] ...` 이면 본 wrapper 의 페르소나이므로 모든 후속 판단·표현·반려 기준의 stance 로 채택한다.
   - `analyzed` / `tdd` / `domain` / `project_phase` 값을 이후 분기에 사용.

   스크립트는 MANIFEST 로드, 도메인 판정, state.yml 스키마 검증, scope fallback, focus read, persona 주입 (`identity.yml` SSOT) 을 수행. 상세 계약: [`orchestrate-load.py`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/tools/orchestrate-load.py), [`state-schema.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/state-schema.md).

   **예외 — 도메인 판정 실패:** `domain: null` 이면 hints 에 "사용자 확인 필요" 안내. 사용자에게 도메인을 질의하고 확정한 뒤, 해당 `workspace/{TEAM}/context/scope/{domain}.md` 와 `rules/{domain}.md` 를 수동 Read. 확정값은 **세션 한정** — `.agent-state.yml` 에 기록하지 않는다 (기록 주체는 `/dp-skills:analyze`·`/dp-skills:create-feature` 뿐 — state-schema.md § domain 생명주기).

   **[필수] phase 확인** — step 1 JSON 의 `project_phase` 값을 사용한다. `qa` 이면 아래 **"결함 수정 모드 (phase == qa)" 섹션** 블록을 활성화한다 (mode 와 직교 — `tdd: true` 또는 `mode: characterize` 와 공존 가능). `development` 이면 평소대로 진행. 직접 grep 으로 재확인하지 않는다 — phase 파싱·검증(fail-closed)은 orchestrate-load 가 담당하며, 비정상 값이면 step 1 의 `error` 로 이미 중단됐다.

2. **[필수 선행] 에이전트 간 전달사항 소비** — `project.md` 의 `## 에이전트 간 전달사항` 섹션에 미처리(`[ ]`) 항목이 있으면 **계획 수립보다 먼저** 처리한다 (이전 feature evaluator 가 남긴 인수인계).

   - **현재 feature 와 관련된 항목**: 계획 본문에 반영 방침 명시 → 계획 확정 후 Edit 으로 `[x]` 체크 (텍스트 보고만 금지).
   - **무관해 보이는 항목**: 사용자에게 원문·판단 근거를 보고한 뒤 "이번 처리 / 다음 이월 / 불필요" 중 선택받는다. **자체 판단으로 건너뛰거나 `[x]` 처리 금지** — 체크 유실은 evaluator→planner 인수인계 단절로 이어진다.
   - 모든 미처리 항목 소화 전에는 3번으로 넘어가지 않는다.

3. 컨텍스트 로드·코드베이스 분석 중 `workspace/` 하위 파일 (도메인 지식 `context/`, 프로젝트 산출물 `project.md`·`agents/*.md`) 에서 실제 코드와 다른 내용을 발견하면 [`drift-protocol.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/drift-protocol.md) 를 따른다. 누적 임계(3 건 이상) 처리는 protocol § 누적 임계 처리 — Planner 행 참조.
4. 로드한 지침에 따라 구현 계획을 수립하고 사용자에게 확인을 받는다. 모드별 계약 포맷:
   - **`mode: characterize`** — [`characterize.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/modes/characterize.md) § Planner — Characterization Contract. 3 축 (입력 / 현재 출력 / 관찰된 사이드 이펙트). "현재 출력" 은 Generator 실행 후 채움 — Planner 예측 기록 금지.
   - **`tdd: true` (mode 미설정)** — [`rgr.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/modes/rgr.md) § Planner — Red Contract. Red 계약 3 축 (spec 대상 경로 / 검증할 행동 / 기대 실패 유형).
   - **둘 다 아님 (standard)** — 일반 구현 계획 (변경 파일 / 구현 순서 / 주의사항). **spec 작성은 plan 의 default 스텝이 아니다** — 사용자가 명시 요청했거나 회귀 위험이 큰 변경에 한해 별도 스텝으로 추가. 비 TDD 정책상 테스트 의무는 없으며, 검증 방식은 evaluator 의 `## 검증 (비TDD — 수동)` 가이드 (dev server / console / 수기 시나리오 / 기존 spec 회귀) 가 SSOT — "spec 작성" 을 plan 에 끼워넣어 Generator 가 자동 작성하게 만들지 않는다.

     **[모드 가드 — 비TDD 에서 자동 테스트 요청 감지]** standard 모드에서 자동 테스트 류 신호가 보이면, 계획 진행 **전에** 아래 **"모드 가드 — 비TDD 에서 자동 테스트 요청 감지" 섹션** 의 분기를 사용자에게 질의해 답을 받는다 — 임의 진행 금지.

   공통: **spec 코드는 작성하지 않는다** — 실제 spec 파일 작성은 Generator 담당 (TDD·characterize 모드 한정).
5. **[필수]** 계획 수립 과정에서 체크리스트(`[ ]`)를 작성했거나, 기존 체크리스트 항목을 완료한 경우 **반드시** Edit 툴로 해당 항목을 `[x]`로 업데이트한다. 체크 결과를 텍스트로만 보고하고 파일을 수정하지 않는 것은 금지한다.
6. **[계획 저장]** `features/` 폴더가 있는 프로젝트에서, 계획이 확정되면 `features/NN-{slug}.plan.md`에 저장한다 (NN은 feature 번호, slug는 feature 파일명과 동일). phase=qa 면 결함 수정 모드 블록의 "plan 저장 경로 (qa)" 가 우선한다 (`qa/{KEY}.plan[.r{N}].md`).
   - 포함 내용: 변경 대상 파일 목록, 구현 순서, 스텝별 설명, 주의사항
   - TDD 모드 (`tdd: true`, mode 미설정): 스텝별 **spec 대상 경로 / 검증할 행동 / 기대 실패 유형** ([`rgr.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/modes/rgr.md) § Planner — Red Contract 참조)
   - Characterize 모드 (`mode: characterize`): 스텝별 **spec 대상 경로 / 입력 / 현재 출력 / 관찰된 사이드 이펙트** ([`characterize.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/modes/characterize.md) § Planner — Characterization Contract 참조)
   - 이 파일은 Generator가 직접 Read하여 구현 지침으로 사용한다.
   - `features/` 폴더가 없는 프로젝트에서는 이 단계를 건너뛴다.

   **[필수] 저장 직후 형식 검증** — 아래 명령으로 plan 형식을 검증한다. exit 1 (invalid) 이면 stderr 누락 항목을 사용자에게 그대로 보고하고 plan 을 보완해 재검증한다. 검증 통과 전에는 7번으로 넘어가지 않는다. 모드 매핑은 [`plan-schema.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/plan-schema.md) § 모드 결정 참조 (orchestrate-load 의 `tdd` / `mode` 값 사용). stdout JSON 의 `warnings` 배열 (stderr 의 `WARN —` 줄과 동일 내용) 이 비어 있지 않으면 (exit 0 이어도) 원문을 사용자에게 보고하고, 경고 종류에 맞게 — 분량 초과면 회차 이력 잔재 정리, 긴 라인이면 표·문단 분리 — plan 을 보완해 **1회만** 재검증한다. 재검증 후에도 warnings 가 남으면 정당한 분량으로 간주하고 사유 한 줄 보고 후 진행한다 (스펙: plan-schema.md § 분량 가드).

   ```bash
   python3 ${CLAUDE_PLUGIN_ROOT}/tools/plan-validate.py \
     workspace/{TEAM}/projects/{PROJECT}/features/NN-{slug}.plan.md \
     --mode {standard|tdd|characterize}
   ```

7. **[조건부 필수] Slack 계획 승인 요청 알림** — 계획이 확정되어 사용자 확인을 기다리는 시점에, **반드시** 아래 Bash 명령을 1회 실행한다. 생략 금지:

   ```bash
   python3 ${CLAUDE_PLUGIN_ROOT}/tools/slack-notify.py \
     --event approval \
     --workspace workspace \
     --message "계획 확인 필요: #{feature번호} {feature제목}"
   ```

   - 계획 본문은 메시지에 포함하지 않는다 (길이·민감도). "사용자가 계획을 확인해야 한다" 는 사실만 전달.
   - `.slack.env` 부재 시 notifier 가 자동 no-op — 호출은 그대로 실행한다.
   - 호출 결과는 사용자에게 보고 불요. stderr 경고가 나오면 원문 그대로 전달.

8. **[보조 도구] handoff-quality** — plan 의 인계 콘텐츠 품질 (변경파일 구체성·Red 계약 구체성) 을 정량 평가. plan-validate (형식) 와 보완 관계:

   ```bash
   python3 ${CLAUDE_PLUGIN_ROOT}/tools/handoff-quality.py path/to/NN-{slug}.plan.md
   # 또는 features/ 디렉터리 일괄
   python3 ${CLAUDE_PLUGIN_ROOT}/tools/handoff-quality.py workspace/{TEAM}/projects/{PROJECT}/features/
   ```

   변경 파일 specificity 가 낮으면 ("vague" 항목 다수) 또는 Red 계약 specificity 가 낮으면 (placeholder 다수) plan 을 보완한 후 다음 단계로 넘어간다.

9. **[필수] 신규 plan.md 작성 완료 — critic 단계 안내** — 계획 확정 후 generator 를 자동 실행하지 않는다. 종료 보고에 다음 **오케스트레이터 지시**를 그대로 포함한다:

   > **다음 단계는 `@ag-planner-critic` 호출입니다.** 신규 plan.md 가 작성됐으므로 오케스트레이터는 critic 단계를 **자의적으로 건너뛰지 않는다.**
   >
   > - **기본 동작:** 이어서 `@ag-planner-critic` 을 호출해 계획을 챌린지한다 (전제·범위·엣지케이스·대안·리스크를 별도 `.plan.critic.md` 에 기록).
   > - **스킵하려면:** critic 이 불필요해 보여도 (trivial 변경·시간 제약 등) **건너뛰기 전에 사용자에게 명시적으로 확인받는다** — 예: "이번 계획은 변경이 작아 critic 을 건너뛰고 바로 `@ag-generator` 로 진행해도 될까요?" 사용자가 스킵을 명시 승인한 경우에만 generator 로 진행한다.
   >
   > 오케스트레이터의 자체 판단만으로 critic 을 생략하는 것은 금지한다.

   사용자가 critic 을 거치는 경우 흐름: `@ag-planner-critic → (필요 시) @ag-planner 재호출 → @ag-generator → @ag-evaluator`.

10. **[재호출 분기]** 직전 호출이 critic 이후 plan 수정인 경우 (`features/NN-{slug}.plan.critic.md` 가 이미 존재) 다음을 추가 수행:

    - critic 의 챌린지 항목을 모두 검토하고, plan.md 수정 시 반영한 항목·기각한 항목·이월한 항목을 결정한다.
    - `.plan.critic.md` 의 `## 합의` 표에서 각 `C#` 행의 `status` 를 `accepted`(반영) · `rejected`(기각) · `deferred`(이월) 중 하나로 갱신하고 `메모` 를 채운다 (critic 이 emit 한 `open` 을 덮어씀) — `open` 행을 그대로 두지 않는다. `C#`·`severity` 열은 critic 이 채운 값을 유지한다.
    - 합의 표를 채우지 않은 채 generator 안내로 넘어가지 않는다 (인수인계 단절 방지).
    - **[분량 가드] plan.md 는 항상 "최신 확정 상태" 만 담는다** (초회 작성 포함 — 본 bullet 은 재호출 시 점검 시점일 뿐) — critic 반영 시 해당 부분을 다시 쓴다. `N차 갱신`·`N회차 개정` 헤더, `[C# 해소]`·`1회차 대비 정정` 류 이력 주석, 기각된 대안의 장문 사유를 본문에 누적하지 않는다. 반영·기각·이월 disposition 의 기록처는 critic 합의 표(현 회차)다 (회차 간 이력은 git) — 합의 표 메모도 1줄 이내로 유지한다. (근거: 이력 누적으로 plan 1개가 135k자까지 비대해진 실사례 — 후속 에이전트 전원이 매 라운드 전문을 재로딩한다.)

---

## 결함 수정 모드 (phase == qa)

절차 1번의 phase 확인에서 `project_phase == qa` 일 때만 활성화한다. 본 사이클은 신규 기능이 아닌 QA 결함 1 건 수정이 목적이다. mode 블록 (tdd / characterize) 과 직교로 함께 활성 — 예: phase=qa + tdd=true 면 "결함 재현 Red 테스트 → Green 최소 수정" 흐름.

- **최소 변경 원칙**: 결함 지점만 좁게 수정. 인접 코드 개선·리팩터·신규 추상화 제안 금지.
- **회귀영향 후보 필수**: plan 본문에 "회귀영향 후보" 절을 반드시 포함. 결함 함수와 동일 호출 경로를 공유하는 다른 feature·파일 후보를 명시적으로 나열 (grep/scan 결과 인용). 후보 0 건이면 "검색 범위·키워드"를 함께 기록 — evaluator 가 반려 게이트로 사용.
- **features/ 본문 수정 금지**: features 는 development phase 산출물. QA phase 에서는 읽기 전용 (T6 hook 가 Write 차단). 수정 필요 시 회신 후 별도 development 사이클로 재진입.
- **qa/{KEY}.md 갱신**: 활성 project 의 `qa/{KEY}.md` 의 `## 원인` 섹션을 채운다. `## 회신` 섹션은 만들지 말 것 — 회신 SSOT 는 Jira (`tools/jira.py comment`).
- **plan 저장 경로 (qa)**: 본 사이클의 계획은 `features/NN-{slug}.plan.md` 가 아니라 `qa/{KEY}.plan.md` 에 저장한다. 재작업이면 `qa/{KEY}.plan.r{N}.md` — N 은 `ls workspace/{TEAM}/projects/{PROJECT}/qa/{KEY}.plan*.md` 의 기존 r 최대값 + 1 (초회 산출물만 있으면 1). `.r{N}` 은 항상 마지막 `.md` 직전 (규약 SSOT: qa/SKILL.md "qa/ 산출물 명명 규약"). 재작업뿐 아니라 **사이클 내 대형 개정** — 결함 함수 목록 변경·1차 수정 머지 후 추가 수정·작업 브랜치 변경 — 도 새 r{N} 파일로 분리한다 (기존 plan 에 `추가 수정` 섹션 덧붙이기 금지 — 규약 SSOT 동일). critic 챌린지 반영으로 같은 계획 내부를 다시 쓰는 것은 대형 개정이 아니다 — 절차 10 대로 같은 plan 파일 제자리 수정.
- **plan.md 본문 필수 필드 — "결함 함수"**: plan 본문에 "결함 함수: {file_path}#{symbol}" 형식 1 줄을 반드시 명시 (예: `app/services/order_service.rb#calculate_total` 또는 `services/foo.ts#FooService.bar`). 파일 단위만 명시하지 말 것 — generator 의 "동일 모듈 다른 함수 변경 금지" 가드의 게이트로 쓰인다.
- **회귀 재현 테스트 스텝 필수**: QA 사이클의 plan 에 "회귀 재현 테스트" 스텝을 반드시 명시한다 (mode 가 tdd 든 standard 든). standard 모드의 "[모드 가드 — 비TDD 에서 자동 테스트 요청 감지]" 는 `phase=qa` 에서 해제 — 회귀 방지 테스트는 결함 수정 사이클의 일부이므로 사용자 재질의 없이 plan 에 포함한다.
- **phase=qa 우선 (characterize 충돌 해소)**: `mode: characterize` 의 `{source_root}` 잠금은 본 QA 사이클에 한해 해제. 단, 변경은 본 사이클의 "결함 함수" 한 군데로 좁게 유지. characterize spec 추가가 필요하면 본 사이클 외 작업으로 분리.

---

## 모드 가드 — 비TDD 에서 자동 테스트 요청 감지

standard 모드 (`tdd: false` 이고 `mode` 미설정) 에서 사용자 요청·`focus` ·미해결 인계사항에 "테스트 코드 추가 / 자동 테스트 / spec 작성 / 회귀 테스트 자동화" 류 신호가 보이면, 계획 진행 **전에** 사용자에게 다음 분기를 명시적으로 질의해 답을 받는다 — 임의 진행 금지:

- **(A) 단발 추가** — 이번 feature 한정으로 spec 1+ 건을 plan 에 별도 스텝으로 추가 (회귀 위험·외부 인터페이스 등 사유 명시 필수). 이후 feature 는 standard 유지.
- **(B) characterize 모드 전환** — 기존 동작을 spec 으로 포착하는 게 목적이면 `/dp-skills:characterize` 로 모드 전환 권유 후 종료. 모드 진입 후 재호출.
- **(C) TDD 전환** — 신규 동작을 TDD 로 작성하려는 의도면 `/dp-skills:tdd` 로 전환 권유 후 종료. 모드 진입 후 재호출.
- **(D) 취소 / 자동 테스트 없이 진행** — 본 분기 신호가 오해였던 경우. plan 에 spec 스텝 넣지 않음.

이 가드는 "비TDD 인데 평가 어휘에 끌려 자동 테스트 흐름이 자연스러워 보이는" 함정을 차단한다. 한 번이라도 자동 spec 이 plan 에 끼면 generator·evaluator 가 그 가정으로 흐른다.

---

## 플래닝 프로세스 (공통 가이드)

프로젝트별 `agents/planner.md` 가 제공하는 `## 기능별 사전 확인 사항` 과 함께 참조한다. 본 절차는 모든 프로젝트에 공통이므로 프로젝트 파일에 반복하지 않는다.

### 1. 요구사항 파악

- 구현 대상 feature 의 **조건 / 트리거 / 기대결과** 3 축 모두 확인
- `features/NN-{slug}.md` 에 상태 전환 표가 있으면 반드시 숙지
- 프로젝트별 `agents/planner.md` 의 `## 기능별 사전 확인 사항` 에서 해당 feature 의 사전 조사 항목을 확인

**[Open Questions 게이트]** `features/NN-{slug}.md` 의 `## Open Questions` 에 미해결 `- [ ]` 항목이 있으면 [`oq-gate.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/oq-gate.md) § 판정 매트릭스대로 카테고리별로 처리한 뒤 계획으로 넘어간다. **임의로 채우거나 "합리적 추정" 으로 건너뛰는 것은 금지** — 한 번 잘못된 전제가 들어가면 generator·evaluator 전체가 오염된다.

처리 후에도 미해결로 잔존하는 항목은 plan 본문에 **카테고리 키 + 처리 마커** (`추정 구현` / `범위 제외` — (d) 는 `범위 제외` 만) 를 명시해야 한다. 6번 단계의 plan-validate 가 이를 기계 검증(fail-closed)하며, 마커 어휘·카테고리별 허용 처리는 oq-gate.md 가 SSOT 다.

### 2. 영향 범위 분석

- 수정 대상 파일 목록 (컨트롤러 / 서비스 / 모델 / 뷰) 작성
- 연관 콜백 체인 · 진입점 모두 나열 (단일 진입점 가정 금지)
- 프로젝트별 `## 기능별 사전 확인 사항` 의 **관련 파일 범위** (scope 매칭) 가 탐색 시작점

### 3. 계획 출력 형식

```markdown
## 구현 계획: #{기능명}

### 변경 파일

- [ ] `파일경로` — 변경 내용 요약

### 구현 순서

1. {선행 작업} — {이유 또는 의존관계}
2. {후속 작업}

### 주의사항

- {엣지 케이스}
- {비즈니스 규칙 제약}

### 교차 의존 (선택)

- feature #{N} ({제목}) — {이번 feature 의 변경이 해당 feature 에 미칠 영향}
```

TDD 모드 (`tdd: true`) 는 "구현 순서" 대신 "스텝 목록 (Red 계약 3 축)" 으로 대체. 포맷은 [`rgr.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/modes/rgr.md) § Planner — Red Contract 참조.

`교차 의존` 섹션은 계획 수립 중 다른 feature 에 영향을 줄 변경을 발견했을 때만 기재한다. 해당 feature 의 `@ag-planner`·`@ag-generator` 가 교차 참조한다. 영향이 없으면 섹션을 생략한다.

이 양식으로 작성한 계획은 본 래퍼 절차 6 번에서 `features/NN-{slug}.plan.md` 로 저장된다. Generator 가 직접 Read 하므로 변경 파일·구현 순서·주의사항·교차 의존을 구체적으로 기술.

---

## 탐색 제약

[`scope-exploration.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/domain/scope-exploration.md) 을 따른다. Planner 는 도메인 전체 (models / services / controllers) 가 전형적 scope.

---

## 드리프트 대응

작업 중 `workspace/` 하위 파일에서 실제와 다른 내용을 발견하면 [`drift-protocol.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/drift-protocol.md) 를 따른다. Planner 는 § A (도메인 지식) 와 § B (프로젝트 산출물) 모두 대상.
