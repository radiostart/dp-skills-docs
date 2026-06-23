# Plan — MANIFEST 부재 정책·domain 생명주기 일원화

> status: approved (설계 판단 위임분 — 최종 검토는 PR 리뷰에서)
> created: 2026-06-10
> author: jpseo@jpblog.co.kr
> 관련 메타: 팀 컨텍스트 게이트 · `.agent-state.yml` domain 필드 · messages.md SSOT

## 0. 문제 (확정된 사실 + 탐색 검증 결과)

### 0-1. MANIFEST 부재 시 동작 비일관

| 위치 | 현재 동작 | 비고 |
| --- | --- | --- |
| `skills/context/shared/preamble.md` P4 (project·issue) | "messages.md 의 팀 컨텍스트 미설정 안내 후 종료" | **참조하는 메시지 key 가 messages.md 에 실재하지 않음 (drift)** |
| `skills/discover/SKILL.md` 사전 확인 5 | 종료 + team-init 안내 | 메시지 하드코딩 (messages.md 미경유) |
| `skills/create-feature/SKILL.md` 3-bis A2 | lookup 실패 → spec 진행 (abort 안 함) | 사유 미기재 |
| `skills/analyze/SKILL.md` cross-domain detect A2 | lookup 실패 → 분석 정상 진행 | 사유 미기재 |
| `agents/ag-code-reviewer.md` | domain 미식별 → scope 로드 skip (workspace-less 모드) | 사유 기재됨 |
| `skills/context/INDEX.md` Fallback 표 | "안내하고 종료" 1행뿐 | 계층 구분 없음 |

### 0-2. domain 결정 시점·주체 분산 (탐색으로 보정된 사실)

- `state-schema.md` `### domain`: "null → analyze 진입 시 사용자 질의 필수, 자동 추론 기록 금지" — **생명주기 전체 (소비·교정 경로) 는 미문서화**.
- `analyze`·`create-feature`: 도메인 결정 prelude 로 질의 후 기록 (기록 주체).
- wrapper agents 4종 (planner·critic·generator·evaluator): `domain: null` 이면 질의 후 scope/rules 수동 Read — **확정값을 state 에 기록하는지 여부 미정의** (모호).
- `pr`·`commit`·`slack`: **domain 의존 없음** (과제 기술과 달리 "미결정 시 실패" 하는 스킬은 실재하지 않음 — grep 검증 완료). 도메인을 전제하는 것은 wrapper 에이전트들이며, 전부 질의 폴백을 가진다.
- non-null 값의 **변경(교정) 경로가 어디에도 없음** — analyze 는 non-null 이면 재질의 금지, state-schema 는 수동 편집 금지.

## 1. 설계 결정

### D1. MANIFEST 부재 비대칭 = 의도된 설계 — "게이트 / best-effort" 2계층 명문화

- **게이트 (부재 = 종료)**: `project`·`issue` (P4), `discover`. 진입·생산 스킬 — 팀 컨텍스트 없이 진입하면 이후 사이클 전체가 도메인 지식 없이 동작하고, discover 는 MANIFEST 색인 자체를 갱신하는 생산자라 색인 없는 곳에 누적 불가.
- **best-effort (부재 = enrichment 생략, 진행)**: `analyze`·`create-feature` 의 detect, `@ag-code-reviewer`. 활성 프로젝트 내부 스킬 — 정상 흐름이면 게이트를 이미 통과했으므로 부재 = 사후 드리프트. 진행 중인 사용자 작업물을 abort 로 버리지 않는다. 드리프트 감지는 doctor 책임.
- 원칙 한 줄: **"진입·생산은 게이트, 진입 후는 관용 (fail at the gate, degrade inside)"**.
- 정책 SSOT: `skills/context/INDEX.md` "팀 컨텍스트" 절에 "MANIFEST 부재 정책" 소절 + 분류 표. 각 스킬에는 사유 1줄 + SSOT 참조만.

### D2. `manifest_missing` 메시지 key 신설 (messages.md)

P4 의 깨진 참조 (실재하지 않는 key) 해소 + discover 하드코딩 제거. 톤은 기존 `team_missing` 과 동일 (상황 설명 → 해결 명령).

### D3. domain 생명주기 절 — `state-schema.md` `### domain` 에 SSOT 소절 신설

1. **초기 = null** (project 신규 생성. 자동 추론 기록 금지)
2. **첫 기록** = analyze·create-feature 진입 시 후보 제시 → **사용자 확정** → 기록. 기록 주체는 이 둘뿐.
3. **소비 (read-only)** = P4 · orchestrate-load → wrapper 4종 · ag-code-reviewer · doctor.
4. **null 인 채 wrapper 진입** = 세션 한정 질의, state 기록 안 함 (모호했던 지점을 "기록 안 함" 으로 확정 — 기록 경로를 2개 스킬로 좁게 유지해야 오염 추적이 가능).
5. **변경 (교정)** = `domain` 을 null 로 되돌린 뒤 `/dp-skills:analyze --force` 재분석. 수동 편집 금지 원칙의 **명시된 유일한 예외** (전용 스킬 신설은 YAGNI — 빈도가 낮고 절차 2단계로 충분).

### D4. 참조 보강 (1줄 단위)

preamble P4 (create-feature 누락 보정 + 생명주기 참조), domain-decision.md (SSOT 역참조), wrapper agents 4종 ("세션 한정, 기록 안 함" 1문장).

### D5. 비채택 항목

- 새 pytest 추가 안 함 — messages.md key 참조 검증은 참조 표기 패턴이 다양해 (key 단독·`messages.md:key`·산문) 취약한 테스트가 됨. docs_build `--check` (pre-push) 가 reference 동기화는 보장.
- pr·commit·slack 수정 없음 — domain 의존이 실재하지 않음 (0-2).
- 게이트→best-effort 통일 (전부 진행) 비채택 — project 진입을 MANIFEST 없이 허용하면 "팀 셋업 안 된 상태로 사이클 진입" 이라는 더 나쁜 비대칭이 생긴다. 비대칭 제거가 아니라 **사유 명문화** 가 올바른 해소.

## 2. 변경 목록 (파일별)

- [ ] **T1** `skills/context/shared/messages.md` — 전제조건 메시지 절 `state_corrupt` 뒤에 추가:

  ```markdown
  ### `manifest_missing`

  workspace/{TEAM}/context/MANIFEST.md 가 없습니다. 먼저 팀 컨텍스트를 설정하세요.

    /dp-skills:team-init {TEAM}
  ```

- [ ] **T2** `skills/context/INDEX.md` — ① "팀 컨텍스트" 절 끝 (경로 컨트랙트 문단 뒤) 에 `### MANIFEST 부재 정책` 소절 신설 (D1 의 2계층 표 + 원칙 1줄 + `manifest_missing`·doctor 참조). ② Fallback 규칙 표의 MANIFEST 행을 "스킬 계층별 상이 — MANIFEST 부재 정책 표 참조" 로 교체.

- [ ] **T3** `skills/context/shared/preamble.md` — P4 1번: "팀 컨텍스트 미설정 안내" → "`manifest_missing` 출력 후 종료" + 게이트 사유 1줄 + INDEX 참조. P4 3번: "`/dp-skills:analyze` 가 확정한다" → analyze·create-feature 병기 + state-schema 생명주기 참조.

- [ ] **T4** `skills/discover/SKILL.md` 사전 확인 5 — 하드코딩 메시지 → `manifest_missing` key 참조 + 생산자 사유 1줄 + INDEX 참조.

- [ ] **T5** `skills/create-feature/SKILL.md` 3-bis A2 fallback — 의도 사유 블록 1개 추가 (게이트는 진입 시 통과 완료, MANIFEST 는 enrichment 용도, 작업물 보존).

- [ ] **T6** `skills/analyze/SKILL.md` cross-domain detect A2 fallback — best-effort 의도 1문장 + INDEX 참조.

- [ ] **T7** `skills/context/lifecycle/state-schema.md` — ① `### domain` 절에 `**생명주기 (SSOT)**` 5단계 소절 추가 (D3). ② 금지 사항 "수동 편집 금지" 행에 "유일한 예외: domain null reset (§ 생명주기 5)" 추가.

- [ ] **T8** `skills/analyze/references/domain-decision.md` — 서두에 "본 문서는 생명주기 중 '첫 기록' 단계의 절차 상세 — SSOT 는 state-schema.md § domain 생명주기" 1줄.

- [ ] **T9** `agents/ag-planner.md`·`ag-planner-critic.md`·`ag-generator.md`·`ag-evaluator.md` — 도메인 판정 실패 처리 문장에 "확정값은 세션 한정 — state 기록은 analyze·create-feature 만 수행" 보강.

## 3. 검증·릴리스

- [ ] **V1** `python3 tools/docs_build.py` 재실행 → `docs/reference/**` 동기화, `--check` exit 0 확인.
- [ ] **V2** `pytest tests/tools -q` — 450 passed (worktree 베이스라인) 유지.
- [ ] **V3** `.claude-plugin/plugin.json` `0.16.12` → `0.16.13` (patch — 문서·정책 정합화, 신규 기능 없음).
- [ ] **V4** 커밋 → push → PR (base `main`, squash). pre-push hook (`docs_build --check` 포함) 통과 필수, `--no-verify` 금지.
