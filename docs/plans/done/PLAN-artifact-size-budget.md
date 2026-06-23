# 사이클 산출물 분량 가드 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** plan/critic 산출물의 비대(이력 누적·회차 잔재)를 막는다 — ① plan-validate 분량 WARN(기계 게이트), ② planner/critic 의 "최신 확정 상태만 보존" 원칙 명문화, ③ 사이클 내 대형 개정의 r{N} 분리 규약 확장.

**Architecture:** 강제력의 핵심은 plan-validate (ag-planner 절차 6 의 통과-전-진행-금지 게이트) 에 추가하는 비차단 WARN — exit code 는 불변, `warnings` 필드 + stderr 로 보고한다. 지침(②)은 무엇을 지울지 정의하고, WARN(①)은 안 지웠을 때 잡아낸다. 임계 스펙 SSOT 는 plan-schema.md, 명명 규약 확장(③) SSOT 는 qa/SKILL.md §4-1.

**Tech Stack:** Python 3 표준 라이브러리 (plan-validate), unittest + tempfile (기존 테스트 컨벤션), markdown 명세.

---

## 배경 (실데이터 근거 — nimda Presend-Admin, 2026-06-12 실측)

| 파일 | 크기 | 추정 토큰 | 비고 |
| --- | --- | --- | --- |
| QAPRJ-5394.plan.md | 198KB / 135,265자 | ≈65k | "2회차 개정"+"추가 수정"+"3·4차 갱신" 한 파일 누적 |
| QAPRJ-5394.plan.critic.md | 148KB / 96,519자 | ≈50k | 챌린지마다 "1회차 대비 정정·보강" 이력 서술 |
| (규약 이후 최대) QAPRJ-5438.plan.r1.md | 38KB | ≈12k | r{N} 분리(0.16.8) 이후 정상 범위 |

- 두 파일만 ≈115k 토큰 — 후속 에이전트(critic·planner 재호출·generator·evaluator)가 매 라운드 전문을 Read 하므로 200k 윈도우 절반 이상을 코드 읽기 전에 선소모. 반복 라운드의 input 재독이 비용 주범.
- **지침만으로는 부족함이 증명됨**: critic 에는 "본문 챌린지는 최신 상태만 보존" 규칙(ag-planner-critic.md 절차 5)이 이미 있었는데도 148KB 가 됐다 → 기계 게이트(①) 필요.
- 최장 라인 1,606자 — Read 툴 라인 절단(2,000자) 위험 구간 근접.
- 토큰 경제 가드의 비대칭: orchestrate-load 는 컨텍스트 문서에 가드가 있으나(보조 문서 hint-only, MAX_BOUNDARY_DOCS) 사이클 산출물에는 전무.

## 설계 결정

1. **WARN 비차단** — 복잡한 결함의 디테일(file:line 인용·SQL 분석)은 generator 인계 품질의 원천. 단순 길이 fail 은 디테일부터 깎는 역효과 → WARN 으로 시작, 재발 시 임계 강화는 후속.
2. **임계는 문자 수 기준** (줄 수 아님) — 5394 는 500자 초과 라인이 112개라 줄 수로는 과소 감지. `SIZE_WARN_CHARS = 30_000` (≈토큰 10~15k, 규약 이후 정상 plan 20~27k자 바로 위), `LINE_WARN_CHARS = 1_500` (Read 절단 2,000자 안전 마진).
3. **지울 대상의 정의** = 회차 이력 잔재 (`N차 갱신` 헤더, `[C# 해소]`·`1회차 대비 정정` 주석, 기각된 대안의 장문 사유). 이력 SSOT 는 critic 합의 표 — plan/critic 본문이 아니다.

---

### Task 0: 작업 브랜치

- [ ] **Step 1: 브랜치 생성**

```bash
cd /Users/jay-p/Projects/deali-skills-plugin && git checkout -b feat/artifact-size-budget
```

---

### Task 1: plan-validate 분량 WARN (TDD)

**Files:**
- Modify: `dp-skills/tools/plan-validate.py`
- Test: `dp-skills/tests/tools/test_plan_validate.py` (신규 클래스 추가)

- [ ] **Step 1: 실패하는 테스트 작성**

`tests/tools/test_plan_validate.py` 의 `ValidateIntegration` 클래스 위에 신규 클래스 추가 (모듈 전역 `m`, `VALID_STANDARD` fixture 재사용 — 파일 상단에 이미 정의돼 있음):

```python
class SizeWarnings(unittest.TestCase):
    """분량 가드 — WARN 비차단. 스펙: plan-schema.md § 분량 가드.

    배경: nimda QAPRJ-5394 plan 198KB(135k자) — 회차 이력 누적으로 비대,
    후속 에이전트가 매 라운드 전문 재로딩. 임계 초과는 WARN 만 (exit 불변).
    """

    def test_small_plan_no_warnings(self):
        self.assertEqual(m.size_warnings(VALID_STANDARD), [])

    def test_oversize_plan_warns(self):
        # 짧은 라인 다수로 30,000자 초과 생성 (라인 경고와 분리)
        text = VALID_STANDARD + "\n" + "\n".join(["- 항목 설명 텍스트"] * 3000)
        warnings = m.size_warnings(text)
        self.assertEqual(len(warnings), 1)
        self.assertIn("분량", warnings[0])
        self.assertIn("30,000", warnings[0])

    def test_long_line_warns(self):
        text = VALID_STANDARD + "\n" + ("가" * 1600)
        warnings = m.size_warnings(text)
        self.assertEqual(len(warnings), 1)
        self.assertIn("1,500", warnings[0])
        self.assertIn("절단", warnings[0])

    def test_validate_carries_warnings_but_stays_valid(self):
        text = VALID_STANDARD + "\n" + "\n".join(["- 항목 설명 텍스트"] * 3000)
        with tempfile.TemporaryDirectory() as td:
            p = Path(td) / "01-big.plan.md"
            p.write_text(text, encoding="utf-8")
            result = m.validate(p, "standard")
        self.assertTrue(result["valid"])
        self.assertEqual(len(result["warnings"]), 1)

    def test_valid_plan_has_empty_warnings_field(self):
        with tempfile.TemporaryDirectory() as td:
            p = Path(td) / "01-small.plan.md"
            p.write_text(VALID_STANDARD, encoding="utf-8")
            result = m.validate(p, "standard")
        self.assertTrue(result["valid"])
        self.assertEqual(result["warnings"], [])

    def test_cli_exit_0_with_warning_on_stderr(self):
        text = VALID_STANDARD + "\n" + "\n".join(["- 항목 설명 텍스트"] * 3000)
        with tempfile.TemporaryDirectory() as td:
            p = Path(td) / "01-big.plan.md"
            p.write_text(text, encoding="utf-8")
            proc = subprocess.run(
                ["python3", str(TOOL_PATH), str(p), "--mode", "standard"],
                capture_output=True,
                text=True,
            )
        self.assertEqual(proc.returncode, 0)
        self.assertIn("WARN", proc.stderr)
```

- [ ] **Step 2: 실패 확인**

Run: `cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills && /opt/homebrew/bin/pytest tests/tools/test_plan_validate.py::SizeWarnings -v`
Expected: 전부 FAIL — `AttributeError: module 'plan_validate_mod' has no attribute 'size_warnings'` (마지막 2개는 KeyError/AssertionError 가능).

- [ ] **Step 3: 구현**

`tools/plan-validate.py` 수정 3곳.

(a) `MODES = (...)` 정의 바로 아래에 상수 + 함수 추가:

```python
# 분량 가드 (WARN, 비차단) — 스펙: skills/context/lifecycle/plan-schema.md § 분량 가드
# 근거: 규약 이후 정상 plan 20~27k자. 초과분은 대부분 회차 이력 잔재
# (실사례: 이력 누적 plan 135k자 ≈ 토큰 65k — 후속 에이전트가 매 라운드 전문 재로딩).
SIZE_WARN_CHARS = 30_000
LINE_WARN_CHARS = 1_500  # Read 툴 라인 절단(2,000자) 안전 마진


def size_warnings(text: str) -> list[str]:
    """분량 가드 — 임계 초과를 WARN 메시지 리스트로 반환 (빈 리스트 = 통과).

    exit code 에 영향을 주지 않는다. planner 는 WARN 시 회차 이력 잔재
    (`N차 갱신` 헤더·`1회차 대비 정정` 주석·기각 사유 장문)를 정리하고
    최신 확정 상태만 남긴다 — 이력 SSOT 는 critic 합의 표.
    """
    warnings: list[str] = []
    total = len(text)
    if total > SIZE_WARN_CHARS:
        warnings.append(
            f"plan 분량 {total:,}자 — 상한 {SIZE_WARN_CHARS:,}자 초과. "
            "회차 이력 잔재를 정리하고 최신 확정 상태만 남길 것 "
            "(스펙: plan-schema.md § 분량 가드)"
        )
    longest = max((len(ln) for ln in text.splitlines()), default=0)
    if longest > LINE_WARN_CHARS:
        warnings.append(
            f"최장 라인 {longest:,}자 — {LINE_WARN_CHARS:,}자 초과. "
            "Read 툴 라인 절단(2,000자) 위험 — 표·문단 분리 필요"
        )
    return warnings
```

(b) `validate()` 의 result 초기 dict 에 `"warnings": []` 키 추가 + 본문 검증 진입 후 채움:

```python
    result: dict = {
        "valid": True,
        "mode": mode,
        "missing_sections": [],
        "step_errors": [],
        "errors": [],
        "warnings": [],
        "oq": _empty_oq_result(),
    }

    if not text.strip():
        result["valid"] = False
        result["errors"].append("file is empty")
        return result

    result["warnings"] = size_warnings(text)
```

(`result["warnings"] = size_warnings(text)` 는 기존 `if not has_plan_title(text):` 라인 바로 앞에 삽입.)

(c) `main()` 의 JSON print 다음, exit 분기 앞에 stderr 출력 추가:

```python
    for warn in result.get("warnings", []):
        print(f"plan-validate: WARN — {warn}", file=sys.stderr)
```

- [ ] **Step 4: 테스트 통과 + 전체 회귀**

Run: `cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills && /opt/homebrew/bin/pytest tests/tools/test_plan_validate.py -v && /opt/homebrew/bin/pytest tests/tools -q`
Expected: 전부 PASS (기존 ValidateIntegration·CliExit 회귀 포함).

- [ ] **Step 5: Commit**

```bash
git add dp-skills/tools/plan-validate.py dp-skills/tests/tools/test_plan_validate.py
git commit -m "feat(plan-validate): 분량 가드 WARN 추가

본문 30,000자·최장 라인 1,500자 초과를 warnings 필드 + stderr 로
보고 (exit code 불변 — 비차단). nimda QAPRJ-5394 plan 135k자
이력 누적 사례 재발 방지의 기계 게이트.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: plan-schema.md 에 분량 가드 스펙 추가

**Files:**
- Modify: `dp-skills/skills/context/lifecycle/plan-schema.md` (`## 호출` 섹션 직전에 신규 섹션)

- [ ] **Step 1: 신규 섹션 삽입**

`## Open Questions 게이트 검사 (모든 모드 공통)` 섹션 끝과 `## 호출` 사이에 삽입:

```markdown
## 분량 가드 (모든 모드 공통 — WARN, 비차단)

plan 은 Generator·Critic·Evaluator 가 매 라운드 전문을 Read 하는 인계 문서다. 분량 초과는 후속 에이전트의 컨텍스트 윈도우를 선소모하므로 plan-validate 가 WARN 으로 보고한다. exit code 는 불변 — 차단하지 않는다.

| 검사 | 임계 | 근거 |
| --- | --- | --- |
| 본문 길이 | 30,000자 초과 | ≈ 토큰 10~15k — 한 라운드 인계 문서 상한. 초과분은 대부분 회차 이력 잔재 |
| 최장 라인 | 1,500자 초과 | Read 툴 라인 절단(2,000자) 안전 마진 |

WARN 시 planner 는 회차 이력 잔재 — `N차 갱신`·`N회차 개정` 헤더, `[C# 해소]`·`1회차 대비 정정` 류 주석, 기각된 대안의 장문 사유 — 를 제거하고 **최신 확정 상태만** 남긴다. 반영·기각·이월 이력의 SSOT 는 critic 합의 표다 (ag-planner 절차 10).
```

- [ ] **Step 2: docs 동기화 + 린트**

Run: `python3 /Users/jay-p/Projects/deali-skills-plugin/dp-skills/tools/docs_build.py && /opt/homebrew/bin/pytest /Users/jay-p/Projects/deali-skills-plugin/dp-skills/tests/tools/test_docs_build.py -q`
Expected: drift 없음 (또는 재생성 후 무변경), PASS.

- [ ] **Step 3: Commit**

```bash
git add dp-skills/skills/context/lifecycle/plan-schema.md dp-skills/docs/
git commit -m "docs(plan-schema): 분량 가드 스펙 섹션 추가

plan-validate WARN 임계(30,000자·라인 1,500자)와 정리 대상
(회차 이력 잔재)의 SSOT 명세.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: ag-planner — reconcile "최신 확정 상태만 보존" + WARN 대응

**Files:**
- Modify: `dp-skills/agents/ag-planner.md` (절차 6 plan-validate 블록, 절차 10 재호출 분기)

- [ ] **Step 1: 절차 6 — WARN 처리 문장 추가**

절차 6 의 형식 검증 단락 (`**[필수] 저장 직후 형식 검증** — ... 검증 통과 전에는 7번으로 넘어가지 않는다.`) 끝에 한 문장 추가:

```markdown
stdout JSON 의 `warnings` 가 비어 있지 않으면 (exit 0 이어도) 원문을 사용자에게 보고하고, 회차 이력 잔재를 정리해 1회 재검증한다 — 정리할 잔재가 없는 정당한 분량이면 사유를 한 줄 보고하고 진행한다 (스펙: plan-schema.md § 분량 가드).
```

- [ ] **Step 2: 절차 10 — 분량 가드 bullet 추가**

절차 10 의 기존 3개 bullet (`- critic 의 챌린지 항목을 모두 검토하고...` / `- .plan.critic.md 의 ## 합의 표에...` / `- 합의 표를 채우지 않은 채...`) 아래에 추가:

```markdown
    - **[분량 가드] plan.md 는 항상 "최신 확정 상태" 만 담는다** — critic 반영 시 해당 부분을 다시 쓴다. `N차 갱신`·`N회차 개정` 헤더, `[C# 해소]`·`1회차 대비 정정` 류 이력 주석, 기각된 대안의 장문 사유를 본문에 누적하지 않는다. 반영·기각·이월 이력의 SSOT 는 critic 합의 표다. (근거: 이력 누적으로 plan 1개가 135k자까지 비대해진 실사례 — 후속 에이전트 전원이 매 라운드 전문을 재로딩한다.)
```

- [ ] **Step 3: docs 동기화 + Commit**

Run: `python3 /Users/jay-p/Projects/deali-skills-plugin/dp-skills/tools/docs_build.py && /opt/homebrew/bin/pytest /Users/jay-p/Projects/deali-skills-plugin/dp-skills/tests/tools/test_docs_build.py -q`
Expected: PASS.

```bash
git add dp-skills/agents/ag-planner.md dp-skills/docs/
git commit -m "feat(planner): reconcile 시 최신 확정 상태만 보존 + 분량 WARN 대응

절차 10 에 이력 잔재 누적 금지 명문화 (이력 SSOT = critic 합의 표),
절차 6 에 plan-validate warnings 처리 절차 추가.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 4: ag-planner-critic — 회차 이력 서술 금지 강화

**Files:**
- Modify: `dp-skills/agents/ag-planner-critic.md` (절차 5 출력 bullet)

- [ ] **Step 1: 절차 5 의 덮어쓰기 bullet 강화**

기존:

```markdown
   - 기존 파일이 있으면 덮어쓴다 (누적은 합의 표로 관리, 본문 챌린지는 최신 상태만 보존).
```

→ Edit 으로 교체:

```markdown
   - 기존 파일이 있으면 덮어쓴다 (누적은 합의 표로 관리, 본문 챌린지는 최신 상태만 보존). **재검증 회차의 이력 서술 금지** — `1회차 대비 정정`·`N회차 대비 보강`·`[유지 · 메커니즘 정정]` 류 회차 비교 서술을 챌린지 본문에 남기지 않는다. 재검증 결과는 severity·챌린지·제안 필드의 *현재 상태 갱신* 으로만 반영하고, 이전 회차와의 차이가 꼭 필요하면 합의 표 메모 1줄로 제한한다. (근거: 동일 규칙이 있었음에도 이력 서술 누적으로 critic 1개가 96k자까지 비대해진 실사례.)
```

- [ ] **Step 2: docs 동기화 + Commit**

Run: `python3 /Users/jay-p/Projects/deali-skills-plugin/dp-skills/tools/docs_build.py && /opt/homebrew/bin/pytest /Users/jay-p/Projects/deali-skills-plugin/dp-skills/tests/tools/test_docs_build.py -q`
Expected: PASS.

```bash
git add dp-skills/agents/ag-planner-critic.md dp-skills/docs/
git commit -m "fix(critic): 재검증 회차 이력 서술 금지 명문화

기존 '최신 상태만 보존' 규칙을 금지 패턴 예시와 함께 구체화 —
회차 비교 서술은 합의 표 메모 1줄로 제한.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 5: 사이클 내 대형 개정의 r{N} 분리 (qa 규약 확장)

**Files:**
- Modify: `dp-skills/skills/qa/SKILL.md` (§4-1 산출물 명명 규약 bullet 추가)
- Modify: `dp-skills/agents/ag-planner.md` (결함 수정 모드 "plan 저장 경로 (qa)" bullet)

- [ ] **Step 1: qa/SKILL.md §4-1 — bullet 추가**

§4-1 의 기존 bullet 목록 (`- 기존 비규약 파일은 자동 rename 하지 않는다 ...` 가 마지막) 끝에 추가:

```markdown
- **사이클 내 대형 개정도 새 회차로 분리** — 같은 티켓에서 plan 의 전제가 바뀌는 개정(결함 함수 목록 변경·1차 수정 머지 후 추가 수정·작업 브랜치 변경)은 기존 plan 에 `추가 수정` 류 섹션을 덧붙이지 않고 `qa/{KEY}.plan.r{N}.md` 새 파일로 시작한다. critic·eval 도 같은 r 을 따른다. (근거: 한 plan 파일에 추가 수정 2건·4차 갱신이 누적돼 135k자까지 비대해진 실사례.)
```

- [ ] **Step 2: ag-planner 결함 수정 모드 — 분리 기준 문장 추가**

"plan 저장 경로 (qa)" bullet 끝 (`... .r{N} 은 항상 마지막 .md 직전 (규약 SSOT: qa/SKILL.md "qa/ 산출물 명명 규약").`) 에 한 문장 추가:

```markdown
재작업뿐 아니라 **사이클 내 대형 개정** — 결함 함수 목록·작업 브랜치가 바뀌는 수정 — 도 새 r{N} 파일로 분리한다 (기존 plan 에 `추가 수정` 섹션 덧붙이기 금지 — 규약 SSOT 동일).
```

- [ ] **Step 3: docs 동기화 + 전체 테스트 + Commit**

Run: `python3 /Users/jay-p/Projects/deali-skills-plugin/dp-skills/tools/docs_build.py && cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills && /opt/homebrew/bin/pytest tests/tools -q`
Expected: drift 없음, 전부 PASS.

```bash
git add dp-skills/skills/qa/SKILL.md dp-skills/agents/ag-planner.md dp-skills/docs/
git commit -m "feat(qa): 사이클 내 대형 개정도 r{N} 새 파일로 분리

결함 함수·브랜치가 바뀌는 개정을 기존 plan 에 덧붙이지 않고
새 회차로 시작 — 단일 파일 누적 비대 차단.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 6: 버전 bump + PR

**Files:**
- Modify: `dp-skills/.claude-plugin/plugin.json`

- [ ] **Step 1: plugin.json bump**

`"version": "0.19.0"` → `"0.20.0"` (minor — 신규 가드 + 규약 확장. main 에 선행 머지로 0.19.x 가 올라가 있으면 그 기준 +1 minor).

- [ ] **Step 2: 전체 테스트 최종 확인**

Run: `cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills && /opt/homebrew/bin/pytest tests/tools -q`
Expected: 전부 PASS.

- [ ] **Step 3: Commit + PR**

```bash
git add dp-skills/.claude-plugin/plugin.json
git commit -m "chore(release): bump 0.20.0

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
git push -u origin HEAD
gh pr create --base main --title "feat(plan): 산출물 분량 가드 + 최신 상태 보존 (0.20.0)" --body "$(cat <<'EOF'
## 요약
- plan-validate 분량 가드 WARN — 본문 30,000자·최장 라인 1,500자 초과 보고 (비차단, exit 불변)
- planner 절차 10·critic 절차 5 에 "최신 확정 상태만 보존" 명문화 — 회차 이력 잔재 누적 금지 (이력 SSOT = critic 합의 표)
- qa 규약 확장 — 사이클 내 대형 개정(결함 함수·브랜치 변경)도 r{N} 새 파일로 분리

## 배경
nimda Presend-Admin QAPRJ-5394 에서 plan 198KB(≈65k 토큰)·critic 148KB(≈50k 토큰) — 회차 이력 누적이 원인. 후속 에이전트 전원이 매 라운드 전문을 재로딩해 컨텍스트 윈도우 절반 이상을 선소모, opus 반복 라운드의 input 재독이 비용 주범.

critic 에 동일 취지 규칙이 있었음에도 비대가 발생 → 지침과 함께 기계 게이트(plan-validate, planner 의 통과-전-진행-금지 게이트) 도입. 디테일 훼손을 피하려 WARN 비차단으로 시작 — 재발 시 임계 강화.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

머지 전략: squash (linear history — CLAUDE.md 컨벤션).

---

## 본 plan 의 비범위 (후속 후보)

- **critic 파일 자체의 기계 게이트** — critic 은 validate 게이트가 없다. 비대의 주원인(이력 서술)은 Task 4 가 차단하므로 우선 지침으로 — 재발 시 critic 출력 검증 도구 신설.
- **handoff-quality 길이 메트릭** — plan-validate WARN 과 중복이라 불채택. handoff-quality 는 구체성(specificity) 축 유지.
- **features/ phase 의 대형 개정 r{N} 분리** — features plan 은 명명 규약에 r{N} 자체가 없다. 필요해지면 별도 plan.
- **기존 비대 파일 정리** — nimda 산출물은 이미 생성된 이력. 자동 정리하지 않는다.
