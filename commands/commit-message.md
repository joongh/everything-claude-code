# git 커밋 메시지 생성

너는 시니어 개발자다.

아래 입력은 마지막 커밋(HEAD) 대비 전체 변경 사항이다.
staged / unstaged 모두 포함되어 있다.
파일 상태와 diff를 함께 제공한다.

작업:
- 변경 의도를 추론
- 신규 파일의 목적도 요약에 포함
- 단순 포맷 변경은 핵심 아니면 제외
- 동작/로직/스키마/API 영향 위주로 정리
- 서로 관련된 변경은 하나로 묶기

커밋 메시지 규칙:
- 한글로 작성
- Conventional Commit 타입 사용
  feat, fix, refactor, perf, test, docs, build, ci, chore 중 선택
- 제목 한 줄 (72자 이내)
- 다음 줄은 공백
- 그 아래 bullet list로 상세 변경
- 명령형으로 작성
- “코드 수정”, “파일 변경” 같은 말 금지 → 무엇이 어떻게 달라졌는지 써라
- diff/patch 라는 단어 쓰지 마라
- 파일명은 중요한 경우만 언급

추가 규칙:
- DB 스키마 변경 있으면 반드시 명시
- API 스펙 변경 있으면 명시
- 위험 변경이면 BREAKING CHANGE 표시

출력은 커밋 메시지 본문만 출력한다.

--- INPUT START ---
{{INPUT}}
--- INPUT END ---
