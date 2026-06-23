# 팀 workspace 부트스트랩

!!! info "한 줄 요약"
    새 팀이 dp-skills 를 처음 도입할 때 `workspace/` 메타구조를 한 번에 생성합니다. 팀당 1회.

## 언제 쓰나

- 팀이 dp-skills 를 **처음 도입**할 때
- 한 저장소에서 **다른 팀 workspace 를 추가**할 때 (팀별 workspace 분리)

이미 `workspace/{TEAM}/` 가 있다면 다시 실행해도 안전합니다 — idempotent 이라 기존 파일을 덮어쓰지 않습니다.

## 전제

- dp-skills 플러그인이 설치돼 있을 것 ([Quick Start](../tutorial/quick-start.md) 참고)
- 작업 저장소 루트에서 실행할 것

## 절차

### 1. 팀 부트스트랩 실행

```text
/dp-skills:team-init myteam
```

생성물:

```text
workspace/
├── TEAM.md                    # 팀명
├── _common/
│   └── config.md              # 전사 toolchain 기본값 (언어·도구)
└── myteam/
    ├── STATE.md                # 진행중 프로젝트 추적 (빈 테이블)
    └── context/
        ├── MANIFEST.md          # 도메인 지식 — 팀이 자유롭게 채움
        └── team.config.md       # 팀 런타임 설정 (Ignore·팀 설정)
```

코드 리뷰 룰 파일은 **만들지 않습니다** — 필요해질 때 [`/dp-skills:code-review-init`](self-code-review.md) 가 별도로 셋업합니다.

### 2. (선택) MANIFEST.md 채우기

비워둬도 [`/dp-skills:project`](../tutorial/quick-start.md) 는 동작합니다. 도메인을 명시하면 에이전트가 알맞은 `scope`·`rules` 를 자동 로드합니다. 자유 형식이지만 도메인 분류 표가 가장 흔합니다.

```markdown
## 도메인 분류 (예시 — 자유 형식)

| 도메인 | scope | rules | 설명 |
| ------ | ----- | ----- | ---- |
| checkout | `scope/checkout.md` | `rules/checkout.md` | 결제·주문 흐름 |
```

### 3. 정합성 확인

```text
/dp-skills:doctor
```

## 흔한 실수

- **`scope/`·`rules/`·`enums/` 폴더를 미리 만들지 않습니다.** 도메인 파일을 실제로 추가할 때 생성됩니다. 빈 폴더를 미리 만들 필요 없습니다.
- **`## Ignore` 를 MANIFEST.md 에 적지 않습니다.** 0.3.0 split 이후 `team.config.md` 로 이동했습니다 — MANIFEST 에 적으면 파서가 인식하지 않습니다.
- `_common/config.md` 의 `{test_command}` 등을 비워두면 TDD·evaluator 절차에서 등록 안내 후 중단됩니다. 코드 작업 전까지는 비워둬도 됩니다.

## 다음 단계

- How-to: [도메인 스코프 추출](domain-discover.md) — 코드베이스에서 `scope` 자동 생성
- Reference: [`/dp-skills:team-init`](../reference/skills/team-init.md)
