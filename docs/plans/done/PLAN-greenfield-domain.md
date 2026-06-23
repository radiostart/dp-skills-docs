# Plan — greenfield 도메인 즉석 등재 (domain-decision 확장)

> status: approved
> created: 2026-06-11
> updated: 2026-06-11 — 설계 승인 (트리거·시드 소스·접근법 사용자 확정)
> author: jpseo@jpblog.co.kr
> 관련 메타: domain 생명주기 · analyze prelude · knowledge-sync 합류

## 0. 배경·문제

신규 서비스가 추가되어 새 도메인으로 등재해야 하는데 **개발 전이라 코드
진입점이 없는** 케이스 (greenfield) 는 `/dp-skills:discover` 를 쓸 수 없다
(discover 는 "코드에 문자 그대로 있는 것만 기록" — 코드 전제).

현행 경로는 `analyze/references/domain-decision.md` (c) 의 한 줄이 전부다:

> 폴더가 비어 있으면 사용자에게 자유 입력을 요청하고, MANIFEST 등록 후
> 재진입을 안내.

두 가지 갭이 있다:

1. **동선 끊김** — 사용자가 analyze 를 중단하고 MANIFEST 행·scope 골격을
   손으로 만든 뒤 재진입해야 한다. 행 형식·골격 형식을 안내하는 절차도 없다.
2. **부분 정의** — scope 폴더에 기존 도메인이 있으면서 이번 도메인만 신규인
   케이스 (선택지 밖 자유 입력) 는 명시 규칙이 없다.

knowledge-sync (0.18.0) 가 "scope 부재 = detected" 로 환류를 보장하므로,
시작 골격만 즉석에서 만들어주면 greenfield 전체 경로가 완결된다:
**즉석 등재 → 사이클 환류 축적 → 코드 축적 후 discover 백필.**

## 1. 목표·범위

domain-decision (c) 를 **즉석 등재 절차** 로 확장한다. analyze 진입 중
미등재 도메인을 자유 입력하면 그 자리에서 MANIFEST 행 + scope 시드 +
rules 스켈레톤을 생성하고 기존 절차 (`.agent-state.yml` 기록 → scope Read)
에 합류한다.

### in scope

- `analyze/references/domain-decision.md` (c) — 즉석 등재 절차로 확장 (본체)
- `analyze/references/greenfield-scope.md` **신규** — scope 시드 골격 형식·
  features 추출 규칙·`[greenfield]` 마커 정의
- `analyze/SKILL.md` · `create-feature/SKILL.md` — 도메인 결정 요약에
  즉석 등재 언급 각 1 줄
- `context/lifecycle/state-schema.md` § domain 생명주기 2 (첫 기록) —
  즉석 등재 한 줄
- `context/lifecycle/knowledge-sync.md` — `[greenfield]` 마커 합류 규칙 1-2 줄
- 매뉴얼 (docs/) 해당 페이지 한 단락 + `plugin.json` minor bump (0.19.0)

### out of scope

- discover 에 greenfield/skeleton 모드 신설 — 기획서 시드는 discover 의
  "추측 금지" 헌장과 충돌. 등재 주체는 analyze prelude (접근법 결정, §8)
- 별도 선제 등재 진입점 (전용 스킬·플래그) — analyze 즉석 경로로 충분
- wrapper agents (`ag-planner` 등)·orchestrate-load·doctor 코드 변경 —
  시드 scope 가 discover 표준 형식과 호환이라 무수정 소비
- pilot 동기화 — pilot 은 별도 레포 (역이식은 그쪽 사이클에서)
- lifecycle/INDEX.md 갱신 — lifecycle/ 파일 추가 없음 (수동 갱신 계약 비대상)

## 2. 플로우 — domain-decision (c) 확장

1. **선택지에 "신규 도메인 등재" 경로 상시 노출.** scope 폴더가 비어 있으면
   선택지 없이 바로 신규 경로. 자유 입력값이 기존 도메인과 유사하면 오타
   방지 확인 1 줄 ("기존 `{유사후보}` 와 다른 신규 도메인입니까?").
2. **식별자 검증** — discover 와 동일 규칙 준용: 영숫자·하이픈·언더스코어·
   계층 `/` 허용, 소문자화, 연속 하이픈 축약, `..`·절대 경로 거부.
3. **미리 보기 + 승인 게이트** — 기록할 파일 3 개 (MANIFEST 행·scope 시드·
   rules 스켈레톤) 와 시드 표를 제시하고 y/n. 거부 시 아무것도 쓰지 않고
   도메인 질의로 복귀 (discover 단계 3 의 승인 게이트·Abort 계약 준용).
4. **승인 후 기록** — 순서: `scope/{domain}.md` 시드 Write →
   `rules/{domain}.md` 빈 스켈레톤 (**부재 시에만**, discover
   `references/output-format.md` 템플릿 준용) → MANIFEST `## 도메인 분류`
   행 추가 (discover `references/manifest-update.md` detect + conform 준용).
5. **기존 절차 합류** — `.agent-state.yml.domain` Edit 기록 → scope Read.

> 시드 소스는 **features/** 다 (docs/ 원문 아님). 도메인 결정은 analyze
> 단계 5 의 전제로, 단계 3-4 의 features 분할·저장이 끝난 시점이다.
> create-feature 도 prompt-origin feature 1 건이 이미 생성된 뒤라 동일
> 로직으로 커버된다 (별도 fallback 불필요).

## 3. scope 시드 골격 — `greenfield-scope.md` (신규)

```markdown
# {domain} 도메인 스코프

> [greenfield] 본 문서는 features 기반 **예정 항목** 시드다 (코드 미반영).
> 구현 진행 시 knowledge-sync 환류·discover 재실행이 코드 기준으로
> 대체한다. 아래 항목을 코드 사실로 간주하지 말 것.

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

- **H2 헤딩·표 컬럼은 discover 표준과 정확히 동일** — analyze 5-2 의
  관련 파일 추출과 planner/wrapper 의 scope 소비가 무수정 호환.
- greenfield 표식은 상단 blockquote `[greenfield]` 마커 하나. 파일 전체가
  예정 항목이므로 행 단위 표시는 두지 않는다.
- 코드 경로 컬럼 (컨트롤러#액션·파일) 은 features 에 없으므로 `(미정)`.
- features 에서 추출할 항목이 0 건이면 빈 표 + 마커만 남긴다 (자동 빈 골격
  fallback — 별도 분기 아님).
- 추출 규칙: features 본문 명세 (`NN-{slug}.md`) 의 엔드포인트·모델·서비스
  언급을 표 행으로 변환. 추측 금지 — features 에 명시된 것만.

## 4. 합류 규칙 (오염 차단)

| 합류 지점 | 규칙 |
| --------- | ---- |
| discover 재실행 | scope 단순 덮어쓰기 → 마커 자연 소멸 (기존 규칙, 명시 1 줄만 추가) |
| knowledge-sync 반영 | scope 에 `[greenfield]` 마커가 있으면: 구현 확인된 항목을 실제 값으로 갱신, 전 항목 코드 반영 시 마커 블록 제거. 항목 5 건 이상이면 기존 규칙대로 discover 재실행 권장 |
| doctor | MANIFEST 행 + scope 파일 동시 생성이라 정합 유지. `analyzed_at` 은 analyze 완료 시 기록 → scope mtime 보다 늦어 drift WARN 미발생 |

## 5. 변경 파일

| 파일 | 변경 |
| ---- | ---- |
| `skills/analyze/references/domain-decision.md` | (c) 즉석 등재 절차로 확장 (본체) |
| `skills/analyze/references/greenfield-scope.md` | **신규** — §3 형식·추출 규칙·마커 정의 |
| `skills/analyze/SKILL.md` | 도메인 결정 요약 (133 행 부근) 에 1 줄 |
| `skills/create-feature/SKILL.md` | prelude 준용부 (105 행 부근) 에 1 줄 |
| `skills/context/lifecycle/state-schema.md` | § domain 생명주기 2 에 1 줄 |
| `skills/context/lifecycle/knowledge-sync.md` | §4 합류 규칙 1-2 줄 |
| `docs/how-to/analyze-docs.md` · `docs/how-to/domain-discover.md` | greenfield 경로 한 단락씩 (analyze: 즉석 등재, discover: 코드 없을 때 대안 안내), `docs_build.py` 재실행 |
| `plugin.json` | 0.19.0 minor bump |

## 6. 엣지 케이스

- **유사 이름 입력** — 기존 도메인과 편집 거리 가까우면 확인 1 줄 (§2-1).
- **MANIFEST 행은 있는데 scope 파일이 없는 비정합** (수동 편집 잔재) —
  conform 규칙대로 행 갱신 + scope 생성 (비정합 해소 효과).
- **계층 식별자** (`{part}/{domain}`) — `scope/{part}/{domain}.md` 생성
  (discover 와 동일).
- **승인 거부** — 무기록, 도메인 질의 복귀.
- **features 추출 0 건** — 빈 표 + 마커 (§3).

## 7. 검증

- doctor 실행 — MANIFEST ↔ scope 정합 PASS 확인.
- 수기 시나리오 2 건: ① 빈 워크스페이스에서 analyze 진입 → 신규 등재 →
  5-2 관련 파일 표가 시드 기준으로 채워지는지, ② 미리 보기에서 거부 →
  디스크 무기록 확인.
- 도구 코드 (`.py`) 변경 없음 — pytest 신규 불필요.

## 8. 결정 기록

| 질문 | 결정 | 근거 |
| ---- | ---- | ---- |
| 트리거 지점 | analyze 진입 중 즉석 등재 | 동선 끊김 제거. 선제 등재 진입점은 YAGNI |
| scope 골격 내용 | features 기반 예정 항목 시드 | 초기 planner 힌트 가치. 오염은 `[greenfield]` 마커 + 합류 규칙으로 차단 |
| 로직 위치 | 접근법 A — domain-decision prelude 인라인 | 시드 소스 (기획서→features) 를 읽는 주체가 analyze. discover 에 넣으면 "추측 금지" 헌장과 충돌. 기록 주체 2 개 (analyze·create-feature) 는 state-schema domain 생명주기가 이미 선언한 모델의 자연 확장 |
