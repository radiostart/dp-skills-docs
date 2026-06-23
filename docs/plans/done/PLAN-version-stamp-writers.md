# plugin_version Writer 정합 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** analyze·regen 의 state writer 지침에 `plugin_version` 갱신을 추가해, doctor/orchestrate-load 의 "업그레이드 → `--regen-agents` 권장" WARN 이 실제로 해소 가능한 사이클을 만든다.

**Architecture:** 감지 측은 이미 완비 — doctor(`integrity.py:920+`)와 orchestrate-load(`compare_plugin_version`) 모두 minor+ gap 에서 regen 권고 WARN 을 낸다. 문제는 writer 측: state-schema 의 writer 표는 analyze/regen 이 `plugin_version` 을 갱신한다고 선언하지만, 실제 LLM 이 따르는 [agents-update.md] step 3 과 [regen-mode.md] step 4 에 해당 필드가 없다 → 스탬프 고착(nimda 0.9.5) → WARN 영구 잔존. 문서 수정만으로 해결되는 doc-only 변경.

**Tech Stack:** markdown (LLM writer 지침). Python 변경 없음.

---

### Task 1: agents-update.md — plugin_version 갱신 + stale schema 문구 수정

**Files:**
- Modify: `dp-skills/skills/analyze/references/agents-update.md:169-176`

- [ ] **Step 1**: step 2 의 지원 schema 문구 수정 — `(v1, v1.1)` → `(v1.1, v1.2, v1.3)` (orchestrate-load `SUPPORTED_SCHEMAS` 와 동기).
- [ ] **Step 2**: step 3 필드 목록에 추가:

```markdown
   - `plugin_version: "{현재 플러그인 버전}"` — Bash `python3 -c "import json,os; print(json.load(open(os.environ['CLAUDE_PLUGIN_ROOT']+'/.claude-plugin/plugin.json'))['version'])"` 로 획득. 획득 실패 시 해당 행을 건드리지 않는다. **이 갱신이 누락되면 doctor 의 "업그레이드됨 → --regen-agents 권장" WARN 이 regen 후에도 해소되지 않는다** (state-schema writer 표와 정합).
```

- [ ] **Step 3**: step 4 의 "`schema` 가 `v1` 이면 이번 기회에 `v1.1` 로 업그레이드 ..." 절 제거 — step 2 가 v1 을 이미 에러 처리하므로 사문. schema 업그레이드는 migrate-state.py 전담 문구로 교체. 보존 필드 목록에 `phase` 추가.

### Task 2: regen-mode.md — 동일 갱신

**Files:**
- Modify: `dp-skills/skills/analyze/references/regen-mode.md` (동작 목록 4번)

- [ ] **Step 1**: `4. 6-4 — .agent-state.yml 의 analyzed_at, last_analyzed_features 갱신` → `plugin_version` 추가 + 미갱신 시 WARN 잔존 경고 1줄.

### Task 3: sweep + 검증 + bump 0.16.10 + PR

- [ ] **Step 1**: stale 문구 sweep — `grep -rn "v1\`, \`v1.1\|v1.1\` 로 업그레이드" dp-skills/skills/` 로 잔여 확인·수정.
- [ ] **Step 2**: `python3 dp-skills/tools/docs_build.py` + 전체 pytest.
- [ ] **Step 3**: plugin.json `0.16.9` → `0.16.10`, commit, push, PR (base main).

## 비범위

- doctor·orchestrate-load 감지 로직 (이미 regen 권고 포함 — 변경 불필요)
- plugin_version 을 phase 전환 등 다른 writer 이벤트로 확대 (state-schema 계약 변경 — 필요성 입증 후 별도)
