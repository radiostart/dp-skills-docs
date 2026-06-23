# Confluence 동기화

!!! info "한 줄 요약"
    Confluence 기획서를 프로젝트 `docs/` 폴더에 저장하거나, 저장된 내용을 검색합니다.

## 언제 쓰나

- Confluence 기획서를 **프로젝트로 가져와** features 분석의 입력으로 쓰고 싶을 때
- 저장된 `docs/` 에서 **특정 내용을 검색** 하고 싶을 때

`docs/` 파일은 이 커맨드를 통해서만 접근합니다 — 직접 Read 하지 않습니다.

## 전제

Atlassian API 토큰이 필요합니다 (Jira 와 공용). 프로젝트 루트 또는 `workspace/` 에 `.env` 를 둡니다.

```bash
cat > .env <<'EOF'
ATLASSIAN_EMAIL=user@example.com
ATLASSIAN_TOKEN=your-api-token
EOF
echo ".env" >> .gitignore
```

토큰 발급: <https://id.atlassian.com/manage-profile/security/api-tokens>

> 기존 `CONFLUENCE_EMAIL`·`CONFLUENCE_TOKEN` 도 그대로 인식됩니다 (도구별 키 우선, 부재 시 `ATLASSIAN_*` fallback).

## 절차

### 페이지 저장

```text
/dp-skills:confl https://wiki.example.com/pages/12345
```

### 저장된 내용 검색

```text
/dp-skills:confl 배송상태
```

기본 검색은 **Atlassian Rovo MCP 의 시맨틱 검색을 먼저** 시도하고, MCP 미등록·호출 실패·결과 0건이면 **로컬 `docs/` 검색으로 폴백**합니다. 각 결과에는 출처 태그(`[source: rovo-mcp]` 또는 `[source: local]`)가 붙습니다. Rovo 결과는 요약·랭킹이 개입해 원문 인용 근거로는 부적합하니, 원문이 필요하면 `/dp-skills:confl {page_id}` 로 fetch 하세요.

### 로컬 강제 검색 (`--local`)

기획서 vs 구현 정책 점검처럼 **원문 인용·재현성** 이 필요하면 MCP 를 우회해 로컬만 검색합니다.

```text
/dp-skills:confl 배송상태 --local
```

### 검색 + 후속 작업

```text
/dp-skills:confl 배송상태 > project.md 요구사항 정리
```

### 전체 출력

```text
/dp-skills:confl all
```

## 흔한 실수

- **`.env` 와 shell `export` 값이 다르면** `/dp-skills:doctor` 가 credential drift WARN 을 냅니다. 둘 다 있으면 env var 가 우선합니다 — interactive/non-interactive shell 간 토큰 불일치로 401 이 조용히 나는 패턴을 피하세요.
- `confl` 은 `docs/` 에 저장만 합니다. `docs/` → `features/` 가공은 [`/dp-skills:analyze`](analyze-docs.md) 의 몫입니다.
- `tools/*.py` 는 표준 라이브러리만 씁니다 — Confluence fetch 에 별도 `pip install` 이 필요 없습니다.

## 다음 단계

- How-to: [기획서로 features 일괄 생성](analyze-docs.md)
- Reference: [`/dp-skills:confl`](../reference/skills/confl.md)
