# Dooray 연동

## 링크 패턴

Dooray 태스크 링크 형식:
```
https://{org}.dooray.com/project/{projectId}/posts/{postId}
```

사용자가 이 형식의 링크를 공유하면 `projectId`와 `postId`를 파싱하여 MCP 도구로 태스크를 조회한다.

## Dooray 내부 링크 형식 (중요)

태스크/위키 본문에 Dooray 리소스 링크를 작성할 때는 웹 URL이 아닌 **Dooray 내부 링크 형식**을 사용해야 한다.

**organizationId**: `get-project` 응답의 `organization.id` 값. 세션에서 한번 조회 후 캐시하여 재사용한다.

**태스크 링크:**
```markdown
[{taskNumber} {subject}](dooray://{organizationId}/tasks/{taskId} "{workflowClass}")
```
- `taskNumber`: 프로젝트코드/번호 (예: `tc-iaas-API/1902`)
- `taskId`: 태스크 고유 ID
- `workflowClass`: 태스크 상태 (`registered`, `working`, `closed` 등)

**위키 페이지 링크:**
```markdown
[{pageTitle}](dooray://{organizationId}/pages/{pageId} "publish")
```
- `pageId`: 위키 페이지 고유 ID
- 상태는 항상 `"publish"`

## 워크플로우

1. 사용자가 Dooray 링크 공유
2. URL에서 `projectId`, `postId` 추출
3. `get-task`로 태스크 본문 조회
4. 필요시 `get-task-comment-list`로 댓글 확인
5. 요청에 따라 태스크 수정 또는 댓글 작성

## 내용 입력/수정 시 확인 절차 (중요)

Dooray에 내용을 입력하거나 수정하는 **모든 작업**은 실행 전에 반드시 사용자에게 미리보기를 보여주고 명시적 확인을 받아야 한다.

**대상 작업:**
- 태스크 생성 (`create-task`)
- 태스크 수정 (`update-task`)
- 태스크 댓글 작성 (`create-task-comment`) / 수정 (`update-task-comment`)
- 위키 페이지 생성 (`create-wiki-page`) / 수정 (`update-wiki-page`)
- 위키 댓글 작성 (`create-wiki-page-comment`) / 수정 (`update-wiki-page-comment`)

**확인 절차:**
1. 작성할 내용을 markdown 형태로 사용자에게 먼저 보여줄 것
2. 수정의 경우 markdown diff 형태로 변경 전/후를 보여줄 것
3. 사용자의 명시적 확인을 받은 후에만 도구를 실행할 것

**새 내용 작성 예시:**
```markdown
## 태스크 제목: 신규 태스크 제목

### 본문:
여기에 작성할 본문 내용...
```

**기존 내용 수정 예시:**
```diff
- 제목: 기존 태스크 제목
+ 제목: 변경할 태스크 제목

- 본문: 기존 본문 내용
+ 본문: 변경할 본문 내용
```

## 작성 스타일 (간결·단계형)

태스크/위키/댓글 본문은 긴 서술 문장으로 늘이지 말고, 논리 단계를 줄바꿈·불릿·화살표(→)로 끊어 짧게 쓴다.

- 인과·절차는 한 문장에 몰아넣지 말고 `A → B → C` 또는 불릿으로 분해
- 한 항목엔 한 사실만
- 문제점/원인/해결처럼 단계가 있으면 섹션으로 나누고, 각 항목은 짧은 불릿으로
- 단, 간결함은 **문장**에만 적용한다. UUID·호스트명 등 식별자는 축약하지 않는다 ([[api-response-display]] 참고)

**예 (지양 → 지향):**

```
지양: 이 인스턴스는 직전에 host A→B로 옮겨져, 알림을 유발한 행은 DB상 NONE이지만 콘솔에선 host≠source_host로 CANCELED로 표시되어 발송 대상이 아니어야 함

지향:
- 인스턴스가 host A → B로 이동
- DB: NONE
- API: host ≠ source_host → CANCELED 표시
- → 발송 대상 아님 (그러나 DB 기준으론 대상)
```

## 주요 MCP 도구

| 도구 | 용도 |
|------|------|
| `get-task` | 태스크 본문 조회 |
| `update-task` | 태스크 수정 (제목, 본문, 상태 등) |
| `create-task-comment` | 태스크에 댓글 작성 |
| `get-task-comment-list` | 태스크 댓글 목록 조회 |
| `get-project-list` | 프로젝트 목록 조회 |
| `get-project-members` | 프로젝트 멤버 조회 |
| `create-task` | 새 태스크 생성 |
| `get-task-list` | 태스크 목록 조회 |

## API 토큰 발급

1. Dooray 로그인
2. 우측 상단 프로필 > **개인 설정**
3. **API** 탭 > **개인 인증 토큰** 발급
4. `~/.claude.json`의 `mcpServers.dooray.env.DOORAY_API_TOKEN`에 설정

## 설정 방법

`mcp-configs/mcp-servers.json`의 dooray 항목을 `~/.claude.json`에 복사:

```json
{
  "mcpServers": {
    "dooray": {
      "command": "npx",
      "args": ["-y", "@jhl8041/dooray-mcp@latest"],
      "env": {
        "DOORAY_API_TOKEN": "실제_토큰_값"
      }
    }
  }
}
```
