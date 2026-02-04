---
name: tdd-workflow
description: 새 기능 작성, 버그 수정, 코드 리팩토링 시 이 스킬을 사용하세요. 단위, 통합, 슬라이스 테스트를 포함하여 80%+ 커버리지로 테스트 주도 개발을 강제합니다.
---

# 테스트 주도 개발 워크플로우

이 스킬은 모든 코드 개발이 포괄적인 테스트 커버리지와 함께 TDD 원칙을 따르도록 보장합니다.

## 활성화 시점

- 새 기능 또는 기능 작성
- 버그 또는 이슈 수정
- 기존 코드 리팩토링
- API 엔드포인트 추가
- 새 컴포넌트 생성

## 핵심 원칙

### 1. 코드 전에 테스트
항상 먼저 테스트 작성, 그 다음 테스트를 통과시키는 코드 구현.

### 2. 커버리지 요구사항
- 최소 80% 커버리지 (단위 + 통합 + 슬라이스)
- 모든 엣지 케이스 커버
- 에러 시나리오 테스트
- 경계 조건 검증

### 3. 테스트 유형

#### 단위 테스트
- 개별 함수 및 유틸리티
- 컴포넌트 로직
- 순수 함수
- 헬퍼 및 유틸리티

#### 통합 테스트
- API 엔드포인트
- 데이터베이스 작업
- 서비스 상호작용
- 외부 API 호출

#### 슬라이스 테스트
- @WebMvcTest - Controller
- @DataJpaTest - Repository
- @SpringBatchTest - Batch Job

## TDD 워크플로우 단계

### 단계 1: 사용자 스토리 작성
```
[역할]로서, [행동]을 원합니다, 그래서 [이점]을 얻습니다

예시:
사용자로서, 시맨틱하게 시장을 검색하고 싶습니다,
정확한 키워드 없이도 관련 시장을 찾을 수 있도록.
```

### 단계 2: 테스트 케이스 생성
각 사용자 스토리에 대해 포괄적인 테스트 케이스 생성:

```kotlin
@ExtendWith(MockKExtension::class)
class MarketSearchServiceTest {

    @MockK
    private lateinit var marketRepository: MarketRepository

    @InjectMockKs
    private lateinit var marketSearchService: MarketSearchService

    @Test
    fun `쿼리에 대해 관련 시장을 반환한다`() {
        // given
        val query = "선거"
        every { marketRepository.search(query) } returns listOf(mockMarket)

        // when
        val result = marketSearchService.search(query)

        // then
        assertThat(result).hasSize(1)
        verify { marketRepository.search(query) }
    }

    @Test
    fun `빈 쿼리를 우아하게 처리한다`() {
        // 엣지 케이스 테스트
    }

    @Test
    fun `Repository 실패 시 폴백한다`() {
        // 폴백 동작 테스트
    }

    @Test
    fun `결과를 유사도 점수로 정렬한다`() {
        // 정렬 로직 테스트
    }
}
```

### 단계 3: 테스트 실행 (실패해야 함)
```bash
./gradlew test
# 테스트가 실패해야 함 - 아직 구현하지 않았으므로
```

### 단계 4: 코드 구현
테스트를 통과시키는 최소 코드 작성:

```kotlin
@Service
class MarketSearchService(
    private val marketRepository: MarketRepository
) {
    fun search(query: String): List<MarketResponse> {
        // 테스트에 의해 안내된 구현
    }
}
```

### 단계 5: 다시 테스트 실행
```bash
./gradlew test
# 테스트가 이제 통과해야 함
```

### 단계 6: 리팩토링
테스트를 녹색으로 유지하면서 코드 품질 개선:
- 중복 제거
- 네이밍 개선
- 성능 최적화
- 가독성 향상

### 단계 7: 커버리지 확인
```bash
./gradlew test jacocoTestReport
# 80%+ 커버리지 달성 확인
```

## 테스트 패턴

### 단위 테스트 패턴 (JUnit5/MockK)
```kotlin
@ExtendWith(MockKExtension::class)
class UserServiceTest {

    @MockK
    private lateinit var userRepository: UserRepository

    @MockK
    private lateinit var passwordEncoder: PasswordEncoder

    @InjectMockKs
    private lateinit var userService: UserService

    @Test
    fun `올바른 텍스트로 렌더링한다`() {
        // given
        val request = CreateUserRequest(...)
        every { passwordEncoder.encode(any()) } returns "encoded"
        every { userRepository.save(any()) } returns mockUser

        // when
        val result = userService.create(request)

        // then
        assertThat(result.email).isEqualTo(request.email)
        verify { passwordEncoder.encode(request.password) }
    }

    @Test
    fun `disabled가 true일 때 비활성화된다`() {
        // given
        val id = 1L
        every { userRepository.findByIdOrNull(id) } returns null

        // when/then
        assertThrows<UserNotFoundException> {
            userService.findById(id)
        }
    }
}
```

### API 통합 테스트 패턴
```kotlin
@WebMvcTest(UserController::class)
class UserControllerTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @MockkBean
    private lateinit var userService: UserService

    @Test
    fun `시장을 성공적으로 반환한다`() {
        // given
        every { userService.findById(1L) } returns mockUserResponse

        // when/then
        mockMvc.perform(get("/api/v1/users/1"))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.success").value(true))
            .andExpect(jsonPath("$.data.email").value("test@example.com"))
    }

    @Test
    fun `쿼리 파라미터를 검증한다`() {
        // given
        every { userService.findById(any()) } throws UserNotFoundException(1L)

        // when/then
        mockMvc.perform(get("/api/v1/users/1"))
            .andExpect(status().isNotFound)
    }
}
```

### 슬라이스 테스트 패턴 (Repository)
```kotlin
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class UserRepositoryTest {

    companion object {
        @Container
        val postgres = PostgreSQLContainer("postgres:15")

        @DynamicPropertySource
        @JvmStatic
        fun properties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
        }
    }

    @Autowired
    private lateinit var userRepository: UserRepository

    @Test
    fun `이메일로 사용자를 찾는다`() {
        // given
        val user = User(email = "test@example.com", name = "Test")
        userRepository.save(user)

        // when
        val found = userRepository.findByEmail("test@example.com")

        // then
        assertThat(found).isNotNull
        assertThat(found!!.email).isEqualTo("test@example.com")
    }
}
```

## 테스트 파일 구성

```
src/test/kotlin/com/example/
├── controller/
│   ├── UserControllerTest.kt      # 슬라이스 테스트
│   └── MarketControllerTest.kt
├── service/
│   ├── UserServiceTest.kt         # 단위 테스트
│   └── MarketServiceTest.kt
├── repository/
│   ├── UserRepositoryTest.kt      # 슬라이스 테스트
│   └── MarketRepositoryTest.kt
└── integration/
    ├── UserIntegrationTest.kt     # 통합 테스트
    └── MarketIntegrationTest.kt
```

## 외부 서비스 모킹

### Repository 모킹
```kotlin
@MockK
private lateinit var userRepository: UserRepository

every { userRepository.findByIdOrNull(1L) } returns User(...)
```

### 외부 API 모킹
```kotlin
@MockK
private lateinit var externalApiClient: ExternalApiClient

every { externalApiClient.call(any()) } returns ApiResponse(...)
```

## 테스트 커버리지 확인

### 커버리지 보고서 실행
```bash
./gradlew test jacocoTestReport
open build/reports/jacoco/test/html/index.html
```

### 커버리지 임계값
```kotlin
// build.gradle.kts
tasks.jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = "0.80".toBigDecimal()
            }
        }
    }
}
```

## 피해야 할 일반적인 테스트 실수

### ❌ 잘못됨: 구현 세부사항 테스트
```kotlin
// 내부 상태 테스트하지 않기
assertThat(service.internalState).isEqualTo(5)
```

### ✅ 올바름: 사용자 가시적 동작 테스트
```kotlin
// 사용자가 보는 것 테스트
assertThat(result.message).isEqualTo("성공")
```

### ❌ 잘못됨: 테스트 격리 없음
```kotlin
// 테스트가 서로 의존
@Test fun `사용자 생성`() { /* ... */ }
@Test fun `같은 사용자 업데이트`() { /* 이전 테스트에 의존 */ }
```

### ✅ 올바름: 독립적인 테스트
```kotlin
// 각 테스트가 자체 데이터 설정
@Test fun `사용자 생성`() {
    val user = createTestUser()
    // 테스트 로직
}

@Test fun `사용자 업데이트`() {
    val user = createTestUser()
    // 업데이트 로직
}
```

## 지속적 테스트

### 개발 중 Watch 모드
```bash
./gradlew test --continuous
# 파일 변경 시 테스트 자동 실행
```

### Pre-Commit 훅
```bash
# 모든 커밋 전에 실행
./gradlew test && ./gradlew ktlintCheck
```

### CI/CD 통합
```yaml
# GitHub Actions
- name: 테스트 실행
  run: ./gradlew test --info
- name: 커버리지 업로드
  uses: codecov/codecov-action@v3
```

## 모범 사례

1. **먼저 테스트 작성** - 항상 TDD
2. **테스트당 하나의 Assert** - 단일 동작에 집중
3. **설명적 테스트 이름** - 무엇을 테스트하는지 설명
4. **Arrange-Act-Assert** - 명확한 테스트 구조
5. **외부 의존성 모킹** - 단위 테스트 격리
6. **엣지 케이스 테스트** - Null, 빈 값, 큰 값
7. **에러 경로 테스트** - 정상 경로만이 아님
8. **테스트 빠르게 유지** - 단위 테스트 각 50ms 미만
9. **테스트 후 정리** - 부작용 없음
10. **커버리지 보고서 검토** - 갭 식별

## 성공 지표

- 80%+ 코드 커버리지 달성
- 모든 테스트 통과 (녹색)
- 건너뛰거나 비활성화된 테스트 없음
- 빠른 테스트 실행 (단위 테스트 < 30초)
- 슬라이스 테스트가 중요 계층 커버
- 테스트가 프로덕션 전에 버그 발견

---

**기억**: 테스트는 선택 사항이 아닙니다. 자신감 있는 리팩토링, 빠른 개발, 프로덕션 신뢰성을 가능하게 하는 안전망입니다.
