# qa 스킬 오발동 억제 + fail-fast 재배열 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) 구문으로 추적한다.

**Goal:** 사용자가 Jira 티켓 링크만 남겨도 qa 스킬이 자동 발동해 토큰을 소모하는 문제를 3중으로 차단한다 — ① description 트리거 억제, ② fail-fast 게이트를 사전 확인(P-1·P0)보다 앞으로 재배열, ③ STATE.md 파일 부재 분기 명시.

**Architecture:** SKILL.md 는 선언적 명세 문서이므로 코드 테스트 대신 `docs_build.py --check` (reference 동기화) 와 기존 pytest 회귀로 검증한다. `disable-model-invocation` frontmatter 는 플러그인 skill 에서 무시되는 버그([claude-code#22345](https://github.com/anthropics/claude-code/issues/22345), 2026-06 시점 open)가 있어 **description 문구가 유일한 트리거 억제 레버**다. 수행 순서 변경은 qa/SKILL.md 가 SSOT 이고, preamble.md P-1 정의("첫 tool call")와의 충돌은 P-1 에 예외 문구를 추가해 해소한다.

**Tech Stack:** Markdown (SKILL.md 명세), Python (docs_build.py 재생성·검증), pytest (회귀).

**배경 (재현된 문제):** `workspace/dp-team/STATE.md` 미생성 상태에서 사용자가 티켓 링크만 붙여넣었는데 qa 스킬이 자동 발동 → SKILL.md 본문(~250줄) 로드 + P-1(TodoWrite 선로딩·todo 보드) + P0(메모 Read) + P1(TEAM.md Read) 수행 후에야 2단계에서 종료. STATE.md 부재 케이스는 명세에 없어 임기응변 추가 탐색 여지도 있었음.

---

## File Structure

| 파일 | 작업 | 책임 |
| --- | --- | --- |
| `dp-skills/skills/qa/SKILL.md` | Modify | ① frontmatter description 트리거 억제 ② 사전 확인 섹션 fail-fast 순서 재정의 ③ 2단계 STATE.md 부재 분기 |
| `dp-skills/skills/context/shared/preamble.md` | Modify | P-1 "첫 tool call" 정의에 fail-fast 게이트 예외 추가 |
| `dp-skills/docs/how-to/qa-cycle.md` | Modify | "언제 쓰나" 에 명시 호출 전용 정책 1줄 |
| `dp-skills/docs/reference/**` | Regenerate | `docs_build.py` 가 SKILL.md 에서 자동 추출 — 수동 편집 금지 |

**범위 외 (명시):**

- `plugin.json` 버전 bump — fix 성격이라 릴리스 시점에 별도 (`chore(release):`) 처리.
- `disable-model-invocation: true` 필드 추가 — #22345 해결 전까지 무의미. 해결 확인 시 별도 1줄 PR.
- 다른 다단계 스킬(project·issue·analyze)의 fail-fast 재배열 — 오발동 빈도가 확인된 qa 한정 (YAGNI). 필요해지면 preamble P-1 예외 목록에 추가만 하면 됨.
- messages.md 키 통합 — qa 의 인라인 메시지("활성 project 가 없습니다…")는 기존 문구 유지 (별도 정리 대상).

## 사전 준비

- [ ] **Step 0-1: 작업 브랜치 생성** (main 기준)

```bash
git checkout main && git pull && git checkout -b fix/qa-misfire-failfast
```

---

### Task 1: description 트리거 억제 (오발동 차단)

**Files:**
- Modify: `dp-skills/skills/qa/SKILL.md:1-11` (frontmatter)
- Regenerate: `dp-skills/docs/reference/skills/qa.md`

모델은 발동 결정 시점에 frontmatter description 만 본다. negative trigger 문구를 **맨 앞에** 둬야 "Jira 링크 → qa 스킬" 패턴매칭을 끊을 수 있다.

- [ ] **Step 1-1: frontmatter description 교체** (Edit)

old_string:

```yaml
description: >-
  Jira QA 결함 처리 모드 진입 — 활성 project 의 qa/ 에 1건 처리. 인자로 Jira
  이슈 키 (`^[A-Z]+-\d+$`) 가 필수다. 활성 project 컨텍스트 (features·scope·
  rules·agents) 를 그대로 이어받아 결함 1건의 분석·수정 사이클을 시작한다.
```

new_string:

```yaml
description: >-
  사용자가 `/dp-skills:qa {KEY}` 를 명시적으로 호출했을 때만 사용한다 — 대화에
  Jira 티켓 링크·이슈 키가 보이는 것만으로는 발동하지 않는다 (링크 공유·메모는
  처리 요청이 아니다). Jira QA 결함 처리 모드 진입 — 활성 project 의 qa/ 에
  1건 처리. 인자로 Jira 이슈 키 (`^[A-Z]+-\d+$`) 가 필수다. 활성 project
  컨텍스트 (features·scope·rules·agents) 를 그대로 이어받아 결함 1건의
  분석·수정 사이클을 시작한다.
```

(이후 줄 — "development phase 에서 호출하면…" 부터 — 는 변경 없음.)

- [ ] **Step 1-2: 반영 확인**

Run: `head -16 dp-skills/skills/qa/SKILL.md`
Expected: description 첫 문장이 "사용자가 `/dp-skills:qa {KEY}` 를 명시적으로 호출했을 때만" 으로 시작.

- [ ] **Step 1-3: reference 재생성**

Run: `python3 dp-skills/tools/docs_build.py && python3 dp-skills/tools/docs_build.py --check`
Expected: `--check` exit 0. `git status` 에 `dp-skills/docs/reference/` 하위 재생성 파일 표시.

- [ ] **Step 1-4: 커밋**

```bash
git add dp-skills/skills/qa/SKILL.md dp-skills/docs/reference/
git commit -m "fix(qa): 스킬 오발동 억제 — description 명시 호출 한정

티켓 링크·이슈 키 노출만으로 모델이 qa 스킬을 자동 발동해
토큰을 소모하는 문제. disable-model-invocation 은 플러그인
skill 에서 무시되므로 (claude-code#22345) description negative
trigger 가 현재 유일한 억제 수단."
```

---

### Task 2: fail-fast 게이트 재배열 + STATE.md 부재 분기

**Files:**
- Modify: `dp-skills/skills/qa/SKILL.md:28-35` (사전 확인), `:80-91` (2단계)
- Modify: `dp-skills/skills/context/shared/preamble.md` (P-1 예외)
- Regenerate: `dp-skills/docs/reference/**`

오발동하더라도 싸게 죽도록: 인자 검증(Read 0회) → P1(Read 1회) → STATE.md 게이트(Read 1회) 를 먼저, 비용이 드는 P-1(TodoWrite 선로딩·todo 보드)·P0(메모 본문 Read) 는 게이트 통과 후에만.

- [ ] **Step 2-1: qa/SKILL.md "## 사전 확인" 섹션 교체** (Edit)

old_string:

```markdown
## 사전 확인

[preamble.md](../context/shared/preamble.md) 의 **P-1, P0, P1** 수행.

- P-1: TodoWrite 선로딩 (다단계 스킬).
- P0: 컨텍스트의 `MEMORY.md` 인덱스에서 인자의 Jira 키·주변 키워드와 관련된
  메모를 가려 본문을 Read 하여 과거 동일·유사 결함 처리 이력을 컨텍스트에 반영.
- P1: 팀 이름 `{TEAM}` 획득.
```

new_string:

```markdown
## 사전 확인 (fail-fast 순서)

[preamble.md](../context/shared/preamble.md) 의 **P-1, P0, P1** 을 수행하되,
아래 순서를 따른다 — **진입 게이트를 통과한 뒤에만 비용이 드는 P-1·P0 을
수행한다**:

1. 수행 절차 **1 (인자 검증)** — Read 없이 즉시 판정. `off` 면 1-A 로 분기.
2. **P1**: 팀 이름 `{TEAM}` 획득 (`workspace/TEAM.md` 1회 Read).
3. 수행 절차 **2 (활성 project 확인)** — STATE.md 1회 Read 게이트.
4. 게이트 통과 후에만:
   - **P-1**: TodoWrite 선로딩 (다단계 스킬).
   - **P0**: 컨텍스트의 `MEMORY.md` 인덱스에서 인자의 Jira 키·주변 키워드와
     관련된 메모를 가려 본문을 Read 하여 과거 동일·유사 결함 처리 이력을
     컨텍스트에 반영.

게이트에서 종료가 확정된 호출(인자 없음·형식 오류·활성 project 없음)에는
P-1·P0 을 수행하지 않는다 — 곧 종료할 호출에 todo 보드·메모 Read 비용을
쓰지 않는다.
```

- [ ] **Step 2-2: qa/SKILL.md "### 2. 활성 project 확인" 교체** (Edit)

old_string:

````markdown
### 2. 활성 project 확인

`workspace/{TEAM}/STATE.md` 를 Read 하여 본문 표에 `| project | X | ... |` 행이
있는지 확인한다. 없으면 다음 메시지 출력 후 종료:

```
활성 project 가 없습니다. /dp-skills:project {PROJECT} 먼저 호출하세요.
```
````

new_string:

````markdown
### 2. 활성 project 확인

`workspace/{TEAM}/STATE.md` 를 Read 한다.

- **파일이 없으면** (Read 실패) 아래 메시지를 출력하고 **즉시 종료**한다 —
  Glob·find·ls 등 추가 탐색을 하지 않는다 (미셋업은 정상 상태이지 진단 대상이
  아니다).
- 파일이 있으면 본문 표에 `| project | X | ... |` 행이 있는지 확인한다. 없으면
  같은 메시지를 출력하고 종료한다.

```
활성 project 가 없습니다. /dp-skills:project {PROJECT} 먼저 호출하세요.
```
````

(이어지는 "행이 있으면 두 번째 컬럼 값을 `{PROJECT}` 로 사용한다." 와 P2 미사용 각주는 변경 없음.)

- [ ] **Step 2-3: preamble.md P-1 에 예외 문구 추가** (Edit)

old_string:

```markdown
3 단계 이상 수행하는 스킬(`project`, `issue`, `qa`, `analyze`, `feature`, `ai-review`)은 **첫 tool call 로 `ToolSearch select:TodoWrite` 를 수행**한다. 직후 단계 목록을 세워 사용자에게 게시하고 진행하며 갱신한다.
```

new_string:

```markdown
3 단계 이상 수행하는 스킬(`project`, `issue`, `qa`, `analyze`, `feature`, `ai-review`)은 **첫 tool call 로 `ToolSearch select:TodoWrite` 를 수행**한다. 직후 단계 목록을 세워 사용자에게 게시하고 진행하며 갱신한다.

**예외 — fail-fast 게이트 스킬:** 자체 진입 게이트(인자 검증·활성 project 확인)를 가진 스킬은 **게이트 통과 직후**를 P-1 시점으로 본다 — 게이트 실패로 즉시 종료되는 호출에 선로딩·todo 보드 비용을 쓰지 않는다. 적용: `qa` (qa/SKILL.md § 사전 확인이 순서 SSOT).
```

- [ ] **Step 2-4: reference 재생성 + drift 검증**

Run: `python3 dp-skills/tools/docs_build.py && python3 dp-skills/tools/docs_build.py --check`
Expected: `--check` exit 0.

- [ ] **Step 2-5: pytest 회귀** (doctor 등 기존 도구가 SKILL.md 문구에 의존하지 않는지)

Run: `/opt/homebrew/bin/pytest dp-skills/tests/ -q`
Expected: 전부 PASS (`python3 -m pytest` 는 이 환경에서 불가 — homebrew pytest 사용).

- [ ] **Step 2-6: 커밋**

```bash
git add dp-skills/skills/qa/SKILL.md dp-skills/skills/context/shared/preamble.md dp-skills/docs/reference/
git commit -m "fix(qa): fail-fast 게이트 재배열 + STATE.md 부재 분기

종료가 확정된 호출(인자 오류·활성 project 없음)이 P-1 TodoWrite
선로딩·P0 메모 Read 를 다 치른 뒤에야 죽는 순서 문제. 게이트
(인자→P1→STATE.md)를 앞으로 옮기고, STATE.md 파일 부재를 행 부재와
동일 처리로 명세화 — 임기응변 추가 탐색 차단."
```

---

### Task 3: how-to 문서에 명시 호출 전용 정책 1줄

**Files:**
- Modify: `dp-skills/docs/how-to/qa-cycle.md` ("## 언제 쓰나" 섹션 끝)

- [ ] **Step 3-1: 정책 문단 추가** (Edit)

old_string:

```markdown
project 컨텍스트가 필요 없는 단발 운영 장애·핫픽스는 [`/dp-skills:issue`](../reference/skills/issue.md) 를 사용합니다 — 두 스킬은 공존합니다.
```

new_string:

```markdown
project 컨텍스트가 필요 없는 단발 운영 장애·핫픽스는 [`/dp-skills:issue`](../reference/skills/issue.md) 를 사용합니다 — 두 스킬은 공존합니다.

본 스킬은 **명시 호출 전용**입니다 — 대화에 Jira 링크·이슈 키를 붙여넣는 것만으로는 발동하지 않으며, `/dp-skills:qa {KEY}` 로 직접 호출해야 합니다.
```

- [ ] **Step 3-2: docs drift 확인**

Run: `python3 dp-skills/tools/docs_build.py --check`
Expected: exit 0 (how-to 는 수동 문서라 재생성 대상 아님 — drift 없어야 정상).

- [ ] **Step 3-3: 커밋**

```bash
git add dp-skills/docs/how-to/qa-cycle.md
git commit -m "docs(how-to): qa 명시 호출 전용 정책 안내 추가"
```

---

## 수동 검증 (push 전, 확률적 동작이라 관찰 항목)

- [ ] 새 Claude Code 세션에서 STATE.md 없는 워크스페이스에 Jira 티켓 링크만 붙여넣기 → qa 스킬 **비발동** 확인 (description 기반 억제는 확률적 — 1회 비발동이 보증은 아니나 회귀 신호로 충분).
- [ ] `/dp-skills:qa FAKE-1` 명시 호출 (STATE.md 없는 상태) → P-1 todo 보드·메모 Read 없이 "활성 project 가 없습니다…" 즉시 종료 확인.
- [ ] push 시 pre-push hook 통과 (`--no-verify` 금지).

## 후속 추적

- [claude-code#22345](https://github.com/anthropics/claude-code/issues/22345) 해결 확인 시 `disable-model-invocation: true` 를 qa frontmatter 에 추가하는 1줄 후속 PR (description 억제 문구는 이중 방어로 유지).
