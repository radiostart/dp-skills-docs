# toolchain·언어 설정

!!! info "한 줄 요약"
    `_common/config.md` 의 `## 언어·도구 기본값` 표를 채워, 에이전트가 테스트 러너·소스 루트 같은 toolchain 값을 알게 합니다.

## 언제 쓰나

- 팀이 dp-skills 를 처음 설정할 때 — **TDD·evaluator 를 쓰려면 필수**
- 프로젝트의 언어·테스트 명령·소스 루트를 에이전트에 알려야 할 때

!!! warning "이 표가 비면 멈춥니다"
    `## 언어·도구 기본값` 표가 비어 있으면 TDD 사이클과 evaluator 의 테스트 실행 절차가 동작하지 않습니다. 언어와 무관하게 키를 채워야 합니다.

## 전제

- 팀 workspace 가 부트스트랩돼 있을 것 ([팀 부트스트랩](team-bootstrap.md))

## 절차

### 1. `_common/config.md` 의 표 채우기

`workspace/_common/config.md` 의 `## 언어·도구 기본값` 표에 값을 적습니다. 표 헤더·키 이름은 **고정 스키마** 라 그대로 두고 값만 채웁니다.

```markdown
## 언어·도구 기본값

| 키 | 값 | 용도 |
| --- | --- | --- |
| `language` | `ruby` | 프로젝트 주 언어 (hint) |
| `test_command` | `bundle exec rspec` | evaluator 가 변경 관련 테스트 실행 시 |
| `source_root` | `app/` | 소스 루트 — characterize 잠금·scope-guard 기준 |
| `test_path_convention` | `spec/**/*_spec.rb` | 테스트 파일 경로 규약 |
| `pr_default_base` | `develop` | `/dp-skills:pr` 의 base branch default |
```

주요 키:

| 키 | 필수? | 용도 |
|---|---|---|
| `language` | 권장 | 주 언어 hint |
| `test_command` | TDD·evaluator 에 필수 | 테스트 러너 |
| `source_root` | 권장 | 소스 루트 — characterize 모드 잠금 대상 |
| `test_path_convention` | 권장 | 테스트 파일 경로 패턴 |
| `lint_command`·`coverage_command` | 선택 | 린트·커버리지 |
| `conventions_doc` | 선택 | 언어 관행 산문 경로 (아래 3번) |
| `conventions_evals` | 선택 | 검증 케이스 JSON 경로 (아래 3번) |
| `pr_default_base` | 선택 | `/dp-skills:pr` base branch |

사용하지 않는 키는 행을 지우거나 값을 비웁니다 — 플러그인이 해당 기능만 스킵합니다.

### 2. (선택) 팀이 다른 toolchain 을 쓰면 override

전사 기본값과 다른 값을 쓰는 팀은 `workspace/{TEAM}/context/team.config.md` 의 `## 언어·도구 기본값 (override)` 표에 **다른 키만** 다시 선언합니다. 비워 두면 `_common` 값이 그대로 쓰입니다.

### 3. (선택) 코딩 컨벤션·검증 케이스 연결

플러그인은 *언어 중립* 원칙만 내장합니다 (`coding.md`·`evals/coding.json`). 언어·프레임워크 고유 관행은 워크스페이스가 공급합니다.

- `conventions_doc` — 언어·프레임워크 관행을 적은 산문 문서 경로 (예: Rails 매크로, Kotlin 코루틴 규약). `@ag-generator` 가 자동 로드합니다.
- `conventions_evals` — 언어·프레임워크별 코드 검증 케이스 JSON 경로. 역시 generator 가 자동 로드해 제출 전 체크리스트로 씁니다.

`config.md` 에 경로를 선언한 뒤 **실제 파일을 그 경로에 생성** 해야 합니다 (예: `workspace/_common/coding.md`, `workspace/_common/evals/coding.json`).

## 흔한 실수

- **고정 스키마 표의 헤더·키 이름을 바꾸지 않습니다.** 플러그인이 정확히 그 형태로 파싱합니다 — 임의 키를 추가해도 읽히지 않습니다.
- `conventions_doc`·`conventions_evals` 는 경로만 선언하고 파일을 안 만들면 로드할 게 없습니다 — 선언과 파일 생성을 함께.
- `test_command` 가 비면 TDD 사이클이 시작되지 않습니다 — 등록 안내 후 중단됩니다.

## 다음 단계

- Explanation: [워크스페이스 설정](../explanation/workspace-config.md) — `_common` vs 팀 레이어·override 우선순위
- How-to: [TDD 모드 활성화](tdd-mode.md) — `test_command` 가 채워져야 동작
