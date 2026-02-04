# Kotlin 개발 컨텍스트

Kotlin 언어로 개발할 때 자동으로 적용되는 컨텍스트입니다.

## 언어 기본 설정

- Kotlin 1.9+ 버전 사용
- JVM 타겟: 21
- Kotlin DSL for Gradle (build.gradle.kts)

## Null Safety 규칙

```kotlin
// ✅ 권장: Safe call
val length = user?.name?.length

// ✅ 권장: Elvis operator
val name = user?.name ?: "Unknown"

// ✅ 권장: let으로 null 처리
user?.let { sendEmail(it.email) }

// ❌ 피할 것: Non-null assertion
val name = user!!.name
```

## Data Class 사용

```kotlin
// ✅ 불변 data class 권장
data class User(
    val id: Long,
    val email: String,
    val name: String
)

// ❌ var 사용 피할 것
data class User(
    var id: Long,
    var email: String
)
```

## 확장 함수 활용

```kotlin
// 유틸리티를 확장 함수로 작성
fun String.toSlug(): String =
    this.lowercase()
        .replace(Regex("[^a-z0-9\\s-]"), "")
        .replace(Regex("\\s+"), "-")

// Entity to DTO 변환
fun User.toResponse() = UserResponse(
    id = this.id!!,
    email = this.email,
    name = this.name
)
```

## Scope 함수 가이드

| 함수 | 수신 객체 | 반환값 | 용도 |
|------|----------|--------|------|
| let | it | Lambda 결과 | null 체크, 변환 |
| run | this | Lambda 결과 | 객체 설정 + 결과 |
| with | this | Lambda 결과 | 객체 메서드 여러 개 호출 |
| apply | this | 수신 객체 | 객체 설정 (빌더 대체) |
| also | it | 수신 객체 | 부수 효과 (로깅) |

## 컬렉션 처리

```kotlin
// filter, map
val activeEmails = users
    .filter { it.status == UserStatus.ACTIVE }
    .map { it.email }

// associate
val usersById = users.associateBy { it.id }

// groupBy
val usersByStatus = users.groupBy { it.status }

// 대량 데이터: Sequence 사용
val result = users.asSequence()
    .filter { it.status == UserStatus.ACTIVE }
    .map { it.email }
    .take(100)
    .toList()
```

## 코루틴 (필요 시)

```kotlin
// suspend 함수
suspend fun fetchUser(id: Long): User {
    return userRepository.findById(id)
}

// 병렬 실행
suspend fun fetchAllData(userId: Long): UserData = coroutineScope {
    val userDeferred = async { userRepository.findById(userId) }
    val ordersDeferred = async { orderRepository.findByUserId(userId) }

    UserData(
        user = userDeferred.await(),
        orders = ordersDeferred.await()
    )
}
```

## 테스트 작성

```kotlin
@Test
fun `사용자 생성 시 비밀번호를 암호화한다`() {
    // given
    val request = CreateUserRequest(...)
    every { passwordEncoder.encode(any()) } returns "encoded"

    // when
    val result = userService.create(request)

    // then
    assertThat(result.email).isEqualTo(request.email)
    verify { passwordEncoder.encode(request.password) }
}
```

## 빌드 설정

```kotlin
// build.gradle.kts
plugins {
    kotlin("jvm") version "1.9.22"
    kotlin("plugin.spring") version "1.9.22"
}

kotlin {
    compilerOptions {
        freeCompilerArgs.add("-Xjsr305=strict")
        jvmTarget.set(JvmTarget.JVM_21)
    }
}
```
