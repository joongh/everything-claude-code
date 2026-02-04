# 프로젝트 가이드라인 스킬 (예시)

이것은 프로젝트별 스킬의 예시입니다. 자신의 프로젝트 템플릿으로 사용하세요.

실제 프로덕션 애플리케이션 기반: [Zenith](https://zenith.chat) - AI 기반 고객 발굴 플랫폼.

---

## 사용 시점

이 스킬을 참조할 때: 해당 프로젝트에서 작업할 때. 프로젝트 스킬에 포함되는 내용:
- 아키텍처 개요
- 파일 구조
- 코드 패턴
- 테스트 요구사항
- 배포 워크플로우

---

## 아키텍처 개요

**기술 스택:**
- **백엔드**: Spring Boot 3.2 (Kotlin)
- **데이터베이스**: PostgreSQL + jOOQ
- **캐시**: Redis
- **빌드**: Gradle (Kotlin DSL)
- **테스트**: JUnit5, MockK, TestContainers

**서비스:**
```
┌─────────────────────────────────────────────────────────────┐
│                         API Server                          │
│  Spring Boot 3.2 + Kotlin + jOOQ                           │
│  배포: Kubernetes / Docker                                  │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │PostgreSQL│   │  Redis   │   │ External │
        │ Database │   │  Cache   │   │   APIs   │
        └──────────┘   └──────────┘   └──────────┘
```

---

## 파일 구조

```
project/
├── src/main/kotlin/com/example/
│   ├── config/           # 설정 클래스
│   ├── controller/       # REST Controller
│   ├── service/          # 비즈니스 로직
│   ├── repository/       # 데이터 접근
│   ├── domain/           # Entity, VO
│   ├── dto/              # Request/Response
│   ├── exception/        # 커스텀 예외
│   └── util/             # 유틸리티
│
├── src/main/resources/
│   ├── application.yml
│   ├── application-dev.yml
│   ├── application-prod.yml
│   └── db/migration/     # Flyway 마이그레이션
│
├── src/test/kotlin/com/example/
│   ├── controller/       # Controller 테스트
│   ├── service/          # Service 테스트
│   └── integration/      # 통합 테스트
│
├── build.gradle.kts
└── docker-compose.yml
```

---

## 코드 패턴

### API 응답 형식 (Kotlin)

```kotlin
data class ApiResponse<T>(
    val success: Boolean,
    val data: T? = null,
    val error: String? = null
) {
    companion object {
        fun <T> ok(data: T) = ApiResponse(success = true, data = data)
        fun <T> fail(error: String) = ApiResponse<T>(success = false, error = error)
    }
}
```

### Controller 패턴

```kotlin
@RestController
@RequestMapping("/api/v1/users")
class UserController(
    private val userService: UserService
) {
    @GetMapping("/{id}")
    fun findById(@PathVariable id: Long): ResponseEntity<ApiResponse<UserResponse>> {
        val user = userService.findById(id)
        return ResponseEntity.ok(ApiResponse.ok(user))
    }

    @PostMapping
    fun create(@Valid @RequestBody request: CreateUserRequest): ResponseEntity<ApiResponse<UserResponse>> {
        val user = userService.create(request)
        return ResponseEntity.created(URI.create("/api/v1/users/${user.id}"))
            .body(ApiResponse.ok(user))
    }
}
```

### Service 패턴

```kotlin
@Service
@Transactional(readOnly = true)
class UserService(
    private val userRepository: UserRepository
) {
    fun findById(id: Long): UserResponse {
        val user = userRepository.findByIdOrNull(id)
            ?: throw UserNotFoundException(id)
        return UserResponse.from(user)
    }

    @Transactional
    fun create(request: CreateUserRequest): UserResponse {
        // 비즈니스 로직
        val user = userRepository.save(request.toEntity())
        return UserResponse.from(user)
    }
}
```

---

## 테스트 요구사항

### 백엔드 (JUnit5 + MockK)

```bash
# 모든 테스트 실행
./gradlew test

# 커버리지와 함께
./gradlew test jacocoTestReport

# 특정 테스트 파일 실행
./gradlew test --tests "UserServiceTest"
```

**테스트 구조:**
```kotlin
@ExtendWith(MockKExtension::class)
class UserServiceTest {

    @MockK
    private lateinit var userRepository: UserRepository

    @InjectMockKs
    private lateinit var userService: UserService

    @Test
    fun `사용자 생성 시 비밀번호를 암호화한다`() {
        // given
        val request = CreateUserRequest(...)
        every { userRepository.save(any()) } returns mockUser

        // when
        val result = userService.create(request)

        // then
        assertThat(result.email).isEqualTo(request.email)
        verify { userRepository.save(any()) }
    }
}
```

### 통합 테스트 (TestContainers)

```kotlin
@SpringBootTest
@Testcontainers
class UserIntegrationTest {

    companion object {
        @Container
        val postgres = PostgreSQLContainer("postgres:15")
            .withDatabaseName("test")

        @DynamicPropertySource
        @JvmStatic
        fun properties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
        }
    }

    @Test
    fun `사용자 생성 및 조회`() {
        // 통합 테스트 로직
    }
}
```

---

## 배포 워크플로우

### 사전 배포 체크리스트

- [ ] 모든 테스트 로컬 통과
- [ ] `./gradlew build` 성공
- [ ] 하드코딩된 비밀 없음
- [ ] 환경 변수 문서화됨
- [ ] 데이터베이스 마이그레이션 준비됨

### 배포 명령어

```bash
# 빌드
./gradlew bootJar

# Docker 이미지 빌드
docker build -t myapp:latest .

# Docker Compose로 실행
docker-compose up -d
```

### 환경 변수

```bash
# 필수
SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/myapp
SPRING_DATASOURCE_USERNAME=
SPRING_DATASOURCE_PASSWORD=
JWT_SECRET=

# 선택
SPRING_PROFILES_ACTIVE=dev
REDIS_HOST=localhost
REDIS_PORT=6379
```

---

## 중요 규칙

1. **이모지 금지** - 코드, 주석, 문서에
2. **불변성** - 객체나 배열 변경하지 않음
3. **TDD** - 구현 전 테스트 작성
4. **80% 커버리지** 최소
5. **다수의 작은 파일** - 200-400줄 일반적, 최대 800줄
6. **println 금지** - 프로덕션 코드에
7. **적절한 에러 처리** - try/catch로
8. **입력 검증** - Bean Validation으로

---

## 관련 스킬

- `coding-standards.md` - 일반 코딩 모범 사례
- `backend-patterns.md` - API 및 데이터베이스 패턴
- `spring-boot-patterns.md` - Spring Boot 패턴
- `tdd-workflow/` - 테스트 주도 개발 방법론
