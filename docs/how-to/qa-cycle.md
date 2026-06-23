# QA 사이클 진행

!!! info "한 줄 요약"
    development 가 끝난 project 의 phase 를 qa 로 전환해 Jira 결함 1건을 분석·수정·회신합니다. 본 스킬은 1건 처리 전용이며, 다건 추적은 Jira UI 가 SSOT 입니다.

## 언제 쓰나

- features 가 완성된 project 에 **QA 가 결함을 Jira 로 보고** 했을 때
- 결함 수정 시 **project 컨텍스트 (features·scope·rules·agents) 가 필요** 할 때

project 컨텍스트가 필요 없는 단발 운영 장애·핫픽스는 [`/dp-skills:issue`](../reference/skills/issue.md) 를 사용합니다 — 두 스킬은 공존합니다.

본 스킬은 **명시 호출 전용**입니다 — 대화에 Jira 링크·이슈 키를 붙여넣는 것만으로는 발동하지 않으며, `/dp-skills:qa {KEY}` 로 직접 호출해야 합니다.

## 전제

- 활성 project 가 있을 것 (`workspace/{TEAM}/STATE.md` 에 `| project | X | ... |` 행 존재).
- features/ 가 한 번이라도 채워져 있을 것 (`doctor.py` 가 phase=qa + features/ 부재 시 ERROR).
- Atlassian env vars 설정 — `ATLASSIAN_BASE_URL`·`ATLASSIAN_EMAIL`·`ATLASSIAN_TOKEN` (Confluence·Jira 공용). 기존 `JIRA_*` 도 그대로 인식. 자세한 내용은 [`jira.py`](../reference/tools/jira.md) 참조.

## 절차

### 1. development → qa 전환

```text
/dp-skills:project MyProject --qa on
```

- `.agent-state.yml` 의 `phase` 가 `qa` 로 갱신되고 `qa_started_at` 이 기록됩니다 (phase 의 단일 SSOT).
- STATE.md status 칸은 바뀌지 않습니다 — 활성 project 행은 `| project | MyProject | 진행중 |` 그대로입니다.
- 이후 hook 가 features/ 본문 쓰기를 차단합니다.

`--qa` 만 써도 `--qa on` alias 로 동작합니다. 인자 없는 `--qa` 는 on 입니다.

> 참고: `/dp-skills:qa {KEY}` 를 development phase 에서 직접 호출해도 qa 로 자동 전환되므로, 이 수동 전환은 결함 없이 phase 만 미리 바꾸고 싶을 때만 필요합니다.

### 2. Jira 결함 1건 진입

```text
/dp-skills:qa QAPRJ-1234
```

내부 동작:

- 인자·형식 검증 (`^[A-Z]+-\d+$`).
- 활성 project 확인 + phase=qa 확인.
- 같은 KEY 가 다른 project 에 이미 있으면 WARN + confirm.
- 신규 진입이면 `python3 ${CLAUDE_PLUGIN_ROOT}/tools/jira.py fetch QAPRJ-1234` 로 본문 prefill 후 `qa/QAPRJ-1234.md` lazy 생성.
- @ag-planner 핸드오프 직전 **작업 브랜치 생성 여부를 묻습니다** — base 는 `fix/{KEY}`. 같은 티켓이 해결 후 다시 진행되는 재작업이면 기존 `fix/{KEY}` 브랜치와 충돌하지 않도록 `fix/{KEY}-2`·`-3` … 로 번호를 증가시킵니다. `Y` 면 현재 HEAD 에서 `git checkout -b` 로 생성, `n` 이면 현재 브랜치 유지.

`qa/{KEY}.md` 는 다음 섹션을 갖습니다 — **현상·연관 feature·재현 경로·원인·조치·회귀영향**. **회신 섹션은 없습니다** — 회신 이력의 SSOT 는 Jira 코멘트입니다.

### 3. planner → generator → evaluator 사이클

스킬이 끝나면 안내에 따라 에이전트 사이클을 호출합니다.

```text
@ag-planner
@ag-generator
@ag-evaluator
```

phase=qa 분기로 각 에이전트의 책임이 확장됩니다.

| 에이전트 | qa 분기 추가 |
|---|---|
| `@ag-planner` | 결함 수정 모드 — 최소 변경 우선. 회귀영향 후보 파일 명시 |
| `@ag-generator` | 결함 지점 국소 수정. 동일 함수 외 변경 금지 |
| `@ag-evaluator` | **회귀영향 평가 섹션 필수** — 동일 호출 경로 feature 명단·영향 판단 |

### 4. 커밋

```text
/dp-skills:commit
```

`qa/{KEY}.md` 분석 결과와 코드 수정을 함께 커밋합니다.

### 5. Jira 회신

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/tools/jira.py comment QAPRJ-1234 "원인: ... / 조치: ... / 회귀영향: ..."
```

- 성공 → stdout `OK` (또는 silent), exit 0.
- 실패 → stderr 첫 줄 `[ERROR] COMMENT FAILED for QAPRJ-1234` 헤더, exit 1. 사용자가 흘려보내지 않도록 헤더가 항상 stderr 첫 줄.

회신 본문은 `qa/{KEY}.md` 의 원인·조치·회귀영향 요약을 그대로 옮기면 됩니다. 로컬에는 "회신했음" 기록을 두지 않습니다 — 언제 회신했나는 Jira 가 답합니다.

### 6. (선택) development 복귀

회귀 sprint 등으로 features/ 본문을 다시 수정해야 한다면:

```text
/dp-skills:qa off
# 또는 다른 project 로 전환하며 탈출:
/dp-skills:project MyProject --qa off
```

qa 스킬 안에서 바로 나가려면 `/dp-skills:qa off` 가 간편합니다 — 현재 활성 project 를 development 로 되돌립니다.

`qa/` 파일은 보존되고 `qa_started_at` 도 유지됩니다. 필요시 다시 `--qa on` 으로 전환 가능.

## 다건 추적은 Jira UI

본 스킬은 **건 1개 처리 전용** 입니다. 잔여 결함 목록·우선순위·재오픈 현황·진행 카운트는 Jira board 에서 확인하세요. 본 플러그인은 **로컬 보드뷰·INDEX·진행 요약** 을 만들지 않습니다 — 동시에 여러 project 가 QA phase 일 수 있는 운영 전제 ([Lifecycle phases — 설계 전제](../explanation/lifecycle-phases.md#_1)) 에서 SSOT 가 둘이 되면 불일치가 발생하기 때문입니다. `jira.py` 에도 의도적으로 `list` 명령이 없습니다.

## 흔한 실수

- **features/ 본문을 phase=qa 에서 수정 시도** — hook 가 차단합니다. features 수정이 필요하면 `/dp-skills:qa off` (또는 `/dp-skills:project X --qa off`) 로 development 복귀 후 별도 사이클.
- **`jira.py comment` 실패를 흘려보냄** — stderr 첫 줄 `[ERROR] COMMENT FAILED` 헤더와 non-zero exit 를 항상 확인하세요.
- **인자 없이 `/dp-skills:qa`** — 즉시 에러 + 사용법 안내. 본 스킬은 인자가 필수입니다.

## 다음 단계

- Explanation: [Lifecycle phases — development / qa](../explanation/lifecycle-phases.md) — phase 개념·전환 흐름.
- Reference: [`/dp-skills:qa`](../reference/skills/qa.md) · [`/dp-skills:project`](../reference/skills/project.md) · [`jira.py`](../reference/tools/jira.md)
- How-to: [TDD 모드 활성화](tdd-mode.md) — qa phase 에서도 함께 켤 수 있습니다 (phase ⊥ mode).
