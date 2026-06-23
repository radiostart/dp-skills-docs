# `/dp-skills:code-review-init`

> 워크스페이스에 언어별 코드 리뷰 룰 파일 (`{lang}.md`) 을 셋업한다. `@ag-code-reviewer` 가 변경 dominant 언어의 룰 파일 부재를 보고하면 본 스킬로 1 회성 생성. 3 가지 시작 전략 (예시 복사 / 빈 템플릿 / AI 생성) 중 사용자가 선택. 생성 후 사용자가 본문 편집해 팀 컨벤션 반영.

워크스페이스의 언어별 코드 리뷰 룰 파일 `{lang}.md` 를 셋업한다.

대상 언어: $ARGUMENTS

---

## 사전 확인

1. `$ARGUMENTS` 파싱 — 첫 번째 토큰을 `{lang}` 슬러그로 사용 (`python` / `typescript` / `javascript` / `ruby` / `php` / `kotlin` / `java` / 또는 임의 슬러그).
   - 비어있으면: 현재 디렉토리에서 dominant 확장자 감지하여 슬러그 추론.

     ```bash
     git ls-files | awk -F. '{print $NF}' | sort | uniq -c | sort -rn | head -5
     ```

     매핑 (ag-code-reviewer.md:50 과 동일): `.py/.pyi → python` · `.ts/.tsx → typescript` · `.js/.jsx/.mjs/.cjs → javascript` · `.rb/.rake/.erb → ruby` · `.php/.phtml → php` · `.kt/.kts → kotlin` · `.java → java` · `.go → go` · `.rs → rust` · `.swift → swift` · `.sql → sql` · 그 외 → 사용자에게 슬러그 질의.
     사용자에게 "**{추정 lang} 로 진행할까요?**" 확인.

2. 팀 식별:
   - `workspace/TEAM.md` 첫 줄을 `{TEAM}` 으로 읽음.
   - 없으면 **"워크스페이스 미초기화. `/dp-skills:team-init {팀명}` 먼저 실행하세요."** 안내 후 종료.

3. 대상 경로 결정:
   - 사용자에게 **scope 선택** 질의:
     - **`team`** (기본) → `workspace/{TEAM}/context/code-review/{lang}.md`
     - **`common`** (전사 공통) → `workspace/_common/code-review/{lang}.md`
   - 단일 팀 워크스페이스면 `team` 자동 선택.

4. 이미 존재 시:
   - **"이미 `{경로}` 존재. 덮어쓸까요? (덮어쓰기 / 백업 후 새로 생성 / 취소)"** 질의.
   - 백업 선택 시 `{경로}.bak.{timestamp}` 로 이동.

5. `rules.md` 동반 셋업 (최초 1 회):
   - `workspace/{TEAM}/context/code-review/rules.md` (또는 scope=common 선택 시 `workspace/_common/code-review/rules.md`) 부재 시 다음 안내 후 질의:

     ```
     팀 도메인 룰 파일 (rules.md) 도 함께 생성할까요?

     rules.md 는 언어 무관 팀·도메인 룰을 담습니다. 5 카테고리 구조:
       1. 보안          — 시크릿·SQL/XSS 인젝션·인증/권한
       2. 로깅·관측      — PII·시크릿 노출·로그 레벨·trace ID
       3. API 호환성     — public 시그니처·계약·deprecation
       4. 의존성·설정    — env 변수·config·패키지·feature flag
       5. 도메인 컨벤션  — 팀 service/controller/repository 패턴

     {lang}.md 와 분리: 언어 문법·프레임워크 안티패턴 (Rails N+1·Spring @Transactional 등) 은 {lang}.md 에.

     생성할까요? (예/아니오)
     ```

   - 예: `${CLAUDE_PLUGIN_ROOT}/skills/context/lifecycle/setup/templates/code-review.rules.md.template` Read → `{TEAM}` 치환 → 대상 경로 Write.
   - 아니오: skip. 본 스킬을 다시 호출해도 같은 질의 반복 안 함 (이미 존재 시 자동 skip).

---

## 동작 — 시작 전략 3 종

사용자에게 다음 3 가지 중 택 1 질의:

### 전략 A — 사전 작성된 예시 복사

`${CLAUDE_PLUGIN_ROOT}/examples/code-review/{lang}.md` 존재 시 활성화. 부재 시 비활성 (전략 B/C 만 제시).

1. 파일 Read.
2. 헤더 "**참고용 예시**" 안내 블록 (`>` 시작 라인 3~4 줄) 제거.
3. 사용자에게 **사용 프레임워크 확인** (해당 시):
   - `ruby` → "Rails 사용? (예/아니오)"
   - `php` → "Laravel·Symfony·둘 다·둘 다 아님 중 선택"
   - `kotlin` → "Android·서버사이드 중 선택"
   - `java` → "Spring 사용? (예/아니오)"
   - `javascript`·`typescript` → "React 사용? (예/아니오)"
4. 미사용 프레임워크 섹션 (`## Rails 특화 룰` / `## Spring 특화` 등) 본문에서 제거.
5. 대상 경로에 Write.

### 전략 B — 빈 형식 템플릿 복사

`${CLAUDE_PLUGIN_ROOT}/skills/context/lifecycle/setup/templates/code-review.lang.md.template` 사용.

1. 파일 Read.
2. 헤더 placeholder 치환:
   - `{LANG}` → `Python` / `Ruby` / `Java` 등 (사람이 읽는 이름)
   - `{lang-slug}` → `python` / `ruby` / `java` 등
   - `{확장자}` → 슬러그별 확장자 (`.py` / `.rb` / `.java` 등)
   - `{ext1}, {ext2}, …` → 동일
3. 대상 경로에 Write.
4. 사용자에게 **"빈 형식 생성 완료. 본문의 `Grep-friendly` / `Judgment-based` 섹션을 직접 채우세요."** 안내.

### 전략 C — AI 생성 (코드베이스 기반)

본 스킬이 직접 LLM 생성:

1. 코드베이스 탐색:

   ```bash
   git ls-files | grep -E '\.({lang 확장자})$' | head -50
   ```

   상위 5~10 개 파일을 Read 해 패턴 파악 (사용 중인 logger·테스트 프레임워크·DI 컨테이너·ORM 등).
   **Read 는 한 메시지에 묶어 fan-out** — 5~10 개 Read 호출을 한 turn 의 parallel tool calls 로 보낸다 (read-only · 순서 의존 없음 — sequential Read 는 wall-clock 만 길어진다).
2. `templates/code-review.lang.md.template` 형식을 베이스로, 발견된 컨벤션·안티패턴을 자동 추출해 룰화.
3. 생성 결과를 사용자에게 **미리보기** 로 보여주고 "이대로 저장 / 수정 후 저장 / 취소" 질의.
4. 확정 시 대상 경로에 Write.

**전략 C 주의**: 생성된 룰은 LLM 추측 기반. 사용자가 검토·편집 필요. 본 스킬은 "draft" 생성기지 "정답" 생성기 아님.

---

## 결과 출력

```
코드 리뷰 룰 생성: {lang}.md
경로: {대상 경로}
전략: {examples | empty-template | ai-generated}

요약:
  Grep-friendly 룰: {N}개
  Judgment-based 룰: {N}개
  프레임워크 섹션: {유지 항목 또는 "없음"}
  rules.md 동반 생성: {yes | no | already-exists}

▶ 다음 단계:
  1. {대상 경로} 를 열어 본문 검토·편집
  2. (rules.md 생성한 경우) 팀 도메인 룰도 함께 검토·확장
  3. 팀 컨벤션과 충돌하는 룰은 삭제
  4. `@ag-code-reviewer` 재실행 시 본 파일이 자동 로드됨
```

---

## Do-NOT

- 기존 파일을 **사용자 확인 없이** 덮어쓰지 않는다 (사전 확인 4 항).
- 전략 C 의 결과를 **자동 저장하지 않는다** (반드시 미리보기 → 사용자 승인).
- 생성된 룰의 정확성을 **보증하지 않는다** — 사용자 책임. 본 스킬은 draft 도우미.
- `workspace/` 외부 경로에 Write 금지.

---

## 호출 시점

- `@ag-code-reviewer` 가 `{lang}.md` 부재 보고 → reviewer 의 `next` 필드 안내에 따라 사용자가 명시 호출
- 새 언어 도입 시 (예: TS 프로젝트에 Python 스크립트 추가) 팀이 선제적 호출
- `/dp-skills:team-init` 후 주력 언어 셋업
