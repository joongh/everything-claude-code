# Spring Security 패턴

Spring Security를 사용한 인증/인가 구현 패턴을 정의합니다.

## 기본 설정

### build.gradle.kts
```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("io.jsonwebtoken:jjwt-api:0.12.3")
    runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.3")
    runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.3")
    testImplementation("org.springframework.security:spring-security-test")
}
```

### SecurityConfig
```kotlin
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
class SecurityConfig(
    private val jwtAuthenticationFilter: JwtAuthenticationFilter,
    private val authenticationEntryPoint: CustomAuthenticationEntryPoint
) {
    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        return http
            .csrf { it.disable() }
            .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
            .exceptionHandling { it.authenticationEntryPoint(authenticationEntryPoint) }
            .authorizeHttpRequests { auth ->
                auth
                    .requestMatchers("/api/auth/**").permitAll()
                    .requestMatchers("/api/public/**").permitAll()
                    .requestMatchers("/actuator/health").permitAll()
                    .requestMatchers("/api/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated()
            }
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter::class.java)
            .build()
    }

    @Bean
    fun passwordEncoder(): PasswordEncoder = BCryptPasswordEncoder()

    @Bean
    fun authenticationManager(config: AuthenticationConfiguration): AuthenticationManager {
        return config.authenticationManager
    }
}
```

## JWT 인증

### JWT Provider
```kotlin
@Component
class JwtTokenProvider(
    @Value("\${jwt.secret}") private val secret: String,
    @Value("\${jwt.expiration-ms}") private val expirationMs: Long
) {
    private val key: SecretKey by lazy {
        Keys.hmacShaKeyFor(secret.toByteArray())
    }

    fun generateToken(authentication: Authentication): String {
        val userDetails = authentication.principal as UserDetails
        val now = Date()
        val expiryDate = Date(now.time + expirationMs)

        return Jwts.builder()
            .subject(userDetails.username)
            .claim("roles", userDetails.authorities.map { it.authority })
            .issuedAt(now)
            .expiration(expiryDate)
            .signWith(key)
            .compact()
    }

    fun getUsernameFromToken(token: String): String {
        return Jwts.parser()
            .verifyWith(key)
            .build()
            .parseSignedClaims(token)
            .payload
            .subject
    }

    fun validateToken(token: String): Boolean {
        return try {
            Jwts.parser()
                .verifyWith(key)
                .build()
                .parseSignedClaims(token)
            true
        } catch (e: JwtException) {
            false
        } catch (e: IllegalArgumentException) {
            false
        }
    }
}
```

### JWT Filter
```kotlin
@Component
class JwtAuthenticationFilter(
    private val jwtTokenProvider: JwtTokenProvider,
    private val userDetailsService: UserDetailsService
) : OncePerRequestFilter() {

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        try {
            val token = extractToken(request)

            if (token != null && jwtTokenProvider.validateToken(token)) {
                val username = jwtTokenProvider.getUsernameFromToken(token)
                val userDetails = userDetailsService.loadUserByUsername(username)

                val authentication = UsernamePasswordAuthenticationToken(
                    userDetails,
                    null,
                    userDetails.authorities
                )
                authentication.details = WebAuthenticationDetailsSource().buildDetails(request)

                SecurityContextHolder.getContext().authentication = authentication
            }
        } catch (e: Exception) {
            logger.error("인증 처리 중 오류 발생", e)
        }

        filterChain.doFilter(request, response)
    }

    private fun extractToken(request: HttpServletRequest): String? {
        val bearerToken = request.getHeader("Authorization")
        return if (bearerToken?.startsWith("Bearer ") == true) {
            bearerToken.substring(7)
        } else {
            null
        }
    }
}
```

### Authentication EntryPoint
```kotlin
@Component
class CustomAuthenticationEntryPoint : AuthenticationEntryPoint {

    private val objectMapper = ObjectMapper()

    override fun commence(
        request: HttpServletRequest,
        response: HttpServletResponse,
        authException: AuthenticationException
    ) {
        response.status = HttpStatus.UNAUTHORIZED.value()
        response.contentType = MediaType.APPLICATION_JSON_VALUE

        val errorResponse = mapOf(
            "code" to "UNAUTHORIZED",
            "message" to "인증이 필요합니다"
        )

        objectMapper.writeValue(response.outputStream, errorResponse)
    }
}
```

## UserDetailsService

```kotlin
@Service
class CustomUserDetailsService(
    private val userRepository: UserRepository
) : UserDetailsService {

    override fun loadUserByUsername(username: String): UserDetails {
        val user = userRepository.findByEmail(username)
            ?: throw UsernameNotFoundException("사용자를 찾을 수 없습니다: $username")

        return CustomUserDetails(user)
    }
}

class CustomUserDetails(
    private val user: User
) : UserDetails {

    override fun getAuthorities(): Collection<GrantedAuthority> {
        return user.roles.map { SimpleGrantedAuthority("ROLE_${it.name}") }
    }

    override fun getPassword(): String = user.password
    override fun getUsername(): String = user.email
    override fun isAccountNonExpired(): Boolean = true
    override fun isAccountNonLocked(): Boolean = !user.isLocked
    override fun isCredentialsNonExpired(): Boolean = true
    override fun isEnabled(): Boolean = user.isActive

    fun getUserId(): Long = user.id!!
    fun getUser(): User = user
}
```

## 인증 Controller

```kotlin
@RestController
@RequestMapping("/api/auth")
class AuthController(
    private val authenticationManager: AuthenticationManager,
    private val jwtTokenProvider: JwtTokenProvider,
    private val userService: UserService
) {
    @PostMapping("/login")
    fun login(@Valid @RequestBody request: LoginRequest): ResponseEntity<TokenResponse> {
        val authentication = authenticationManager.authenticate(
            UsernamePasswordAuthenticationToken(request.email, request.password)
        )

        val token = jwtTokenProvider.generateToken(authentication)
        return ResponseEntity.ok(TokenResponse(token))
    }

    @PostMapping("/signup")
    fun signup(@Valid @RequestBody request: SignupRequest): ResponseEntity<UserResponse> {
        val user = userService.create(request)
        return ResponseEntity.status(HttpStatus.CREATED).body(user)
    }

    @PostMapping("/refresh")
    fun refresh(@RequestHeader("Authorization") bearerToken: String): ResponseEntity<TokenResponse> {
        val token = bearerToken.removePrefix("Bearer ")
        val username = jwtTokenProvider.getUsernameFromToken(token)
        val userDetails = userService.loadUserByUsername(username)

        val authentication = UsernamePasswordAuthenticationToken(
            userDetails, null, userDetails.authorities
        )
        val newToken = jwtTokenProvider.generateToken(authentication)

        return ResponseEntity.ok(TokenResponse(newToken))
    }
}

data class LoginRequest(
    @field:NotBlank val email: String,
    @field:NotBlank val password: String
)

data class TokenResponse(
    val accessToken: String,
    val tokenType: String = "Bearer"
)
```

## Method Security

### @PreAuthorize / @PostAuthorize
```kotlin
@Service
class OrderService(
    private val orderRepository: OrderRepository
) {
    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.userId")
    fun findByUserId(userId: Long): List<Order> {
        return orderRepository.findByUserId(userId)
    }

    @PreAuthorize("hasRole('ADMIN')")
    fun deleteAll() {
        orderRepository.deleteAll()
    }

    @PostAuthorize("returnObject.userId == authentication.principal.userId or hasRole('ADMIN')")
    fun findById(id: Long): Order {
        return orderRepository.findById(id)
            .orElseThrow { OrderNotFoundException(id) }
    }

    @PreAuthorize("@orderSecurityService.canAccess(#orderId)")
    fun updateStatus(orderId: Long, status: OrderStatus): Order {
        val order = orderRepository.findById(orderId)
            .orElseThrow { OrderNotFoundException(orderId) }
        order.status = status
        return orderRepository.save(order)
    }
}
```

### 커스텀 Security 서비스
```kotlin
@Service
class OrderSecurityService(
    private val orderRepository: OrderRepository
) {
    fun canAccess(orderId: Long): Boolean {
        val authentication = SecurityContextHolder.getContext().authentication
        val userDetails = authentication.principal as? CustomUserDetails
            ?: return false

        val order = orderRepository.findById(orderId).orElse(null)
            ?: return false

        return order.userId == userDetails.getUserId() ||
               userDetails.authorities.any { it.authority == "ROLE_ADMIN" }
    }
}
```

## 현재 사용자 정보 접근

### @AuthenticationPrincipal
```kotlin
@RestController
@RequestMapping("/api/users")
class UserController(
    private val userService: UserService
) {
    @GetMapping("/me")
    fun getCurrentUser(
        @AuthenticationPrincipal userDetails: CustomUserDetails
    ): ResponseEntity<UserResponse> {
        return ResponseEntity.ok(UserResponse.from(userDetails.getUser()))
    }

    @PutMapping("/me")
    fun updateCurrentUser(
        @AuthenticationPrincipal userDetails: CustomUserDetails,
        @Valid @RequestBody request: UpdateUserRequest
    ): ResponseEntity<UserResponse> {
        val updated = userService.update(userDetails.getUserId(), request)
        return ResponseEntity.ok(updated)
    }
}
```

### 커스텀 Argument Resolver
```kotlin
@Target(AnnotationTarget.VALUE_PARAMETER)
@Retention(AnnotationRetention.RUNTIME)
annotation class CurrentUser

class CurrentUserArgumentResolver : HandlerMethodArgumentResolver {

    override fun supportsParameter(parameter: MethodParameter): Boolean {
        return parameter.hasParameterAnnotation(CurrentUser::class.java)
    }

    override fun resolveArgument(
        parameter: MethodParameter,
        mavContainer: ModelAndViewContainer?,
        webRequest: NativeWebRequest,
        binderFactory: WebDataBinderFactory?
    ): Any? {
        val authentication = SecurityContextHolder.getContext().authentication
        return (authentication.principal as? CustomUserDetails)?.getUser()
    }
}

// 사용
@GetMapping("/me")
fun getCurrentUser(@CurrentUser user: User): ResponseEntity<UserResponse> {
    return ResponseEntity.ok(UserResponse.from(user))
}
```

## CORS 설정

```kotlin
@Configuration
class CorsConfig {
    @Bean
    fun corsConfigurationSource(): CorsConfigurationSource {
        val configuration = CorsConfiguration().apply {
            allowedOrigins = listOf("http://localhost:3000", "https://myapp.com")
            allowedMethods = listOf("GET", "POST", "PUT", "DELETE", "OPTIONS")
            allowedHeaders = listOf("*")
            allowCredentials = true
            maxAge = 3600
        }

        return UrlBasedCorsConfigurationSource().apply {
            registerCorsConfiguration("/api/**", configuration)
        }
    }
}
```

## 테스트

### @WithMockUser
```kotlin
@WebMvcTest(UserController::class)
class UserControllerTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @Test
    @WithMockUser(username = "user@test.com", roles = ["USER"])
    fun `should return current user`() {
        mockMvc.perform(get("/api/users/me"))
            .andExpect(status().isOk)
    }

    @Test
    @WithMockUser(roles = ["ADMIN"])
    fun `should access admin endpoint with admin role`() {
        mockMvc.perform(get("/api/admin/users"))
            .andExpect(status().isOk)
    }

    @Test
    @WithMockUser(roles = ["USER"])
    fun `should deny admin endpoint for regular user`() {
        mockMvc.perform(get("/api/admin/users"))
            .andExpect(status().isForbidden)
    }
}
```

### 커스텀 @WithMockUser
```kotlin
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
@WithSecurityContext(factory = WithMockCustomUserSecurityContextFactory::class)
annotation class WithMockCustomUser(
    val userId: Long = 1L,
    val email: String = "test@example.com",
    val roles: Array<String> = ["USER"]
)

class WithMockCustomUserSecurityContextFactory : WithSecurityContextFactory<WithMockCustomUser> {

    override fun createSecurityContext(annotation: WithMockCustomUser): SecurityContext {
        val user = User(
            id = annotation.userId,
            email = annotation.email,
            name = "Test User",
            password = "encoded",
            roles = annotation.roles.map { Role.valueOf(it) }.toSet()
        )

        val userDetails = CustomUserDetails(user)
        val authentication = UsernamePasswordAuthenticationToken(
            userDetails, null, userDetails.authorities
        )

        return SecurityContextHolder.createEmptyContext().apply {
            this.authentication = authentication
        }
    }
}

// 사용
@Test
@WithMockCustomUser(userId = 123, email = "custom@test.com", roles = ["ADMIN"])
fun `should access with custom user`() {
    // ...
}
```

## OAuth2 로그인 (선택)

```kotlin
@Configuration
class OAuth2Config {
    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        return http
            .oauth2Login { oauth2 ->
                oauth2
                    .userInfoEndpoint { it.userService(customOAuth2UserService()) }
                    .successHandler(oAuth2SuccessHandler())
            }
            .build()
    }

    @Bean
    fun customOAuth2UserService() = DefaultOAuth2UserService()

    @Bean
    fun oAuth2SuccessHandler() = SimpleUrlAuthenticationSuccessHandler("/")
}
```
