# Open Questions 게이트 단일화 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** feature 명세 `## Open Questions` (a)~(d) 의 미해결 항목 판정 기준을 단일 SSOT 문서(`skills/context/lifecycle/oq-gate.md`)로 추출하고, `plan-validate.py` 가 "미해결 OQ ↔ plan 처리 마커" 를 기계 검증해 planner·generator·evaluator 세 에이전트가 동일 판정을 공유하게 한다. generator 의 막다른 흐름(종료된 planner 에 재확인 요청)은 명시적 에스컬레이션 경로로 해소한다.

**Architecture:** ① 판정 매트릭스·마커 어휘·에스컬레이션 경로를 `oq-gate.md` 로 추출하고 wrapper 3종은 참조만 남긴다. ② `plan-validate.py` 가 plan 경로에서 feature 파일을 자동 유도(`NN-{slug}.plan.md` → `NN-{slug}.md`)해 `doctor/integrity.py` 의 기존 OQ 파서로 미해결 항목을 읽고, plan 본문의 처리 마커와 대조해 `oq` 필드로 보고 + 위반 시 invalid(exit 1). planner step 6(저장 직후)·generator step 2(Read 직전)가 **이미 같은 도구를 호출**하므로 추가 훅 없이 동일 판정이 강제된다.

**Tech Stack:** Python 3 표준 라이브러리(`doctor.integrity` 재사용 — `oq-eval.py` 의 sys.path 패턴), unittest(기존 `test_plan_validate.py` 컨벤션), markdown wrapper/SSOT 문서.

---

## 배경 — 분산·상이한 3곳의 판정 기준

| 위치 | 현행 기준 | 문제 |
|---|---|---|
| `agents/ag-planner.md` ~136행 | 카테고리별 표: (a) 추가 Read, (b) discover 권고 + "그냥 진행" 시 plan 에 "산출물 부재 상태에서 추정 구현" 명시, (c) 사용자 질의 → 범위 제외 + plan TODO 마킹, (d) 반드시 사용자 질의 | (c) 의 "범위 제외" 경로가 다른 두 에이전트 기준에 없음 |
| `agents/ag-generator.md` 52-54행 | 미해결 `- [ ]` 가 **카테고리 불문** 있으면 구현 중단, "Planner 에 재확인을 요청하세요" 안내 후 종료. 예외는 plan 의 "산출물 부재 상태에서 추정 구현" 문구뿐 | ① planner 는 이미 종료된 인스턴스 — 재확인 경로가 막다른 흐름. ② planner 의 합법 경로인 (c) "범위 제외 + TODO" plan 을 반려한다 (문구 불일치) |
| `agents/ag-evaluator.md` step 3 | (d) 임의결정 → Major, (b)/(c) 미해결 + plan "추정 구현" 미명시 → Major | (a) 미해결은 기준에 없음(무조건 통과), (c) "범위 제외" 경로 인식 불가 |

어긋남 시나리오: planner 가 (c) 를 "범위 제외 + TODO" 로 합법 처리 → generator 가 "추정 구현" 문구 부재로 반려 → 사용자가 종료된 planner 에 재확인할 방법이 불명확. 또는 (a) 미해결 잔존 → generator 반려 / evaluator 통과 로 엇갈림.

## 설계 결정

**채택: ① SSOT 문서 + ② plan-validate 기계 검증 (조합).**

- ① `skills/context/lifecycle/oq-gate.md` — lifecycle/ 의 기존 계약 문서 패턴(plan-schema.md·state-schema.md)과 동일 위치·성격. 판정 기준의 "의미" SSOT.
- ② `plan-validate.py` 확장 — 판정의 "강제" 메커니즘. planner(step 6)·generator(step 2)가 이미 호출하는 도구라 추가 배선 없이 fail-closed. `doctor/integrity.py` 의 `_extract_oq_section`/`_parse_oq_categories` 재사용(`oq-eval.py` 선례).

**기각: ③ orchestrate-load.py OQ 상태 JSON 노출.**

- orchestrate-load 는 팀·프로젝트 수준 컨텍스트만 알고 "이번 사이클의 feature" 를 특정할 인자가 없다 (project_phase 와 달리 OQ 는 feature 단위 상태).
- planner 시점에는 plan 이 존재하지 않아 "plan 마커 검증" 자체가 불가능 — 게이트의 핵심은 feature-plan 짝 검증이며 그 자리는 plan-validate.
- 전체 features 의 OQ 요약 노출은 `doctor --oq-stats` 와 중복 (YAGNI).

### 마커 어휘 (기계 검증 계약)

미해결 `- [ ]` 항목이 있는 카테고리마다 plan 본문(fenced 코드블록 제외)에 처리 마커가 필요하다:

1. **정밀 마커 (권장)**: 카테고리 키(`(a)`~`(d)`)와 마커 어휘가 **같은 라인**에 등장.
   - `추정 구현` — 산출물·spec 없이 추정으로 구현 (사용자 "그냥 진행" 명시가 전제). 권장 전체 문구: "산출물 부재 상태에서 추정 구현".
   - `범위 제외` (또는 `범위에서 제외`) — 해당 인터페이스·결정 영역을 이번 구현 범위에서 제외.
2. **포괄 마커 (하위 호환)**: 키 없이 "산출물 부재 상태에서 추정 구현" 전체 문구 — 기존 운영 plan 어휘. **(d) 를 제외한** 모든 미해결 카테고리를 커버.
3. **(d) 특칙**: 비즈니스 결정 영역은 `추정 구현` 불인정 (임의 결정 금지의 기계적 표현). `(d)` 키 + `범위 제외` 마커(사용자가 결정 보류를 명시한 경우) 또는 해결(`[x]`)만 통과.

### 에스컬레이션 경로 (generator 막다른 흐름 해소)

게이트 실패 시 "Planner 에 재확인 요청" 이라는 모호한 안내 대신, **planner 는 stateless 재호출 가능한 단계**임을 전제로 두 갈래를 사용자에게 제시:

- **① plan 보완**: 오케스트레이터가 `@ag-planner` 를 재호출 — planner 가 oq-gate 매트릭스대로 항목을 해결/마킹해 plan 갱신 → plan-validate 재통과 후 `@ag-generator` 재진입.
- **② (d) 직접 해결**: 사용자가 결정을 답하면 메인 세션이 feature 파일 해당 항목을 `[x] {항목} — 결정: {내용}` 으로 Edit 한 뒤 중단된 단계를 재호출.

자동 라우팅 금지 — 항상 사용자 보고 후 선택 (기존 evaluator "직접 planner 호출 금지 — 오케스트레이션이 라우팅" 원칙과 일치).

### 비스코프

- pilot 플러그인 (별도 트리) — 대상 아님.
- analyze/create-feature 의 OQ **생성** 규칙 — 변경 없음 (생성 SSOT 는 그쪽 유지).
- evaluator REPORT 의 `open_questions` gate 라인 형식 — verify-report-lint 계약 불변.
- 기존 운영 plan 중 표준 어휘 없이 작성된 (c) "범위 제외" 류 plan 은 재검증 시 invalid 가 될 수 있다 — fail-closed 의 의도된 결과이며 planner 재호출로 보완 (oq-gate.md 에 명시).

---

### Task 1: `plan-validate.py` OQ 게이트 검증 (TDD)

**Files:**
- Modify: `dp-skills/tools/plan-validate.py`
- Test: `dp-skills/tests/tools/test_plan_validate.py`

- [ ] **Step 1: 실패하는 테스트 작성**

`test_plan_validate.py` 말미(`if __name__` 위)에 추가:

```python
FEATURE_WITH_OQ = """# #1 주문 검증

## 요구사항

- 주문 금액 검증

## Open Questions

### (a) 같은 도메인 추가 read 필요
- (없음)

### (b) cross-domain 산출물 부재
- [ ] 결제 도메인 산출물 부재

### (c) 외부 시스템 spec 부재
- [x] PG API spec 확보됨

### (d) 비즈니스 결정 영역
- (없음)
"""

FEATURE_ALL_RESOLVED = """# #1 주문 검증

## Open Questions

### (a) 같은 도메인 추가 read 필요
- (없음)

### (b) cross-domain 산출물 부재
- [x] 결제 도메인 산출물 확보

### (c) 외부 시스템 spec 부재
- (없음)

### (d) 비즈니스 결정 영역
- (없음)
"""

FEATURE_WITH_D_OPEN = """# #1 주문 검증

## Open Questions

### (a) 같은 도메인 추가 read 필요
- (없음)

### (b) cross-domain 산출물 부재
- (없음)

### (c) 외부 시스템 spec 부재
- (없음)

### (d) 비즈니스 결정 영역
- [ ] 부분 환불 시 수수료 부담 주체
"""


class DeriveFeaturePath(unittest.TestCase):
    def test_plan_md(self):
        p = m.derive_feature_path(Path("/w/features/01-foo.plan.md"))
        self.assertEqual(p, Path("/w/features/01-foo.md"))

    def test_plan_rN_md(self):
        p = m.derive_feature_path(Path("/w/qa/KEY-1.plan.r2.md"))
        self.assertEqual(p, Path("/w/qa/KEY-1.md"))

    def test_non_plan_returns_none(self):
        self.assertIsNone(m.derive_feature_path(Path("/w/features/01-foo.md")))


class OpenQuestionsGate(unittest.TestCase):
    def _validate(self, plan_text: str, feature_text: str | None,
                  feature_name: str = "01-foo.md") -> dict:
        with tempfile.TemporaryDirectory() as td:
            plan = Path(td) / "01-foo.plan.md"
            plan.write_text(plan_text, encoding="utf-8")
            if feature_text is not None:
                (Path(td) / feature_name).write_text(
                    feature_text, encoding="utf-8"
                )
            return m.validate(plan, "standard")

    def test_no_feature_file_skips(self):
        r = self._validate(VALID_STANDARD, None)
        self.assertTrue(r["valid"])
        self.assertFalse(r["oq"]["checked"])

    def test_feature_without_oq_section_skips(self):
        r = self._validate(VALID_STANDARD, "# #1 주문 검증\n\n본문만.\n")
        self.assertTrue(r["valid"])
        self.assertFalse(r["oq"]["checked"])

    def test_all_resolved_passes(self):
        r = self._validate(VALID_STANDARD, FEATURE_ALL_RESOLVED)
        self.assertTrue(r["valid"])
        self.assertTrue(r["oq"]["checked"])
        self.assertEqual(r["oq"]["errors"], [])

    def test_unresolved_b_without_marker_fails(self):
        r = self._validate(VALID_STANDARD, FEATURE_WITH_OQ)
        self.assertFalse(r["valid"])
        self.assertTrue(any("(b)" in e for e in r["oq"]["errors"]))

    def test_unresolved_b_with_keyed_assume_marker_passes(self):
        plan = VALID_STANDARD + (
            "\n### Open Questions 처리\n\n"
            "- (b) 결제 도메인: 추정 구현 — 사용자 승인 (그냥 진행)\n"
        )
        r = self._validate(plan, FEATURE_WITH_OQ)
        self.assertTrue(r["valid"])

    def test_unresolved_b_with_keyed_exclude_marker_passes(self):
        plan = VALID_STANDARD + (
            "\n### Open Questions 처리\n\n"
            "- (b) 결제 도메인 연동부: 구현 범위에서 제외 (TODO 마킹)\n"
        )
        r = self._validate(plan, FEATURE_WITH_OQ)
        self.assertTrue(r["valid"])

    def test_legacy_blanket_marker_covers_b(self):
        plan = VALID_STANDARD + "\n산출물 부재 상태에서 추정 구현.\n"
        r = self._validate(plan, FEATURE_WITH_OQ)
        self.assertTrue(r["valid"])

    def test_legacy_blanket_marker_not_cover_d(self):
        plan = VALID_STANDARD + "\n산출물 부재 상태에서 추정 구현.\n"
        r = self._validate(plan, FEATURE_WITH_D_OPEN)
        self.assertFalse(r["valid"])
        self.assertTrue(any("(d)" in e for e in r["oq"]["errors"]))

    def test_d_with_assume_marker_fails(self):
        plan = VALID_STANDARD + "\n- (d) 수수료 부담 주체: 추정 구현\n"
        r = self._validate(plan, FEATURE_WITH_D_OPEN)
        self.assertFalse(r["valid"])

    def test_d_with_exclude_marker_passes(self):
        plan = VALID_STANDARD + (
            "\n- (d) 수수료 부담 주체: 범위 제외 — 사용자가 결정 보류 명시\n"
        )
        r = self._validate(plan, FEATURE_WITH_D_OPEN)
        self.assertTrue(r["valid"])

    def test_marker_in_fenced_code_ignored(self):
        plan = VALID_STANDARD + (
            "\n```\n- (b) 결제 도메인: 추정 구현\n```\n"
        )
        r = self._validate(plan, FEATURE_WITH_OQ)
        self.assertFalse(r["valid"])

    def test_cli_exit_1_on_oq_violation(self):
        with tempfile.TemporaryDirectory() as td:
            plan = Path(td) / "01-foo.plan.md"
            plan.write_text(VALID_STANDARD, encoding="utf-8")
            (Path(td) / "01-foo.md").write_text(
                FEATURE_WITH_OQ, encoding="utf-8"
            )
            proc = subprocess.run(
                ["python3", str(TOOL_PATH), str(plan), "--mode", "standard"],
                capture_output=True, text=True,
            )
            self.assertEqual(proc.returncode, 1)
            self.assertIn("(b)", proc.stderr)
```

- [ ] **Step 2: 실패 확인**

Run: `cd dp-skills && pytest tests/tools/test_plan_validate.py -q`
Expected: `AttributeError: ... derive_feature_path` 류로 신규 클래스 전건 FAIL, 기존 테스트 전건 PASS.

- [ ] **Step 3: 구현**

`plan-validate.py` 에 추가 (docstring 의 작동 흐름·JSON 계약도 갱신):

```python
# (파일 상단 import 아래) doctor.integrity 의 OQ 파서 재사용 — oq-eval.py 와 동일 패턴
_TOOLS_DIR = Path(__file__).resolve().parent
if str(_TOOLS_DIR) not in sys.path:
    sys.path.insert(0, str(_TOOLS_DIR))

from doctor.integrity import (  # noqa: E402
    OQ_CATEGORY_KEYS,
    _extract_oq_section,
    _parse_oq_categories,
)

# Open Questions 게이트 — 스펙: skills/context/lifecycle/oq-gate.md
PLAN_SUFFIX_RE = re.compile(r"^(?P<stem>.+)\.plan(?:\.r\d+)?\.md$")
OQ_MARKER_ASSUME = "추정 구현"
OQ_MARKER_EXCLUDE_RE = re.compile(r"범위(?:에서)?\s*제외")
OQ_LEGACY_BLANKET = "산출물 부재 상태에서 추정 구현"


def derive_feature_path(plan_path: Path) -> Path | None:
    """plan 경로 → 대응 feature 파일 경로. plan 명명 규약 외 파일이면 None.

    `NN-{slug}.plan.md` → `NN-{slug}.md`, `qa/{KEY}.plan.r{N}.md` → `qa/{KEY}.md`.
    """
    m_ = PLAN_SUFFIX_RE.match(plan_path.name)
    if not m_:
        return None
    return plan_path.with_name(m_.group("stem") + ".md")


def check_open_questions(plan_text: str, feature_path: Path | None) -> dict:
    """미해결 OQ ↔ plan 처리 마커 대조. 스펙: oq-gate.md § 마커 어휘.

    Returns:
        {"checked": bool, "feature_file": str|None,
         "unresolved": {cat: [item, ...]}, "errors": [str, ...]}
    """
    out: dict = {
        "checked": False,
        "feature_file": None,
        "unresolved": {},
        "errors": [],
    }
    if feature_path is None or not feature_path.is_file():
        return out
    out["feature_file"] = str(feature_path)
    try:
        feature_text = feature_path.read_text(encoding="utf-8")
    except Exception:
        return out
    section = _extract_oq_section(feature_text)
    if section is None:
        return out

    out["checked"] = True
    parsed = _parse_oq_categories(section)
    unresolved = {
        cat: parsed[cat]["open"]
        for cat in OQ_CATEGORY_KEYS
        if parsed[cat]["open"]
    }
    out["unresolved"] = unresolved
    if not unresolved:
        return out

    # plan 본문에서 마커 라인 수집 (fenced 코드블록 제외)
    lines = plan_text.splitlines()
    mask = _fenced_mask(lines)
    body_lines = [ln for i, ln in enumerate(lines) if not mask[i]]
    has_blanket = any(OQ_LEGACY_BLANKET in ln for ln in body_lines)

    def keyed_marker(cat: str, allow_assume: bool) -> bool:
        for ln in body_lines:
            if cat not in ln:
                continue
            if OQ_MARKER_EXCLUDE_RE.search(ln):
                return True
            if allow_assume and OQ_MARKER_ASSUME in ln:
                return True
        return False

    for cat, items in unresolved.items():
        if cat == "(d)":
            # 비즈니스 결정 영역 — 추정 구현 불인정, 포괄 마커 불인정
            if not keyed_marker(cat, allow_assume=False):
                out["errors"].append(
                    f"(d) 미해결 {len(items)}건 — 사용자 결정 필요. "
                    "plan 진행은 `(d) ...: 범위 제외` 마커(사용자 결정 보류 "
                    "명시)로만 가능 (스펙: oq-gate.md)"
                )
        else:
            if not keyed_marker(cat, allow_assume=True) and not has_blanket:
                out["errors"].append(
                    f"{cat} 미해결 {len(items)}건 — plan 에 처리 마커 없음. "
                    f"`{cat} ...: 추정 구현` 또는 `{cat} ...: 범위 제외` "
                    "라인 필요 (스펙: oq-gate.md)"
                )
    return out
```

`validate()` 끝부분(`return result` 직전)에 배선:

```python
    oq = check_open_questions(text, derive_feature_path(plan_path))
    result["oq"] = oq
    if oq["errors"]:
        result["valid"] = False
```

`format_human_summary()` 에 추가:

```python
    for oerr in result.get("oq", {}).get("errors", []):
        lines.append(f"  - open questions: {oerr}")
```

주의: `validate()` 의 빈 파일 early-return 경로(`result` 에 `oq` 키 부재)는 `format_human_summary` 가 `.get("oq", {})` 로 방어한다. early-return 분기에도 `"oq": {"checked": False, ...}` 를 포함하면 JSON 계약이 균일해진다 — `result` 초기 템플릿에 `"oq": {"checked": False, "feature_file": None, "unresolved": {}, "errors": []}` 를 넣고 끝에서 덮어쓴다.

- [ ] **Step 4: 테스트 통과 확인**

Run: `cd dp-skills && pytest tests/tools/test_plan_validate.py -q`
Expected: 전건 PASS (기존 + 신규).

- [ ] **Step 5: 전체 테스트 회귀 확인**

Run: `cd dp-skills && pytest tests/tools -q`
Expected: 전건 PASS (베이스라인 460 + 신규).

- [ ] **Step 6: 커밋**

```bash
git add tools/plan-validate.py tests/tools/test_plan_validate.py
git commit -m "feat(tools): plan-validate 에 Open Questions 게이트 기계 검증 추가"
```

### Task 2: `oq-gate.md` SSOT 문서 신설

**Files:**
- Create: `dp-skills/skills/context/lifecycle/oq-gate.md`

- [ ] **Step 1: 문서 작성** — 판정 매트릭스(카테고리 × phase)·마커 어휘(Task 1 구현과 1:1)·에스컬레이션 경로·generator TODO 주석 규약·변경 시 동기화 목록을 본 PLAN 의 "설계 결정" 절 내용대로 작성. 카테고리 정의·생성 기준은 `create-feature/SKILL.md` 가 SSOT 임을 명시(중복 금지, 링크만).

- [ ] **Step 2: 커밋**

```bash
git add skills/context/lifecycle/oq-gate.md
git commit -m "docs(context): Open Questions 게이트 판정 SSOT 문서 신설"
```

### Task 3: wrapper 3종 게이트 블록 교체

**Files:**
- Modify: `dp-skills/agents/ag-planner.md` (~136-143행 — 게이트 문단 + 표)
- Modify: `dp-skills/agents/ag-generator.md` (52-54행 — 인계 확인 블록)
- Modify: `dp-skills/agents/ag-evaluator.md` (step 3, 74-77행 — 게이트 블록)

- [ ] **Step 1: ag-planner** — 카테고리 표를 oq-gate.md 참조 1문단으로 교체. "미해결 잔존 항목은 plan 에 카테고리 키 + 처리 마커 명시 필수 — step 6 plan-validate 가 강제, 어휘는 oq-gate.md SSOT" 명시.
- [ ] **Step 2: ag-generator** — 직접 판정 제거. "plan-validate(기존 step 2 호출)가 OQ 게이트 포함 — `oq.errors` 실패 시 구현 미시작, oq-gate.md § 에스컬레이션 경로 두 갈래(① `@ag-planner` 재호출로 plan 보완 재진입 ② (d) 는 사용자 직접 결정 → feature `[x]` 갱신 후 재호출)를 사용자에게 안내 후 종료" 로 교체. `추정 구현` 항목의 TODO 주석 규약 유지(oq-gate.md 참조).
- [ ] **Step 3: ag-evaluator** — step 3 게이트를 oq-gate.md 기준으로 교체: (d) 임의 결정 → Major(불변) / 미해결 카테고리 + plan 마커 부재 → Major(4 카테고리 일관) / `추정 구현` 항목 TODO 주석 부재 → Minor / `[x]` 항목 육안 검증(불변). 보조 도구로 plan-validate 의 `oq` 필드 언급.
- [ ] **Step 4: 커밋**

```bash
git add agents/ag-planner.md agents/ag-generator.md agents/ag-evaluator.md
git commit -m "feat(agents): OQ 게이트 판정을 oq-gate.md SSOT 참조로 단일화"
```

### Task 4: 계약 문서·README 동기화 + docs_build + 버전 bump

**Files:**
- Modify: `dp-skills/skills/context/lifecycle/plan-schema.md` (OQ 게이트 검사 절 + JSON 계약 + 동기화 목록)
- Modify: `dp-skills/skills/context/lifecycle/INDEX.md` (상황별 진입 문서 표 + 문서 성격 표 + 폴더 구조 트리에 oq-gate.md)
- Modify: `dp-skills/README.md` (395행 plan-validate 도구 표 행에 OQ 게이트 언급)
- Modify: `dp-skills/.claude-plugin/plugin.json` (0.16.13 → 0.16.14)

- [ ] **Step 1: plan-schema.md** — `## Open Questions 게이트 검사` 절 추가(모든 모드 공통, feature 파일 자동 유도·skip 조건·oq-gate.md 링크), JSON 예시에 `oq` 필드 반영, `## 변경 시` 에 oq-gate.md 추가.
- [ ] **Step 2: INDEX.md·README.md** — 라우팅 표·트리·도구 표 갱신.
- [ ] **Step 3: plugin.json bump** — `0.16.14`.
- [ ] **Step 4: docs_build 재생성 + 검증**

Run: `python3 dp-skills/tools/docs_build.py && python3 dp-skills/tools/docs_build.py --check`
Expected: wrote N files / drift 없음 (exit 0).

- [ ] **Step 5: 전체 테스트 + 커밋**

Run: `cd dp-skills && pytest tests/tools -q` → 전건 PASS.

```bash
git add -A
git commit -m "chore(release): bump 0.16.14 — OQ 게이트 단일화 문서·동기화"
```

### Task 5: PR

- [ ] **Step 1:** 본 PLAN 파일 커밋 포함 확인 → push → `gh pr create --base main` (squash 머지 전제, 제목 `feat: Open Questions 게이트 단일화 (0.16.14)`).

## Self-Review 체크 결과

- 스펙 커버: 3곳 분산 기준 단일화(Task 2+3), 기계 강제(Task 1), 막다른 흐름 해소(Task 3 Step 2 + oq-gate.md 에스컬레이션 경로) — 충족.
- (c) "범위 제외" 경로가 세 에이전트 + 도구 모두에서 동일하게 인식됨 — 어긋남 시나리오 해소.
- 타입·명칭 일관: `oq` JSON 필드, `derive_feature_path`/`check_open_questions`, 마커 어휘 상수 — Task 간 동일.
