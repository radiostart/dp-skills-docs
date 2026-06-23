# `docs_build.py`

> dp-skills docs_build — agents/skills/tools SSOT 를 docs/reference/ 로 추출.
>
> 매뉴얼 사이트(MkDocs)의 Reference 섹션을 플러그인 소스에서 자동 생성한다.
> 표준 라이브러리만 사용한다 (pre-push 훅이 시스템 python 으로 테스트를 돌리므로
> 외부 의존성 금지 — 다른 tools/*.py 와 동일한 컨트랙트).
>
> 호출:
>     python3 dp-skills/tools/docs_build.py            # 생성 (덮어쓰기)
>     python3 dp-skills/tools/docs_build.py --check    # 디스크 상태가 source 와 일치하는지 검증
>     python3 dp-skills/tools/docs_build.py --root PATH # 다른 dp-skills 디렉토리 대상
>
> 입력:
>     {root}/agents/*.md
>     {root}/skills/*/SKILL.md
>     {root}/tools/*.py
>
> 출력 (git 추적 — docs_build 로 재생성·커밋, SSOT 는 입력 소스):
>     {root}/docs/reference/agents/{name}.md
>     {root}/docs/reference/skills/{name}.md
>     {root}/docs/reference/tools/{tool_stem}.md
>
> 부수 동기화:
>     {root}/mkdocs.yml  — extra.version 을 .claude-plugin/plugin.json 의 version 으로 스탬프
>
> Exit:
>     0 — 성공
>     1 — --check 모드에서 drift 발견 또는 root 디렉토리 누락

소스: [`tools/docs_build.py`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/tools/docs_build.py)
