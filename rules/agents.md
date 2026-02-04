# 에이전트 오케스트레이션

## 사용 가능한 에이전트

`~/.claude/agents/`에 위치:

| 에이전트 | 목적 | 사용 시점 |
|---------|------|----------|
| planner | 구현 계획 | 복잡한 기능, 리팩토링 |
| architect | 시스템 설계 | 아키텍처 결정 |
| tdd-guide | 테스트 주도 개발 | 새 기능, 버그 수정 |
| code-reviewer | 코드 리뷰 | 코드 작성 후 |
| security-reviewer | 보안 분석 | 커밋 전 |
| build-error-resolver | 빌드 에러 수정 | 빌드 실패 시 |
| gradle-build-resolver | Gradle 빌드 전문 | Gradle 에러 시 |
| spring-test-guide | Spring Boot 테스트 | Spring 테스트 |
| refactor-cleaner | 데드 코드 정리 | 코드 유지보수 |
| doc-updater | 문서화 | 문서 업데이트 |

## 즉시 에이전트 사용

사용자 프롬프트 필요 없음:
1. 복잡한 기능 요청 - **planner** 에이전트 사용
2. 코드 방금 작성/수정 - **code-reviewer** 에이전트 사용
3. 버그 수정 또는 새 기능 - **tdd-guide** 에이전트 사용
4. 아키텍처 결정 - **architect** 에이전트 사용

## 병렬 작업 실행

독립적인 작업에 항상 병렬 Task 실행 사용:

```markdown
# 좋음: 병렬 실행
3개 에이전트 병렬 실행:
1. 에이전트 1: auth.kt 보안 분석
2. 에이전트 2: 캐시 시스템 성능 검토
3. 에이전트 3: utils.kt 타입 검사

# 나쁨: 불필요한 순차 실행
에이전트 1 먼저, 그 다음 에이전트 2, 그 다음 에이전트 3
```

## 다중 관점 분석

복잡한 문제에 분할 역할 서브 에이전트 사용:
- 사실 검토자
- 시니어 엔지니어
- 보안 전문가
- 일관성 검토자
- 중복 검사자
