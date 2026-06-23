# PLAN-lean-cleanup — 0.20.3 전반 점검 후속 정리

> **상태: 종결 (CLOSED, 2026-06-23) — Tier 1+2 머지·릴리스 `dp-skills-v0.21.0`, Tier 3 의도적 생략.** 전반 점검(오버엔지니어링·버전진화 잉여·설계 재점검) 산출. PR #67(Tier 1)·#68(Tier 2).
> 헤드라인: **규모 대비 오버엔지니어링은 적음.** 코드 머신러리는 대체로 값을 함.
> 진짜 잉여는 코드가 아니라 *완료된 PLAN 문서 408KB*. 큰 리팩터 불필요.

---

## 0. 절대 하지 말 것 (검증으로 기각한 "거짓 발견")

자동 감사가 제안했으나 **load-bearing 결정을 되돌리는 퇴행**이라 금지:

1. `STATE.md` + `.agent-state.yml` 병합 — durable 결정 위반(human/machine 분리, P2 `진행중` 리터럴 탐지). 메모리 `qa-off-toggle-deferred`.
2. planner↔critic·generator↔evaluator 루프 자동수렴화 — 휴먼게이트 의도적. 메모리 `loop-engineering-assessment-verdict`.
3. verify-report-lint·regen-verify·handoff-quality·cross-ref·migrate-manifest-split 삭제 — **전부 live 연결**(각각 ag-evaluator·analyze·ag-planner·analyze·`doctor --fix`). orphan 아님.
4. preamble P-레벨 단순화 — 이미 잘 된 DRY(SSOT 1곳 + 적용표). P-1은 deferred-tool InputValidationError 회피용 실제 워크어라운드.
5. orchestrate-load.py(921줄) 분해 — 4 에이전트의 동일 계약 참조는 서브에이전트 컨텍스트 비공유라 불가피.

---

## 1. Tier 1 — `docs/plans/` 완료 설계서 정리 (저위험·고가치)

**근거:** 27개 PLAN 전부 마지막 커밋 = 머지 커밋 → 전량 shipped. 408KB. git이 이력 보존.

- [x] **1a. PLAN-qa-phase §0 의존 제거** (블로커) — `phase-transition.md`·`qa/SKILL.md` 의 `(PLAN-qa-phase §0)` 인용을 제거하고 핵심 근거("다건 SSOT=Jira·이중화 회피")를 self-contained 로 inline. §0 의 5개 운영 전제는 이미 두 living 문서에 분산 인코딩돼 손실 없음.
- [x] **1b. 27개 PLAN → `docs/plans/done/` git mv** — 삭제 아닌 이동(인수인계 설계근거 보존). `docs/plans/` 직하는 `PLAN-lean-cleanup.md`(활성) 만 잔존.
- [x] **1c. CLAUDE.md 컨벤션 명문화** — "완료 PLAN → done/ 이동" 규칙 추가(자기영속 + done/ 도 `plans/*` 글롭 제외).
- [x] **1d. 검증 PASS** — docs_build --check(drift 0)·doctor --schema(6 PASS)·mkdocs --strict(done/ 정상 제외). markdownlint 는 로컬 미설치이나 config 가 MD013(line-length) off + 변경이 구조 무수정이라 clean.

## 2. Tier 2 — 판단 2건 (결정 C/C → 실행, PR #68)

- [x] **2a. oq-eval → C (게이트화).** oq-synthetic 8 fixture(frozen actual.md)를 `baseline-final.json` 대조하는 회귀 테스트 추가 — idle eval 을 살아있는 게이트 회귀로 전환(공유 OQ 파서 보호). analyze(LLM) 불필요(결정론적). 삭제·아카이브 아닌 "값을 하게" 택1.
- [x] **2b. dp-code-review → C (네이티브 수렴).** 병렬 래퍼 은퇴(네이티브 `/code-review` 가 parallel·dedup·severity 흡수). 해자(팀 계층 언어룰·5축·`fix-review` 연계)는 `@ag-code-reviewer` 로 보존. `self-code-review` 에 마이그레이션 안내. bump 0.21.0. 코드리뷰 sprawl "추가 전 통합" 규율은 유지.

## 3. Tier 3 — 코스메틱 문서 정리 → **의도적 생략 (CLOSED)**

**판정: 하지 않음.** 참조 web 실측상 각 병합이 6~17 refs(cycle agents·`plan-validate.py`·`orchestrate-load.py` 포함)를 건드리고, 빌드 게이트(docs_build·mkdocs·pytest)가 못 잡는 **런타임 LLM 참조 리스크**(에이전트가 옮겨진 섹션 미발견)를 동반. 감소는 −3 파일(state-schema 분리는 오히려 +1)로 200+ 파일 대비 무의미. **churn·리스크 > cosmetic 가치** → 위 "단독 비권장"과 정합. 해당 파일을 다른 이유로 만질 때만 기회적으로.

- (생략) INDEX 2→1 / state-schema 분리 / oq-gate+plan-schema 병합 / rgr→tdd-activation / "SSOT" 라벨

---

## 결과 (2026-06-23)

- Tier 1(PR #67)·Tier 2(PR #68) 머지 → **`dp-skills-v0.21.0` 릴리스**. 검증: pytest 534 passed · docs_build --check(drift 0) · doctor --schema(19스킬·6 PASS) · mkdocs --strict.
- Tier 3 생략. 본 plan 종결 — `done/` 아카이브(완료 설계 근거 보존).
