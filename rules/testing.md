# 테스트 요구사항

## 최소 테스트 커버리지: 80%

테스트 유형 (모두 필수):
1. **단위 테스트** - 개별 함수, 유틸리티, 서비스 (MockK)
2. **통합 테스트** - API 엔드포인트, 데이터베이스 작업 (@SpringBootTest)
3. **슬라이스 테스트** - Controller, Repository 계층별 테스트

## 테스트 주도 개발

필수 워크플로우:
1. 테스트 먼저 작성 (RED)
2. 테스트 실행 - 실패해야 함
3. 최소 구현 작성 (GREEN)
4. 테스트 실행 - 통과해야 함
5. 리팩토링 (IMPROVE)
6. 커버리지 확인 (80%+)

## 테스트 구조 (Given-When-Then)

```kotlin
@Test
fun `사용자 생성 시 비밀번호를 암호화한다`() {
    // given
    val request = CreateUserRequest(
        email = "test@example.com",
        password = "password123",
        name = "Test User"
    )
    every { userRepository.existsByEmail(any()) } returns false
    every { passwordEncoder.encode(any()) } returns "encoded"
    every { userRepository.save(any()) } answers { firstArg() }

    // when
    val result = userService.create(request)

    // then
    assertThat(result.email).isEqualTo(request.email)
    verify { passwordEncoder.encode(request.password) }
}
```

## 테스트 실패 시 문제 해결

1. **spring-test-guide** 에이전트 사용
2. 테스트 격리 확인
3. Mock이 올바른지 확인
4. 테스트가 아닌 구현 수정 (테스트가 틀린 경우 제외)

## 에이전트 지원

- **tdd-guide** - 새 기능에 적극적으로 사용, 테스트 먼저 작성 강제
- **spring-test-guide** - Spring Boot 테스트 전문가

## 테스트 유형별 어노테이션

| 테스트 유형 | 어노테이션 | 용도 |
|-----------|-----------|------|
| 단위 테스트 | `@ExtendWith(MockKExtension::class)` | Service, Util |
| Controller | `@WebMvcTest` | API 엔드포인트 |
| Repository | `@DataJpaTest` | JPA 쿼리 |
| 통합 테스트 | `@SpringBootTest` | 전체 컨텍스트 |
| Batch | `@SpringBatchTest` | Job, Step |
| 실제 DB | `@Testcontainers` | PostgreSQL, Redis |

## 커버리지 명령어

```bash
# 테스트 + 커버리지 리포트
./gradlew test jacocoTestReport

# 커버리지 검증
./gradlew jacocoTestCoverageVerification

# 리포트 확인
open build/reports/jacoco/test/html/index.html
```
