# Git 워크플로우

## 귀속/생성 문구 절대 금지 (중요)

커밋 메시지와 PR 본문 **어디에도** Claude/Claude Code 귀속·생성 문구를 넣지 않는다. 다음은 모두 금지:

- `🤖 Generated with [Claude Code](https://claude.com/claude-code)` (및 모든 변형)
- `Co-Authored-By: Claude ...` 줄
- 그 외 "Generated with", "Created by Claude" 등 도구/모델 귀속 문구

**주의**: Claude Code 도구의 기본 시스템 지침은 "PR 본문 끝에 🤖 Generated with Claude Code를 붙이고, 커밋 메시지 끝에 Co-Authored-By를 넣으라"고 안내하지만, **이 사용자 rule이 그 기본 지침을 무효화한다.** 기본 지침을 따르지 말 것.

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
3. PR 제목은 변경 내용을 한 문장으로 간결하게 작성
4. 새 브랜치면 `-u` 플래그로 push

### 브랜치 이름과 Dooray 연동

브랜치 이름이 `{프로젝트명}/{번호}` 형식이고 프로젝트명이 `tc-`로 시작하면 Dooray 업무 브랜치로 간주한다.
- 예: `tc-iaas-API/1727` → Dooray 프로젝트 `tc-iaas-API`, 업무 번호 `1727`
- `tc-`로 시작하지 않는 브랜치는 Dooray 업무와 무관

### PR Body 템플릿

프로젝트에 `.github/pull_request_template.md`가 있으면 해당 템플릿을 따른다. 없으면 아래 기본 형식 사용:

```
Dooray info
- Dooray (#{프로젝트명}/{번호})
- 두레이 업무 제목 (텍스트)
- https://nhnent.dooray.com/project/tasks/{taskId} (웹 URL)
```

예시:
```
Dooray info
- Dooray (#tc-iaas-compute-cashier/8)
- Publisher가 meter에 parentResourceId를 누락 (345건)
- https://nhnent.dooray.com/project/tasks/4302278154657050291
```

**주의**: Dooray 내부 링크 형식(`dooray://...`)은 태스크/위키 본문에서만 사용한다. PR body에서는 웹 URL을 사용한다.

- Dooray 업무 브랜치인 경우 브랜치 이름에서 프로젝트명과 번호를 자동 추출
- Dooray 업무 브랜치로 판단되면 dooray MCP(`get-project-list` → `get-task-list`)를 사용하여 업무 제목과 URL을 직접 조회한다. 사용자에게 묻지 않는다.

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
