# `migrate-manifest-split.py`

> migrate-manifest-split.py — 0.3.0 split 일회성 마이그레이션. [ARCHIVED]
>
> [ARCHIVED] 0.3.0 마이그레이션 완료 후 신규 사용 대상 아님 — 어떤 스킬·
> 에이전트도 참조하지 않는다. test_tools_review_fixes.py 의 회귀 테스트
> 가드를 위해 보존한다 (삭제 시 해당 테스트도 함께 정리할 것).
>
> 기존 workspace 의 MANIFEST.md 들에서 파서 대상 섹션을 분리해
> config.md / team.config.md 로 이동시킨다. 도메인 지식 (자유 섹션) 은 보존.
>
> 분리 규칙:
>     workspace/_common/MANIFEST.md
>         → `## 언어·도구 기본값`        →  workspace/_common/config.md
>     workspace/{TEAM}/context/MANIFEST.md
>         → `## Ignore`                  →  workspace/{TEAM}/context/team.config.md
>         → `## 언어·도구 기본값` (드묾) →  workspace/{TEAM}/context/team.config.md
>         → `## 팀 설정`                 →  workspace/{TEAM}/context/team.config.md
>
> Idempotent: MANIFEST.md 에 옛 섹션이 이미 없으면 no-op.
> 대상 config.md 가 이미 있으면 섹션을 append (중복 제거 시도).
>
> 사용:
>     python3 migrate-manifest-split.py workspace            # dry-run (변경 보여주기만)
>     python3 migrate-manifest-split.py workspace --apply    # 실제 적용

소스: [`tools/migrate-manifest-split.py`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/tools/migrate-manifest-split.py)
