# Tools 위생 묶음 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** HTTP 타임아웃 환경변수화 + 테스트 공백 2건(credential drift·certifi fallback) 보충 + dead tool 아카이브 표기를 한 PR 로 처리.

**Architecture:** `_net.py` 에 `http_timeout(env_key, default)` 공용 헬퍼 추가 — confluence(`ATLASSIAN_HTTP_TIMEOUT`, 기본 30)·jira(동일)·slack(`SLACK_HTTP_TIMEOUT`, 기본 3)이 소비. 비정상 env 값은 default 폴백. `migrate-manifest-split.py` 는 회귀 테스트(test_tools_review_fixes.py)가 지키고 있어 삭제 대신 docstring ARCHIVED 표기.

**Tech Stack:** Python 표준 라이브러리, unittest.

---

### Task 1: http_timeout 헬퍼 + 3개 도구 적용 (TDD)

**Files:**
- Modify: `dp-skills/tools/_net.py`, `confluence.py:317`, `jira.py:382`, `slack-notify.py:41,46`
- Test: `dp-skills/tests/tools/test_net_ssl.py` (HttpTimeout 클래스 추가)

- [ ] 테스트: env 미설정→default, 유효값→float 파싱, 비수치→default, 0 이하→default.
- [ ] 구현: `_net.http_timeout`; confluence `timeout=http_timeout("ATLASSIAN_HTTP_TIMEOUT", 30)`; jira `timeout: float | None = None` + None 시 동일 헬퍼 해석; slack `HTTP_TIMEOUT_SEC = http_timeout("SLACK_HTTP_TIMEOUT", 3)`.
- [ ] 전체 회귀 + Commit.

### Task 2: certifi fallback 테스트 확대

- [ ] `test_net_ssl.py`: ① `sys.modules["certifi"]=None` 패치로 import 실패 → stdlib 기본 컨텍스트로 폴백·무예외, ② SSL_CERT_FILE 설정 시 certifi.where() 미호출(MagicMock 검증).

### Task 3: credential drift 테스트 신규

- [ ] `tests/tools/test_doctor_credential_drift.py`: 동일값→무경고, 상이→WARN+원문 미노출(sha 프리픽스만), 한쪽만 존재→skip, .env 부재→빈 리스트. 대상: `doctor/integrity.py check_credential_drift`.

### Task 4: dead tool 아카이브 + bump 0.16.13 + PR

- [ ] `migrate-manifest-split.py` docstring 에 `[ARCHIVED]` 표기 (v0.3.0 일회성, 신규 사용 비권장, 회귀 테스트 가드용 보존).
- [ ] docs_build + 전체 pytest + bump + PR (base main, squash).
