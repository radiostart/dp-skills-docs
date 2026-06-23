# `confluence.py`

> Confluence docs fetcher / searcher.
>
> Usage:
>   python confluence.py fetch <url_or_page_id> [--force]
>   python confluence.py search <keyword>
>   python confluence.py search-local <keyword>      # search 의 alias (SKILL.md 와 정합)
>   python confluence.py all
>
> Flags:
>   --force          기존 저장 파일이 있어도 덮어쓴다 (fetch 모드 한정)
>
> Required env vars (둘 중 하나로 설정):
>   1) 프로젝트 루트 `.env` 또는 `workspace/.env` 파일에 기록
>   2) shell profile에서 `export`
>
>   ATLASSIAN_EMAIL     — Atlassian 계정 이메일 (예: user@example.com)
>                         Jira (`tools/jira.py`) 와 공용 — 같은 토큰이 양쪽에 사용됨.
>   ATLASSIAN_TOKEN     — Atlassian API 토큰 (https://id.atlassian.com/manage-profile/security/api-tokens)
>
> Optional env var:
>   ATLASSIAN_BASE_URL  — Atlassian Cloud 루트 URL (예: https://dealicious.atlassian.net).
>                         설정 시 `{BASE_URL}/wiki` 가 Confluence host 로 사용된다.
>                         미설정 시 https://dealicious.atlassian.net/wiki 가 fallback.
>
> 도구별 키 (CONFLUENCE_EMAIL / CONFLUENCE_TOKEN) 가 설정돼 있으면 그쪽이 우선 —
> 기존 .env 와 backward compatible.

소스: [`tools/confluence.py`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/tools/confluence.py)
