# Slack 알림 설정

!!! info "한 줄 요약"
    프로젝트별 Slack Incoming Webhook 으로 작업 완료·승인 요청을 자동 알림합니다. 설정한 프로젝트만 활성화됩니다.

## 언제 쓰나

- 에이전트 작업 **완료** 나 **승인 요청** 을 Slack 으로 받고 싶을 때

설정하지 않은 프로젝트는 완전 no-op 입니다 — 켜고 싶은 프로젝트에서만 설정합니다.

## 전제

- 활성 프로젝트가 있을 것
- Slack Incoming Webhook URL 을 발급받았을 것

## 절차

### 1. 설정

```text
/dp-skills:slack
```

현재 활성 프로젝트에 `.slack.env` 를 생성하고 채널·이벤트를 대화식으로 설정합니다. SSOT 는 `workspace/{TEAM}/projects/{PROJECT}/.slack.env` 입니다.

```env
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...
SLACK_CHANNEL=#my-channel          # 표시용 라벨
SLACK_EVENTS=complete,approval      # 생략 시 둘 다
```

### 2. 테스트 발송

```text
/dp-skills:slack test
```

### 3. 상태 확인 / 비활성화

```text
/dp-skills:slack status     # 설정·gitignore 보호·tracked 여부 요약
/dp-skills:slack disable    # 비활성화 안내
```

### 발송 이벤트

| 이벤트 | 트리거 |
|---|---|
| `complete` | `@ag-evaluator` 가 `status: READY` |
| `approval` | 권한 다이얼로그(PermissionRequest 훅) |
| `approval` | `@ag-planner` 가 계획 확정 후 사용자 확인 대기 |

## 흔한 실수

- **`.slack.env` 는 절대 커밋하지 않습니다.** `/dp-skills:doctor` 가 리포 루트 `.gitignore` 에 패턴을 자동 주입합니다. tracked 상태로 발견되면 doctor 가 `[ERROR]` (exit 1) 로 차단하고, `slack-notify.py` 도 POST 전 `git ls-files` 로 이중 확인합니다.
- webhook URL 은 메시지 본문·`status` 출력에 노출되지 않습니다 — "설정됨 / 비어있음" 만 표시됩니다.
- 훅 호출 이력이 필요하면 `export DP_SLACK_DEBUG=1` 후 Claude Code 를 재시작하면 `/tmp/dp-skills-slack-hook.log` 에 기록됩니다.

## 다음 단계

- Reference: [`/dp-skills:slack`](../reference/skills/slack.md) · [`slack-notify.py`](../reference/tools/slack-notify.md)
