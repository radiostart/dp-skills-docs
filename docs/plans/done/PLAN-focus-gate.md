# Focus 항목별 반영 게이트 (evaluator) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `.focus.md` 의 사용자 지시가 planner·generator 에게 무시돼도 감지 주체가 없는 공백을, evaluator 의 "focus 항목별 반영 확인" 게이트로 닫는다.

**Architecture:** VERIFICATION REPORT 에 기존 `scope` gate (범위 *이탈* 검사) 와 별개의 `focus` gate (항목별 *반영* 검사) 를 신설한다. evaluator 는 focus 본문을 결정·금지·우선순위 항목으로 분해해 plan·구현 대조 후 4 분류((a) 반영 / (b) 위반 / (c) 미반영 / (d) 무관) 판정하고, (b)/(c) 가 있으면 `focus: fail` + `issues_to_fix` escalate. `verify-report-lint.py` 는 신규 gate 를 enum 검증하되, **focus 라인이 없는 구형 REPORT 는 error 가 아닌 warn** 으로 처리해 기존 `.eval.md` 산출물 호환을 유지한다 (`missing_gate_legacy`).

**Tech Stack:** Python 3 (stdlib only) · unittest (pytest 호환) · markdown wrapper 문서.

**설계 결정 (요청서의 "설계 판단은 직접" 대응):**

- **scope 확장이 아닌 별도 gate**: scope 는 네거티브 검사("벗어나지 않았는가"), focus 는 포지티브 검사("각 지시가 실제 반영됐는가"). 한 gate 에 합치면 증거 축이 섞이고, 기존 scope 의미가 바뀌어 오히려 호환성이 깨진다.
- **lint 하위 호환은 warn 강등으로**: `focus` gate 부재 → `missing_gate_legacy` (severity warn, exit 0 유지). 다른 gate 부재는 종전대로 `missing_gate` (error). lint 는 텍스트만 보므로 REPORT 생성 시점(버전)을 알 수 없다 — 부재를 error 로 하면 기존 워크스페이스의 모든 `.eval.md` 재lint 가 깨진다.
- **mode 직교**: `MODE_GATE_SKIP_RULES` 에 focus 를 넣지 않는다. focus 는 tdd/characterize/standard 와 무관하게 `.focus.md` 존재 여부로만 pass|fail|skip 이 갈린다.
- **planner 누락도 evaluator 가 감지**: plan 에 "focus 반영 사항" 절이 없고 반영 증거도 없으면 (c) 미반영 — planner 단계의 누락이 generator 를 거쳐도 최종 게이트에서 잡힌다.

**관련 파일 (참조):** [ag-evaluator.md](../../agents/ag-evaluator.md) · [verify-report-lint.py](../../tools/verify-report-lint.py) · [guardrails.md](../../skills/context/shared/guardrails.md) · [messages.md](../../skills/context/shared/messages.md) · [focus SKILL.md](../../skills/focus/SKILL.md)

---

### Task 1: verify-report-lint.py — focus gate 스키마 + 하위 호환 (TDD)

**Files:**
- Modify: `dp-skills/tools/verify-report-lint.py:45-54` (GATE_ENUMS), `:225-263` (validate rule 4), `:8-13` (docstring)
- Modify: `dp-skills/tests/tools/test_verify_report_lint.py`
- Modify: `dp-skills/tests/fixtures/verify-reports/valid/01-ready-tdd.md` ~ `05-not-ready-drift.md` (focus 라인 추가)
- Create: `dp-skills/tests/fixtures/verify-reports/valid/06-legacy-no-focus.md`
- Create: `dp-skills/tests/fixtures/verify-reports/invalid/08-focus-ready-fail.md`
- Create: `dp-skills/tests/fixtures/verify-reports/invalid/09-focus-invalid-enum.md`

- [x] **Step 1: 픽스처 갱신·신설**

valid/01~05 의 `- scope: ...` 라인 바로 아래에 focus 라인 추가:

| 픽스처 | 추가 라인 |
|---|---|
| `valid/01-ready-tdd.md` | `  - focus: pass — 항목 1/1 반영 (결제 우선 지시)` |
| `valid/02-ready-characterize.md` | `  - focus: skip — .focus.md 없음` |
| `valid/03-ready-standard.md` | `  - focus: pass — 항목 2/2 반영` |
| `valid/04-not-ready-with-issues.md` | `  - focus: pass — 항목 1/1 반영` |
| `valid/05-not-ready-drift.md` | `  - focus: skip — .focus.md 없음` |

`valid/06-legacy-no-focus.md` 신설 (focus gate 도입 이전 형식 — 기존 03 내용 그대로, focus 라인 없음):

```markdown
## VERIFICATION REPORT
- status: READY
- feature: #11 어드민 메모 UI
- mode: standard
- gates:
  - requirements: pass — features/11-admin-memo.md
  - tdd_evidence: skip — tdd:false & mode≠characterize
  - capture_lockdown: skip — mode 가 characterize 아님
  - test_run: skip — 수기 시나리오로 대체
  - scope: pass — .focus.md 범위 내
  - open_questions: pass — (d) 임의결정 없음
  - drift: none
- metrics:
  - coverage: skip
- issues_to_fix:
  - none
- next: #12 어드민 메모 삭제
```

`invalid/08-focus-ready-fail.md` 신설 (READY 인데 focus fail — 모순):

```markdown
## VERIFICATION REPORT
- status: READY
- feature: #09 배송 메모
- mode: standard
- gates:
  - requirements: pass — features/09-delivery-memo.md
  - tdd_evidence: skip — tdd:false & mode≠characterize
  - capture_lockdown: skip — mode 가 characterize 아님
  - test_run: skip — 수기 시나리오로 대체
  - scope: pass — .focus.md 범위 내
  - focus: fail — 항목 1/2 반영, 미반영: 소프트 딜리트 제외 지시
  - open_questions: pass — (d) 임의결정 없음
  - drift: none
- metrics:
  - coverage: skip
- issues_to_fix:
  - [Major] focus 미반영: 소프트 딜리트 제외 지시 무시 — .focus.md
- next: null
```

`invalid/09-focus-invalid-enum.md` 신설 (focus 값 enum 외):

```markdown
## VERIFICATION REPORT
- status: READY
- feature: #09 배송 메모
- mode: standard
- gates:
  - requirements: pass — features/09-delivery-memo.md
  - tdd_evidence: skip — tdd:false & mode≠characterize
  - capture_lockdown: skip — mode 가 characterize 아님
  - test_run: skip — 수기 시나리오로 대체
  - scope: pass — .focus.md 범위 내
  - focus: maybe — 판단 보류
  - open_questions: pass — (d) 임의결정 없음
  - drift: none
- metrics:
  - coverage: skip
- issues_to_fix:
  - none
- next: null
```

- [x] **Step 2: 실패하는 테스트 작성**

`tests/tools/test_verify_report_lint.py` 의 `CLIBehavior` 클래스 앞에 추가:

```python
class FocusGate(unittest.TestCase):
    """focus gate — .focus.md 항목별 반영 판정 + 구형 REPORT 하위 호환."""

    def test_focus_pass_parsed(self):
        text = (FIXTURES / "valid" / "01-ready-tdd.md").read_text(encoding="utf-8")
        r = m.parse_report(m.extract_report_block(text))
        self.assertEqual(r["gates"]["focus"]["value"], "pass")

    def test_focus_skip_valid_no_errors(self):
        text = (FIXTURES / "valid" / "02-ready-characterize.md").read_text(encoding="utf-8")
        violations, present = m.lint(text)
        self.assertTrue(present)
        errs = [v for v in violations if v.severity == m.Violation.SEVERITY_ERROR]
        self.assertEqual(errs, [], msg=f"errors: {[v.message for v in errs]}")

    def test_legacy_report_without_focus_warns_not_errors(self):
        text = (FIXTURES / "valid" / "06-legacy-no-focus.md").read_text(encoding="utf-8")
        violations, present = m.lint(text)
        self.assertTrue(present)
        errs = [v for v in violations if v.severity == m.Violation.SEVERITY_ERROR]
        warns = [v for v in violations if v.severity == m.Violation.SEVERITY_WARN]
        self.assertEqual(errs, [])
        self.assertIn("missing_gate_legacy", [v.code for v in warns])

    def test_legacy_report_exit_code_pass(self):
        text = (FIXTURES / "valid" / "06-legacy-no-focus.md").read_text(encoding="utf-8")
        violations, present = m.lint(text)
        out = m.render_text(violations, present)
        self.assertIn("WARN", out)
        self.assertNotIn("FAIL", out)

    def test_ready_with_focus_fail_inconsistent(self):
        text = (FIXTURES / "invalid" / "08-focus-ready-fail.md").read_text(encoding="utf-8")
        violations, _ = m.lint(text)
        self.assertIn("status_gate_inconsistency", [v.code for v in violations])

    def test_focus_invalid_enum(self):
        text = (FIXTURES / "invalid" / "09-focus-invalid-enum.md").read_text(encoding="utf-8")
        violations, _ = m.lint(text)
        self.assertIn("invalid_gate_value", [v.code for v in violations])

    def test_not_ready_focus_fail_counts_as_failure_evidence(self):
        text = (
            "## VERIFICATION REPORT\n"
            "- status: NOT_READY\n"
            "- feature: #09 배송 메모\n"
            "- mode: standard\n"
            "- gates:\n"
            "  - requirements: pass\n"
            "  - tdd_evidence: skip\n"
            "  - capture_lockdown: skip\n"
            "  - test_run: skip\n"
            "  - scope: pass\n"
            "  - focus: fail — 항목 0/1 반영\n"
            "  - open_questions: pass\n"
            "  - drift: none\n"
            "- metrics:\n"
            "  - coverage: skip\n"
            "- issues_to_fix:\n"
            "  - [Major] focus 미반영 — .focus.md\n"
            "- next: null\n"
        )
        violations, _ = m.lint(text)
        self.assertNotIn("status_no_failure_evidence", [v.code for v in violations])
```

- [x] **Step 3: 테스트 실행 — 실패 확인**

Run: `cd dp-skills && /opt/homebrew/bin/pytest tests/tools/test_verify_report_lint.py -q`
Expected: FAIL — `missing_gate_legacy` 미구현 (`test_legacy_report_without_focus_warns_not_errors` 등). `test_focus_pass_parsed` 는 파서가 이미 임의 gate 를 수용하므로 픽스처 갱신만으로 통과할 수 있음 — 실패 대상은 lint 규칙 테스트.

- [x] **Step 4: lint 구현**

`tools/verify-report-lint.py` — GATE_ENUMS 에 focus 추가 (scope 다음 위치) + legacy 상수:

```python
GATE_ENUMS: dict[str, set[str]] = {
    "requirements": {"pass", "fail"},
    "tdd_evidence": {"pass", "fail", "skip"},
    "capture_lockdown": {"pass", "fail", "skip"},
    "test_run": {"pass", "fail", "skip"},
    "scope": {"pass", "fail"},
    "focus": {"pass", "fail", "skip"},
    "open_questions": {"pass", "fail", "skip"},
    "drift": {"none", "detected"},
}
REQUIRED_GATES = list(GATE_ENUMS.keys())

# focus gate (0.16.16) 도입 이전 REPORT 호환 — 부재 시 error 대신 warn.
# lint 는 텍스트만 보므로 산출물 생성 버전을 알 수 없다.
LEGACY_OPTIONAL_GATES = {"focus"}
```

validate() rule 4 분기:

```python
    # 4. gate 항목 존재 + enum
    gates = report.get("gates", {})
    for gate_name, allowed in GATE_ENUMS.items():
        if gate_name not in gates:
            if gate_name in LEGACY_OPTIONAL_GATES:
                v.append(Violation(
                    "missing_gate_legacy",
                    f"gate 항목 부재: {gate_name} (구형 REPORT 호환 — 신규 산출물은 포함 필수)",
                    Violation.SEVERITY_WARN,
                ))
            else:
                v.append(Violation("missing_gate", f"gate 항목 부재: {gate_name}"))
            continue
        gate_val = gates[gate_name]["value"]
        if gate_val not in allowed:
            v.append(Violation(
                "invalid_gate_value",
                f"gate {gate_name} 값이 enum 외: {gate_val} (허용: {sorted(allowed)})"
            ))
```

모듈 docstring 의 검증 항목 리스트에 1 줄 추가: `- focus gate 부재 시 warn (구형 REPORT 호환 — missing_gate_legacy)`.

`MODE_GATE_SKIP_RULES` 는 **변경하지 않는다** (focus 는 mode 직교). READY↔focus:fail 모순과 NOT_READY 증거 인정은 기존 rule 5·6 이 gates dict 순회로 자동 커버 — 코드 추가 불필요.

- [x] **Step 5: 테스트 실행 — 통과 확인**

Run: `cd dp-skills && /opt/homebrew/bin/pytest tests/tools/test_verify_report_lint.py -q`
Expected: PASS (기존 + 신규 전부)

Run: `cd dp-skills && /opt/homebrew/bin/pytest tests/tools -q`
Expected: 467 passed (기존 460 + 신규 7)

- [x] **Step 6: Commit**

```bash
git add dp-skills/tools/verify-report-lint.py dp-skills/tests/tools/test_verify_report_lint.py dp-skills/tests/fixtures/verify-reports/ dp-skills/docs/plans/PLAN-focus-gate.md
git commit -m "feat(lint): VERIFICATION REPORT focus gate 검증 추가"
```

---

### Task 2: ag-evaluator.md — focus 항목별 반영 게이트 절차

**Files:**
- Modify: `dp-skills/agents/ag-evaluator.md:27` (step 1 focus 불릿), `:74-77` (step 3 게이트 블록 추가), `:91-98` (REPORT 템플릿), `:108-110` (gate 매핑)

- [x] **Step 1: step 1 의 focus 불릿 강화**

기존 (line 27):

```markdown
   - `focus` 값이 있으면 검토 관점에 반영 (관련 체크 항목 추가·비중 상향).
```

변경:

```markdown
   - `focus` 값이 있으면 검토 관점에 반영 (관련 체크 항목 추가·비중 상향). 또한 step 3 의 **[focus 게이트]** 입력 — 본문을 결정·금지·우선순위 항목 단위로 분해해 둔다.
```

- [x] **Step 2: step 3 에 [focus 게이트] 블록 추가**

`**[Open Questions 게이트]**` 블록 (3번 항목 끝, line 77) 바로 뒤에 추가:

```markdown
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
```

- [x] **Step 3: REPORT 템플릿에 focus 라인 추가**

step 7 템플릿의 `- scope:` 라인 바로 아래에 추가:

```
     - focus:            pass | fail | skip — {항목 N/M 반영 + 미반영 항목 요약, .focus.md 없으면 skip}
```

- [x] **Step 4: gate 매핑 절에 focus 항목 추가**

`**gate 매핑:**` 리스트 (capture_lockdown 다음) 에 추가:

```markdown
   - `focus` — `.focus.md` 항목별 반영 판정 (step 3 [focus 게이트]). `.focus.md` 부재 시 skip. mode 와 직교. `scope` (범위 *이탈* 검사) 와 별개 축 — scope 는 "벗어나지 않았는가", focus 는 "각 지시가 실제 반영됐는가".
```

- [x] **Step 5: lint 로 템플릿 정합 확인 (수동)**

Run: `cd dp-skills && grep -c "focus:" agents/ag-evaluator.md`
Expected: 2 이상 (템플릿 + gate 매핑). 템플릿 placeholder 는 lint 대상 아님 — 형식 검증은 Task 1 픽스처가 담당.

- [x] **Step 6: Commit**

```bash
git add dp-skills/agents/ag-evaluator.md
git commit -m "feat(evaluator): focus 항목별 반영 게이트 절차 추가"
```

---

### Task 3: 주변 SSOT 문서 정합 (guardrails · messages · focus SKILL · code-reviewer 경계)

**Files:**
- Modify: `dp-skills/skills/context/shared/guardrails.md:21-23` (focus 판정 축 추가)
- Modify: `dp-skills/skills/context/shared/messages.md:147-180` (verification_report_example 현행화), `:138` (evaluator 책임 영역 열거)
- Modify: `dp-skills/skills/focus/SKILL.md:85-89` (래퍼와의 상호작용)
- Modify: `dp-skills/agents/ag-code-reviewer.md:144` (gate 영역 열거)
- Modify: `dp-skills/skills/dp-code-review/SKILL.md:79` (gate 영역 열거)

- [x] **Step 1: guardrails.md 에 `### focus` 축 추가**

`### scope` 절 바로 뒤에 추가:

```markdown
### focus

`.focus.md` 의 각 결정·금지·우선순위 항목이 plan·구현에 실제 반영됐는지 항목별로 확인한다. 증거는 항목별 판정표 — (a) 반영 / (b) 위반 / (c) 미반영 / (d) 무관. scope 가 "범위를 벗어나지 않았는가" 라면 focus 는 "지시가 빠짐없이 반영됐는가" — 별개 축. `.focus.md` 부재 시 skip.
```

- [x] **Step 2: messages.md `verification_report_example` 현행화**

READY 예시를 현행 템플릿 (mode·capture_lockdown·focus·open_questions·metrics 포함) 으로 교체:

```
## VERIFICATION REPORT
- status: READY
- feature: #10 주문 취소 API
- mode: red_contract
- gates:
  - requirements: pass — features/10-order-cancel.md § 조건/트리거/기대결과
  - tdd_evidence: pass — .plan.md 스텝 1~4 모두 [Red][Green] 기록
  - capture_lockdown: skip — mode 가 characterize 아님
  - test_run: pass — {test_command} {test_path}/order_cancel_spec.{ext} (exit 0)
  - scope: pass — .focus.md 범위 내 ({source_root}/services/order_cancel_service.{ext})
  - focus: pass — 항목 1/1 반영 (취소 사유 enum 고정)
  - open_questions: pass — (d) 임의결정 없음
  - drift: none
- metrics:
  - coverage: skip
- issues_to_fix:
  - none
- next: #11 취소 이력 조회
```

NOT_READY 예시도 동일 형식으로 교체하되 focus fail 사례를 시연:

```
## VERIFICATION REPORT
- status: NOT_READY
- feature: #10 주문 취소 API
- mode: red_contract
- gates:
  - requirements: fail — 환불 트리거 미구현 (features/10-order-cancel.md § 기대결과 3)
  - tdd_evidence: fail — .plan.md 스텝 3 [Red] 누락
  - capture_lockdown: skip — mode 가 characterize 아님
  - test_run: pass — {test_command} {test_path}/order_cancel_spec.{ext} (exit 0)
  - scope: pass — .focus.md 범위 내
  - focus: fail — 항목 1/2 반영, 미반영: "환불 큐 비동기 전환 제외" 지시
  - open_questions: pass — (d) 임의결정 없음
  - drift: none
- metrics:
  - coverage: skip
- issues_to_fix:
  - [Major] 환불 트리거 미구현 — {source_root}/services/order_cancel_service.{ext}:42
  - [Major] 스텝 3 [Red] 증거 누락 — .plan.md:15
  - [Major] focus 미반영: 환불 큐 비동기 전환 제외 지시가 구현에서 무시됨 — .focus.md
- next: null
```

두 예시를 Task 1 의 lint 로 검증: `python3 tools/verify-report-lint.py` 에 stdin 으로 넣어 PASS 확인 (아래 Step 6).

- [x] **Step 3: messages.md code-review 예시의 evaluator 책임 영역 열거에 focus 추가**

line 138:

```
  - evaluator 책임 영역 (requirements·tdd_evidence·test_run·capture_lockdown·open_questions·focus)
```

- [x] **Step 4: focus/SKILL.md 래퍼와의 상호작용에 감지 장치 명시**

`## 래퍼와의 상호작용` 첫 불릿 뒤에 추가:

```markdown
- `@ag-evaluator` 는 focus 본문을 항목 단위로 분해해 plan·구현 반영 여부를 판정하고 VERIFICATION REPORT 의 `focus` gate (pass|fail|skip) 로 보고한다. 미반영·위반 항목은 `issues_to_fix` 로 escalate — focus 가 조용히 무시되는 것을 감지하는 장치.
```

- [x] **Step 5: code-reviewer 경계 열거에 focus 추가 (2 곳)**

`agents/ag-code-reviewer.md:144`:

```
5. evaluator gate 영역 지적 금지 (requirements / tdd_evidence / test_run / capture_lockdown / open_questions / focus)
```

`skills/dp-code-review/SKILL.md:79`:

```
- evaluator gate 영역 (requirements·tdd_evidence·test_run·capture_lockdown·open_questions·focus) 금지.
```

- [x] **Step 6: messages.md 예시를 lint 로 검증**

Run (dp-skills/ 에서):

```bash
awk '/^### `verification_report_example`/,/^### `slack.activated`/' skills/context/shared/messages.md | awk '/^## VERIFICATION REPORT/{n++} n==1' | python3 tools/verify-report-lint.py
awk '/^### `verification_report_example`/,/^### `slack.activated`/' skills/context/shared/messages.md | awk '/^## VERIFICATION REPORT/{n++} n==2' | python3 tools/verify-report-lint.py
```

Expected: 둘 다 `PASS: 위반 없음` (exit 0).

- [x] **Step 7: Commit**

```bash
git add dp-skills/skills/context/shared/guardrails.md dp-skills/skills/context/shared/messages.md dp-skills/skills/focus/SKILL.md dp-skills/agents/ag-code-reviewer.md dp-skills/skills/dp-code-review/SKILL.md
git commit -m "docs(shared): focus 판정 축·REPORT 예시·책임 경계 정합"
```

---

### Task 4: docs 재생성 + 버전 bump + 전체 검증

**Files:**
- Modify: `dp-skills/.claude-plugin/plugin.json` (0.16.15 → 0.16.16)
- Regenerate: `dp-skills/docs/reference/agents/*.md`, `dp-skills/docs/reference/skills/*.md` (docs_build.py 산출)

- [x] **Step 1: docs_build 재실행**

Run: `cd dp-skills && python3 tools/docs_build.py`
Expected: exit 0, `docs/reference/agents/ag-evaluator.md`·`ag-code-reviewer.md`, `docs/reference/skills/focus.md`·`dp-code-review.md` 갱신.

- [x] **Step 2: plugin.json patch bump**

`dp-skills/.claude-plugin/plugin.json` 의 `"version": "0.16.15"` → `"0.16.16"`.

- [x] **Step 3: 전체 테스트 + doctor 스키마 검증**

Run: `cd dp-skills && /opt/homebrew/bin/pytest tests/tools -q && python3 tools/doctor.py --schema`
Expected: 467 passed + doctor exit 0.

- [x] **Step 4: Commit**

```bash
git add dp-skills/docs/reference/ dp-skills/.claude-plugin/plugin.json
git commit -m "chore(release): bump 0.16.16"
```

---

### Task 5: PR

- [x] **Step 1: 브랜치 정리 + push**

```bash
git branch -m worktree-focus-evaluator-gate feat/focus-evaluator-gate
git push -u origin feat/focus-evaluator-gate
```

pre-push hook 이 dp-skills 검증 (doctor --schema · tests/tools · tests/hooks · docs drift) 을 실행 — `--no-verify` 금지.

- [x] **Step 2: PR 생성 (base main, squash 머지 전제)**

제목: `feat(evaluator): focus 항목별 반영 게이트 (0.16.16)`

본문 요지: 문제 (focus 무시 감지 주체 공백) → 해법 (evaluator focus gate + lint 하위 호환 warn) → 검증 (테스트 수, messages 예시 lint PASS).
