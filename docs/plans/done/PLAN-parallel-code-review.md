# Plan — 병렬 셀프 코드 리뷰 스킬 `/dp-skills:dp-code-review`

> status: approved
> created: 2026-05-29
> updated: 2026-05-29 — 설계 승인 (관계·fan-out 단위·이름 3개 결정)
> author: jpseo@jpblog.co.kr
> 관련 메타: code-review 사이클 · ag-code-reviewer 재활용 · 병렬 에이전트 오케스트레이션

## 0. 운영 전제

- **읽기 전용 작업이다.** worker (`@ag-code-reviewer`) 는 Edit/Write 권한이
  없으므로 (`agents/ag-code-reviewer.md:5`) 병렬 실행해도 파일 충돌·race 가
  원천적으로 없다. worktree 격리 불필요.
- **기존 `@ag-code-reviewer` 멘션은 깨지지 않는다.** 인자 없이 호출하면
  오늘과 100% 동일 — 단일 에이전트가 5축을 순차 실행. 본 스킬은 그 위에
  얹는 오케스트레이션 레이어일 뿐이다. 16개 참조 파일은 점진 안내로 흡수.
- **출력 컨트랙트는 불변.** 단일 CODE REVIEW REPORT (동일 필드) 를 내고
  `next` 로 `/dp-skills:fix-review` 로 라우팅한다. 하위 도구 무수정 호환.
- **공식 `/code-review` 와 분리.** 본 스킬은 dp-skills 사이클 내부 셀프
  리뷰 (팀 컨벤션·언어 룰·cross-cutting 일관성) 전용. 공식 도구는 범용
  안전망 — 역할이 다르며 본 스킬이 대체하지 않는다.
- **승인 게이트 없음.** 리뷰는 본래 사용자 게이트가 없는 단계 — fan-out 이
  중간 가시성을 줄여도 영향이 작다.

## 1. 목표·범위

`@ag-code-reviewer` 의 5축 순차 리뷰를 **병렬 하이브리드 fan-out** 으로
재구성해 (1) 큰 PR 에서 sampling (1500라인/30파일 초과 시 상위 10 hunk 만
검토, `agents/ag-code-reviewer.md:40`) 없이 전체를 커버하고 (2) 5축 동시
실행으로 체감 지연을 줄인다. 공식 plugin 의 "5 병렬 리뷰어 + confidence
필터" 패턴과 수렴하되, dp-skills 의 컨벤션·rubric·출력 컨트랙트를 그대로
유지한다.

### in scope

- 새 스킬 `dp-skills/skills/dp-code-review/SKILL.md` — 오케스트레이터
  (범위 수집 → fan-out → barrier → 병합/dedup/필터 → 단일 리포트)
- `agents/ag-code-reviewer.md` 에 스코프 인자 2개 추가: `--axis <목록>` ·
  `--files <목록>`. 인자 미지정 시 기존 동작 (전체 5축) 유지.
- README · `plugin.json` 버전 bump (0.14.0) · docs reference 색인 동기화

### out of scope

- `@ag-code-reviewer` 멘션 폐기·deprecated 처리 (공존 유지)
- discover·analyze 병렬화 (별도 PLAN — 본 작업은 code-review 만)
- 공식 `/code-review` 와의 출력 normalize 어댑터 (필요 시 후속)

## 2. worker 구현 방식 — M1 (인자 확장)

`@ag-code-reviewer` 에 스코프 인자 2개만 추가하고, 스킬이 이를 주어
병렬 N회 호출한다. 에이전트가 5축 로직의 단일 SSOT 로 유지된다.

- `--axis <comma-list>`: 지정 시 해당 축만 실행. 미지정 → 전체 5축 (= 오늘).
  - 슬러그: `cross-cutting` (축1) · `lang` (축2) · `hunk` (축3) ·
    `callers` (축4) · `missing` (축5).
- `--files <comma-list>`: 지정 시 diff 수집을 해당 파일로 한정. 미지정 →
  전체 변경 파일 (= 오늘).

검토 절차 §4 "검토 5축 실행" 에 1 블록 추가: "`--axis` 주어지면 나열된
축만, `--files` 주어지면 그 파일로 범위 한정. 둘 다 미지정이면 종전대로
전체." 나머지 (범위 수집·언어 감지·컨텍스트 로드·출력 포맷) 불변.

거부된 대안:
- (M2) 축별 worker 5개 신규 파일 → 5축 로직 중복·drift.
- (M3) 스킬에 프롬프트 인라인 → 에이전트 파일과 SSOT 이중화.

## 3. 하이브리드 fan-out 설계

### 3.1 축 분류

| 분류 | 축 | 이유 |
|---|---|---|
| **전역 (full diff 1회)** | 축1 cross-cutting · 축4 callers | 레이어 간·호출자 grep — 파일을 쪼개면 일관성/호환 검사가 깨짐 |
| **청크 (파일그룹 분할)** | 축2 lang · 축3 hunk · 축5 missing | 파일 로컬 판정 — 분할해도 정확성 유지, sampling 해소 |

### 3.2 worker 편성

한 assistant turn 에 여러 Agent 호출을 묶어 병렬 실행 (harness 가 동시 실행):

- worker A: `@ag-code-reviewer --axis cross-cutting --base <B>`
- worker B: `@ag-code-reviewer --axis callers --base <B>`
- worker C₁…C_G: `@ag-code-reviewer --axis lang,hunk,missing --files <group_i> --base <B>`

총 병렬 에이전트 = 2 (전역) + G (청크).

### 3.3 그룹 수 G 결정 (소규모 fast path)

- 변경 ≤ **30 파일 AND ≤ 1500 라인** → `G = 1`. 청크 worker 1개가 전체
  파일에 대해 축2·3·5 수행. (또는 전체가 매우 작으면 — 예: ≤ 3 파일 —
  스킬이 단일 all-axes worker 1개로 폴백해 spawn 오버헤드 회피. 선택을
  `log` 으로 남긴다.)
- 초과 → `G = ceil(변경파일수 / 12)`. 청크 크기 12 는 단일 에이전트가
  sampling 없이 다룰 수 있는 경험적 상한. (튜닝 가능 — `--chunk N` 인자
  후보, 초판은 고정 12.)

임계값 (30/1500) 은 기존 sampling 임계와 동일 — 그 이하는 단일 에이전트도
sampling 없이 처리하므로 굳이 잘게 쪼갤 이유가 없다.

## 4. barrier → 병합 (메인 루프)

1. 모든 worker 리포트 수집 (barrier — 전부 모인 뒤 진행).
2. **dedup**: `(path:line, severity)` 키로 중복 finding 제거. 전역 축과
   청크 축이 같은 라인을 짚을 수 있으므로 필수.
3. **confidence 필터**: `--threshold` (기본 80) 미만 dismiss.
   `scope_skipped` 에 `low-confidence: N filtered (threshold T)` 기재.
4. **verdict 산정** (필터 통과분만): `critical/major ≥1 → block` ·
   `minor/nit 만 → nit` · 빈 → `approve`. (기존 규칙 그대로)

## 5. 출력 — 기존 컨트랙트 그대로

단일 CODE REVIEW REPORT. 필수 필드 불변: `verdict · summary · base ·
scope_reviewed · scope_skipped · findings · next`.

- `scope_reviewed` 에 **"worker N개 / 그룹 G개 병렬 실행, sampling 없음"**
  명시 (no silent caps — 잘림 없음을 사용자에게 투명하게).
- 단일 all-axes 폴백 시 그 사실도 명시.
- `next` → verdict 별 `/dp-skills:fix-review` 라우팅 (기존과 동일).

## 6. 변경 파일

| 파일 | 작업 |
|---|---|
| `dp-skills/skills/dp-code-review/SKILL.md` | **신규** — 오케스트레이션 + 병합/dedup/필터 + 리포트 조립 |
| `dp-skills/agents/ag-code-reviewer.md` | `--axis`·`--files` 인자 2개 + "스코프 분기" 1 블록 추가 (기존 동작 보존) |
| `dp-skills/README.md` | 스킬 17→18종, "다른 도구와의 분리" 에 병렬/단일 구분 1줄 |
| `dp-skills/.claude-plugin/plugin.json` | version → 0.14.0 |
| docs reference 색인 | `python3 dp-skills/tools/docs_build.py` 재실행 (pre-push hook 강제) |

## 7. 미해결 질문

- 청크 크기 12 가 적정한가 — 초판 고정, 운영 후 `--chunk N` 노출 검토.
- 소규모 단일 폴백의 "매우 작음" 임계 (≤3 파일?) — 초판 보수적으로, 측정 후 조정.
- `dp-code-review` 호출명에 dp 중복 (`dp-skills:dp-`) — 사용자 명시 선택, 유지.
