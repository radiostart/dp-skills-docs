# 도메인 지식 환류 (knowledge-sync) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** evaluator 사이클 종료 시 이번 변경분이 도메인 지식 문서 (`scope/`·enums·boundaries·MANIFEST) 에 반영할 신규·변경 지식을 만들었는지 판정해 REPORT 에 기록하고, 사용자 승인 게이트를 거쳐 도메인 문서를 갱신하는 환류 경로를 만든다.

**Architecture:** 판정 기준·기록 절차의 SSOT 는 신규 `skills/context/lifecycle/knowledge-sync.md`. evaluator 는 **감지·보고만** 담당 (REPORT `metrics.domain_impact` — gate 아님, status 비영향) 하고, 기록은 사용자 승인 후 메인 루프가 drift-protocol § A 와 동일한 보고→승인→Edit 경로로 수행한다. run-cycle 은 feature 마다 멈추지 않고 detected 항목을 누적해 큐 종료 시 일괄 질의한다. `verify-report-lint.py` 가 새 필드를 legacy-tolerant 로 검증한다.

**Tech Stack:** markdown 명세 (skills/context), Python 3 표준 라이브러리 + unittest (verify-report-lint), 기존 테스트 컨벤션 (`tests/tools/`, importlib 동적 로드).

---

## 배경 (실데이터 근거 — 2026-06-11 측정)

지식 루프가 현재 **단방향**이다: discover (생산) → orchestrate-load (소비) → 사이클이 코드를 변경 → scope 는 그대로.

- nimda 레포 2026-03 이후 **978 커밋** 동안 `workspace/dp-team/context/scope/` 갱신은 **2건** — 둘 다 유지보수 (표 폭 정리·폴더 구조 작업) 이며 코드 변경의 지식 환류는 **0건**.
- 같은 기간 Presend-Admin 프로젝트는 **31개 feature + QA 결함 11건**으로 retail 도메인 소스를 변경했다.
- Presend-Admin 의 eval REPORT **19건 전부 `drift: none`** — drift-protocol 은 "에이전트가 읽다가 발견한 기존 문서의 오류" 만 잡는 우연적 경로라, **누락 (코드에 새로 생겼는데 문서에 없는 지식) 은 구조적으로 감지 불가**다.

evaluator 는 사이클 말미에 diff 전체를 이미 검토한 상태이므로, 도메인 영향 판정의 한계비용이 가장 낮은 지점이다.

## 설계 결정 (확정)

| 결정 | 내용 | 근거 |
| --- | --- | --- |
| 감지 주체 | evaluator (사이클 종료 시) | diff 전체를 이미 검토 — 한계비용 최소 |
| 표기 위치 | REPORT `metrics.domain_impact` (**gate 아님**) | lint rule 5 가 `drift: detected` + READY 를 모순 처리 — domain_impact 는 READY 와 공존이 정상 케이스. metrics 의 기존 계약("참고 지표, status 비영향")과 일치. parser 가 metrics 임의 키를 이미 수집해 변경 최소 |
| 기록 주체 | 사용자 승인 후 **메인 루프** (evaluator 직접 Edit 금지) | drift-protocol 공통 원칙 1 "즉시 Edit 금지" — 판정자·기록자 분리 유지 |
| 질의 시점 (수동) | evaluator chat 응답 말미 안내 블록 → 사용자 승인 → 메인 루프 Edit | 사용자 요구 "이벨류에이터 종료 후 물어보고 진행" |
| 질의 시점 (run-cycle) | feature 마다 묻지 않고 **누적 → 큐 종료(완료·정지) 시 일괄 질의** | green 자동 진행 원칙 훼손 방지 — drift-protocol 누적 임계 처리와 동일 사상 |
| 환류 대상 | `scope/{domain}.md`(분할 포함)·enums(팀 운영 시)·`boundaries/{A}--{B}.md`·MANIFEST 외부 reference 표 | discover 의 생산 대상과 동일 |
| 환류 비대상 | `rules/{domain}.md` | 정책은 코드에서 추론 불가 (discover 와 동일 경계). 구현이 rules 와 상충하면 그건 drift § A 경로 |
| phase 적용 | development·qa 직교 적용 | QA 결함 수정도 상태값·검증 규칙을 바꿈 (Presend-Admin 실사례) |
| lint 호환 | `domain_impact` 부재 시 **warn** (focus 의 `missing_gate_legacy` 패턴) | 기존 eval 산출물 19건+ 이 FAIL 로 바뀌면 안 됨 |
| 대규모 변경 | 항목 5건 이상 또는 구조 개편이면 `/dp-skills:discover` 재실행 안내 | scope·MANIFEST 는 단순 덮어쓰기 idempotent — 재실행이 더 안전 |

**범위 제외 (YAGNI):** doctor 의 domain_impact 통계 검사, 자동 Edit (승인 없는 기록), pilot 레포 동기 반영 (별도 역이식 사이클).

---

### Task 1: `knowledge-sync.md` 신설 (SSOT)

**Files:**
- Create: `dp-skills/skills/context/lifecycle/knowledge-sync.md`

- [ ] **Step 1: 문서 작성**

아래 내용으로 신규 생성:

````markdown
# 도메인 지식 환류 (knowledge-sync)

사이클 (development feature · qa 결함) 의 소스 변경이 `workspace/{TEAM}/context/`
의 도메인 지식 문서에 반영할 **신규·변경 지식**을 만들었는지 판정하고,
사용자 승인 후 문서에 기록하는 절차. [drift-protocol.md](drift-protocol.md) 가
"기존 문서가 코드와 다름 (우연 발견)" 을 다룬다면, 본 프로토콜은 "이번 변경이
만든 지식의 누락 (사이클 종료 시 체계 점검)" 을 다룬다.

---

## 판정 (evaluator — 감지·보고만)

evaluator 가 사이클 검토 완료 후 이번 변경 diff 를 기준으로 판정한다.
**evaluator 는 context/ 파일을 직접 Edit 하지 않는다** — 기록 주체는 사용자
승인을 받은 메인 루프다 (drift-protocol 공통 원칙 1 과 동일).

### detected — 다음 중 1개 이상이 diff 에 존재

discover 단계 2 의 수집 카테고리와 동일 어휘:

1. 라우트/엔드포인트 추가·제거·시그니처 변경
2. 도메인 모델·테이블·컬럼 추가/변경
3. enum·상태값·도메인 상수 추가/변경/제거
4. 외부 의존 (API·MQ·DB·타 도메인 호출) 추가/제거
5. 비즈니스 용어·개념 신설
6. scope 문서에 **이미 기록된** 항목의 동작 변경

### none — 노이즈 가드

내부 리팩터 (공개 표면 불변)·버그 수정 (문서 기록 단위의 동작 불변)·테스트만
변경·스타일/주석. 판단 휴리스틱: **"이 변경을 모르는 다음 planner 가 잘못된
계획을 세울 수 있는가"** — 아니면 none.

### skip

`domain: null` (판정 기준 도메인 없음). scope 파일 부재는 skip 이 아니라
detected — evidence 에 "scope 문서 없음 — discover 신규 작성 권장" 을 적는다.

## REPORT 표기

VERIFICATION REPORT `metrics` 블록에 기록한다 (**gate 아님** — status 판정
비영향, READY 와 detected 공존이 정상):

```
- metrics:
  - coverage: ...
  - domain_impact: none | detected | skip — {유형}: {요약} → {대상 문서} (항목별 `;` 구분)
```

evidence 예: `enum: 발송상태 CANCELED 추가 → scope/retail.md; 외부 의존: 재고 API 호출 신설 → boundaries/retail--inventory.md`

## 질의·기록 절차

### 수동 사이클

1. evaluator 는 `detected` 시 chat 응답 말미 (REPORT 직후) 에 안내 블록을 출력한다:

   ```
   ## 도메인 지식 환류 제안
   이번 변경이 도메인 지식 문서에 반영할 항목을 만들었습니다:
   | # | 유형 | 내용 | 대상 문서 |
   |---|---|---|---|
   | 1 | {유형} | {요약} | {경로} |
   기록할까요? 승인 시 항목별 before/after 미리 보기 후 반영합니다.
   (절차: knowledge-sync.md § 기록)
   ```

2. 사용자 승인 시 **메인 루프**가 아래 § 기록 을 수행한다. 거부 시 기록하지
   않고 종료 — 미기록 지식은 이후 사이클에서 drift 로 발견될 수 있다 (의도된
   안전망).

### run-cycle (오토파일럿)

feature 마다 질의로 정지하지 않는다. READY 시 detected 항목을 메모리에
누적하고, **큐 종료 시점 (완료 요약 또는 정지 보고)** 에 누적 표를 출력해
일괄 질의한다. 이후 절차는 수동 사이클과 동일.

## 기록 (메인 루프 — 승인 후)

1. 항목별 대상 문서의 before/after 를 제시하고 최종 승인을 받는다
   (drift-protocol 공통 원칙 2 와 동일 형식).
2. Edit 으로 반영. scope 구조·200줄 제약은
   [split-policy](../../discover/references/split-policy.md), MANIFEST 갱신은
   [manifest-update](../../discover/references/manifest-update.md) 를 따른다.
3. **항목 5건 이상 또는 섹션 구조 개편**이면 개별 Edit 대신
   `/dp-skills:discover {entry} {domain}` 재실행을 권장한다 (scope·MANIFEST 는
   단순 덮어쓰기 — 재실행이 더 안전).
4. `rules/{domain}.md` 는 환류 대상이 아니다 — 정책은 코드에서 추론 불가.
   구현과 rules 의 상충 발견은 [drift-protocol § A](drift-protocol.md) 경로.

## 경계

| 축 | drift (drift-protocol) | domain_impact (본 문서) |
| --- | --- | --- |
| 무엇 | 기존 문서가 실제와 다름 | 이번 변경이 만든 신규·변경 지식의 미반영 |
| 발견 | 사이클 중 우연 (읽다가) | 사이클 종료 시 체계 점검 (diff 기준) |
| REPORT | `gates.drift` (READY 와 모순) | `metrics.domain_impact` (READY 와 공존) |

`project.md` 의 "에이전트 간 전달사항" (evaluator step 6) 은 **프로젝트 내부**
feature 간 전달용 — 팀 공유 도메인 지식과 별개 축이다.
````

- [ ] **Step 2: 마크다운 확인 및 커밋**

```bash
cd /Users/jay-p/Projects/deali-skills-plugin
git add dp-skills/skills/context/lifecycle/knowledge-sync.md
git commit -m "feat(lifecycle): knowledge-sync 프로토콜 신설 — 사이클 종료 시 도메인 지식 환류"
```

---

### Task 2: `verify-report-lint.py` — `metrics.domain_impact` 검증 (TDD)

**Files:**
- Modify: `dp-skills/tools/verify-report-lint.py` (모듈 docstring, 스키마 상수 ~L60, `validate()` 끝부분 ~L322)
- Test: `dp-skills/tests/tools/test_verify_report_lint.py` (클래스 추가)

- [ ] **Step 1: 실패하는 테스트 작성**

`tests/tools/test_verify_report_lint.py` 끝에 추가 (기존 `_MINIMAL` 헬퍼가 없으므로 블록을 직접 구성 — 기존 `ValidateRules` 의 fixture 사용 패턴과 동일하게 텍스트 기반):

```python
def _report(metrics_lines: str) -> str:
    """domain_impact 테스트용 최소 유효 REPORT (standard mode)."""
    return (
        "## VERIFICATION REPORT\n"
        "- status: READY\n"
        "- feature: #01 테스트\n"
        "- mode: standard\n"
        "- gates:\n"
        "  - requirements: pass — features/01.md\n"
        "  - tdd_evidence: skip — tdd:false\n"
        "  - capture_lockdown: skip — mode standard\n"
        "  - test_run: skip — 수동 시나리오\n"
        "  - scope: pass — 범위 내\n"
        "  - focus: skip — .focus.md 없음\n"
        "  - open_questions: skip — 섹션 없음\n"
        "  - drift: none\n"
        "- metrics:\n"
        f"{metrics_lines}"
        "- issues_to_fix:\n"
        "  - none\n"
        "- next: null\n"
    )


class DomainImpactRules(unittest.TestCase):
    def _codes(self, text, severity=None):
        violations, _ = m.lint(text)
        return [v.code for v in violations
                if severity is None or v.severity == severity]

    def test_detected_with_ready_is_not_error(self):
        text = _report("  - coverage: skip\n"
                       "  - domain_impact: detected — enum: 상태 추가 → scope/retail.md\n")
        self.assertEqual(self._codes(text, m.Violation.SEVERITY_ERROR), [])

    def test_invalid_enum_is_error(self):
        text = _report("  - coverage: skip\n"
                       "  - domain_impact: maybe — ???\n")
        self.assertIn("invalid_metric_value", self._codes(text))

    def test_absent_is_legacy_warn(self):
        text = _report("  - coverage: skip\n")
        self.assertIn("missing_metric_legacy",
                      self._codes(text, m.Violation.SEVERITY_WARN))
        self.assertNotIn("missing_metric_legacy",
                         self._codes(text, m.Violation.SEVERITY_ERROR))

    def test_none_and_skip_valid(self):
        for val in ("none", "skip"):
            text = _report(f"  - coverage: skip\n  - domain_impact: {val}\n")
            self.assertEqual(self._codes(text, m.Violation.SEVERITY_ERROR), [],
                             msg=f"value={val}")
```

- [ ] **Step 2: 실패 확인**

```bash
cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills
/opt/homebrew/bin/pytest tests/tools/test_verify_report_lint.py -k DomainImpact -v
```

Expected: FAIL — `invalid_metric_value`·`missing_metric_legacy` 코드 미존재 (AssertionError).

- [ ] **Step 3: 최소 구현**

`tools/verify-report-lint.py` 의 `LEGACY_OPTIONAL_GATES` 선언 아래에 추가:

```python
# metrics 항목 중 enum 검증 대상 (gate 아님 — status 판정 비영향).
# domain_impact (0.18.0): 사이클 변경분의 도메인 지식 영향. knowledge-sync.md SSOT.
# 부재 시 warn (도입 이전 REPORT 호환 — focus 의 missing_gate_legacy 패턴).
METRIC_ENUMS: dict[str, set[str]] = {
    "domain_impact": {"none", "detected", "skip"},
}
LEGACY_OPTIONAL_METRICS = {"domain_impact"}
```

`validate()` 의 rule 7 (mode ↔ gate skip 매핑) 뒤, `return v` 앞에 추가:

```python
    # 9. metrics enum 검증 (gate 아님 — status 와 무관, 형식만 본다)
    metrics = report.get("metrics", {})
    for metric_name, allowed in METRIC_ENUMS.items():
        if metric_name not in metrics:
            if metric_name in LEGACY_OPTIONAL_METRICS:
                v.append(Violation(
                    "missing_metric_legacy",
                    f"metrics 항목 부재: {metric_name} (도입 이전 REPORT 호환 — 신규 산출물은 포함 필수)",
                    Violation.SEVERITY_WARN,
                ))
            continue
        first = metrics[metric_name].split()[0] if metrics[metric_name] else ""
        if first not in allowed:
            v.append(Violation(
                "invalid_metric_value",
                f"metrics {metric_name} 값이 enum 외: {first} (허용: {sorted(allowed)})",
            ))
```

모듈 docstring 의 검증 목록에 1줄 추가: `- metrics.domain_impact enum (none/detected/skip) — 부재 시 warn (legacy 호환)`

- [ ] **Step 4: 전체 테스트 통과 확인**

```bash
/opt/homebrew/bin/pytest tests/tools/test_verify_report_lint.py -v
```

Expected: 기존 27 + 신규 4 = 31 PASS. (기존 valid fixture 들은 domain_impact 부재 → warn 만 발생 — error 없음 assertion 그대로 통과.)

- [ ] **Step 5: 커밋**

```bash
git add dp-skills/tools/verify-report-lint.py dp-skills/tests/tools/test_verify_report_lint.py
git commit -m "feat(tools): verify-report-lint 에 metrics.domain_impact 검증 추가"
```

---

### Task 3: `ag-evaluator.md` 배선 — 판정·REPORT·안내 블록

**Files:**
- Modify: `dp-skills/agents/ag-evaluator.md` (step 3 끝·step 7 템플릿·step 7 매핑 절)

- [ ] **Step 1: step 3 에 판정 블록 추가**

step 3 의 `[focus 게이트 — 항목별 반영 확인]` 블록 뒤에 추가:

```markdown
   **[도메인 지식 환류 판정]** 검토 완료 후 이번 변경 diff 를
   [`knowledge-sync.md`](${CLAUDE_PLUGIN_ROOT}/skills/context/lifecycle/knowledge-sync.md)
   § 판정 기준으로 분류한다 — `detected | none | skip`. detected 항목은
   `{유형}: {요약} → {대상 문서}` 형식으로 수집한다 (step 7 의
   `metrics.domain_impact` 과 chat 안내 블록에 사용). **gate 아님** — status
   판정에 영향을 주지 않으며, READY 와 detected 는 정상 공존한다.
   `workspace/{TEAM}/context/` 파일을 직접 Edit 하지 않는다 — 기록 주체는
   사용자 승인 후 메인 루프 (knowledge-sync.md § 기록).
```

- [ ] **Step 2: step 7 REPORT 템플릿의 metrics 에 라인 추가**

```
   - metrics:
     - coverage: {before%→after% / skip — coverage_command 미설정 또는 측정 실패}
     - domain_impact: none | detected | skip — {유형}: {요약} → {대상 문서} (항목 `;` 구분, skip 은 domain null)
```

step 7 의 `**metrics.coverage**` 설명 단락 옆에 추가:

```markdown
   **metrics.domain_impact** 는 참고 지표 (gate 아님). 판정 기준·기록 절차 SSOT:
   [`knowledge-sync.md`](${CLAUDE_PLUGIN_ROOT}/skills/context/lifecycle/knowledge-sync.md).
   `detected` 시 chat 응답 말미 (REPORT 직후) 에 knowledge-sync.md § 수동 사이클의
   "도메인 지식 환류 제안" 안내 블록을 출력한다 — 기록 여부는 사용자가 결정.
```

- [ ] **Step 3: 커밋**

```bash
git add dp-skills/agents/ag-evaluator.md
git commit -m "feat(agents): evaluator 에 도메인 지식 환류 판정·domain_impact 보고 추가"
```

---

### Task 4: 예시·판정 축 동기화 (`messages.md`·`guardrails.md`)

**Files:**
- Modify: `dp-skills/skills/context/shared/messages.md` (`verification_report_example` 2건, ~L171·L194)
- Modify: `dp-skills/skills/context/shared/guardrails.md` (§ 기본 판정 축, drift 축 뒤)

- [ ] **Step 1: messages.md READY 예시의 metrics 갱신**

READY 예시 (READY + detected 공존이 정상임을 보여주는 교육적 케이스):

```
- metrics:
  - coverage: skip
  - domain_impact: detected — enum: 취소사유 CANCEL_REASON 추가 → scope/order.md
```

NOT_READY 예시:

```
- metrics:
  - coverage: skip
  - domain_impact: none
```

- [ ] **Step 2: guardrails.md § 기본 판정 축에 항목 추가**

`### drift` 섹션 뒤:

```markdown
### domain_impact (참고 축 — status 비영향)

이번 사이클 변경분이 도메인 지식 문서 (scope·enums·boundaries·MANIFEST) 에
반영할 신규·변경 지식을 만들었는지. 증거는 `{유형}: {요약} → {대상 문서}`.
판정·기록 절차 SSOT 는 [knowledge-sync.md](../lifecycle/knowledge-sync.md) —
drift (기존 문서 오류) 와 별개 축이며 READY 와 detected 는 공존한다.
```

- [ ] **Step 3: lint 재실행으로 예시 회귀 확인 후 커밋**

```bash
/opt/homebrew/bin/pytest tests/tools/test_verify_report_lint.py -v
git add dp-skills/skills/context/shared/messages.md dp-skills/skills/context/shared/guardrails.md
git commit -m "docs(shared): REPORT 예시·판정 축에 domain_impact 반영"
```

---

### Task 5: run-cycle 누적·일괄 질의

**Files:**
- Modify: `dp-skills/skills/run-cycle/SKILL.md` (§2.5·§3·§4·Do-NOT)

- [ ] **Step 1: §2.5 evaluator 처리에 누적 규칙 추가**

`READY →` 항목을 다음으로 교체:

```markdown
- `READY` → `project.md` 해당 feature 체크박스 `[x]` 갱신. REPORT 의
  `metrics.domain_impact` 가 `detected` 면 항목을 **메모리에 누적**하고
  (질의·정지 없음 — green 자동 진행 유지) → **자동 다음 feature**(2.1 로).
- `NOT_READY`·`block` → **⏸ §3 정지**.
```

- [ ] **Step 2: §3 정지 정책·§4 큐 완료에 일괄 질의 추가**

§3 끝에:

```markdown
- 누적된 domain_impact 항목이 있으면 정지 보고에 "도메인 지식 환류 제안
  (누적)" 표를 함께 출력한다 — 처리 여부는 사용자 결정
  ([knowledge-sync.md](../context/lifecycle/knowledge-sync.md) § run-cycle).
```

§4 를 다음으로 교체:

```markdown
## 4. 큐 완료

모든 feature READY → 완료 요약 출력 (처리한 feature 목록·각 verdict).
누적된 domain_impact 항목이 있으면 "도메인 지식 환류 제안 (누적)" 표를
출력하고 기록 여부를 질의한다. 승인 시
[knowledge-sync.md](../context/lifecycle/knowledge-sync.md) § 기록 절차를
수행한다 (항목 5건 이상이면 `/dp-skills:discover` 재실행 권장).
```

- [ ] **Step 3: Do-NOT 에 1줄 추가**

```markdown
- feature 마다 domain_impact 질의로 정지 금지 — 누적 후 큐 종료 시 일괄 질의.
```

- [ ] **Step 4: 커밋**

```bash
git add dp-skills/skills/run-cycle/SKILL.md
git commit -m "feat(run-cycle): domain_impact 누적·큐 종료 시 일괄 환류 질의"
```

---

### Task 6: drift-protocol 상호 링크

**Files:**
- Modify: `dp-skills/skills/context/lifecycle/drift-protocol.md` (도입부)

- [ ] **Step 1: 경계 안내 1줄 추가**

도입 단락 뒤에:

```markdown
> **경계:** 본 프로토콜은 "기존 문서가 실제와 다름 (우연 발견)" 을 다룬다.
> 이번 사이클 변경이 만든 **신규·변경 지식의 환류** (사이클 종료 시 체계 점검)
> 는 [knowledge-sync.md](knowledge-sync.md) 가 담당한다.
```

- [ ] **Step 2: 커밋**

```bash
git add dp-skills/skills/context/lifecycle/drift-protocol.md
git commit -m "docs(lifecycle): drift-protocol 에 knowledge-sync 경계 링크 추가"
```

---

### Task 7: 버전 bump·docs 동기화·최종 검증

**Files:**
- Modify: `dp-skills/.claude-plugin/plugin.json` (`"version": "0.17.1"` → `"0.18.0"` — minor: 기능 추가)

- [ ] **Step 1: 버전 bump**

`dp-skills/.claude-plugin/plugin.json` 의 `"version": "0.17.1"` 을 `"version": "0.18.0"` 으로 Edit.

- [ ] **Step 2: docs reference 동기화 + 전체 테스트**

```bash
cd /Users/jay-p/Projects/deali-skills-plugin
python3 dp-skills/tools/docs_build.py
/opt/homebrew/bin/pytest dp-skills/tests/ -q
```

Expected: docs drift 없음 (변경 발생 시 산출물 함께 staging), 전체 테스트 PASS.
(주의: `python3 -m pytest` 불가 환경 — `/opt/homebrew/bin/pytest` 사용.)

- [ ] **Step 3: 커밋 + PR**

```bash
git add dp-skills/.claude-plugin/plugin.json
git commit -m "chore(release): bump 0.18.0"
```

PR 은 `/dp-skills:pr` 또는 `gh pr create --base main` (squash 머지, pre-push hook 우회 금지).

---

## Self-Review 체크 결과

- **요구 충족**: "evaluator 종료 후" (Task 3 step 1 — 검토 완료 후 판정) · "물어보고" (Task 3 step 2 안내 블록 + Task 5 일괄 질의 — 자동 Edit 없음) · "도메인 지식 업데이트 진행" (knowledge-sync.md § 기록 — 승인 후 메인 루프 Edit / discover 재실행) 모두 태스크로 매핑됨.
- **기존 산출물 호환**: lint 는 legacy warn (Task 2 step 3) — nimda 의 기존 eval 19건+ 이 FAIL 로 바뀌지 않음. 기존 valid fixture 무수정 통과 (Task 2 step 4 에서 확인).
- **타입·어휘 일관성**: enum `none | detected | skip` 이 Task 1(SSOT)·2(lint)·3(템플릿)·4(예시) 에서 동일. evidence 형식 `{유형}: {요약} → {대상 문서}` 동일.
