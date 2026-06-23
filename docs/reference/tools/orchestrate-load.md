# `orchestrate-load.py`

> dp-skills orchestrate-load — wrapper agents 용 컨텍스트 로드 의사결정.
>
> 래퍼 (@ag-planner / @ag-planner-critic / @ag-generator / @ag-evaluator) 가 프로젝트
> workspace 를 조사해서 어떤 파일을 Read 해야 하는지 결정하는 로직을 여기로 이관.
>
> 사이클 밖 에이전트 (@ag-code-reviewer) 는 본 도구를 사용하지 않는다.
> self-contained 형태로 자체 컨텍스트를 로드한다.
>
> 입력:
>     --phase {planner|planner-critic|generator|evaluator}
>     --workspace PATH              (default: ./workspace)
>     --project NAME                (optional — 미지정 시 STATE.md 진행중)
>
> 출력 (stdout, JSON):
>     {
>       "phase": "planner",
>       "team": "myteam",
>       "project": "MyProject",
>       "domain": "{도메인 식별자}" | null,
>       "analyzed": bool,
>       "tdd": bool,
>       "project_phase": "development" | "qa",
>       "focus": string | null,
>       "files_to_read": [...],   # 순서대로
>       "hints": [...],           # 자유 형식 힌트 (래퍼 프롬프트에 포함)
>       "error": string | null
>     }
>
> Exit:
>     0 — 성공
>     1 — 치명 오류 (error 필드 참조)

소스: [`tools/orchestrate-load.py`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/tools/orchestrate-load.py)
