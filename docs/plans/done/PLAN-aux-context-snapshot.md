# 보조 컨텍스트 로드 계약 + 스냅샷 안내 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** ① 사용자가 프로젝트 루트에 만든 보조 문서(schema.md·DIAGRAM.md·RESUME.md 등)를 orchestrate-load 가 hints 로 노출해 에이전트가 인지하게 하고, ② 프로젝트 폴더 수동 복사(`-bak`) 워크어라운드를 doctor 가 감지해 git 스냅샷 경로를 안내한다.

**Architecture:** ① `list_supplementary_docs(project_dir)` 헬퍼 — 프로젝트 루트의 `*.md` 중 `project.md`·dotfile 제외, 정렬 후 hints 1줄 (자동 Read 는 안 함 — 25KB+ 컨텍스트 폭증 방지, 에이전트가 관련성 판단). ② `check_backup_copy_projects(workspace, team)` — `projects/` 하위 디렉터리명이 백업 복사 패턴(`-bak`·`_bak`·`.bak`·`-backup` 접미)이면 WARN + git 권장 hint. `check_team` 에 배선. GUIDE.md 에 스냅샷 권고 절 추가.

**Tech Stack:** Python 3 표준 라이브러리, unittest (기존 컨벤션), markdown.

---

## 배경 (nimda 실데이터)

- Presend-Admin 루트의 `RESUME.md`(8.8KB)·`DIAGRAM.md`(6.7KB)·`schema.md`(9.9KB) 를 파이프라인이 전혀 인지 못함 — 사용자가 focus.md 에서 *"schema.md Step 3 에 명시 완료"* 처럼 수동으로 다리를 놓는 중.
- `Presend-Admin-bak/`(1.1MB 통째 복사) — 큰 전환 전 복원점이 없어 수동 복사. 정식 프로젝트처럼 `.agent-state.yml` 을 갖고 나란히 존재해 검사·자동화 노이즈.

### Task 1: orchestrate-load 보조 문서 hint (TDD)

**Files:**
- Modify: `dp-skills/tools/orchestrate-load.py` (헬퍼 + build_load_plan 의 project.md 섹션 직후 배선 + docstring hints 설명)
- Test: `dp-skills/tests/tools/test_orchestrate_load.py`

- [ ] **Step 1: 실패하는 테스트** — 유닛 (`list_supplementary_docs`): 보조 문서만 반환(정렬), project.md·dotfile 제외, 빈/부재 디렉터리는 빈 리스트. e2e (`MainProjectPhase` 패턴 재사용): scaffold 에 schema.md 추가 → hints 에 `[컨텍스트]` 라인 존재; 보조 문서 없으면 해당 hint 부재.
- [ ] **Step 2: 실패 확인** — `pytest tests/tools/test_orchestrate_load.py -q` → AttributeError.
- [ ] **Step 3: 구현**

```python
SUPPLEMENTARY_EXCLUDE = {"project.md"}


def list_supplementary_docs(project_dir: Path) -> list[str]:
    """프로젝트 루트의 보조 컨텍스트 문서(*.md) 목록 — 정렬된 파일명.

    project.md(관리 파일)·dotfile(.focus.md 등 별도 채널)은 제외.
    자동 Read 하지 않고 hints 로만 노출 — 대형 문서로 인한 컨텍스트
    폭증 방지, 관련성 판단은 에이전트 몫.
    """
    if not project_dir.is_dir():
        return []
    return sorted(
        p.name
        for p in project_dir.iterdir()
        if p.is_file()
        and p.suffix == ".md"
        and not p.name.startswith(".")
        and p.name not in SUPPLEMENTARY_EXCLUDE
    )
```

build_load_plan 의 project.md 처리(섹션 2) 직후 배선:

```python
    supplementary = list_supplementary_docs(workspace / team / "projects" / project)
    if supplementary:
        shown = supplementary[:8]
        extra = f" 외 {len(supplementary) - 8}건" if len(supplementary) > 8 else ""
        hints.append(
            f"[컨텍스트] 프로젝트 보조 문서: {', '.join(shown)}{extra} — "
            "작업과 관련되면 Read 권장 (자동 로드 안 함)"
        )
```

- [ ] **Step 4: 통과 + 전체 회귀 + Commit** (`feat(orchestrate): 프로젝트 보조 문서 hints 노출`)

### Task 2: doctor 백업 복사본 감지 (TDD)

**Files:**
- Modify: `dp-skills/tools/doctor/integrity.py` (신규 check + `check_team` 말미 배선)
- Test: `dp-skills/tests/tools/test_doctor_backup_projects.py` (신규)

- [ ] **Step 1: 실패하는 테스트** — `P-bak`·`P_bak`·`P.bak`·`P-backup` 디렉터리 → WARN(각 1건, hint 에 "git"), 정상명·dotdir·파일은 무시, projects/ 부재 시 빈 리스트.
- [ ] **Step 2: 실패 확인** → AttributeError.
- [ ] **Step 3: 구현**

```python
_BACKUP_DIR_RE = re.compile(r"([-_.]bak|-backup)$", re.IGNORECASE)


def check_backup_copy_projects(workspace: Path, team: str) -> list[Result]:
    """projects/ 하위 수동 백업 복사본 의심 디렉터리 감지.

    프로젝트 폴더 통째 복사(-bak 등)는 복원점 부재의 워크어라운드 —
    검사·자동화 노이즈를 만들고 state 가 이중으로 잡힌다. 복원점은
    git 커밋/태그가 정식 경로 (nimda Presend-Admin-bak 1.1MB 사례)."""
    projects_dir = workspace / team / "projects"
    if not projects_dir.is_dir():
        return []
    results: list[Result] = []
    for p in sorted(projects_dir.iterdir()):
        if not p.is_dir() or p.name.startswith("."):
            continue
        if _BACKUP_DIR_RE.search(p.name):
            results.append(
                Result(
                    Result.WARN,
                    f"projects/{p.name}",
                    "수동 백업 복사본 의심 — 프로젝트 폴더 복사는 검사·자동화 노이즈",
                    "복원점은 git 커밋/태그 권장. 확인 후 복사본 삭제 또는 projects/ 밖으로 이동",
                )
            )
    return results
```

`check_team` 반환 직전에 `results.extend(check_backup_copy_projects(workspace, team))` 배선 (check_team 의 results 변수·반환 구조는 구현 시 확인).

- [ ] **Step 4: 통과 + 전체 회귀 + Commit** (`feat(doctor): 프로젝트 백업 복사본 감지`)

### Task 3: GUIDE 스냅샷 권고 + bump 0.16.11 + PR

- [ ] **Step 1**: `skills/context/lifecycle/projects/GUIDE.md` 의 "drift 감지" 섹션 뒤에 절 추가:

```markdown
### 스냅샷 — 프로젝트 폴더 복사 금지

재분석(`--regen-agents`)·qa 진입 같은 큰 전환 전 복원점이 필요하면 **git 커밋/태그**를 사용한다. `{PROJECT}-bak` 식 폴더 복사는 `.agent-state.yml` 이 이중으로 잡혀 검사·자동화 노이즈를 만든다 — doctor 가 WARN 으로 감지한다. (agents/ 한정 복원은 regen 이 자동 생성하는 `.agents.bak/` 이 이미 담당.)
```

- [ ] **Step 2**: docs_build + 전체 pytest + plugin.json `0.16.10` → `0.16.11` + commit + push + PR (base main).

## 비범위

- 보조 문서의 자동 files_to_read 편입 (컨텍스트 비용 — hints 우선, 사용 데이터 본 뒤 판단)
- 스냅샷 생성/복원 스킬 (git 으로 충분한지 운영 후 판단)
