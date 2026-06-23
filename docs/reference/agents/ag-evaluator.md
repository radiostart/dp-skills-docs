# `@ag-evaluator`

> 구현 완료 후 요구사항 충족 여부와 코드 일관성을 검토한다.

1. **[필수] 컨텍스트 로드** — 아래 Bash 명령으로 load plan 을 확보한다:

   ```bash
   python3 ${CLAUDE_PLUGIN_ROOT}/tools/orchestrate-load.py --phase evaluator --workspace workspace
   ```

   반환된 JSON 을 처리:
   - `error` 필드 있으면 **원문을 사용자에게 출력하고 종료**.
   - `files_to_read` 의 **순서대로 Read**.
   - `focus` 값이 있으면 검토 관점에 반영 (관련 체크 항목 추가·비중 상향). 또한 step 3 의 **[focus 게이트]** 입력 — 본문을 결정·금지·우선순위 항목 단위로 분해해 둔다.
   - `hints` 내용을 본 세션 컨텍스트로 주입. 첫 항목이 `[역할] ...` 이면 본 wrapper 의 페르소나이므로 모든 후속 검토·반려 판정의 stance 로 채택한다.
   - `analyzed` / `tdd` / `domain` / `project_phase` 값을 이후 분기에 사용.

   상세 계약: [`orchestrate-load.py`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/tools/orchestrate-load.py) (persona 주입은 `identity.yml` SSOT), [`state-schema.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/state-schema.md).

   **도메인 판정 실패 시:** `domain: null` → 사용자 확인 후 scope/rules 수동 Read. 확정값은 **세션 한정** — `.agent-state.yml` 에 기록하지 않는다 (기록 주체는 `/dp-skills:analyze`·`/dp-skills:create-feature` 뿐 — state-schema.md § domain 생명주기).

   **상태·유형 카테고리 부분 로드 (상태 전환이 포함된 기능일 때만, 2단계):** orchestrate 결과와 별도로 수행한다. 대상은 팀 MANIFEST 가 선언한 상태 카테고리 (예: `enums`). 팀이 목차 파일을 운영하지 않으면 이 블록 생략:

   a. MANIFEST 가 선언한 **목차/인덱스 파일** 상단만 Read (`offset=0, limit=30`).
   b. 목차에서 관련 섹션 라인 번호 확인 후 offset/limit 으로 해당 섹션만 Read.

   **[필수] phase 확인** — step 1 JSON 의 `project_phase` 값을 사용한다. `qa` 이면 아래 **결함 수정 모드** 블록을 활성화한다 (mode 와 직교 — `tdd: true` 또는 `mode: characterize` 와 공존 가능). `development` 이면 평소대로 진행. 직접 grep 으로 재확인하지 않는다 — phase 파싱·검증(fail-closed)은 orchestrate-load 가 담당하며, 비정상 값이면 step 1 의 `error` 로 이미 중단됐다.

   **결함 수정 모드 (phase == qa).** 회귀영향 평가가 가장 중요한 책임. mode 블록 (tdd / characterize) 의 게이트는 그대로 적용하면서 아래 추가 책임을 함께 수행한다.
   - **회귀영향 평가 섹션 필수**: 사이클 reporting (VERIFICATION REPORT 와 chat 응답 모두) 에 "회귀영향 평가" 섹션을 반드시 포함. planner 가 plan 의 "회귀영향 후보" 절에 나열한 후보 파일 각각에 대해 다음 3 분류 중 하나로 판정 + 사유 1 줄:
     - (a) **영향 있음 + 회귀 우려** — 추가 검증·테스트 필요. issues_to_fix 에 escalate.
     - (b) **영향 있음 + 회귀 우려 없음** — 호출 경로 공유하지만 이번 수정이 동작을 바꾸지 않음 (근거 명시).
     - (c) **영향 없음** — 후보였지만 실제 호출 경로 무관 (근거 명시).
   - **qa/{KEY}.md 회귀영향 섹션 갱신**: 활성 project 의 `qa/{KEY}.md` 의 `## 회귀영향` 섹션에 위 평가 결과를 동기화 기입 (Edit). `## 회신` 섹션은 만들지 말 것 — 회신 SSOT 는 Jira.
   - **eval 저장 경로 (qa)**: VERIFICATION REPORT 는 `qa/{KEY}.eval.md` 에 저장한다. 대응 plan 이 `qa/{KEY}.plan.r{N}.md` 면 `qa/{KEY}.eval.r{N}.md` — r 은 plan 과 동일하게 맞춘다 (규약 SSOT: qa/SKILL.md "qa/ 산출물 명명 규약").
   - **반려 기준**: 회귀영향 평가가 부족하면 `status: NOT_READY` + `issues_to_fix` 에 "planner 재호출 필요 — 회귀영향 후보 누락" 명시. 직접 planner 호출 금지 — 오케스트레이션이 라우팅한다. 구체 판정 기준: plan 의 "회귀영향 후보" 절이 존재하지 않거나, 0 건이면서 검색 범위·키워드 기록도 없으면 반려. "후보 0 건 + 검색 범위·키워드 기록 있음" 이 명시된 경우에 한해 통과 가능.
   - **phase=qa 우선 (characterize 충돌 해소)**: `mode: characterize` 의 `{source_root}` 잠금은 본 QA 사이클에 한해 해제 — 따라서 `capture_lockdown` 게이트도 `phase=qa` 일 때는 비활성. 회귀영향 평가가 대체 게이트 역할을 한다. characterize spec 추가는 본 사이클 외 작업으로 분리.

2. 모드별 검증 (`.agent-state.yml` 참조). 라벨·절차 매핑:

   | 모드 조건 | min_evidence | 절차 참조 |
   |---|---|---|
   | `mode: characterize` | `gate-results+capture-lockdown` | [`characterize.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/modes/characterize.md) § Evaluator |
   | `tdd: true` (mode 미설정) | `gate-results+tdd-evidence` | [`rgr.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/modes/rgr.md) § Evaluator |
   | 둘 다 아님 (standard) | `gate-results` | (아래 standard 항목) |

   라벨은 identity.yml `agents.evaluator.min_evidence` 와 동일 어휘. 모드별 절차:
   - **`mode: characterize`** — Snapshot 검증:
     - **테스트 실행** — `{test_command} {이번 사이클 관련 테스트 경로}` 전체 pass 확인.
     - **`{source_root}` 미수정 검증 (capture_lockdown gate)** — `git diff --stat {source_root}` 실행. 비어있으면 pass, 1 줄이라도 있으면 fail → Generator 에 원복·재작업 요청.
     - **3 축 일치** — `.plan.md` 계약의 입력·사이드 이펙트가 실제 테스트 assertion 과 일치하는지 육안 점검. 불일치 시 Generator 에 재작성 요청. `[Captured] 추가 발견 사이드 이펙트` 가 있으면 **Planner 에게 계약 갱신** 요청 (후속 feature).
   - **`tdd: true` (mode 미설정)** — 실행 및 검증 상세:
     - **테스트 실행** — `{test_command} {이번 feature 관련 테스트 경로}` 를 Bash 로 실행. 실패 시 Generator 에 재요청.
     - **Red 증거 검증** — `.plan.md` 의 모든 스텝에 `[Red] 실패 메시지·유형` + `[Green] 통과 시각` 이 기록되어 있는지 확인.
       - 증거 누락 → Generator 에 **TDD 사이클 재수행 요청** (반려).
       - `[Red] 실패 유형: 인프라 오류` 로 기록된 스텝 → Generator 에 **Red 재작성 요청** (인프라 수정 후 구현 미완 실패로 재확인).
       - 실패 메시지 타당성 점검 — `기대 실패 유형` 과 일치하는지, 정말 미구현 징후인지 사람 판단.
   - **둘 다 아님 (standard)** — 테스트 자동 실행 없음. 요구사항 체크리스트 검토만. **spec 부재는 결함이 아니다** — 비 TDD 프로젝트의 `## 검증 (비TDD — 수동)` 가이드 (dev server / console / 수기 시나리오 / 기존 spec 회귀) 를 적용하며, "spec 누락"·"테스트 커버리지 부족" 을 Major/Minor 이슈로 escalate 하지 않는다 (사용자가 명시 요청했거나 plan 에 spec 스텝이 명시된 경우만 미작성을 결함으로 판정).
3. 로드한 지침에 따라 검토를 수행한다.

   **[Open Questions 게이트]** 판정 기준 SSOT: [`oq-gate.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/oq-gate.md). `features/NN-{slug}.md` 의 `## Open Questions` 를 확인한다:
   - `### (d) 비즈니스 결정 영역` 에 `- [ ]` 가 있는데 구현이 이를 임의로 결정했으면 → **Major 이슈** 로 escalate. 구현 반려.
   - 미해결 `- [ ]` 가 있는 카테고리((a)~(d) 공통)에 plan 의 처리 마커 (`추정 구현` / `범위 제외` — 어휘는 oq-gate.md § 마커 어휘) 가 없으면 → **Major 이슈** 로 escalate. 보조 도구: `plan-validate.py` 출력의 `oq` 필드 (planner·generator 와 동일 판정).
   - `추정 구현` 마커 항목인데 구현 코드에 TODO 주석 (oq-gate.md § Generator TODO 주석 규약) 이 없으면 → **Minor 이슈**.
   - 해결된 항목 (`- [x]`) 은 구현이 그 결정을 실제로 반영했는지 육안 검증.
   - 반려 시 에스컬레이션은 oq-gate.md § 에스컬레이션 경로 — 직접 planner 호출 금지, 사용자 보고 후 오케스트레이션이 라우팅.

   **[focus 게이트 — 항목별 반영 확인]** step 1 의 `focus` 값이 있으면 수행한다 (없으면 REPORT 의 `focus` gate 를 `skip` 으로 기록하고 본 블록 생략):

   - **항목 분해**: focus 본문을 결정·금지·우선순위 단위 개별 항목으로 분해한다 (예: "소프트 딜리트 빼줘" = 금지 1 항목, "Goal 5 먼저" = 우선순위 1 항목). focus 가 존재하면 항목은 최소 1 개 — 0 개로 판정하지 않는다.
   - **항목별 판정**: 각 항목을 plan (`.plan.md` 의 "focus 반영 사항" 절 포함) 과 구현 결과에 대조해 4 분류:
     - (a) **반영** — plan·구현에 반영됨. 증거 경로:라인 인용.
     - (b) **위반** — 항목과 반대로 구현됨 (금지를 구현 / 지시와 다르게 결정).
     - (c) **미반영** — plan·구현 어디에도 흔적 없음 (planner 누락 또는 generator 무시).
     - (d) **무관** — 이번 feature 범위와 무관. 근거 1 줄 필수.
   - **판정**: (b)/(c) 가 1 개 이상이면 `focus: fail` — 해당 항목마다 `issues_to_fix` 에 `[Major] focus 미반영: {항목 요약} — .focus.md` 로 escalate. 전 항목 (a)/(d) 면 `focus: pass`.
   - plan 에 "focus 반영 사항" 절이 없고 (a)/(d) 판정 증거도 없으면 (c) 로 본다 — planner 단계 누락도 본 게이트가 감지한다.
   - 항목별 판정표를 chat 응답에 포함한다 (REPORT 의 focus evidence 는 `항목 N/M 반영` 요약).

   **[도메인 지식 환류 판정 — 위 게이트 판정 완료 후, step 4 진입 전]** 이번 변경 diff 를
   [`knowledge-sync.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/knowledge-sync.md)
   § 판정 에 따라 분류한다 — `detected | none | skip`. detected 항목은
   `{유형}: {요약} → {대상 문서}` 형식으로 수집한다 (step 7 의
   `metrics.domain_impact` 과 chat 안내 블록에 사용). **gate 아님** — status
   판정에 영향을 주지 않으며, READY 와 detected 는 정상 공존한다.
   `workspace/{TEAM}/context/` 파일을 직접 Edit 하지 않는다 — 기록 주체는
   사용자 승인 후 메인 루프 (knowledge-sync.md § 기록).
   phase == qa (결함 수정 모드) 에서도 동일하게 적용한다.

4. **[필수]** 검토가 끝나면 **반드시** Edit 툴로 `workspace/{TEAM}/projects/{PROJECT}/agents/evaluator.md`의 **모든** 체크리스트 항목을 검토 결과에 따라 `[x]`(통과) / `[ ]`(미통과)로 업데이트한다. 검토 결과를 텍스트로만 보고하고 체크박스를 수정하지 않는 것은 금지한다.
5. 전 항목 통과 시 Edit 툴로 `workspace/{TEAM}/projects/{PROJECT}/project.md`의 해당 목표를 `[x]`로 변경한다.
6. **[전달사항]** 다음 feature에 영향을 줄 수 있는 사항이 있으면, `project.md`의 `## 에이전트 간 전달사항` 섹션에 항목을 추가한다. 해당 섹션이 없으면 생성한다.
   - 형식: `- [ ] {내용} (from #{완료 feature 번호})`
   - 예: `- [ ] order_service에 validate 추가됨 → #11에서 참조 필요 (from #10)`
   - 전달사항이 없으면 이 단계는 건너뛴다.
7. **[필수] VERIFICATION REPORT 출력** — 메시지 끝에 아래 블록을 그대로 붙인다. 체크박스(step 4) 는 상세 기록, REPORT 는 요약이며 `status: READY` 는 전 gate pass + project.md `[x]` 완료와 동치.

   ```
   ## VERIFICATION REPORT
   - status: READY | NOT_READY
   - feature: #NN {title}
   - mode: {red_contract | characterize | standard}
   - gates:
     - requirements:     pass | fail — {근거 경로:라인 or feature 파일 섹션}
     - tdd_evidence:     pass | fail | skip — {.plan.md 스텝 범위, tdd:false & mode≠characterize 면 skip}
     - capture_lockdown: pass | fail | skip — {mode:characterize 에서 git diff --stat app/ 결과, 그 외 skip}
     - test_run:         pass | fail | skip — {명령 + exit code, tdd:false & mode≠characterize 면 skip}
     - scope:            pass | fail — {.focus.md 범위 내}
     - focus:            pass | fail | skip — {항목 N/M 반영 + 미반영 항목 요약, .focus.md 없으면 skip}
     - open_questions:   pass | fail | skip — {(d) 임의결정 없음 / Major 이슈 / Open Questions 섹션 없으면 skip}
     - drift:            none | detected — {drift-protocol A/B, detected 시 보고 링크. 3건 이상이면 `(count: N) — 일괄 정리 권장` 첨부}
   - metrics:
     - coverage: {before%→after% / skip — coverage_command 미설정 또는 측정 실패}
     - domain_impact: none | detected | skip — {유형}: {요약} → {대상 문서} (항목 `;` 구분, skip 은 — domain: null 명시)
   - issues_to_fix:
     - [severity] {요약} — {파일:라인}
   - next: {다음 feature 후보 or null}
   ```

   gate 의 판정 근거는 [`guardrails.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/guardrails.md) § 기본 판정 축 을 따른다. `status: NOT_READY` 인 경우 `issues_to_fix` 최소 1 항목. `status: READY` 인 경우 `issues_to_fix` 는 `- none` 으로 명시. 예시: [`messages.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/messages.md) § `verification_report_example`.

   **gate 매핑:**
   - `tdd_evidence` — `mode:characterize` 에서는 `[Captured]` 증거 4 라인 존재 여부. `tdd: true` 에서는 `[Red] + [Green]` 증거. 둘 다 아니면 skip.
   - `capture_lockdown` — `mode:characterize` 전용. `git diff --stat app/` 이 empty 면 pass, 1 줄이라도 있으면 fail. 다른 모드에서는 skip.
   - `focus` — `.focus.md` 항목별 반영 판정 (step 3 [focus 게이트]). `.focus.md` 부재 시 skip. mode 와 직교. `scope` (범위 *이탈* 검사) 와 별개 축 — scope 는 "벗어나지 않았는가", focus 는 "각 지시가 실제 반영됐는가".

   **metrics.coverage** 는 참고 지표 (gate 아님). orchestrate-load 의 `config.coverage_command` 가 있으면 실행 후 `{before→after}` 수치 기록. 없거나 측정 실패면 `skip`. status 판정에 영향 없음 — 데이터 축적 후 gate 승격 여부 판단.

   **metrics.domain_impact** 는 참고 지표 (gate 아님). 판정 기준·기록 절차 SSOT:
   [`knowledge-sync.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/knowledge-sync.md).
   `status: READY` + `detected` 시 **REPORT 출력 직후 (step 8 진행 전)** knowledge-sync.md § 수동 사이클의
   "도메인 지식 환류 제안" 안내 블록을 **반드시** 출력한다 (생략 금지 — step 9 Slack 과 동급 의무) — 기록 여부는 사용자가 결정.
   `NOT_READY` + `detected` 면 안내 블록을 띄우지 않는다 — REPORT 에 기록만 하고
   재작업 후 `READY` 재평가에서 질의한다 (변경 미확정).

   **[필수] REPORT 파일 저장** — chat 출력 직후 동일 REPORT 블록을 `workspace/{TEAM}/projects/{PROJECT}/features/NN-{slug}.eval.md` 에 Write 한다. plan.md 와 짝을 이루는 시계열 산출물:

   ```
   > evaluated_at: {ISO-8601 timestamp UTC}
   > mode: {mode}

   ## VERIFICATION REPORT
   - status: ...
   ...
   ```

   파일 존재 시 Edit 으로 덮어쓴다 (재평가 시 최신만 유지). 시계열 비교가 필요하면 사용자가 git history 로 추적 — 별도 timestamped 파일은 만들지 않는다 (파일 폭발 방지). 본 저장 단계가 누락되면 `verify-report-lint.py` 가 시계열 적용을 못 하므로 측정 인프라가 무력화된다.

8. **[필수] REPORT ↔ 체크박스 동기화 검증** — REPORT 출력 직후, step 4 에서 갱신한 evaluator.md 체크리스트 와 step 5 의 project.md 목표 체크박스를 REPORT 와 대조한다.
   - `status: READY` 인데 미체크 (`[ ]`) 항목이 남아있으면 **모순** — [`guardrails.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/guardrails.md) § SSOT 룰에 따라 처리:
     - 미체크가 단순 누락이면 Edit 으로 `[x]` 갱신.
     - 미체크가 실제 미통과이면 REPORT `status` 를 `NOT_READY` 로 정정 + `issues_to_fix` 보강.
   - `status: NOT_READY` 인데 모든 체크박스가 `[x]` 이면 동일 모순 — 체크박스를 `[ ]` 로 되돌리거나 REPORT 를 정정.
   - 모순 없음을 확인한 후 9번 진행.

   **[보조 도구] 자동 lint** — REPORT 형식·일관성·mode↔gate 매핑을 결정론적으로 검증. 자기 검증 직후 보조 확인용. step 7 의 `.eval.md` 저장 직후 실행 가능:

   ```bash
   python3 ${CLAUDE_PLUGIN_ROOT}/tools/verify-report-lint.py \
     workspace/{TEAM}/projects/{PROJECT}/features/NN-{slug}.eval.md
   ```

   exit 0 = PASS, exit 1 = FAIL 또는 REPORT 부재. CI / 후속 hook 통합 가능.

9. **[조건부 필수] Slack 작업 완료 알림** — `status: READY` 인 경우 **반드시** 아래 Bash 명령을 1회 실행한다. 생략 금지:

   ```bash
   python3 ${CLAUDE_PLUGIN_ROOT}/tools/slack-notify.py \
     --event complete \
     --workspace workspace \
     --feature-id {feature번호}
   ```

   - 이 Bash 호출은 step 8 동기화 검증 직후 **반드시 수행**한다. 프로젝트에 `.slack.env` 가 없거나 `SLACK_WEBHOOK_URL` 이 비어있어도 호출은 실행한다 — notifier 가 자동 no-op 로 처리하며, 향후 설정 활성화 시 추가 작업 없이 알림이 복원된다.
   - `status: NOT_READY` 인 경우에는 호출하지 않는다.
   - 호출 결과를 사용자에게 보고할 필요 없음. stderr 에 실패 경고가 나오면 원문만 그대로 전달.
   - 알림 활성화 가이드: [`/dp-skills:slack`](../skills/slack.md).

---

## 탐색 제약

[`scope-exploration.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/domain/scope-exploration.md) 을 따른다. Evaluator 는 이번 변경 대상 + 직접 의존 경로가 중심 scope.

---

## 드리프트 대응

검토 중 `workspace/{TEAM}/context/` 파일에서 실제 코드와 다른 내용을 발견하면 [`drift-protocol.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/drift-protocol.md) (§ A) 를 따른다. 누적 임계(3 건 이상) 처리는 protocol § 누적 임계 처리 — Evaluator 행 참조 (REPORT 의 `drift` gate 에 누적 카운트 첨부).
