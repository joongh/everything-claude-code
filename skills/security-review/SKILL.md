---
name: security-review
description: 인증 추가, 사용자 입력 처리, 비밀 작업, API 엔드포인트 생성, 결제/민감한 기능 구현 시 이 스킬을 사용하세요. 포괄적인 보안 체크리스트와 패턴을 제공합니다.
---

# 보안 리뷰 스킬

이 스킬은 모든 코드가 보안 모범 사례를 따르고 잠재적 취약점을 식별하도록 보장합니다.

## 활성화 시점

- 인증 또는 인가 구현
- 사용자 입력 또는 파일 업로드 처리
- 새 API 엔드포인트 생성
- 비밀 또는 자격증명 작업
- 결제 기능 구현
- 민감한 데이터 저장 또는 전송
- 서드파티 API 통합

## 보안 체크리스트

### 1. 비밀 관리

#### ❌ 절대 하지 말 것
```kotlin
val apiKey = "sk-proj-xxxxx"  // 하드코딩된 비밀
val dbPassword = "password123" // 소스 코드에
```

#### ✅ 항상 할 것
```kotlin
val apiKey = System.getenv("OPENAI_API_KEY")
val dbUrl = System.getenv("DATABASE_URL")

// 비밀 존재 확인
if (apiKey.isNullOrBlank()) {
    throw IllegalStateException("OPENAI_API_KEY가 설정되지 않았습니다")
}
```

#### 확인 단계
- [ ] 하드코딩된 API 키, 토큰, 비밀번호 없음
- [ ] 모든 비밀이 환경 변수에
- [ ] application-local.yml이 .gitignore에
- [ ] git 히스토리에 비밀 없음
- [ ] 프로덕션 비밀은 호스팅 플랫폼에

### 2. 입력 검증

#### 항상 사용자 입력 검증
```kotlin
data class CreateUserRequest(
    @field:NotBlank(message = "이메일은 필수입니다")
    @field:Email(message = "올바른 이메일 형식이 아닙니다")
    val email: String,

    @field:NotBlank(message = "비밀번호는 필수입니다")
    @field:Size(min = 8, message = "비밀번호는 8자 이상이어야 합니다")
    val password: String
)

// 컨트롤러에서 @Valid 사용
@PostMapping
fun create(@Valid @RequestBody request: CreateUserRequest) {
    // ...
}
```

#### 확인 단계
- [ ] 모든 사용자 입력이 스키마로 검증됨
- [ ] 파일 업로드 제한됨 (크기, 타입, 확장자)
- [ ] 쿼리에 사용자 입력 직접 사용하지 않음
- [ ] 화이트리스트 검증 (블랙리스트 아님)

### 3. SQL 인젝션 방지

#### ❌ 절대 SQL 연결하지 말 것
```kotlin
// 위험 - SQL 인젝션 취약점
val query = "SELECT * FROM users WHERE email = '$userEmail'"
```

#### ✅ 항상 파라미터화된 쿼리 사용
```kotlin
// 안전 - jOOQ 사용
dsl.selectFrom(USERS)
    .where(USERS.EMAIL.eq(userEmail))
    .fetchOne()

// 안전 - JPA
@Query("SELECT u FROM User u WHERE u.email = :email")
fun findByEmail(@Param("email") email: String): User?
```

### 4. 인증 및 인가

#### JWT 토큰 처리
```kotlin
// ❌ 잘못됨: localStorage (XSS에 취약)
// ✅ 올바름: httpOnly 쿠키
response.addCookie(Cookie("token", token).apply {
    isHttpOnly = true
    secure = true
    path = "/"
    maxAge = 3600
})
```

#### 인가 검사
```kotlin
@GetMapping("/api/user/{id}")
@PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.id")
fun getUser(@PathVariable id: Long): UserResponse {
    // ...
}
```

### 5. 속도 제한

```kotlin
@Configuration
class RateLimitConfig {
    @Bean
    fun rateLimiter(): RateLimiter = RateLimiter.builder()
        .limit(100)
        .window(Duration.ofMinutes(1))
        .build()
}

@PostMapping("/api/search")
@RateLimiter(name = "search", fallbackMethod = "searchFallback")
fun search(@RequestBody request: SearchRequest): SearchResponse {
    // ...
}
```

### 6. 민감한 데이터 노출

#### 로깅
```kotlin
// ❌ 잘못됨: 민감한 데이터 로깅
logger.info("User login: email=$email, password=$password")

// ✅ 올바름: 로그 정제
logger.info("User login: userId=$userId")
```

#### 에러 메시지
```kotlin
// ❌ 잘못됨: 내부 세부사항 노출
catch (e: Exception) {
    throw ResponseStatusException(HttpStatus.INTERNAL_SERVER_ERROR, e.message)
}

// ✅ 올바름: 일반 에러 메시지
catch (e: Exception) {
    logger.error("Internal error", e)
    throw ResponseStatusException(HttpStatus.INTERNAL_SERVER_ERROR, "오류가 발생했습니다")
}
```

## 배포 전 보안 체크리스트

모든 프로덕션 배포 전:

- [ ] **비밀**: 하드코딩된 비밀 없음, 모두 환경 변수에
- [ ] **입력 검증**: 모든 사용자 입력 검증됨
- [ ] **SQL 인젝션**: 모든 쿼리 파라미터화됨
- [ ] **인증**: 올바른 토큰 처리
- [ ] **인가**: 역할 검사 적용됨
- [ ] **속도 제한**: 모든 엔드포인트에 활성화됨
- [ ] **HTTPS**: 프로덕션에서 강제됨
- [ ] **에러 처리**: 로그에 민감한 데이터 없음
- [ ] **의존성**: 최신, 취약점 없음

## 리소스

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Spring Security 문서](https://spring.io/projects/spring-security)
- [Kotlin 보안 가이드](https://kotlinlang.org/docs/security.html)

---

**기억**: 보안은 선택 사항이 아닙니다. 하나의 취약점이 전체 플랫폼을 위험에 빠뜨릴 수 있습니다. 확신이 없으면 주의 쪽으로 기울이세요.
