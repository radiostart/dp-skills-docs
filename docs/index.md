---
hide:
  - navigation
  - toc
---

# dp-skills

DP팀 에이전트 오케스트레이션 플러그인 — Claude Code 안에서 *plan → critic → generate → evaluate* 의 명시적 사이클로 기획서 분석부터 코드 구현·리뷰까지 진행합니다.

!!! tip "v0.21.1"
    - `plan-validate.py` OQ 게이트 — 미해결 Open Questions 카테고리별 처리 마커 강제 (a–d)
    - `@ag-planner-critic` / `@ag-code-reviewer` 에이전트 정식 도입
    - 신규 가이드: [Plan 검증 게이트 통과](how-to/plan-gates.md) · [도메인 지식 환류](how-to/knowledge-sync.md)

---

## 처음이라면

<div class="grid cards" markdown>

-   :material-rocket-launch:{ .lg .middle } __Tutorial__

    ---

    설치부터 첫 plan 까지 — 한 사이클을 직접 돌려보면서 워크플로우의 모양을 익힙니다.

    [:octicons-arrow-right-24: Quick Start](tutorial/quick-start.md)

</div>

## 특정 작업이 필요할 때

<div class="grid cards" markdown>

-   :material-tools:{ .lg .middle } __How-to__

    ---

    팀 부트스트랩, TDD 활성화, 기획서 분석, 셀프 코드 리뷰 등 *작업별 레시피*.

    [:octicons-arrow-right-24: How-to 목록](how-to/index.md)

-   :material-book-open-variant:{ .lg .middle } __Reference__

    ---

    스킬·에이전트·CLI 도구의 *정확한 동작과 인자*.

    [:octicons-arrow-right-24: Reference 목록](reference/index.md)

-   :material-lightbulb-on:{ .lg .middle } __Explanation__

    ---

    스킬 vs 에이전트, workspace 레이아웃, 모드 — *막힐 때 보는 최소 개념*.

    [:octicons-arrow-right-24: Explanation 목록](explanation/index.md)

</div>

---

## 한 줄로

dp-skills 는 메인 대화의 의도와 도메인 컨텍스트를 잃지 않도록 `workspace/{TEAM}/` 메타구조와 분리된 에이전트(`ag-planner` / `ag-planner-critic` / `ag-generator` / `ag-evaluator` / `ag-code-reviewer`)를 통해 *명시 호출* 흐름을 강제합니다. 자동 파이프라인이 아니라 *phase 사이 사용자 개입이 가능* 한 워크플로우입니다.

[GitHub](https://github.com/dealicious-inc/deali-skills-plugin){ .md-button }
