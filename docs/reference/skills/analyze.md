# `/dp-skills:analyze`

> 이미 저장된 docs/ 기획서를 features/ 기능 명세로 분할·구조화할 때 사용한다. PM 작성 표 중심 기획서를 기능 단위 문서로 변환하고 project.md 의 목표 섹션 과 agents/ 파일(planner·generator·evaluator)을 자동 갱신한다. 기획서 fetch 는 `/dp-skills:confl`, 프롬프트 기반 단일 기능 추가는 `/dp-skills:create-feature` 를 사용한다.

## 역할

구현 의도를 정확히 읽는 분석가. PM 문서의 모호한 표현을 추측으로 채우지 않고, 원문에 명시된 것만 feature 로 분리한다. 불확실한 영역은 Open Questions 로 남긴다.

---

# /dp-skills:analyze

Confluence 기획서(docs/)를 분석하여 기능별 구조화된 명세(features/)를 생성한다.
PM이 작성한 표 중심 기획서를 AI가 읽기 쉬운 형태로 변환한다.

대상: $ARGUMENTS

---

## 사전 확인

[preamble.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/preamble.md) 의 **P-1, P0, P1, P2** 수행.

- P-1: TodoWrite 선로딩 (다단계 스킬).
- P0: 컨텍스트의 `MEMORY.md` 인덱스에서 `{PROJECT}`·`$ARGUMENTS` 키워드와 관련된 메모를 가려 본문을 Read 하여 과거 분석·구현 이력 확인.
- P1, P2: `{TEAM}`, `{PROJECT}` 획득. 실패 시 [messages.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/messages.md) 의 `team_missing` / `no_active_project` 출력 후 종료.

추가: `workspace/{TEAM}/projects/{PROJECT}/docs/` 폴더를 Glob 으로 확인한다.

- 폴더가 없거나 `.md` 파일이 없으면 [messages.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/messages.md) 의 `docs_missing` 출력 후 종료.

---

## 인자 판별

`$ARGUMENTS`에서 `--force` · `--regen-agents` 플래그를 먼저 분리하고, 나머지 텍스트로 모드를 결정한다.

| 플래그 / 나머지 텍스트 | 모드 | 동작 |
| ---------------------- | ---- | ---- |
| `--regen-agents` (단독) | **재생성 전용** | docs/features 변화 여부와 무관하게 현재 features/ 기반으로 `agents/*.md` 만 재작성. 상세: [`references/regen-mode.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/analyze/references/regen-mode.md) |
| 없음 (빈 문자열) | 전체 분석 | docs/ 내 **모든** 원본 파일의 전체 내용 분석 |
| 파일명 또는 page_id | 파일 지정 | 해당 파일만 분석 |
| 그 외 텍스트 (키워드) | 필터 분석 | docs/ 전체 파일에서 **키워드 관련 기능만** 추출하여 분석 |

### 필터 분석 모드

기획서에는 프론트엔드, API, 관리자 등 여러 영역의 기능이 혼재되어 있다.
키워드가 주어지면 docs/ 전체 파일을 읽되, **키워드와 관련된 섹션만** 추출하여 features/ 파일을 생성한다.

- 예: `/dp-skills:analyze 관리자 기능` → 관리자(어드민) 관련 기능만 분석
- 예: `/dp-skills:analyze 결제 흐름` → 결제 관련 기능만 분석
- 키워드 매칭은 섹션 제목과 내용 모두에서 판단한다.
- 관련 없는 섹션은 스킵한다.

---

## 분석 프로세스

### 1. 대상 파일 결정

- `workspace/{TEAM}/projects/{PROJECT}/docs/*.md` 에서 원본 파일 목록을 수집한다.
- `--force` 가 없으면: `features/` 폴더에 이미 분석 파일이 존재하는 원본은 스킵한다.
  - 스킵 판단: features/ 파일 상단 `> source:` 메타데이터에 원본 파일명이 기록됨
- 분석 대상이 없으면 [messages.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/messages.md) 의 `analyze_all_done` 출력 후 종료한다.

#### `--force` 실행 시 prompt-origin 보호

`--force` 는 기존 features/ 파일을 덮어쓸 수 있다. 그 전에 `/dp-skills:create-feature` 로 생성된 **prompt-origin features** 가 있는지 확인하고 사용자에게 승인받는다 (자동 진행 시 사용자 의도로 만든 기능 명세가 소리 없이 사라져 데이터 손실로 직결되기 때문):

1. `features/*.md` 를 Grep 하여 `> source: prompt` 태그가 있는 파일 목록 수집
2. 1 건 이상 있으면 사용자에게 경고 + 승인 대기:

   ```
   ⚠ --force 재분석이 prompt-origin features 를 덮어쓸 가능성이 있습니다:
     - features/05-refund-policy.md (source: prompt)
     - features/07-notification-queue.md (source: prompt)

   이 파일들은 /dp-skills:create-feature 로 생성됐으며, docs/ 에 대응 원본이
   없습니다. --force 진행 시 slug 충돌이 발생하면 덮어쓰여 데이터가 손실될
   수 있습니다. 계속? (y/n)
   ```

3. 사용자 `n` → 종료. `y` → 진행. 명시 승인 없이 자동 진행하지 않는다.

0 건이면 이 절차를 건너뛴다.

### 2. 원본 파일 읽기

- 대상 파일을 Read 툴로 로드한다.
- **default path** — 파일 크기·H2 목차 파악 → 관심 섹션 targeted Read. 파일 크기별 limit 분기 및 재시도 규칙은 [`references/read-strategy.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/analyze/references/read-strategy.md).
- **추측 금지 원칙**: 읽지 않은 섹션은 features/ 생성 대상에서 제외. 전체 스캔하지 않았다면 **사용자에게 범위 보고** 후 확정. 사용자가 "전체 분석" 을 요청했고 파일이 대형이면 H2 목차 기반으로 모든 섹션을 순회한다.

### 3. 기능 분할 및 구조화

원본 문서를 아래 기준으로 기능 단위로 분할한다:

**분할 기준:**

- H2(`##`) 섹션을 기능 단위로 인식
- 번호 패턴(`#N`, `N.`, `N)`)이 있으면 기능 번호로 사용
- 번호가 없으면 순차 번호 부여

**각 기능에서 추출할 항목** — features 파일 템플릿 (요구사항 3축·상태 전환·비즈니스 규칙·예외 케이스 구조): [`references/feature-template.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/analyze/references/feature-template.md).

**분석 시 주의사항:**

- 원본의 표(table)는 의미를 해석하여 서술형으로 풀어쓰되, 상태 전환표처럼 표 형태가 더 명확한 경우는 표를 유지한다.
- 원본에 없는 내용을 추측하여 추가하지 않는다 (한 번 추측이 들어가면 후속 6 단계의 agents/ 갱신·planner·generator 가 모두 잘못된 사실을 기반으로 동작한다).
- Figma 링크 등 디자인 참조는 유지한다.
- 하나의 H2 섹션이 여러 기능을 포함하면 기능별로 분리한다.

### 4. 파일 저장

- 저장 경로: `workspace/{TEAM}/projects/{PROJECT}/features/`
- 기능 명세 파일명: `{NN}-{slug}.md`
  - `NN`: 기능 번호 (2자리 zero-padding, 예: `12`, `13`)
  - `slug`: 기능명을 kebab-case로 변환 (한글 허용, 특수문자 제거, 최대 30자)
  - 예: `13-order-modal.md`, `19-receipt-list.md`
- **배치 저장 (병렬 Write)** — features 파일은 서로 독립이므로, 동일 assistant turn 안에 여러 Write tool_use 를 묶어 호출한다. harness 가 병렬 실행한다. 실무상 **3~5 개 단위 묶음** 이 현실적 (파일당 사고 비용 > I/O 비용). 10 개 이상을 한 번에 묶으면 컨텍스트가 혼잡해지므로 분할 권장.

### 5. project.md 자동 갱신

features/ 생성 후 `project.md` 의 `## 목표` 와 `## 관련 파일` 을 자동 동기화한다.

**도메인 결정 (5-1, 5-2 공통 전제):**

우선순위: `.agent-state.yml.domain` (non-null 이면 사용) → `project.md` 제한사항 `- domain: {x}` 라인 → MANIFEST 도메인 분류 표 매칭 → 사용자 질의. 사용자 응답은 `.agent-state.yml.domain` 에 Edit 로 기록. 결정된 도메인으로 `scope/{domain}.md` 를 Read. 자동 판정은 **후보 제시용** 일 뿐 항상 사용자 확인을 거친다 (한 번 잘못 기록되면 후속 분석이 전부 오염되어 features 전체 재분석이 필요). 미등재 신규 도메인 (greenfield) 은 그 자리에서 즉석 등재한다 — MANIFEST 행 + scope 시드 + rules 스켈레톤 ([`references/domain-decision.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/analyze/references/domain-decision.md) § 신규 도메인 즉석 등재).

상세 절차·질의 포맷: [`references/domain-decision.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/analyze/references/domain-decision.md).

#### 5-1. `## 목표` 갱신

1. `workspace/{TEAM}/projects/{PROJECT}/project.md`를 Read한다.
2. `## 목표` 섹션을 찾는다 (없으면 `## 에이전트 호출 흐름` 바로 앞에 생성한다).
3. 생성된 features/ 파일 목록을 기준으로 목표 체크리스트를 갱신한다:
   - **신규 feature:** `- [ ] {기능명} -> [상세](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/analyze/features/{NN}-{slug}.md)` 항목을 추가한다.
   - **기존 항목 유지:** 이미 `[x]`로 완료 처리된 항목은 변경하지 않는다 (사용자가 이미 끝낸 일을 미완료로 되돌리면 진행 상황이 거꾸로 보임).
   - **삭제된 feature:** `--force` 재분석으로 features/ 파일이 교체된 경우, 기존 항목 중 대응하는 features/ 파일이 없는 항목은 제거한다.
4. 목표 항목 순서는 features/ 파일의 번호(NN) 순서를 따른다.

갱신 예시: [`references/project-md-update.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/analyze/references/project-md-update.md).

#### 5-2. `## 관련 파일` 갱신

로드한 `scope/{domain}.md` 의 `## Routes`, `## Models`, `## Services` 표를 추출해 project.md 의 `## 관련 파일` 표를 자동 기입한다.

갱신 시 features/ 에 명시된 모델·서비스·라우트는 누락 금지, 기존 사용자 수동 기입 행은 보존. 표 기입 프로세스 (Endpoints/Models/Services)·갱신 규칙 상세: [`references/project-md-update.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/analyze/references/project-md-update.md).

**cross-domain 의존성 detect:**

features/ 키워드와 scope/{domain}.md 매칭 시도 후, cover 되지 않는 외부 클래스/도메인 reference 를 감지한다.

- `workspace/{TEAM}/context/MANIFEST.md` 의 `## 외부 도메인 reference` 표 lookup:
  - 매칭되는 도메인 있으면 아래 형식으로 1 줄 출력:
    `[INFO] features/ 의 일부 영역이 {외부 도메인} 도메인에 의존 — /dp-skills:discover {추천 경로} (전체) 또는 /dp-skills:discover --boundary {외부 도메인} --from {domain} (경계만) 후 재분석 권장`
  - 표 부재 시 또는 매칭 없으면: 안내 없이 진행 (abort 안 함).

> **A2 runtime fallback**: lookup 실패 시 → 분석 정상 진행, INFO 출력 안 함. MANIFEST 는 여기서 enrichment 용도 — 부재 시 detect 만 생략하는 best-effort 가 의도된 동작이다 ([INDEX.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/INDEX.md) § MANIFEST 부재 정책).

**Open Questions 섹션 보존 + 갱신:**

features/ 갱신 시 기존 features/NN-*.md 파일의 `## Open Questions` 섹션을 다음 규칙으로 처리한다. (외부 cross-reference 자동 발견 보조 도구는 [`references/cross-ref.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/analyze/references/cross-ref.md).)

- 기존 `## Open Questions` 섹션이 있으면 보존한다. 기존 `- (없음)` 행, 작성자 수동 기입 행 모두 그대로 유지.
- cross-domain detect 결과 (외부 도메인 매칭) → 해당 feature 의 `### (b) cross-domain 산출물 부재` 에 행 추가만 (중복 행은 skip).
- 신규 features/NN-*.md 생성 시에는 `## Open Questions` 4 카테고리 섹션 + `- (없음)` 을 반드시 포함한다.
- `## Open Questions` 섹션이 아예 없는 기존 파일은 수정하지 않는다 (doctor 가 INFO 로 안내).

> **A2 runtime fallback**: Open Questions 갱신 실패 시 → 해당 파일 skip, 분석 정상 진행.

#### 5-3. MCP cross-reference 검증 (Optional)

`mcp__atlassian__search` 사용 가능 시, 외부 컨텍스트 (회의록·검토·Jira) 와 features 의 cross-reference 를 시도해 Open Questions 보강 후보를 사용자에게 요약 제시한다. 자동 주입은 하지 않음. MCP 도구 없거나 인증 실패면 skip (abort·INFO 출력 안 함).

상세 절차 (전제조건·키워드 검색·도구 호출·노이즈 제외): [`references/cross-ref.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/analyze/references/cross-ref.md).

### 6. agents/ 자동 갱신

features/ 분석 결과로 `agents/planner.md`, `agents/generator.md`, `agents/evaluator.md` 를 갱신하고 `.agent-state.yml` 의 `analyzed: true` 게이트를 켠다.

상세 절차 (6-1 ~ 6-5): [`references/agents-update.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/analyze/references/agents-update.md).

> **편집물 보존 가드** — 6-1/6-2/6-3 은 `[analyze-managed]` 섹션을 **Edit** 으로 splice. 기존 파일 Write/Bash destructive 는 PreToolUse 훅 `protect-managed.sh` 가 차단. `--regen-agents` 처럼 의도적 재작성은 백업 후 `DP_PROTECT_BYPASS=1` 로 우회.

### 7. 분석 품질 자가 검증

6-5 (doctor) 가 끝나면 분석 내용 자체의 품질을 4 항목 (커버리지·구조·정합성·추측 혐의) 으로 자가 점검한다.

상세 절차 (7-1 ~ 7-4) + 출력 형식: [`references/self-verify.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/analyze/references/self-verify.md).

### 8. 결과 출력

분석 완료 후 아래 형식으로 요약한다:

```
분석 완료: {원본 파일명}

생성된 features:
  - features/{NN}-{slug}.md — {기능명}
  ...

총 {N}개 기능 명세 생성.

갱신된 파일:
  - project.md — 목표 {N}개 동기화 + 관련 파일(Models/Endpoints/Services) 자동 기입
  - agents/planner.md — 기능별 사전 확인 사항 갱신
  - agents/generator.md — 기술 레퍼런스 갱신
  - agents/evaluator.md — 체크리스트 갱신
  - .agent-state.yml — analyzed: true

검증: {7 단계 요약 한 줄 — "all checks passed" 또는 "WARN N건" 등}
```

---

## 참고

- `features/` 파일은 프로젝트 에이전트(@ag-planner, @ag-generator)가 직접 Read하여 사용한다.
- `docs/` 파일은 원본 보관용이며 `/dp-skills:confl` 커맨드를 통해서만 접근한다.
- 분석 품질이 낮으면 `/dp-skills:analyze --force` 로 재분석할 수 있다.
- TDD 모드에서는 @ag-planner 가 Red 단계에서 `features/NN-{slug}.md` 를 직접 읽어 실패 테스트를 작성한다 (상세: [`rgr.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/modes/rgr.md)).
