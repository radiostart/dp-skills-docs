# `slack-notify.py`

> dp-skills slack notifier — 프로젝트별 Slack Incoming Webhook 으로 알림 전송.
>
> 설정 SSOT: `workspace/{TEAM}/projects/{PROJECT}/.slack.env`
>     SLACK_WEBHOOK_URL=...          (필수. 비어있거나 파일 없음 → no-op)
>     SLACK_CHANNEL=#project-x       (표시용 라벨)
>     SLACK_EVENTS=complete,approval,pr (생략 시 모두)
>
> No-op 조건 (어느 하나라도 → exit 0, 파이프라인 차단 금지):
>     - 프로젝트 미해석 (workspace/TEAM.md · STATE.md 부재)
>     - .slack.env 부재
>     - SLACK_WEBHOOK_URL 비어있음
>     - 이벤트가 SLACK_EVENTS 목록 밖
>     - .slack.env 가 git tracked 상태 → 전송 차단 + [CRITICAL] 경고
>
> Usage:
>     python3 slack-notify.py --event {complete|approval|pr}
>                            [--project PATH] [--workspace PATH]
>                            [--message TEXT] [--feature-id ID]
>
> 모든 실패 경로에서 exit 0 을 반환한다 (훅·파이프라인이 알림 실패 때문에
> 차단되지 않도록). stderr 에 원인만 1줄 기록.

소스: [`tools/slack-notify.py`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/tools/slack-notify.py)
