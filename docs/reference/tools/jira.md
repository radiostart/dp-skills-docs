# `jira.py`

> Jira issue fetcher / commenter (QA phase 어댑터).
>
> `confluence.py` 와 같은 패턴을 따른다 — stdlib (`urllib.request` / `json` /
> `base64`) 만 사용, 외부 의존성 없음. T2 산출물 (QA phase 도입 계획 §7 Step 2).
>
> Usage:
>   python3 tools/jira.py fetch <KEY>            # GET /rest/api/3/issue/{KEY}
>   python3 tools/jira.py comment <KEY> "<msg>"  # POST /rest/api/3/issue/{KEY}/comment
>
> `list` 명령은 의도적으로 미구현 — 다건 추적은 Jira UI 가 SSOT (PLAN-qa-phase §0·§1).
>
> Required env vars (둘 중 하나로 설정):
>   1) 프로젝트 루트 `.env` 또는 `workspace/.env` 파일에 기록
>   2) shell profile 에서 `export`
>
>   ATLASSIAN_BASE_URL  — Atlassian Cloud 루트 URL (예: https://dealicious.atlassian.net)
>                         Jira 와 Confluence 가 같은 사이트라면 본 키 하나로 양쪽 도구가 공용.
>   ATLASSIAN_EMAIL     — Atlassian 계정 이메일
>   ATLASSIAN_TOKEN     — Atlassian API 토큰 (https://id.atlassian.com/manage-profile/security/api-tokens)
>
>   도구별 키 (JIRA_BASE_URL / JIRA_EMAIL / JIRA_TOKEN) 가 설정돼 있으면 그쪽이 우선
>   — 기존 .env 와 backward compatible. 둘 다 부재 시 ATLASSIAN_* 권장 메시지를 안내.
>
> Optional env var:
>   JIRA_QA_PROJECT_KEY — 키 화이트리스트 (예: QAPRJ). 미설정 시 모든 `^[A-Z]+-\d+$` 허용.
>                        `_common/config.md` 의 `jira_qa_project_key:` 라인이 우선.
>
> TEAM 해석:
>   단일 팀 설치 — `workspace/` 직속 단일 팀 폴더 자동 사용.
>   다중 팀 설치 — env `DP_TEAM` 으로 지정. (`confluence.py` 의 `get_team()` 과 달리 본
>   스크립트는 화이트리스트 조회 외 TEAM 의존성 없음 — 해석 실패해도 fetch/comment 는 동작.)
>
> Exit codes:
>   0 — 성공
>   1 — 인자/환경/네트워크/HTTP/사용자 거절 등 모든 실패. comment 실패는 stderr 첫 줄에
>       `[ERROR] COMMENT FAILED for {KEY}` 헤더 강제 출력.

소스: [`tools/jira.py`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/tools/jira.py)
