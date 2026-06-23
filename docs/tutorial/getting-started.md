# Deep Walkthrough

[Quick Start](quick-start.md)가 5분짜리 첫 사이클이라면, 이 페이지는 dp-skills 의 전체 흐름을 한 번 끝까지 따라가는 walkthrough 입니다. 신규 기획서로 기능 하나를 구현해 PR 까지 올리는 시나리오입니다.

## 시나리오

새 팀에서 Confluence 기획서를 받아 기능을 구현하고, 셀프 코드 리뷰를 거쳐 PR 을 올립니다.

## 1. 설치·부트스트랩

[설치 및 초기 세팅](installation.md)을 따라 플러그인을 설치하고 팀 workspace 를 만듭니다.

```text
/dp-skills:team-init myteam
```

신규 팀이라면 `context/MANIFEST.md` 는 비워둬도 됩니다 — 모든 파일이 fallback 으로 동작합니다. 도메인을 등록하려면 자유 형식으로 분류 표만 적습니다.

```markdown
## 도메인 분류

| 도메인 | scope | rules | 설명 |
| ------ | ----- | ----- | ---- |
| checkout | `scope/checkout.md` | `rules/checkout.md` | 결제·주문 흐름 |
```

## 2. 프로젝트 시작 — 기획서 fetch 포함

URL 을 함께 주면 프로젝트 생성 + Confluence fetch + analyze 가 한 번에 실행됩니다. `--tdd` 로 TDD 모드도 켭니다.

```text
/dp-skills:project MyFeature https://wiki.example.com/pages/12345 --tdd
```

이제 `workspace/myteam/projects/MyFeature/` 아래에 `project.md`·`.agent-state.yml`·`agents/`·`features/` 가 준비됩니다. 기획서는 `docs/` 에 저장되고, `features/NN-{slug}.md` 명세로 분할됩니다.

기획서 없이 시작한다면 프롬프트로 단건을 추가합니다.

```text
/dp-skills:create-feature "지연 주문 목록 UI"
```

## 3. 계획 — `@ag-planner`

```text
@ag-planner
```

planner 가 `features/01-{slug}.md` 를 읽어 `features/01-{slug}.plan.md` 계획을 작성하고, 사용자 확인을 기다립니다. TDD 모드이므로 계획에 스텝 분할과 실패 테스트(Red)가 포함됩니다.

## 4. (선택) 계획 챌린지 — `@ag-planner-critic`

변경이 복잡하면 구현 전에 계획을 검토합니다.

```text
@ag-planner-critic
```

`.plan.critic.md` 에 전제·범위·엣지케이스·대안·리스크 5축 챌린지가 기록됩니다. blocking 지적이 있으면 `@ag-planner` 를 다시 호출해 **계획을 고칩니다** (코드가 아니라).

trivial 한 변경이면 이 단계는 건너뜁니다.

## 5. 구현 — `@ag-generator`

```text
@ag-generator
```

확정된 계획대로 구현합니다. TDD 모드면 Red 테스트를 통과시키는 최소 구현(Green)을 만듭니다.

## 6. 중간에 방향을 바꾸고 싶다면 — `/dp-skills:focus`

메인 대화에서 내린 결정은 에이전트에게 자동 전달되지 않습니다. `.focus.md` 에 기록해야 합니다.

```text
/dp-skills:focus "소프트 딜리트는 archived_at 타임스탬프로 대체"
@ag-planner          # 지시 반영된 새 계획
```

지시 반영이 끝나면 `/dp-skills:focus --clear` 로 정리합니다.

## 7. 검증 — `@ag-evaluator`

```text
@ag-evaluator
```

feature 계획 대비 구현 충족도를 심사하고 `VERIFICATION REPORT` 를 냅니다. TDD 모드면 변경 관련 테스트만 실행합니다. `NOT_READY` 가 나오면 지적사항을 반영해 다시 사이클을 돕니다.

## 8. 커밋

```text
/dp-skills:commit
```

팀 커밋 메시지 규칙(scope·한글·50자 권장)에 맞춰 커밋합니다.

## 9. PR 직전 셀프 코드 리뷰

사이클과 별개로, PR 전에 누적 변경 전체를 한 번 더 검토합니다.

```text
@ag-code-reviewer --base develop
/dp-skills:fix-review
```

`CODE REVIEW REPORT` 의 finding 별로 `/dp-skills:fix-review` 가 처리 경로(trivial / one-shot / full-cycle / new-feature / dismiss)를 추천합니다. 표를 보고 묶음별로 명시 호출합니다. 자세한 내용은 [PR 직전 셀프 코드 리뷰](../how-to/self-code-review.md)를 참고하세요.

## 10. PR 생성

```text
/dp-skills:pr
```

base branch 를 자동 타겟팅하고, 컨벤션에 맞춰 본문을 작성해 PR 을 올립니다. PR 후 외부 AI 리뷰 코멘트는 [`/dp-skills:ai-review`](../how-to/ai-review-followup.md)로 처리합니다.

## 정리 — 한 사이클의 모양

```text
team-init → project → (confl/analyze 또는 create-feature)
  → @ag-planner → (선택) @ag-planner-critic → @ag-generator → @ag-evaluator
  → commit → @ag-code-reviewer → fix-review → pr
```

각 단계는 **자동 연결이 아니라 명시 호출** 입니다 — 어느 phase 에서든 멈추고 검토·수정할 수 있는 것이 dp-skills 의 핵심입니다.

## 다음 단계

- How-to: [작업별 레시피 전체](../how-to/index.md)
- Explanation: [에이전트 흐름](../explanation/agent-flow.md) · [모드](../explanation/modes.md)
