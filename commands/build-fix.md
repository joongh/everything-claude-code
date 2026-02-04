---
description: Gradle 빌드 에러를 점진적으로 수정합니다. 컴파일 에러, 의존성 충돌, 설정 문제를 해결합니다.
---

# 빌드 수정

Gradle/Kotlin 빌드 에러를 점진적으로 수정합니다:

1. 빌드 실행: `./gradlew build --stacktrace`

2. 에러 출력 파싱:
   - 파일별로 그룹화
   - 심각도별 정렬

3. 각 에러별:
   - 에러 컨텍스트 표시 (전후 5줄)
   - 문제 설명
   - 수정 제안
   - 수정 적용
   - 빌드 재실행
   - 에러 해결 확인

4. 중단 조건:
   - 수정이 새로운 에러를 유발할 경우
   - 동일 에러가 3회 시도 후에도 지속될 경우
   - 사용자가 일시 중지 요청 시

5. 요약 표시:
   - 수정된 에러
   - 남은 에러
   - 새로 발생한 에러

안전을 위해 한 번에 하나의 에러만 수정합니다!

## 주요 명령어

```bash
# Gradle 빌드
./gradlew build --stacktrace

# Kotlin 컴파일만
./gradlew compileKotlin --stacktrace

# 클린 빌드
./gradlew clean build --stacktrace

# 의존성 트리 확인
./gradlew dependencies

# 의존성 충돌 확인
./gradlew dependencyInsight --dependency 라이브러리명

# 캐시 없이 빌드
./gradlew build --no-build-cache
```

## 공통 에러 패턴

### Kotlin 컴파일 에러
- Null Safety 위반
- 타입 불일치
- 누락된 import
- 제네릭 타입 에러

### Gradle 설정 에러
- 의존성 충돌
- 플러그인 버전 문제
- 레포지토리 설정 오류

### Spring Boot 에러
- Bean 주입 실패
- 설정 속성 누락
- 프로파일 설정 오류
