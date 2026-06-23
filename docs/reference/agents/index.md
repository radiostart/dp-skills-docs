# Agents

전체 5 항목 — `dp-skills/` SSOT 에서 자동 추출.

- [`ag-code-reviewer`](ag-code-reviewer.md) — PR 직전 셀프 코드 리뷰어. 누적 변경의 cross-cutting 일관성·언어 규약·hunk 디테일·호출자 영향·누락 마무리를 본다. evaluator 책임 영역 (요구사항·TDD 증거·테스트 실행) 은 절대 다루지 않는다. 파일 수정 권한 없음 — 보고만.
- [`ag-evaluator`](ag-evaluator.md) — 구현 완료 후 요구사항 충족 여부와 코드 일관성을 검토한다.
- [`ag-generator`](ag-generator.md) — 코드를 구현한다. Planner 계획 확정 후 실행. 패턴·서비스·모델 참조해 일관성 있게 작성.
- [`ag-planner-critic`](ag-planner-critic.md) — Planner 가 작성한 plan.md 를 adversarial 시각으로 챌린지한다. plan.md 를 직접 수정하지 않고 별도 `.plan.critic.md` 에 챌린지를 기록한다. Planner 와 Generator 사이에서 선택적으로 호출.
- [`ag-planner`](ag-planner.md) — 새 기능 구현 시작 시 구현 계획을 수립한다. 요구사항 분석, 영향 범위 파악, 단계별 계획 작성.
