# `/dp-skills:issue`

> 운영 이슈 처리 모드 활성화. `workspace/{TEAM}/issues/{이슈명}/` 폴더를 생성 또는 로드하고, STATE.md를 갱신하며, 도메인 컨텍스트를 로드한다. 버그 대응, 장애 분석, 핫픽스 등 단발성 이슈 단위 작업을 시작할 때 사용한다. 지속적인 기능 개발·feature 단위 작업은 `/dp-skills:project` 를 사용한다.

운영 이슈 처리 모드를 활성화한다.

> QA 결함 처리는 `/dp-skills:qa` 를 사용하세요. issue 는 project 컨텍스트 독립 운영 장애 전용입니다.

대상 이슈: $ARGUMENTS

## 사전 확인

[preamble.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/preamble.md) 의 **P-1, P0, P1** 수행.

- P-1: TodoWrite 선로딩 (다단계 스킬).
- P0: 컨텍스트의 `MEMORY.md` 인덱스에서 인자의 이슈명·키워드와 관련된 메모를 가려 본문을 Read 하여 유사 이슈 이력 반영.
- P1: 팀 이름 `{TEAM}` 획득.

## 수행 절차

1. `$ARGUMENTS` 가 비어있으면 이슈명 없이 바로 처리 모드로 진입한다.
2. `$ARGUMENTS` 가 있으면 `workspace/{TEAM}/issues/{이슈명}/` 존재 여부를 Glob 으로 확인한다.
   - 없으면 `skills/issues/GUIDE.md` 를 로드하여 새 이슈 폴더를 생성한다.
   - 있으면 `issue.md` 만 로드한다 (GUIDE.md 는 로드하지 않는다).
3. [preamble.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/preamble.md) 의 **P3** 수행 — STATE.md 테이블 본문을 `| issue | {이슈명} | 진행중 |` **1행으로 교체**. 기존 다른 이름 행은 모두 삭제 (이력은 git log).
   - 이슈명이 없으면 `| issue | - | 진행중 |` 로 기록한다.
4. [preamble.md](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/skills/context/shared/preamble.md) 의 **P4** 수행 — 도메인 컨텍스트 로드.
5. 이슈 컨텍스트를 요약하고 다음 작업을 안내한다.
