# 도메인 스코프 추출

!!! info "한 줄 요약"
    코드베이스 진입점을 받아 2홉을 추적하며 도메인 지식을 추출해 `scope/{domain}.md` 에 누적합니다.

## 언제 쓰나

- 신규 도메인의 **scope 문서를 코드에서 자동 생성** 하고 싶을 때
- 기존 도메인 scope 를 **보강·갱신** 하고 싶을 때

scope 파일은 모든 phase(planner/generator/evaluator)에서 항상 컨텍스트에 들어옵니다 — 에이전트가 도메인을 이해하는 1차 자료입니다.

!!! note "코드가 아직 없다면 (greenfield)"
    discover 는 코드 전제입니다 — 개발 전 신규 도메인은 `/dp-skills:analyze`
    진입 중 도메인 질의에서 즉석 등재됩니다 (`/dp-skills:create-feature` 도
    동일합니다 — [기획서로 features 일괄 생성](analyze-docs.md) 참고). 코드가
    쌓인 뒤 discover 를 실행하면 시드가 코드 기준으로 대체됩니다.

## 전제

- 활성 팀 workspace 가 있을 것
- `_common/config.md` 에 `source_root`·`test_path_convention` 이 채워져 있을 것 (언어 휴리스틱이 이 두 값만 의존)

## 절차

### 1. 진입점을 지정해 실행

컨트롤러·서비스 파일이나 폴더를 진입점으로 줍니다.

```text
/dp-skills:discover app/services/checkout checkout
```

대화형 4단계로 진행됩니다.

1. entry·domain 확정
2. 2홉 탐색
3. 추출 결과 미리 보기 → **승인**
4. 디스크 기록 — `scope/{domain}.md` + `MANIFEST.md` 도메인 행 추가/갱신

이때 `rules/{domain}.md` 가 없으면 **빈 스켈레톤도 함께 생성**되고 `MANIFEST.md` 의 `rules` 칸도 채워집니다. 정책 내용은 추출하지 않습니다 — 스켈레톤만 만들고 채우는 건 팀 몫입니다 ([도메인 rules 작성](write-domain-rules.md)). 이미 `rules/{domain}.md` 가 있으면 건드리지 않습니다.

### 2. (선택) 미리 보기만

```text
/dp-skills:discover app/services/checkout checkout --dry-run
```

`--dry-run` 은 디스크 기록과 승인 프롬프트를 모두 생략합니다.

### 3. (선택) 경계만 학습 — `--boundary`

외부 도메인 전체를 학습하기엔 비용이 크고, 실제로 필요한 것은 **내 도메인이 호출하는 표면**뿐인 경우:

```text
/dp-skills:discover --boundary schoice --from orders
```

`orders` 가 실제 호출하는 `schoice` 의 클래스·메서드 시그니처, 관찰된 상태값, 트랜잭션 중첩만 추출해 `workspace/{TEAM}/context/boundaries/orders--schoice.md` 를 생성합니다. 비용은 접점 크기에 비례합니다. `--from` 은 생략하면 활성 프로젝트의 도메인으로 추론됩니다.

**이 명령은 외울 필요가 없습니다.** 일반 discover 가 미학습 외부 도메인을 감지하면 (1) 실행 직후 경계 학습 여부를 선택지로 묻고, (2) "나중에" 를 골라도 MANIFEST 표와 다음 planner 호출 힌트가 복붙 가능한 완성 명령을 처방합니다.

생성된 경계 문서는 별도 등록 없이 자동으로 로드됩니다 — planner 호출 시 활성 도메인 기준 **정방향**(`{domain}--*.md`: 내가 호출하는 표면)과 **역방향**(`*--{domain}.md`: 다른 도메인이 나를 호출하는 표면 — 영향 분석용)을 모두 적재합니다 (상한 6건). 미학습 외부 의존이 MANIFEST 에 기록돼 있으면 planner 호출 시 boundary 처방이 힌트로 안내됩니다.

!!! note "수동 교차 도메인 표를 운영 중이라면"
    MANIFEST 에 "작업 도메인 X 면 `rules/Y.md` 추가 로드" 형식의 수동 교차 참조 표를 운영하는 팀이 있습니다. 역할이 다릅니다 — 수동 표는 **정책 (rules) 로드 안내**, 경계 문서는 **호출 표면 (시그니처·상태값·트랜잭션 중첩) 의 자동 로드**입니다. 둘은 공존 가능하며, 수동 표의 행이 다른 도메인의 코드 표면을 가리키고 있었다면 해당 쌍을 `--boundary` 로 추출해 이전하는 것을 권장합니다.

!!! tip "전체 discover vs boundary"
    경계 문서는 부분 커버입니다 — MANIFEST 의 `## 외부 도메인 reference` 행은 유지되며, 이후 해당 도메인을 전체 discover 하면 행이 제거되고 도메인 산출물 (`scope/{domain}.md`) 이 경계 문서보다 우선합니다.

## 흔한 실수

- **추측하지 않습니다.** 추출 항목마다 `(file:line)` 인용이 붙습니다 — 인용 없는 내용은 discover 의 결과가 아닙니다.
- 발견 파일이 50개를 넘으면 범위 축소·`--depth 1`·강행 중 선택을 묻습니다. 무작정 강행하면 컨텍스트가 비대해집니다.
- 사용자가 거부·취소하면 **어떤 파일도 기록되지 않습니다** (Abort cleanup 계약).
- 인덱스 파일이 180줄을 넘으면 `scope/{domain}/{section}.md` 로 자동 분할됩니다 — 분할된 하위 파일도 자동 로드되니 걱정하지 않아도 됩니다.

## 다음 단계

- How-to: [도메인 rules 작성](write-domain-rules.md) — `discover` 가 만든 `rules/{domain}.md` 스켈레톤 채우기
- How-to: [팀 workspace 부트스트랩](team-bootstrap.md)
- Reference: [`/dp-skills:discover`](../reference/skills/discover.md)
