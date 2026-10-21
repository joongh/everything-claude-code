# CodeOwl 리뷰 대응

CodeOwl(`codeowl[bot]`)이 PR에 남긴 코드 리뷰 의견에는 **항상 응답 댓글**을 단다. 수정 여부와 무관하게 응답한다.

## 핵심 규칙

1. **모든 리뷰 의견에 응답** — CodeOwl이 남긴 각 리뷰 코멘트(인라인/요약)에 대해, 수정을 했든 안 했든 반드시 댓글로 응답한다.
   - 예외: 리뷰가 "총 0개의 이슈"로 끝나면 답글을 달지 않는다. 사용자에게 결과만 알린다 (2026-10-21 사용자 지시, i.compute-cashier#146)
   - 수정함: 무엇을 어떻게 고쳤는지 + 관련 커밋 SHA
   - 수정 안 함: 왜 안 고치는지 (현재 동작이 안전한 이유 / 의도된 설계 / 보류 사유)
2. **codeowl 멘션** — 응답 댓글 본문에 `@codeowl`을 멘션한다.
3. **음슴체** — 응답은 항상 음슴체로 쓴다 (예: "반영함", "안 고침", "현재 패턴에선 안전함", "보류함").
4. **인사말·치하 금지** — codeowl은 사람이 아니라 AI 에이전트다. "리뷰 고맙습니다", "감사합니다", "좋은 지적입니다" 같은 인사·감사·치하 표현은 넣지 않는다. 곧바로 본론(반영/미반영 + 이유)만 쓴다.
5. **reply로 단다** — 별도 최상위 댓글이 아니라 각 리뷰 코멘트의 답글(`in_reply_to`)로 단다. 단 리뷰가 일반 PR 코멘트로만 왔으면(인라인 없음) 같은 PR 에 일반 코멘트 하나로 항목별로 답한다.

## PR 생성 후 리뷰 대기 (자동)

PR을 만들면 CodeOwl 리뷰가 올라오는지 **백그라운드로 최대 10분** 확인한다. 사용자가 따로 요청하지 않아도 한다.

- 시점: `gh pr create` 직후
  - 그 PR에 새 커밋을 push한 뒤에는 기다리지 않는다. CodeOwl은 PR을 열 때만 리뷰하는 것으로 보임 (아래 실측)
- 방법: `Bash`를 `run_in_background: true`로 실행한다. 끝나면 알림으로 다시 호출되니, 포그라운드에서 sleep으로 기다리지 않는다. PR이 있는 리포 디렉터리에서 실행한다 (gh가 remote로 GitHub Enterprise 호스트를 찾음)

```bash
REPO={owner}/{repo}; PR={pr}
HEAD=$(gh pr view "$PR" -R "$REPO" --json headRefOid -q .headRefOid)
SEL="[.[] | select(.user.login == \"codeowl[bot]\" and (.body | contains(\"head-sha=$HEAD\")))] | length"
for i in $(seq 1 20); do   # 30초 × 20 = 10분
  # 리뷰가 PR 리뷰(pulls/reviews)로 올 때도, 일반 PR 코멘트(issues/comments)로 올 때도 있다
  r=$(gh api "repos/$REPO/pulls/$PR/reviews" --jq "$SEL")
  c=$(gh api "repos/$REPO/issues/$PR/comments" --jq "$SEL")
  if [ "${r:-0}" -gt 0 ]; then echo "CODEOWL_REVIEW_FOUND kind=review pr=$PR head=$HEAD elapsed=$(( (i - 1) * 30 ))s"; exit 0; fi
  if [ "${c:-0}" -gt 0 ]; then echo "CODEOWL_REVIEW_FOUND kind=comment pr=$PR head=$HEAD elapsed=$(( (i - 1) * 30 ))s"; exit 0; fi
  sleep 30
done
echo "CODEOWL_REVIEW_TIMEOUT pr=$PR head=$HEAD"
```

- 현재 HEAD로 거르는 이유: 리뷰 body 첫 줄에 `<!-- codeowl:review:head-sha={sha} -->`가 들어 있다 (PR 리뷰·일반 코멘트 둘 다). push 전의 옛 리뷰를 새 리뷰로 착각하지 않게 한다
- 결과별 처리
  - `FOUND kind=review` → 리뷰 요약(`pulls/{pr}/reviews`)과 인라인 코멘트(`pulls/{pr}/comments`)를 조회 → 코멘트마다 반영할지 판단 → 아래 "게시 전 확인" 절차를 따른다
  - `FOUND kind=comment` → 지적이 일반 코멘트(`issues/{pr}/comments`) 본문 하나에 다 들어 있고 인라인 코멘트가 없다. 지적마다 반영할지 판단하고, 답글은 같은 PR 에 **일반 코멘트 하나**로 항목별로 묶어 단다 (`in_reply_to` 를 쓸 대상이 없음)
  - 두 경우 모두 리뷰가 "총 0개의 이슈"면 답글 초안을 만들지 않고 결과만 알린다
  - 실측 (i.compute-cashier#146, 2026-10-21): 이슈 0건일 때도 리뷰를 일반 PR 코멘트로 남김 (90초 만에)
  - `TIMEOUT` → "10분 안에 CodeOwl 리뷰가 안 올라옴"이라고 알리고 끝낸다. 더 기다리지 않는다. 이슈가 0건일 때 리뷰를 남기는지는 확인 못 했으니 "이슈 없음"으로 단정하지 않는다
- 리뷰가 올라와도 바로 코드를 고치거나 답글을 달지 않는다. 반영 여부와 답글 초안을 먼저 보여주고 확인받는다
- 실측 (i.compute-cashier#121, 2026-09-29)
  - PR 생성 후 1분 안에 리뷰가 올라옴
  - 그 뒤 리뷰 반영 push, rebase force push 두 번 모두 10분 안에 재리뷰 없었음 (TIMEOUT)
- 실측 (IaaS.OpenStack.API.SDK#462, 2026-10-12)
  - 리뷰가 PR 리뷰가 아니라 일반 PR 코멘트로 올라옴 (인라인 코멘트 없음). `pulls/reviews` 만 보던 스크립트는 TIMEOUT 으로 놓쳤음

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
