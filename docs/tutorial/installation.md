# 설치 및 초기 세팅

dp-skills 를 설치하고 팀 workspace 를 준비하는 과정입니다. 한 번만 하면 됩니다.

## 1. 플러그인 등록

Claude Code 의 marketplace 기반 플러그인 시스템을 사용합니다. dp-skills 는 사내 공용 멀티플러그인 레포(`dealicious-inc/deali-skills-plugin`)의 `dp-skills/` 폴더로 배포됩니다.

```text
/plugin marketplace add dealicious-inc/deali-skills-plugin
/plugin install dp-skills@deali-skills
```

- `marketplace add` — GitHub 레포를 marketplace 로 등록. 레포 루트의 `.claude-plugin/marketplace.json` 을 읽습니다. 로컬 개발 중이면 경로도 허용됩니다.
- `plugin install` — `{플러그인명}@{마켓플레이스명}` 형태. 마켓플레이스 이름은 `deali-skills` 입니다.

설치 후 Claude Code 재시작(또는 `/plugin reload`) 시 에이전트·스킬·훅이 자동 등록됩니다.

**확인** — slash 커맨드 자동완성에 `/dp-skills:project`·`/dp-skills:team-init` 이 노출되고 `@ag-planner`·`@ag-generator`·`@ag-evaluator` subagent 호출이 가능하면 성공입니다.

업데이트는 `/plugin marketplace update deali-skills` → `/plugin update dp-skills@deali-skills`.

## 2. `/plugin` 이 막힌 환경 — `dp-skills-update`

IDE 내장 세션·관리형 환경에서 `/plugin` 이 `isn't available in this environment.` 로 차단될 때, 캐시를 직접 fast-forward 하는 헬퍼 스크립트를 씁니다.

설치 (1회, `~/.zshrc` 또는 `~/.bashrc` 에 추가):

```bash
alias dp-skills-update='bash ~/.claude/plugins/marketplaces/deali-skills/dp-skills/tools/dp-skills-update.sh'
```

사용:

```bash
dp-skills-update           # 원격 main 으로 fast-forward + 버전 비교
dp-skills-update --check   # pull 하지 않고 업데이트 유무만 확인
```

!!! warning "세션 재시작 필요"
    실행 후 열려 있는 Claude Code 세션은 구버전 프롬프트를 이미 로드한 상태입니다 — 새 내용을 반영하려면 세션을 재시작하세요.

## 3. 팀 workspace 부트스트랩

```text
/dp-skills:team-init myteam
```

`workspace/` 아래에 `TEAM.md`·`_common/config.md`·`{팀}/STATE.md`·`{팀}/context/` 구조가 생성됩니다. 자세한 내용과 MANIFEST 채우기는 [팀 workspace 부트스트랩](../how-to/team-bootstrap.md)을 참고하세요.

부트스트랩 직후 `_common/config.md` 의 언어·테스트 러너 등 toolchain 기본값을 채워야 TDD·evaluator 가 동작합니다 — [toolchain·언어 설정](../how-to/configure-toolchain.md) 참고.

## 4. 환경변수 (Atlassian — Confluence / Jira 공용)

`/dp-skills:confl` (Confluence) 과 `/dp-skills:qa` (Jira) 는 같은 Atlassian 토큰을 공유합니다. 프로젝트 루트 또는 `workspace/` 에 `.env` 를 둡니다.

```bash
cat > .env <<'EOF'
ATLASSIAN_EMAIL=user@example.com
ATLASSIAN_TOKEN=your-api-token
# (선택) Atlassian Cloud 루트 — 미설정 시 https://dealicious.atlassian.net 가 fallback
# ATLASSIAN_BASE_URL=https://dealicious.atlassian.net
EOF
echo ".env" >> .gitignore
```

토큰 발급: <https://id.atlassian.com/manage-profile/security/api-tokens>

!!! info "backward compatibility"
    기존 `.env` 의 `CONFLUENCE_EMAIL`/`CONFLUENCE_TOKEN`·`JIRA_BASE_URL`/`JIRA_EMAIL`/`JIRA_TOKEN` 도 그대로 인식됩니다. 항목별로 도구별 키 (`JIRA_*`, `CONFLUENCE_*`) 가 먼저 조회되고, 비어있으면 `ATLASSIAN_*` 가 fallback. 신규 셋업은 `ATLASSIAN_*` 단일 세트를 권장.

!!! note "Python 의존성 없음"
    모든 `tools/*.py` 는 표준 라이브러리만 사용합니다 (Confluence fetch 포함). 별도 `pip install` 이 필요 없습니다.

## 다음 단계

- Tutorial: [Quick Start](quick-start.md) — 5분 만에 한 사이클 돌려보기
- Tutorial: [Deep Walkthrough](getting-started.md) — 전체 흐름 따라가기
