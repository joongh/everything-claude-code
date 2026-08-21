# CodeOwl 리뷰 대응

CodeOwl(`codeowl[bot]`)이 PR에 남긴 코드 리뷰 의견에는 **항상 응답 댓글**을 단다. 수정 여부와 무관하게 응답한다.

## 핵심 규칙

1. **모든 리뷰 의견에 응답** — CodeOwl이 남긴 각 리뷰 코멘트(인라인/요약)에 대해, 수정을 했든 안 했든 반드시 댓글로 응답한다.
   - 수정함: 무엇을 어떻게 고쳤는지 + 관련 커밋 SHA
   - 수정 안 함: 왜 안 고치는지 (현재 동작이 안전한 이유 / 의도된 설계 / 보류 사유)
2. **codeowl 멘션** — 응답 댓글 본문에 `@codeowl`을 멘션한다.
3. **음슴체** — 응답은 항상 음슴체로 쓴다 (예: "반영함", "안 고침", "현재 패턴에선 안전함", "보류함").
4. **인사말·치하 금지** — codeowl은 사람이 아니라 AI 에이전트다. "리뷰 고맙습니다", "감사합니다", "좋은 지적입니다" 같은 인사·감사·치하 표현은 넣지 않는다. 곧바로 본론(반영/미반영 + 이유)만 쓴다.
5. **reply로 단다** — 별도 최상위 댓글이 아니라 각 리뷰 코멘트의 답글(`in_reply_to`)로 단다.

## 게시 전 확인 (필수)

댓글을 **실제로 달기 전에 반드시 사용자에게 내용을 보여주고 확인**을 받는다. 확인 전에는 절대 게시하지 않는다.

- 각 리뷰별 응답 초안을 markdown으로 보여준다.
- 사용자의 명시적 확인(`AskUserQuestion` 등) 후에만 게시한다.
- 확인 절차는 [[user-confirmation]] 규칙을 따른다.

## 도구

GitHub PR 리뷰 코멘트 조회/응답:

```bash
# CodeOwl 인라인 리뷰 코멘트 조회 (각 코멘트의 id 확보)
gh api repos/{owner}/{repo}/pulls/{pr}/comments

# 리뷰 요약(전체 review body) 조회
gh api repos/{owner}/{repo}/pulls/{pr}/reviews

# 특정 리뷰 코멘트에 답글 (in_reply_to)
gh api repos/{owner}/{repo}/pulls/{pr}/comments -F in_reply_to={comment_id} -f body='@codeowl ...'
```

## 음슴체 예시

- 반영: `@codeowl 반영함. {메서드}에 {수정} 적용해서 {문제} 해소됨. ({commit})`
- 미반영: `@codeowl 안 고침. {현재 동작이 안전한/의도된 이유}. 필요해지면 그때 반영하겠음.`
