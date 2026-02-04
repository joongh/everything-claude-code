# 예제 프로젝트 CLAUDE.md

이것은 프로젝트 수준의 예제 CLAUDE.md 파일입니다. 프로젝트 루트에 배치하세요.

## 프로젝트 개요

Spring Boot + Kotlin 기반 백엔드 API 서버입니다.

### 기술 스택
- Kotlin 1.9
- Spring Boot 3.2
- Spring Data JPA
- QueryDSL / jOOQ
- PostgreSQL
- Redis

## 중요 규칙

### 1. 코드 구성

- 적은 큰 파일보다 많은 작은 파일
- 높은 응집도, 낮은 결합도
- 파일당 200-400줄 일반적, 최대 800줄
- 타입별이 아닌 기능/도메인별 구성

### 2. 코드 스타일

- 코드, 주석, 문서에 이모지 금지
- 항상 불변성 - data class와 val 사용
- 프로덕션 코드에 println 금지
- try/catch로 적절한 에러 처리
- Bean Validation으로 입력 검증

### 3. 테스트

- TDD: 테스트 먼저 작성
- 최소 80% 커버리지
- 유틸리티에 단위 테스트
- API에 통합 테스트
- 중요 로직에 슬라이스 테스트

### 4. 보안

- 하드코딩된 비밀 금지
- 민감한 데이터는 환경 변수로
- 모든 사용자 입력 검증
- 파라미터화된 쿼리만 사용
- Spring Security 적용

## 파일 구조

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
```

## 주요 패턴

### API 응답 형식

```kotlin
data class ApiResponse<T>(
    val success: Boolean,
    val data: T? = null,
    val error: String? = null
)
```

### 에러 처리

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {
    @ExceptionHandler(BusinessException::class)
    fun handleBusinessException(e: BusinessException): ResponseEntity<ApiResponse<Nothing>> {
        return ResponseEntity
            .status(e.errorCode.status)
            .body(ApiResponse(success = false, error = e.message))
    }
}
```

### Service 패턴

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

## 환경 변수

```bash
# 필수
SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/myapp
SPRING_DATASOURCE_USERNAME=
SPRING_DATASOURCE_PASSWORD=
JWT_SECRET=

# 선택
SPRING_PROFILES_ACTIVE=dev
REDIS_HOST=localhost
REDIS_PORT=6379
```

## 사용 가능한 명령어

- `/tdd` - 테스트 주도 개발 워크플로우
- `/build-fix` - Gradle 빌드 에러 수정
- `/test-coverage` - JaCoCo 테스트 커버리지
- `/spring-test` - Spring Boot 테스트 실행

## Git 워크플로우

- Conventional commits: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`
- main에 직접 커밋 금지
- PR에 리뷰 필요
- 병합 전 모든 테스트 통과 필요

## Gradle 명령어

```bash
# 빌드
./gradlew build

# 테스트
./gradlew test

# 테스트 + 커버리지
./gradlew test jacocoTestReport

# 실행
./gradlew bootRun

# 클린 빌드
./gradlew clean build
```
