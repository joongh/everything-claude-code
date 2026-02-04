---
description: 테스트 주도 개발(TDD) 워크플로우를 실행합니다. 인터페이스 스캐폴딩, 테스트 먼저 작성, 최소 구현, 80%+ 커버리지 보장.
---

# TDD 명령어

이 명령어는 **tdd-guide** 에이전트를 호출하여 테스트 주도 개발 방법론을 적용합니다.

## 이 명령어가 하는 일

1. **인터페이스 스캐폴딩** - 타입/인터페이스 먼저 정의
2. **테스트 먼저 작성** - 실패하는 테스트 작성 (RED)
3. **최소 코드 구현** - 테스트 통과할 만큼만 작성 (GREEN)
4. **리팩토링** - 테스트를 통과시키면서 코드 개선 (REFACTOR)
5. **커버리지 확인** - 80%+ 테스트 커버리지 보장

## 사용 시점

`/tdd` 사용 시점:
- 새로운 기능 구현
- 새로운 함수/컴포넌트 추가
- 버그 수정 (버그 재현 테스트 먼저 작성)
- 기존 코드 리팩토링
- 핵심 비즈니스 로직 구축

## 작동 방식

tdd-guide 에이전트가 수행하는 작업:

1. **인터페이스 정의** - 입력/출력 타입 정의
2. **실패하는 테스트 작성** (아직 코드가 없으므로)
3. **테스트 실행** - 올바른 이유로 실패하는지 확인
4. **최소 구현 작성** - 테스트 통과하도록
5. **테스트 실행** - 통과하는지 확인
6. **리팩토링** - 테스트를 통과시키면서 코드 개선
7. **커버리지 확인** - 80% 미만이면 테스트 추가

## TDD 사이클

```
RED → GREEN → REFACTOR → REPEAT

RED:      실패하는 테스트 작성
GREEN:    통과할 최소 코드 작성
REFACTOR: 코드 개선, 테스트 계속 통과
REPEAT:   다음 기능/시나리오
```

## 예시 사용법

```
User: /tdd 주문 생성 서비스를 구현해야 합니다

Agent (tdd-guide):
# TDD 세션: 주문 생성 서비스

## Step 1: 인터페이스 정의 (SCAFFOLD)

```kotlin
// domain/Order.kt
data class CreateOrderRequest(
    val productId: Long,
    val quantity: Int
)

data class OrderResponse(
    val id: Long,
    val productId: Long,
    val quantity: Int,
    val totalPrice: Long,
    val status: OrderStatus
)
```

## Step 2: 실패하는 테스트 작성 (RED)

```kotlin
// service/OrderServiceTest.kt
@ExtendWith(MockKExtension::class)
class OrderServiceTest {

    @MockK
    private lateinit var orderRepository: OrderRepository

    @MockK
    private lateinit var productService: ProductService

    @InjectMockKs
    private lateinit var orderService: OrderService

    @Test
    fun `주문 생성 시 재고가 충분하면 성공한다`() {
        // given
        val request = CreateOrderRequest(productId = 1L, quantity = 5)
        val product = Product(id = 1L, name = "상품", price = 10000, stock = 10)
        val savedOrder = Order(id = 1L, productId = 1L, quantity = 5, totalPrice = 50000)

        every { productService.findById(1L) } returns product
        every { productService.decreaseStock(1L, 5) } just Runs
        every { orderRepository.save(any()) } returns savedOrder

        // when
        val result = orderService.createOrder(request)

        // then
        assertThat(result.totalPrice).isEqualTo(50000)
        verify { productService.decreaseStock(1L, 5) }
    }

    @Test
    fun `주문 생성 시 재고가 부족하면 예외를 던진다`() {
        // given
        val request = CreateOrderRequest(productId = 1L, quantity = 100)
        val product = Product(id = 1L, name = "상품", price = 10000, stock = 10)

        every { productService.findById(1L) } returns product

        // when & then
        assertThrows<InsufficientStockException> {
            orderService.createOrder(request)
        }
    }
}
```

## Step 3: 테스트 실행 - 실패 확인

```bash
./gradlew test --tests OrderServiceTest

FAIL OrderServiceTest
  ✕ 주문 생성 시 재고가 충분하면 성공한다 (2 ms)
    Error: OrderService 클래스가 존재하지 않습니다

1 test failed
```

✅ 예상대로 테스트 실패. 구현 준비 완료.

## Step 4: 최소 코드 구현 (GREEN)

```kotlin
// service/OrderService.kt
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val productService: ProductService
) {
    @Transactional
    fun createOrder(request: CreateOrderRequest): OrderResponse {
        val product = productService.findById(request.productId)

        if (product.stock < request.quantity) {
            throw InsufficientStockException("재고 부족")
        }

        productService.decreaseStock(request.productId, request.quantity)

        val order = Order(
            productId = request.productId,
            quantity = request.quantity,
            totalPrice = product.price * request.quantity
        )

        val saved = orderRepository.save(order)
        return OrderResponse.from(saved)
    }
}
```

## Step 5: 테스트 실행 - 통과 확인

```bash
./gradlew test --tests OrderServiceTest

PASS OrderServiceTest
  ✓ 주문 생성 시 재고가 충분하면 성공한다 (3 ms)
  ✓ 주문 생성 시 재고가 부족하면 예외를 던진다 (2 ms)

2 tests passed
```

✅ 모든 테스트 통과!

## Step 6: 커버리지 확인

```bash
./gradlew test jacocoTestReport

File              | % Lines | % Branches | % Methods
------------------|---------|------------|----------
OrderService.kt   |   100   |    100     |    100

Coverage: 100% ✅ (Target: 80%)
```

✅ TDD 세션 완료!
```

## TDD 모범 사례

**해야 할 것:**
- ✅ 구현 전에 테스트 먼저 작성
- ✅ 구현 전에 테스트가 실패하는지 확인
- ✅ 테스트 통과를 위한 최소 코드 작성
- ✅ 테스트가 통과한 후에만 리팩토링
- ✅ 엣지 케이스와 에러 시나리오 추가
- ✅ 80%+ 커버리지 목표 (핵심 코드는 100%)

**하지 말아야 할 것:**
- ❌ 테스트 전에 구현 작성
- ❌ 변경 후 테스트 실행 건너뛰기
- ❌ 한 번에 너무 많은 코드 작성
- ❌ 실패하는 테스트 무시
- ❌ 구현 세부사항 테스트 (동작을 테스트)
- ❌ 모든 것을 Mock (통합 테스트 선호)

## 포함할 테스트 유형

**단위 테스트** (함수 수준):
- Happy path 시나리오
- 엣지 케이스 (빈 값, null, 최대값)
- 에러 조건
- 경계값

**통합 테스트** (컴포넌트 수준):
- API 엔드포인트
- 데이터베이스 작업
- 외부 서비스 호출

**슬라이스 테스트**:
- @WebMvcTest (Controller)
- @DataJpaTest (Repository)
- @SpringBatchTest (Batch)

## 커버리지 요구사항

- **최소 80%** 모든 코드
- **100% 필수** 대상:
  - 금융 계산
  - 인증 로직
  - 보안 중요 코드
  - 핵심 비즈니스 로직

## 중요 사항

**필수**: 테스트는 구현 전에 작성해야 합니다. TDD 사이클:

1. **RED** - 실패하는 테스트 작성
2. **GREEN** - 통과하도록 구현
3. **REFACTOR** - 코드 개선

RED 단계를 건너뛰지 마세요. 테스트 전에 코드를 작성하지 마세요.

## 다른 명령어와의 통합

- 먼저 `/plan`으로 무엇을 만들지 이해
- `/tdd`로 테스트와 함께 구현
- 빌드 에러 발생 시 `/build-fix`
- `/test-coverage`로 커버리지 확인

## 관련 에이전트

이 명령어는 `tdd-guide` 에이전트를 호출합니다.
