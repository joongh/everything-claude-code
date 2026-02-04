# Git 워크플로우

## 커밋 메시지 형식

```
<type>: <description>

<optional body>
```

타입: feat, fix, refactor, docs, test, chore, perf, ci

참고: ~/.claude/settings.json에서 전역적으로 귀속 비활성화됨.

## Pull Request 워크플로우

PR 생성 시:
1. 전체 커밋 히스토리 분석 (최신 커밋만이 아님)
2. `git diff [base-branch]...HEAD`로 모든 변경 확인
3. 포괄적인 PR 요약 작성
4. TODO가 포함된 테스트 계획 포함
5. 새 브랜치면 `-u` 플래그로 push

## 기능 구현 워크플로우

1. **먼저 계획**
   - **planner** 에이전트로 구현 계획 생성
   - 의존성 및 위험 식별
   - 단계별로 분해

2. **TDD 접근**
   - **tdd-guide** 에이전트 사용
   - 먼저 테스트 작성 (RED)
   - 테스트 통과하도록 구현 (GREEN)
   - 리팩토링 (IMPROVE)
   - 80%+ 커버리지 확인

3. **코드 리뷰**
   - 코드 작성 직후 **code-reviewer** 에이전트 사용
   - 치명적 및 높은 이슈 해결
   - 가능하면 중간 이슈 수정

4. **커밋 & Push**
   - 상세한 커밋 메시지
   - conventional commits 형식 준수
