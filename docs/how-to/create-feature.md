# 기획서 없이 feature 추가

!!! info "한 줄 요약"
    원하는 기능을 프롬프트로 서술해 단일 기능 명세를 추가합니다. 기획서 문서 없이 바로 작업을 시작할 때 씁니다.

## 언제 쓰나

- 작은 기능·개선을 **기획서 없이** 바로 명세화하고 싶을 때
- 코드 리뷰 후 [`/dp-skills:fix-review`](self-code-review.md)가 `new-feature` 경로를 추천했을 때

여러 기능이 담긴 기획서를 일괄 분할하려면 [기획서로 features 일괄 생성](analyze-docs.md)을 쓰세요.

## 전제

- 활성 프로젝트가 있을 것

## 절차

### 1. 프롬프트로 명세 추가

```text
/dp-skills:create-feature "지연 주문 목록 UI"
```

`features/NN-{slug}.md` 가 prompt-origin 템플릿으로 생성됩니다. 그리고 `/dp-skills:analyze` 와 **동일한 절차** 로 `project.md`(목표·관련 파일)와 `agents/*.md`(planner·generator·evaluator)가 자동 동기화됩니다.

도메인을 지정하지 않으면 한 번 질의한 뒤 `.agent-state.yml` 에 기록합니다.

### 2. 구현 시작

```text
@ag-planner → (선택) @ag-planner-critic → @ag-generator → @ag-evaluator
```

## 흔한 실수

- create-feature 는 **자동 파이프라인이 아닙니다.** 명세 생성까지만 하고, 구현은 `@ag-planner` 부터 사용자가 명시 호출합니다.
- 기존 features 의 `[analyze-managed]` 섹션 내용은 보존됩니다 — create-feature 가 덮어쓰지 않습니다.

## 다음 단계

- How-to: [기획서로 features 일괄 생성](analyze-docs.md)
- Reference: [`/dp-skills:create-feature`](../reference/skills/create-feature.md)
