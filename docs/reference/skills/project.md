# `/dp-skills:project`

> 새 프로젝트를 시작하거나 기존 프로젝트를 재개할 때 사용한다. `workspace/{TEAM}/projects/{PROJECT}/` 폴더를 생성·로드하고 STATE.md 를 갱신한 뒤 도메인 컨텍스트를 적재한다. 인자로 Confluence URL 이 오면 내부적으로 confl·analyze 를 위임 호출하고, `--tdd` 플래그로 TDD 모드를 켤 수 있다. 단발 이슈는 `/dp-skills:issue` 를 사용한다.

프로젝트 개발 모드를 활성화한다.

대상 프로젝트: $ARGUMENTS

**옵션:**

- `--tdd` — 프로젝트를 TDD 모드로 활성화한다 (신규·기존 모두 적용)
- `--qa [on|off]` — QA phase 전환. `--qa` (bare) 또는 `--qa on` 은 진입, `--qa off` 는 development 로 복귀. 부재 시 phase 미변경.
- `{URL}` — Confluence 기획서 URL 또는 page_id. 기획서 저장과 분석까지 한번에 수행

**사용 예:**

```
/dp-skills:project MyProject
/dp-skills:project MyProject --tdd
/dp-skills:project MyProject https://wiki.example.com/pages/12345
/dp-skills:project MyProject https://wiki.example.com/pages/12345 --tdd
/dp-skills:project MyProject --qa
/dp-skills:project MyProject --qa off
```

---

## 사전 확인

[preamble.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/preamble.md) 의 **P-1, P0, P1** 수행.

- P-1: TodoWrite 선로딩 (다단계 스킬).
- P0: 컨텍스트의 `MEMORY.md` 인덱스에서 `{PROJECT}`·`{CONFL_URL}`·주변 키워드와 관련된 메모를 가려 본문을 Read 하여 과거 동일·유사 주제의 이력을 컨텍스트에 반영 (선행 구현 코드·정책 점검 이력 등).
- P1: `{TEAM}` 획득.

---

## 수행 절차

아래 순서대로 진행한다. 중간 단계에서 "다음 작업" 안내를 출력하지 않고 모든 단계를 마친 뒤 결과 출력에서 한 번만 안내한다 — 단계마다 안내를 흘리면 사용자가 파편적으로 인지하고 후속 단계 호출을 놓치는 경우가 잦았다.

### 1. 인자 파싱

`$ARGUMENTS` 를 아래 4가지 토큰으로 분류한다:

| 토큰 패턴 | 분류 |
| --- | --- |
| `--tdd` | `{TDD}` 플래그 |
| `--qa` (bare) 또는 `--qa on` | `{QA} = on` |
| `--qa off` | `{QA} = off` |
| `http://`, `https://` 로 시작 또는 순수 숫자 | `{CONFL_URL}` |
| 그 외 | `{PROJECT}` |

`on`/`off` 는 **직전 토큰이 `--qa` 인 경우에만** 페어로 해석한다. 단독 `on` 또는 `off` 토큰은 `{PROJECT}` 로 분류된다 (예약어 검증에서 일반 이름과 동일하게 다뤄짐). 부재 시 `{QA}` 는 none (phase 변경 없음).

검증:

- `{PROJECT}` 가 비어있으면 [messages.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/messages.md) 의 `project_name_required` 출력 후 종료.
- `{PROJECT}` 가 예약어(`example`, `workspace`, `STATE`, `TEAM`)이면 [messages.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/messages.md) 의 `project_name_reserved` 출력 후 종료.

### 2. 프로젝트 폴더 생성/로드

`workspace/{TEAM}/projects/{PROJECT}/` 폴더 + 핵심 파일 4 종을 **각각 확인**한다. Glob 1 회로 폴더만 보고 끝내지 말 것 — 폴더가 있는데 파일이 일부만 있는 상태가 흔하다 (사용자가 git checkout 으로 일부 파일만 가져왔거나, .gitignore 영향으로 일부만 추적된 경우 등).

```bash
ls -1 workspace/{TEAM}/projects/{PROJECT}/ 2>/dev/null
ls -1 workspace/{TEAM}/projects/{PROJECT}/agents/ 2>/dev/null
```

각각의 핵심 파일 존재 여부를 별도로 판정한다:

- `project.md`
- `agents/planner.md`
- `agents/generator.md`
- `agents/evaluator.md`
- `.agent-state.yml`

**분기**:

- **폴더 자체가 없음** (`ls` 첫 명령 실패): 신규 — example 템플릿 복사 진행.
- **폴더 있음 + 모든 핵심 파일 존재**: 기존 — **Read 만** 수행. 어떤 Write/Bash 도 금지.
- **폴더 있음 + 일부 파일만 존재**: 사용자에게 즉시 보고하고 중단. 자동 보완 금지 — example 템플릿을 덮어쓰면 누락처럼 보이는 파일이 사실 git checkout 중인 상태 등 의도적 부분 보유일 수 있음. 메시지 예: `"폴더는 있는데 일부 파일만 발견됐습니다: {존재}, 누락: {목록}. (a) git status·git stash 확인 후 복원, (b) example 템플릿으로 누락분만 채울지 확인 필요. 어떻게 진행할까요?"`

#### 2-A. 신규 생성 절차 (분기 "폴더 자체가 없음" 일 때만)

[skills/context/lifecycle/projects/example/](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/projects/example) 의 파일 4종 (`project.md`, `agents/planner.md`, `agents/generator.md`, `agents/evaluator.md`) 을 **그대로 복사**한 뒤 `{프로젝트명}` 토큰만 실제 프로젝트명으로 치환한다. **그 외 본문은 일절 재작성·요약·환각·도메인 예시 삽입 금지.**

- example 은 실구현 콘텐츠 없이 구조·세만틱 명시만 담긴 순수 템플릿이다. 본문의 `{…}` 플레이스홀더와 `_(analyze 실행 전 …)_` 같은 표식은 그대로 유지한다.
- 섹션명·순서는 `/dp-skills:analyze` 주입 대상과 동기화되어 있다. 임의 변경 금지.
- `## 개요` / `## 제한사항` / `## 목표` / `관련 파일` 표는 이후 사용자가 수동으로 또는 `/dp-skills:analyze` 가 자동으로 채운다. 스캐폴딩 단계에서 추측·기입하지 않는다.
- 구조·섹션 의미에 대한 상세는 [GUIDE.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/projects/GUIDE.md) 를 참조하되 **가이드 본문을 생성물에 복사하지 않는다** (가이드는 구조 참고용, example 이 실제 스캐폴딩 소스).

`.agent-state.yml` 초기화 — 프로젝트 루트에 아래 내용으로 생성한다 (스키마 상세: [state-schema.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/state-schema.md)):

```yaml
schema: v1.2
analyzed: false
tdd: false
domain: null
plugin_version: "{PLUGIN_VERSION}"
```

`{PLUGIN_VERSION}` 은 Bash 로 `python3 -c "import json,os; print(json.load(open(os.environ['CLAUDE_PLUGIN_ROOT']+'/.claude-plugin/plugin.json'))['version'])"` 를 실행해 얻은 현재 플러그인 버전으로 치환한다. 환경변수 미설정 등으로 값을 얻지 못하면 해당 라인 자체를 생략한다 (`plugin_version` 은 optional — 다음 writer 이벤트에서 채워짐).

`domain` 은 `/dp-skills:analyze` 진입 시 사용자 확인을 거쳐 값이 채워진다. 여기서 자동 추론·기입 금지.

#### 2-B. 기존 폴더 로드 절차 (분기 "폴더 있음 + 모든 핵심 파일 존재" 일 때만)

`project.md` 및 `agents/` 문서를 **Read 만** 한다.

- **절대 금지**: 프로젝트 폴더 내 기존 파일 (managed + 사용자 메모·스크래치 등 모든 파일) 의 덮어쓰기·삭제. 동일 프로젝트명 재호출은 example 재복사·폴더 reset·재초기화 모두 금지.
- **허용**: 누락된 `.agent-state.yml` 신규 생성, `STATE.md` 의 활성 프로젝트 행 갱신 (3 단계), 기존 파일에 대한 Edit (splice).
- PreToolUse 훅 `protect-managed.sh` 가 결정론적으로 강제 (Write + destructive Bash 차단). 의도적 reset 은 `DP_PROTECT_BYPASS=1` 환경변수로 우회 (백업 권장).
- `.agent-state.yml` 이 없거나 `schema: v1` 이면 **사용자에게 `python3 ${CLAUDE_PLUGIN_ROOT}/tools/migrate-state.py workspace --apply` 실행을 안내** (v1 → v1.2 업그레이드 필요).
- **drift 체크** (state 가 v1.1+ 이고 `analyzed: true`, `analyzed_at` 존재 시):
  - `docs_last_fetched_at > analyzed_at` → "기획서가 analyze 이후 변경됐습니다. 재분석할까요? (y/n)" 질의. `y` 면 6·7·8단계 수행, `n` 이면 스킵.
  - `count(features/NN-{slug}.md 본문 명세 — 파생 산출물 제외) > last_analyzed_features + 1` → "features 가 N 건 추가됐습니다. `/dp-skills:analyze --regen-agents` 권장" 안내 (자동 실행 X).
  - `scope/{domain}.md` mtime > `analyzed_at` → "scope 업데이트됨 → `--regen-agents` 권장" 안내.

### 2-C. phase 전환 (`{QA}` 있을 때만)

[phase-transition.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/phase-transition.md) 의 절차를 수행한다.

- **`{QA} == on`** → `to-qa` 절차.
  추가로 `features/` 폴더가 없거나 `features/NN-{slug}.md` 본문 명세가 0건이면
  다음 WARN 출력 (종료는 안 함):

  ```
  features/ 가 비어있는데 QA phase 진입. /dp-skills:doctor 또는 /dp-skills:analyze 먼저 실행 권장.
  ```

- **`{QA} == off`** → `to-development` 절차.

> `{TDD}` 플래그와 직교 — `--qa` 와 `--tdd` 가 동시 입력돼도 양쪽 모두 적용된다 (state-schema.md 의 phase 와 mode/tdd 직교 원칙).

### 3. STATE.md 갱신

[preamble.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/preamble.md) 의 **P3** 를 수행한다 — 활성
project 행을 `| project | {PROJECT} | 진행중 |` 로 둔다. status 칸은 phase 를
비추지 않으므로 `{QA}` 입력 여부와 무관하게 항상 `진행중` 이다. phase 전환
(`{QA}` 입력) 은 [phase-transition.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/phase-transition.md)
가 `.agent-state.yml` 만 갱신하며 STATE.md 는 건드리지 않는다.

### 4. 도메인 컨텍스트 로드

[preamble.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/preamble.md) 의 **P4** 수행.

### 5. TDD 모드 적용 (`{TDD}` 있을 때만)

[tdd-activation.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/modes/tdd-activation.md) 절차를 수행한다.

### 6. 기획서 저장 (`{CONFL_URL}` 있을 때만)

아래 Bash 명령을 이 턴에 직접 실행한다. `/dp-skills:confl` 스킬을 nested 호출하지 않고 동일 도구를 Bash 로 직접 사용한다 — nested 호출은 컨텍스트가 두 번 적재되거나 작업 경로가 어긋나 후속 7·8 단계와 동기화되지 않는다.

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/tools/confluence.py fetch "{CONFL_URL}"
```

- 성공 시 저장된 파일 경로와 섹션 목록을 출력한다.
- 실패 시 에러 메시지를 안내하고 7·8단계를 건너뛰어 9단계로 진행한다. 2단계 템플릿은 유지된다.

### 7. 기능 분석 (6단계 성공 시만)

사용자에게 분석 범위를 질의한다:

- **전체 분석** — docs/ 모든 내용
- **키워드 필터** — 특정 영역만 (다운로드된 docs 의 H2 섹션을 훑어 실제 키워드 2~4개를 예시로 제시. 예: "ADMIN" · "API" · "정책")
- **건너뛰기** — 나중에 `/dp-skills:analyze` 로 별도 실행

사용자 선택에 따라 [../analyze/SKILL.md](analyze.md) 의 "분석 프로세스" 1~5단계를 실행한다. 건너뛰거나 분석 실패 시 8단계를 건너뛰고 9단계로 진행한다.

### 8. project.md 및 agents/ 갱신 (7단계에서 분석 수행 시만)

[../analyze/SKILL.md](analyze.md) 의 "분석 프로세스" 6~7단계를 실행한다 (project.md 목표 동기화 + agents/ 갱신).

### 9. 무결성 검증 (자동)

모든 단계 완료 후 아래를 실행한다:

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/tools/doctor.py workspace
```

- ERROR 또는 WARN 있으면 **원문을 사용자에게 그대로 출력**한다.
- 모두 PASS 면 `doctor: all checks passed` 한 줄만 표시.

### 10. 결과 출력

프로젝트 컨텍스트를 요약하고 다음 작업을 안내한다.

- `project.md` 제한사항에 **TDD 모드** 문구가 있으면: "이 프로젝트는 TDD 모드입니다. `@ag-planner` 호출 시 스텝 분할과 실패 테스트 작성이 함께 수행됩니다." 안내 추가.

**다음 단계 안내** — `.agent-state.yml` 의 `phase` 와 features/ 존재 여부로 분기. 아래 순서를 **명시적 if / elif / else** 로 적용해 한 LLM 호출에 두 안내가 동시에 나오지 않도록 한다:

1. **phase == qa 면**: "이 project 는 QA phase 입니다. 결함 처리는 `/dp-skills:qa {JIRA-KEY}` 로 진입." 한 줄 출력 후 종료. (features/ 유무 별도 분기 없음 — qa phase 진입 자체가 features 완료 전제.)
2. **elif (phase != qa) features/ 가 비어있지 않으면**: 생성된 features 와 갱신된 파일을 요약하고 "`@ag-planner` 를 호출해 구현을 시작하세요." 안내.
3. **else (phase != qa, features/ 비어있음)**: 생성된 파일(`project.md`, `agents/`)을 요약하고 아래 순서 안내:
   1. `project.md` 의 개요·목표·관련 파일을 채운다
   2. **기획서 기반 다건 생성** — `/dp-skills:confl {url}` → `/dp-skills:analyze`
   3. **프롬프트 기반 단건 생성** — `/dp-skills:create-feature "<원하는 기능 설명>"` (기획서 없을 때 — 한 줄 요약이 아니라 원하는 기능을 충분히 서술)
   4. 준비되면 `@ag-planner` 호출

**선택 안내 (공통):** "작업 완료·승인 알림을 Slack 으로 받고 싶으면 `/dp-skills:slack` 으로 활성화하세요." 한 줄을 덧붙인다. 기존 프로젝트 재활성화 시에는 출력하지 않는다 (이미 알고 있거나 의도적으로 비활성).
