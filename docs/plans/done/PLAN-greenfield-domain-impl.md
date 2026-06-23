# greenfield 도메인 즉석 등재 — 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** analyze 진입 중 미등재 도메인을 자유 입력하면 그 자리에서 MANIFEST 행 + scope 시드 + rules 스켈레톤을 생성하는 즉석 등재 경로를 만든다.

**Architecture:** 설계는 [PLAN-greenfield-domain.md](PLAN-greenfield-domain.md) (approved). 로직은 domain-decision prelude 인라인 (접근법 A) — 시드 형식은 신규 reference `greenfield-scope.md` 가 SSOT, MANIFEST 행 추가·rules 스켈레톤은 discover 의 기존 references 준용. 도구 코드 (`.py`) 변경 없음 — 전부 markdown 절차 문서.

**Tech Stack:** markdown 절차 문서 · `dp-skills/tools/docs_build.py` (reference 동기화·lint) · plugin.json semver.

**브랜치:** `feature/greenfield-domain` (`feature/knowledge-sync` 위 스택).

---

### Task 1: `greenfield-scope.md` 신규 reference 작성

**Files:**

- Create: `dp-skills/skills/analyze/references/greenfield-scope.md`

- [ ] **Step 1-1: 파일 생성** — 아래 전문으로 Write:

````markdown
# greenfield scope 시드 골격

미등재 신규 도메인을 analyze 진입 중 즉석 등재할 때 생성하는
`scope/{domain}.md` 시드의 형식·추출 규칙 SSOT. 절차 본문은
[domain-decision.md](domain-decision.md) § 신규 도메인 즉석 등재 — 본 파일을
참조만 하고 복붙하지 않는다 (drift 방지).

## 골격

```markdown
# {domain} 도메인 스코프

> [greenfield] 본 문서는 features 기반 **예정 항목** 시드다 (코드 미반영).
> 구현 진행 시 knowledge-sync 환류·discover 재실행이 코드 기준으로 대체한다.
> 아래 항목을 코드 사실로 간주하지 말 것.

## Routes

| Method | Path | 컨트롤러#액션 | 목적 |
| ------ | ---- | ------------- | ---- |

## Models

| Class | DB 테이블 | 목적 |
| ----- | --------- | ---- |

## Services

| Class | 파일 | 목적 |
| ----- | ---- | ---- |
```

## 형식 규칙

- H2 헤딩 (`## Routes`·`## Models`·`## Services`) 과 표 컬럼은 discover 의
  미리 보기 표 ([output-format.md](../../discover/references/output-format.md))
  와 **정확히 동일**하게 유지한다 — analyze 5-2 의 관련 파일 추출과
  planner/wrapper 의 scope 소비가 무수정 호환되는 전제.
- greenfield 표식은 상단 blockquote `[greenfield]` 마커 하나. 파일 전체가
  예정 항목이므로 행 단위 표시는 두지 않는다.
- 코드 경로 컬럼 (컨트롤러#액션·파일) 은 features 에 없으므로 `(미정)` 으로
  채운다.

## 시드 추출 규칙

- 소스는 **features 본문 명세** (`features/NN-{slug}.md`) 다 — docs/ 원문이
  아니다 (도메인 결정은 analyze 단계 5 의 전제로, features 분할이 완료된
  시점. create-feature 는 방금 생성된 feature 1 건이 소스).
- features 에 명시된 엔드포인트·모델·서비스 언급만 표 행으로 변환한다.
  **추측 금지** — discover 의 원칙과 동일.
- 추출 항목이 0 건이면 빈 표 + 마커만 남긴다 (별도 분기 아님).

## 마커 해소 (합류)

- `/dp-skills:discover {entry} {domain}` 재실행 — scope 단순 덮어쓰기로 마커
  자연 소멸.
- knowledge-sync 환류 — 구현 확인된 항목을 실제 값으로 갱신, 전 항목 코드
  반영 시 마커 블록 제거
  ([knowledge-sync.md](../../context/lifecycle/knowledge-sync.md) § 기록).
````

- [ ] **Step 1-2: 링크 대상 존재 확인**

Run: `ls dp-skills/skills/discover/references/output-format.md dp-skills/skills/context/lifecycle/knowledge-sync.md dp-skills/skills/analyze/references/domain-decision.md`
Expected: 3개 경로 모두 출력 (no such file 없음)

- [ ] **Step 1-3: Commit**

```bash
git add dp-skills/skills/analyze/references/greenfield-scope.md
git commit -m "feat(analyze): greenfield scope 시드 골격 reference 신설"
```

---

### Task 2: `domain-decision.md` (c) 즉석 등재 절차로 확장

**Files:**

- Modify: `dp-skills/skills/analyze/references/domain-decision.md:15-27`

- [ ] **Step 2-1: (c) 질의 블록 교체** — 변경 전:

```markdown
   - (c) 사용자에게 질의:

     ```
     이 프로젝트의 도메인을 확인해주세요.
     자동 판정 후보: {후보} (근거: {근거 한 줄})
     선택지: {workspace/{TEAM}/context/scope/*.md 파일명을 / 로 나열}
     ```

     선택지 목록은 `workspace/{TEAM}/context/scope/` 폴더의 `*.md` 파일명에서 동적으로 읽는다 (`.md` 제외). 폴더가 비어 있으면 사용자에게 자유 입력을 요청하고, MANIFEST 등록 후 재진입을 안내.

   - 사용자 응답을 `.agent-state.yml` 의 `domain` 필드에 Edit 로 기록.
3. 결정된 도메인으로 `workspace/{TEAM}/context/scope/{domain}.md` 를 Read.
```

변경 후:

```markdown
   - (c) 사용자에게 질의:

     ```
     이 프로젝트의 도메인을 확인해주세요.
     자동 판정 후보: {후보} (근거: {근거 한 줄})
     선택지: {workspace/{TEAM}/context/scope/*.md 파일명을 / 로 나열} / 신규 도메인 등재
     ```

     선택지 목록은 `workspace/{TEAM}/context/scope/` 폴더의 `*.md` 파일명에서 동적으로 읽는다 (`.md` 제외). 폴더가 비어 있으면 선택지 질의를 생략하고 곧장 아래 § 신규 도메인 즉석 등재 로 진행한다 (자유 입력). 입력값이 선택지에 없는 신규 도메인일 때도 같은 절로 진행하되, 기존 도메인과 이름이 유사하면 오타 방지 확인을 1 줄 거친다 — "기존 `{유사후보}` 와 다른 신규 도메인입니까?".

   - 사용자 응답을 `.agent-state.yml` 의 `domain` 필드에 Edit 로 기록.
3. 결정된 도메인으로 `workspace/{TEAM}/context/scope/{domain}.md` 를 Read.
```

- [ ] **Step 2-2: 파일 말미에 즉석 등재 섹션 추가** — 다음 전문을 append:

````markdown

---

## 신규 도메인 즉석 등재 (greenfield)

위 (c) 에서 미등재 도메인을 받은 경우 analyze 를 중단하지 않고 그 자리에서
등재한다. 코드 진입점이 없는 greenfield 가 주 대상 — 코드가 이미 있는
도메인이었다면 등재 후 결과 출력에 `/dp-skills:discover {entry} {domain}`
재실행을 안내한다 (시드를 코드 기준으로 대체).

1. **식별자 검증** — discover 와 동일 규칙: 영숫자·하이픈·언더스코어·계층
   `/` 만 허용, 소문자화, 연속 하이픈은 단일로 축약, `..`·절대 경로 거부.
2. **시드 추출** — features 본문 명세에서 예정 Routes/Models/Services 를
   추출한다. 골격·추출 규칙: [greenfield-scope.md](greenfield-scope.md).
3. **미리 보기 + 승인** — 거부 시 아무것도 쓰지 않고 (c) 질의로 복귀한다
   (discover 단계 3 의 승인 게이트·Abort cleanup 계약과 동일):

   ```
   신규 도메인 등재 미리 보기 (domain: {domain})

   {시드 Routes/Models/Services 표}

   기록할 파일:
     workspace/{TEAM}/context/scope/{domain}.md    created (greenfield 시드)
     workspace/{TEAM}/context/rules/{domain}.md    {created (빈 스켈레톤)|skipped (이미 존재)}
     workspace/{TEAM}/context/MANIFEST.md          (도메인 행 추가·갱신)

   계속할까요? (y/n)
   ```

4. **기록** — 순서: `scope/{domain}.md` 시드 Write → `rules/{domain}.md` 빈
   스켈레톤 (**부재 시에만** — discover
   [output-format.md](../../discover/references/output-format.md) 템플릿 준용)
   → MANIFEST `## 도메인 분류` 행 추가
   ([manifest-update.md](../../discover/references/manifest-update.md)
   detect + conform 준용 — MANIFEST 에 행만 있고 scope 가 없던 비정합도 같은
   경로로 해소된다). 계층 식별자면 `scope/{part}/{domain}.md` (discover 와
   동일).

등재 완료 후 위 절차 2 말미로 합류한다 — 사용자 응답 (신규 도메인명) 을
`.agent-state.yml` 의 `domain` 에 기록 → scope Read. 등재가 거부·중단되면
`domain` 은 기록하지 않는다 (오염 방지).
````

- [ ] **Step 2-3: 검증** — 헤딩·링크 확인

Run: `grep -n "신규 도메인 즉석 등재\|greenfield-scope.md" dp-skills/skills/analyze/references/domain-decision.md`
Expected: § 헤딩 1건 + (c) 본문·섹션 내 greenfield-scope.md 링크 출력

- [ ] **Step 2-4: Commit**

```bash
git add dp-skills/skills/analyze/references/domain-decision.md
git commit -m "feat(analyze): domain-decision 에 신규 도메인 즉석 등재 절차"
```

---

### Task 3: `analyze/SKILL.md` · `create-feature/SKILL.md` 요약 한 줄

**Files:**

- Modify: `dp-skills/skills/analyze/SKILL.md:133`
- Modify: `dp-skills/skills/create-feature/SKILL.md:105`

- [ ] **Step 3-1: analyze/SKILL.md 133행 문단 끝에 한 문장 추가** — 변경 전 (문단 끝부분):

```markdown
자동 판정은 **후보 제시용** 일 뿐 항상 사용자 확인을 거친다 (한 번 잘못 기록되면 후속 분석이 전부 오염되어 features 전체 재분석이 필요).
```

변경 후:

```markdown
자동 판정은 **후보 제시용** 일 뿐 항상 사용자 확인을 거친다 (한 번 잘못 기록되면 후속 분석이 전부 오염되어 features 전체 재분석이 필요). 미등재 신규 도메인 (greenfield) 은 그 자리에서 즉석 등재한다 — MANIFEST 행 + scope 시드 + rules 스켈레톤 ([`references/domain-decision.md`](references/domain-decision.md) § 신규 도메인 즉석 등재).
```

- [ ] **Step 3-2: create-feature/SKILL.md 105행 항목 끝에 한 문장 추가** — 변경 전:

```markdown
- **도메인 결정** (5 단계 prelude). `.agent-state.yml.domain` 이 null 이면 analyze 와 동일한 우선순위로 후보 제시 후 사용자 확인 → state 에 기록. non-null 이면 그대로 사용 (재질의 금지).
```

변경 후:

```markdown
- **도메인 결정** (5 단계 prelude). `.agent-state.yml.domain` 이 null 이면 analyze 와 동일한 우선순위로 후보 제시 후 사용자 확인 → state 에 기록. non-null 이면 그대로 사용 (재질의 금지). 미등재 신규 도메인이면 domain-decision 의 즉석 등재 절차를 동일 준용한다 (시드 소스는 방금 생성된 feature 1 건).
```

- [ ] **Step 3-3: Commit**

```bash
git add dp-skills/skills/analyze/SKILL.md dp-skills/skills/create-feature/SKILL.md
git commit -m "docs(skills): analyze·create-feature 도메인 결정에 즉석 등재 명시"
```

---

### Task 4: lifecycle 문서 합류 규칙 (state-schema · knowledge-sync)

**Files:**

- Modify: `dp-skills/skills/context/lifecycle/state-schema.md:90`
- Modify: `dp-skills/skills/context/lifecycle/knowledge-sync.md:98-100`

- [ ] **Step 4-1: state-schema.md § domain 생명주기 항목 2 끝에 한 문장 추가** — 변경 전:

```markdown
2. **첫 기록** — `/dp-skills:analyze` 또는 `/dp-skills:create-feature` 진입 시 도메인 결정 prelude ([domain-decision.md](../../analyze/references/domain-decision.md)) 가 후보를 제시하고 **사용자 확인 후** 기록한다.
```

변경 후:

```markdown
2. **첫 기록** — `/dp-skills:analyze` 또는 `/dp-skills:create-feature` 진입 시 도메인 결정 prelude ([domain-decision.md](../../analyze/references/domain-decision.md)) 가 후보를 제시하고 **사용자 확인 후** 기록한다. 미등재 신규 도메인이면 prelude 가 즉석 등재 (MANIFEST 행·scope 시드·rules 스켈레톤, 승인 게이트 후) 를 먼저 수행하고 기록한다 — greenfield 도 동일 경로.
```

- [ ] **Step 4-2: knowledge-sync.md § 기록 항목 2 에 greenfield 마커 처리 추가** — 변경 전:

```markdown
2. Edit 으로 반영. scope 구조·200줄 제약은
   [split-policy](../../discover/references/split-policy.md), MANIFEST 갱신은
   [manifest-update](../../discover/references/manifest-update.md) 를 따른다.
```

변경 후:

```markdown
2. Edit 으로 반영. scope 구조·200줄 제약은
   [split-policy](../../discover/references/split-policy.md), MANIFEST 갱신은
   [manifest-update](../../discover/references/manifest-update.md) 를 따른다.
   scope 상단에 `[greenfield]` 마커
   ([greenfield-scope](../../analyze/references/greenfield-scope.md) 시드) 가
   있으면 구현 확인된 항목을 실제 값으로 갱신하고, 전 항목이 코드 반영되면
   마커 블록을 제거한다.
```

- [ ] **Step 4-3: Commit**

```bash
git add dp-skills/skills/context/lifecycle/state-schema.md dp-skills/skills/context/lifecycle/knowledge-sync.md
git commit -m "docs(lifecycle): domain 생명주기·환류 기록에 greenfield 합류 규칙"
```

---

### Task 5: 매뉴얼 2건 (how-to)

**Files:**

- Modify: `dp-skills/docs/how-to/analyze-docs.md:32` (§1 분석 실행 끝)
- Modify: `dp-skills/docs/how-to/domain-discover.md:11` (§ 언제 쓰나 끝)

- [ ] **Step 5-1: analyze-docs.md §1 끝 (키워드 예시 코드블록 다음) 에 단락 추가:**

```markdown

#### 신규 도메인이면

도메인 확인 질의에서 미등재 도메인을 입력하면 analyze 를 중단하지 않고 그
자리에서 등재까지 진행됩니다 — 승인 후 MANIFEST 행·`scope/{domain}.md` 시드
(features 기반 예정 항목)·`rules/{domain}.md` 빈 스켈레톤이 생성됩니다. 코드
진입점이 아직 없는 신규 서비스 (greenfield) 도 같은 경로입니다. 코드가 쌓이면
[도메인 스코프 추출](domain-discover.md) 재실행이 시드를 코드 기준으로
대체합니다.
```

- [ ] **Step 5-2: domain-discover.md § 언제 쓰나 끝 (scope 파일 설명 문단 다음) 에 note 추가:**

```markdown

!!! note "코드가 아직 없다면 (greenfield)"
    discover 는 코드 전제입니다 — 개발 전 신규 도메인은 `/dp-skills:analyze`
    진입 중 도메인 질의에서 즉석 등재됩니다 ([기획서로 features 일괄
    생성](analyze-docs.md) 참고). 코드가 쌓인 뒤 discover 를 실행하면 시드가
    코드 기준으로 대체됩니다.
```

- [ ] **Step 5-3: Commit**

```bash
git add dp-skills/docs/how-to/analyze-docs.md dp-skills/docs/how-to/domain-discover.md
git commit -m "docs(manual): greenfield 즉석 등재 경로 안내 (analyze·discover)"
```

---

### Task 6: reference 동기화 + 버전 bump + 최종 검증

**Files:**

- Modify: `dp-skills/.claude-plugin/plugin.json` (`"version": "0.18.0"` → `"0.19.0"`)
- 재생성 가능: `dp-skills/docs/reference/**` (docs_build.py 산출)

- [ ] **Step 6-1: docs reference 동기화·lint**

Run: `python3 dp-skills/tools/docs_build.py`
Expected: 정상 종료 (exit 0). `git status` 에 `docs/reference/**` drift 가 생기면 그대로 staged 대상에 포함 (SKILL.md 변경분 반영 — 정상).

- [ ] **Step 6-2: drift 발생 시 커밋**

```bash
git add dp-skills/docs/reference
git commit -m "docs(reference): analyze·create-feature 변경 반영 재생성"
```

(drift 없으면 skip.)

- [ ] **Step 6-3: plugin.json bump** — `"version": "0.18.0"` 을 `"version": "0.19.0"` 으로 Edit 후:

```bash
git add dp-skills/.claude-plugin/plugin.json
git commit -m "chore(release): bump 0.19.0"
```

- [ ] **Step 6-4: 최종 검증 3종**

1. 링크 정합: `grep -rn "greenfield-scope.md" dp-skills/skills/ | wc -l` → 2 이상 (domain-decision 1곳·knowledge-sync 1곳)
2. 수기 시나리오 리뷰 (문서 워크스루): domain-decision (c) → § 즉석 등재 → 합류 흐름을 처음부터 읽어 ① 빈 scope 폴더, ② 선택지 밖 입력, ③ 승인 거부 3 케이스가 모두 막힘 없이 라우팅되는지 확인
3. `git log --oneline feature/knowledge-sync..HEAD` → Task 1~6 커밋 + 설계 문서 커밋 확인

---

## Self-check (스펙 대비 커버리지)

| 스펙 (PLAN-greenfield-domain.md) | Task |
| --- | --- |
| §2 플로우 5단계 | Task 2 |
| §3 시드 골격·추출 규칙 | Task 1 |
| §4 합류 규칙 (knowledge-sync·discover·doctor) | Task 4 (discover 덮어쓰기·doctor 는 기존 동작 — 무변경) |
| §5 변경 파일 8건 | Task 1~6 (8건 전부) |
| §6 엣지 5건 | Task 2 (유사 이름·비정합·계층·거부·0건은 Task 1) |
| §7 검증 | Task 6-4 (doctor 는 MANIFEST↔scope 동시 생성으로 정합 자명 — 수기 워크스루로 대체) |

## 실행 중 보정 (Task 2 품질 리뷰 반영)

c9afc28 품질 리뷰 지적을 후속 커밋에서 보정했다 — Task 2 본문 블록과 실제
파일이 다른 부분은 본 절이 우선한다:

- (c) 빈 폴더 경로에 도메인명 자유 입력 질의를 명시 (state-schema "질의 후
  기록" 원칙 정합).
- (c) 선택지 열거에 계층 식별자 포함 + § 즉석 등재 1 에 기존 scope 존재 가드
  (학습된 scope 를 시드로 덮어쓰는 경로 차단)·MANIFEST 부재 시 team-init 안내.
- 식별자 규칙에 `_`/`.` 선행 문자 거부 추가, 합류 문구를 "검증·정규화된
  식별자" 기록으로 교정, 유사 이름 확인의 "아니오" 분기 명시.
- discover 안내 문구 재작성 (판단 주체 사용자) + `discover/SKILL.md` ## 참고
  에 greenfield 시드 생산자 한 줄 (변경 파일 +1).

### Task 1·3·4·5 품질 리뷰 Minor 폴리시 (일괄 1 커밋)

- greenfield-scope 형식 규칙의 출처 분리 (표 컬럼 ↔ output-format, H2 헤딩 ↔ analyze 5-2·split-policy) 와 마커 제거 완료 기준 (`(미정)` 잔존 없음) 명시.
- create-feature SKILL 의 domain-decision 직접 링크, state-schema 교정 항목 (생명주기 5) 에 즉석 등재 산출물 정리 추가.
- 매뉴얼: rules 스켈레톤 조건형 표기 (analyze-docs), create-feature 동일 경로 언급 (domain-discover).
