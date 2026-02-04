# 보안 가이드라인

## 필수 보안 체크

커밋 전 확인:
- [ ] 하드코딩된 비밀 없음 (API 키, 비밀번호, 토큰)
- [ ] 모든 사용자 입력 검증됨
- [ ] SQL 인젝션 방지 (파라미터화된 쿼리)
- [ ] XSS 방지 (HTML 이스케이프)
- [ ] CSRF 보호 활성화됨
- [ ] 인증/인가 확인됨
- [ ] 모든 엔드포인트에 Rate limiting
- [ ] 에러 메시지가 민감한 데이터 누출 안 함

## 비밀 관리

```kotlin
// ❌ 절대 금지: 하드코딩된 비밀
val apiKey = "sk-proj-xxxxx"

// ✅ 항상: 환경 변수
@Value("\${jwt.secret}")
private lateinit var jwtSecret: String

// ✅ 또는: ConfigurationProperties
@ConfigurationProperties(prefix = "app")
data class AppProperties(
    val jwt: JwtProperties
) {
    data class JwtProperties(
        val secret: String,
        val expirationMs: Long
    )
}
```

## Spring Security 패턴

### SecurityFilterChain 설정
```kotlin
@Bean
fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
    return http
        .csrf { it.disable() }  // REST API의 경우
        .sessionManagement {
            it.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        }
        .authorizeHttpRequests { auth ->
            auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
        }
        .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter::class.java)
        .build()
}
```

### Method Security
```kotlin
@PreAuthorize("hasRole('ADMIN')")
fun deleteUser(id: Long) { ... }

@PreAuthorize("#userId == authentication.principal.userId")
fun getUserData(userId: Long) { ... }

@PostAuthorize("returnObject.userId == authentication.principal.userId")
fun findOrder(orderId: Long): Order { ... }
```

## 입력 검증

### Bean Validation
```kotlin
data class CreateUserRequest(
    @field:NotBlank
    @field:Email
    val email: String,

    @field:NotBlank
    @field:Size(min = 8, max = 100)
    @field:Pattern(
        regexp = "^(?=.*[A-Za-z])(?=.*\\d).*$",
        message = "비밀번호는 문자와 숫자를 포함해야 합니다"
    )
    val password: String
)
```

### Controller에서 검증
```kotlin
@PostMapping
fun create(@Valid @RequestBody request: CreateUserRequest): ResponseEntity<*> {
    // @Valid가 자동으로 검증
    return ResponseEntity.ok(userService.create(request))
}
```

## SQL 인젝션 방지

```kotlin
// ❌ 위험: 문자열 연결
val query = "SELECT * FROM users WHERE email = '$email'"

// ✅ 안전: Spring Data JPA
@Query("SELECT u FROM User u WHERE u.email = :email")
fun findByEmail(@Param("email") email: String): User?

// ✅ 안전: jOOQ
dsl.selectFrom(USERS)
    .where(USERS.EMAIL.eq(email))
    .fetchOne()

// ✅ 안전: QueryDSL
queryFactory.selectFrom(user)
    .where(user.email.eq(email))
    .fetchOne()
```

## 비밀번호 보안

```kotlin
@Service
class UserService(
    private val passwordEncoder: PasswordEncoder
) {
    fun create(request: CreateUserRequest): User {
        val user = User(
            email = request.email,
            // 항상 암호화
            password = passwordEncoder.encode(request.password)
        )
        return userRepository.save(user)
    }

    fun authenticate(email: String, password: String): Boolean {
        val user = userRepository.findByEmail(email)
            ?: return false
        return passwordEncoder.matches(password, user.password)
    }
}
```

## Rate Limiting

```kotlin
@Component
class RateLimitFilter : OncePerRequestFilter() {

    private val buckets = ConcurrentHashMap<String, Bucket>()

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        chain: FilterChain
    ) {
        val ip = request.remoteAddr
        val bucket = buckets.computeIfAbsent(ip) { createBucket() }

        if (bucket.tryConsume(1)) {
            chain.doFilter(request, response)
        } else {
            response.status = HttpStatus.TOO_MANY_REQUESTS.value()
            response.writer.write("Rate limit exceeded")
        }
    }

    private fun createBucket(): Bucket {
        return Bucket.builder()
            .addLimit(Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1))))
            .build()
    }
}
```

## 민감한 데이터 로깅 금지

```kotlin
// ❌ 위험
log.info("User login: email=${user.email}, password=${password}")

// ✅ 안전
log.info("User login: userId=${user.id}")

// ✅ 마스킹
log.info("User login: email=${user.email.take(3)}***")
```

## 보안 문제 대응 프로토콜

보안 문제 발견 시:
1. 즉시 중단
2. **security-reviewer** 에이전트 사용
3. CRITICAL 이슈 먼저 수정
4. 노출된 비밀 교체
5. 유사 이슈 전체 코드베이스 검토
