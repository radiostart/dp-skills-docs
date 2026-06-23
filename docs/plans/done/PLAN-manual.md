# Plan — dp-skills 매뉴얼 사이트 (MkDocs Material)

> status: approved
> created: 2026-05-22
> 도구: **MkDocs Material**
> 정보구조: **Diátaxis 4 분류** (Tutorial / How-to / Reference / Explanation)
> 호스팅: **GitHub Pages** (Private — org 직원 한정 노출)

## 1. 목표·범위

`dp-skills/README.md` 가 1139 줄·40+ 헤더로 비대해 처음 사용자는 압도되고 숙련
사용자는 reference 를 못 찾는다. 매뉴얼을 별도 사이트로 분리해 *역할별 진입* 을
제공한다. dp-skills 의 명칭·구조와 이 repo 의 CI 제약을 반영한다.

### in scope

- MkDocs Material 정적 사이트, GitHub Pages 배포.
- Diátaxis 4 분류 정보구조.
- `skills/`·`agents/`·`tools/` 사용자 facing 메타데이터 자동 추출 → reference.
- 루트·dp-skills README 슬림화 (사이트가 SSOT, README 는 stub + 링크).
- Mermaid 다이어그램 4 개.

### out of scope

- 다국어 (한국어 단일 — SSOT 가 한국어).
- 인터랙티브 데모.
- 커스텀 JS (Material 기본 제공 외 추가 안 함).
- 페르소나·SSOT 메타 서술 — *실용 매뉴얼* 지향, 내부 구현 디테일 배제.

## 2. 반영해야 할 dp-skills 고유 구조·제약

| 항목 | dp-skills |
|---|---|
| 에이전트 (5종) | `ag-planner`·`ag-planner-critic`·`ag-generator`·`ag-evaluator`·`ag-code-reviewer` |
| 스킬 | `discover`·`team-init` 등 19종 (코드 리뷰는 `ag-code-reviewer`+`fix-review`+`ai-review`+`dp-code-review`) |
| workspace | `workspace/{TEAM}/projects/` — 팀 레이어 |
| 프로젝트 에이전트 파일 | `agents/` |
| 배포 | Actions 불가 (Hosted Runner IP 차단) → 로컬 gh-deploy + pre-push 훅 |
| README | 2 개 (루트 + dp-skills) |

## 3. 사이트 구조 (Diátaxis 4 분류)

`mkdocs.yml` 위치 = `dp-skills/mkdocs.yml`. 실행 `cd dp-skills && mkdocs serve`.

```
dp-skills/docs/
├── index.md                     # 랜딩 (가치 제안 + 4 경로 카드)
├── tutorial/
│   ├── installation.md
│   ├── quick-start.md            # 5 분 데모
│   └── getting-started.md        # 30 분 deep walkthrough
├── how-to/                       # 13 페이지 (작업별 레시피)
│   ├── team-bootstrap.md
│   ├── tdd-mode.md
│   ├── characterize-mode.md
│   ├── critic-review.md
│   ├── focus-direction.md
│   ├── analyze-docs.md
│   ├── create-feature.md
│   ├── domain-discover.md
│   ├── self-code-review.md
│   ├── ai-review-followup.md
│   ├── doctor-diagnose.md
│   ├── confluence-sync.md
│   └── slack-notify.md
├── reference/
│   ├── agents/                   # 자동 추출 (5)
│   ├── skills/                   # 자동 추출 (17)
│   ├── tools/                    # 자동 추출
│   ├── state-schema.md
│   ├── plan-schema.md
│   ├── hooks.md
│   └── supported-env.md
└── explanation/                  # 4 페이지 (최소 — 막힐 때 보는 개념)
    ├── concepts.md
    ├── agent-flow.md
    ├── workspace-layout.md
    └── modes.md
```

## 4. 자동 추출 — `dp-skills/tools/docs_build.py` (신규)

- 입력: `agents/*.md`, `skills/*/SKILL.md`, `tools/*.py`.
- 출력: `docs/reference/{agents,skills,tools}/*.md` — **커밋 대상**. GitHub 에서 docs/
  를 직접 열람할 때도 reference 링크가 살아야 하므로 산출물을 저장소에 둔다.
- 변환: frontmatter → H1/subtitle, wrapper 인용 블록 제거; tools 는 docstring.
- `--check` 모드: 커밋된 reference 가 SSOT 와 어긋나면 exit 1 (drift 감지). stdlib
  전용이라 mkdocs 없이 pre-push 에서 항상 실행 가능.
- 테스트: `tests/tools/test_docs_build.py` (agents·skills·tools fixture 각 1).

## 5. 빌드·배포 (CI 제약 반영)

이 repo 는 Hosted Runner 가 IP allowlist 로 차단 → Actions 자동배포 불가.

- 로컬 개발: `cd dp-skills && mkdocs serve`.
- 배포: `cd dp-skills && python3 tools/docs_build.py && mkdocs gh-deploy --strict --force`
  → `gh-pages` 브랜치에 정적 HTML push. Pages 는 정적 파일만 서빙 → 러너 불필요.
- 검증: `.githooks/pre-push` 에 `docs_build.py --check` + `mkdocs build --strict` 추가.
- Pages 활성화 (admin 권한): Settings → Pages → Source `gh-pages` → Visibility `Private`.

## 6. README 슬림화

루트·dp-skills README 양쪽에서 사이트로 이전한 섹션 제거 → 사이트 링크 대체.
dp-skills README 는 설치·Quick Start stub + 사이트 안내만 유지.

## 7. Mermaid 다이어그램 (4)

1. `agent-flow.md` — ag-planner→critic→generator→evaluator 흐름 (sequence).
2. `workspace-layout.md` — `workspace/{TEAM}/` 팀 레이어 트리 (graph).
3. `modes.md` — Standard / TDD / Characterize 진입 분기 (flowchart).
4. `doctor-diagnose.md` — schema 마이그레이션 트리 (flowchart).

## 8. 단계별 구현 순서 (각 단계 별도 PR 가능)

1. `mkdocs.yml` + `docs/index.md` + 4 분류 골격 + `quick-start.md` → 로컬 serve 확인.
2. `tools/docs_build.py` + 단위 테스트.
3. 자동 추출로 `reference/` 채움.
4. How-to 13 페이지 (표준 구조: 요약 / 언제 쓰나 / 전제 / 절차 / 흔한 실수 / 다음 단계).
5. Explanation 4 페이지 + Mermaid.
6. Tutorial 3 페이지.
7. 루트·dp-skills README 슬림화.
8. `.githooks/pre-push` docs 검증 step.
9. 첫 배포 + admin 에 Pages Private 활성화 요청 + 링크 검증.

## 9. 확정 결정

- 배포 — 로컬 `mkdocs gh-deploy` + pre-push 훅 (Actions 미사용).
- reference — `docs_build.py` 자동 추출, 출력은 커밋 (GitHub 직접 열람 대응), drift 는 pre-push 의 `--check` 감지.
- README — 슬림화 + 사이트 링크 (루트·dp-skills 양쪽).
- 언어 — 한국어 단일.
- 매뉴얼 무게중심 — How-to 가 메인, 페르소나·메타 서술 배제.
