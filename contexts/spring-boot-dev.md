# Spring Boot 개발 컨텍스트

Spring Boot 프로젝트에서 자동으로 적용되는 컨텍스트입니다.

## 프레임워크 버전

- Spring Boot 3.2+
- Spring Framework 6.1+
- Jakarta EE (javax 아님)

## 계층 아키텍처

```
Controller → Service → Repository → Database
```

### Controller
```kotlin
@RestController
@RequestMapping("/api/v1/users")
class UserController(
    private val userService: UserService
) {
    @GetMapping("/{id}")
    fun findById(@PathVariable id: Long): ResponseEntity<UserResponse> {
        return ResponseEntity.ok(userService.findById(id))
    }

    @PostMapping
    fun create(@Valid @RequestBody request: CreateUserRequest): ResponseEntity<UserResponse> {
        val user = userService.create(request)
        return ResponseEntity.created(URI.create("/api/v1/users/${user.id}")).body(user)
    }
}
```

### Service
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
    }
}
```

### Repository
```kotlin
interface UserRepository : JpaRepository<User, Long> {
    fun findByEmail(email: String): User?
    fun existsByEmail(email: String): Boolean
}
```

## 의존성 주입

```kotlin
// ✅ 생성자 주입 (권장)
@Service
class UserService(
    private val userRepository: UserRepository,
    private val passwordEncoder: PasswordEncoder
)

// ❌ 필드 주입 피할 것
@Autowired
private lateinit var userRepository: UserRepository
```

## 트랜잭션 관리

```kotlin
@Service
@Transactional(readOnly = true)  // 클래스 레벨: 읽기 전용 기본
class OrderService {

    @Transactional  // 쓰기 작업
    fun createOrder(request: CreateOrderRequest): Order {
        // ...
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)  // 별도 트랜잭션
    fun processPayment(orderId: Long) {
        // ...
    }
}
```

## 예외 처리

```kotlin
// 커스텀 예외
sealed class BusinessException(
    val errorCode: ErrorCode,
    override val message: String
) : RuntimeException(message)

class UserNotFoundException(id: Long) : BusinessException(
    ErrorCode.USER_NOT_FOUND,
    "사용자를 찾을 수 없습니다: $id"
)

// 전역 예외 핸들러
@RestControllerAdvice
class GlobalExceptionHandler {
    @ExceptionHandler(BusinessException::class)
    fun handleBusinessException(e: BusinessException): ResponseEntity<ErrorResponse> {
        return ResponseEntity
            .status(e.errorCode.status)
            .body(ErrorResponse(e.errorCode.name, e.message))
    }
}
```

## 입력 검증

```kotlin
data class CreateUserRequest(
    @field:NotBlank(message = "이메일은 필수입니다")
    @field:Email(message = "올바른 이메일 형식이 아닙니다")
    val email: String,

    @field:NotBlank(message = "비밀번호는 필수입니다")
    @field:Size(min = 8, message = "비밀번호는 8자 이상이어야 합니다")
    val password: String
)
```

## 설정 관리

```kotlin
@ConfigurationProperties(prefix = "app")
data class AppProperties(
    val jwt: JwtProperties,
    val cache: CacheProperties
) {
    data class JwtProperties(
        val secret: String,
        val expirationMs: Long = 3600000
    )
}
```

```yaml
# application.yml
app:
  jwt:
    secret: ${JWT_SECRET}
    expiration-ms: 3600000
```

## 테스트 어노테이션

| 어노테이션 | 용도 | 로드 범위 |
|-----------|------|----------|
| `@SpringBootTest` | 통합 테스트 | 전체 컨텍스트 |
| `@WebMvcTest` | Controller 테스트 | Web 계층만 |
| `@DataJpaTest` | Repository 테스트 | JPA 계층만 |
| `@SpringBatchTest` | Batch Job 테스트 | Batch 계층 |

## Gradle 명령어

```bash
# 빌드
./gradlew build

# 테스트
./gradlew test

# 실행
./gradlew bootRun

# 프로필 지정 실행
./gradlew bootRun --args='--spring.profiles.active=dev'

# 실행 가능한 JAR 생성
./gradlew bootJar
```

## 프로젝트 구조

```
src/main/kotlin/com/example/
├── config/           # 설정 클래스
├── controller/       # REST Controller
├── service/          # 비즈니스 로직
├── repository/       # 데이터 접근
├── domain/           # Entity, VO
├── dto/              # Request/Response
├── exception/        # 커스텀 예외
└── util/             # 유틸리티

src/main/resources/
├── application.yml
├── application-dev.yml
├── application-prod.yml
└── db/migration/     # Flyway 마이그레이션
```
