# 순차 사이클 오토파일럿 run-cycle — 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> 설계 SSOT: [PLAN-run-cycle.md](PLAN-run-cycle.md). 본 문서는 그 설계를 단계별 작업으로 분해한 것.

**Goal:** 활성 project 의 미완료 feature 를 순서대로 자동 순회하며 planner→critic→generator→evaluator 사이클을 진행하는 순차 오토파일럿 스킬 `/dp-skills:run-cycle` 을 추가한다.

**Architecture:** 메인 루프 오케스트레이터 스킬 1개. project.md 의 `[ ]` 미완료 feature 에서 큐를 파생(stateless)하고, 각 feature 에 `@ag-*` 사이클을 순서대로 호출한다. green(critic 합의·evaluator READY)엔 자동 진행, plan 승인·critic blocking·NOT_READY·block 에선 정지. 상태 파일·스키마 변경 없음.

**Tech Stack:** Claude Code 플러그인 (markdown skill). 검증 = `tools/doctor.py`·`tools/docs_build.py`·`markdownlint-cli`·`tests/tools/`. 코드 실행 없음.

> **공통 검증 명령** (pre-push hook 와 동일, `dp-skills/` 에서):
> ```bash
> python3 tools/doctor.py --schema
> python3 tools/docs_build.py --check
> npx -y markdownlint-cli "skills/**/*.md" "agents/**/*.md" "README.md"
> ```

---

### Task 1: `run-cycle` 스킬 신규 작성

**Files:**
- Create: `dp-skills/skills/run-cycle/SKILL.md`

- [ ] **Step 1: SKILL.md 전체 작성**

`dp-skills/skills/run-cycle/SKILL.md` 를 아래 내용으로 생성 (BEGIN/END 마커 제외):

--- BEGIN FILE CONTENT ---
````markdown
---
name: run-cycle
description: >-
  활성 project 의 미완료 feature 들을 순서대로 자동 순회하며
  planner→critic→generator→evaluator 사이클을 진행하는 순차 오토파일럿.
  green 신호(critic 합의·evaluator READY)에는 자동 진행하고, plan 승인·
  critic blocking·NOT_READY·block 에서만 일시정지한다. 큐는 project.md 의
  [ ] 미완료 feature 에서 파생하며 별도 상태 파일을 쓰지 않는다 (stateless,
  재실행=재개). 병렬 아님 — 한 번에 1 feature. 수동 @mention 사이클은 불변.
---

# /dp-skills:run-cycle

활성 project 의 development phase 에서 미완료 feature 들을 순차 자동 처리하는
오케스트레이터. 메인 루프에서 `@ag-planner`·`@ag-planner-critic`·
`@ag-generator`·`@ag-evaluator` 를 순서대로 호출한다 (subagent 가 subagent 를
띄우지 않음). 기존 수동 사이클은 그대로 — 본 스킬은 선택적 순차 러너.

대상: $ARGUMENTS  (옵션: feature 부분집합 `#3-5` · `#3 #4`)

## 사전 확인

[preamble.md](../context/shared/preamble.md) **P-1**(TodoWrite 선로딩) · **P1**(팀) ·
**P2**(활성 project) 수행. 활성 project 없으면 P2 메시지로 종료. `.agent-state.yml`
의 `phase` 가 `qa` 면 "run-cycle 은 development phase 전용" 안내 후 종료.

## 1. 큐 구성 (project.md 파생)

1. 활성 project 의 `project.md` `## 목표` 에서 `[ ]` 미완료 feature 를 **목록
   순서대로** 수집 (`[x]` 제외). 의존성 메타데이터가 없으므로 순서를
   재계산하지 않고 목록 순서를 신뢰한다.
2. 인자로 `#3-5`·`#3 #4` 명시 시 그 부분집합으로 필터.
3. 큐가 비면 → `"신규 작업 없음 (모든 목표 완료)"` 출력 후 종료.
4. 큐 표시 + 확인: `"큐: #3 {slug} → #4 {slug} → #5 {slug} (순서대로). 진행할까요?"`
   — 사용자 승인 후 시작.

## 2. feature 1건 사이클 루프

큐의 각 feature 를 순서대로 처리한다.

### 2.1 재개 지점 판정 (stateless)

feature 산출물로 "가장 이른 미완료 단계" 를 정한다:

- `features/NN-{slug}.plan.md` 없음 → **planner 부터**
- 유효한 `.plan.md` 있음 (`tools/plan-validate.py` 통과) → **critic/generator 부터**
- `features/NN-{slug}.eval.md` 가 `NOT_READY` → **그 지점부터**

완료(`[x]`) feature 는 큐에 없으므로 도달하지 않는다.

### 2.2 planner

`@ag-planner` 호출 → `.plan.md` 산출.
**⏸ plan 승인 게이트**: 산출된 plan 을 사용자에게 제시하고 승인을 기다린다.
승인 전 다음 단계 진행 금지 (feature 마다 1회). 미승인·거부 → §3 정지.

### 2.3 critic

`@ag-planner-critic` 호출 (기본 실행) → `.plan.critic.md`.

- 미해결 blocking 챌린지 有 → **⏸ §3 정지**.
- nit·suggestion 만 → 합의 표 `[x]` 처리 후 generator 진행.

### 2.4 generator

`@ag-generator` 호출 → 구현.

### 2.5 evaluator

`@ag-evaluator` 호출 → VERIFICATION REPORT.

- `READY` → `project.md` 해당 feature 체크박스 `[x]` 갱신 → **자동 다음 feature**(2.1 로).
- `NOT_READY`·`block` → **⏸ §3 정지**.

## 3. 정지 정책

문제 게이트(plan 미승인·critic blocking·NOT_READY·block) 도달 시:

- **큐 전체 정지.** 뒤 feature 로 진행하지 않는다 — N+1 이 N 에 의존할 수
  있고 의존성 데이터가 없어 알 길이 없다.
- 정지 사유·해당 REPORT 출력.
- 안내: `/dp-skills:fix-review` 로 처리 → 수정 후 **`/dp-skills:run-cycle`
  재실행(=재개)**. 부분집합으로 돌렸다면 같은 인자를 다시 넘긴다.

## 4. 큐 완료

모든 feature READY → 완료 요약 출력 (처리한 feature 목록·각 verdict).

## Do-NOT

- feature 병렬 진행 금지 — 한 번에 1 feature (`.focus.md`·`STATE.md` 싱글톤·
  generator 파일 충돌 회피).
- plan 승인·NOT_READY·critic blocking 자동 통과 금지 — 반드시 정지.
- 정지 후 뒤 feature 자동 진행 금지.
- 상태 파일 생성 금지 — 큐·진행은 `project.md` + 산출물에서 파생.
- 수동 사이클·`@ag-*` 동작 변경 금지 — 본 스킬은 호출 순서만 자동화.
````
--- END FILE CONTENT ---

- [ ] **Step 2: 검증 (from `dp-skills/`)**
```bash
cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills
python3 tools/docs_build.py          # generates docs/reference/skills/run-cycle.md
python3 tools/doctor.py --schema
npx -y markdownlint-cli "skills/run-cycle/SKILL.md"
```
Expected: docs_build writes run-cycle reference, doctor `--schema` PASS (frontmatter valid), markdownlint clean.

- [ ] **Step 3: frontmatter name 일치 확인**
```bash
grep -n "name: run-cycle" dp-skills/skills/run-cycle/SKILL.md
```
Expected: exactly 1 match. Directory name `run-cycle` matches frontmatter `name`.

- [ ] **Step 4: Commit**
```bash
cd /Users/jay-p/Projects/deali-skills-plugin
git add dp-skills/skills/run-cycle/SKILL.md dp-skills/docs/reference/skills/run-cycle.md
git commit -m "feat(skills): run-cycle 순차 사이클 오토파일럿 추가

project.md 미완료 feature 를 순서대로 자동 순회 — planner→critic→
generator→evaluator. Balanced 게이트, stateless 재개.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 2: README · plugin.json 동기화

**Files:**
- Modify: `dp-skills/README.md` (스킬 개수 헤딩 + 표 행)
- Modify: `dp-skills/.claude-plugin/plugin.json` (version)

- [ ] **Step 1: 스킬 개수 19→20**

`dp-skills/README.md` 의 `## 스킬 커맨드 (19종)` 를 `## 스킬 커맨드 (20종)` 으로 변경.

- [ ] **Step 2: 스킬 표에 행 추가**

`dp-skills/README.md` 스킬 표에서 `project` 행 (`| \`project\` | 프로젝트 활성화 ...`) 바로 위에 추가 (project lifecycle 관련 스킬끼리 인접 배치):
```
| `run-cycle` | 미완료 feature 순차 사이클 오토파일럿 | — |
```

- [ ] **Step 3: 버전 bump**

`dp-skills/.claude-plugin/plugin.json` 의 `"version": "0.14.0"` 를 `"version": "0.15.0"` 으로 변경.

(참고: main 기준 현재 0.13.0 이나, dp-code-review PR(0.14.0)이 머지되면 0.15.0 이 맞다. 머지 순서에 따라 충돌 시 최신 + 1 minor 로 조정.)

- [ ] **Step 4: 검증**
```bash
cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills
npx -y markdownlint-cli "README.md"
python3 -c "import json; json.load(open('.claude-plugin/plugin.json'))"
python3 tools/docs_build.py --check
```
Expected: markdownlint clean, JSON valid, docs_build `--check` PASS.

- [ ] **Step 5: Commit**
```bash
cd /Users/jay-p/Projects/deali-skills-plugin
git add dp-skills/README.md dp-skills/.claude-plugin/plugin.json
git commit -m "chore(release): bump 0.15.0

run-cycle 스킬 추가 반영 (스킬 20종).

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 3: 전체 검증 (pre-push 동등)

- [ ] **Step 1: reference 재생성 + clean 확인**
```bash
cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills
python3 tools/docs_build.py
cd .. && git status --short
```
Expected: working tree clean (reference 가 이미 Task 1 에서 커밋됨). 변경 남으면 add/commit.

- [ ] **Step 2: pre-push 검증 스위트 전체**
```bash
cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills
python3 tools/doctor.py --schema
for t in tests/tools/test_*.py; do [ -f "$t" ] && env -u GIT_DIR -u GIT_WORK_TREE -u GIT_INDEX_FILE -u GIT_PREFIX python3 "$t"; done
npx -y markdownlint-cli "skills/**/*.md" "agents/**/*.md" "README.md"
python3 tools/docs_build.py --check
```
Expected: doctor 0 ERROR (version>tag WARN 정상), tools 테스트 전부 통과, markdownlint clean, docs_build `--check` PASS.

- [ ] **Step 3: 큐 의미론 정합성 (수동 read-through)**

`skills/run-cycle/SKILL.md` 를 통독해 확인:
- 빈 큐 → "신규 작업 없음" 종료 경로가 §1.3 에 있다.
- 정지 후 재실행=재개가 §2.1 + §3 로 일관된다 (상태 파일 미생성).
- Do-NOT 가 병렬 금지·자동 통과 금지·상태 파일 금지를 명시한다.

---

## Self-Review 체크

- **Spec coverage:** §2 큐→Task1 §1 · §3 사이클/게이트→Task1 §2 · §4 정지→Task1 §3 · §5 stateless→Task1 §2.1/§3 · §6 TDD(evaluator 게이트 흡수, 별도 코드 없음 — SKILL Do-NOT/§2.5 가 신뢰) · §7 Slack(초판 SKILL 본문엔 미명시 — 기존 hook 자동 발화에 의존, 별도 작업 불필요) · §8 변경파일→Task1·2·3. 매핑 확인.
- **Placeholder scan:** TBD/TODO 없음. SKILL 전체 내용·경로·명령 제공.
- **Type consistency:** 게이트 이름(plan 승인·critic blocking·NOT_READY·block)·feature 식별(`NN-{slug}`)·산출물(`.plan.md`/`.eval.md`/`.plan.critic.md`)이 spec·SKILL 간 일치.
- **Note:** §7 Slack 은 기존 `slack-notify.py` hook 이 approval/complete 이벤트를 자동 발화하므로 run-cycle SKILL 에 별도 호출 코드를 넣지 않는다. 정지/완료 시점이 그 이벤트와 자연 정렬되는지는 운영 후 검토 (미해결 질문 아님 — 의존 동작).
