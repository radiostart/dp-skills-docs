# QA 산출물 명명 규약 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** qa phase 사이클 산출물(plan·critic·eval)의 파일 명명을 재작업(rework) 회차까지 포함해 단일 규약으로 명세화하고, doctor 가 규약 위반·고아 산출물을 감지하게 한다.

**Architecture:** 규약의 SSOT 는 `skills/qa/SKILL.md` 의 신규 "산출물 명명 규약" 섹션. 에이전트 3종(ag-planner·ag-planner-critic·ag-evaluator)의 qa 분기 블록에 경로 규칙을 1줄씩 주입하고, `tools/doctor/integrity.py` 에 `check_qa_artifact_naming()` 검사를 추가해 드리프트를 기계적으로 잡는다. 기존 비규약 파일은 마이그레이션하지 않는다 — doctor WARN 으로 수동 rename 을 안내만 한다.

**Tech Stack:** Python 3 표준 라이브러리 (doctor), unittest + tempfile (기존 테스트 컨벤션), markdown 명세.

---

## 배경 (실데이터 근거)

플러그인은 현재 `qa/{KEY}.md` (티켓 본문) 만 정의한다. plan·critic·eval 산출물 경로는 에이전트가 즉흥 결정해 왔고, nimda Presend-Admin 의 QAPRJ-5438 재작업에서 **같은 티켓 안에 두 컨벤션이 공존**하게 됐다:

```text
QAPRJ-5438.plan.critic.r1.md   ← rN 이 마지막 .md 직전
QAPRJ-5438.plan.r1.critic.md   ← rN 이 중간 (동일 의미, 다른 즉흥 컨벤션)
```

## 확정 규약 (canonical)

| 산출물 | 초회 | 재작업 N회차 |
| --- | --- | --- |
| 티켓 본문 | `qa/{KEY}.md` | (재생성 금지 — 기존 유지) |
| plan | `qa/{KEY}.plan.md` | `qa/{KEY}.plan.r{N}.md` |
| plan critic | `qa/{KEY}.plan.critic.md` | `qa/{KEY}.plan.critic.r{N}.md` |
| eval | `qa/{KEY}.eval.md` | `qa/{KEY}.eval.r{N}.md` |

- **`.r{N}` 은 항상 마지막 `.md` 바로 앞.** (nimda 실데이터 4건 중 3건과 호환 — 기존 다수파 채택)
- **N 결정**: `qa/{KEY}.plan*.md` 중 기존 r 최대값 + 1. r 표기 없는 초회 산출물만 있으면 N=1.
- critic 파일명 = 대응 plan 파일명에서 `.plan` → `.plan.critic` 치환 (r 꼬리표 유지).
- eval 의 r = 대응 plan 의 r.

---

### Task 1: doctor `check_qa_artifact_naming()` 검사

**Files:**
- Modify: `dp-skills/tools/doctor/integrity.py` (신규 함수 + `check_project` 배선, 1049행 부근)
- Test: `dp-skills/tests/tools/test_doctor_qa_naming.py` (신규)

- [ ] **Step 1: 실패하는 테스트 작성**

`dp-skills/tests/tools/test_doctor_qa_naming.py` 신규 생성:

```python
"""qa/ 산출물 명명 규약 검사 테스트.

규약: {KEY}.md / {KEY}.(plan|plan.critic|eval)(.r{N})?.md — rN 은 마지막 .md 직전.
배경: nimda QAPRJ-5438 재작업에서 plan.r1.critic.md / plan.critic.r1.md 혼재.
"""

from __future__ import annotations

import importlib.util
import tempfile
import unittest
from pathlib import Path

THIS_DIR = Path(__file__).resolve().parent
PLUGIN_ROOT = THIS_DIR.parent.parent
INTEGRITY_PATH = PLUGIN_ROOT / "tools" / "doctor" / "integrity.py"


def _load_integrity():
    import sys

    doctor_pkg_path = PLUGIN_ROOT / "tools" / "doctor"
    if str(doctor_pkg_path.parent) not in sys.path:
        sys.path.insert(0, str(doctor_pkg_path.parent))
    spec = importlib.util.spec_from_file_location(
        "doctor.integrity",
        INTEGRITY_PATH,
        submodule_search_locations=[str(doctor_pkg_path)],
    )
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)
    return module


class QaArtifactNaming(unittest.TestCase):
    def _run(self, names: list[str]):
        mod = _load_integrity()
        with tempfile.TemporaryDirectory() as td:
            qa = Path(td) / "qa"
            qa.mkdir()
            for name in names:
                (qa / name).write_text("x", encoding="utf-8")
            return mod.check_qa_artifact_naming(qa, "myproj")

    def test_valid_initial_cycle_silent(self):
        results = self._run(
            [
                "QAPRJ-5394.md",
                "QAPRJ-5394.plan.md",
                "QAPRJ-5394.plan.critic.md",
                "QAPRJ-5394.eval.md",
            ]
        )
        self.assertEqual(results, [])

    def test_valid_rework_cycle_silent(self):
        results = self._run(
            [
                "QAPRJ-5438.md",
                "QAPRJ-5438.plan.r1.md",
                "QAPRJ-5438.plan.critic.r1.md",
                "QAPRJ-5438.eval.r2.md",
            ]
        )
        self.assertEqual(results, [])

    def test_misplaced_rn_warns(self):
        # nimda 실사례: rN 이 중간에 끼어든 변형
        results = self._run(["QAPRJ-5438.md", "QAPRJ-5438.plan.r1.critic.md"])
        self.assertEqual(len(results), 1)
        self.assertEqual(results[0].level, "WARN")
        self.assertIn("plan.critic.r1.md", results[0].hint)

    def test_orphan_artifact_warns(self):
        # 티켓 본문 {KEY}.md 없이 산출물만 존재
        results = self._run(["QAPRJ-9999.plan.md"])
        self.assertEqual(len(results), 1)
        self.assertIn("고아", results[0].message)

    def test_unknown_file_warns(self):
        results = self._run(["QAPRJ-5394.md", "notes.md"])
        self.assertEqual(len(results), 1)
        self.assertEqual(results[0].level, "WARN")

    def test_missing_qa_dir_silent(self):
        mod = _load_integrity()
        self.assertEqual(
            mod.check_qa_artifact_naming(Path("/nonexistent/qa"), "myproj"), []
        )


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: 실패 확인**

Run: `cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills && python3 -m pytest tests/tools/test_doctor_qa_naming.py -v`
Expected: 전부 FAIL — `AttributeError: ... has no attribute 'check_qa_artifact_naming'`.

- [ ] **Step 3: 구현**

`dp-skills/tools/doctor/integrity.py` 에 신규 함수 추가 (`check_eval_md_pairing` 아래):

```python
_QA_TICKET_RE = re.compile(r"^([A-Z][A-Z0-9]*-\d+)\.md$")
_QA_ARTIFACT_RE = re.compile(
    r"^([A-Z][A-Z0-9]*-\d+)\.(plan|plan\.critic|eval)(\.r\d+)?\.md$"
)
# rN 이 잘못된 위치에 끼어든 변형 감지용 (예: plan.r1.critic.md)
_QA_MISPLACED_RN_RE = re.compile(
    r"^([A-Z][A-Z0-9]*-\d+)\.(plan)(\.r(\d+))\.(critic)\.md$"
)


def check_qa_artifact_naming(qa_dir: Path, project: str) -> list[Result]:
    """qa/ 산출물 명명 규약 검사.

    규약: {KEY}.md / {KEY}.(plan|plan.critic|eval)(.r{N})?.md — `.r{N}` 은
    항상 마지막 `.md` 직전. 규약 위반·고아 산출물(티켓 본문 없는 파생물)을
    WARN 으로 보고한다. 자동 rename 은 하지 않는다 (수동 안내만).
    """
    if not qa_dir.is_dir():
        return []

    results: list[Result] = []
    ticket_keys: set[str] = set()
    artifacts: list[tuple[str, str]] = []  # (filename, key)

    for p in sorted(qa_dir.glob("*.md")):
        m = _QA_TICKET_RE.match(p.name)
        if m:
            ticket_keys.add(m.group(1))
            continue
        m = _QA_ARTIFACT_RE.match(p.name)
        if m:
            artifacts.append((p.name, m.group(1)))
            continue
        # 규약 위반 — rN 위치 변형이면 기대 이름을 hint 로 제시
        mis = _QA_MISPLACED_RN_RE.match(p.name)
        hint = (
            f"기대 이름: {mis.group(1)}.plan.critic.r{mis.group(4)}.md"
            if mis
            else "규약: {KEY}.(plan|plan.critic|eval)(.r{N})?.md — rN 은 마지막 .md 직전"
        )
        results.append(
            Result(
                Result.WARN,
                f"{project} qa/{p.name}",
                "산출물 명명 규약 위반",
                hint,
            )
        )

    for name, key in artifacts:
        if key not in ticket_keys:
            results.append(
                Result(
                    Result.WARN,
                    f"{project} qa/{name}",
                    f"고아 산출물 — 티켓 본문 qa/{key}.md 없음",
                    "`/dp-skills:qa {KEY}` 재진입으로 본문 생성 또는 파일 정리",
                )
            )
    return results
```

- [ ] **Step 4: `check_project` 에 배선**

`integrity.py:1049` 의 `results.extend(check_eval_md_pairing(features_dir, project_md, project))` 바로 아래에 추가:

```python
    results.extend(check_qa_artifact_naming(state_yml.parent / "qa", project))
```

(`state_yml` 은 `check_project` 에서 이미 resolve 된 `.agent-state.yml` 경로 — 프로젝트 루트 기준 `qa/` 파생.)

- [ ] **Step 5: 테스트 통과 + 전체 회귀**

Run: `cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills && python3 -m pytest tests/tools/test_doctor_qa_naming.py -v && python3 -m pytest tests/tools -q`
Expected: 전부 PASS.

- [ ] **Step 6: Commit**

```bash
git add dp-skills/tools/doctor/integrity.py dp-skills/tests/tools/test_doctor_qa_naming.py
git commit -m "feat(doctor): qa/ 산출물 명명 규약 검사 추가

규약 위반(rN 위치 변형)·고아 산출물을 WARN 으로 감지.
nimda QAPRJ-5438 재작업의 명명 혼재 사례 반영.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: qa/SKILL.md 에 규약 SSOT 섹션 추가

**Files:**
- Modify: `dp-skills/skills/qa/SKILL.md` (4단계 재작업 분기 + 신규 섹션)

- [ ] **Step 1: 재작업 분기에 rN 결정 절차 추가**

`qa/SKILL.md:126` 의 기존 문장:

```markdown
- **존재함** (= 같은 티켓 재작업) → `qa/{KEY}.md` 를 Read 만 수행하고 6 단계로 진행. `qa/{KEY}.md` 재생성·Jira 재fetch 금지 (기존 내용 보존). **단, 6 단계 작업 브랜치 생성의 git Bash 는 허용한다** — 재작업도 새 작업 브랜치가 필요하다.
```

을 Edit 으로 다음과 같이 확장 (끝에 한 문장 추가):

```markdown
- **존재함** (= 같은 티켓 재작업) → `qa/{KEY}.md` 를 Read 만 수행하고 6 단계로 진행. `qa/{KEY}.md` 재생성·Jira 재fetch 금지 (기존 내용 보존). **단, 6 단계 작업 브랜치 생성의 git Bash 는 허용한다** — 재작업도 새 작업 브랜치가 필요하다. 재작업 회차 N 은 `qa/{KEY}.plan*.md` 의 기존 r 최대값 + 1 로 정하고 (초회 산출물만 있으면 N=1), 이후 사이클 산출물은 아래 "qa/ 산출물 명명 규약"의 `.r{N}` 명명을 따른다.
```

- [ ] **Step 2: 신규 규약 섹션 추가**

위에서 수정한 4단계 분기 블록 바로 아래(다음 `###` 헤더 직전)에 신규 섹션 삽입:

```markdown
### 4-1. qa/ 산출물 명명 규약

사이클 산출물의 파일명은 아래 표가 SSOT 다. 에이전트(@ag-planner·@ag-planner-critic·@ag-evaluator)의 qa 분기가 이 규약을 따르고, `/dp-skills:doctor` 가 위반을 WARN 으로 감지한다.

| 산출물 | 초회 | 재작업 N회차 |
| --- | --- | --- |
| 티켓 본문 | `qa/{KEY}.md` | (재생성 금지) |
| plan | `qa/{KEY}.plan.md` | `qa/{KEY}.plan.r{N}.md` |
| plan critic | `qa/{KEY}.plan.critic.md` | `qa/{KEY}.plan.critic.r{N}.md` |
| eval | `qa/{KEY}.eval.md` | `qa/{KEY}.eval.r{N}.md` |

- `.r{N}` 은 **항상 마지막 `.md` 바로 앞**. (`{KEY}.plan.r1.critic.md` 같은 중간 삽입 금지)
- critic 파일명 = 대응 plan 파일명에서 `.plan` → `.plan.critic` 치환 (r 꼬리표 유지).
- eval 의 r = 대응 plan 의 r.
- 기존 비규약 파일은 자동 rename 하지 않는다 — doctor WARN 의 기대 이름으로 수동 정리.
```

- [ ] **Step 3: 린트 + docs 동기화**

Run: `python3 /Users/jay-p/Projects/deali-skills-plugin/dp-skills/tools/docs_build.py && python3 -m pytest /Users/jay-p/Projects/deali-skills-plugin/dp-skills/tests/tools/test_docs_build.py -q`
Expected: drift 없음, PASS.

- [ ] **Step 4: Commit**

```bash
git add dp-skills/skills/qa/SKILL.md dp-skills/docs/
git commit -m "docs(qa): 산출물 명명 규약 SSOT 섹션 추가

plan·critic·eval 의 초회·재작업(rN) 파일명 규약을 qa/SKILL.md 에 명세.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: 에이전트 3종 qa 분기에 경로 규칙 주입

**Files:**
- Modify: `dp-skills/agents/ag-planner.md:37-44` (결함 수정 모드 블록), `:68` (계획 저장 스텝)
- Modify: `dp-skills/agents/ag-planner-critic.md:49` (plan 탐색), `:89` 부근 (산출물 경로)
- Modify: `dp-skills/agents/ag-evaluator.md:43-49` (qa 블록)

- [ ] **Step 1: ag-planner — plan 저장 경로 bullet 추가**

`ag-planner.md` 의 결함 수정 모드 블록(37-44행, `- **qa/{KEY}.md 갱신**` bullet 다음)에 추가:

```markdown
   - **plan 저장 경로 (qa)**: 본 사이클의 계획은 `features/NN-{slug}.plan.md` 가 아니라 `qa/{KEY}.plan.md` 에 저장한다. 재작업이면 `qa/{KEY}.plan.r{N}.md` — N 은 `ls workspace/{TEAM}/projects/{PROJECT}/qa/{KEY}.plan*.md` 의 기존 r 최대값 + 1 (초회 산출물만 있으면 1). `.r{N}` 은 항상 마지막 `.md` 직전 (규약 SSOT: qa/SKILL.md "qa/ 산출물 명명 규약").
```

68행의 계획 저장 스텝:

```markdown
6. **[계획 저장]** `features/` 폴더가 있는 프로젝트에서, 계획이 확정되면 `features/NN-{slug}.plan.md`에 저장한다 (NN은 feature 번호, slug는 feature 파일명과 동일).
```

→ 끝에 한 문장 추가:

```markdown
6. **[계획 저장]** `features/` 폴더가 있는 프로젝트에서, 계획이 확정되면 `features/NN-{slug}.plan.md`에 저장한다 (NN은 feature 번호, slug는 feature 파일명과 동일). phase=qa 면 결함 수정 모드 블록의 "plan 저장 경로 (qa)" 가 우선한다 (`qa/{KEY}.plan[.r{N}].md`).
```

- [ ] **Step 2: ag-planner-critic — plan 탐색 확장 + critic 산출물 경로**

49행:

```bash
   ls -t workspace/{TEAM}/projects/{PROJECT}/features/*.plan.md 2>/dev/null | head -3
```

→

```bash
   ls -t workspace/{TEAM}/projects/{PROJECT}/features/*.plan.md \
         workspace/{TEAM}/projects/{PROJECT}/qa/*.plan*.md 2>/dev/null | head -3
```

critic 산출물(`.plan.critic.md`) 경로를 정의하는 본문(89행 부근, `> 입력 plan:` 양식 근처)에 다음 1줄 추가:

```markdown
   critic 산출물 파일명 = 대상 plan 파일명에서 `.plan` → `.plan.critic` 치환 (`.r{N}` 꼬리표는 그대로 유지 — 예: `QAPRJ-5438.plan.r1.md` → `QAPRJ-5438.plan.critic.r1.md`).
```

- [ ] **Step 3: ag-evaluator — eval 저장 경로 bullet 추가**

`ag-evaluator.md` 의 qa 블록(43-49행, `- **qa/{KEY}.md 회귀영향 섹션 갱신**` bullet 다음)에 추가:

```markdown
   - **eval 저장 경로 (qa)**: VERIFICATION REPORT 는 `qa/{KEY}.eval.md` 에 저장한다. 대응 plan 이 `qa/{KEY}.plan.r{N}.md` 면 `qa/{KEY}.eval.r{N}.md` — r 은 plan 과 동일하게 맞춘다 (규약 SSOT: qa/SKILL.md).
```

- [ ] **Step 4: 린트 + docs 동기화 + 전체 테스트**

Run: `python3 /Users/jay-p/Projects/deali-skills-plugin/dp-skills/tools/docs_build.py && cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills && python3 -m pytest tests/tools -q`
Expected: drift 없음, 전부 PASS.

- [ ] **Step 5: Commit**

```bash
git add dp-skills/agents/ dp-skills/docs/
git commit -m "feat(agents): qa 사이클 산출물 경로 규약 주입

planner·critic·evaluator 의 qa 분기에 qa/{KEY}.{종류}[.r{N}].md
명명 규칙을 명시 — 즉흥 명명으로 인한 혼재 차단.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 4: 버전 bump + PR

**Files:**
- Modify: `dp-skills/.claude-plugin/plugin.json`

- [ ] **Step 1: plugin.json bump**

현재 최신 버전 +1 patch (PLAN-feature-count-integrity 가 먼저 머지돼 0.16.7 이면 → `"0.16.8"`).

- [ ] **Step 2: 전체 테스트 최종 확인**

Run: `cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills && python3 -m pytest tests/tools -q`
Expected: 전부 PASS.

- [ ] **Step 3: Commit + PR**

```bash
git add dp-skills/.claude-plugin/plugin.json
git commit -m "chore(release): bump 0.16.8

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
git push -u origin HEAD
gh pr create --base main --title "feat(qa): 산출물 명명 규약 + doctor 검사 (0.16.8)" --body "$(cat <<'EOF'
## 요약
- qa/ 산출물 명명 규약 SSOT 를 qa/SKILL.md 에 명세 (초회·재작업 rN)
- 에이전트 3종 qa 분기에 경로 규칙 주입
- doctor `check_qa_artifact_naming` — 규약 위반·고아 산출물 WARN

## 배경
nimda QAPRJ-5438 재작업에서 `plan.critic.r1.md` 와 `plan.r1.critic.md` 가 같은 티켓에 공존 — 재작업 명명이 미정의라 에이전트가 즉흥 결정해 온 결과.

기존 비규약 파일은 자동 rename 하지 않음 — doctor WARN 의 기대 이름으로 수동 정리.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

머지 전략: squash (linear history — CLAUDE.md 컨벤션).

---

## 본 plan 의 비범위 (후속 plan 후보)

- phase 판정 orchestrate-load 중앙화·fail-closed (우선순위 3)
- 플러그인 업그레이드 → `--regen-agents` 권고 연결 (우선순위 4)
- 보조 컨텍스트 문서(RESUME.md·schema.md 등) 로드 계약·프로젝트 스냅샷 경로 (우선순위 5)
