# PR 직전 셀프 코드 리뷰

!!! info "한 줄 요약"
    PR 을 올리기 전 누적 변경 전체를 `@ag-code-reviewer` 로 검토하고, finding 별 처리 경로를 추천받습니다.

## 언제 쓰나

- PR 을 올리기 **직전**, 여러 feature 의 누적 변경을 한 번 더 점검할 때
- 사이클(planner→generator→evaluator) 밖의 hotfix·외부 변경을 검토할 때

`@ag-code-reviewer` 는 **사이클 밖의 선택적 단계** 입니다. 자동으로 발동되지 않고 슬래시 진입점도 없습니다 — 사용자가 `@ag-code-reviewer` 를 직접 호출할 때만 동작합니다. trivial 한 변경이면 건너뛰어도 됩니다.

이건 evaluator 와 **다른 축** 입니다. evaluator 는 요구사항·TDD 증거·테스트 실행을 보고, code-reviewer 는 cross-cutting 일관성·언어 규약·hunk 디테일·호출자 영향·누락 마무리만 봅니다.

## 전제

- 검토할 변경(commit 또는 작업 트리)이 있을 것
- 활성 프로젝트는 없어도 됩니다 — `@ag-code-reviewer` 는 workspace-less 로 동작합니다

## 절차

### 1. (최초 1회) 코드 리뷰 룰 셋업

`@ag-code-reviewer` 가 변경 언어의 룰 파일 부재를 감지하면 안내합니다.

```text
/dp-skills:code-review-init ruby
```

3가지 시작 전략(예시 복사 / 빈 템플릿 / AI 생성) 중 인터랙티브로 선택합니다. `rules.md`(팀 도메인 룰)와 `{lang}.md`(언어 규약)를 함께 셋업합니다.

### 2. 리뷰 실행

```text
@ag-code-reviewer --base develop
```

`CODE REVIEW REPORT` 가 출력됩니다 — verdict(`block` / `nit` / `approve`)와 finding 목록. finding 마다 confidence 점수(0~100)가 붙고, `--threshold` 미만(기본 80)은 자동 dismiss 됩니다.

큰 PR(30+ 파일)이면 단일 `@ag-code-reviewer` 는 상위 hunk 만 sampling 합니다 — 팀 룰 기반 깊이가 필요하면 `--files` 로 파일그룹을 나눠 여러 번 호출하고, 일반 안전망 breadth 는 네이티브 `/code-review` 를 병행하세요. (구 `/dp-skills:dp-code-review` 병렬 래퍼는 네이티브 `/code-review` 수렴으로 0.21.0 에서 제거됨.)

### 3. 처리 경로 추천받기

```text
/dp-skills:fix-review
```

finding 별 처리 경로를 표로 추천합니다 — **자동 실행하지 않습니다**.

| 경로 | 처리 |
|---|---|
| `trivial` | 직접 Edit |
| `one-shot` | `@ag-generator` 1회 |
| `full-cycle` | `@ag-planner → @ag-generator → @ag-evaluator` |
| `new-feature` | `/dp-skills:create-feature "..."` |
| `dismiss` | 룰 보강 또는 acknowledgement |

### 4. 표를 보고 묶음별 명시 호출

```text
@ag-generator    # one-shot 묶음
/dp-skills:commit
/dp-skills:pr
```

## 코드 리뷰 컨텍스트 파일

`@ag-code-reviewer` 는 호출 시 워크스페이스의 룰 파일을 자체 로드합니다. 세 종류가 있고, 사는 위치가 다릅니다.

| 파일 | 위치 | 목적 | 누가 만드나 |
|---|---|---|---|
| `generic.md` | 플러그인 ship (`skills/context/shared/code-review/`) | 언어 무관 hunk-level 베이스 룰 — 모든 변경에 1차 통과 | 플러그인 (자동 로드) |
| `rules.md` | `_common/` 또는 `{TEAM}/context/` 의 `code-review/` | 언어 무관 팀·전사 정책 (보안·로깅 PII·API 계약 등) | `/dp-skills:code-review-init` |
| `{lang}.md` | 동일 | 언어 문법·프레임워크 안티패턴 (예: `python.md`, `ruby.md`) | `/dp-skills:code-review-init` |

- **`_common/` vs `{TEAM}/`** — 회사 전체에 적용할 룰은 `_common/code-review/`, 팀 특화·추가 룰은 `{TEAM}/context/code-review/`. 충돌 시 좁은 scope(팀)가 우선합니다. ([워크스페이스 설정](../explanation/workspace-config.md) 참고)
- `{lang}.md` 가 없으면 reviewer 는 언어 규약 축만 graceful 하게 건너뛰고 `next` 필드로 `code-review-init` 를 안내합니다.
- 룰 형식·confidence threshold 등 운영 디테일은 [`README` 의 코드 리뷰 컨텍스트](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/README.md#코드-리뷰-컨텍스트) 참조.

## 흔한 실수

- **`@ag-code-reviewer` 는 파일을 수정하지 않습니다** — 보고만 합니다. `/dp-skills:fix-review` 도 분류·추천만 하고 자동 Edit·자동 호출이 없습니다.
- code-reviewer 는 *작성된 코드* 를 봅니다. *작성된 계획* 검토는 [`@ag-planner-critic`](critic-review.md) 의 몫입니다.
- `dismiss` 케이스에서 grep 룰 보강 제안이 자주 나옵니다 — `code-review/{lang}.md`·`rules.md` 가 점진 개선되는 자연스러운 경로입니다.

## 다음 단계

- How-to: [PR AI 리뷰 후속 처리](ai-review-followup.md)
- Reference: [`@ag-code-reviewer`](../reference/agents/ag-code-reviewer.md) · [`/dp-skills:fix-review`](../reference/skills/fix-review.md) · [`/dp-skills:code-review-init`](../reference/skills/code-review-init.md)
