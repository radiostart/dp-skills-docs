# Cross-domain Boundary 모드 역이식 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** pilot 커밋 [43d9daf](https://github.com/radiostart/claude-plugins/commit/43d9daf444d7ee6b490222faa9175f53dd90a1bd) (cross-domain 경계 계약 — boundary 모드 + 자동 로드 배선) 를 dp-skills 에 역이식한다 — discover 에 외부 도메인 reference **생산자** 단계와 `--boundary` 경량 학습 모드를 추가하고, orchestrate-load 가 경계 문서를 정·역방향 자동 로드하며, doctor 가 부분 커버 상태를 INFO 로 표시한다.

**Architecture:** 3 레이어 + 소비자 병기. (1) discover 스킬 md — 2홉 탐색 중 외부 도메인 클래스 reference 를 분류해 MANIFEST `## 외부 도메인 reference` 표에 기록 (기존 doctor·analyze·create-feature 가 이미 소비하는 표의 생산자 완성), `--boundary B --from A` 모드로 A 가 호출하는 B 표면만 `workspace/{TEAM}/context/boundaries/{A}--{B}.md` 에 추출. (2) `tools/orchestrate-load.py` — 활성 도메인 기준 경계 문서 정방향(`{A}--*.md`)·역방향(`*--{A}.md`) 자동 로드 (상한 6) + 미학습 외부 의존 boundary 처방 힌트 (상한 3). (3) `tools/doctor/integrity.py` — 외부 reference 행이 경계 문서로 부분 커버되면 INFO.

**Tech Stack:** Python 3 (stdlib only), pytest (`/opt/homebrew/bin/pytest` — `python3 -m pytest` 불가), 선언적 SKILL.md (언어 하드코드 금지 원칙).

---

## 검토 결론 — pilot 커밋 항목별 적용 판정

| pilot 커밋 항목 | dp-skills 적용 | 적응 포인트 |
| --- | --- | --- |
| `--boundary B --from A` 모드 (learn) | ✅ discover 에 추가 | 경로 `workspace/{TEAM}/context/boundaries/` (멀티팀), 기존 미리 보기 승인 게이트·Abort 계약·`--dry-run` 동일 적용, 계층 식별자 `/`→`__` 토큰 치환 |
| orchestrate-load 경계 로드+힌트 | ✅ 거의 1:1 | `workspace/{team}/context/` 경로, `build_load_plan` 에 team 인자 이미 존재 |
| doctor boundary-covered INFO | ✅ 거의 1:1 | `check_workspace_external_domain_section` 이 이미 존재 (98d613d 역이식분) — pilot diff 그대로 적용 가능 |
| 외부 reference 표 추천 컬럼 boundary 병기 | ✅ | **선행 갭**: dp-skills discover 에는 표 생산자 (pilot Phase 2·5 상당) 자체가 없음 → 본 계획에 포함 |
| transaction nesting (pilot Phase 3 재사용) | ⚠️ 경량 적용 | dp-skills 에 Phase 3 가 없음 — boundary 모드 내부 전용 경량 Grep 휴리스틱으로만 (scope 본문 갱신하는 Phase 4 상당은 YAGNI 제외) |
| docs (how-to) | ✅ | `docs/how-to/domain-discover.md` + `docs_build.py` 재생성 |

**선행 갭 (이번 계획의 절반):** 98d613d 가 pilot→dp-skills 로 **소비자만** 역이식했다 — analyze·create-feature 는 `## 외부 도메인 reference` 표를 lookup 하고 doctor 는 스키마를 검증하지만, 생산자인 discover 에 detect·기록 단계가 없다 (doctor INFO 메시지 "첫 cross-domain reference detect 시 `/dp-skills:discover` 가 자동 작성" 은 현재 공약 상태). boundary 처방 힌트 체인 (detect → 표 → 처방 → 경계 학습 → 자동 로드) 이 성립하려면 생산자가 필요하다.

**고정 인터페이스 (전 Task 공통):**

- 호출: `/dp-skills:discover --boundary <B> --from <A> [--team TEAM] [--dry-run]` (boundary 모드에서 entry 생략)
- 산출: `workspace/{TEAM}/context/boundaries/{A}--{B}.md` (본문 ≤ 150 줄, 파일명 구분자 `--` 고정)
- 계층 식별자 (`backend/orders`) 는 파일명 토큰에서 `/` → `__` 치환 (`backend__orders--billing.md`). 치환 함수명 `boundary_token`. **알려진 제약 (Task 1 품질 리뷰 발견)**: orchestrate-load `main()` 의 기존 도메인 검증 (`has_path_traversal`) 이 `/` 포함 활성 도메인을 무시하므로 계층 도메인의 정방향 자동 로드는 CLI 경로에서 미작동 — `build_load_plan` 단위 동작과 MANIFEST ext_domain 쪽 토큰화는 유효. main() 완화는 scope/rules 로드 동작까지 영향이 가는 기존 제약 해소라 본 계획 범위 외. **→ 해소됨 (후속 사이클)**: domain 전용 검증 `has_domain_traversal` (내부 `/` 허용, `..`·`\`·빈 세그먼트 거부) 도입으로 CLI 경로에서도 계층 도메인 자동 로드 작동 — `MainHierarchicalDomain` e2e 로 고정.
- MANIFEST 표 스키마는 doctor 가 강제하는 3컬럼 고정: `| 추정 도메인 | 클래스 (개수) | 추천 후속 학습 |` (integrity.py:1175 `EXT_EXPECTED_HEADERS`)
- 경계 문서는 MANIFEST `## 도메인 분류` 에 등록하지 않는다 — 로드는 orchestrate-load 글롭이 담당.

---

### Task 1: orchestrate-load — `parse_manifest_external_refs` + `boundary_token` (TDD)

**Files:**
- Modify: `dp-skills/tools/orchestrate-load.py` (`def read_focus(` 직전, ~line 396)
- Test: `dp-skills/tests/tools/test_orchestrate_load.py` (`class BuildLoadPlanIntegration` 직전, ~line 241)

- [ ] **Step 1: 실패하는 테스트 작성**

`test_orchestrate_load.py` 의 `class BuildLoadPlanIntegration` 정의 직전에 추가:

```python
class ParseManifestExternalRefs(unittest.TestCase):
    def _write(self, body: str) -> Path:
        f = tempfile.NamedTemporaryFile("w", suffix=".md", delete=False, encoding="utf-8")
        f.write(body)
        f.close()
        return Path(f.name)

    def test_extracts_rows(self):
        p = self._write(
            "## 외부 도메인 reference (discover 미완료)\n\n"
            "| 추정 도메인 | 클래스 (개수) | 추천 후속 학습 |\n"
            "| --- | --- | --- |\n"
            "| schoice | Schoice::Order, Schoice::Box (2) | `/dp-skills:discover app/models/schoice/` (auto) |\n"
            "| billing | Billing::Invoice (1) | `/dp-skills:discover app/services/billing/` (auto) |\n"
        )
        refs = m.parse_manifest_external_refs(p)
        self.assertEqual([r[0] for r in refs], ["schoice", "billing"])
        self.assertIn("Schoice::Order", refs[0][1])

    def test_no_section_returns_empty(self):
        p = self._write("## 도메인 분류\n\n| orders | `orders.md` | x |\n")
        self.assertEqual(m.parse_manifest_external_refs(p), [])

    def test_header_row_skipped(self):
        p = self._write(
            "## 외부 도메인 reference\n\n"
            "| 추정 도메인 | 클래스 (개수) | 추천 후속 학습 |\n"
            "| --- | --- | --- |\n"
        )
        self.assertEqual(m.parse_manifest_external_refs(p), [])

    def test_missing_file_returns_empty(self):
        self.assertEqual(
            m.parse_manifest_external_refs(Path("/nonexistent/MANIFEST.md")), []
        )


class BoundaryToken(unittest.TestCase):
    def test_flat_unchanged(self):
        self.assertEqual(m.boundary_token("orders"), "orders")

    def test_hierarchical_replaced(self):
        self.assertEqual(m.boundary_token("backend/orders"), "backend__orders")
```

- [ ] **Step 2: 실패 확인**

Run: `/opt/homebrew/bin/pytest dp-skills/tests/tools/test_orchestrate_load.py::ParseManifestExternalRefs dp-skills/tests/tools/test_orchestrate_load.py::BoundaryToken -v`
Expected: ERROR/FAIL — `AttributeError: module ... has no attribute 'parse_manifest_external_refs'`

- [ ] **Step 3: 구현**

`orchestrate-load.py` 의 `def read_focus(` 직전에 추가:

```python
def parse_manifest_external_refs(manifest_md: Path) -> list[tuple[str, str]]:
    """MANIFEST `## 외부 도메인 reference` 표에서 (추정 도메인, 클래스 목록 문자열) 추출.

    헤더 서픽스 (예: "(discover 미완료)") 허용. discover 가 학습 완료 도메인 행을
    제거하므로 반환 목록 = 아직 학습되지 않은 외부 의존. 매칭 실패 시 빈 리스트 (graceful).
    """
    if not manifest_md.is_file():
        return []
    try:
        text = manifest_md.read_text(encoding="utf-8")
    except Exception:
        return []
    sec = re.search(
        r"^##\s*외부\s*도메인\s*reference[^\n]*\n([\s\S]*?)(?:\n##\s|\Z)", text, re.M
    )
    if not sec:
        return []
    refs: list[tuple[str, str]] = []
    for line in sec.group(1).splitlines():
        row = re.match(r"^\|\s*`?([^`|\s]+)`?\s*\|\s*([^|]+?)\s*\|", line)
        if not row:
            continue
        dom = row.group(1).strip()
        classes = row.group(2).strip()
        # 헤더 행("추정 도메인" → 공백 전 "추정")·구분선 행 제외
        if dom == "추정" or set(dom) <= {"-", ":"}:
            continue
        refs.append((dom, classes))
    return refs


# 경계 계약 문서 로드 상한 — 초과분은 hint 로 안내 (토큰 경제)
MAX_BOUNDARY_DOCS = 6


def boundary_token(domain: str) -> str:
    """계층 도메인 식별자 (backend/orders) 를 boundary 파일명 토큰으로 변환."""
    return domain.replace("/", "__")
```

- [ ] **Step 4: 통과 확인**

Run: `/opt/homebrew/bin/pytest dp-skills/tests/tools/test_orchestrate_load.py -v`
Expected: 전체 PASS (기존 테스트 포함 회귀 없음)

- [ ] **Step 5: 커밋**

```bash
git add dp-skills/tools/orchestrate-load.py dp-skills/tests/tools/test_orchestrate_load.py
git commit -m "feat(orchestrate): MANIFEST 외부 도메인 reference 파서 + boundary 토큰

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: orchestrate-load — 경계 문서 정·역방향 자동 로드 + 미학습 힌트 (TDD)

**Files:**
- Modify: `dp-skills/tools/orchestrate-load.py` — `build_load_plan` 의 5) rules 절 (line ~556-561) 과 6) generator 절 (line ~563) 사이
- Test: `dp-skills/tests/tools/test_orchestrate_load.py` — `class BuildLoadPlanIntegration` 내부

- [ ] **Step 1: 실패하는 테스트 작성**

`BuildLoadPlanIntegration` 클래스 안 (예: `test_planner_phase_skips_conventions_files` 류 기존 메서드 뒤) 에 추가:

```python
    def _ws_with_boundaries(self, td: str) -> Path:
        """boundaries/ 경계 계약 문서가 있는 workspace (팀 T, 활성 도메인 orders)."""
        ws = Path(td)
        bdir = ws / "T" / "context" / "boundaries"
        bdir.mkdir(parents=True)
        (bdir / "orders--payments.md").write_text("# 경계: orders → payments\n", encoding="utf-8")
        (bdir / "shipping--orders.md").write_text("# 경계: shipping → orders\n", encoding="utf-8")
        (bdir / "shipping--billing.md").write_text("# 경계: shipping → billing\n", encoding="utf-8")
        (ws / "T" / "projects" / "P").mkdir(parents=True)
        return ws

    def test_boundary_docs_loaded_both_directions(self):
        """domain=orders 면 orders--*.md (정방향) 와 *--orders.md (역방향) 만 로드."""
        with tempfile.TemporaryDirectory() as td:
            ws = self._ws_with_boundaries(td)
            files, hints, _ = m.build_load_plan(
                workspace=ws, team="T", project="P",
                domain="orders", tdd=False, phase="planner",
            )
            self.assertIn("workspace/T/context/boundaries/orders--payments.md", files)
            self.assertIn("workspace/T/context/boundaries/shipping--orders.md", files)
            self.assertNotIn("workspace/T/context/boundaries/shipping--billing.md", files)
            self.assertTrue(any("경계 계약" in h for h in hints))

    def test_no_boundaries_dir_silent(self):
        with tempfile.TemporaryDirectory() as td:
            ws = Path(td)
            (ws / "T" / "projects" / "P").mkdir(parents=True)
            files, _, _ = m.build_load_plan(
                workspace=ws, team="T", project="P",
                domain="orders", tdd=False, phase="planner",
            )
            self.assertFalse(any("boundaries" in f for f in files))

    def test_hierarchical_domain_token_match(self):
        """계층 식별자 (backend/orders) 는 파일명 토큰 backend__orders 로 매칭.

        주의: main() 의 도메인 검증이 `/` 를 차단해 CLI 경로에선 미도달 —
        build_load_plan 라이브러리 동작 보장용 (고정 인터페이스 § 알려진 제약).
        """
        with tempfile.TemporaryDirectory() as td:
            ws = Path(td)
            bdir = ws / "T" / "context" / "boundaries"
            bdir.mkdir(parents=True)
            (bdir / "backend__orders--billing.md").write_text("# b\n", encoding="utf-8")
            (ws / "T" / "projects" / "P").mkdir(parents=True)
            files, _, _ = m.build_load_plan(
                workspace=ws, team="T", project="P",
                domain="backend/orders", tdd=False, phase="planner",
            )
            self.assertIn(
                "workspace/T/context/boundaries/backend__orders--billing.md", files
            )

    def test_external_refs_hint_with_boundary_prescription(self):
        """MANIFEST 외부 도메인 reference 행 → 미학습 안내 + boundary discover 처방 hint."""
        with tempfile.TemporaryDirectory() as td:
            ws = Path(td)
            (ws / "T" / "context").mkdir(parents=True)
            (ws / "T" / "context" / "MANIFEST.md").write_text(
                "## 도메인 분류\n\n"
                "| orders | `scope/orders.md` | 주문 |\n\n"
                "## 외부 도메인 reference (discover 미완료)\n\n"
                "| 추정 도메인 | 클래스 (개수) | 추천 후속 학습 |\n"
                "| --- | --- | --- |\n"
                "| schoice | Schoice::Order (1) | `/dp-skills:discover app/models/schoice/` (auto) |\n",
                encoding="utf-8",
            )
            (ws / "T" / "projects" / "P").mkdir(parents=True)
            _, hints, _ = m.build_load_plan(
                workspace=ws, team="T", project="P",
                domain="orders", tdd=False, phase="planner",
            )
            joined = " ".join(hints)
            self.assertIn("schoice", joined)
            self.assertIn("--boundary", joined)

    def test_boundary_covered_ref_not_hinted(self):
        """경계 문서가 이미 있는 외부 ref 는 처방 hint 를 내지 않는다."""
        with tempfile.TemporaryDirectory() as td:
            ws = Path(td)
            bdir = ws / "T" / "context" / "boundaries"
            bdir.mkdir(parents=True)
            (bdir / "orders--schoice.md").write_text("# b\n", encoding="utf-8")
            (ws / "T" / "context" / "MANIFEST.md").write_text(
                "## 외부 도메인 reference (discover 미완료)\n\n"
                "| 추정 도메인 | 클래스 (개수) | 추천 후속 학습 |\n"
                "| --- | --- | --- |\n"
                "| schoice | Schoice::Order (1) | `/dp-skills:discover app/models/schoice/` (auto) |\n",
                encoding="utf-8",
            )
            (ws / "T" / "projects" / "P").mkdir(parents=True)
            _, hints, _ = m.build_load_plan(
                workspace=ws, team="T", project="P",
                domain="orders", tdd=False, phase="planner",
            )
            self.assertFalse(any("미학습 외부 도메인 의존" in h for h in hints))
```

- [ ] **Step 2: 실패 확인**

Run: `/opt/homebrew/bin/pytest dp-skills/tests/tools/test_orchestrate_load.py::BuildLoadPlanIntegration -v -k "boundary or external_refs"`
Expected: 신규 5건 FAIL (boundary 파일이 files 에 없음 / hint 없음)

- [ ] **Step 3: 구현**

`build_load_plan` 의 5) rules 절과 6) generator 절 사이에 삽입:

```python
    # 5-bis) cross-domain — 경계 계약 문서(boundaries/) 로드 + 외부 도메인 reference 힌트
    #    정방향({domain}--B: 내가 호출하는 표면) 과 역방향(B--{domain}: 남이 나를
    #    호출하는 표면 — 영향 분석용) 모두 로드한다. 문서는 호출 표면만 담아 작게 유지.
    if domain:
        token = boundary_token(domain)
        bdir = workspace / team / "context" / "boundaries"
        matched: list[Path] = []
        if bdir.is_dir():
            matched = sorted(
                p
                for p in bdir.glob("*.md")
                if p.name.startswith(f"{token}--") or p.stem.endswith(f"--{token}")
            )
            for p in matched[:MAX_BOUNDARY_DOCS]:
                files.append(f"workspace/{team}/context/boundaries/{p.name}")
                hints.append(f"경계 계약 로드: boundaries/{p.name}")
            if len(matched) > MAX_BOUNDARY_DOCS:
                hints.append(
                    f"경계 계약 {len(matched)}건 중 {MAX_BOUNDARY_DOCS}건만 로드 — "
                    "나머지는 필요 시 수동 Read"
                )
        covered = {p.name for p in matched}
        ext_refs = parse_manifest_external_refs(manifest_abs)
        shown = 0
        for ext_domain, classes in ext_refs:
            if ext_domain == domain:
                continue
            if f"{token}--{boundary_token(ext_domain)}.md" in covered:
                continue  # 경계 계약 문서가 이미 커버
            if shown >= 3:
                hints.append(
                    f"외부 도메인 reference 총 {len(ext_refs)}건 — "
                    "MANIFEST `## 외부 도메인 reference` 표 참조"
                )
                break
            hints.append(
                f"미학습 외부 도메인 의존: {ext_domain} ({classes}) — "
                f"경계만 필요하면 `/dp-skills:discover --boundary {ext_domain} --from {domain}`, "
                "전체 학습은 표의 추천 명령 참조"
            )
            shown += 1
```

주의: `manifest_abs` 는 같은 함수 1) 절 (line ~485) 에서 이미 정의돼 있다 — 재사용.

- [ ] **Step 4: 통과 확인**

Run: `/opt/homebrew/bin/pytest dp-skills/tests/tools/test_orchestrate_load.py -v`
Expected: 전체 PASS

- [ ] **Step 5: 커밋**

```bash
git add dp-skills/tools/orchestrate-load.py dp-skills/tests/tools/test_orchestrate_load.py
git commit -m "feat(orchestrate): 경계 계약 문서 정·역방향 자동 로드 + 미학습 boundary 처방 힌트

활성 도메인 기준 boundaries/{A}--*.md (정방향) 와 *--{A}.md (역방향) 를
상한 6건 로드. MANIFEST 외부 도메인 reference 미학습 행은 boundary
처방 힌트로 안내 (상한 3건, 경계 문서 커버 행 제외).

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: doctor — 경계 문서 부분 커버 INFO (TDD)

**Files:**
- Modify: `dp-skills/tools/doctor/integrity.py:1214-1235` (`check_workspace_external_domain_section` 의 stale-row 루프)
- Create: `dp-skills/tests/tools/test_doctor_external_domain.py`

- [ ] **Step 1: 실패하는 테스트 작성**

`dp-skills/tests/tools/test_doctor_external_domain.py` 신규 (로드 헬퍼는 `test_doctor_project_hygiene.py` 와 동일 패턴):

```python
"""check_workspace_external_domain_section 테스트.

- 섹션 부재 → INFO (backward-compat)
- stale row (## 도메인 분류 에 이미 등록) → INFO
- 경계 계약 문서 (boundaries/*--{B}.md) 부분 커버 → INFO (0.17.0)
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
    """integrity.py 를 (의존성 import 포함) 로드. doctor 패키지 컨텍스트 흉내."""
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


doctor = _load_integrity()

MANIFEST_WITH_EXT = (
    "## 도메인 분류\n\n"
    "| 도메인 | 진입 파일 | 설명 |\n"
    "| --- | --- | --- |\n"
    "| orders | `scope/orders.md` | 주문 |\n\n"
    "## 외부 도메인 reference (discover 미완료)\n\n"
    "| 추정 도메인 | 클래스 (개수) | 추천 후속 학습 |\n"
    "| --- | --- | --- |\n"
    "| schoice | Schoice::Order (1) | `/dp-skills:discover app/models/schoice/` (auto) |\n"
)


class ExternalDomainSectionMissing(unittest.TestCase):
    def test_info_when_section_absent(self):
        with tempfile.TemporaryDirectory() as td:
            mf = Path(td) / "MANIFEST.md"
            mf.write_text("## 도메인 분류\n", encoding="utf-8")
            results = doctor.check_workspace_external_domain_section(mf)
            self.assertTrue(any(r.level == "INFO" for r in results))
            self.assertFalse(any(r.level == "ERROR" for r in results))


class ExternalDomainStaleRow(unittest.TestCase):
    def test_info_when_already_learned(self):
        with tempfile.TemporaryDirectory() as td:
            mf = Path(td) / "MANIFEST.md"
            mf.write_text(
                MANIFEST_WITH_EXT.replace("| schoice |", "| orders |"),
                encoding="utf-8",
            )
            results = doctor.check_workspace_external_domain_section(mf)
            info = " ".join(r.message for r in results if r.level == "INFO")
            self.assertIn("orders", info)
            self.assertIn("행 제거 권장", info)


class ExternalDomainBoundaryCovered(unittest.TestCase):
    """외부 reference 행의 도메인에 경계 계약 문서(boundaries/*--{B}.md)가 있으면 INFO."""

    def test_info_boundary_covered(self):
        with tempfile.TemporaryDirectory() as td:
            ctx = Path(td) / "context"
            (ctx / "boundaries").mkdir(parents=True)
            (ctx / "boundaries" / "orders--schoice.md").write_text(
                "# 경계: orders → schoice\n", encoding="utf-8"
            )
            (ctx / "MANIFEST.md").write_text(MANIFEST_WITH_EXT, encoding="utf-8")
            results = doctor.check_workspace_external_domain_section(ctx / "MANIFEST.md")
            errors = [r for r in results if r.level == "ERROR"]
            self.assertEqual(errors, [], f"ERROR 없어야 함: {[r.message for r in errors]}")
            info_messages = " ".join(r.message for r in results if r.level == "INFO")
            self.assertIn("경계 계약", info_messages)
            self.assertIn("schoice", info_messages)

    def test_no_boundary_dir_no_extra_info(self):
        with tempfile.TemporaryDirectory() as td:
            ctx = Path(td) / "context"
            ctx.mkdir(parents=True)
            (ctx / "MANIFEST.md").write_text(MANIFEST_WITH_EXT, encoding="utf-8")
            results = doctor.check_workspace_external_domain_section(ctx / "MANIFEST.md")
            self.assertFalse(any("경계 계약" in r.message for r in results))


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: 실패 확인**

Run: `/opt/homebrew/bin/pytest dp-skills/tests/tools/test_doctor_external_domain.py -v`
Expected: `ExternalDomainBoundaryCovered::test_info_boundary_covered` FAIL (경계 계약 INFO 미구현). 나머지 (부재 INFO·stale row) 는 기존 동작이라 PASS.

- [ ] **Step 3: 구현 — pilot diff 1:1 적용 (문구만 discover 로)**

`integrity.py:1214` 의 stale-row 루프를 다음으로 교체 (기존 코드):

```python
    # stale row 검증: 외부 도메인 표의 추정 도메인이 ## 도메인 분류 표에도 있으면 INFO
    domain_tables = _parse_md_tables_in_section(text, "## 도메인 분류")
    learned_domains: set[str] = set()
    if domain_tables:
        for row in domain_tables[0][1:]:
            if row and len(row) >= 1:
                learned_domains.add(row[0].strip().lower())

    for row in ext_table[1:]:
        if not row or len(row) < 1:
            continue
        estimated_domain = row[0].strip().lower()
        if estimated_domain and estimated_domain in learned_domains:
            results.append(
                Result(
                    Result.INFO,
                    "context/MANIFEST.md",
                    f"## 외부 도메인 reference 표: '{estimated_domain}' 은 이미 ## 도메인 분류 에 등록됨 — 해당 행 제거 권장 (idempotency)",
                )
            )

    return results
```

신규 코드:

```python
    # stale row 검증: 외부 도메인 표의 추정 도메인이 ## 도메인 분류 표에도 있으면 INFO
    domain_tables = _parse_md_tables_in_section(text, "## 도메인 분류")
    learned_domains: set[str] = set()
    if domain_tables:
        for row in domain_tables[0][1:]:
            if row and len(row) >= 1:
                learned_domains.add(row[0].strip().lower())

    boundaries_dir = manifest_path.parent / "boundaries"

    for row in ext_table[1:]:
        if not row or len(row) < 1:
            continue
        estimated_domain = row[0].strip().lower()
        if not estimated_domain:
            continue
        if estimated_domain in learned_domains:
            results.append(
                Result(
                    Result.INFO,
                    "context/MANIFEST.md",
                    f"## 외부 도메인 reference 표: '{estimated_domain}' 은 이미 ## 도메인 분류 에 등록됨 — 해당 행 제거 권장 (idempotency)",
                )
            )
            continue
        # 경계 계약 문서 (boundaries/{A}--{B}.md) 가 있으면 호출 표면은 커버된 상태
        if boundaries_dir.is_dir():
            covered = sorted(boundaries_dir.glob(f"*--{estimated_domain}.md"))
            if covered:
                names = ", ".join(p.name for p in covered)
                results.append(
                    Result(
                        Result.INFO,
                        "context/MANIFEST.md",
                        (
                            f"'{estimated_domain}' 은 경계 계약 문서로 부분 커버됨 ({names}) "
                            "— 호출 표면만 학습된 상태, 전체 discover 시 행이 제거됨"
                        ),
                    )
                )

    return results
```

- [ ] **Step 4: 통과 확인**

Run: `/opt/homebrew/bin/pytest dp-skills/tests/tools/test_doctor_external_domain.py dp-skills/tests/tools/test_doctor_known_domains.py dp-skills/tests/tools/test_doctor_project_hygiene.py -v`
Expected: 전체 PASS (doctor 회귀 없음)

- [ ] **Step 5: 커밋**

```bash
git add dp-skills/tools/doctor/integrity.py dp-skills/tests/tools/test_doctor_external_domain.py
git commit -m "feat(doctor): 외부 reference 행의 경계 계약 부분 커버 INFO

boundaries/*--{B}.md 가 존재하는 외부 도메인 reference 행은
호출 표면만 학습된 상태임을 INFO 로 표시. 기존 검사(부재 INFO·
stale row) 테스트도 함께 신설 (98d613d 당시 누락분).

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

> **구현 후 기록 (품질 리뷰 반영)**: Step 3 처방의 `glob(f"*--{estimated_domain}.md")` 는 사용자 편집 MANIFEST 셀 보간이라 셀에 `**` 가 오면 Python ≤3.12 pathlib ValueError 로 doctor 전체 크래시 (3.13+ 에서는 과매칭) — pilot 원본 diff 의 결함. 고정 `glob("*.md")` 1회 수집 + suffix 문자열 매칭 (계층 토큰 `/`→`__` 치환 포함) 으로 교체 구현됨 (후속 커밋, 회귀 가드 테스트 4건 동반).

---

### Task 4: discover — `references/cross-domain.md` 신규 (detect·MANIFEST 표·boundary 절차)

**Files:**
- Create: `dp-skills/skills/discover/references/cross-domain.md`

- [ ] **Step 1: 파일 작성**

전문 (pilot `cross-domain.md` 를 dp-skills 의 언어중립·멀티팀·scope/rules 구조로 적응):

````markdown
# discover — cross-domain detect & boundary 모드

`/dp-skills:discover` 의 외부 도메인 reference 추출·MANIFEST 갱신·boundary 모드 상세. 절차는 SKILL.md 본문에서 짧게 호출되며 상세는 여기에 둔다. 기존 `## 외부 의존 (API·MQ·DB)` 카테고리 (extraction.md §4 — 외부 *시스템*) 와 구분: 본 문서는 **같은 코드베이스 안의 다른 도메인** 참조를 다룬다.

---

## 외부 도메인 reference detect (단계 2 확장)

2홉 탐색 중 발견된 클래스/모듈 reference 를 내부 vs 외부로 분류한다.

- **내부 판정**: 진입점에서 추론된 본 도메인 namespace/패키지/모듈 prefix 와 일치 (예: 진입점이 `{source_root}/services/orders/` 이면 orders 계열 namespace 가 내부).
- **외부 후보**: 그 외 namespace 첫 segment 가 있는 reference. 언어별 구분자 (`::`, `.`, `/`) 는 하드코드하지 않고 import·참조 구문에서 관찰된 형태를 따른다.
- **ignore 패턴 필터**: 외부 후보에서 표준 라이브러리·프레임워크 기본 클래스를 제외한다.

  > **config lookup**: `workspace/_common/config.md` 의 `## discover 외부 도메인 ignore 패턴` 섹션을 먼저 조회한다. 섹션·행이 없으면 언어 공통 휴리스틱으로 제외 — 프레임워크 base class (예: `ApplicationRecord`, `@Entity` 계열), 언어 기본 타입 (String/Hash/Array/Integer 등), 테스트 더블.
  >
  > **runtime fallback**: namespace 추출 실패 시 (구분자 없음 등) → 해당 클래스 무시 + `[WARN] 외부 클래스 namespace 추출 실패: {class_name} — 수동 확인 권장` 1 줄. abort 하지 않는다.

- 필터 후 남은 외부 클래스 목록을 **메모리에 누적** (추정 도메인별 grouping). 단계 4 의 MANIFEST 갱신에서 사용.
- `--dry-run`·미리 보기 거부 시 다른 산출물과 동일하게 기록하지 않는다 (Abort cleanup 계약).

---

## MANIFEST `## 외부 도메인 reference` 표 갱신 (단계 4 확장)

detect 에서 누적된 외부 클래스 목록을 처리한다. 표 스키마는 doctor 가 강제하는 3컬럼 고정: `| 추정 도메인 | 클래스 (개수) | 추천 후속 학습 |`.

**idempotency — 현재 discover 도메인의 stale row 제거:**

- 현재 discover 의 `{domain}` 이 표에 행으로 존재하면 그 행을 제거한다 (이미 학습됨 — 더 이상 "미완료" 아님).

**외부 클래스 목록이 0 이면:** 섹션 자체를 추가하지 않는다 (빈 섹션 방지). 이미 존재하는 섹션은 그대로 유지.

**외부 클래스 목록이 1 이상이면:**

- 추정 도메인별 grouping: namespace 첫 segment 를 **CamelCase → snake_case 로 변환** (예: `ApiExceptions::Custom` → `api_exceptions`). 연속 대문자 (`XMLParser` → `xml_parser`) 도 동일 규칙. namespace 없는 클래스는 skip.
- 추정 도메인이 이미 `## 도메인 분류` 표에 등록된 도메인이면 제외 (이미 학습됨).
- 추천 경로 탐색: `{source_root}` 하위에서 도메인 슬라이스 키워드 폴더 Glob (예: `{source_root}/{models,services,controllers}/{추정 도메인}/`). 존재하면 그 경로, 없으면 `(경로 자동 추정 실패 — 사용자 직접 지정)`.
- 표가 이미 존재하면: 같은 추정 도메인 행은 클래스 목록·개수 갱신 (행 내용 교체), 사용자 수동 추가 행 (`(auto)` 마커 없는 행) 은 보존, 없으면 새 행 추가 (행 끝에 공백 + `(auto)` 마커).
- 섹션이 없으면 새 섹션 + 표 생성:

  ```markdown
  ## 외부 도메인 reference (discover 미완료)

  | 추정 도메인 | 클래스 (개수) | 추천 후속 학습 |
  | --- | --- | --- |
  | {추정 도메인} | {Class1}, {Class2}... ({N}) | `/dp-skills:discover {추천 경로}` 또는 `/dp-skills:discover --boundary {추정 도메인} --from {domain}` (auto) |
  ```

  - 추천 컬럼은 두 경로를 함께 제시한다 — 전체 학습과 경계만 학습 (`--boundary`). 경로 자동 추정 실패 시 전체 학습 안내는 `(경로 자동 추정 실패 — 사용자 직접 지정)` 으로 대체하되 boundary 안내는 유지.

> **runtime fallback**: 추정 도메인 추출 전체 실패 시 → 섹션 작성 skip + `[WARN] 외부 도메인 reference 추출 실패 — 수동 관리 권장` 1 줄. discover 본 작업은 정상 종료.

---

## Boundary 모드 — 호출 표면 추출 (`--boundary {B} --from {A}`)

외부 도메인 `{B}` 를 전체 학습하지 않고, **학습된 도메인 `{A}` 가 실제 호출하는 `{B}` 의 표면만** 포착해 `workspace/{TEAM}/context/boundaries/{A}--{B}.md` 를 생성한다. 비용은 O(접점 크기) — spec 작성에 필요한 정보 대부분은 접점에 있다.

### 0. 파일명 토큰

도메인 식별자가 계층형 (`backend/orders`) 이면 파일명 토큰에서 `/` 를 `__` 로 치환한다: `backend__orders--billing.md`. 구분자는 `--` 고정 — 도메인명 sanitize 는 영숫자·하이픈·언더스코어·`/`(계층) 만 허용·소문자화하며, **연속 하이픈은 단일 하이픈으로 축약**해 도메인명에 `--` 가 들어가지 않도록 보장한다 (축약 없이는 `a--b` 도메인명이 구분자와 모호). orchestrate-load (글롭 + 토큰 치환)·doctor (suffix 문자열 매칭 + 토큰 치환) 도 같은 토큰 규칙을 쓴다.

### 1. 호출처 수집

- `{A}` 의 소스 파일 목록 확보: MANIFEST `## 도메인 분류` 의 `{A}` 행이 가리키는 `scope/{A}.md` (분할 시 `scope/{A}/`) 가 인용하는 `file:line` 경로들 + 해당 파일들에 1홉 의존성 추적 (extraction.md §2홉 그래프의 references() 재사용, depth 1).
- 그 파일들에서 `{B}` namespace reference 를 Grep (예: `{B의 CamelCase}::` / `{B 패키지}.`). 위 ignore 패턴 동일 적용.
- **호출처 0 건** → "경계 없음: {A} 는 {B} 를 직접 호출하지 않음" 보고 후 종료. 파일을 생성하지 않는다.

### 2. 표면 추출

- 각 호출처 라인 ±10 줄 Read — 호출 메서드·인자 형태·반환값 사용 방식 수집 (extraction.md §파일별 Read 전략의 Targeted Read 와 동일).
- `{B}` 의 정의 파일 탐색: `{source_root}` 하위 도메인 슬라이스 폴더 Glob (위 추천 경로 휴리스틱과 동일). 발견 시 **호출된 심볼의 시그니처·관련 상태값만** Targeted Read. 전체 학습 금지. 미발견 시 호출부 사용 형태만으로 기록하고 `정의 (B)` 컬럼은 `(미확인)`.
- 트랜잭션 중첩 (해당 시): 호출처 파일에서 트랜잭션 블록 패턴 Grep (`\.transaction\s*do|@Transactional|BEGIN\b` 등 — 언어별 관찰 형태) → 매치 ±20 줄 Read 로 `{B}` receiver 의 read/write/create/destroy 호출만 캡처. 본 도메인 receiver 는 제외.
- **추측 금지** — 코드에 문자 그대로 있는 것만, 모든 행에 `file:line` 인용 (extraction.md §추측 금지와 동일 원칙).

### 3. 산출 형식 — `boundaries/{A}--{B}.md`

본문 ≤ 150 줄.

```markdown
# 경계 계약: {A} → {B}

> `/dp-skills:discover --boundary` 생성. {A} 가 실제 호출하는 {B} 표면만 기록 — {B} 전체 학습이 아니다.
> 전체 학습: `/dp-skills:discover {B 추천 경로}` (완료 시 본 문서보다 도메인 산출물 — scope/{B}.md — 이 우선)

## 호출 표면

| {B} 심볼 | 사용 형태 (인자 → 반환) | 호출처 ({A}) | 정의 ({B}) |
| --- | --- | --- | --- |

## 상태값·상수 (관찰된 것만)

## 트랜잭션 중첩 (해당 시)

| 본 도메인 entry | 외부 도메인 영향 | 변경 type | file:line |
| --- | --- | --- | --- |

## 미해결 (코드만으로 불명)

- (없음) 또는 전체 discover·사용자 확인이 필요한 항목
```

### 4. 색인·idempotency

- MANIFEST `## 외부 도메인 reference` 표에 `{B}` 행이 있으면 추천 컬럼 끝에 공백 + `· 경계: {A}--{B}.md` 표기를 추가한다. **행을 제거하지 않는다** — 행 제거는 전체 discover 완료의 신호다.
- `rules/{B}.md` 스켈레톤은 만들지 않는다 — boundary 는 `{B}` 의 학습이 아니다.
- 같은 `{A}--{B}` 재실행 시 파일 전체 재생성 (scope 재실행과 동일한 단순 덮어쓰기 — 충돌은 `git diff`).
- 로드는 orchestrate-load 의 boundaries 글롭이 담당 — MANIFEST `## 도메인 분류` 에 등록하지 않는다.

> **runtime fallback**: 표면 추출 중 심볼 정의 탐색 실패·namespace 판별 실패는 해당 항목을 `미해결` 섹션으로 내리고 진행. abort 하지 않는다.
````

- [ ] **Step 2: 커밋**

```bash
git add dp-skills/skills/discover/references/cross-domain.md
git commit -m "feat(discover): cross-domain detect·boundary 모드 reference 신설

외부 도메인 reference 추출(언어중립)·MANIFEST 표 갱신·boundary
호출 표면 추출 절차를 references/cross-domain.md 로 작성.
98d613d 에서 소비자(analyze·create-feature·doctor)만 역이식되고
생산자가 빠져 있던 갭을 메운다.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 5: discover — SKILL.md·manifest-update.md·output-format.md 배선

**Files:**
- Modify: `dp-skills/skills/discover/SKILL.md`
- Modify: `dp-skills/skills/discover/references/manifest-update.md`
- Modify: `dp-skills/skills/discover/references/output-format.md`

- [ ] **Step 1: SKILL.md frontmatter description 갱신**

기존 description 마지막 문장 뒤에 추가 (Edit):

```
old: 기존 scope-exploration.md 가 소비 규칙이라면 본 스킬이
  생산자 역할을 담당한다.
new: 기존 scope-exploration.md 가 소비 규칙이라면 본 스킬이
  생산자 역할을 담당한다. `--boundary B --from A` 는 외부 도메인 B 전체
  대신 A 가 호출하는 표면만 `boundaries/{A}--{B}.md` 로 포착한다
  (cross-domain 경량 학습).
```

- [ ] **Step 2: SKILL.md 인자 파싱 블록 갱신**

`### 1. 인자 파싱` 의 코드 블록과 옵션 목록을 갱신:

```
old:
\```
<entry> [domain] [--team <TEAM>] [--depth N] [--dry-run]
\```
new:
\```
<entry> [domain] [--team <TEAM>] [--depth N] [--dry-run]
--boundary <B> --from <A> [--team <TEAM>] [--dry-run]   # 경계 계약 모드 (entry 생략)
\```
```

옵션 설명 목록 끝 (`--dry-run` 항목 뒤) 에 추가:

```markdown
- `--boundary <B> --from <A>`: 경계 계약 모드 — `<A>` (학습된 도메인) 가 호출하는 `<B>` (미학습 외부 도메인) 표면만 추출. entry 는 생략한다 (지정 시 에러 + 사용법 안내). 상세: 아래 `## Boundary 모드` 섹션.
  - `--from` 생략 시 MANIFEST `## 도메인 분류` 의 도메인 목록을 제시하고 사용자에게 질의.
  - `<A>`·`<B>` 는 영숫자·하이픈·언더스코어·`/`(계층 식별자) 만 허용, 소문자화, 연속 하이픈은 단일로 축약 (`--` 구분자 모호성 방지). `..`·절대 경로 거부 (기존 entry 검증과 동일 원칙).
```

- [ ] **Step 3: SKILL.md 단계 2·단계 4 확장**

단계 2 (2홉 탐색) 의 수집 카테고리 줄 뒤에 추가:

```markdown
- 탐색 중 발견된 클래스/모듈 reference 를 내부 vs 외부 도메인으로 분류해 외부 후보를 메모리에 누적한다 (단계 4 의 MANIFEST `## 외부 도메인 reference` 표 기록용). 분류·ignore 패턴: [`references/cross-domain.md`](references/cross-domain.md) § 외부 도메인 reference detect.
```

단계 4 (디스크 기록) 의 3. MANIFEST 항목 뒤에 항목 추가 (기존 4. 재실행 항목은 5. 로):

```markdown
4. `MANIFEST.md` `## 외부 도메인 reference` 표 갱신 — detect 결과가 1 건 이상일 때만. 현재 도메인의 stale row 제거 (idempotency). 상세: [`references/cross-domain.md`](references/cross-domain.md) § MANIFEST 표 갱신.
```

- [ ] **Step 4: SKILL.md `## Boundary 모드` 섹션 신설**

`## 출력 구조 결정 규칙` 섹션 앞에 삽입:

```markdown
## Boundary 모드 — `--boundary {B} --from {A}`

외부 도메인 `{B}` 를 전체 학습하지 않고, **학습된 도메인 `{A}` 가 실제 호출하는 `{B}` 의 표면만** 포착해 `workspace/{TEAM}/context/boundaries/{A}--{B}.md` 를 생성한다. cross-domain feature 에서 `{B}` 전체 discover 의 경량 대안 — 비용은 O(접점 크기).

**전제 검증** (사전 확인 1~3·5 와 동일 + 추가):

- `{A}` 가 MANIFEST `## 도메인 분류` 에 등록돼 있어야 한다. 미등록이면 에러 + `/dp-skills:discover {진입점} {A}` 안내 후 종료.
- `{B}` 가 이미 `## 도메인 분류` 에 등록돼 있으면 안내 후 종료 — 이미 학습된 도메인은 scope 가 우선.

**절차** — 3 단계. 상세: [`references/cross-domain.md`](references/cross-domain.md) § Boundary 모드.

1. **호출처 수집** — `{A}` 소스에서 `{B}` namespace reference Grep. 0 건이면 "경계 없음" 보고 후 종료 — 파일 생성 안 함.
2. **표면 추출** — 호출처 ±10 줄 Read + `{B}` 정의 파일은 **호출된 심볼만** Targeted Read. 전체 학습 금지.
3. **미리 보기 → 생성 + 색인** — 단계 3 미리 보기·승인 게이트·Abort 계약·`--dry-run` 을 동일 적용. 승인 후 `boundaries/{A}--{B}.md` Write (본문 ≤ 150 줄) → MANIFEST 외부 reference 표의 `{B}` 행 추천 컬럼에 공백 + `· 경계: {A}--{B}.md` 표기 (행 제거 금지 — 전체 discover 완료가 아님).

**로드 배선** — orchestrate-load 가 활성 도메인 기준 `boundaries/{domain}--*.md` (정방향: 내가 호출하는 표면) 와 `*--{domain}.md` (역방향: 남이 나를 호출하는 표면 — 영향 분석) 를 자동 로드한다 (상한 6). 별도 MANIFEST 등록 불필요. 계층 식별자는 파일명 토큰에서 `/`→`__` 치환.
```

- [ ] **Step 5: manifest-update.md 에 외부 reference 표 규칙 추가**

`## 카테고리 표 갱신` 섹션 앞에 삽입:

```markdown
## `## 외부 도메인 reference` 표 갱신

detect 결과가 1 건 이상일 때만 갱신한다. 행 grouping·추천 경로·(auto) 마커·stale row 제거·boundary 병기 상세: [`cross-domain.md`](cross-domain.md) § MANIFEST 표 갱신. 표 스키마는 doctor 검증과 일치해야 한다 (3컬럼 고정):

\```markdown
## 외부 도메인 reference (discover 미완료)

| 추정 도메인 | 클래스 (개수) | 추천 후속 학습 |
| --- | --- | --- |
\```

Edit 절차·검증은 본 문서의 `## Edit 절차`·`## 검증` 과 동일 (파이프 수 일치, 다른 행 보존).
```

- [ ] **Step 6: output-format.md 에 미리 보기·결과 템플릿 추가**

`## 단계 3 — 미리 보기 표` 의 "기록할 파일" 목록에 한 줄 추가:

```
old:
  workspace/{TEAM}/context/MANIFEST.md         (도메인 행 추가·갱신)
new:
  workspace/{TEAM}/context/MANIFEST.md         (도메인 행 추가·갱신{, 외부 도메인 reference {N}건})
```

파일 끝에 boundary 모드 템플릿 추가:

````markdown
## Boundary 모드 — 미리 보기·결과 출력

미리 보기 (승인 게이트는 단계 3 과 동일):

```
경계 계약 미리 보기: {A} → {B}

[호출 표면]
| {B} 심볼 | 사용 형태 | 호출처 ({A}) | 정의 ({B}) |
| -------- | --------- | ------------ | ---------- |
| ...      | ...       | ...          | ...        |

기록할 파일:
  workspace/{TEAM}/context/boundaries/{A}--{B}.md   {created|updated}
  workspace/{TEAM}/context/MANIFEST.md              (외부 reference {B} 행에 경계 표기 병기)

계속할까요? (y/n)
```

결과 출력:

```
discover --boundary 완료: {A} → {B}

기록된 파일:
  workspace/{TEAM}/context/boundaries/{A}--{B}.md   {created|updated}  ({N}줄)
  workspace/{TEAM}/context/MANIFEST.md              {updated|unchanged}

호출 표면: {N}개 심볼 · 상태값 {N}개 · 트랜잭션 중첩 {N}건

▶ 다음 단계:
  planner 호출 시 자동 로드됨 (정방향 {A}--*.md)
  전체 학습이 필요해지면: /dp-skills:discover {B 추천 경로}
```
````

- [ ] **Step 7: 커밋**

```bash
git add dp-skills/skills/discover/
git commit -m "feat(discover): --boundary 모드 + 외부 도메인 reference 생산 배선

SKILL.md 인자·단계 2(detect)·단계 4(MANIFEST 표)·Boundary 모드
섹션, manifest-update·output-format 템플릿 추가. 미리 보기 승인
게이트·Abort 계약·--dry-run 은 boundary 모드에도 동일 적용.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 6: 소비자 메시지에 boundary 대안 병기 (analyze·create-feature)

**Files:**
- Modify: `dp-skills/skills/analyze/SKILL.md:160-161` (cross-domain detect INFO)
- Modify: `dp-skills/skills/create-feature/references/open-questions.md:10-12`

- [ ] **Step 1: analyze INFO 메시지 병기**

```
old:
  - 매칭되는 도메인 있으면 아래 형식으로 1 줄 출력:
    `[INFO] features/ 의 일부 영역이 {외부 도메인} 도메인에 의존 — /dp-skills:discover {추천 경로} 후 재분석 권장`
new:
  - 매칭되는 도메인 있으면 아래 형식으로 1 줄 출력:
    `[INFO] features/ 의 일부 영역이 {외부 도메인} 도메인에 의존 — /dp-skills:discover {추천 경로} (전체) 또는 /dp-skills:discover --boundary {외부 도메인} --from {domain} (경계만) 후 재분석 권장`
```

- [ ] **Step 2: create-feature open-questions 병기**

```
old:
     - Open Questions `### (b) cross-domain 산출물 부재` 에 `- [ ] {외부 도메인} 산출물 부재 → \`/dp-skills:discover {추천 경로}\` 권장` 행 추가.
     - INFO 1 줄: `[INFO] 이 feature 는 {외부 도메인} 의존성이 감지됨 — 먼저 \`/dp-skills:discover {추천 경로}\` 권장`
new:
     - Open Questions `### (b) cross-domain 산출물 부재` 에 `- [ ] {외부 도메인} 산출물 부재 → \`/dp-skills:discover {추천 경로}\` 또는 \`/dp-skills:discover --boundary {외부 도메인} --from {domain}\` (경계만) 권장` 행 추가.
     - INFO 1 줄: `[INFO] 이 feature 는 {외부 도메인} 의존성이 감지됨 — 먼저 \`/dp-skills:discover {추천 경로}\` (전체) 또는 \`--boundary\` (경계만) 권장`
```

- [ ] **Step 3: 커밋**

```bash
git add dp-skills/skills/analyze/SKILL.md dp-skills/skills/create-feature/references/open-questions.md
git commit -m "feat(skills): cross-domain 안내에 boundary 대안 병기

analyze·create-feature 의 미학습 외부 도메인 안내에 전체 discover
와 --boundary 경량 경로를 함께 제시.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 7: how-to 문서 + reference 재생성 + 버전 bump + 전체 검증

**Files:**
- Modify: `dp-skills/docs/how-to/domain-discover.md` (`## 흔한 실수` 앞에 섹션 추가)
- Modify: `dp-skills/.claude-plugin/plugin.json` (`0.16.19` → `0.17.0`)
- Regenerate: `dp-skills/docs/reference/skills/discover.md` (`docs_build.py` 자동)

- [ ] **Step 1: how-to 섹션 추가**

`domain-discover.md` 의 `### 2. (선택) 미리 보기만` 섹션 뒤, `## 흔한 실수` 앞에 추가:

```markdown
### 3. (선택) 경계만 학습 — `--boundary`

외부 도메인 전체를 학습하기엔 비용이 크고, 실제로 필요한 것은 **내 도메인이 호출하는 표면**뿐인 경우:

\```text
/dp-skills:discover --boundary schoice --from orders
\```

`orders` 가 실제 호출하는 `schoice` 의 클래스·메서드 시그니처, 관찰된 상태값, 트랜잭션 중첩만 추출해 `workspace/{TEAM}/context/boundaries/orders--schoice.md` 를 생성합니다. 비용은 접점 크기에 비례합니다.

생성된 경계 문서는 별도 등록 없이 자동으로 로드됩니다 — planner 호출 시 활성 도메인 기준 **정방향**(`{domain}--*.md`: 내가 호출하는 표면)과 **역방향**(`*--{domain}.md`: 다른 도메인이 나를 호출하는 표면 — 영향 분석용)을 모두 적재합니다 (상한 6건). 미학습 외부 의존이 MANIFEST 에 기록돼 있으면 planner 호출 시 boundary 처방이 힌트로 안내됩니다.

!!! tip "전체 discover vs boundary"
    경계 문서는 부분 커버입니다 — MANIFEST 의 `## 외부 도메인 reference` 행은 유지되며, 이후 해당 도메인을 전체 discover 하면 행이 제거되고 도메인 산출물 (`scope/{domain}.md`) 이 경계 문서보다 우선합니다.
```

- [ ] **Step 2: 버전 bump**

`dp-skills/.claude-plugin/plugin.json` 의 `"version": "0.16.19"` → `"version": "0.17.0"` (minor — 기능 추가, CLAUDE.md 컨벤션).

- [ ] **Step 3: docs reference 재생성 + drift 검증**

Run: `python3 dp-skills/tools/docs_build.py`
Expected: `docs/reference/skills/discover.md` 재생성 (SKILL.md 변경 반영). exit 0.

- [ ] **Step 4: 전체 테스트**

Run: `/opt/homebrew/bin/pytest dp-skills/tests/ -x -q`
Expected: 전체 PASS

- [ ] **Step 5: 커밋**

```bash
git add dp-skills/docs/ dp-skills/.claude-plugin/plugin.json
git commit -m "chore(release): bump 0.17.0 — cross-domain boundary 모드

how-to(domain-discover) boundary 섹션 + reference 재생성.
pilot 43d9daf 역이식 완료: discover 외부 reference 생산자,
--boundary 모드, orchestrate-load 자동 로드, doctor INFO.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

## 범위 제외 (YAGNI)

- **pilot Phase 4 상당 (Cross-domain Transaction Contracts sub-section 을 scope 본문에 삽입)** — boundary 문서 안의 `## 트랜잭션 중첩` 섹션으로만 캡처. scope 구조 변경은 별도 사이클.
- **team-init 의 boundaries/ 스켈레톤 생성** — 첫 boundary 실행 시 mkdir 로 충분. known-domains·관리외 검사는 `context/scope/`·프로젝트 폴더만 보므로 `context/boundaries/` 는 충돌 없음 (검증 완료).
- **`--from` 자동 추론 (pilot 은 `.agent-state.yml` 사용)** — discover 는 프로젝트 상태를 읽지 않는 팀 스킬. 생략 시 사용자 질의로 단순화.
- **MAX_BOUNDARY_DOCS 의 config 화** — 상수 6 으로 시작, 필요 시 후속.
- **orchestrate-load `main()` 의 계층 도메인 (`/` 포함) 허용 완화** — 현재 `has_path_traversal` 이 활성 도메인의 `/` 를 차단해 계층 도메인은 scope·boundary 자동 로드가 CLI 에서 작동하지 않는 기존 제약. 완화 시 scope/rules 경로 보간 (`scope/backend/orders.md`) 동작 검증이 함께 필요 — 별도 사이클. **→ 후속 사이클에서 해소됨**: `has_domain_traversal` 도입 (위 "고정 인터페이스 § 알려진 제약" 의 해소 표기 참조).
