# protect-managed.sh Phase Fail-Closed Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** protect-managed.sh 의 qa-features 보호가 phase 값 비정상 시 조용히 풀리는 fail-open 을 닫는다 (0.16.9 에서 orchestrate-load 측만 처리된 잔여).

**Architecture:** `check_path` 의 features/*.md 분기에서 ① state 파일이 존재하는데 읽기 불가 → 차단, ② phase 값이 qa/development 외 → 차단 (fail-closed). phase 부재·state 부재는 종전대로 development 로 통과 (state-schema 기본값). `DP_PROTECT_BYPASS=1` 우회 유지. 테스트는 Python subprocess 로 hook 을 stdin JSON + 임시 워크스페이스로 구동.

**Tech Stack:** bash (hook), Python unittest subprocess 테스트.

---

### Task 1: hook 시나리오 테스트 (TDD) + fail-closed 구현

**Files:**
- Modify: `dp-skills/hooks/protect-managed.sh` (features/*.md 분기, 헤더 주석)
- Test: `dp-skills/tests/tools/test_protect_managed_hook.py` (신규)

- [ ] **Step 1: 실패하는 테스트** — 시나리오: qa→차단(exit 2), development→통과(0), phase 부재→통과, **invalid(qaa)→차단(신규)**, **state 존재+읽기불가→차단(신규)**, invalid 라도 qa/ 폴더는 통과, BYPASS=1→통과. hook 구동: `bash hooks/protect-managed.sh` + stdin `{"tool_name":"Write","tool_input":{"file_path":...}}` + env `CLAUDE_PROJECT_DIR={tmp}`.
- [ ] **Step 2: 실패 확인** — invalid·읽기불가 2개 시나리오만 FAIL (현재 통과돼버림).
- [ ] **Step 3: 구현** — features/*.md 분기에 elif 추가:

```bash
    if [[ "$phase" == "qa" ]]; then
      (기존 차단)
    elif [[ -n "$phase" && "$phase" != "development" ]]; then
      (차단 — phase 값 비정상: fail-closed. state 수정 또는 migrate-state.py 안내)
    fi
```

읽기불가는 phase 읽기 전에 `[[ -f "$state_file" && ! -r "$state_file" ]]` 로 차단. 헤더 주석에 fail-closed 규칙 1줄 추가.

- [ ] **Step 4: 통과 + 전체 회귀 + Commit**

### Task 2: bump 0.16.12 + PR (base main, squash)

## 비범위
- Bash destructive target 추출 로직 (별개 축)
- scope-guard.sh 등 다른 hook
