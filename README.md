# Everything Claude Code

**Spring Boot/Kotlin/Java 백엔드 개발자를 위한 Claude Code 설정 모음**

10개월 이상의 실제 프로덕션 개발 경험을 통해 발전시킨 에이전트, 스킬, 훅, 명령어, 규칙 및 MCP 설정입니다.

---

## 가이드

이 저장소는 설정 코드만 포함합니다. 가이드에서 모든 것을 설명합니다.

### 시작하기: 간략 가이드

<img width="592" height="445" alt="image" src="https://github.com/user-attachments/assets/1a471488-59cc-425b-8345-5245c7efbcef" />

**[The Shorthand Guide to Everything Claude Code](https://x.com/affaanmustafa/status/2012378465664745795)**

기초 - 각 설정 유형의 역할, 설정 구조화 방법, 컨텍스트 윈도우 관리, 이 설정들의 철학. **먼저 읽으세요.**

---

### 그 다음: 상세 가이드

<img width="609" height="428" alt="image" src="https://github.com/user-attachments/assets/c9ca43bc-b149-427f-b551-af6840c368f0" />

**[The Longform Guide to Everything Claude Code](https://x.com/affaanmustafa/status/2014040193557471352)**

고급 기술 - 토큰 최적화, 세션 간 메모리 지속, 검증 루프 및 평가, 병렬화 전략, 서브에이전트 오케스트레이션, 지속적 학습.

| 주제 | 배울 내용 |
|------|----------|
| 토큰 최적화 | 모델 선택, 시스템 프롬프트 최소화, 백그라운드 프로세스 |
| 메모리 지속 | 세션 간 컨텍스트 자동 저장/로드 훅 |
| 지속적 학습 | 세션에서 패턴 자동 추출하여 재사용 가능한 스킬로 |
| 검증 루프 | 체크포인트 vs 연속 평가, 그레이더 유형, pass@k 메트릭 |
| 병렬화 | Git worktrees, cascade 방법, 인스턴스 확장 시점 |
| 서브에이전트 오케스트레이션 | 컨텍스트 문제, 반복적 검색 패턴 |

---

## 포함 내용

```
everything-claude-code/
├── agents/           # 위임을 위한 전문 서브에이전트
│   ├── planner.md              # 기능 구현 계획
│   ├── architect.md            # 시스템 설계 결정
│   ├── tdd-guide.md            # 테스트 주도 개발 (JUnit5, MockK)
│   ├── code-reviewer.md        # 품질 및 보안 리뷰
│   ├── security-reviewer.md    # 취약점 분석
│   ├── build-error-resolver.md # Gradle/Kotlin 빌드 에러 해결
│   ├── gradle-build-resolver.md # Gradle 빌드 전문
│   ├── spring-test-guide.md    # Spring Boot 테스트 가이드
│   ├── refactor-cleaner.md     # 데드 코드 정리
│   └── doc-updater.md          # 문서 동기화
│
├── skills/           # 워크플로우 정의 및 도메인 지식
│   ├── coding-standards.md          # Kotlin/Java 모범 사례
│   ├── backend-patterns.md          # Spring Boot API, DB, 캐싱 패턴
│   ├── spring-boot-patterns.md      # Controller/Service/Repository 패턴
│   ├── kotlin-patterns.md           # Null safety, data class, 코루틴
│   ├── jooq-querydsl-patterns.md    # 타입 안전 쿼리 패턴
│   ├── spring-batch-patterns.md     # Job/Step 설계, Chunk 처리
│   ├── spring-security-patterns.md  # 인증/인가, JWT, Method Security
│   ├── continuous-learning/         # 세션에서 패턴 자동 추출
│   └── strategic-compact/           # 수동 압축 제안
│
├── commands/         # 빠른 실행을 위한 슬래시 명령어
│   ├── tdd.md              # /tdd - 테스트 주도 개발
│   ├── plan.md             # /plan - 구현 계획
│   ├── code-review.md      # /code-review - 품질 리뷰
│   ├── build-fix.md        # /build-fix - Gradle 빌드 에러 수정
│   ├── gradle-build.md     # /gradle-build - Gradle 빌드 실행
│   ├── spring-test.md      # /spring-test - Spring Boot 테스트
│   ├── test-coverage.md    # /test-coverage - JaCoCo 커버리지
│   ├── refactor-clean.md   # /refactor-clean - 데드 코드 제거
│   └── learn.md            # /learn - 세션 중 패턴 추출
│
├── rules/            # 항상 따라야 할 가이드라인
│   ├── security.md         # 필수 보안 체크, Spring Security
│   ├── coding-style.md     # Kotlin 불변성, 파일 구성
│   ├── testing.md          # TDD, 80% 커버리지, JUnit5/MockK
│   ├── git-workflow.md     # 커밋 형식, PR 프로세스
│   ├── agents.md           # 서브에이전트 위임 시점
│   └── performance.md      # 모델 선택, 컨텍스트 관리
│
├── hooks/            # 트리거 기반 자동화
│   ├── hooks.json                # 모든 훅 설정
│   ├── memory-persistence/       # 세션 라이프사이클 훅
│   │   ├── pre-compact.sh        # 압축 전 상태 저장
│   │   ├── session-start.sh      # 이전 컨텍스트 로드
│   │   └── session-end.sh        # 종료 시 학습 내용 저장
│   └── strategic-compact/        # 압축 제안
│
├── contexts/         # 동적 시스템 프롬프트 주입 컨텍스트
│   ├── kotlin-dev.md         # Kotlin 개발 컨텍스트
│   ├── spring-boot-dev.md    # Spring Boot 개발 컨텍스트
│   ├── dev.md                # 개발 모드 컨텍스트
│   ├── review.md             # 코드 리뷰 모드 컨텍스트
│   └── research.md           # 연구/탐색 모드 컨텍스트
│
├── examples/         # 예제 설정 및 세션
│   ├── CLAUDE.md             # 예제 프로젝트 설정
│   ├── user-CLAUDE.md        # 예제 사용자 설정
│   └── sessions/             # 예제 세션 로그 파일
│
├── mcp-configs/      # MCP 서버 설정
│   └── mcp-servers.json      # GitHub, PostgreSQL, Redis, Railway 등
│
└── plugins/          # 플러그인 생태계 문서
    └── README.md             # 플러그인, 마켓플레이스, 스킬 가이드
```

---

## 빠른 시작

### 1. 필요한 것 복사

```bash
# 저장소 클론
git clone https://github.com/affaan-m/everything-claude-code.git

# 에이전트를 Claude 설정에 복사
cp everything-claude-code/agents/*.md ~/.claude/agents/

# 규칙 복사
cp everything-claude-code/rules/*.md ~/.claude/rules/

# 명령어 복사
cp everything-claude-code/commands/*.md ~/.claude/commands/

# 스킬 복사
cp -r everything-claude-code/skills/* ~/.claude/skills/

# 컨텍스트 복사
cp everything-claude-code/contexts/*.md ~/.claude/contexts/
```

### 2. hooks를 settings.json에 추가

`hooks/hooks.json`의 훅을 `~/.claude/settings.json`에 복사합니다.

### 3. MCP 설정

`mcp-configs/mcp-servers.json`에서 원하는 MCP 서버를 `~/.claude.json`에 복사합니다.

**중요:** `YOUR_*_HERE` 플레이스홀더를 실제 API 키로 교체하세요.

### 4. 가이드 읽기

정말로, 가이드를 읽으세요. 컨텍스트와 함께 이 설정들이 10배 더 이해됩니다.

1. **[간략 가이드](https://x.com/affaanmustafa/status/2012378465664745795)** - 설정 및 기초
2. **[상세 가이드](https://x.com/affaanmustafa/status/2014040193557471352)** - 고급 기술 (토큰 최적화, 메모리 지속, 평가, 병렬화)

---

## 핵심 개념

### 대상 기술 스택

이 설정은 다음 기술 스택을 위해 최적화되어 있습니다:

- **언어:** Kotlin, Java
- **프레임워크:** Spring Boot, Spring Batch
- **빌드:** Gradle (Kotlin DSL)
- **데이터베이스:** jOOQ, QueryDSL, Spring Data JPA
- **테스트:** JUnit5, MockK, TestContainers
- **보안:** Spring Security, JWT

### 에이전트

서브에이전트는 제한된 범위의 위임된 작업을 처리합니다. 예시:

```markdown
---
name: build-error-resolver
description: Gradle/Kotlin 빌드 에러 해결 전문가
tools: Read, Write, Edit, Bash, Grep, Glob
model: opus
---

Gradle, Kotlin, Java 빌드 및 컴파일 에러를 해결하는 전문가입니다...
```

### 스킬

스킬은 명령어나 에이전트가 호출하는 워크플로우 정의입니다:

```markdown
# Spring Boot 패턴

## 계층 구조

Controller → Service → Repository → Database

### Controller
- HTTP 요청/응답 처리
- 입력 검증 (@Valid)
- ResponseEntity 반환

### Service
- 비즈니스 로직
- @Transactional 관리
- 예외 처리
```

### 훅

훅은 도구 이벤트에서 실행됩니다. 예시 - Kotlin 파일에서 println 경고:

```json
{
  "matcher": "tool == \"Edit\" && tool_input.file_path matches \"\\\\.kt$\"",
  "hooks": [{
    "type": "command",
    "command": "#!/bin/bash\ngrep -n 'println' \"$file_path\" && echo '[Hook] Remove println statements' >&2"
  }]
}
```

### 규칙

규칙은 항상 따라야 할 가이드라인입니다. 모듈화하세요:

```
~/.claude/rules/
  security.md      # 하드코딩된 비밀 금지, Spring Security
  coding-style.md  # Kotlin 불변성, 파일 제한
  testing.md       # TDD, JUnit5, 커버리지 요구사항
```

---

## 기여

**기여를 환영하고 권장합니다.**

이 저장소는 커뮤니티 리소스입니다. 다음이 있다면:
- 유용한 에이전트나 스킬
- 영리한 훅
- 더 나은 MCP 설정
- 개선된 규칙

기여해 주세요! 가이드라인은 [CONTRIBUTING.md](CONTRIBUTING.md)를 참조하세요.

### 기여 아이디어

- 언어별 스킬 (Python, Go, Rust 패턴)
- 프레임워크별 설정 (Django, Rails, Laravel)
- DevOps 에이전트 (Kubernetes, Terraform, AWS)
- 테스트 전략 (다양한 프레임워크)
- 도메인별 지식 (ML, 데이터 엔지니어링, 모바일)

---

## 중요 사항

### 컨텍스트 윈도우 관리

**중요:** 모든 MCP를 한 번에 활성화하지 마세요. 너무 많은 도구가 활성화되면 200k 컨텍스트 윈도우가 70k로 줄어들 수 있습니다.

경험칙:
- 20-30개 MCP 설정
- 프로젝트당 10개 미만 활성화
- 80개 미만 도구 활성

사용하지 않는 것은 프로젝트 설정의 `disabledMcpServers`로 비활성화하세요.

### 커스터마이제이션

이 설정은 제 워크플로우에 맞습니다. 다음을 권장합니다:
1. 공감되는 것부터 시작
2. 자신의 스택에 맞게 수정
3. 사용하지 않는 것 제거
4. 자신만의 패턴 추가

---

## 링크

- **간략 가이드 (시작):** [The Shorthand Guide to Everything Claude Code](https://x.com/affaanmustafa/status/2012378465664745795)
- **상세 가이드 (고급):** [The Longform Guide to Everything Claude Code](https://x.com/affaanmustafa/status/2014040193557471352)
- **팔로우:** [@affaanmustafa](https://x.com/affaanmustafa)
- **zenith.chat:** [zenith.chat](https://zenith.chat)

---

## 라이선스

MIT - 자유롭게 사용하고, 필요에 따라 수정하고, 가능하면 기여해 주세요.

---

**도움이 되었다면 스타를 눌러주세요. 두 가이드를 읽으세요. 멋진 것을 만드세요.**
