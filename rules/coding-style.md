# 코딩 스타일

## 불변성 (중요)

Kotlin data class와 val을 활용하여 불변성 유지:

```kotlin
// ❌ 잘못된 예: 가변 객체
data class User(var name: String, var email: String)

fun updateUser(user: User, name: String): User {
    user.name = name  // 변경!
    return user
}

// ✅ 올바른 예: 불변 객체
data class User(val name: String, val email: String)

fun updateUser(user: User, name: String): User {
    return user.copy(name = name)
}
```

## 파일 구성

많은 작은 파일 > 적은 큰 파일:
- 높은 응집도, 낮은 결합도
- 일반적으로 200-400줄, 최대 800줄
- 큰 컴포넌트에서 유틸리티 추출
- 타입별이 아닌 기능/도메인별 구성

## 에러 처리

항상 포괄적으로 에러 처리:

```kotlin
fun riskyOperation(): Result {
    return try {
        val result = performOperation()
        result
    } catch (e: BusinessException) {
        log.warn("비즈니스 예외: ${e.message}")
        throw e
    } catch (e: Exception) {
        log.error("예상치 못한 에러", e)
        throw InternalServerException("작업 실패")
    }
}
```

## 입력 검증

항상 사용자 입력 검증 (Bean Validation):

```kotlin
data class CreateUserRequest(
    @field:NotBlank(message = "이메일은 필수입니다")
    @field:Email(message = "올바른 이메일 형식이 아닙니다")
    val email: String,

    @field:NotBlank(message = "비밀번호는 필수입니다")
    @field:Size(min = 8, max = 100, message = "비밀번호는 8자 이상이어야 합니다")
    val password: String,

    @field:NotBlank(message = "이름은 필수입니다")
    @field:Size(max = 50, message = "이름은 50자를 초과할 수 없습니다")
    val name: String
)
```

## Kotlin 컨벤션

### Null Safety
```kotlin
// ✅ 안전한 호출
val length = user?.name?.length

// ✅ Elvis 연산자
val name = user?.name ?: "Unknown"

// ❌ 가급적 피할 것
val name = user!!.name
```

### 확장 함수 활용
```kotlin
// 유틸리티를 확장 함수로
fun String.toSlug(): String =
    this.lowercase()
        .replace(Regex("[^a-z0-9\\s-]"), "")
        .replace(Regex("\\s+"), "-")

// 사용
val slug = title.toSlug()
```

### Scope 함수
```kotlin
// let: null 체크 및 변환
user?.let { sendEmail(it.email) }

// apply: 객체 설정
val user = User().apply {
    name = "John"
    email = "john@example.com"
}

// also: 부수 효과 (로깅 등)
val saved = repository.save(user).also {
    log.info("User saved: ${it.id}")
}
```

## 코드 품질 체크리스트

작업 완료 전 확인:
- [ ] 코드가 읽기 쉽고 이름이 적절함
- [ ] 함수가 작음 (<50줄)
- [ ] 파일이 집중됨 (<800줄)
- [ ] 깊은 중첩 없음 (>4단계)
- [ ] 적절한 에러 처리
- [ ] println 문 없음
- [ ] 하드코딩된 값 없음
- [ ] 불변 패턴 사용
- [ ] Null safety 준수
