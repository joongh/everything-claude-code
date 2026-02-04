---
description: JaCoCo로 테스트 커버리지를 분석하고 누락된 테스트를 생성합니다. 80%+ 커버리지 목표.
---

# 테스트 커버리지

테스트 커버리지를 분석하고 누락된 테스트를 생성합니다:

1. 커버리지와 함께 테스트 실행: `./gradlew test jacocoTestReport`

2. 커버리지 리포트 분석 (`build/reports/jacoco/test/html/index.html`)

3. 80% 커버리지 임계값 미달 파일 식별

4. 커버리지 부족 파일별:
   - 테스트되지 않은 코드 경로 분석
   - 함수에 대한 단위 테스트 생성
   - API에 대한 통합 테스트 생성
   - 중요 플로우에 대한 슬라이스 테스트 생성

5. 새 테스트 통과 확인

6. 전후 커버리지 지표 표시

7. 프로젝트가 전체 80%+ 커버리지 달성 확인

## 주요 명령어

```bash
# 테스트 + 커버리지 리포트
./gradlew test jacocoTestReport

# 커버리지 검증 (임계값 체크)
./gradlew jacocoTestCoverageVerification

# 특정 테스트만 실행
./gradlew test --tests "UserServiceTest"

# 테스트 + 커버리지 리포트 열기
./gradlew test jacocoTestReport && open build/reports/jacoco/test/html/index.html
```

## 집중 포인트

- Happy path 시나리오
- 에러 처리
- 엣지 케이스 (null, 빈 값, 경계값)
- 예외 조건

## 테스트 유형별 전략

### 단위 테스트 (Service, Util)
```kotlin
@ExtendWith(MockKExtension::class)
class UserServiceTest {
    @MockK
    private lateinit var userRepository: UserRepository

    @InjectMockKs
    private lateinit var userService: UserService

    @Test
    fun `정상 케이스 테스트`() { ... }

    @Test
    fun `예외 케이스 테스트`() { ... }

    @Test
    fun `경계값 테스트`() { ... }
}
```

### 슬라이스 테스트 (Controller)
```kotlin
@WebMvcTest(UserController::class)
class UserControllerTest {
    @Autowired
    private lateinit var mockMvc: MockMvc

    @MockkBean
    private lateinit var userService: UserService

    @Test
    fun `GET 요청 테스트`() { ... }

    @Test
    fun `POST 요청 유효성 검증 테스트`() { ... }
}
```

### Repository 테스트
```kotlin
@DataJpaTest
class UserRepositoryTest {
    @Autowired
    private lateinit var userRepository: UserRepository

    @Test
    fun `커스텀 쿼리 테스트`() { ... }
}
```

### 통합 테스트
```kotlin
@SpringBootTest
@Testcontainers
class UserIntegrationTest {
    companion object {
        @Container
        @JvmStatic
        val postgres = PostgreSQLContainer("postgres:15-alpine")
    }

    @Test
    fun `전체 플로우 테스트`() { ... }
}
```

## 커버리지 임계값 설정

```kotlin
// build.gradle.kts
tasks.jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = "0.80".toBigDecimal()
            }
        }
        rule {
            element = "CLASS"
            excludes = listOf(
                "*.config.*",
                "*.dto.*",
                "*Application*"
            )
            limit {
                minimum = "0.80".toBigDecimal()
            }
        }
    }
}
```

## 커버리지 제외 대상

일반적으로 제외해도 되는 항목:
- Configuration 클래스
- DTO/Request/Response 클래스
- Application main 클래스
- 상수 클래스

```kotlin
tasks.jacocoTestReport {
    classDirectories.setFrom(
        files(classDirectories.files.map {
            fileTree(it) {
                exclude(
                    "**/config/**",
                    "**/dto/**",
                    "**/*Application*"
                )
            }
        })
    )
}
```

## 리포트 해석

| 지표 | 설명 | 목표 |
|------|------|------|
| Line Coverage | 실행된 코드 라인 비율 | 80%+ |
| Branch Coverage | 실행된 분기 비율 | 80%+ |
| Method Coverage | 테스트된 메서드 비율 | 80%+ |
| Class Coverage | 테스트된 클래스 비율 | 80%+ |
