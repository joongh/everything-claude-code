# 일반적인 패턴

## API 응답 형식

```kotlin
data class ApiResponse<T>(
    val success: Boolean,
    val data: T? = null,
    val error: String? = null,
    val meta: Meta? = null
)

data class Meta(
    val total: Int,
    val page: Int,
    val limit: Int
)
```

## 커스텀 훅 패턴

```kotlin
// Debounce를 사용한 검색
fun <T> useDebounce(value: T, delayMs: Long): T {
    var debouncedValue by remember { mutableStateOf(value) }

    LaunchedEffect(value) {
        delay(delayMs)
        debouncedValue = value
    }

    return debouncedValue
}
```

## Repository 패턴

```kotlin
interface Repository<T, ID> {
    fun findAll(filters: Filters? = null): List<T>
    fun findById(id: ID): T?
    fun create(data: CreateDto): T
    fun update(id: ID, data: UpdateDto): T
    fun delete(id: ID)
}
```

## 스켈레톤 프로젝트

새 기능 구현 시:
1. 검증된 스켈레톤 프로젝트 검색
2. 병렬 에이전트로 옵션 평가:
   - 보안 평가
   - 확장성 분석
   - 관련성 점수
   - 구현 계획
3. 최적의 매치를 기반으로 클론
4. 검증된 구조 내에서 반복
