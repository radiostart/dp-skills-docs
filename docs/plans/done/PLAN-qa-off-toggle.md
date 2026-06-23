# Plan — `/dp-skills:qa off` 토글 + phase 전환 SSOT 통합

> status: approved
> created: 2026-05-29
> updated: 2026-05-29 — 설계 승인 (추출 범위·off 동작·STATE stale 포함 3개 결정)
> author: jpseo@jpblog.co.kr
> 관련 메타: qa phase lifecycle · phase 전환 SSOT · STATE.md stale 결함

## 0. 운영 전제

- **off 는 순수 phase 탈출 토글.** `/dp-skills:qa off` 는 결함 처리를 하지
  않고 qa → development 복귀만 수행한다. 다건·Jira 잔여 조회 없음 (보드뷰 금지
  원칙 유지 — PLAN-qa-phase §0).
- **phase 전환은 단일 SSOT 로.** `.agent-state.yml` phase 갱신 + STATE.md 행
  갱신을 `context/lifecycle/phase-transition.md` 한 파일로 추출하고 qa·project
  양쪽이 참조한다 (복붙 금지 — drift 방지). 메모 deferred 처방 그대로.
- **순수 리팩터 + 결함 1건 수정.** project 의 기존 동작은 100% 보존하며,
  의도된 동작 변화는 단 하나 — qa 자동진입이 STATE.md 행도 `QA` 로 갱신 (기존
  stale 결함 수정).
- **state-schema.md 변경 없음.** `phase`·`qa_started_at` 필드는 이미 존재.

## 1. 목표·범위

`/dp-skills:qa off` 를 추가해 qa phase 탈출을 qa 스킬 안에서 가능하게 한다
(현재는 `/dp-skills:project {P} --qa off` 로만 가능 — 비대칭·불편). 동시에
phase 전환 절차를 공유 SSOT 로 추출해, qa 직접 진입 시 STATE.md 가 `진행중`
으로 남는 stale 결함을 함께 해결한다.

### in scope

- 신규 `dp-skills/skills/context/lifecycle/phase-transition.md` (to-qa·to-development SSOT)
- `qa/SKILL.md`: `off` 인자 분기 + 자동진입의 SSOT 참조 (STATE.md 갱신 포함)
- `project/SKILL.md`: 2-C·3 을 SSOT 참조로 전환 (동작 보존)
- README · `docs/how-to/qa-cycle.md` · plugin.json 동기화

### out of scope

- `state-schema.md` 변경 (필드 이미 존재)
- `/dp-skills:project --qa off` 폐기 (공존 — 타 project 전환 겸 탈출 경로)
- Jira 잔여 결함 조회·보드뷰 (원칙상 영구 제외)

## 2. 신규 SSOT — `context/lifecycle/phase-transition.md`

phase 전환의 단일 정의. 두 절차 + 공통 Edit 규칙을 담는다.

### 2.1 공통 Edit 규칙

- `.agent-state.yml` 은 **Edit 도구만** 사용 (Write 금지 — 타 필드 유실 방지).
- `phase:` 줄 부재 시 `tdd:` 또는 `domain:` 줄 다음에 삽입. `schema:` 줄은
  손대지 않음 (migrate-state 별도).
- STATE.md 는 preamble **P3** 규칙 — 테이블 본문 1행 교체. 단 `status` 칸은
  phase 로 분기 (아래).

### 2.2 to-qa 절차

1. `.agent-state.yml`: `phase: qa` 로 교체/삽입.
2. `qa_started_at:` 을 현재 UTC ISO 8601 (`date -u +%Y-%m-%dT%H:%M:%SZ`) 로
   교체/삽입. 재실행 시 갱신 허용 (마지막 진입 시점).
3. STATE.md 행 → `| project | {PROJECT} | QA |`.
4. idempotent: `phase: qa` 재적용은 무변화, `qa_started_at` 만 갱신.

### 2.3 to-development 절차

1. `.agent-state.yml`: `phase: development` 로 교체/삽입.
2. `qa_started_at:` **보존** (감사 이력). `qa/` 폴더 **보존**.
3. STATE.md 행 → `| project | {PROJECT} | 진행중 |`.
4. idempotent: 이미 development 면 무변화.

> STATE.md 의 `QA` 는 P3 의 literal `진행중` 을 대체하는 **project 모드 의도된
> 예외** — preamble 으로 되돌리지 말 것. (기존 project §3 주석 이관)

## 3. qa/SKILL.md 변경

### 3.1 인자 분기 (1단계 앞단)

`$ARGUMENTS` 가 `off` (대소문자 무시) → **off 모드** (§3.3). 그 외 → 기존
`^[A-Z]+-\d+$` Jira 키 검증 유지.

### 3.2 자동진입 SSOT 참조 (기존 3단계, 현 75-103)

`phase: development`·또는 부재 → 자동 qa 전환 시 **phase-transition.md
§2.2 (to-qa) 수행**. 기존 인라인 절차(phase·qa_started_at 갱신)를 참조로
대체하되, **STATE.md QA 행 갱신이 포함**되어 stale 결함이 해소된다. migrate
advisory(phase 부재 시 한 줄)는 qa SKILL 에 유지.

### 3.3 off 모드 절차 (신규)

1. preamble P1(팀)·P2(활성 project) 수행.
2. 현재 `.agent-state.yml` 의 `phase` 확인:
   - `phase != qa` (development·부재) → `"이미 development phase 입니다."`
     안내 후 종료 (idempotent, 변경 없음).
   - `phase == qa` → 3 으로.
3. phase-transition.md §2.3 (to-development) 수행.
4. 안내 1줄: `"qa → development phase 복귀. features/ 수정 가능."`
5. off 모드는 결함 처리·Jira 조회를 하지 않는다.

### 3.4 안내 문구 갱신 (현 200-208 "명시 비분기")

features 수정 복귀 안내를 `/dp-skills:qa off` 또는
`/dp-skills:project {PROJECT} --qa off` 둘 다로 갱신.

## 4. project/SKILL.md 변경

- **2-C (phase 전환)**: `{QA} == on` → phase-transition.md §2.2, `{QA} == off`
  → §2.3 참조로 대체. features/ 비어있을 때 WARN·TDD 직교 주석은 project 에 유지.
- **3 (STATE.md 갱신)**: phase 분기 행 형식을 phase-transition.md 가 정의하므로,
  project §3 은 "to-qa/to-development 절차가 STATE.md 행을 갱신한다" 로 축약하고
  P3 예외 주석은 SSOT 로 이관 (중복 제거).
- `--qa off` 옵션·인자 파싱·사용 예는 **그대로 유지**.

## 5. 문서·버전 동기화

| 파일 | 작업 |
|---|---|
| `dp-skills/skills/context/lifecycle/phase-transition.md` | **신규** SSOT |
| `dp-skills/skills/qa/SKILL.md` | off 분기 + 자동진입 SSOT 참조 + 안내 갱신 |
| `dp-skills/skills/project/SKILL.md` | 2-C·3 SSOT 참조 전환 (동작 보존) |
| `dp-skills/README.md` | qa 스킬 설명에 off 토글 1줄 (스킬 수 불변) |
| `dp-skills/docs/how-to/qa-cycle.md` | off 토글 사용법 반영 |
| `dp-skills/.claude-plugin/plugin.json` | minor bump |
| docs reference 색인 | `python3 tools/docs_build.py` 재생성 |

## 6. 위험·검증

- **회귀 위험**: project 의 phase 전환·STATE.md 동작이 추출 후에도 동일해야
  함. 추출은 "문구 이동"이며 분기·Edit 규칙·예외 주석을 보존한다.
- **검증**: `doctor.py --schema` · `docs_build.py --check` · `markdownlint` ·
  `tests/tools/`. doctor 의 phase·STATE 정합 체크가 있으면 그대로 통과해야 함.
- off idempotent·이미-development 분기를 SKILL read-through 로 확인.

## 7. 미해결 질문

- phase-transition.md 를 `context/lifecycle/` 직하에 둘지 하위 폴더에 둘지 —
  초판은 직하 (state-schema.md·drift-protocol.md 와 동렬).
- doctor 에 "phase==qa 인데 STATE.md 가 진행중" 정합 체크를 추가할지 — 본 PLAN
  이 결함을 코드로 해소하므로 초판 제외, 운영 후 검토.
