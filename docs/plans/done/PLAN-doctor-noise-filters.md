# PLAN: doctor 노이즈 필터 2건 (0.16.18)

nimda 실워크스페이스 사본 점검 (0.16.17) 에서 발견된 doctor 노이즈 2건 수정.
회귀가 아닌 기존 검사의 필터 누락.

## 결함 1 — Open Questions INFO 가 파생 산출물에도 발생

- **위치**: `tools/doctor/integrity.py` `check_features_open_questions`
- **현상**: 대상 필터가 `glob("*.md")` + `^\d{2}-` prefix + `.plan.md` suffix 제외라
  `31-address-masking-permission.eval.md` · `44-presend-code-review-fix.plan.critic.md`
  같은 파생 산출물이 통과 → "## Open Questions 섹션 부재" INFO 노이즈.
  OQ 섹션은 본문 명세 (`NN-{slug}.md`) 에만 의미가 있다.
- **수정**: 0.16.7 에서 도입된 `doctor/_common.py` 의 `is_feature_spec()` 단일 규칙을 쓰는
  `_list_feature_md()` 로 대상 필터 교체 (`count_real_features` 와 동일 규칙).

## 결함 2 — 관리외 파일 INFO 에 qa/ 포함

- **위치**: 같은 파일 `check_managed_files_integrity` 의 `MANAGED_TOP_DIRS`
- **현상**: qa phase 가 1급 기능 (0.11.0~) 이 된 후 목록 미갱신 —
  `{"agents", "features", "docs", ".agents.bak"}` 에 `qa` 누락으로
  qa/ 디렉터리가 "관리외 파일 (사용자 메모·스크래치)" 로 분류.
- **수정**: `"qa"` 추가.

## 작업 단계 (TDD)

1. RED — `tests/tools/test_doctor_project_hygiene.py` 에 실패 테스트 추가:
   - 파생 산출물만 있는 features/ → OQ INFO 0 건 (+ 본문 명세 INFO 유지 회귀 가드)
   - qa/ 디렉터리 보유 프로젝트 → 관리외 INFO 0 건 (+ 진짜 관리외 디렉터리 INFO 유지)
2. GREEN — 필터 교체 + `"qa"` 추가.
3. `pytest tests/tools -q` 전체 통과 (기존 482 passed 유지).
4. `plugin.json` 0.16.17 → 0.16.18 (patch bump).
5. PR — base `main`, squash.

## 범위 외

- `collect_open_questions_stats` (--oq-stats 모드) 도 동일한 구식 필터를 사용해
  파생 산출물이 `features_total`·`features_without_oq_section` 을 부풀린다.
  doctor 기본 출력 노이즈가 아니므로 본 건 범위 외 — 별도 후속.
