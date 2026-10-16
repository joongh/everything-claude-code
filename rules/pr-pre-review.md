# PR 생성 전 코드 리뷰

PR을 만들기 전에 **반드시** `code-review` 스킬로 PR에 올라갈 코드를 리뷰한다. 사용자가 "PR 만들어줘"라고만 해도 리뷰를 먼저 한다.

## 시점

- `gh pr create` 직전. 커밋과 push까지는 해도 되지만 PR은 리뷰 뒤에 만든다
- PR을 만든 뒤 커밋을 더 올릴 때는 이 규칙 대상이 아니다 (CodeOwl 리뷰 반영 등)

## 리뷰 범위

`code-review` 스킬은 기본으로 커밋 안 된 변경(`git diff HEAD`)만 본다. PR 직전엔 보통 이미 커밋돼 있으므로 **범위를 base 브랜치 기준으로 넘긴다.**

```
Skill(code-review, args: "git diff {base}...HEAD 범위를 리뷰. 변경 파일: git diff --name-only {base}...HEAD")
```

- `{base}`는 PR 대상 브랜치 (예: `origin/development`). 리뷰 전에 `git fetch`로 최신화한다
- 이 PR 브랜치의 커밋 전체가 대상이다. 최신 커밋만 보지 않는다

## 결과 처리

리뷰 결과를 사용자에게 먼저 보여준다. 결과가 없어도 "리뷰 결과 이슈 없음"이라고 알린다.

| 심각도 | 처리 |
|---|---|
| 치명적, 높음 | PR을 만들지 않는다. 수정안을 보여주고 확인받은 뒤 고치고, 다시 리뷰한다 |
| 중간, 낮음 | 결과를 보여주고 반영할지 사용자에게 묻는다. 반영 안 하기로 하면 PR을 만든다 |
| 이슈 없음 | 바로 PR을 만든다 |

- 지적이 맞는지 직접 확인한다. 틀린 지적은 왜 틀렸는지 적고 넘긴다 (예: Mockito 기본 응답은 List에 빈 리스트를 돌려줘서 NPE 아님)
- 수정 확인 절차는 [[user-confirmation]]을 따른다

## 관련 규칙

- PR 생성 절차 전체: [[git-workflow]]
- PR 생성 후 CodeOwl 리뷰 대기: [[codeowl-review]]
