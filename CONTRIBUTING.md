# Everything Claude Code 기여 가이드

기여해 주셔서 감사합니다. 이 저장소는 Claude Code 사용자를 위한 커뮤니티 리소스입니다.

## 기여 유형

### 에이전트

특정 작업을 잘 처리하는 새로운 에이전트:
- 언어별 리뷰어 (Python, Go, Rust)
- 프레임워크 전문가 (Django, Rails, Laravel, Spring)
- DevOps 전문가 (Kubernetes, Terraform, CI/CD)
- 도메인 전문가 (ML 파이프라인, 데이터 엔지니어링, 모바일)

### 스킬

워크플로우 정의 및 도메인 지식:
- 언어 모범 사례
- 프레임워크 패턴
- 테스트 전략
- 아키텍처 가이드
- 도메인별 지식

### 명령어

유용한 워크플로우를 호출하는 슬래시 명령어:
- 배포 명령어
- 테스트 명령어
- 문서화 명령어
- 코드 생성 명령어

### 훅

유용한 자동화:
- 린팅/포맷팅 훅
- 보안 검사
- 유효성 검증 훅
- 알림 훅

### 규칙

항상 따라야 할 가이드라인:
- 보안 규칙
- 코드 스타일 규칙
- 테스트 요구사항
- 네이밍 컨벤션

### MCP 설정

새로운 또는 개선된 MCP 서버 설정:
- 데이터베이스 연동
- 클라우드 제공자 MCP
- 모니터링 도구
- 커뮤니케이션 도구

---

## 기여 방법

### 1. 저장소 포크

```bash
git clone https://github.com/YOUR_USERNAME/everything-claude-code.git
cd everything-claude-code
```

### 2. 브랜치 생성

```bash
git checkout -b add-python-reviewer
```

### 3. 기여 내용 추가

적절한 디렉토리에 파일 배치:
- `agents/` - 새 에이전트
- `skills/` - 스킬 (단일 .md 또는 디렉토리)
- `commands/` - 슬래시 명령어
- `rules/` - 규칙 파일
- `hooks/` - 훅 설정
- `mcp-configs/` - MCP 서버 설정

### 4. 형식 준수

**에이전트** frontmatter 필수:

```markdown
---
name: agent-name
description: 설명
tools: Read, Grep, Glob, Bash
model: sonnet
---

지시사항...
```

**스킬**은 명확하고 실행 가능해야 함:

```markdown
# 스킬 이름

## 사용 시점

...

## 작동 방식

...

## 예시

...
```

**명령어**는 기능 설명 포함:

```markdown
---
description: 명령어 설명
---

# 명령어 이름

상세 지시사항...
```

**훅**은 설명 포함:

```json
{
  "matcher": "...",
  "hooks": [...],
  "description": "이 훅의 기능"
}
```

### 5. 기여 테스트

제출 전 Claude Code로 설정이 작동하는지 확인하세요.

### 6. PR 제출

```bash
git add .
git commit -m "Add Python code reviewer agent"
git push origin add-python-reviewer
```

PR에 포함할 내용:
- 추가한 내용
- 유용한 이유
- 테스트 방법

---

## 가이드라인

### 권장

- 설정을 집중적이고 모듈화되게 유지
- 명확한 설명 포함
- 제출 전 테스트
- 기존 패턴 준수
- 의존성 문서화

### 금지

- 민감한 데이터 포함 (API 키, 토큰, 경로)
- 과도하게 복잡하거나 특수한 설정 추가
- 테스트되지 않은 설정 제출
- 중복 기능 생성
- 대안 없이 특정 유료 서비스 필요한 설정 추가

---

## 파일 네이밍

- 소문자와 하이픈 사용: `python-reviewer.md`
- 설명적으로: `tdd-workflow.md` (workflow.md 아님)
- 에이전트/스킬 이름과 파일명 일치

---

## 문의

이슈를 열거나 X에서 연락: [@affaanmustafa](https://x.com/affaanmustafa)

---

기여해 주셔서 감사합니다. 함께 훌륭한 리소스를 만들어 갑시다.
