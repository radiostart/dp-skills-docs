# `regen-verify.py`

> dp-skills regen-verify — `--regen-agents` 후 수동 편집 영역 보존 검증.
>
> `agents/*.md` 의 `<!-- [analyze-managed] -->` 주석이 달린 섹션은 regen 으로
> 덮어쓰여도 정상. 그 외 섹션 (사용자 수동 편집 영역) 이 변경됐다면 위반.
>
> 본 도구는 regen 전 백업 (`.agents.bak/{timestamp}/`) 과 현재 `agents/` 의
> non-managed 섹션을 비교하여 변경 라인을 보고한다.
>
> Usage:
>     python3 tools/regen-verify.py \
>       --before workspace/{TEAM}/projects/{PROJECT}/.agents.bak/{timestamp}/ \
>       --after  workspace/{TEAM}/projects/{PROJECT}/agents/
>
>     --json 으로 JSON 출력 가능.
>
> Exit:
>     0 — 보존 영역 변경 없음 (안전)
>     1 — 보존 영역 변경 감지 (사용자 검토 필요)
>     2 — 입력 오류 (디렉터리 없음 등)

소스: [`tools/regen-verify.py`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/tools/regen-verify.py)
