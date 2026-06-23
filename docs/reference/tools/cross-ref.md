# `cross-ref.py`

> dp-skills cross-ref — features ↔ MCP search 결과 cross-reference 매핑.
>
> 워크플로:
>     1. /dp-skills:analyze 로 features/*.md 생성
>     2. Claude Code 에서 mcp__atlassian__search 호출 → 결과를 JSON 으로 저장
>        (스키마: {"query": "...", "results": [{"title", "url", "type", "text"}, ...]})
>     3. 본 도구 실행 → feature 별 관련 외부 링크 후보 출력
>     4. 사용자가 features 의 Open Questions (b) cross-domain 산출물 부재 에 후보 반영
>
> 매칭 알고리즘:
>     - feature 의 제목 + H2/H3 헤딩에서 키워드 추출
>     - search result 의 title + text 에서 동일 키워드 substring 매칭
>     - 매칭 키워드 수 기준 점수 (정규화: 0~1)
>
> Usage:
>     python3 tools/cross-ref.py \
>       --features workspace/{TEAM}/projects/{PROJECT}/features/ \
>       --search-results workspace/{TEAM}/projects/{PROJECT}/.search-results.json \
>       [--top-n 5] [--min-score 0.2] [--json]
>
> Exit:
>     0 — 정상 종료 (매칭 결과 무관)
>     1 — 입력 오류

소스: [`tools/cross-ref.py`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/tools/cross-ref.py)
