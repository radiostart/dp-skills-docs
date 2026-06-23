# `oq-eval.py`

> dp-skills oq-eval — Open Questions 분류 정확도 평가 도구 (Phase 2).
>
> 워크플로:
>     1. tests/fixtures/oq-synthetic/case-NN-*/spec.md 를 analyze 에 입력
>     2. analyze 결과로 생성된 features/NN-{slug}.md 를 case-NN-*/actual.md 로 복사
>     3. 본 스크립트 실행 → expected.yml vs actual.md 비교 → 정확도 표 출력
>
> 평가 기준:
>     - 기대 항목의 keyword 가 모두 actual 카테고리 항목 텍스트에 등장하면 match
>     - per-category precision / recall / F1
>     - false positive (negative case 에서 OQ 가 잡힌 경우)
>
> Usage:
>     python3 tools/oq-eval.py [CASES_DIR]
>     CASES_DIR 기본값: tests/fixtures/oq-synthetic/
>
> Exit:
>     0 — 평가 정상 종료 (정확도와 무관)
>     1 — fixture 경로 오류 등 치명 오류

소스: [`tools/oq-eval.py`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/tools/oq-eval.py)
