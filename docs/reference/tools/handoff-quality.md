# `handoff-quality.py`

> dp-skills handoff-quality — planner ↔ generator 인계 콘텐츠 품질 측정 (Phase β).
>
> plan-validate 가 검증하지 않는 영역 — **콘텐츠 품질** 정량 측정:
>
> - **변경 파일 구체성** (`### 변경 파일`): 각 항목이 실제 파일 경로 vs 모호 표현
> - **변경 사유 명시도**: 각 항목에 "왜" 가 있는가
> - **Red 계약 구체성** (TDD plan): 라벨 값이 placeholder vs 구체 텍스트
>
> Usage:
>     # 단일 plan 파일
>     python3 tools/handoff-quality.py path/to/03-foo.plan.md
>     python3 tools/handoff-quality.py path/to/03-foo.plan.md --mode tdd
>
>     # 디렉터리 일괄 (features/*.plan.md 모두)
>     python3 tools/handoff-quality.py workspace/team1/projects/proj1/features/
>
>     # JSON
>     python3 tools/handoff-quality.py path/to/plan.md --json
>
> Exit:
>     0 — 평가 정상 종료 (점수 무관)
>     1 — 입력 오류 (파일 없음 등)

소스: [`tools/handoff-quality.py`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/tools/handoff-quality.py)
