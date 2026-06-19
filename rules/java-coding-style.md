# Java 코딩 스타일

## 상수 선언

반복 사용되는 문자열 리터럴, 숫자 등은 메서드 내 로컬 변수가 아닌 클래스 레벨 `private static final`로 선언:

```java
// ❌ 잘못된 예: 메서드 내 로컬 변수
public RepeatStatus execute() {
    String key = "IMAGE_PREVIOUS_STATUS";
    context.putString(key, status);
}

// ✅ 올바른 예: 클래스 레벨 상수
private static final String IMAGE_PREVIOUS_STATUS = "IMAGE_PREVIOUS_STATUS";

public RepeatStatus execute() {
    context.putString(IMAGE_PREVIOUS_STATUS, status);
}
```

## Null Safety

문자열 비교 시 리터럴을 앞에 두어 NPE 방지:

```java
// ❌ 위험: variable이 null이면 NPE
if (status.equals("active")) { ... }

// ✅ 안전: "active".equals(null)은 false 반환
if ("active".equals(status)) { ... }
```

## 불변성

가능하면 `final` 필드와 불변 객체 사용:

```java
// ❌ 잘못된 예
private String name;

// ✅ 올바른 예
private final String name;
```
