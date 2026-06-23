# Quick Start

!!! info "한 줄 요약"
    플러그인 설치 → 팀 workspace 부트스트랩 → 프로젝트 시작 → 첫 plan 까지. 5 분이면 dp-skills 의 한 사이클을 직접 돌려볼 수 있습니다.

## 1. 플러그인 설치

Claude Code 의 marketplace 기반 플러그인 시스템을 사용합니다. 두 명령으로 설치합니다.

```text
/plugin marketplace add dealicious-inc/deali-skills-plugin
/plugin install dp-skills@deali-skills
```

설치 후 Claude Code 재시작(또는 `/plugin reload`) 시 에이전트·스킬·훅이 자동 등록됩니다.

**확인** — slash 커맨드 자동완성에 `/dp-skills:project`·`/dp-skills:team-init` 이 노출되고, `@ag-planner` subagent 호출이 가능하면 성공입니다.

!!! warning "`/plugin` 이 막힌 환경"
    IDE 내장 세션·관리형 환경에서 `/plugin` 이 차단되면 `dp-skills-update` 헬퍼 스크립트로 캐시를 직접 갱신합니다. 자세한 내용은 [설치 및 초기 세팅](installation.md)을 참고하세요.

## 2. 팀 workspace 부트스트랩 (신규 팀 1회)

```text
/dp-skills:team-init myteam
```

`workspace/` 아래에 `TEAM.md`·`_common/config.md`·`{팀}/STATE.md`·`{팀}/context/` 구조가 생성됩니다. 모든 파일은 비워둬도 동작하므로(fallback) 바로 다음 단계로 넘어가도 됩니다.

## 3. 프로젝트 시작

```text
/dp-skills:project MyFeature
```

`workspace/{팀}/projects/MyFeature/` 가 생성되고 STATE.md 가 갱신됩니다.

## 4. feature 명세 추가

기획서 없이 프롬프트로 단건 추가 (원하는 기능을 충분히 서술):

```text
/dp-skills:create-feature "지연 주문 목록 UI"
```

`features/NN-{slug}.md` 명세가 생성됩니다. (Confluence 기획서가 있으면 `/dp-skills:confl` → `/dp-skills:analyze` 로 벌크 생성도 가능합니다.)

## 5. 첫 plan — `@ag-planner` 호출

```text
@ag-planner
```

planner 가 `features/NN-*.md` 를 읽어 `features/NN-*.plan.md` 구현 계획을 작성합니다. 여기까지가 한 사이클의 시작 — plan 을 확인한 뒤 다음 phase 로 진행합니다.

```text
@ag-planner-critic   # (선택) plan 을 챌린지 → .plan.critic.md
@ag-generator        # plan 기반 구현
@ag-evaluator        # 요구사항 충족 검증 → VERIFICATION REPORT
```

각 phase 는 **자동 연결이 아니라 명시 호출** 입니다. phase 사이에서 멈추고 검토·수정할 수 있습니다.

## 다음 단계

- How-to: [팀 workspace 부트스트랩](../how-to/index.md) — MANIFEST·도메인 채우기
- Explanation: [스킬 vs 에이전트](../explanation/index.md) — 호출 흐름의 개념
- Reference: [스킬·에이전트 목록](../reference/index.md) — 정확한 동작과 인자
