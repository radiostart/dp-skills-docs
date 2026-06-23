# Feature 카운트 정합성 회복 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** feature 카운트 규칙을 명세·doctor 구현·analyze 기록 3곳에서 단일 규칙으로 통일하고, 부풀려진 `last_analyzed_features` 를 감지·복구하는 역방향 drift 검사를 추가한다.

**Architecture:** 카운트 술어를 `tools/doctor/_common.py` 의 `is_feature_spec()` 하나로 모으고(doctor 의 `count_real_features`·`_list_feature_md` 가 공유), drift 검사를 `check_feature_count_drift()` 함수로 추출해 역방향(기록값 > 실제) 감지 + `--fix` 재기록을 붙인다. 명세 문서(state-schema·analyze references·project SKILL·GUIDE)는 동일 문구로 동기화한다.

**Tech Stack:** Python 3 표준 라이브러리만 (기존 tools 컨벤션), unittest + tempfile (기존 테스트 컨벤션), markdown 명세 문서.

---

## 배경 (실데이터 근거)

nimda `Presend-Admin` 프로젝트에서 확인된 연쇄 결함:

1. 명세([state-schema.md:26](../../skills/context/lifecycle/state-schema.md)·[agents-update.md:174](../../skills/analyze/references/agents-update.md))의 카운트 규칙은 "`.plan.md` 제외"만 명시 → analyze(LLM)가 `.eval.md` 를 포함해 `last_analyzed_features: 44` 를 기록 (실제 본문 명세 32건).
2. doctor 의 `count_real_features` 는 `.plan.md`·`.eval.md` 는 제외하지만 **`.plan.critic.md`·`.plan.r1.md`·`.eval.r1.md` 같은 파생 산출물은 카운트**한다 (`endswith` 2종만 검사).
3. drift WARN 은 증가 방향만 검사 (`feature_count > last_count + 1`) → 기준값이 부풀려진 프로젝트는 feature 가 13건 더 추가될 때까지 **어떤 경고도 발생하지 않음**. 감소 방향 불일치는 검사 항목 자체가 없음.

**단일 규칙 (canonical):** feature 본문 명세 = `features/NN-{slug}.md` — `NN-` 2자리 숫자 접두 + **파일명에 점(.)이 확장자 1개뿐**인 파일. 파생 산출물(`.plan.md`·`.eval.md`·`.plan.critic.md`·`.plan.r{N}.md`·`.eval.r{N}.md`)은 점이 2개 이상이므로 자동 제외된다.

---

### Task 1: 공유 술어 `is_feature_spec()` — 파생 산출물 완전 제외

**Files:**
- Modify: `dp-skills/tools/doctor/_common.py:244-261` (`_FEATURE_NAME_RE`·`count_real_features`)
- Modify: `dp-skills/tools/doctor/integrity.py:1230-1240` (`_list_feature_md`), `integrity.py:16` (import 목록)
- Test: `dp-skills/tests/tools/test_doctor_feature_count.py` (신규)

- [ ] **Step 1: 실패하는 테스트 작성**

`dp-skills/tests/tools/test_doctor_feature_count.py` 신규 생성:

```python
"""feature 카운트 단일 규칙 테스트 — NN-{slug}.md 본문 명세만, 파생 산출물 전부 제외.

배경: count_real_features 가 .plan.md/.eval.md 만 endswith 로 제외해
.plan.critic.md·.plan.r1.md 등이 카운트에 섞여 drift 기준값이 오염됐다
(nimda Presend-Admin 실데이터).
"""

from __future__ import annotations

import importlib.util
import tempfile
import unittest
from pathlib import Path

THIS_DIR = Path(__file__).resolve().parent
PLUGIN_ROOT = THIS_DIR.parent.parent
COMMON_PATH = PLUGIN_ROOT / "tools" / "doctor" / "_common.py"


def _load_common():
    import sys

    doctor_pkg_path = PLUGIN_ROOT / "tools" / "doctor"
    if str(doctor_pkg_path.parent) not in sys.path:
        sys.path.insert(0, str(doctor_pkg_path.parent))
    spec = importlib.util.spec_from_file_location("doctor._common", COMMON_PATH)
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)
    return module


class CountRealFeatures(unittest.TestCase):
    def _count(self, names: list[str]) -> int:
        common = _load_common()
        with tempfile.TemporaryDirectory() as td:
            features = Path(td) / "features"
            features.mkdir()
            for name in names:
                (features / name).write_text("x", encoding="utf-8")
            return common.count_real_features(features)

    def test_spec_files_counted(self):
        self.assertEqual(self._count(["01-login.md", "02-logout.md"]), 2)

    def test_plan_and_eval_excluded(self):
        self.assertEqual(
            self._count(["01-login.md", "01-login.plan.md", "01-login.eval.md"]), 1
        )

    def test_critic_artifact_excluded(self):
        # 종전 버그: .plan.critic.md 는 endswith(".plan.md") 에 안 걸려 카운트됐다
        self.assertEqual(self._count(["01-login.md", "01-login.plan.critic.md"]), 1)

    def test_rework_artifacts_excluded(self):
        self.assertEqual(
            self._count(
                ["01-login.md", "01-login.plan.r1.md", "01-login.eval.r1.md"]
            ),
            1,
        )

    def test_no_nn_prefix_excluded(self):
        self.assertEqual(self._count(["README.md", "01-login.md"]), 1)

    def test_missing_dir_returns_zero(self):
        common = _load_common()
        self.assertEqual(
            common.count_real_features(Path("/nonexistent/features")), 0
        )


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: 실패 확인**

Run: `cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills && python3 -m pytest tests/tools/test_doctor_feature_count.py -v`
Expected: `test_critic_artifact_excluded` 와 `test_rework_artifacts_excluded` 가 FAIL (현재 구현은 2 를 반환), 나머지 PASS.

- [ ] **Step 3: `_common.py` 구현 교체**

`dp-skills/tools/doctor/_common.py` 의 `_FEATURE_NAME_RE`(244행)~`count_real_features`(247-261행)를 다음으로 교체:

```python
_FEATURE_NAME_RE = re.compile(r"^\d{2}-")

# feature 본문 명세 = NN-{slug}.md — 점이 확장자 1개뿐인 파일명만.
# 파생 산출물 (.plan.md·.eval.md·.plan.critic.md·.plan.r{N}.md 등) 은
# 점이 2개 이상이라 자동 제외된다.
_FEATURE_SPEC_RE = re.compile(r"^\d{2}-[^.]+\.md$")


def is_feature_spec(p: Path) -> bool:
    """features/NN-{slug}.md 본문 명세 여부 — 카운트·목록의 단일 규칙."""
    return p.is_file() and bool(_FEATURE_SPEC_RE.match(p.name))


def count_real_features(features_dir: Path) -> int:
    """features/NN-{slug}.md 본문 명세 수. is_feature_spec 단일 규칙 사용.

    (이전엔 .plan.md/.eval.md 만 endswith 제외해 .plan.critic.md·.rN.md
    파생 산출물이 카운트를 부풀렸다 — analyzed 일관성·drift 검사 오작동.)"""
    if not features_dir.is_dir():
        return 0
    return sum(1 for p in features_dir.iterdir() if is_feature_spec(p))
```

`_FEATURE_NAME_RE` 는 다른 곳에서 참조될 수 있으므로 유지한다 (`grep -n "_FEATURE_NAME_RE" tools/doctor/*.py` 로 확인 후, `count_real_features` 외 사용처가 없으면 제거해도 된다).

- [ ] **Step 4: `integrity.py` 의 `_list_feature_md` 를 같은 술어로 교체**

`dp-skills/tools/doctor/integrity.py:1230-1240` 의 `_list_feature_md` 를 교체:

```python
def _list_feature_md(features_dir: Path) -> list[Path]:
    """features/NN-{slug}.md 본문 파일만 — is_feature_spec 단일 규칙."""
    if not features_dir.is_dir():
        return []
    return sorted(p for p in features_dir.glob("*.md") if is_feature_spec(p))
```

`integrity.py:16` 의 `from doctor._common import (...)` 목록에 `is_feature_spec` 추가.

- [ ] **Step 5: 테스트 통과 확인 + 전체 회귀**

Run: `cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills && python3 -m pytest tests/tools/test_doctor_feature_count.py -v && python3 -m pytest tests/tools -q`
Expected: 신규 테스트 전부 PASS, 기존 스위트 전부 PASS (특히 `test_doctor_project_hygiene.py` — `_list_feature_md` 규칙 변경 영향 확인).

- [ ] **Step 6: Commit**

```bash
git add dp-skills/tools/doctor/_common.py dp-skills/tools/doctor/integrity.py dp-skills/tests/tools/test_doctor_feature_count.py
git commit -m "fix(doctor): feature 카운트 단일 술어 is_feature_spec 도입

.plan.critic.md·.rN.md 파생 산출물이 feature 수에 섞여
drift 기준값을 부풀리던 문제 수정 (Presend-Admin 실데이터 확인).

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: 역방향 drift 검사 + `--fix` 재기록

**Files:**
- Modify: `dp-skills/tools/doctor/integrity.py:946-961` (기존 inline drift 블록을 함수로 추출 + 역방향 추가)
- Test: `dp-skills/tests/tools/test_doctor_feature_count.py` (Task 1 파일에 클래스 추가)

- [ ] **Step 1: 실패하는 테스트 작성**

`test_doctor_feature_count.py` 에 추가 (파일 상단 `_load_common` 아래에 `_load_integrity` 를 `tests/tools/test_doctor_project_hygiene.py:20-34` 와 동일 패턴으로 복사):

```python
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


class FeatureCountDrift(unittest.TestCase):
    def _run(self, last_count: int, actual: int, analyzed: bool = True):
        """state 더미 + features 더미로 check_feature_count_drift 실행."""
        mod = _load_integrity()
        self.td = tempfile.TemporaryDirectory()
        root = Path(self.td.name)
        state_yml = root / ".agent-state.yml"
        state_yml.write_text(
            f"schema: v1.3\nanalyzed: {str(analyzed).lower()}\n"
            f"last_analyzed_features: {last_count}\n",
            encoding="utf-8",
        )
        state_data = {
            "analyzed": analyzed,
            "last_analyzed_features": last_count,
        }
        results = mod.check_feature_count_drift(
            state_data, actual, state_yml, "myproj"
        )
        return results, state_yml

    def tearDown(self):
        if hasattr(self, "td"):
            self.td.cleanup()

    def test_forward_drift_warns(self):
        results, _ = self._run(last_count=3, actual=5)
        self.assertEqual(len(results), 1)
        self.assertEqual(results[0].level, "WARN")
        self.assertIn("3 → 5", results[0].message)

    def test_within_tolerance_silent(self):
        results, _ = self._run(last_count=3, actual=4)
        self.assertEqual(results, [])

    def test_reverse_drift_warns(self):
        # 기록값 > 실제 — 과거 카운트 오염 또는 feature 삭제
        results, _ = self._run(last_count=44, actual=32)
        self.assertEqual(len(results), 1)
        self.assertEqual(results[0].level, "WARN")
        self.assertIsNotNone(results[0].fix)

    def test_reverse_drift_fix_rewrites_state(self):
        results, state_yml = self._run(last_count=44, actual=32)
        ok, msg = results[0].fix()
        self.assertTrue(ok, msg)
        self.assertIn(
            "last_analyzed_features: 32",
            state_yml.read_text(encoding="utf-8"),
        )

    def test_not_analyzed_skips(self):
        results, _ = self._run(last_count=44, actual=32, analyzed=False)
        self.assertEqual(results, [])

    def test_missing_field_skips(self):
        mod = _load_integrity()
        results = mod.check_feature_count_drift(
            {"analyzed": True}, 5, Path("/tmp/none.yml"), "myproj"
        )
        self.assertEqual(results, [])
```

- [ ] **Step 2: 실패 확인**

Run: `cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills && python3 -m pytest tests/tools/test_doctor_feature_count.py -v`
Expected: `FeatureCountDrift` 전부 FAIL — `AttributeError: ... has no attribute 'check_feature_count_drift'`.

- [ ] **Step 3: 함수 추출 + 역방향 구현**

`dp-skills/tools/doctor/integrity.py` 에 신규 함수 추가 (기존 `check_eval_md_pairing` 근처, 1260행 부근):

```python
def _make_recount_fix(state_yml: Path, actual: int):
    """last_analyzed_features 를 실제 카운트로 재기록하는 fix callable."""

    def fix() -> tuple[bool, str]:
        try:
            text = state_yml.read_text(encoding="utf-8")
            new, n = re.subn(
                r"(?m)^last_analyzed_features:\s*\d+\s*$",
                f"last_analyzed_features: {actual}",
                text,
            )
            if n != 1:
                return False, f"last_analyzed_features 행 매칭 {n}건 — 수동 수정 필요"
            state_yml.write_text(new, encoding="utf-8")
            return True, f"last_analyzed_features → {actual} 재기록"
        except Exception as e:
            return False, f"재기록 실패: {e}"

    return fix


def check_feature_count_drift(
    state_data: dict, feature_count: int, state_yml: Path, project: str
) -> list[Result]:
    """features 카운트 drift — 증가(재분석 권장)·감소(기준값 오염) 양방향.

    감소 방향은 과거 카운트 규칙 버그(.eval.md 등 파생 산출물 포함 기록)
    또는 feature 삭제로 발생 — 기준값이 부풀면 증가 감지가 무력화된다.
    """
    last_count = state_data.get("last_analyzed_features")
    if not state_data.get("analyzed", False) or not isinstance(last_count, int):
        return []

    if feature_count > last_count + 1:
        return [
            Result(
                Result.WARN,
                f"{project} drift",
                f"features {last_count} → {feature_count} (증가 {feature_count - last_count})",
                "`/dp-skills:analyze --regen-agents` 로 agents/*.md 재생성 권장",
            )
        ]
    if feature_count < last_count:
        return [
            Result(
                Result.WARN,
                f"{project} drift",
                f"last_analyzed_features {last_count} > 실제 본문 명세 {feature_count} — "
                "과거 카운트 규칙 오염 또는 feature 삭제",
                "기준값이 부풀어 있으면 증가 drift 감지가 무력화됨",
                fix=_make_recount_fix(state_yml, feature_count),
            )
        ]
    return []
```

- [ ] **Step 4: `check_project` 의 inline 블록을 호출로 교체**

`integrity.py:946-961` 의 기존 블록:

```python
    # Drift 감지 (optional 필드 — 없으면 skip)
    last_count = state_data.get("last_analyzed_features")
    analyzed_at = state_data.get("analyzed_at")
    docs_fetched_at = state_data.get("docs_last_fetched_at")

    # features 증가 drift
    if analyzed_flag and isinstance(last_count, int):
        if feature_count > last_count + 1:
            results.append(
                Result(
                    Result.WARN,
                    f"{project} drift",
                    f"features {last_count} → {feature_count} (증가 {feature_count - last_count})",
                    "`/dp-skills:analyze --regen-agents` 로 agents/*.md 재생성 권장",
                )
            )
```

을 다음으로 교체 (`analyzed_at`·`docs_fetched_at` 는 후속 검사에서 쓰이므로 유지):

```python
    # Drift 감지 (optional 필드 — 없으면 skip)
    analyzed_at = state_data.get("analyzed_at")
    docs_fetched_at = state_data.get("docs_last_fetched_at")

    results.extend(
        check_feature_count_drift(state_data, feature_count, state_yml, project)
    )
```

- [ ] **Step 5: 테스트 통과 + 전체 회귀**

Run: `cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills && python3 -m pytest tests/tools/test_doctor_feature_count.py -v && python3 -m pytest tests/tools -q`
Expected: 전부 PASS.

- [ ] **Step 6: Commit**

```bash
git add dp-skills/tools/doctor/integrity.py dp-skills/tests/tools/test_doctor_feature_count.py
git commit -m "feat(doctor): 역방향 feature drift 감지 + --fix 재기록

기록값 > 실제 카운트인 프로젝트는 증가 감지가 무력화되므로
WARN + last_analyzed_features 자동 재기록 fix 를 제공한다.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: 명세 문서 카운트 규칙 통일

**Files:**
- Modify: `dp-skills/skills/context/lifecycle/state-schema.md:26,95-100`
- Modify: `dp-skills/skills/analyze/references/agents-update.md:174,182`
- Modify: `dp-skills/skills/project/SKILL.md:123,131`
- Modify: `dp-skills/skills/context/lifecycle/projects/GUIDE.md:73`

통일 문구 (canonical): **"features/NN-{slug}.md 본문 명세만 — 파생 산출물(`.plan.md`·`.eval.md`·`.plan.critic.md`·`.r{N}.md`)은 파일명에 점이 2개 이상이므로 제외"**

- [ ] **Step 1: state-schema.md 수정**

26행:

```markdown
last_analyzed_features: 3            # 그 시점 features/*.md 개수 (.plan.md 제외)
```

→

```markdown
last_analyzed_features: 3            # 그 시점 features/NN-{slug}.md 본문 명세 개수 (파생 산출물 제외 — 아래 §last_analyzed_features)
```

95-100행 섹션 본문:

```markdown
마지막 analyze 실행 시점의 `features/*.md` (`.plan.md` 제외) 개수.

- 부재 → drift 체크 skip.
- 존재 + 현재 features 개수가 `last_analyzed_features + 1` 초과 → doctor 가 "features 가 증가함, `--regen-agents` 권장" WARN.
```

→

```markdown
마지막 analyze 실행 시점의 features 본문 명세 개수. **카운트 규칙**: `features/NN-{slug}.md` — `NN-` 2자리 숫자 접두 + 파일명에 점이 확장자 1개뿐인 파일만. 파생 산출물(`.plan.md`·`.eval.md`·`.plan.critic.md`·`.r{N}.md`)은 점이 2개 이상이므로 제외 (doctor `is_feature_spec` 과 동일 규칙).

- 부재 → drift 체크 skip.
- 존재 + 현재 본문 명세 개수가 `last_analyzed_features + 1` 초과 → doctor 가 "features 가 증가함, `--regen-agents` 권장" WARN.
- 존재 + 현재 본문 명세 개수가 `last_analyzed_features` **미만** → doctor 가 "기준값 오염 또는 feature 삭제" WARN (`--fix` 로 재기록 가능).
```

- [ ] **Step 2: agents-update.md 수정 (analyze 가 기록하는 지점 — 가장 중요)**

174행:

```markdown
   - `last_analyzed_features: {현재 features/*.md (.plan.md 제외) 개수}`
```

→

```markdown
   - `last_analyzed_features: {현재 features/NN-{slug}.md 본문 명세 개수 — 파일명에 점이 확장자 1개뿐인 파일만. .plan.md·.eval.md·.plan.critic.md·.r{N}.md 등 파생 산출물 제외}`
```

182행 ("features 개수가 `last_analyzed_features + 1` 초과 → ...") 아래에 한 줄 추가:

```markdown
- features 개수가 `last_analyzed_features` 미만 → "기준값 오염/삭제" WARN (`doctor --fix` 재기록)
```

- [ ] **Step 3: project/SKILL.md 수정**

123행의 `count(features/*.md) > last_analyzed_features + 1` →
`count(features/NN-{slug}.md 본문 명세 — 파생 산출물 제외) > last_analyzed_features + 1`

131행의 `features/*.md` (`.plan.md` 제외) 가 0건이면 →
`features/NN-{slug}.md` 본문 명세가 0건이면

- [ ] **Step 4: GUIDE.md 표 수정**

73행 표 행 `| features_count > last_analyzed_features + 1 | ... |` 바로 아래에 행 추가:

```markdown
| `features_count < last_analyzed_features` | 기준값 오염(과거 파생 산출물 포함 기록) 또는 feature 삭제 | `doctor --fix` 로 재기록 |
```

- [ ] **Step 5: 잔여 불일치 sweep**

Run: `grep -rn "plan.md 제외" /Users/jay-p/Projects/deali-skills-plugin/dp-skills/skills/ /Users/jay-p/Projects/deali-skills-plugin/dp-skills/docs/`
Expected: 0건 (남은 곳이 있으면 동일 문구로 수정).

- [ ] **Step 6: docs reference 동기화 + 린트**

Run: `python3 /Users/jay-p/Projects/deali-skills-plugin/dp-skills/tools/docs_build.py && python3 -m pytest /Users/jay-p/Projects/deali-skills-plugin/dp-skills/tests/tools/test_docs_build.py -q`
Expected: drift 없음, PASS. (pre-push hook 이 동일 검사를 강제하므로 여기서 선반영.)

- [ ] **Step 7: Commit**

```bash
git add dp-skills/skills/ dp-skills/docs/
git commit -m "docs(state): feature 카운트 규칙 단일화 — 파생 산출물 제외 명시

state-schema·agents-update·project SKILL·GUIDE 의 카운트 문구를
doctor is_feature_spec 규칙과 동일하게 통일.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 4: 버전 bump + PR

**Files:**
- Modify: `dp-skills/.claude-plugin/plugin.json` (`"version": "0.16.6"` → 현재 최신 +1 patch)

- [ ] **Step 1: plugin.json bump**

`"version": "0.16.6"` → `"version": "0.16.7"` (이 plan 실행 시점의 최신 버전 기준 +1 — 다른 PR 이 먼저 머지됐으면 그 다음 번호).

- [ ] **Step 2: 전체 테스트 최종 확인**

Run: `cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills && python3 -m pytest tests/tools -q`
Expected: 전부 PASS.

- [ ] **Step 3: Commit + PR**

```bash
git add dp-skills/.claude-plugin/plugin.json
git commit -m "chore(release): bump 0.16.7

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
git push -u origin HEAD
gh pr create --base main --title "fix(doctor): feature 카운트 정합성 회복 (0.16.7)" --body "$(cat <<'EOF'
## 요약
- feature 카운트 단일 술어 `is_feature_spec` 도입 — `.plan.critic.md`·`.r{N}.md` 파생 산출물 제외
- 역방향 drift 검사 (기록값 > 실제) + `doctor --fix` 재기록
- 명세 4개 문서의 카운트 문구를 doctor 구현과 통일

## 배경
nimda Presend-Admin 에서 `last_analyzed_features: 44` vs 실제 본문 명세 32건 — 부풀려진 기준값으로 증가 drift 감지가 무력화돼 있었다.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

머지 전략: squash (linear history — CLAUDE.md 컨벤션).
