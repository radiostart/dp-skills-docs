# `/dp-skills:create-feature`

> 활성 프로젝트에 사용자 프롬프트(원하는 기능 서술)로 단일 feature 명세를 추가할 때 사용한다. features/NN-{slug}.md 를 prompt-origin 템플릿으로 생성하고 `/dp-skills:analyze` 와 동일하게 project.md (목표·관련 파일) 와 agents/* (planner·generator·evaluator) 를 함께 동기화한다. 기획서(docs/) 기반 다건 분할은 `/dp-skills:analyze` 를 사용한다. 실행은 @ag-planner 호출로 시작 — 자동 파이프라인 아님.

활성 프로젝트에 **단일 기능** 을 프롬프트로 추가한다. `features/` 폴더에 명세 파일을 생성하고 `project.md` (목표·관련 파일) 와 `agents/*.md` 를 `/dp-skills:analyze` 와 동일한 절차로 동기화한다. 구현 흐름은 `@ag-planner` 호출로 시작.

대상: $ARGUMENTS (기능 지시문 — 예: "지연 주문 UI 정렬 기능 — 기본 내림차순, 출고일 필터")

**사용 예:**

```
/dp-skills:create-feature 지연 주문 UI 에 정렬·필터 추가
/dp-skills:create-feature 알림 발송 기능 — Sidekiq 큐 사용
```

---

## 사전 확인

[preamble.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/preamble.md) 의 **P1, P2** 수행.

- P1, P2: `{TEAM}`, `{PROJECT}` 획득. 실패 시 [messages.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/messages.md) 의 `team_missing` / `no_active_project` 출력 후 종료.

`$ARGUMENTS` 가 비어있으면 **"기능 지시문을 입력하세요. 예: `/dp-skills:create-feature 정렬 기능 추가`"** 안내 후 종료.

### phase 가드 (사전 확인 직후)

`workspace/{TEAM}/projects/{PROJECT}/.agent-state.yml` 를 Read 하여 `phase` 필드를 확인한다. `state-schema.md` 규칙에 따라 `phase` 필드 부재는 `development` 로 간주한다.

- `phase == qa` → 다음 메시지 출력 후 종료:

  ```
  QA phase 중에는 신규 feature 추가가 차단됩니다. development 로 복귀하려면 /dp-skills:qa off (또는 /dp-skills:project {PROJECT} --qa off) 실행 후 재시도하세요.
  ```

- `phase == development` 또는 `phase` 필드 부재 → 진행 (기존 흐름 유지).

`.agent-state.yml` 자체가 없는 경우는 가드 미적용 (legacy/신규 폴더 — 본 스킬이 차후 단계에서 생성/갱신).

---

## 동작

### 1. 기능명·slug·번호 결정

프롬프트에서 추출:

- **기능명** — 지시문의 핵심 표현 (한국어 허용, 30자 이내)
- **slug** — 기능명을 kebab-case 로 변환 (영문·한글 혼합 허용, 특수문자 제거, 최대 30 자)
- **NN** — `workspace/{TEAM}/projects/{PROJECT}/features/*.md` 의 `.plan.md` 제외 번호 중 최댓값 + 1. 폴더 없으면 `01` 부터.

예:

```
지시문: "지연 주문 UI 에 정렬·필터 추가"
→ 기능명: "지연 주문 UI 정렬·필터"
→ slug: "delayed-order-sort-filter" (또는 "지연-주문-정렬-필터")
→ NN: 06 (기존 features 5 개인 경우)
```

slug 결정이 모호하면 사용자에게 후보 2-3 개 제시 후 선택받는다.

### 2. features/ 폴더 확인·생성

`workspace/{TEAM}/projects/{PROJECT}/features/` 가 없으면 생성.

### 3. `features/NN-{slug}.md` 작성

prompt-origin 템플릿으로 Write — 전체 템플릿 (`> source: prompt` 메타 + 요구사항·상태 전환·비즈니스 규칙·예외 케이스 + Open Questions 4 카테고리): [`references/feature-template.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/create-feature/references/feature-template.md).

프롬프트에서 명시적으로 추출 가능한 요소 (상태 값·정렬 기준·트리거 등) 는 해당 섹션에 채워 넣는다. 추측성 내용은 **넣지 않는다** — placeholder 로 둠.

**Open Questions 작성 규칙:**

- 4 카테고리 헤더는 항상 모두 포함한다. 카테고리 자체를 생략하지 않는다.
- 한 카테고리에 질문이 없으면 `- (없음)` 으로 표시 (작성자가 "정말 없는지" 의식적 확인 강제).
- 산출물 lookup 시 답할 수 없는 영역이 발견되면 아래 3-bis 의 분류 기준에 따라 해당 카테고리에 `- [ ] {질문}: ...` 행을 추가한다.
- A2 runtime fallback: detect 알고리즘 실패 시 → 4 카테고리 헤더 + `- (없음)` placeholder 만 작성. abort 안 함.

### 3-bis. cross-domain 의존성 detect + Open Questions 분류

feature spec 작성 후, MANIFEST 를 조회해 cross-domain 의존성을 감지하고 결과를 Open Questions 4 카테고리로 분류한다. detect 절차 (MANIFEST lookup·행 추가·INFO 출력 형식)·카테고리 분류 기준: [`references/open-questions.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/create-feature/references/open-questions.md).

> **A2 runtime fallback**: MANIFEST lookup 실패 또는 키워드 추출 실패 시 → spec 진행 (abort 안 함). Open Questions 에는 4 카테고리 헤더 + `- (없음)` placeholder 만 기입. INFO 출력 안 함.
>
> 의도된 비대칭 — MANIFEST 게이트는 `project`/`issue` 진입 (preamble P4) 이 이미 수행했고, 여기서 MANIFEST 는 cross-domain detect enrichment 용도다. 활성 프로젝트 안에서 부재가 발견되면 (사후 드리프트) 작성 중인 feature spec 을 버리지 않고 detect 만 생략한다 ([INDEX.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/INDEX.md) § MANIFEST 부재 정책).

> **편집물 보존 가드** — 신규 `features/NN-{slug}.md` 는 Write (파일 없음). 기존 파일 (`project.md`·`agents/*.md`·`.agent-state.yml`·다른 `features/*.md`) 은 **Edit** 으로 splice. PreToolUse 훅 `protect-managed.sh` 가 강제. NN 충돌 시 다음 번호로 재할당.

### 4. `project.md` 및 `agents/*` 자동 갱신

신규 feature 파일 생성 후 `project.md` (목표·관련 파일) 와 `agents/*` 를 **현재 features/ 전체** 기준으로 갱신한다. 항목별 상세는 아래 각 reference 를 직접 따른다 — `analyze/SKILL.md` 는 읽지 않는다 (analyze 와 동일 절차이며 소스만 docs/ 가 아닌 features/ 전체).

수행 항목:

- **도메인 결정** (5 단계 prelude). `.agent-state.yml.domain` 이 null 이면 analyze 와 동일한 우선순위로 후보 제시 후 사용자 확인 → state 에 기록. non-null 이면 그대로 사용 (재질의 금지). 미등재 신규 도메인이면 [domain-decision](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/analyze/references/domain-decision.md) 의 즉석 등재 절차를 동일 준용한다 (시드 소스는 방금 생성된 feature 1 건).
- **5-1. `## 목표` 갱신** — features 전체로 체크리스트 정렬 갱신. 기존 `[x]` 체크는 보존, 신규 feature 항목이 NN 순서에 추가된다. 갱신 예시: [`project-md-update.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/analyze/references/project-md-update.md).
- **5-2. `## 관련 파일` 갱신** — `scope/{domain}.md` 의 Routes/Models/Services 표를 추출해 `## 관련 파일` 표 자동 기입. 표 기입·갱신 규칙: [`project-md-update.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/analyze/references/project-md-update.md).
- **6-1 ~ 6-3. agents/\* 갱신** — `[analyze-managed]` 섹션을 features 전체 + scope 매칭 결과로 regen. `[analyze-managed]` 밖 사용자 수동 편집 영역 (`## 주의사항`·`## 구현 패턴` 등) 과 evaluator 의 `[x]` 체크는 보존. 상세 절차: [`agents-update.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/analyze/references/agents-update.md).
- **6-4. `.agent-state.yml` 갱신** — `analyzed: true`, `analyzed_at`, `last_analyzed_features` 기록. domain 이 이번에 처음 결정됐다면 함께 기록.

**보존 규칙 요약** (analyze 와 동일):

- 기존 features 의 `### {기능명}` 블록은 그대로 유지되고 신규 feature 의 블록만 추가된다 (NN 순).
- 첫 feature 추가 시 (`analyzed: false`) 는 example 템플릿 placeholder 가 features 기반 실내용으로 교체된다 (보존할 게 없음).
- `[analyze-managed]` 안에 사용자가 수동으로 끼워 넣은 내용은 덮어써진다 (analyze 와 동일한 트레이드오프).

### 5. 무결성 검증 (자동)

무결성 자동 검증 — 아래를 실행한다:

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/tools/doctor.py workspace
```

ERROR 또는 WARN 있으면 원문을 사용자에게 그대로 출력. 모두 PASS 면 `doctor: all checks passed` 한 줄만.

### 6. 결과 요약

```
Feature 생성 완료: #{NN} {기능명}

생성:
  - features/{NN}-{slug}.md  (prompt-origin 템플릿)

갱신:
  - project.md           (목표 +1, 관련 파일 동기화)
  - agents/planner.md    (기능별 사전 확인 사항)
  - agents/generator.md  (기술 레퍼런스)
  - agents/evaluator.md  (체크리스트)
  - .agent-state.yml     (analyzed: true)

검증: {doctor 결과 한 줄 요약}

다음 단계:
→ 명세가 부족하면 features/{NN}-{slug}.md 직접 편집
→ `@ag-planner` 를 호출해 구현 계획을 수립하세요.
```

---

## 제약

- **이 스킬은 에이전트를 자동 호출하지 않는다.** Planner → Generator → Evaluator 흐름은 사용자가 각각 명시 호출. feature 의 시작점은 `@ag-planner`.
- **prompt-origin feature 는 `> source: prompt`** 로 표시된다. `/dp-skills:analyze --force` 실행 시 이 tag 를 감지해 사용자에게 덮어쓰기 여부를 확인한다 (의도되지 않은 데이터 손실 방지).
- **docs 기반 feature 생성은 `/dp-skills:analyze`** 사용. 이 스킬은 docs 없이 단건 추가 용도.

---

## 참고

- `/dp-skills:analyze` — docs/ 기획서 벌크 분석
- `/dp-skills:focus` — ad-hoc 사용자 지시 기록 (features/ 밖)
- `@ag-planner` — features/{NN}-{slug}.md 를 읽어 구현 계획 수립
