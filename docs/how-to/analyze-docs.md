# 기획서로 features 일괄 생성

!!! info "한 줄 요약"
    저장된 `docs/` 기획서를 `features/` 기능 명세로 분할하고, `project.md` 목표와 `agents/*.md` 를 자동 동기화합니다.

## 언제 쓰나

- Confluence 등에서 가져온 **기획서가 여러 기능을 담고** 있어 기능 단위로 쪼개야 할 때
- PM 작성 표 중심 기획서를 **구현 가능한 명세** 로 바꿔야 할 때

기획서 없이 프롬프트로 단건만 추가하려면 [기획서 없이 feature 추가](create-feature.md)를 쓰세요.

## 전제

- `docs/` 에 기획서가 저장돼 있을 것 — 없으면 먼저 [Confluence 동기화](confluence-sync.md)
- 활성 프로젝트가 있을 것

## 절차

### 1. 분석 실행

```text
/dp-skills:analyze
```

`docs/` → `features/NN-{slug}.md` 명세를 생성하고, `project.md` 의 목표 섹션과 `agents/*.md`(planner·generator·evaluator)를 자동 갱신합니다.

키워드·파일명으로 범위를 좁힐 수도 있습니다.

```text
/dp-skills:analyze 배송상태
```

#### 신규 도메인이면

도메인 확인 질의에서 미등재 도메인을 입력하면 analyze 를 중단하지 않고 그
자리에서 등재까지 진행됩니다 — 승인 후 MANIFEST 행과 `scope/{domain}.md` 시드
(features 기반 예정 항목) 가 생성되고, `rules/{domain}.md` 빈 스켈레톤은 없을
때만 함께 생성됩니다 (이미 있으면 건드리지 않습니다). 코드 진입점이 아직 없는
신규 서비스 (greenfield) 도 같은 경로입니다. 코드가 쌓이면
[도메인 스코프 추출](domain-discover.md) 재실행이 시드를 코드 기준으로
대체합니다.

### 2. 구현 시작

생성된 features 는 자동 구현되지 않습니다. `@ag-planner` 부터 명시 호출합니다.

```text
@ag-planner → (선택) @ag-planner-critic → @ag-generator → @ag-evaluator
```

## 흔한 실수

- **`--force` 는 기존 features 를 덮어씁니다.** prompt-origin features 가 감지되면 데이터 손실 방지를 위해 사용자 승인을 기다립니다 — 무심코 승인하지 마세요.
- `--regen-agents` 는 docs 변화가 없어도 `agents/*.md` 만 재작성합니다 (drift 대응). 자동 백업(`.agents.bak/{timestamp}/`) 후 중복 섹션을 post-check 합니다.
- analyze 는 `docs/` 를 입력으로 씁니다 — 기획서를 먼저 `/dp-skills:confl` 로 저장해야 합니다.

## 다음 단계

- How-to: [Confluence 동기화](confluence-sync.md) — `docs/` 채우기
- How-to: [기획서 없이 feature 추가](create-feature.md)
- Reference: [`/dp-skills:analyze`](../reference/skills/analyze.md)
