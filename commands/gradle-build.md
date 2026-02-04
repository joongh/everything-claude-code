---
description: Gradle 빌드 명령어를 실행합니다. 컴파일, 테스트, 패키징을 수행합니다.
---

# Gradle 빌드

Gradle 프로젝트 빌드 및 관리 명령어입니다.

## 기본 빌드 명령어

```bash
# 전체 빌드 (컴파일 + 테스트 + 패키징)
./gradlew build

# 클린 빌드
./gradlew clean build

# 컴파일만
./gradlew compileKotlin

# 테스트 컴파일
./gradlew compileTestKotlin

# 테스트 실행
./gradlew test

# 패키징 (JAR 생성)
./gradlew bootJar
```

## 빌드 옵션

```bash
# 스택트레이스 표시
./gradlew build --stacktrace

# 상세 로그
./gradlew build --info

# 디버그 로그
./gradlew build --debug

# 병렬 빌드
./gradlew build --parallel

# 캐시 없이 빌드
./gradlew build --no-build-cache

# 의존성 새로고침
./gradlew build --refresh-dependencies

# 빌드 스캔 (웹 리포트)
./gradlew build --scan
```

## 의존성 관리

```bash
# 전체 의존성 트리
./gradlew dependencies

# 특정 설정의 의존성
./gradlew dependencies --configuration runtimeClasspath

# 의존성 충돌 분석
./gradlew dependencyInsight --dependency 라이브러리명

# 의존성 업데이트 확인
./gradlew dependencyUpdates
```

## 멀티 모듈 빌드

```bash
# 특정 모듈 빌드
./gradlew :module-name:build

# 특정 모듈 테스트
./gradlew :module-name:test

# 모든 모듈 클린
./gradlew clean

# 프로젝트 구조 확인
./gradlew projects
```

## Spring Boot 전용

```bash
# 애플리케이션 실행
./gradlew bootRun

# 프로필 지정 실행
./gradlew bootRun --args='--spring.profiles.active=dev'

# 실행 가능한 JAR 생성
./gradlew bootJar

# Docker 이미지 빌드 (Spring Boot 2.3+)
./gradlew bootBuildImage
```

## 코드 생성

```bash
# jOOQ 코드 생성
./gradlew generateJooq

# QueryDSL Q클래스 생성
./gradlew compileKotlin

# Flyway 마이그레이션
./gradlew flywayMigrate
./gradlew flywayInfo
./gradlew flywayValidate
```

## 테스트 명령어

```bash
# 전체 테스트
./gradlew test

# 특정 테스트 클래스
./gradlew test --tests "UserServiceTest"

# 특정 테스트 메서드
./gradlew test --tests "UserServiceTest.사용자_생성_테스트"

# 테스트 + 커버리지
./gradlew test jacocoTestReport

# 커버리지 검증
./gradlew jacocoTestCoverageVerification

# 연속 테스트 (파일 변경 감지)
./gradlew test --continuous
```

## 캐시 및 정리

```bash
# 빌드 캐시 정리
./gradlew clean

# Gradle 데몬 중지
./gradlew --stop

# 데몬 상태 확인
./gradlew --status

# 캐시 완전 삭제
rm -rf ~/.gradle/caches/
rm -rf .gradle/
```

## 유용한 태스크

```bash
# 사용 가능한 모든 태스크
./gradlew tasks

# 특정 그룹 태스크
./gradlew tasks --group=build

# 속성 확인
./gradlew properties

# 빌드 환경 정보
./gradlew buildEnvironment
```

## 문제 해결

```bash
# 의존성 해결 문제
./gradlew build --refresh-dependencies --stacktrace

# 캐시 문제
./gradlew build --no-build-cache --no-daemon

# 메모리 부족
# gradle.properties에 추가:
# org.gradle.jvmargs=-Xmx4g

# 느린 빌드
./gradlew build --parallel --build-cache
```
