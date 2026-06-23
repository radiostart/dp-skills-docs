# `verify-report-lint.py`

> dp-skills verify-report-lint — evaluator VERIFICATION REPORT 형식·일관성 검증.
>
> evaluator wrapper step 7 이 출력하는 `## VERIFICATION REPORT` 블록을
> 결정론적으로 파싱해 다음을 검증:
>
> - 필수 섹션 / gate 존재
> - 각 gate 값 enum (pass/fail/skip/none/detected)
> - status (READY|NOT_READY) ↔ gates 일관성
> - mode (red_contract|characterize|standard) ↔ gate skip 매핑
> - status: NOT_READY ↔ issues_to_fix 비어있지 않음
> - drift: detected → issues_to_fix 에 언급
> - focus gate 부재 시 warn (구형 REPORT 호환 — missing_gate_legacy)
> - metrics.domain_impact enum (none/detected/skip) — 부재 시 warn (legacy 호환)
>
> Usage:
>     cat report.md | python3 tools/verify-report-lint.py
>     python3 tools/verify-report-lint.py path/to/report.md
>     python3 tools/verify-report-lint.py path/to/report.md --json
>
> Exit:
>     0 — 위반 없음 (PASS)
>     1 — 위반 있음 (FAIL) 또는 REPORT 블록 부재
>     2 — 입력 오류

소스: [`tools/verify-report-lint.py`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/tools/verify-report-lint.py)
