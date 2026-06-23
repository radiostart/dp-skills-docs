# qa off 토글 + phase 전환 SSOT — 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> 설계 SSOT: [PLAN-qa-off-toggle.md](PLAN-qa-off-toggle.md). 본 문서는 그 설계를 단계별 작업으로 분해한 것.

**Goal:** `/dp-skills:qa off` 로 qa→development 복귀를 qa 스킬 안에서 가능하게 하고, phase 전환 절차를 공유 SSOT 로 추출하며 qa 자동진입의 STATE.md stale 결함을 함께 해결한다.

**Architecture:** phase 전환(`.agent-state.yml` phase/qa_started_at + STATE.md 행)을 `context/lifecycle/phase-transition.md` 한 파일로 추출하고, qa·project 양쪽이 참조한다. qa SKILL 에 off 인자 분기를 추가하고 자동진입을 SSOT 참조로 바꿔 STATE.md 갱신을 포함시킨다.

**Tech Stack:** Claude Code 플러그인 (markdown skill). 검증 = `tools/doctor.py`·`tools/docs_build.py`·`markdownlint-cli`·`tests/tools/`. 코드 실행 없음.

> **공통 검증** (pre-push 동일, `dp-skills/` 에서):
> ```bash
> python3 tools/doctor.py --schema
> python3 tools/docs_build.py --check
> npx -y markdownlint-cli "skills/**/*.md" "agents/**/*.md" "README.md"
> ```

> **회귀 핵심:** project 의 phase 전환·STATE.md 동작은 추출 후에도 **동일**해야 한다. 추출은 "문구 이동"이며 분기·Edit 규칙·예외 주석을 보존한다. 유일한 의도된 동작 변화 = qa 자동진입이 STATE.md 행을 `QA` 로 갱신.

---

### Task 1: 공유 SSOT `phase-transition.md` 신규 작성

**Files:**
- Create: `dp-skills/skills/context/lifecycle/phase-transition.md`

- [ ] **Step 1: 파일 작성**

`dp-skills/skills/context/lifecycle/phase-transition.md` 를 아래 내용으로 생성 (BEGIN/END 마커 제외):

--- BEGIN FILE CONTENT ---
# phase 전환 절차 (SSOT)

development ⇄ qa phase 전환의 단일 정의. `project`·`qa` 스킬이 본 파일을
참조한다 — 절차를 복붙하지 말 것 (drift 방지).

전환은 두 곳을 함께 갱신한다: 프로젝트의 `.agent-state.yml` 과 팀의 `STATE.md`.

## 공통 Edit 규칙

- `.agent-state.yml` 은 **Edit 도구만** 사용한다 (Write 금지 — 파일 전체
  덮어쓰기로 다른 필드 유실 위험).
- `phase:` 줄이 부재하면 (migrate-state 미수행 케이스) `tdd:` 줄 또는 `domain:`
  줄 다음에 한 줄을 Edit 으로 삽입한다. `schema:` 줄은 손대지 않는다 (schema
  업그레이드는 `migrate-state.py` 소관).
- `STATE.md` 는 [preamble.md](../shared/preamble.md) **P3** 규칙 — 테이블 본문을
  1행으로 교체, 기존 다른 이름 행은 모두 삭제 (이력은 git log). 현재 활성
  이름이 `{PROJECT}` 와 동일하면 행은 유지하되 status 칸만 갱신한다.

> **STATE.md status 칸의 `QA` 는 P3 의 literal `진행중` 을 대체하는 project
> 모드의 의도된 예외다 — preamble 으로 되돌리지 말 것.**

> phase 카운트·결함 잔여 등 동적 정보는 STATE.md 에 기록하지 않는다 — Jira UI
> 가 SSOT (PLAN-qa-phase §0 운영 전제).

## to-qa (development → qa)

1. `.agent-state.yml`: `phase:` 줄을 `phase: qa` 로 교체 (부재 시 삽입).
2. `.agent-state.yml`: `qa_started_at:` 행을 현재 UTC ISO 8601
   (`date -u +%Y-%m-%dT%H:%M:%SZ`) 값으로 교체. 행이 없으면 `phase:` 줄 다음에
   `qa_started_at: "{timestamp}"` 한 줄을 삽입한다.
3. `STATE.md` 행 → `| project | {PROJECT} | QA |`.
4. **idempotent**: 이미 `phase: qa` 면 phase 는 무변화, `qa_started_at` 만
   갱신한다 (마지막 진입 시점 기록 의도).

## to-development (qa → development)

1. `.agent-state.yml`: `phase:` 줄을 `phase: development` 로 교체 (부재 시
   `tdd:` 또는 `domain:` 줄 다음에 삽입).
2. `qa_started_at:` 은 **삭제하지 않고 보존**한다 (감사 이력 — 마지막 qa 진입
   시점). `qa/` 폴더도 **보존**한다.
3. `STATE.md` 행 → `| project | {PROJECT} | 진행중 |`.
4. **idempotent**: 이미 `phase: development` (또는 부재) 면 무변화, 다른 필드
   불변.
--- END FILE CONTENT ---

- [ ] **Step 2: 검증 (from `dp-skills/`)**
```bash
cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills
python3 tools/doctor.py --schema
npx -y markdownlint-cli "skills/context/lifecycle/phase-transition.md"
```
Expected: doctor PASS, markdownlint clean. (이 파일은 SKILL.md 가 아니므로 docs_build reference 생성 대상 아님 — `tools/docs_build.py` 는 `skills/*/SKILL.md`·`agents/*`·`tools/*` 만 추출.)

- [ ] **Step 3: 상대 링크 확인**
`../shared/preamble.md` 가 `skills/context/lifecycle/` 기준 `skills/context/shared/preamble.md` 로 resolve 되는지 확인:
```bash
ls dp-skills/skills/context/shared/preamble.md
```
Expected: 파일 존재.

- [ ] **Step 4: Commit**
```bash
cd /Users/jay-p/Projects/deali-skills-plugin
git add dp-skills/skills/context/lifecycle/phase-transition.md
git commit -m "feat(lifecycle): phase 전환 SSOT phase-transition.md 추가

to-qa·to-development 절차 + 공통 Edit 규칙. qa·project 가 참조할 단일 정의.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 2: `project/SKILL.md` 2-C·3 을 SSOT 참조로 전환 (동작 보존)

**Files:**
- Modify: `dp-skills/skills/project/SKILL.md` (§2-C ≈ 126-146, §3 ≈ 148-159)

- [ ] **Step 1: §2-C 본문을 SSOT 참조로 교체**

`### 2-C. phase 전환 (`{QA}` 있을 때만)` 헤딩 다음의 전체 블록(현재 "`​.agent-state.yml` 의 `phase` 필드를 in-place Edit 한다 …" 부터 "> `{TDD}` 플래그와 직교 …" 직전까지)을 아래로 교체:

```
[phase-transition.md](../context/lifecycle/phase-transition.md) 의 절차를 수행한다.

- **`{QA} == on`** → `to-qa` 절차.
  추가로 `features/` 폴더가 없거나 `features/*.md` (`.plan.md` 제외) 가 0건이면
  다음 WARN 출력 (종료는 안 함):

  ```
  features/ 가 비어있는데 QA phase 진입. /dp-skills:doctor 또는 /dp-skills:analyze 먼저 실행 권장.
  ```

- **`{QA} == off`** → `to-development` 절차.
```

(헤딩 `### 2-C. phase 전환 (`{QA}` 있을 때만)` 과 그 뒤 `> `{TDD}` 플래그와 직교 …` 주석 줄은 그대로 유지.)

- [ ] **Step 2: §3 STATE.md 갱신 본문을 SSOT 참조로 축약**

`### 3. STATE.md 갱신` 헤딩 다음의 본문(현재 "[preamble.md]… P3 수행 …" 부터 phase 분기 행 형식·예외 주석·동적 정보 주석까지)을 아래로 교체:

```
STATE.md 행 갱신은 [phase-transition.md](../context/lifecycle/phase-transition.md)
의 `to-qa`·`to-development` 절차에 포함되어 있다 (`{QA}` 입력 시 2-C 에서 함께
수행). `{QA}` 미입력(phase 변경 없음)일 때는 [preamble.md](../context/shared/preamble.md)
의 **P3** 를 수행하되, status 칸은 현재 `.agent-state.yml` 의 `phase` 로 분기한다:

- `phase == qa` → `| project | {PROJECT} | QA |`
- 그 외 (`development` 또는 부재) → `| project | {PROJECT} | 진행중 |`
```

- [ ] **Step 3: 회귀 read-through**

`dp-skills/skills/project/SKILL.md` 를 통독해 확인:
- 인자 파싱(§1)·`--qa off` 옵션·사용 예가 **변경 없이 유지**된다.
- `{QA} == on` 시 features WARN·idempotent·TDD 직교 주석이 유지된다.
- `{QA} == off` 가 to-development 로 매핑된다.
- `{QA}` 미입력 시 STATE.md status 분기(qa→QA, 그 외→진행중)가 보존된다.

- [ ] **Step 4: 검증 + Commit**
```bash
cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills
python3 tools/docs_build.py
python3 tools/doctor.py --schema
npx -y markdownlint-cli "skills/project/SKILL.md"
cd /Users/jay-p/Projects/deali-skills-plugin
git add dp-skills/skills/project/SKILL.md dp-skills/docs/reference/skills/project.md
git commit -m "refactor(project): phase 전환·STATE 갱신을 phase-transition.md 참조로

2-C·3 의 인라인 절차를 공유 SSOT 참조로 대체 (동작 보존). 복붙 제거.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 3: `qa/SKILL.md` — off 분기 + 자동진입 SSOT 참조

**Files:**
- Modify: `dp-skills/skills/qa/SKILL.md` (frontmatter, 전제 ≈ 16-23, §1 인자검증 ≈ 47-60, §3 자동진입 ≈ 75-103, 명시 비분기 ≈ 200-208)

- [ ] **Step 1: frontmatter description 에 off 추가**

`dp-skills/skills/qa/SKILL.md` frontmatter 의 description 끝 문장 앞(현재 "다건 추적·잔여 현황·재오픈은 Jira UI 가 SSOT …" 줄 앞)에 한 문장 추가:
```
  `off` 인자로 development phase 로 복귀할 수 있다.
```

- [ ] **Step 2: §1 인자 검증에 off 분기 추가**

`### 1. 인자 검증 (필수)` 헤딩 다음, 기존 `$ARGUMENTS 가 비어있으면 …` 분기 **앞에** 아래 블록을 삽입:

```
- `$ARGUMENTS` 가 `off` (대소문자 무시) 이면 → **off 모드** (아래 "### 1-A. off 모드" 로 진행, 이후 단계 건너뜀).
```

그리고 `### 1. 인자 검증 (필수)` 절 전체가 끝난 직후(현재 "검증 통과 시 입력값을 `{KEY}` 로 사용한다." 줄 다음)에 새 절을 삽입:

```
### 1-A. off 모드 (phase 탈출)

`$ARGUMENTS == off` 일 때만 수행하며, 완료 후 즉시 종료한다 (KEY 검증·결함
처리·이후 모든 단계 건너뜀).

1. [preamble.md](../context/shared/preamble.md) **P1**(팀) · **P2**(활성 project) 수행.
2. 활성 project 의 `.agent-state.yml` 의 `phase` 확인:
   - `phase != qa` (development 또는 부재) → `"이미 development phase 입니다."` 출력 후 종료 (변경 없음).
   - `phase == qa` → 다음으로.
3. [phase-transition.md](../context/lifecycle/phase-transition.md) 의 `to-development` 절차 수행.
4. 다음 한 줄 출력 후 종료:

   ```
   qa → development phase 복귀. features/ 수정 가능.
   ```

off 모드는 결함 처리·Jira 조회를 하지 않는다.
```

- [ ] **Step 3: §3 자동진입을 SSOT 참조로**

`### 3.` 의 phase 전환 블록(현재 "- `phase: development` **또는 `phase` 필드 부재** → 자동으로 qa phase 로 전환 후 진행:" 아래 1~4 번 절차)에서, `.agent-state.yml` 갱신 1번 항목을 SSOT 참조로 교체한다. 해당 블록을 아래로 교체:

```
- `phase: qa` → 다음 단계로 진행.
- `phase: development` **또는 `phase` 필드 부재** → 자동으로 qa phase 로 전환
  후 진행:

  1. [phase-transition.md](../context/lifecycle/phase-transition.md) 의 `to-qa`
     절차를 수행한다 (`.agent-state.yml` 의 `phase: qa`·`qa_started_at` 갱신
     **+ STATE.md 행 → `| project | {PROJECT} | QA |`**).

  2. 다음 한 줄을 출력한다:

     ```
     development → qa phase 자동 전환됨.
     ```

  3. `phase` 필드 부재인 경우에만 추가로 advisory 한 줄 출력 (종료 아님):

     ```
     (선택) schema 업그레이드는 'python3 ${CLAUDE_PLUGIN_ROOT}/tools/migrate-state.py workspace --apply' — 동작에는 필수 아님
     ```

  4. 다음 단계로 진행.
```

> 이 교체의 핵심: 기존엔 `.agent-state.yml` 만 갱신했으나 이제 to-qa 가
> STATE.md 행도 함께 갱신 → stale 결함 해소.

- [ ] **Step 4: 명시 비분기 안내 갱신**

`## 명시 비분기 (out of scope)` 의 features/ 수정 안내 항목(현재 "features 본문 수정이 필요하면 `/dp-skills:project {PROJECT} --qa off` 로 development 로 복귀한 뒤 별도 사이클 진행." 부분)을 아래로 교체:

```
  features 본문 수정이 필요하면 `/dp-skills:qa off` 또는
  `/dp-skills:project {PROJECT} --qa off` 로 development 로 복귀한 뒤 별도
  사이클 진행.
```

- [ ] **Step 5: 회귀 read-through**

`dp-skills/skills/qa/SKILL.md` 통독:
- KEY 입력(`^[A-Z]+-\d+$`) 경로가 종전과 동일하게 동작한다.
- off 모드가 KEY 검증·결함 처리를 건너뛴다.
- 자동진입이 to-qa 참조로 STATE.md 까지 갱신한다.

- [ ] **Step 6: 검증 + Commit**
```bash
cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills
python3 tools/docs_build.py
python3 tools/doctor.py --schema
npx -y markdownlint-cli "skills/qa/SKILL.md"
cd /Users/jay-p/Projects/deali-skills-plugin
git add dp-skills/skills/qa/SKILL.md dp-skills/docs/reference/skills/qa.md
git commit -m "feat(qa): off 토글 추가 + 자동진입 STATE.md stale 결함 수정

/dp-skills:qa off — qa→development 복귀. 자동진입을 to-qa SSOT 참조로 바꿔
STATE.md 행도 QA 로 갱신 (기존 stale 결함 해소).

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 4: how-to `qa-cycle.md` 동기화 + plugin.json bump

**Files:**
- Modify: `dp-skills/docs/how-to/qa-cycle.md` (§6 복귀 ≈ 85-93, 흔한 실수 ≈ 101-102)
- Modify: `dp-skills/.claude-plugin/plugin.json`

- [ ] **Step 1: §6 development 복귀에 qa off 추가**

`dp-skills/docs/how-to/qa-cycle.md` 의 `### 6. (선택) development 복귀` 절에서 복귀 명령 예시(현재 `/dp-skills:project MyProject --qa off`)를 두 경로로 갱신. 해당 코드블록을 아래로 교체:

```
/dp-skills:qa off
# 또는 다른 project 로 전환하며 탈출:
/dp-skills:project MyProject --qa off
```

그리고 그 아래 설명 줄에 한 문장 추가:
```
qa 스킬 안에서 바로 나가려면 `/dp-skills:qa off` 가 간편합니다 — 현재 활성 project 를 development 로 되돌립니다.
```

- [ ] **Step 2: 흔한 실수의 stale 문구 수정**

`dp-skills/docs/how-to/qa-cycle.md` 흔한 실수 항목 중 자동전환 관련 stale 문구를 현재 동작에 맞게 교체. 아래 두 줄을 찾아:
```
- **phase 전환 없이 `/dp-skills:qa` 호출** — `phase: development` 상태에서 호출하면 "QA 진입하려면 `/dp-skills:project X --qa` 먼저 호출" 안내 후 종료됩니다. 자동 전환 없음.
- **features/ 본문을 phase=qa 에서 수정 시도** — hook 가 차단합니다. features 수정이 필요하면 `--qa off` 로 복귀 후 별도 development 사이클.
```
아래로 교체:
```
- **features/ 본문을 phase=qa 에서 수정 시도** — hook 가 차단합니다. features 수정이 필요하면 `/dp-skills:qa off` (또는 `/dp-skills:project X --qa off`) 로 development 복귀 후 별도 사이클.
```

(주: 첫 줄은 현재 qa SKILL 이 자동 전환을 하므로 사실과 달라 삭제한다.)

- [ ] **Step 3: §1 전환 안내에 자동전환 명시 (정합)**

`### 1. development → qa 전환` 절 끝에 한 문장 추가 (자동전환과 정합):
```
> 참고: `/dp-skills:qa {KEY}` 를 development phase 에서 직접 호출해도 qa 로 자동 전환되므로, 이 수동 전환은 결함 없이 phase 만 미리 바꾸고 싶을 때만 필요합니다.
```

- [ ] **Step 4: plugin.json minor bump**

`dp-skills/.claude-plugin/plugin.json` 의 `"version"` 을 한 minor 올린다. main 기준 현재값을 확인 후 +0.1.0:
```bash
grep version dp-skills/.claude-plugin/plugin.json
```
현재 `0.13.0` 이면 `0.14.0` 으로 Edit. (dp-code-review·run-cycle PR 이 먼저 머지되면 충돌 — 그때 최신 + 0.1.0 으로 재조정.)

- [ ] **Step 5: 검증**
```bash
cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills
npx -y markdownlint-cli "docs/how-to/qa-cycle.md"
python3 -c "import json; json.load(open('.claude-plugin/plugin.json'))"
python3 tools/docs_build.py --check
```
Expected: markdownlint clean, JSON valid, docs_build `--check` PASS.

- [ ] **Step 6: Commit**
```bash
cd /Users/jay-p/Projects/deali-skills-plugin
git add dp-skills/docs/how-to/qa-cycle.md dp-skills/.claude-plugin/plugin.json
git commit -m "docs(qa): qa-cycle how-to 에 off 토글 반영 + stale 자동전환 문구 수정

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 5: 전체 검증 (pre-push 동등)

- [ ] **Step 1: reference 재생성 + clean 확인**
```bash
cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills
python3 tools/docs_build.py
cd .. && git status --short
```
Expected: working tree clean (reference 가 Task 2·3 에서 커밋됨). 남으면 add/commit.

- [ ] **Step 2: pre-push 스위트 전체**
```bash
cd /Users/jay-p/Projects/deali-skills-plugin/dp-skills
python3 tools/doctor.py --schema
for t in tests/tools/test_*.py; do [ -f "$t" ] && env -u GIT_DIR -u GIT_WORK_TREE -u GIT_INDEX_FILE -u GIT_PREFIX python3 "$t"; done
npx -y markdownlint-cli "skills/**/*.md" "agents/**/*.md" "README.md"
python3 tools/docs_build.py --check
```
Expected: doctor 0 ERROR (version>tag WARN 정상), tools 테스트 전부 통과, markdownlint clean, docs_build `--check` PASS.

- [ ] **Step 3: SSOT 참조 무결성 read-through**
- `phase-transition.md` 가 qa·project 양쪽에서 정확한 상대경로로 링크되는지 (`../context/lifecycle/phase-transition.md` from skills/{qa,project}/).
- qa off·자동진입·project 2-C 가 모두 같은 to-qa/to-development 절차를 가리키는지 (복붙 흔적 없음).

---

## Self-Review 체크

- **Spec coverage:** §2 phase-transition.md→Task1 · §3 qa(off·자동진입·안내)→Task3 · §4 project 참조전환→Task2 · §5 문서/버전→Task4 · §6 검증→Task5. 전 항목 매핑.
- **Placeholder scan:** TBD/TODO 없음. 모든 Edit 에 정확한 교체 문자열 제공.
- **Type consistency:** 절차명 `to-qa`/`to-development`, 파일명 `phase-transition.md`, STATE 행 형식(`| project | {P} | QA |` / `진행중`)이 Task1·2·3 간 일치.
- **README 주의:** 이 브랜치(main 기반) README 는 17종이며 qa 행이 없다 — qa off 는 스킬 이름·개수를 바꾸지 않으므로 README 스킬 표는 건드리지 않는다 (불필요한 #26 충돌 회피). description·how-to 로 충분.
