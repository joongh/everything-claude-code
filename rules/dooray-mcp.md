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

**멤버 멘션:**
```markdown
[@{name}](dooray://{organizationId}/members/{organizationMemberId} "member")
```
- `organizationMemberId`: `get-project-member-list` / `get-my-member-info` 응답의 `organizationMemberId`
- 타이틀은 항상 `"member"` (조회 시 자기 자신은 `"me"`로 표시되지만, 작성할 때는 `"member"`를 쓴다)
- 예: `[@양성연](dooray://1387695619080878080/members/4131706313960528971 "member") 전임님, 추가 질문 있습니다.`

**⚠️ 평문 `@이름`은 멘션이 아니다.** 그냥 텍스트로 남아 상대방에게 알림이 가지 않는다. 태스크·위키·댓글에서 사람을 부를 때는 반드시 위 링크 형식을 쓴다.

- 이름 뒤 호칭(전임님/책임님 등)은 링크 **밖에** 둔다
- 상대의 `organizationMemberId`를 모르면 `get-project-member-list`로 조회한다. 같은 태스크의 기존 댓글을 `get-task-comment-list`로 읽어 거기 있는 멘션 링크를 그대로 재사용하는 것도 확실한 방법이다

## 워크플로우

1. 사용자가 Dooray 링크 공유
2. URL에서 `projectId`, `postId` 추출
3. `get-task`로 태스크 본문 조회
4. 필요시 `get-task-comment-list`로 댓글 확인
5. 요청에 따라 태스크 수정 또는 댓글 작성

## 수정 전 최신 내용 재조회 (중요)

기존 태스크·위키·댓글을 **수정**하기 전에는 반드시 `get-*` 도구로 **원본을 다시 조회**한다. 세션 컨텍스트에 남아 있는 내용을 기준으로 수정하지 않는다.

**이유**: 내가 생성/수정한 뒤에 사용자나 다른 사람이 Dooray에서 직접 제목·본문을 고쳤을 수 있다. 오래된 컨텍스트를 기준으로 `update-*`를 호출하면 그 수동 수정이 조용히 사라진다(덮어쓰기 유실).

**수정 직전 재조회 필수:**

| 수정 도구 | 재조회 도구 |
|------|------|
| `update-task` | `get-task` |
| `update-task-comment` | `get-task-comment-list` |
| `update-wiki-page` | `get-wiki-page` |
| `update-wiki-page-comment` | `get-wiki-page-comment` |

**절차:**
1. `get-*`로 현재 원본(제목·본문) 조회
2. 컨텍스트에 있던 내용과 비교
   - 동일 → 그대로 수정 진행
   - 다름 → **덮어쓰지 않는다.** 달라진 부분을 사용자에게 알리고, 사용자 수정분을 유지한 병합안을 만들어 확인받는다
3. 재조회한 최신 본문을 기준으로 diff를 만들어 아래 "내용 입력/수정 시 확인 절차"를 진행한다

**주의:**
- 같은 세션에서 방금 내가 만든 태스크라도 예외 없이 재조회한다 (생성 직후 사용자가 손댔을 수 있음)
- 상태·담당자 등 메타 필드만 바꾸는 경우에도, `update-task`가 본문을 함께 전송하는 구조이므로 재조회 필수
- 재조회를 생략해도 되는 경우는 없다. 조회 1회 비용 < 유실된 사용자 수정 복구 비용
- 새 내용 **작성**(`create-task`, `create-task-comment`, `create-wiki-page` 등)은 기존 내용을 덮어쓰지 않으므로 재조회 대상 아님

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

## 댓글 답글 스타일 (사용자 실제 형식)

동료·팀장의 댓글에 답하는 초안은 사용자가 실제로 쓰는 형식을 따른다. 이 경우엔 위 "작성 스타일"보다 이 절이 우선한다. 태스크 본문이나 조사 결과를 정리하는 댓글은 위 작성 스타일(불릿·표)을 그대로 쓴다.

섹션을 나누고 불릿을 여러 개 달고 근거까지 늘어놓은 초안은 쓰이지 않았다. 사용자가 4문장으로 직접 다시 썼다 (tc-iaas-API/2094, 2026-09-30).

- 첫 줄은 `[@이름](dooray://.../members/{id} "member") 책임님,`처럼 호칭 뒤에 쉼표를 찍고 줄을 바꾼다. "확인 감사합니다" 같은 인사 없이 바로 본론을 쓴다
- 본문은 존댓말(`~습니다`)로 짧은 문단 2~4문장. 섹션 제목을 달지 않고, 불릿도 보통 쓰지 않는다
- 내용은 핵심만 쓴다
  1. 상대 제안에 걸리는 사실 한 가지
  2. 결정은 상대나 팀에 넘기고, 내가 할 일만 적는다 ("~로 결정이 되면 구현 추가하도록 하겠습니다"). 대안은 괄호로 가볍게 적는다 ("(또는 null 로 무시?)")
  3. 필요한 정보는 질문으로 묻고 끝낸다 ("혹시 ~ 고객이 있나요?")
- 코드 경로, 파일명, 세부 근거는 넣지 않는다. 상대가 물어보면 그때 답한다
- 영어 단어 뒤 조사는 띄어 쓴다: `nginx 에서`, `admin_pass 를`
- cc는 꼭 알려야 할 사람이 있을 때만 단다
- 굵게·색 강조는 팀장에게 결론을 보고하는 첫 댓글 정도에만 쓴다

**예 (실제 사용자 댓글):**

```
[@김용현](dooray://.../members/... "member") 책임님,

현재 metadata 수정은 nginx 에서 nova 로 바로 보내주고 있기 때문에 차단하기 위해서는 addons 에서 metadata 수정을 받아주는 구현이 추가로 필요합니다.
addons 에서 막아야하는 것 (또는 null 로 무시?) 으로 결정이 되면 구현 추가하도록 하겠습니다.

혹시 현재 metadata 의  admin_pass 를 사용하고 있는 고객이 있나요?
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
