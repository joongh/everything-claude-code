# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 저장소 개요

Spring Boot/Kotlin/Java 백엔드 개발자를 위한 Claude Code 구성 파일(agents, skills, commands, hooks, rules, MCP configs)을 모아둔 문서/설정 저장소입니다. 빌드 시스템이나 테스트가 없으며, 기여는 markdown과 JSON 파일로 이루어집니다.

### 대상 기술 스택
- Spring Boot, Spring Batch
- Kotlin, Java
- Gradle 빌드
- jOOQ, QueryDSL (데이터베이스 쿼리)
- JUnit5, MockK, TestContainers (테스트)
- 백엔드 전용 (프론트엔드 없음)

## 저장소 구조

- `agents/` - 서브에이전트 정의 (frontmatter: name, description, tools, model)
- `skills/` - 워크플로우 정의 및 도메인 지식 (단일 .md 또는 SKILL.md 포함 디렉토리)
- `commands/` - 슬래시 명령어 정의 (frontmatter 필수)
- `rules/` - 항상 따라야 할 가이드라인 (모듈화된 규칙 파일)
- `hooks/` - 도구 이벤트 자동화용 훅 설정 및 셸 스크립트
- `contexts/` - 동적 시스템 프롬프트 주입 컨텍스트
- `mcp-configs/` - MCP 서버 설정 (JSON)
- `examples/` - 예제 설정 파일
- `plugins/` - 플러그인 생태계 문서

## 파일 형식 규칙

**Agents** frontmatter 필수:
```markdown
---
name: agent-name
description: 설명
tools: Read, Grep, Glob, Bash
model: sonnet
---
```

**Commands** frontmatter 필수:
```markdown
---
description: 명령어 설명
---
```

**Skills**는 단일 .md 파일 또는 SKILL.md를 포함한 디렉토리로 구성.

**Hooks**는 hooks.json에서 matcher 표현식과 훅 정의 사용.

## 주요 구성 요소

### Agents
- `build-error-resolver.md` - Gradle/Kotlin 빌드 에러 해결
- `tdd-guide.md` - TDD 워크플로우 (JUnit5, MockK)
- `gradle-build-resolver.md` - Gradle 빌드 전문
- `spring-test-guide.md` - Spring Boot 테스트

### Skills
- `spring-boot-patterns.md` - Controller/Service/Repository 패턴
- `kotlin-patterns.md` - Null safety, data class, 코루틴
- `jooq-querydsl-patterns.md` - 타입 안전 쿼리
- `spring-batch-patterns.md` - Job/Step 설계
- `spring-security-patterns.md` - 인증/인가, JWT

### Commands
- `/build-fix` - Gradle 빌드 에러 수정
- `/tdd` - TDD 워크플로우
- `/test-coverage` - JaCoCo 커버리지
- `/gradle-build` - Gradle 빌드 실행
- `/spring-test` - Spring Boot 테스트

## 기여 가이드라인

- 파일명은 소문자와 하이픈 사용: `spring-batch-patterns.md`
- 설정은 집중적이고 모듈화되게 유지
- API 키, 토큰, 민감한 경로 절대 포함 금지
- 제출 전 Claude Code로 설정 테스트 필수
- 각 디렉토리의 기존 패턴 따르기
- 모든 코드 예시는 Kotlin/Java로 작성
