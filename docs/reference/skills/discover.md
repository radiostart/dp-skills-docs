# `/dp-skills:discover`

> 코드베이스 진입점(controller·service·폴더)을 받아 도메인 지식을 추출해 `workspace/{TEAM}/context/scope/{domain}.md` 와 `MANIFEST.md` 에 누적할 때 사용한다. `rules/{domain}.md` 가 없으면 빈 스켈레톤도 함께 만든다. 신규 도메인 스코프 작성·기존 도메인 보강·기존 도메인 갱신 모두 본 스킬로 수행한다. 기존 scope-exploration.md 가 소비 규칙이라면 본 스킬이 생산자 역할을 담당한다. `--boundary B --from A` 는 외부 도메인 B 전체 대신 A 가 호출하는 표면만 `boundaries/{A}--{B}.md` 로 포착한다 (cross-domain 경량 학습).

## 역할

코드에서 도메인 지식을 추출하는 테크니컬 라이터. 코드에 문자 그대로 있는 것만 기록하고, 추측은 하지 않는다. 확인되지 않은 영역은 빈칸으로 남기거나 Open Questions 로 표시한다.

---

# /dp-skills:discover

코드베이스 진입점을 2홉 추적하여 도메인 지식을 추출하고 `scope/{domain}.md`·`MANIFEST.md` 에 기록한다. 도메인의 `rules/{domain}.md` 가 없으면 빈 스켈레톤도 함께 만든다.

대상: $ARGUMENTS

---

## 사전 확인

### 1. 인자 파싱

`$ARGUMENTS` 에서 아래 순서로 파싱한다:

```
<entry> [domain] [--team <TEAM>] [--depth N] [--dry-run]
--boundary <B> --from <A> [--team <TEAM>] [--dry-run]   # 경계 계약 모드 (entry 생략)
```

- `entry` (필수): 진입점 경로 (컨트롤러·서비스·폴더). 누락 시 아래 안내 후 종료:

  ```
  사용법: /dp-skills:discover <entry> [domain] [--team TEAM] [--depth N] [--dry-run]
  예시:   /dp-skills:discover app/controllers/orders_controller.rb order
  ```

- `domain`: 누락 시 entry 경로·클래스명에서 추론 후 사용자 확정.
- `--team <TEAM>`:
  - 미지정 시 `workspace/TEAM.md` 를 읽는다.
  - `workspace/` 하위에 팀 폴더가 여러 개 존재하면 (예: `fin-team/`·`retail-team/` 동시) `--team` 명시를 강제 요구하고, 미지정이면 사용 가능한 팀 목록을 안내 후 종료한다.
  - `--team {TEAM}` 으로 명시한 팀 폴더가 없으면 `team-init` 안내 후 종료.
- `--depth N`: 기본값 2.
- `--dry-run`: 디스크 기록 없이 추출 결과만 출력.
- `--boundary <B> --from <A>`: 경계 계약 모드 — `<A>` (학습된 도메인) 가 호출하는 `<B>` (미학습 외부 도메인) 표면만 추출. entry 는 생략한다 (지정 시 에러 + 사용법 안내). 상세: 아래 `## Boundary 모드` 섹션.
  - `--from` 생략 시 활성 프로젝트의 도메인을 기본값으로 추론한다 — `workspace/{TEAM}/STATE.md` 의 진행중 프로젝트 → `projects/{P}/project.md` 의 `domain:` 키 (orchestrate-load 의 도메인 판정과 동일 소스). 추론값은 사용자에게 확정받는다. 추론 불가 (활성 프로젝트 없음·`domain:` 키 없음·진행중 복수) 시 MANIFEST `## 도메인 분류` 의 도메인 목록을 제시하고 질의.
  - `<A>`·`<B>` 는 영숫자·하이픈·언더스코어·`/`(계층 식별자) 만 허용, 소문자화, 연속 하이픈은 단일로 축약 (`--` 구분자 모호성 방지). `..`·절대 경로 거부 (기존 entry 검증과 동일 원칙), `_`/`.` 선행 문자 거부 (`_` prefix·dotfile 은 도메인 후보 자동 제외 대상이므로 — 아래 `## 출력 구조 결정 규칙`).

### 2. workspace·TEAM 검증

- CWD 기준 `./workspace/{TEAM}/` 폴더 존재 여부 확인.
- 없으면: **"`workspace/{TEAM}/` 가 없습니다. `/dp-skills:team-init {TEAM}` 을 먼저 실행하세요."** 안내 후 종료.

### 3. `_common/config.md` 키 검증

`workspace/_common/config.md` 를 Read 하여 `source_root`·`test_path_convention` 값을 확인한다.

- 파일이 없거나 두 키가 모두 미정의이면: **"`workspace/_common/config.md` 의 `source_root` 와 `test_path_convention` 이 필요합니다. 파일을 먼저 채우세요."** 안내 후 종료.
- 키가 있으면 이후 탐색에서 경로 치환에 사용한다. 언어는 하드코드하지 않는다.

### 4. entry 존재 검증

`entry` 경로가 `{source_root}` 하위에 있는지, 실제로 읽을 수 있는지 확인한다. 없으면 사용자에게 알리고 경로를 재확인 요청.

- 절대 경로·`..` 포함 경로·심볼릭 링크는 거부한다. workspace 외부·CWD 외부 경로 차단을 위해 `_common/config.md` 의 `source_root` 하위 상대 경로만 허용.

### 5. MANIFEST.md 존재 검증

`workspace/{TEAM}/context/MANIFEST.md` 가 없으면: [messages.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/messages.md) 의 `manifest_missing` 출력 후 종료. discover 는 MANIFEST 색인 자체를 갱신하는 **생산자 스킬**이라 색인 없는 곳에 누적할 수 없다 ([INDEX.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/INDEX.md) § MANIFEST 부재 정책).

---

## 사용자 중단 처리 (Abort cleanup 계약)

단계 1~3 의 어떤 사용자 확인 게이트 (도메인 확정·범위 cap 안내·미리 보기 승인) 에서든 사용자가 거부·취소하면 **어떤 Write 도 수행하지 않는다** (메모리 폐기). `MANIFEST.md`·`scope/`·`rules/` 파일은 변경 전 상태 그대로 유지된다. 단계 4 (디스크 기록) 진입 후 batch Write 가 끝난 상태에서는 abort 불가 — 이 시점부터는 `git diff` / revert 로 처리한다.

---

## 동작

### 단계 1 — domain 확정

`domain` 인자가 있으면 그대로 사용. 없으면:

1. entry 경로·파일명·내부 클래스명에서 도메인 후보를 추론한다.
2. `workspace/{TEAM}/context/MANIFEST.md` 의 `## 도메인 분류` 표와 대조하여 기존 도메인 중 일치 여부 확인.
3. 사용자에게 후보를 제시하고 확정을 요청한다:

   ```
   추론 후보: {후보} (근거: {entry 경로·클래스명 등 한 줄})
   기존 도메인: {MANIFEST 의 도메인 목록}
   도메인명을 확정해주세요 (직접 입력 가능):
   ```

### 단계 2 — 2홉 탐색

상세 알고리즘: [`references/extraction.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/discover/references/extraction.md).

- entry 를 0홉으로 두고, 참조 그래프를 따라 `--depth` (기본 2) 홉 이내 파일만 수집한다.
- `Explore` 서브에이전트 호출 시 prompt 에 허용 경로 패턴을 반드시 명시한다 (`{source_root}/**` 등). 무차별 `**/*` 스캔은 금지.
- 수집 카테고리: 라우트/엔드포인트, 컨트롤러·서비스·도메인 모델·리포지토리, 외부 의존(API·MQ·DB), 도메인 enum/상수, 비즈니스 용어.
- 탐색 중 발견된 클래스/모듈 reference 를 내부 vs 외부 도메인으로 분류해 외부 후보를 메모리에 누적한다 (단계 4 의 MANIFEST `## 외부 도메인 reference` 표 기록용). 분류·ignore 패턴: [`references/cross-domain.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/discover/references/cross-domain.md) § 외부 도메인 reference detect.

### 단계 3 — 미리 보기 및 승인

추출 결과를 사용자에게 표로 제시하고 승인을 받는다. 승인 없이 디스크에 기록하지 않는다. 미리 보기 표·기록 파일 목록 형식: [`references/output-format.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/discover/references/output-format.md).

거부·취소 시 어떤 파일도 쓰지 않고 종료한다 (위 **Abort cleanup 계약**).

`--dry-run` 인 경우 단계 3 의 미리 보기 표를 출력한 직후 종료한다. 승인 프롬프트 (y/n) 는 띄우지 않으며 단계 4 (디스크 기록) 으로 진입하지 않는다.

### 단계 4 — 디스크 기록

승인 후:

1. `scope/{domain}.md` 작성 — 출력 구조 결정 규칙 및 200줄 제약 적용. 상세: [`references/split-policy.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/discover/references/split-policy.md).
2. `rules/{domain}.md` 스켈레톤 생성 — **파일이 없을 때만**. 정책은 코드에서 추론할 수 없으므로 내용은 추출하지 않고 빈 스켈레톤만 쓴다 (식별자는 scope 와 동일 — 계층 식별자면 `rules/{part}/{domain}.md`). 스켈레톤 본문: [`references/output-format.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/discover/references/output-format.md).

   이미 `rules/{domain}.md` 가 있으면 **건드리지 않는다** (팀이 채운 내용 보존). 추출한 도메인 지식을 rules 에 쓰지 않는다 — scope 와 rules 의 경계.
3. `MANIFEST.md` 도메인 표 갱신 — scope·rules 컬럼 모두. 상세: [`references/manifest-update.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/discover/references/manifest-update.md).
4. `MANIFEST.md` `## 외부 도메인 reference` 표 갱신 — detect 결과가 1 건 이상일 때만. 현재 도메인의 stale row 제거 (idempotency). 상세: [`references/cross-domain.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/discover/references/cross-domain.md) § MANIFEST 표 갱신.
5. 재실행 시 `scope`·`MANIFEST` 는 단순 덮어쓰기, `rules` 는 부재 시에만 생성. 충돌은 사용자가 `git diff` 로 처리.

---

## Boundary 모드 — `--boundary {B} --from {A}`

외부 도메인 `{B}` 를 전체 학습하지 않고, **학습된 도메인 `{A}` 가 실제 호출하는 `{B}` 의 표면만** 포착해 `workspace/{TEAM}/context/boundaries/{A}--{B}.md` 를 생성한다. cross-domain feature 에서 `{B}` 전체 discover 의 경량 대안 — 비용은 O(접점 크기).

**전제 검증** (사전 확인 1~3·5 와 동일 + 추가):

- `{A}` 가 MANIFEST `## 도메인 분류` 에 등록돼 있어야 한다. 미등록이면 에러 + `/dp-skills:discover {진입점} {A}` 안내 후 종료.
- `{B}` 가 이미 `## 도메인 분류` 에 등록돼 있으면 안내 후 종료 — 이미 학습된 도메인은 scope 가 우선.

**절차** — 3 단계. 상세: [`references/cross-domain.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/discover/references/cross-domain.md) § Boundary 모드.

1. **호출처 수집** — `{A}` 소스에서 `{B}` reference Grep (패턴 소스 우선순위는 상세 참조). 0 건이면 "경계 없음" 보고 후 종료 — 파일 생성 안 함.
2. **표면 추출** — 호출처 ±10 줄 Read + `{B}` 정의 파일은 **호출된 심볼만** Targeted Read. 전체 학습 금지.
3. **미리 보기 → 생성 + 색인** — 단계 3 미리 보기·승인 게이트·Abort 계약·`--dry-run` 을 동일 적용. 승인 후 `boundaries/{A}--{B}.md` Write (본문 ≤ 150 줄) → MANIFEST 외부 reference 표의 `{B}` 행 추천 컬럼에 공백 + `· 경계: {A}--{B}.md` 표기 (행 제거 금지 — 전체 discover 완료가 아님).

**로드 배선** — orchestrate-load 가 활성 도메인 기준 `boundaries/{domain}--*.md` (정방향: 내가 호출하는 표면) 와 `*--{domain}.md` (역방향: 남이 나를 호출하는 표면 — 영향 분석) 를 자동 로드한다 (상한 6). 별도 MANIFEST 등록 불필요. 계층 식별자는 파일명 토큰에서 `/`→`__` 치환.

---

## 출력 구조 결정 규칙

`workspace/{TEAM}/context/scope/` 하위에 다른 도메인 파일이 이미 있으면 그 섹션·표 형식을 모사한다 (기존 도메인과의 일관성 우선). 없으면 코드 구조에 맞춰 자동 결정 — 결정 표·진입 파일 결정·edge case·anti-pattern 상세는 [`references/split-policy.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/discover/references/split-policy.md) 의 `## 구조 결정 휴리스틱`.

**식별자 형식** — `{domain}` 자리에는 두 형식 모두 사용 가능:

- **flat**: `scope/{domain}.md` (예: `scope/orders.md` → 식별자 `orders`)
- **계층**: `scope/{part}/{domain}.md` (예: `scope/backend/orders.md` → 식별자 `backend/orders`) — multi-part 모놀리식 (backend/frontend/lambda 등) 에서 권장. 기존 형제 파일이 계층이면 그 컨벤션을 따른다.

`_archive/`, `_draft/` 같은 `_` prefix 폴더·파일과 `.dotfile` 은 도메인 후보에서 자동 제외 (임시·아카이브용).

---

## 결과 출력

기록된 파일·추출 요약·다음 단계 안내를 출력한다. 분할이 발생한 경우 분할된 파일 목록 안내를 덧붙인다. 출력 형식 (완료 요약·분할 발생 시 안내): [`references/output-format.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/discover/references/output-format.md).

**미학습 외부 도메인 감지 시 후속 게이트**: 단계 4 에서 `## 외부 도메인 reference` 표에 기록·갱신한 행이 1건 이상이면, 결과 출력 직후 경계 학습 여부를 옵션 선택으로 묻는다. 선택 시 같은 세션에서 `## Boundary 모드` 절차로 이어 진행 (도메인별 반복 가능, 각 추출은 미리 보기 승인 게이트 동일 적용), "나중에" 선택 시 표 기록 상태로 종료 — 이후 planner 호출 시 처방 힌트가 재안내한다. 이미 경계 문서가 있는 쌍은 `경계 있음` 표기·`재추출` 라벨로 구분하고, **전부 커버된 경우 게이트를 생략**한다 (존재 안내 1 줄로 대체 — orchestrate-load 의 covered 억제와 동일 기준). 프롬프트 형식: [`references/output-format.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/discover/references/output-format.md) § 외부 도메인 감지 후속 게이트.

---

## 참고

- 본 스킬이 생성하는 `scope/{domain}.md` 는 이후 `@ag-planner`·`@ag-generator`·`@ag-evaluator` 가 직접 Read 하여 탐색 범위를 결정한다 ([scope-exploration.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/domain/scope-exploration.md) 참조).
- 추출 휴리스틱은 언어를 하드코드하지 않는다. `_common/config.md` 의 `source_root`·`test_path_convention` 값으로 경로 패턴을 구성한다.
- 200줄 제약 및 분할 정책 상세는 [`references/split-policy.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/discover/references/split-policy.md).
- MANIFEST 갱신 규칙 상세는 [`references/manifest-update.md`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/discover/references/manifest-update.md).
- 재실행 시 `scope`·`MANIFEST` 는 머지 로직 없이 단순 덮어쓰기. 충돌은
  `git diff` 로 확인. `rules/{domain}.md` 만 예외 — 부재 시에만 스켈레톤
  생성, 존재하면 보존.
- 코드 진입점이 없는 신규 도메인 (greenfield) 의 scope 시드는
  `/dp-skills:analyze` 의 즉석 등재가 생성한다
  ([domain-decision.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/analyze/references/domain-decision.md)
  § 신규 도메인 즉석 등재). 본 스킬을 실행하면 시드가 코드 기준으로 대체된다.
