# How-to

특정 작업을 어떻게 하는지 — 작업별 레시피입니다. 각 페이지는 **언제 쓰나 → 전제 → 절차 → 흔한 실수 → 다음 단계** 구조로, 막혔을 때 바로 펴보도록 만들어졌습니다.

## 부트스트랩

- [팀 workspace 부트스트랩](team-bootstrap.md) — 새 팀이 dp-skills 를 처음 도입할 때
- [toolchain·언어 설정](configure-toolchain.md) — 언어·테스트 러너·소스 루트 등 기본값 설정

## 개발 워크플로우

- [미완료 feature 순차 자동 진행](run-cycle.md) — `/dp-skills:run-cycle` 로 사이클 일괄 진행
- [TDD 모드 활성화](tdd-mode.md) — Red→Green→Refactor 사이클 적용
- [Characterize 모드](characterize-mode.md) — 레거시 코드의 현재 동작 포착
- [Critic 으로 계획 챌린지](critic-review.md) — 구현 전 계획 검토
- [Plan 검증 게이트 통과](plan-gates.md) — 형식·Open Questions·분량 게이트
- [Focus 로 방향 조정](focus-direction.md) — 대화 중 결정을 에이전트에 전달

## 컨텍스트 관리

- [기획서로 features 일괄 생성](analyze-docs.md) — `docs/` → `features/`
- [기획서 없이 feature 추가](create-feature.md) — 프롬프트로 단건 추가
- [도메인 스코프 추출](domain-discover.md) — 코드베이스에서 scope 자동 생성
- [도메인 지식 환류](knowledge-sync.md) — 사이클이 만든 지식을 scope 에 반영
- [도메인 rules 작성](write-domain-rules.md) — 도메인 정책·제약을 수동 작성
- [Confluence 동기화](confluence-sync.md) — 기획서를 `docs/` 로 가져오기

## 코드 리뷰

- [PR 직전 셀프 코드 리뷰](self-code-review.md) — 누적 변경 검토 + 처리 경로 추천
- [PR AI 리뷰 후속 처리](ai-review-followup.md) — 외부 AI 리뷰 코멘트 반영

## 운영

- [Doctor 진단·마이그레이션](doctor-diagnose.md) — 정합성 검사·drift 대응
- [Slack 알림 설정](slack-notify.md) — 작업 완료·승인 알림
- [QA 사이클 진행](qa-cycle.md) — Jira 결함 1건 분석·수정·회신
