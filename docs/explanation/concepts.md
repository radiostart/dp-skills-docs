# 핵심 개념

How-to 를 따라가다 헷갈리는 지점만 추렸습니다. 막힐 때 한 번 보고 마는 페이지입니다.

## 스킬 vs 에이전트

dp-skills 는 두 종류의 호출 단위를 씁니다.

| 종류 | 호출 | 역할 |
|---|---|---|
| **스킬** | `/dp-skills:{이름}` | 환경 세팅·데이터 준비 — 상태 파일 작성, 기획서 저장, 분석 |
| **에이전트** | `@{이름}` | 실제 개발 작업 — 계획·구현·검토. **별도 인스턴스** 로 실행 |

일반 흐름은 **스킬로 환경을 준비하고 → 에이전트로 작업을 수행** 하는 것입니다. 에이전트가 별도 인스턴스라는 점이 중요합니다 — 메인 대화의 내용은 에이전트에게 자동으로 전달되지 않습니다. 그래서 [`/dp-skills:focus`](../how-to/focus-direction.md) 같은 매개 장치가 필요합니다.

## `agents/` 폴더 2가지 (혼동 주의)

이름이 같아 헷갈리지만 성격이 전혀 다른 두 위치가 있습니다.

| 위치 | 정체 | `@ag-planner` 호출 시 | 편집 효과 |
|---|---|---|---|
| `${CLAUDE_PLUGIN_ROOT}/agents/ag-{phase}.md` | **Claude Code subagent 정의** (frontmatter `name:`·`tools:`) | 실제 실행되는 wrapper | 플러그인 업데이트로만 변경 |
| `workspace/{TEAM}/projects/{PROJECT}/agents/{phase}.md` | **프로젝트 컨텍스트 문서** (마크다운) | wrapper 가 Read 로 내용만 참고 | 다음 `@{phase}` 호출에 반영 |

프로젝트 쪽 `agents/*.md` 는 Claude Code subagent 레지스트리에 등록되지 **않습니다**. wrapper 가 진입 시 Read 로 불러들이는 *입력 자료* 일 뿐입니다.

- 프로젝트 `agents/*.md` 를 편집하면 다음 `@{phase}` 호출에 반영됩니다. 단 `<!-- [analyze-managed] -->` 주석이 붙은 섹션은 다음 `--regen-agents` 에서 덮어쓰입니다 → 커스텀은 주석 없는 섹션에 작성합니다.
- wrapper 의 **동작 자체**(단계 순서·tool 권한·model)를 바꾸려면 플러그인 수정이 필요합니다.

## state 파일 2가지

| 파일 | 레벨 | 내용 |
|---|---|---|
| `workspace/{TEAM}/STATE.md` | 팀 | **현재 활성 1행만** 유지. 이력은 `git log` 로 — `보류`·`완료` 행을 누적하지 않음 |
| `workspace/{TEAM}/projects/{PROJECT}/.agent-state.yml` | 프로젝트 | machine-readable 계약 (schema v1.3) — `schema`·`analyzed`·`tdd`·`domain`·`phase` 등 |

`.agent-state.yml` 은 machine-readable 계약이라 문자열 detection 으로 인한 drift 를 구조적으로 차단합니다. 특히 `analyzed` 필드는 **수동 편집 금지** — `/dp-skills:analyze` 가 유일한 정식 writer 입니다.

## 다음 단계

- Explanation: [에이전트 흐름](agent-flow.md) · [workspace 레이아웃](workspace-layout.md)
- Reference: [Skills 목록](../reference/skills/index.md) · [Agents 목록](../reference/agents/index.md)
