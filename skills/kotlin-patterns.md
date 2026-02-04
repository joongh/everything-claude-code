# Kotlin 패턴

Kotlin 언어의 핵심 기능과 관용적인 코드 작성 패턴을 정의합니다.

## Null Safety

### 안전한 호출 연산자
```kotlin
// Safe call operator (?.)
val length = user?.name?.length

// Elvis operator (?:)
val name = user?.name ?: "Unknown"

// Safe call with let
user?.let {
    sendEmail(it.email)
}

// Not-null assertion (!!) - 가급적 피할 것
val name = user!!.name  // NullPointerException 가능
```

### Nullable 타입 처리
```kotlin
// 권장: 명시적 null 체크
fun processUser(user: User?): String {
    if (user == null) {
        return "No user"
    }
    // 이 시점에서 user는 non-null로 스마트 캐스트됨
    return user.name
}

// 권장: require/check 사용
fun processUser(userId: Long?): User {
    requireNotNull(userId) { "User ID must not be null" }
    return userRepository.findById(userId)
        ?: throw UserNotFoundException(userId)
}
```

## Data Class

### 기본 사용법
```kotlin
data class User(
    val id: Long,
    val email: String,
    val name: String,
    val status: UserStatus = UserStatus.ACTIVE,
    val createdAt: LocalDateTime = LocalDateTime.now()
)

// copy() 활용
val updatedUser = user.copy(name = "New Name")

// 구조 분해
val (id, email, name) = user
```

### JPA Entity와 함께 사용
```kotlin
// 주의: JPA Entity는 data class 대신 일반 class 사용
@Entity
@Table(name = "users")
class User(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null,

    @Column(nullable = false, unique = true)
    var email: String,

    @Column(nullable = false)
    var name: String,

    @Enumerated(EnumType.STRING)
    var status: UserStatus = UserStatus.ACTIVE,

    @Column(updatable = false)
    val createdAt: LocalDateTime = LocalDateTime.now()
) {
    fun update(name: String, email: String) {
        this.name = name
        this.email = email
    }

    // equals/hashCode는 id 기반으로 구현
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is User) return false
        return id != null && id == other.id
    }

    override fun hashCode(): Int = javaClass.hashCode()
}
```

## 확장 함수 (Extension Functions)

### 유틸리티 확장
```kotlin
// String 확장
fun String.toSlug(): String =
    this.lowercase()
        .replace(Regex("[^a-z0-9\\s-]"), "")
        .replace(Regex("\\s+"), "-")

// Collection 확장
fun <T> List<T>.secondOrNull(): T? =
    if (size >= 2) this[1] else null

// 사용
val slug = "Hello World!".toSlug()  // "hello-world"
val second = listOf(1, 2, 3).secondOrNull()  // 2
```

### 도메인 특화 확장
```kotlin
// Entity 확장
fun User.toResponse() = UserResponse(
    id = this.id!!,
    email = this.email,
    name = this.name,
    createdAt = this.createdAt
)

// Repository 확장
fun <T, ID> JpaRepository<T, ID>.findByIdOrThrow(id: ID, message: () -> String): T =
    findByIdOrNull(id) ?: throw EntityNotFoundException(message())

// 사용
val user = userRepository.findByIdOrThrow(id) { "User not found: $id" }
```

## Scope Functions

### let, run, with, apply, also 사용 가이드
```kotlin
// let: null 체크, 변환
val userDto = user?.let { UserDto.from(it) }

// run: 객체 설정 후 결과 반환
val result = user.run {
    validateEmail(email)
    sendWelcomeEmail(email)
    "Welcome email sent to $email"
}

// with: 객체의 여러 메서드 호출
val summary = with(user) {
    "$name ($email) - Status: $status"
}

// apply: 객체 설정 (빌더 패턴 대체)
val user = User().apply {
    email = "test@example.com"
    name = "Test User"
}

// also: 부수 효과 (로깅 등)
val user = userRepository.save(newUser).also {
    log.info("User created: ${it.id}")
}
```

## 컬렉션 처리

### 표준 라이브러리 활용
```kotlin
val users = listOf(user1, user2, user3)

// filter, map
val activeEmails = users
    .filter { it.status == UserStatus.ACTIVE }
    .map { it.email }

// groupBy
val usersByStatus = users.groupBy { it.status }

// associate
val usersById = users.associateBy { it.id }

// partition
val (active, inactive) = users.partition { it.status == UserStatus.ACTIVE }

// first, find
val admin = users.firstOrNull { it.role == Role.ADMIN }

// any, all, none
val hasAdmin = users.any { it.role == Role.ADMIN }
val allActive = users.all { it.status == UserStatus.ACTIVE }
```

### Sequence 활용 (대량 데이터)
```kotlin
// 큰 컬렉션에서는 Sequence 사용
val result = users.asSequence()
    .filter { it.status == UserStatus.ACTIVE }
    .map { it.email }
    .take(100)
    .toList()
```

## Sealed Class

### 상태 모델링
```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Throwable) : Result<Nothing>()
    data object Loading : Result<Nothing>()
}

// when 표현식과 함께 사용
fun <T> handleResult(result: Result<T>) = when (result) {
    is Result.Success -> println("Data: ${result.data}")
    is Result.Error -> println("Error: ${result.exception.message}")
    is Result.Loading -> println("Loading...")
    // sealed class이므로 else 불필요
}
```

### API 응답 모델링
```kotlin
sealed class ApiResponse<out T> {
    data class Success<T>(
        val data: T,
        val message: String? = null
    ) : ApiResponse<T>()

    data class Failure(
        val code: String,
        val message: String,
        val details: Map<String, String>? = null
    ) : ApiResponse<Nothing>()
}
```

## 코루틴 (Coroutines)

### 기본 사용법
```kotlin
@Service
class UserService(
    private val userRepository: UserRepository,
    private val externalApiClient: ExternalApiClient
) {
    suspend fun enrichUser(userId: Long): EnrichedUser {
        val user = userRepository.findById(userId)
            ?: throw UserNotFoundException(userId)

        val additionalInfo = externalApiClient.fetchUserInfo(user.externalId)

        return EnrichedUser(user, additionalInfo)
    }
}
```

### 병렬 실행
```kotlin
suspend fun fetchAllData(userId: Long): UserData = coroutineScope {
    val userDeferred = async { userRepository.findById(userId) }
    val ordersDeferred = async { orderRepository.findByUserId(userId) }
    val preferencesDeferred = async { preferenceRepository.findByUserId(userId) }

    UserData(
        user = userDeferred.await() ?: throw UserNotFoundException(userId),
        orders = ordersDeferred.await(),
        preferences = preferencesDeferred.await()
    )
}
```

### Flow 사용
```kotlin
@Service
class EventService(
    private val eventRepository: EventRepository
) {
    fun streamEvents(userId: Long): Flow<Event> = flow {
        eventRepository.findByUserIdStream(userId).collect { event ->
            emit(event)
        }
    }
}
```

## 인라인 함수와 Reified 타입

```kotlin
// reified 타입 파라미터
inline fun <reified T> parseJson(json: String): T {
    return objectMapper.readValue(json, T::class.java)
}

// 사용
val user: User = parseJson(jsonString)

// 고차 함수 인라인
inline fun <T> measureTime(block: () -> T): Pair<T, Long> {
    val start = System.currentTimeMillis()
    val result = block()
    val elapsed = System.currentTimeMillis() - start
    return result to elapsed
}
```

## DSL 구축

```kotlin
// 타입 안전 빌더
class HtmlBuilder {
    private val elements = mutableListOf<String>()

    fun div(block: DivBuilder.() -> Unit) {
        elements.add(DivBuilder().apply(block).build())
    }

    fun build() = elements.joinToString("\n")
}

class DivBuilder {
    var className: String = ""
    private var content: String = ""

    fun text(value: String) {
        content = value
    }

    fun build() = "<div class=\"$className\">$content</div>"
}

fun html(block: HtmlBuilder.() -> Unit): String =
    HtmlBuilder().apply(block).build()

// 사용
val result = html {
    div {
        className = "container"
        text("Hello, World!")
    }
}
```

## 위임 (Delegation)

```kotlin
// 클래스 위임
interface Logger {
    fun log(message: String)
}

class ConsoleLogger : Logger {
    override fun log(message: String) = println(message)
}

class UserService(logger: Logger) : Logger by logger {
    fun createUser(name: String) {
        log("Creating user: $name")
        // ...
    }
}

// 프로퍼티 위임
class User {
    var name: String by Delegates.observable("") { _, old, new ->
        println("Name changed from $old to $new")
    }

    val lazyValue: String by lazy {
        println("Computed once")
        "Hello"
    }
}
```
