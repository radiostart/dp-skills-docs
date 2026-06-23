# PR AI 리뷰 후속 처리

!!! info "한 줄 요약"
    이미 올린 PR 의 AI 리뷰 코멘트에서 필수 수정사항만 추려, fix 브랜치에서 자동 수정하고 후속 PR 을 생성합니다.

## 언제 쓰나

- PR 을 올린 뒤 **AI 리뷰(devops-deali 작성)** 코멘트가 달렸을 때
- 그 코멘트 중 **필수 수정사항만** 골라 반영하고 싶을 때

PR 을 올리기 *전* 셀프 리뷰는 [PR 직전 셀프 코드 리뷰](self-code-review.md)입니다. 이 페이지는 PR *후* 외부 AI 리뷰 처리입니다.

## 전제

- 대상 PR 이 이미 GitHub 에 올라가 있고 AI 리뷰 코멘트가 달려 있을 것

## 절차

### 수정사항 추출 + 자동 반영

```text
/dp-skills:ai-review 1234
```

PR 번호 또는 PR URL 을 받습니다. 동작:

1. AI 리뷰 코멘트에서 **필수 수정사항만** 추출 — 권장 사항이나 "런타임 오류 없음" 판정 항목은 제외
2. 새 fix 브랜치에서 수정 적용
3. 원본 PR 의 base branch 로 후속 PR 자동 생성

## 흔한 실수

- **권장 사항은 반영하지 않습니다.** ai-review 는 *필수* 수정사항만 추립니다 — 권장·"오류 없음" 항목은 의도적으로 제외됩니다.
- 후속 PR 의 base 는 원본 PR 과 동일한 base branch 입니다 — 원본을 향하지 않습니다.

## 다음 단계

- How-to: [PR 직전 셀프 코드 리뷰](self-code-review.md)
- Reference: [`/dp-skills:ai-review`](../reference/skills/ai-review.md) · [`/dp-skills:pr`](../reference/skills/pr.md)
