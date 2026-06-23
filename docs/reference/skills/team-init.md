# `/dp-skills:team-init`

> 새 팀 셋업 — workspace 구조를 초기화한다. `workspace/TEAM.md` 와 `workspace/{TEAM}/context/` 하위의 MANIFEST · rules · scope · enums 스켈레톤을 일괄 생성한다. 처음 dp-skills 를 도입·셋업하는 팀, 또는 기존 팀 구조를 기준으로 새 팀을 추가할 때 사용한다.

신규 팀의 workspace 구조를 일괄 생성한다.

대상 팀: $ARGUMENTS

---

## 사전 확인

1. `$ARGUMENTS` 파싱 — 첫 번째 공백 구분 토큰을 `{TEAM}` 으로 사용.
   - 비어있으면 **"팀 이름을 지정하세요. 예: `/dp-skills:team-init fin-team`"** 안내 후 종료.
   - 예약어 (`example`, `workspace`, `STATE`, `TEAM`, `templates`) 이면 **"예약어는 팀명으로 사용 불가"** 안내 후 종료.
   - 공백·특수문자 (`/`, `\`, `:`, `.`) 포함 시 거부.

2. workspace 경로 결정 — CWD 기준 `./workspace/`.
   - 폴더 없으면 생성 (`mkdir -p workspace`).

---

## 동작

### 1. `workspace/TEAM.md` 처리

- 파일이 없으면: [TEAM.md.template](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/lifecycle/setup/templates/TEAM.md.template) 을 Read 하고 `{TEAM}` 치환 후 Write.
- 파일이 있으면:
  - 첫 줄이 `{TEAM}` 과 동일 → skip (이미 동일 팀).
  - 첫 줄이 다른 팀명 → **"이 workspace 는 이미 `{기존팀}` 팀 소유. 다른 디렉토리에서 실행하거나 먼저 기존 구조를 옮기세요."** 안내 후 종료. **덮어쓰지 않음.**

### 2. 팀 폴더 스켈레톤 생성

아래 각 파일에 대해 **대상 존재 시 skip, 없으면 템플릿으로 생성** (idempotent):

| 템플릿 | 대상 경로 | 치환 |
| ------ | --------- | ---- |
| `templates/STATE.md.template` | `workspace/{TEAM}/STATE.md` | 없음 |
| `templates/MANIFEST.md.template` | `workspace/{TEAM}/context/MANIFEST.md` | `{TEAM}` → 팀명 |
| `templates/team.config.md.template` | `workspace/{TEAM}/context/team.config.md` | `{TEAM}` → 팀명 |

**code-review 파일은 자동 생성하지 않는다.** `rules.md` (팀 도메인 룰) 와 `{lang}.md` (언어 규약) 모두 코드 리뷰가 실제 필요해질 때 `/dp-skills:code-review-init` 로 셋업. 사용 안 하는 팀은 폴더 자체가 안 생김. 빠른 워크플로: 신규 팀 → `team-init` → 도메인 작업 → 첫 PR 직전 `@ag-code-reviewer` → 부재 안내 따라 `/dp-skills:code-review-init`.

**`_common/config.md` 추가 처리:**

`workspace/_common/config.md` 가 없으면 `templates/_common.config.md.template` 으로 생성 (idempotent — 이미 있으면 skip). MANIFEST.md (전사 도메인 지식) 는 다중팀 환경 선택 사항이라 자동 생성하지 않는다.

템플릿 위치: `${CLAUDE_PLUGIN_ROOT}/skills/context/lifecycle/setup/templates/`

절차:

1. `workspace/{TEAM}/context/` 폴더를 필요 시 생성.
2. 각 템플릿을 Read.
3. 치환 대상이 있으면 `{TEAM}` 을 팀명으로 문자열 교체.
4. 대상 경로가 이미 존재하면 skip (마크 `exists`), 없으면 Write (마크 `created`).

> **`rules/`·`scope/`·`enums/` 등 카테고리 폴더는 생성하지 않는다.** 카테고리 구조는 팀이 MANIFEST.md 를 채우면서 결정하고, 실제 도메인 파일을 추가할 때 필요한 폴더를 만든다. 플러그인은 MANIFEST 선언만 따른다.

---

## 결과 출력

아래 형식으로 요약:

```
팀 초기화 완료: {TEAM}
Workspace: {workspace 절대 경로}

파일 상태:
  workspace/TEAM.md                              {created|exists|skipped}
  workspace/{TEAM}/STATE.md                      {created|exists}
  workspace/{TEAM}/context/MANIFEST.md           {created|exists}
  workspace/{TEAM}/context/team.config.md        {created|exists}
  workspace/_common/config.md                    {created|exists}

▶ 다음 단계 (필수):
  /dp-skills:project {프로젝트명}        # 바로 시작 가능

▶ 채우면 좋은 것 (점진적, 미작성 시 fallback 동작):
  - context/MANIFEST.md                  # 첫 도메인 작업 시 도메인 지식 (자유 구조)
  - context/team.config.md               # ## Ignore 패턴이 있을 때만
  - _common/config.md                    # 기본 toolchain 외 언어·테스트 명령
  - context/{rules,scope}/{도메인}.md     # 도메인 파일은 첫 추가 시 폴더 함께 생성

▶ 코드 리뷰 셋업 (선택, 필요해질 때):
  PR 직전 `@ag-code-reviewer` 호출. 룰 파일 부재 시 reviewer 가 안내 →
    /dp-skills:code-review-init {lang}
  으로 팀 룰 (rules.md) + 언어 룰 ({lang}.md) 함께 1 회성 셋업.

> 모든 파일은 비워둬도 동작합니다. 코드 작업 중 필요해지는 순간에 채우세요.
```

---

## 참고

- 이 스킬은 **팀 단위 workspace 를 처음 만들 때만** 사용한다. 기존 팀에 프로젝트를 추가하려면 `/dp-skills:project`.
- 생성되는 모든 파일은 팀이 직접 채워야 하는 스켈레톤이다 (도메인·공통 모델·Ignore 등).
- 생성 후 스켈레톤을 실제 값으로 채우는 작업은 이 스킬 범위 밖. 사용자가 문서를 편집한다.
- 각 스켈레톤 상단에 "이 파일은 `/dp-skills:team-init` 가 생성했다" 주석이 있어, 어디서 왔는지 역추적 가능.
