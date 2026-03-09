# Dooray 연동

## 링크 패턴

Dooray 태스크 링크 형식:
```
https://{org}.dooray.com/project/{projectId}/posts/{postId}
```

사용자가 이 형식의 링크를 공유하면 `projectId`와 `postId`를 파싱하여 MCP 도구로 태스크를 조회한다.

## 워크플로우

1. 사용자가 Dooray 링크 공유
2. URL에서 `projectId`, `postId` 추출
3. `get-task`로 태스크 본문 조회
4. 필요시 `get-task-comment-list`로 댓글 확인
5. 요청에 따라 태스크 수정 또는 댓글 작성

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
