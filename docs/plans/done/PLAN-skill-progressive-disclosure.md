# SKILL.md Progressive Disclosure Implementation Plan

> **For agentic workers:** 본 계획은 task 단위로 순차 실행한다. 각 task 는 독립
> 커밋 — 회귀 시 bisect 가능. 체크박스(`- [ ]`)로 진행 추적.

**Goal:** 장문 SKILL.md (analyze·qa·create-feature·discover) 본문을 절차
골격(분기 조건·실행 순서·금지 사항)만 남기고 상세(알고리즘·템플릿·예시)를
`references/` 로 분리하고, ag-planner.md 는 절차 후반 누락 방지를 위한 구조
재배치를 적용한다.

**Architecture:** `skills/project/SKILL.md` 의 패턴(본문 골격 + 상세 링크)을
따른다. 동작 보존이 최우선 — 지시 내용을 바꾸지 않고 위치만 옮긴다. 이동은
verbatim, 본문에는 한 줄 stub + 링크만 남긴다.

**Tech Stack:** markdown, `tools/docs_build.py` (docs/reference 동기화),
markdownlint, pytest (tests/tools 회귀).

---

## 측정 기준선 (2026-06-10, main=908eff7)

| 파일 | 현재 줄수 | 분리 후 목표 |
| --- | --- | --- |
| skills/analyze/SKILL.md | 281 | ~215 |
| skills/qa/SKILL.md | 287 | ~240 |
| skills/create-feature/SKILL.md | 216 | ~155 |
| skills/discover/SKILL.md | 211 | ~160 |
| agents/ag-planner.md | 191 | 재배치 (줄수 비목표) |

- 테스트 기준선: `pytest tests/tools -q` → **460 passed**
- plugin.json: 0.16.13 → 이번 작업에서 **0.16.14** patch bump

## 동작 보존 제약 (사전 조사 결과)

1. **analyze "분석 프로세스" 단계 번호 (1~8) 변경 금지** —
   `skills/project/SKILL.md` (7·8단계) 와 `skills/create-feature/SKILL.md`
   (4·5단계) 가 "분석 프로세스 N~M단계" 를 번호로 참조한다.
2. **qa "### 4-1. qa/ 산출물 명명 규약" 표는 본문 유지** —
   `agents/ag-planner.md` 가 `규약 SSOT: qa/SKILL.md "qa/ 산출물 명명 규약"`
   으로 텍스트 참조. doctor 검사는 코드(tools/doctor/integrity.py)라 무관하지만
   SSOT 문구가 가리키는 위치를 옮기지 않는다.
3. **docs_build.py 는 `skills/*/SKILL.md` 만 추출** — references/*.md 는 사이트
   미포함, SKILL.md 안의 `references/...` 링크는 GitHub blob URL 로 자동 변환.
   docs_build 수정 불필요. 단 SKILL.md 본문이 줄면 docs/reference/skills/*.md
   재생성 필요 (커밋 대상).
4. **markdownlint 는 `skills/**/*.md` 전체 적용** — 신규 references 파일도
   `.markdownlint.json` 룰 통과 필요.
5. ag-planner 의 인라인 블록 이동은 **같은 파일 내 재배치만** — 활성화 분기는
   원위치(step 1·step 4)에 유지하고 본문 블록만 후방 섹션으로 옮긴다.

## 공통 검증 절차 (각 task 의 커밋 직전 실행)

```bash
cd dp-skills
# 1. 상대 링크 유효성 (해당 스킬 폴더)
python3 - <<'EOF'
import re, sys, pathlib
bad = []
for md in pathlib.Path("skills").rglob("*.md"):
    for m in re.finditer(r"\]\(([^)\s#]+)(?:#[^)\s]*)?\)", md.read_text()):
        t = m.group(1)
        if t.startswith(("http", "mailto:", "${")):
            continue
        if not (md.parent / t).resolve().exists():
            bad.append(f"{md}: {t}")
for md in pathlib.Path("agents").glob("*.md"):
    for m in re.finditer(r"\]\(([^)\s#]+)(?:#[^)\s]*)?\)", md.read_text()):
        t = m.group(1)
        if t.startswith(("http", "mailto:", "${")):
            continue
        if not (md.parent / t).resolve().exists():
            bad.append(f"{md}: {t}")
print("\n".join(bad) or "links OK")
sys.exit(1 if bad else 0)
EOF
# 2. lint
npx -y markdownlint-cli "skills/**/*.md" "agents/**/*.md"
# 3. docs/reference 재생성 (drift 방지 — 커밋에 포함)
python3 tools/docs_build.py
# 4. 회귀
pytest tests/tools -q   # 460 passed 유지
```

---

### Task 0: 계획 문서 커밋

**Files:**

- Create: `dp-skills/docs/plans/PLAN-skill-progressive-disclosure.md` (본 문서)

- [x] **Step 1:** 본 문서 작성 (mkdocs `exclude_docs: plans/*` 로 사이트 자동
  제외 — mkdocs.yml 수정 불필요)
- [x] **Step 2:** 커밋

```bash
git add dp-skills/docs/plans/PLAN-skill-progressive-disclosure.md
git commit -m "docs(plans): SKILL.md progressive disclosure 계획"
```

---

### Task 1: analyze/SKILL.md 분리

**Files:**

- Create: `dp-skills/skills/analyze/references/feature-template.md`
- Create: `dp-skills/skills/analyze/references/project-md-update.md`
- Modify: `dp-skills/skills/analyze/SKILL.md`
- Regen: `dp-skills/docs/reference/skills/analyze.md`

- [ ] **Step 1: feature-template.md 생성** — "### 3. 기능 분할 및 구조화" 의
  `**각 기능에서 추출할 항목:**` 직후 ```` ```markdown ```` 템플릿 블록
  (`# #{번호} {기능명}` ~ 닫는 ```` ``` ````) 을 verbatim 이동. 파일 헤더는
  `# features 파일 템플릿` + 한 줄 설명. 본문 stub:
  `**각 기능에서 추출할 항목** — features 파일 템플릿 (요구사항 3축·상태 전환·비즈니스 규칙·예외 케이스): [\`references/feature-template.md\`](references/feature-template.md).`
  분할 기준 3 bullets 와 "분석 시 주의사항" 4 bullets 는 본문 유지 (금지 사항).
- [ ] **Step 2: project-md-update.md 생성** — 아래 3 블록을 verbatim 이동:
  - 5-1 의 `**갱신 예시:**` + ```` ```markdown ```` 블록
  - 5-2 의 `**프로세스:**` numbered 1~4 전체
  - 5-2 의 `**갱신 규칙:**` 5 bullets 전체
  파일 헤더 `# project.md 갱신 상세 (5-1 목표 · 5-2 관련 파일)`.
  본문 5-1 은 numbered 1~4 단계(분기·보존 규칙) 유지 + 예시만 링크로:
  `갱신 예시: [\`references/project-md-update.md\`](references/project-md-update.md)`.
  본문 5-2 는 첫 문단 유지 + stub:
  `갱신 시 features/ 에 명시된 모델·서비스·라우트는 누락 금지, 기존 사용자 수동 기입 행은 보존. 표 기입 프로세스·갱신 규칙 상세: [\`references/project-md-update.md\`](references/project-md-update.md).`
  cross-domain detect 블록·Open Questions 보존 블록·A2 fallback 은 본문 유지
  (분기·출력 계약).
- [ ] **Step 3:** "분석 프로세스" 1~8 단계 번호 불변 확인 (`grep -n "^### " skills/analyze/SKILL.md`)
- [ ] **Step 4:** 공통 검증 절차 실행 → 전부 통과
- [ ] **Step 5: 커밋**

```bash
git add dp-skills/skills/analyze dp-skills/docs/reference
git commit -m "refactor(analyze): 템플릿·project.md 갱신 상세를 references/ 로 분리"
```

---

### Task 2: qa/SKILL.md 분리

**Files:**

- Create: `dp-skills/skills/qa/references/jira-prefill.md`
- Create: `dp-skills/skills/qa/references/branch-naming.md`
- Modify: `dp-skills/skills/qa/SKILL.md`
- Regen: `dp-skills/docs/reference/skills/qa.md`

- [ ] **Step 1: jira-prefill.md 생성** — 아래 2 블록 verbatim 이동:
  - 5-2 의 `` `jira.py fetch` 의 stdout 은 다음 섹션을 포함한다 (T2 산출물 형식): `` + 5 bullets
  - 5-3 의 주입 규칙 표 (`| 템플릿 토큰·섹션 | 채우는 값 |` 전체) + "fetch 출력
    중 다음은 폐기" 2 bullets + `> jira:`/`> reported:` 보강 문장
  본문 5-2 는 fetch 명령 + 성공/실패 분기 + "빈 파일을 생성하지 않는다" 유지.
  본문 5-3 stub: example 템플릿 경로 문장 유지 +
  `fetch 결과 주입 규칙 (토큰 매핑·폐기 항목·placeholder 유지): [\`references/jira-prefill.md\`](references/jira-prefill.md).`
  + 회신 섹션 금지 blockquote + qa/ lazy 생성 문장 유지.
- [ ] **Step 2: branch-naming.md 생성** — 6-1 의 bash 블록 + 매칭 유/무 분기
  bullets + 예시를 verbatim 이동. 본문 6-1 stub:
  `base 는 \`fix/{KEY}\`. 기존 동일 base 브랜치 (local + remote) 가 있으면 suffix 최댓값 + 1 로 \`fix/{KEY}-{N+1}\` 을 쓴다 — 조사 명령·예시: [\`references/branch-naming.md\`](references/branch-naming.md).`
  6-2 질의 분기 (Y/n·non-interactive fail-safe·HEAD 분기 안내) 전부 본문 유지.
- [ ] **Step 3:** "### 4-1. qa/ 산출물 명명 규약" 이 본문에 그대로 있는지 확인
  (`grep -n "산출물 명명 규약" skills/qa/SKILL.md`)
- [ ] **Step 4:** 공통 검증 절차 실행 → 전부 통과
- [ ] **Step 5: 커밋**

```bash
git add dp-skills/skills/qa dp-skills/docs/reference
git commit -m "refactor(qa): Jira prefill·브랜치 명명 상세를 references/ 로 분리"
```

---

### Task 3: create-feature/SKILL.md 분리

**Files:**

- Create: `dp-skills/skills/create-feature/references/feature-template.md`
- Create: `dp-skills/skills/create-feature/references/open-questions.md`
- Modify: `dp-skills/skills/create-feature/SKILL.md`
- Regen: `dp-skills/docs/reference/skills/create-feature.md`

- [ ] **Step 1: feature-template.md 생성** — "### 3. `features/NN-{slug}.md`
  작성" 의 ```` ```markdown ```` 템플릿 블록 (`# #{NN} {기능명}` ~ 닫는 ```` ``` ````)
  verbatim 이동. 본문 stub:
  `아래 템플릿으로 Write — 전체 템플릿: [\`references/feature-template.md\`](references/feature-template.md).`
  "프롬프트에서 명시적으로 추출 가능한 요소…추측성 내용은 넣지 않는다" 문단과
  "Open Questions 작성 규칙" 4 bullets 는 본문 유지 (금지·분기).
- [ ] **Step 2: open-questions.md 생성** — 3-bis 의 detect 절차 numbered 1~3
  (MANIFEST lookup·행 추가·INFO 출력 형식) + "Open Questions 카테고리 분류
  기준" 4 bullets verbatim 이동. 본문 3-bis 는 첫 문단 +
  `상세 절차·분류 기준: [\`references/open-questions.md\`](references/open-questions.md).`
  + A2 runtime fallback blockquote + 편집물 보존 가드 blockquote 유지.
- [ ] **Step 3:** 4·5 단계의 "analyze 분석 프로세스 5~6단계 / 6-5단계" 참조
  문구가 불변인지 확인
- [ ] **Step 4:** 공통 검증 절차 실행 → 전부 통과
- [ ] **Step 5: 커밋**

```bash
git add dp-skills/skills/create-feature dp-skills/docs/reference
git commit -m "refactor(create-feature): 템플릿·OQ 분류 상세를 references/ 로 분리"
```

---

### Task 4: discover/SKILL.md 분리

**Files:**

- Create: `dp-skills/skills/discover/references/output-format.md`
- Modify: `dp-skills/skills/discover/SKILL.md`
- Regen: `dp-skills/docs/reference/skills/discover.md`

- [ ] **Step 1: output-format.md 생성** — 아래 3 템플릿 블록 verbatim 이동:
  - 단계 3 의 미리 보기 표 코드 블록 (`추출 결과 미리 보기 (domain: {domain})` ~ `계속할까요? (y/n)`)
  - 단계 4-2 의 rules 스켈레톤 ```` ```markdown ```` 블록
  - "## 결과 출력" 의 2 개 코드 블록 (완료 요약 + 분할 발생 시)
  본문 유지: 승인 게이트 분기 (승인 없이 기록 금지·거부 시 Abort 계약·--dry-run
  분기), rules 생성 조건 (부재 시에만·기존 보존·scope/rules 경계), 결과 출력
  1줄 + 링크.
  stub 예 — 단계 3: `미리 보기 표·기록 파일 목록 형식: [\`references/output-format.md\`](references/output-format.md).`
  결과 출력: `출력 형식 (완료 요약·분할 발생 시 안내): [\`references/output-format.md\`](references/output-format.md).`
- [ ] **Step 2:** 공통 검증 절차 실행 → 전부 통과
- [ ] **Step 3: 커밋**

```bash
git add dp-skills/skills/discover dp-skills/docs/reference
git commit -m "refactor(discover): 출력 템플릿을 references/ 로 분리"
```

---

### Task 5: agents/ag-planner.md 구조 재배치

분리가 아니라 **같은 파일 내 재배치** — 절차 후반 (9 critic 안내·10 재호출
분기) 이 거대한 중간 블록에 묻혀 누락되는 문제를 해소한다.

**Files:**

- Modify: `dp-skills/agents/ag-planner.md`
- Regen: `dp-skills/docs/reference/agents/ag-planner.md`

- [ ] **Step 1: 절차 개요 추가** — wrapper preamble blockquote 직후에
  `## 절차 개요` 섹션: 1~10 단계 한 줄씩 (제목만, 내용 중복 없음). 후반 단계
  (9 critic 안내 필수·10 재호출 분기) 가 첫 화면에 보이게 한다.
- [ ] **Step 2: 결함 수정 모드 블록 후방 이동** — step 1 내부의
  `**결함 수정 모드 (phase == qa).**` 문단 + 하위 bullets 전체를
  `## 결함 수정 모드 (phase == qa)` 섹션으로 verbatim 이동 (numbered 절차
  종료 직후·"플래닝 프로세스" 앞). step 1 의 phase 확인 문단은 유지하되
  활성화 지시만 남긴다:
  `\`qa\` 이면 아래 "결함 수정 모드 (phase == qa)" 섹션 블록을 활성화한다 (mode 와 직교 — …공존 가능).`
- [ ] **Step 3: 모드 가드 블록 후방 이동** — step 4 standard bullet 내부의
  `**[모드 가드 — 비TDD 에서 자동 테스트 요청 감지]**` 문단 + (A)~(D) 분기 +
  마무리 문단을 `## 모드 가드 — 비TDD 에서 자동 테스트 요청 감지` 섹션으로
  verbatim 이동 (결함 수정 모드 섹션 다음). step 4 에는 트리거 한 줄 유지:
  `standard 모드에서 자동 테스트 신호 감지 시, 아래 "모드 가드" 섹션의 분기를 먼저 수행 — 임의 진행 금지.`
- [ ] **Step 4:** diff 검토 — 이동 외 문구 변경 없음 확인 (활성화 분기 2줄 +
  개요 섹션만 신규)
- [ ] **Step 5:** 공통 검증 절차 실행 → 전부 통과
- [ ] **Step 6: 커밋**

```bash
git add dp-skills/agents/ag-planner.md dp-skills/docs/reference
git commit -m "refactor(agents): ag-planner 절차 개요 추가 + 모드 블록 후방 재배치"
```

---

### Task 6: 버전 bump + 최종 검증

**Files:**

- Modify: `dp-skills/.claude-plugin/plugin.json` (0.16.13 → 0.16.14)

- [ ] **Step 1:** plugin.json `"version": "0.16.14"` 로 수정
- [ ] **Step 2:** 최종 전체 검증

```bash
cd dp-skills
pytest tests/tools -q                # 460 passed
python3 tools/docs_build.py --check  # drift 없음
npx -y markdownlint-cli "skills/**/*.md" "agents/**/*.md" "README.md" ".claude-plugin/*.md"
python3 tools/doctor.py --schema
```

- [ ] **Step 3:** 분리 전후 줄수 재측정해 본 문서 기준선 표와 함께 PR 본문에 기록
- [ ] **Step 4: 커밋**

```bash
git add dp-skills/.claude-plugin/plugin.json
git commit -m "chore(release): bump 0.16.14"
```

---

### Task 7: PR 생성

- [ ] **Step 1:** push + PR (base `main`, squash 머지 전제)

```bash
git push -u origin feat/skill-progressive-disclosure
gh pr create --base main --title "refactor(skills): SKILL.md progressive disclosure (0.16.14)"
```

PR 본문: 분리 전후 줄수 표, 신규 references 목록, 동작 보존 제약 4건 준수
근거, 검증 결과 (pytest·docs_build·lint).
