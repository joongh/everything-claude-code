---
name: coding-standards
description: Kotlin, Java, Spring Boot 개발을 위한 범용 코딩 표준, 모범 사례 및 패턴.
---

# 코딩 표준 및 모범 사례

모든 프로젝트에 적용되는 범용 코딩 표준.

## 코드 품질 원칙

### 1. 가독성 우선
- 코드는 작성되는 것보다 읽히는 횟수가 더 많음
- 명확한 변수와 함수 이름
- 주석보다 자기 문서화 코드 선호
- 일관된 포맷팅

### 2. KISS (Keep It Simple, Stupid)
- 작동하는 가장 단순한 해결책
- 과도한 엔지니어링 피하기
- 조기 최적화 금지
- 이해하기 쉬운 코드 > 영리한 코드

### 3. DRY (Don't Repeat Yourself)
- 공통 로직을 함수로 추출
- 재사용 가능한 컴포넌트 생성
- 모듈 간 유틸리티 공유
- 복사-붙여넣기 프로그래밍 피하기

### 4. YAGNI (You Aren't Gonna Need It)
- 필요하기 전에 기능 구축하지 않기
- 추측성 일반화 피하기
- 필요할 때만 복잡성 추가
- 단순하게 시작하고 필요 시 리팩토링

## Kotlin 표준

### 변수 네이밍

```kotlin
// ✅ 좋음: 설명적인 이름
val marketSearchQuery = "election"
val isUserAuthenticated = true
val totalRevenue = 1000L

// ❌ 나쁨: 불명확한 이름
val q = "election"
val flag = true
val x = 1000L
```

### 함수 네이밍

```kotlin
// ✅ 좋음: 동사-명사 패턴
suspend fun fetchMarketData(marketId: Long): Market { }
fun calculateSimilarity(a: List<Double>, b: List<Double>): Double { }
fun isValidEmail(email: String): Boolean { }

// ❌ 나쁨: 불명확하거나 명사만
suspend fun market(id: Long): Market { }
fun similarity(a: List<Double>, b: List<Double>): Double { }
fun email(e: String): Boolean { }
```

### Null Safety 패턴 (중요)

```kotlin
// ✅ 안전한 호출 연산자
val length = user?.name?.length

// ✅ Elvis 연산자
val name = user?.name ?: "Unknown"

// ✅ let과 함께 사용
user?.let { sendEmail(it.email) }

// ❌ Non-null assertion은 피할 것
val name = user!!.name  // NullPointerException 가능
```

### 에러 처리

```kotlin
// ✅ 좋음: 포괄적인 에러 처리
fun fetchData(url: String): Result<Data> {
    return try {
        val response = httpClient.get(url)

        if (!response.isSuccessful) {
            return Result.failure(HttpException(response.code))
        }

        Result.success(response.body)
    } catch (e: IOException) {
        log.error("네트워크 에러", e)
        Result.failure(NetworkException("데이터 조회 실패"))
    }
}

// ❌ 나쁨: 에러 처리 없음
fun fetchData(url: String): Data {
    return httpClient.get(url).body!!
}
```

### 컬렉션 처리

```kotlin
// ✅ 좋음: 가능할 때 병렬 실행
val (users, markets, stats) = runBlocking {
    listOf(
        async { fetchUsers() },
        async { fetchMarkets() },
        async { fetchStats() }
    ).awaitAll()
}

// ❌ 나쁨: 불필요하게 순차 실행
val users = fetchUsers()
val markets = fetchMarkets()
val stats = fetchStats()
```

### 타입 안전성

```kotlin
// ✅ 좋음: 명확한 타입
data class Market(
    val id: Long,
    val name: String,
    val status: MarketStatus,
    val createdAt: LocalDateTime
)

enum class MarketStatus {
    ACTIVE, RESOLVED, CLOSED
}

fun getMarket(id: Long): Market {
    // 구현
}

// ❌ 나쁨: Any 사용
fun getMarket(id: Any): Any {
    // 구현
}
```

## Spring Boot 모범 사례

### 계층 구조

```kotlin
// Controller: HTTP 요청/응답 처리
@RestController
@RequestMapping("/api/v1/users")
class UserController(private val userService: UserService) {
    @GetMapping("/{id}")
    fun findById(@PathVariable id: Long) = ResponseEntity.ok(userService.findById(id))
}

// Service: 비즈니스 로직
@Service
@Transactional(readOnly = true)
class UserService(private val userRepository: UserRepository) {
    fun findById(id: Long): UserResponse {
        val user = userRepository.findByIdOrNull(id)
            ?: throw UserNotFoundException(id)
        return UserResponse.from(user)
    }
}

// Repository: 데이터 접근
interface UserRepository : JpaRepository<User, Long> {
    fun findByEmail(email: String): User?
}
```

### DTO 패턴

```kotlin
// Request DTO
data class CreateUserRequest(
    @field:NotBlank
    @field:Email
    val email: String,

    @field:NotBlank
    @field:Size(min = 8)
    val password: String,

    @field:NotBlank
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

### 예외 처리

```kotlin
// 커스텀 예외 정의
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
    fun handleBusinessException(e: BusinessException) =
        ResponseEntity.status(e.errorCode.status)
            .body(ErrorResponse(e.errorCode.name, e.message))
}
```

## 파일 구성

### 프로젝트 구조

```
src/main/kotlin/com/example/
├── config/                 # 설정 클래스
├── controller/             # REST Controller
├── service/               # 비즈니스 로직
├── repository/            # 데이터 접근
├── domain/                # Entity, VO
├── dto/                   # Request/Response DTO
├── exception/             # 커스텀 예외
└── util/                  # 유틸리티
```

### 파일 네이밍

```
UserController.kt           # PascalCase
UserService.kt
UserRepository.kt
User.kt                     # Entity
CreateUserRequest.kt        # DTO
UserResponse.kt
UserNotFoundException.kt    # 예외
DateUtils.kt               # 유틸리티
```

## 주석 및 문서화

### 언제 주석을 작성할까

```kotlin
// ✅ 좋음: WHY 설명, WHAT 아님
// 지수 백오프로 장애 시 API 과부하 방지
val delay = minOf(1000L * 2.0.pow(retryCount).toLong(), 30000L)

// 대용량 배열에서 성능을 위해 의도적으로 mutation 사용
items.add(newItem)

// ❌ 나쁨: 명백한 것 설명
// 카운터를 1 증가
count++

// 이름을 사용자 이름으로 설정
name = user.name
```

### KDoc

```kotlin
/**
 * 시맨틱 유사도를 사용하여 마켓을 검색합니다.
 *
 * @param query 자연어 검색 쿼리
 * @param limit 최대 결과 수 (기본값: 10)
 * @return 유사도 점수 순으로 정렬된 마켓 목록
 * @throws OpenAIException OpenAI API 호출 실패 시
 *
 * @sample
 * ```kotlin
 * val results = searchMarkets("election", 5)
 * println(results[0].name) // "Trump vs Biden"
 * ```
 */
suspend fun searchMarkets(query: String, limit: Int = 10): List<Market>
```

## 코드 스멜 감지

다음 안티패턴 주의:

### 1. 긴 함수
```kotlin
// ❌ 나쁨: 함수가 50줄 초과
fun processMarketData() {
    // 100줄의 코드
}

// ✅ 좋음: 작은 함수로 분리
fun processMarketData() {
    val validated = validateData()
    val transformed = transformData(validated)
    return saveData(transformed)
}
```

### 2. 깊은 중첩
```kotlin
// ❌ 나쁨: 5단계 이상 중첩
if (user != null) {
    if (user.isAdmin) {
        if (market != null) {
            if (market.isActive) {
                // ...
            }
        }
    }
}

// ✅ 좋음: 조기 반환
if (user == null) return
if (!user.isAdmin) return
if (market == null) return
if (!market.isActive) return

// 로직 수행
```

### 3. 매직 넘버
```kotlin
// ❌ 나쁨: 설명 없는 숫자
if (retryCount > 3) { }
delay(500)

// ✅ 좋음: 명명된 상수
companion object {
    private const val MAX_RETRIES = 3
    private const val DEBOUNCE_DELAY_MS = 500L
}

if (retryCount > MAX_RETRIES) { }
delay(DEBOUNCE_DELAY_MS)
```

## 테스트 표준

### 테스트 구조 (Given-When-Then)

```kotlin
@Test
fun `유사도를 올바르게 계산한다`() {
    // Given (Arrange)
    val vector1 = listOf(1.0, 0.0, 0.0)
    val vector2 = listOf(0.0, 1.0, 0.0)

    // When (Act)
    val similarity = calculateCosineSimilarity(vector1, vector2)

    // Then (Assert)
    assertThat(similarity).isEqualTo(0.0)
}
```

### 테스트 네이밍

```kotlin
// ✅ 좋음: 설명적인 테스트 이름
@Test
fun `쿼리와 일치하는 마켓이 없으면 빈 배열을 반환한다`() { }

@Test
fun `API 키가 없으면 예외를 던진다`() { }

@Test
fun `Redis 사용 불가 시 부분 문자열 검색으로 폴백한다`() { }

// ❌ 나쁨: 모호한 테스트 이름
@Test
fun `works`() { }

@Test
fun `test search`() { }
```

**기억하세요**: 코드 품질은 협상 대상이 아닙니다. 명확하고 유지보수하기 쉬운 코드는 빠른 개발과 자신 있는 리팩토링을 가능하게 합니다.
