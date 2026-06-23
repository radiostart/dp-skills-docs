# 병렬 셀프 코드 리뷰 스킬 — 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> 설계 SSOT: [PLAN-parallel-code-review.md](PLAN-parallel-code-review.md). 본 문서는 그 설계를 단계별 작업으로 분해한 것.

**Goal:** `@ag-code-reviewer` 5축을 하이브리드 fan-out 으로 병렬 실행하는 오케스트레이터 스킬 `/dp-skills:dp-code-review` 를 추가한다.

**Architecture:** 새 스킬이 메인 루프 오케스트레이터로 동작 — 범위 수집 → fan-out(전역 2 + 청크 G) → barrier → dedup/필터 → 단일 CODE REVIEW REPORT. worker 는 `@ag-code-reviewer` 에 `--axis`·`--files` 인자를 더해 재활용. 인자 미지정 시 멘션 동작 100% 보존.

**Tech Stack:** Claude Code 플러그인 (markdown 기반 skill/agent), 검증은 `tools/doctor.py`·`tools/docs_build.py`·`markdownlint-cli`. 코드 실행 없음 — "테스트" = 구조·drift·lint 검증.

> **공통 검증 명령** (pre-push hook `.githooks/pre-push` 와 동일, `dp-skills/` 에서 실행):
> ```bash
> python3 tools/doctor.py --schema
> python3 tools/docs_build.py --check
> npx -y markdownlint-cli "skills/**/*.md" "agents/**/*.md" "README.md"
> ```

---

### Task 1: `ag-code-reviewer` 에 스코프 인자 `--axis`·`--files` 추가

**Files:**
- Modify: `dp-skills/agents/ag-code-reviewer.md:20` (호출 인자 줄)
- Modify: `dp-skills/agents/ag-code-reviewer.md:65` (§4 "검토 5축 실행" 진입부)

- [ ] **Step 1: 호출 인자 줄에 인자 2개 추가**

`dp-skills/agents/ag-code-reviewer.md:20` 의 기존 줄:

```
호출 인자: `--base <branch>` · `--commits <range>` · `--pr <N>` · `--lang <slug>` · `--threshold <N>` (0~100, 기본 80 — 미만은 자동 dismiss).
```

다음으로 교체:

```
호출 인자: `--base <branch>` · `--commits <range>` · `--pr <N>` · `--lang <slug>` · `--threshold <N>` (0~100, 기본 80 — 미만은 자동 dismiss) · `--axis <slugs>` (쉼표구분: `cross-cutting`·`lang`·`hunk`·`callers`·`missing` — 미지정 시 전체 5축) · `--files <list>` (쉼표구분 경로 — 미지정 시 전체 변경 파일). `--axis`·`--files` 는 `/dp-skills:dp-code-review` 오케스트레이션이 주입한다.
```

- [ ] **Step 2: §4 진입부에 "스코프 분기" 블록 추가**

`dp-skills/agents/ag-code-reviewer.md:65` 의 `### 4. 검토 5축 실행` 헤딩 바로 다음 줄(기존 "각 축은 evaluator 책임 영역과 **명시적으로 분리**..." 앞)에 아래 단락을 삽입:

```
**스코프 분기 (오케스트레이션용):** `--axis` 가 주어지면 나열된 축만 실행한다 (slug→축: `cross-cutting`→1·`lang`→2·`hunk`→3·`callers`→4·`missing`→5). `--files` 가 주어지면 1단계 변경 범위 수집을 해당 파일로 한정한다. **둘 다 미지정이면 종전대로 전체 변경 파일에 5축 전부 실행** — 멘션 직접 호출 동작 불변.

```

- [ ] **Step 3: 멘션 동작 불변 확인 (read-through)**

`dp-skills/agents/ag-code-reviewer.md` 를 통독해 §1 범위 수집·§3 컨텍스트 로드·§5 출력이 인자 미지정 경로에서 종전과 동일한지 확인. 새 블록이 "둘 다 미지정이면 종전대로"를 명시하는지 확인.

- [ ] **Step 4: 검증**

Run (in `dp-skills/`):
```bash
python3 tools/docs_build.py --check || python3 tools/docs_build.py
python3 tools/doctor.py --schema
npx -y markdownlint-cli "agents/ag-code-reviewer.md"
```
Expected: docs_build 는 drift 시 재생성 후 통과, doctor `--schema` PASS, markdownlint 통과(에러 0).

- [ ] **Step 5: Commit**

```bash
git add dp-skills/agents/ag-code-reviewer.md dp-skills/docs/reference/agents/ag-code-reviewer.md
git commit -m "feat(agents): ag-code-reviewer 에 --axis·--files 스코프 인자 추가

인자 미지정 시 종전 전체 5축 동작 보존. dp-code-review 오케스트레이션이 주입.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 2: `dp-code-review` 스킬 신규 작성

**Files:**
- Create: `dp-skills/skills/dp-code-review/SKILL.md`

- [ ] **Step 1: SKILL.md 전체 작성**

`dp-skills/skills/dp-code-review/SKILL.md` 를 아래 내용으로 생성:

````markdown
---
name: dp-code-review
description: >-
  @ag-code-reviewer 의 5축 셀프 리뷰를 병렬 하이브리드 fan-out 으로 실행한다.
  전역 축(cross-cutting·callers)은 전체 diff 1회, 로컬 축(lang·hunk·missing)은
  파일그룹으로 분할 병렬 실행 후 단일 CODE REVIEW REPORT 로 병합한다. 큰 PR 의
  sampling 한계를 해소하고 5축을 동시 실행한다. 인자 없이 @ag-code-reviewer 를
  멘션하면 종전 단일 순차 동작 그대로 — 본 스킬은 그 위의 오케스트레이션 레이어.
  공식 /code-review 와 별개 (팀 컨벤션·언어 룰 전용, 사이클 밖 도구).
---

# /dp-skills:dp-code-review

`@ag-code-reviewer` 를 worker 로 병렬 호출하는 오케스트레이터. 5축 로직의 SSOT 는
`agents/ag-code-reviewer.md` — 본 스킬은 범위 분할·병렬 호출·병합만 담당한다.
**사이클 밖** 도구이며 공식 `/code-review` 와 별개.

대상: $ARGUMENTS  (옵션: `--base <branch>` · `--threshold <N>` · `--lang <slug>` · `--chunk <N>`)

## 사전 확인

[preamble.md](../context/shared/preamble.md) **P-1** 수행 (3단계 이상 — `ToolSearch select:TodoWrite` 선로딩 후 단계 목록 게시). 프로젝트 컨텍스트 독립 — 활성 프로젝트 없어도 동작 (workspace-less 시 scope 로드만 skip).

## 1. 범위 수집

`--base` 미지정 시 `origin/HEAD` 또는 `workspace/_common/config.md` 의 `pr_default_base`:

```bash
git diff --name-status <base>...HEAD
git diff --shortstat <base>...HEAD
```

변경 없음 → 즉시 종료.

## 2. 언어 감지

`--lang` 미지정 시 dominant 확장자로 감지 (`agents/ag-code-reviewer.md` §2 매핑 동일).

## 3. fan-out 입도 결정 (G)

`git diff --shortstat` 의 파일수·라인수로 분기:

- **≤ 3 파일** → 단일 폴백: `@ag-code-reviewer --base <B> --lang <L> --threshold <T>` 1개만 호출 (종전 동작). 병렬 오버헤드 회피. 4~5단계 skip, 그 리포트를 6단계에서 그대로 사용.
- **≤ 30 파일 AND ≤ 1500 라인** → `G = 1`.
- **초과** → `G = ceil(변경파일수 / chunk)`, chunk 기본 12 (`--chunk N` 으로 조정).

변경 파일 목록을 경로 정렬 후 G개 그룹으로 균등 분배.

## 4. 병렬 fan-out

**한 메시지에 아래 Agent 호출을 모두 묶어** 동시 실행한다 (harness 병렬). 모두 `subagent_type: ag-code-reviewer`, 읽기 전용:

- worker A — `--axis cross-cutting --base <B> --lang <L>`  (전역, 전체 diff)
- worker B — `--axis callers --base <B> --lang <L>`  (전역, 전체 diff)
- worker C_i (i=1..G) — `--axis lang,hunk,missing --files <group_i> --base <B> --lang <L>`  (청크)

각 worker 는 자기 축의 부분 findings(`[severity][confidence:N][path:line] ...`) 만 반환한다.

## 5. barrier → 병합

모든 worker 반환 후:

1. **dedup** — `(path:line, severity)` 동일 finding 1개로 병합 (전역·청크 축 중복 제거).
2. **confidence 필터** — `--threshold` (기본 80) 미만 dismiss, 카운트 보관.
3. **verdict** — 필터 통과분 기준: `critical/major ≥1 → block` · `minor/nit 만 → nit` · 빈 → `approve`.

## 6. 출력 — CODE REVIEW REPORT

[messages.md](../context/shared/messages.md) `code_review_report_example` 포맷으로 단일 리포트 출력. 필수 필드 동일:

- `scope_reviewed` 에 **`parallel: worker {2+G}개 / 그룹 {G}개, sampling 없음`** (단일 폴백 시 `single fallback (≤3 files)`) 명시 — 잘림 없음 투명화.
- `scope_skipped` 에 `low-confidence: N filtered (threshold T)` + evaluator 책임 영역.
- `next` → verdict 별 `/dp-skills:fix-review` 라우팅 (`agents/ag-code-reviewer.md` §5 문구 동일).

## Do-NOT

- worker 결과 외 추측 finding 추가 금지 — 병합만, 새 판정 금지.
- evaluator gate 영역 (requirements·tdd_evidence·test_run·capture_lockdown·open_questions) 금지.
- 파일 수정·생성·삭제 금지 — worker 도 본 스킬도 읽기 전용.
````

- [ ] **Step 2: 구조 검증**

Run (in `dp-skills/`):
```bash
python3 tools/docs_build.py
python3 tools/doctor.py --schema
npx -y markdownlint-cli "skills/dp-code-review/SKILL.md"
```
Expected: docs_build 가 `docs/reference/skills/dp-code-review.md` 를 생성, doctor `--schema` PASS, markdownlint 통과.

- [ ] **Step 3: frontmatter name 일치 확인**

`grep -n "name: dp-code-review" dp-skills/skills/dp-code-review/SKILL.md` → 1건. 디렉터리명(`dp-code-review`)과 frontmatter `name` 이 일치하는지 확인 (docs_build·플러그인 로더 규약).

- [ ] **Step 4: Commit**

```bash
git add dp-skills/skills/dp-code-review/SKILL.md dp-skills/docs/reference/skills/dp-code-review.md
git commit -m "feat(skills): dp-code-review 병렬 셀프 리뷰 오케스트레이터 추가

전역 2 + 청크 G worker fan-out → barrier → dedup/필터 → 단일 REPORT.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 3: README · plugin.json 동기화

**Files:**
- Modify: `dp-skills/README.md:72` (스킬 개수 헤딩)
- Modify: `dp-skills/README.md:90` 부근 (스킬 표에 행 추가)
- Modify: `dp-skills/README.md:359` 부근 (다른 도구와의 분리 표에 행 추가)
- Modify: `dp-skills/.claude-plugin/plugin.json:3` (version)

- [ ] **Step 1: 스킬 개수 17→18**

`dp-skills/README.md:72` 의 `## 스킬 커맨드 (17종)` → `## 스킬 커맨드 (18종)`.

- [ ] **Step 2: 스킬 표에 행 추가**

`dp-skills/README.md` 스킬 표에서 `code-review-init` 행 바로 아래에 추가:

```
| `dp-code-review` | 병렬 하이브리드 셀프 코드 리뷰 | [셀프 코드 리뷰](docs/how-to/self-code-review.md) |
```

- [ ] **Step 3: "다른 도구와의 분리" 표에 행 추가**

`dp-skills/README.md` 의 `### 다른 도구와의 분리` 표에서 `@ag-code-reviewer` 행 바로 아래에 추가:

```
| `/dp-skills:dp-code-review` | `@ag-code-reviewer` 5축 병렬 fan-out (사이클 밖) | git diff | 단일 CODE REVIEW REPORT (chat) |
```

- [ ] **Step 4: 버전 bump**

`dp-skills/.claude-plugin/plugin.json:3` 의 `"version": "0.13.0"` → `"version": "0.14.0"`.

- [ ] **Step 5: 검증**

Run (in `dp-skills/`):
```bash
npx -y markdownlint-cli "README.md"
python3 -c "import json; json.load(open('.claude-plugin/plugin.json'))"
```
Expected: markdownlint 통과, JSON 파싱 에러 없음.

- [ ] **Step 6: Commit**

```bash
git add dp-skills/README.md dp-skills/.claude-plugin/plugin.json
git commit -m "chore(release): bump 0.14.0

dp-code-review 스킬 추가 반영 (스킬 18종) + 도구 분리표 갱신.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 4: 전체 검증 (pre-push 동등) + docs reference 최종 동기화

**Files:**
- Modify (생성됨): `dp-skills/docs/reference/**` (docs_build 산출물)

- [ ] **Step 1: reference 재생성**

Run (in `dp-skills/`):
```bash
python3 tools/docs_build.py
```
Expected: `docs/reference/skills/dp-code-review.md`, `docs/reference/agents/ag-code-reviewer.md` 갱신.

- [ ] **Step 2: pre-push 검증 스위트 전체 실행**

Run (in `dp-skills/`):
```bash
python3 tools/doctor.py --schema
for t in tests/tools/test_*.py; do [ -f "$t" ] && env -u GIT_DIR -u GIT_WORK_TREE -u GIT_INDEX_FILE -u GIT_PREFIX python3 "$t"; done
npx -y markdownlint-cli "skills/**/*.md" "agents/**/*.md" "README.md"
python3 tools/docs_build.py --check
```
Expected: 전부 exit 0, docs_build `--check` drift 없음.

- [ ] **Step 3: 멘션 회귀 스모크 (수동)**

작은 변경(≤3 파일)이 있는 브랜치에서 `@ag-code-reviewer` 인자 없이 멘션 → 종전과 동일하게 단일 5축 CODE REVIEW REPORT 가 나오는지 육안 확인. (인자 경로 추가가 무인자 동작을 깨지 않았음을 보장.)

- [ ] **Step 4: Commit (reference drift 있을 경우만)**

```bash
git add dp-skills/docs/reference
git commit -m "chore(docs): 레퍼런스 인덱스 동기화

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Self-Review 체크

- **Spec coverage:** §2 인자확장→Task 1 · §3·4·5 fan-out/병합→Task 2 · §6 변경범위(README·plugin.json·docs)→Task 3·4. 전 항목 매핑됨.
- **Placeholder scan:** TBD/TODO 없음. 모든 Edit 에 정확한 교체 문자열·경로·명령 포함.
- **Type consistency:** axis slug(cross-cutting·lang·hunk·callers·missing)·verdict 규칙·`--axis`/`--files`/`--chunk` 인자명이 Task 1·2 간 일치.
