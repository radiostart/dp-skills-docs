# `plan-validate.py`

> plan-validate — Planner 가 작성한 `features/NN-{slug}.plan.md` 의 형식 계약을
> 모드별로 검증한다.
>
> 스펙: skills/context/lifecycle/plan-schema.md
> Open Questions 게이트 스펙: skills/context/lifecycle/oq-gate.md
>
> 작동 흐름:
>   1. plan 파일 Read
>   2. 최상단 H2(`## 구현 계획: ...`) 존재 확인
>   3. 모드별 필수 H3 섹션 존재 확인
>   4. tdd/characterize: `### 스텝 목록[...]` 안의 각 스텝 항목별 필수 라벨 확인
>   5. Open Questions 게이트 — plan 경로에서 feature 파일을 유도해
>      미해결 OQ 카테고리별 plan 처리 마커(`추정 구현`/`범위 제외`) 존재 검증.
>      feature 파일·OQ 섹션 부재 시 skip (oq.checked=false)
>   6. 결과를 JSON 으로 stdout 출력 + 누락 요약을 stderr (exit 1 인 경우)
>
> Usage:
>     python3 plan-validate.py <plan_file> --mode {standard|tdd|characterize}
>
> Exit:
>     0 = valid
>     1 = invalid (검증 실패)
>     2 = 사용 오류 (파일 없음 / 인자 오류 등)

소스: [`tools/plan-validate.py`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/tools/plan-validate.py)
