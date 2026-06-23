# Phase 판정 중앙화 + Fail-Closed Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** project phase(development/qa) 판정을 orchestrate-load 로 중앙화해 JSON `project_phase` 로 노출하고, 비정상 값은 error 중단(fail-closed)으로 바꿔 에이전트 3종의 inline grep(fail-open)을 제거한다.

**Architecture:** `orchestrate-load.py` 에 검증 헬퍼 `resolve_project_phase(state)` 를 추가해 main 에서 배선 — 부재는 `development`(v1.2 호환 기본값), `development`/`qa` 외 값은 error + exit 1. 에이전트 wrapper 4종은 JSON 의 `project_phase` 를 소비하도록 문단 교체. state 누락·corrupt·미지원 schema 는 orchestrate-load 가 **이미 fail-closed** — 본 작업은 phase 값 경로만 닫는다.

**Tech Stack:** Python 3 표준 라이브러리, unittest (기존 test_orchestrate_load.py 컨벤션 — 모듈 단위 함수 테스트), markdown wrapper.

---

## 배경

세 에이전트(planner·generator·evaluator)가 동일 문구로 `grep '^phase:'` inline 확인을 수행하며 *"orchestrate-load 가 아직 phase 를 노출하지 않으므로 본 inline 확인이 SSOT"* 라고 명시한 문서화된 부채. grep 실패·오타 값(`qaa` 등)·파일 corrupt 시 **조용히 development 로 폴백**해 qa phase 의 최소 변경·회귀영향 게이트가 전부 풀린다. nimda Presend-Admin 이 현재 `phase: qa` 로 가동 중 — 실위험.

주의: 결과 JSON 의 기존 `phase` 키는 **에이전트 역할**(planner/generator/...)이다. project phase 는 별도 키 `project_phase` 로 노출한다 (키 충돌 회피).

---

### Task 1: `resolve_project_phase()` + main 배선 + 계약 docstring (TDD)

**Files:**
- Modify: `dp-skills/tools/orchestrate-load.py` (docstring 출력 계약, result 템플릿, mode 블록 다음 배선, 신규 헬퍼)
- Test: `dp-skills/tests/tools/test_orchestrate_load.py` (클래스 추가)

- [ ] **Step 1: 실패하는 테스트 작성**

`test_orchestrate_load.py` 에 클래스 추가 (파일 말미 `if __name__` 위):

```python
class ResolveProjectPhase(unittest.TestCase):
    def test_absent_defaults_development(self):
        self.assertEqual(m.resolve_project_phase({}), ("development", None))

    def test_null_value_defaults_development(self):
        # parse_state_yml 은 `phase:` 빈 값·null 을 None 으로 만든다
        self.assertEqual(
            m.resolve_project_phase({"phase": None}), ("development", None)
        )

    def test_qa_passes(self):
        self.assertEqual(m.resolve_project_phase({"phase": "qa"}), ("qa", None))

    def test_development_passes(self):
        self.assertEqual(
            m.resolve_project_phase({"phase": "development"}),
            ("development", None),
        )

    def test_invalid_value_fails_closed(self):
        phase, err = m.resolve_project_phase({"phase": "qaa"})
        self.assertIsNone(phase)
        self.assertIn("qaa", err)
        self.assertIn("development, qa", err)
```

- [ ] **Step 2: 실패 확인**

Run: `cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills && pytest tests/tools/test_orchestrate_load.py -q`
Expected: `ResolveProjectPhase` 5건 FAIL — `AttributeError: ... resolve_project_phase`.

- [ ] **Step 3: 구현**

`orchestrate-load.py` 의 `parse_state_yml` 아래에 추가:

```python
VALID_PROJECT_PHASES = ("development", "qa")


def resolve_project_phase(state: dict) -> tuple[str | None, str | None]:
    """state 의 phase 필드를 검증·정규화. (phase, error) 반환.

    부재·null → ("development", None) — v1.2 호환 기본값.
    유효 값 → (값, None). 그 외 → (None, 에러) — fail-closed:
    종전 inline grep 방식은 오타·corrupt 를 development 로 조용히 폴백해
    qa 게이트(최소 변경·회귀영향)가 풀렸다.
    """
    raw = state.get("phase")
    if raw is None:
        return "development", None
    if raw in VALID_PROJECT_PHASES:
        return raw, None
    return None, (
        f".agent-state.yml 의 phase={raw!r} 가 유효하지 않음 "
        f"(허용: {', '.join(VALID_PROJECT_PHASES)}). "
        "phase 행을 수정하거나 migrate-state.py 로 재생성 필요."
    )
```

main() 의 result 템플릿에 `"project_phase": None,` 추가 (`"mode": None,` 다음 줄). mode 처리 블록 바로 다음에 배선:

```python
    # project phase (v1.3+, 부재=development) — fail-closed: 비정상 값이면 에러 중단
    project_phase, phase_err = resolve_project_phase(state)
    if phase_err:
        result["error"] = phase_err
        print(json.dumps(result, ensure_ascii=False, indent=2))
        return 1
    result["project_phase"] = project_phase
    if project_phase == "qa":
        result["hints"].append(
            "[phase] qa — 결함 수정 모드: 최소 변경·회귀영향 후보 필수·features/ 읽기 전용"
        )
```

docstring 출력 계약(JSON 예시)의 `"tdd": bool,` 아래에 `"project_phase": "development" | "qa",` 추가.

- [ ] **Step 4: 통과 + 전체 회귀**

Run: `pytest tests/tools/test_orchestrate_load.py -q && pytest tests/tools -q`
Expected: 전부 PASS.

- [ ] **Step 5: Commit**

```bash
git add dp-skills/tools/orchestrate-load.py dp-skills/tests/tools/test_orchestrate_load.py
git commit -m "feat(orchestrate): project_phase 노출 + fail-closed 검증

phase 판정을 orchestrate-load 로 중앙화. 비정상 값은 error 중단 —
종전 inline grep 의 development 조용한 폴백(fail-open) 차단.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: 에이전트 wrapper 4종 — inline grep 제거, JSON 소비로 전환

**Files:**
- Modify: `dp-skills/agents/ag-planner.md:29,35`, `ag-generator.md:29,41`, `ag-evaluator.md:29,40`, `ag-planner-critic.md:41`

- [ ] **Step 1: phase 확인 문단 교체 (3 wrapper 동일 문구 — replace_all)**

기존 (3 파일 동일):

```markdown
   **[필수] phase 확인** — 위 컨텍스트 로드 후 `.agent-state.yml` 의 `phase` 필드를 직접 확인한다 (`grep '^phase:' workspace/{TEAM}/projects/{PROJECT}/.agent-state.yml`). 값이 `qa` 이면 아래 **결함 수정 모드** 블록을 활성화한다 (mode 와 직교 — `tdd: true` 또는 `mode: characterize` 와 공존 가능). 값이 없거나 `development` 이면 평소대로 진행. orchestrate-load 가 아직 phase 를 노출하지 않으므로 본 inline 확인이 SSOT.
```

→ 신규:

```markdown
   **[필수] phase 확인** — step 1 JSON 의 `project_phase` 값을 사용한다. `qa` 이면 아래 **결함 수정 모드** 블록을 활성화한다 (mode 와 직교 — `tdd: true` 또는 `mode: characterize` 와 공존 가능). `development` 이면 평소대로 진행. 직접 grep 으로 재확인하지 않는다 — phase 파싱·검증(fail-closed)은 orchestrate-load 가 담당하며, 비정상 값이면 step 1 의 `error` 로 이미 중단됐다.
```

- [ ] **Step 2: JSON 처리 bullet 갱신 (4 wrapper 동일 문구 — replace_all)**

`- analyzed / tdd / domain 값을 이후 분기에 사용.` → `- analyzed / tdd / domain / project_phase 값을 이후 분기에 사용.` (백틱 포함 원문 그대로 매칭)

- [ ] **Step 3: docs 동기화 + 전체 테스트 + Commit**

```bash
python3 dp-skills/tools/docs_build.py && cd dp-skills && pytest tests/tools -q
git add dp-skills/agents/ dp-skills/docs/
git commit -m "refactor(agents): phase inline grep 제거 — project_phase 소비

orchestrate-load 가 노출하는 project_phase 가 단일 판정 경로.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: bump 0.16.9 + PR

- [ ] **Step 1**: `plugin.json` `0.16.8` → `0.16.9`, 전체 테스트, commit `chore(release): bump 0.16.9`.
- [ ] **Step 2**: push + `gh pr create --base main` (squash 머지 전제). PR 본문에 배경(fail-open 실위험 — nimda 활성 qa 프로젝트)과 비범위(protect-managed.sh hook 의 자체 phase 판정은 결정론적 shell — 별도) 명시.

## 비범위

- `protect-managed.sh` hook 의 phase 판정 (결정론적 shell, 별도 강화 대상)
- qa/project 스킬의 대화형 phase Read (LLM 이 파일 전체를 Read — fail-open 아님)
- 우선순위 4(업그레이드→regen 연결)·5(보조 컨텍스트 로드)는 후속 plan
