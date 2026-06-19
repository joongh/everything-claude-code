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

## JUnit5 Kotlin 테스트 작성 주의 (중요)

### `@Nested inner class`의 이름에 한글/비-ASCII 문자를 사용하지 않는다

Kotlin에서 `@Nested inner class `CREATE eventAction - parent_resource_id 결정`` 처럼 클래스 이름에 한글을 쓰면, 컴파일러는 `.class` 파일명에 해당 문자를 그대로 넣는다. 로컬(macOS/APFS UTF-8)에서는 통과하지만 **CI 러너(Linux, 기본 C/POSIX locale)에서 `NoClassDefFoundError` 발생**:

```
Caused by: java.lang.NoClassDefFoundError:
  com/nhncloudservice/cashier/...$CREATE eventAction - parent_resource_id ??
  (wrong name: ...$CREATE eventAction - parent_resource_id 결정)
```

**원인**: Linux CI 러너의 file.encoding/JVM charset이 UTF-8이 아니면 `.class` 파일을 읽을 때 한글 바이트가 `?`로 치환되어 내부 클래스명 일치 실패.

**해결 방법 (선호 순)**:
1. **평탄화**: `@Nested inner class`를 없애고 일반 `@Test` 메서드만 사용. 메서드 이름(Kotlin function name)에는 한글을 써도 됨(JVM이 UTF-8로 인코딩된 메서드 이름을 정상 처리).
2. **영문 클래스 이름**: `@Nested inner class`가 꼭 필요하면 이름을 ASCII로. 예: `inner class CreateEventAction`
3. **테스트 그룹화**: 주석 + 빈 줄로 섹션 구분 (`// --- CREATE eventAction ---`)

**메서드 이름에 한글은 안전**:
```kotlin
@Test
fun `CREATE - attach_status=attached이면 parent=instance_uuid`() { ... }  // O 안전
```

**클래스 이름에 한글은 CI에서 깨짐**:
```kotlin
@Nested
inner class `CREATE eventAction - parent_resource_id 결정` {  // X CI에서 실패
    @Test fun `...`() { }
}
```

### 메서드 이름에 `%`, 공백, 특수문자 주의

Kotlin 컴파일러가 Windows 호환성 경고를 발생시킬 수 있다:
```
w: Name contains character(s) that can cause problems on Windows: %
```
경고는 무시해도 되지만, 가능한 일반적인 문자만 사용한다.

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
