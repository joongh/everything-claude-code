---
description: Spring Boot 테스트를 실행합니다. 단위 테스트, 슬라이스 테스트, 통합 테스트를 지원합니다.
---

# Spring 테스트

Spring Boot 애플리케이션 테스트 실행 명령어입니다.

## 기본 테스트 명령어

```bash
# 전체 테스트 실행
./gradlew test

# 특정 테스트 클래스 실행
./gradlew test --tests "UserServiceTest"

# 특정 테스트 메서드 실행
./gradlew test --tests "UserServiceTest.사용자_생성_테스트"

# 패턴 매칭 테스트
./gradlew test --tests "*ServiceTest"
./gradlew test --tests "com.example.service.*"

# 여러 테스트 클래스
./gradlew test --tests "UserServiceTest" --tests "OrderServiceTest"
```

## 테스트 유형별 실행

```bash
# 단위 테스트만 (빠름)
./gradlew test --tests "*Test" -x integrationTest

# 통합 테스트만
./gradlew integrationTest

# 슬라이스 테스트 (Controller)
./gradlew test --tests "*ControllerTest"

# Repository 테스트
./gradlew test --tests "*RepositoryTest"
```

## 커버리지와 함께 실행

```bash
# 테스트 + JaCoCo 리포트
./gradlew test jacocoTestReport

# 커버리지 리포트 열기
open build/reports/jacoco/test/html/index.html

# 커버리지 임계값 검증
./gradlew jacocoTestCoverageVerification

# 전체 (테스트 + 커버리지 + 검증)
./gradlew test jacocoTestReport jacocoTestCoverageVerification
```

## 테스트 옵션

```bash
# 실패해도 계속 실행
./gradlew test --continue

# 실패한 테스트만 재실행
./gradlew test --rerun-tasks

# 연속 테스트 모드 (파일 변경 감지)
./gradlew test --continuous

# 병렬 실행
./gradlew test --parallel

# 디버그 모드
./gradlew test --debug-jvm
```

## 테스트 필터링

```bash
# 태그 기반 필터 (JUnit 5)
./gradlew test -Dgroups="unit"
./gradlew test -DexcludedGroups="slow"

# 프로파일 지정
./gradlew test -Dspring.profiles.active=test
```

## 테스트 리포트

```bash
# HTML 리포트 위치
# build/reports/tests/test/index.html

# 리포트 열기
open build/reports/tests/test/index.html

# JUnit XML 리포트 (CI용)
# build/test-results/test/*.xml
```

## TestContainers 테스트

```bash
# TestContainers 테스트 실행
./gradlew test --tests "*ContainerTest"

# Docker 상태 확인 필요
docker ps

# Ryuk 컨테이너 비활성화 (선택)
TESTCONTAINERS_RYUK_DISABLED=true ./gradlew test
```

## Spring Batch 테스트

```bash
# Batch Job 테스트
./gradlew test --tests "*BatchTest"
./gradlew test --tests "*JobTest"

# 특정 Step 테스트
./gradlew test --tests "*StepTest"
```

## 성능 테스트

```bash
# 느린 테스트 찾기
./gradlew test --info | grep "finished"

# 타임아웃 설정
# application-test.yml:
# spring.test.timeout: 30s
```

## 모범 사례

### 테스트 구조
```
src/test/kotlin/
├── unit/           # 단위 테스트 (빠름)
├── integration/    # 통합 테스트 (느림)
└── slice/          # 슬라이스 테스트 (중간)
```

### 테스트 네이밍
```kotlin
// 클래스명: 대상클래스Test
class UserServiceTest { ... }

// 메서드명: 한글 백틱 또는 영문 설명
@Test
fun `사용자 생성 시 비밀번호를 암호화한다`() { ... }

@Test
fun createUser_shouldEncryptPassword() { ... }
```

### 테스트 어노테이션
```kotlin
// 단위 테스트
@ExtendWith(MockKExtension::class)

// Controller 슬라이스
@WebMvcTest(UserController::class)

// Repository 슬라이스
@DataJpaTest

// 전체 컨텍스트
@SpringBootTest

// TestContainers
@Testcontainers
```

## 문제 해결

```bash
# 테스트 캐시 문제
./gradlew test --rerun-tasks

# 컨텍스트 캐시 문제
./gradlew test --no-build-cache

# 메모리 부족
# gradle.properties에 추가:
# org.gradle.jvmargs=-Xmx4g

# 포트 충돌 (랜덤 포트 사용)
# @SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
```

## CI/CD 통합

```bash
# CI 환경용 명령어
./gradlew test jacocoTestReport --no-daemon --stacktrace

# 결과 파일 위치
# - JUnit XML: build/test-results/test/*.xml
# - JaCoCo XML: build/reports/jacoco/test/jacocoTestReport.xml
# - HTML 리포트: build/reports/tests/test/index.html
```
