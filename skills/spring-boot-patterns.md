# Spring Boot 패턴

Spring Boot 애플리케이션의 핵심 아키텍처 패턴과 모범 사례를 정의합니다.

## 계층 구조

### Controller 계층
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
        val location = URI.create("/api/v1/users/${user.id}")
        return ResponseEntity.created(location).body(user)
    }

    @PutMapping("/{id}")
    fun update(
        @PathVariable id: Long,
        @Valid @RequestBody request: UpdateUserRequest
    ): ResponseEntity<UserResponse> {
        return ResponseEntity.ok(userService.update(id, request))
    }

    @DeleteMapping("/{id}")
    fun delete(@PathVariable id: Long): ResponseEntity<Unit> {
        userService.delete(id)
        return ResponseEntity.noContent().build()
    }
}
```

### Service 계층
```kotlin
@Service
@Transactional(readOnly = true)
class UserService(
    private val userRepository: UserRepository,
    private val passwordEncoder: PasswordEncoder
) {
    fun findById(id: Long): UserResponse {
        val user = userRepository.findByIdOrNull(id)
            ?: throw UserNotFoundException(id)
        return UserResponse.from(user)
    }

    @Transactional
    fun create(request: CreateUserRequest): UserResponse {
        validateDuplicateEmail(request.email)

        val user = User(
            email = request.email,
            password = passwordEncoder.encode(request.password),
            name = request.name
        )
        return UserResponse.from(userRepository.save(user))
    }

    @Transactional
    fun update(id: Long, request: UpdateUserRequest): UserResponse {
        val user = userRepository.findByIdOrNull(id)
            ?: throw UserNotFoundException(id)

        user.update(request.name, request.email)
        return UserResponse.from(user)
    }

    @Transactional
    fun delete(id: Long) {
        if (!userRepository.existsById(id)) {
            throw UserNotFoundException(id)
        }
        userRepository.deleteById(id)
    }

    private fun validateDuplicateEmail(email: String) {
        if (userRepository.existsByEmail(email)) {
            throw DuplicateEmailException(email)
        }
    }
}
```

### Repository 계층
```kotlin
interface UserRepository : JpaRepository<User, Long> {
    fun findByEmail(email: String): User?
    fun existsByEmail(email: String): Boolean

    @Query("SELECT u FROM User u WHERE u.status = :status")
    fun findAllByStatus(@Param("status") status: UserStatus): List<User>
}
```

## 의존성 주입

### 생성자 주입 (권장)
```kotlin
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val userService: UserService,
    private val eventPublisher: ApplicationEventPublisher
) {
    // ...
}
```

### 조건부 빈 등록
```kotlin
@Configuration
class CacheConfig {
    @Bean
    @ConditionalOnProperty(name = ["cache.enabled"], havingValue = "true")
    fun cacheManager(): CacheManager {
        return CaffeineCacheManager().apply {
            setCaffeine(
                Caffeine.newBuilder()
                    .maximumSize(1000)
                    .expireAfterWrite(Duration.ofMinutes(10))
            )
        }
    }
}
```

## 예외 처리

### 커스텀 예외 정의
```kotlin
sealed class BusinessException(
    val errorCode: ErrorCode,
    override val message: String
) : RuntimeException(message)

class UserNotFoundException(id: Long) : BusinessException(
    errorCode = ErrorCode.USER_NOT_FOUND,
    message = "사용자를 찾을 수 없습니다: $id"
)

class DuplicateEmailException(email: String) : BusinessException(
    errorCode = ErrorCode.DUPLICATE_EMAIL,
    message = "이미 존재하는 이메일입니다: $email"
)

enum class ErrorCode(val status: HttpStatus) {
    USER_NOT_FOUND(HttpStatus.NOT_FOUND),
    DUPLICATE_EMAIL(HttpStatus.CONFLICT),
    INVALID_REQUEST(HttpStatus.BAD_REQUEST),
    INTERNAL_ERROR(HttpStatus.INTERNAL_SERVER_ERROR)
}
```

### 전역 예외 핸들러
```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    private val log = LoggerFactory.getLogger(javaClass)

    @ExceptionHandler(BusinessException::class)
    fun handleBusinessException(e: BusinessException): ResponseEntity<ErrorResponse> {
        log.warn("Business exception: ${e.message}")
        return ResponseEntity
            .status(e.errorCode.status)
            .body(ErrorResponse(e.errorCode.name, e.message))
    }

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidationException(e: MethodArgumentNotValidException): ResponseEntity<ErrorResponse> {
        val errors = e.bindingResult.fieldErrors
            .associate { it.field to (it.defaultMessage ?: "Invalid value") }

        return ResponseEntity
            .badRequest()
            .body(ErrorResponse("VALIDATION_ERROR", "입력값이 올바르지 않습니다", errors))
    }

    @ExceptionHandler(Exception::class)
    fun handleException(e: Exception): ResponseEntity<ErrorResponse> {
        log.error("Unexpected error", e)
        return ResponseEntity
            .internalServerError()
            .body(ErrorResponse("INTERNAL_ERROR", "서버 오류가 발생했습니다"))
    }
}

data class ErrorResponse(
    val code: String,
    val message: String,
    val details: Map<String, String>? = null
)
```

## DTO 패턴

### Request/Response 분리
```kotlin
// Request DTO
data class CreateUserRequest(
    @field:NotBlank(message = "이메일은 필수입니다")
    @field:Email(message = "올바른 이메일 형식이 아닙니다")
    val email: String,

    @field:NotBlank(message = "비밀번호는 필수입니다")
    @field:Size(min = 8, max = 100, message = "비밀번호는 8자 이상이어야 합니다")
    val password: String,

    @field:NotBlank(message = "이름은 필수입니다")
    val name: String
)

// Response DTO
data class UserResponse(
    val id: Long,
    val email: String,
    val name: String,
    val createdAt: LocalDateTime
) {
    companion object {
        fun from(user: User) = UserResponse(
            id = user.id!!,
            email = user.email,
            name = user.name,
            createdAt = user.createdAt
        )
    }
}
```

## 설정 관리

### @ConfigurationProperties
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

    data class CacheProperties(
        val enabled: Boolean = true,
        val ttlMinutes: Int = 10
    )
}

@Configuration
@EnableConfigurationProperties(AppProperties::class)
class AppConfig
```

### application.yml
```yaml
app:
  jwt:
    secret: ${JWT_SECRET}
    expiration-ms: 3600000
  cache:
    enabled: true
    ttl-minutes: 10

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/myapp
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        format_sql: true
        default_batch_fetch_size: 100
```

## 비동기 처리

### @Async 사용
```kotlin
@Configuration
@EnableAsync
class AsyncConfig {
    @Bean
    fun taskExecutor(): TaskExecutor {
        return ThreadPoolTaskExecutor().apply {
            corePoolSize = 5
            maxPoolSize = 10
            queueCapacity = 100
            setThreadNamePrefix("async-")
            initialize()
        }
    }
}

@Service
class NotificationService(
    private val emailSender: EmailSender
) {
    @Async
    fun sendWelcomeEmail(user: User) {
        emailSender.send(
            to = user.email,
            subject = "환영합니다!",
            body = "서비스에 가입해 주셔서 감사합니다."
        )
    }
}
```

### 이벤트 기반 처리
```kotlin
// 이벤트 정의
data class UserCreatedEvent(
    val userId: Long,
    val email: String
)

// 이벤트 발행
@Service
class UserService(
    private val eventPublisher: ApplicationEventPublisher
) {
    @Transactional
    fun create(request: CreateUserRequest): UserResponse {
        val user = userRepository.save(User(...))
        eventPublisher.publishEvent(UserCreatedEvent(user.id!!, user.email))
        return UserResponse.from(user)
    }
}

// 이벤트 리스너
@Component
class UserEventListener(
    private val notificationService: NotificationService
) {
    @Async
    @EventListener
    fun handleUserCreated(event: UserCreatedEvent) {
        notificationService.sendWelcomeEmail(event.userId)
    }
}
```
