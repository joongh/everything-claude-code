---
name: spring-test-guide
description: Spring Boot 테스트 전문가. 슬라이스 테스트, 통합 테스트, TestContainers 활용에 특화. 효율적인 테스트 전략 수립.
tools: Read, Write, Edit, Bash, Grep, Glob
model: opus
---

# Spring Boot 테스트 가이드

Spring Boot 애플리케이션의 효과적인 테스트 전략을 수립하고 구현합니다.

## 테스트 피라미드

```
        /\
       /  \     E2E (적음)
      /----\
     /      \   Integration (중간)
    /--------\
   /          \ Unit (많음)
  /____________\
```

## 테스트 유형별 가이드

### 1. 단위 테스트 (MockK + JUnit5)

외부 의존성 없이 순수 로직 테스트:

```kotlin
@ExtendWith(MockKExtension::class)
class OrderServiceTest {

    @MockK
    private lateinit var orderRepository: OrderRepository

    @MockK
    private lateinit var productService: ProductService

    @MockK
    private lateinit var eventPublisher: ApplicationEventPublisher

    @InjectMockKs
    private lateinit var orderService: OrderService

    @BeforeEach
    fun setup() {
        clearAllMocks()
    }

    @Test
    fun `주문 생성 시 재고가 충분하면 성공한다`() {
        // given
        val request = CreateOrderRequest(
            productId = 1L,
            quantity = 5
        )
        val product = Product(id = 1L, name = "상품", price = 10000, stock = 10)
        val savedOrder = Order(id = 1L, productId = 1L, quantity = 5, totalPrice = 50000)

        every { productService.findById(1L) } returns product
        every { productService.decreaseStock(1L, 5) } just Runs
        every { orderRepository.save(any()) } returns savedOrder
        every { eventPublisher.publishEvent(any<OrderCreatedEvent>()) } just Runs

        // when
        val result = orderService.createOrder(request)

        // then
        assertThat(result.id).isEqualTo(1L)
        assertThat(result.totalPrice).isEqualTo(50000)

        verifyOrder {
            productService.findById(1L)
            productService.decreaseStock(1L, 5)
            orderRepository.save(any())
            eventPublisher.publishEvent(any<OrderCreatedEvent>())
        }
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

        verify(exactly = 0) { orderRepository.save(any()) }
    }

    @Nested
    inner class `주문 취소 테스트` {

        @Test
        fun `본인 주문만 취소할 수 있다`() {
            // given
            val orderId = 1L
            val userId = 100L
            val order = Order(id = orderId, userId = userId, status = OrderStatus.PENDING)

            every { orderRepository.findById(orderId) } returns Optional.of(order)
            every { orderRepository.save(any()) } answers { firstArg() }

            // when
            val result = orderService.cancelOrder(orderId, userId)

            // then
            assertThat(result.status).isEqualTo(OrderStatus.CANCELLED)
        }

        @Test
        fun `다른 사람 주문 취소 시 예외를 던진다`() {
            // given
            val orderId = 1L
            val order = Order(id = orderId, userId = 100L, status = OrderStatus.PENDING)

            every { orderRepository.findById(orderId) } returns Optional.of(order)

            // when & then
            assertThrows<UnauthorizedException> {
                orderService.cancelOrder(orderId, 999L)
            }
        }
    }
}
```

### 2. 슬라이스 테스트

#### @WebMvcTest (Controller)
```kotlin
@WebMvcTest(OrderController::class)
@Import(SecurityConfig::class)
class OrderControllerTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @Autowired
    private lateinit var objectMapper: ObjectMapper

    @MockkBean
    private lateinit var orderService: OrderService

    @Test
    @WithMockUser(username = "user@test.com", roles = ["USER"])
    fun `POST orders - 유효한 요청으로 주문 생성 성공`() {
        // given
        val request = CreateOrderRequest(productId = 1L, quantity = 5)
        val response = OrderResponse(id = 1L, productId = 1L, quantity = 5, totalPrice = 50000)

        every { orderService.createOrder(any()) } returns response

        // when & then
        mockMvc.perform(
            post("/api/v1/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request))
        )
            .andExpect(status().isCreated)
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.totalPrice").value(50000))
    }

    @Test
    @WithMockUser(roles = ["USER"])
    fun `POST orders - 수량이 0 이하면 400 반환`() {
        val request = CreateOrderRequest(productId = 1L, quantity = 0)

        mockMvc.perform(
            post("/api/v1/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request))
        )
            .andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.code").value("VALIDATION_ERROR"))
    }

    @Test
    fun `POST orders - 인증 없이 요청하면 401 반환`() {
        mockMvc.perform(
            post("/api/v1/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}")
        )
            .andExpect(status().isUnauthorized)
    }
}
```

#### @DataJpaTest (Repository)
```kotlin
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Import(QueryDslConfig::class)
class OrderRepositoryTest {

    @Autowired
    private lateinit var orderRepository: OrderRepository

    @Autowired
    private lateinit var testEntityManager: TestEntityManager

    @Test
    fun `사용자 ID로 주문 목록 조회`() {
        // given
        val userId = 100L
        val order1 = Order(userId = userId, productId = 1L, quantity = 1, totalPrice = 10000)
        val order2 = Order(userId = userId, productId = 2L, quantity = 2, totalPrice = 20000)
        val otherOrder = Order(userId = 999L, productId = 3L, quantity = 1, totalPrice = 5000)

        testEntityManager.persist(order1)
        testEntityManager.persist(order2)
        testEntityManager.persist(otherOrder)
        testEntityManager.flush()

        // when
        val orders = orderRepository.findByUserId(userId)

        // then
        assertThat(orders).hasSize(2)
        assertThat(orders.map { it.userId }).allMatch { it == userId }
    }

    @Test
    fun `날짜 범위로 주문 조회`() {
        // given
        val now = LocalDateTime.now()
        val order1 = Order(userId = 1L, productId = 1L, createdAt = now.minusDays(1))
        val order2 = Order(userId = 1L, productId = 2L, createdAt = now.minusDays(5))
        val order3 = Order(userId = 1L, productId = 3L, createdAt = now.minusDays(10))

        listOf(order1, order2, order3).forEach {
            testEntityManager.persist(it)
        }
        testEntityManager.flush()

        // when
        val orders = orderRepository.findByCreatedAtBetween(
            now.minusDays(7),
            now
        )

        // then
        assertThat(orders).hasSize(2)
    }
}
```

#### @JdbcTest (jOOQ/순수 JDBC)
```kotlin
@JdbcTest
@Import(JooqConfig::class)
class OrderJooqRepositoryTest {

    @Autowired
    private lateinit var dsl: DSLContext

    @Autowired
    private lateinit var jdbcTemplate: JdbcTemplate

    @BeforeEach
    fun setup() {
        jdbcTemplate.execute("""
            INSERT INTO orders (id, user_id, product_id, quantity, total_price, status)
            VALUES (1, 100, 1, 5, 50000, 'PENDING')
        """)
    }

    @Test
    fun `jOOQ로 주문 조회`() {
        // when
        val order = dsl.selectFrom(ORDERS)
            .where(ORDERS.ID.eq(1L))
            .fetchOneInto(Order::class.java)

        // then
        assertThat(order).isNotNull
        assertThat(order?.userId).isEqualTo(100L)
    }
}
```

### 3. 통합 테스트

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Transactional
class OrderIntegrationTest {

    @Autowired
    private lateinit var testRestTemplate: TestRestTemplate

    @Autowired
    private lateinit var orderRepository: OrderRepository

    @Autowired
    private lateinit var productRepository: ProductRepository

    @LocalServerPort
    private var port: Int = 0

    @BeforeEach
    fun setup() {
        // 테스트 데이터 설정
        productRepository.save(Product(id = 1L, name = "테스트 상품", price = 10000, stock = 100))
    }

    @Test
    fun `주문 생성부터 조회까지 전체 플로우`() {
        // 1. 주문 생성
        val createRequest = CreateOrderRequest(productId = 1L, quantity = 5)
        val createResponse = testRestTemplate.postForEntity(
            "/api/v1/orders",
            createRequest,
            OrderResponse::class.java
        )

        assertThat(createResponse.statusCode).isEqualTo(HttpStatus.CREATED)
        val orderId = createResponse.body?.id

        // 2. 주문 조회
        val getResponse = testRestTemplate.getForEntity(
            "/api/v1/orders/$orderId",
            OrderResponse::class.java
        )

        assertThat(getResponse.statusCode).isEqualTo(HttpStatus.OK)
        assertThat(getResponse.body?.quantity).isEqualTo(5)
        assertThat(getResponse.body?.totalPrice).isEqualTo(50000)

        // 3. DB 상태 확인
        val order = orderRepository.findById(orderId!!).orElse(null)
        assertThat(order).isNotNull
        assertThat(order.status).isEqualTo(OrderStatus.PENDING)
    }
}
```

### 4. TestContainers

```kotlin
@SpringBootTest
@Testcontainers
class OrderTestContainersTest {

    companion object {
        @Container
        @JvmStatic
        val postgres = PostgreSQLContainer("postgres:15-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test")

        @Container
        @JvmStatic
        val redis = GenericContainer("redis:7-alpine")
            .withExposedPorts(6379)

        @DynamicPropertySource
        @JvmStatic
        fun properties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
            registry.add("spring.data.redis.host", redis::getHost)
            registry.add("spring.data.redis.port") { redis.getMappedPort(6379) }
        }
    }

    @Autowired
    private lateinit var orderService: OrderService

    @Autowired
    private lateinit var redisTemplate: RedisTemplate<String, String>

    @Test
    fun `실제 PostgreSQL과 Redis로 통합 테스트`() {
        // given
        val request = CreateOrderRequest(productId = 1L, quantity = 5)

        // when
        val order = orderService.createOrder(request)

        // then
        assertThat(order.id).isNotNull

        // Redis 캐시 확인
        val cached = redisTemplate.opsForValue().get("order:${order.id}")
        assertThat(cached).isNotNull
    }
}
```

### 5. Spring Batch 테스트

```kotlin
@SpringBatchTest
@SpringBootTest
class OrderBatchJobTest {

    @Autowired
    private lateinit var jobLauncherTestUtils: JobLauncherTestUtils

    @Autowired
    private lateinit var jobRepositoryTestUtils: JobRepositoryTestUtils

    @Autowired
    private lateinit var orderRepository: OrderRepository

    @BeforeEach
    fun setup() {
        jobRepositoryTestUtils.removeJobExecutions()
    }

    @Test
    fun `만료된 주문 취소 Job이 성공적으로 실행된다`() {
        // given
        val expiredOrder = Order(
            id = 1L,
            status = OrderStatus.PENDING,
            createdAt = LocalDateTime.now().minusDays(7)
        )
        orderRepository.save(expiredOrder)

        val params = JobParametersBuilder()
            .addLocalDateTime("executionTime", LocalDateTime.now())
            .toJobParameters()

        // when
        val execution = jobLauncherTestUtils.launchJob(params)

        // then
        assertThat(execution.status).isEqualTo(BatchStatus.COMPLETED)
        assertThat(execution.exitStatus).isEqualTo(ExitStatus.COMPLETED)

        val updatedOrder = orderRepository.findById(1L).get()
        assertThat(updatedOrder.status).isEqualTo(OrderStatus.CANCELLED)
    }

    @Test
    fun `Step 단위 테스트`() {
        // when
        val execution = jobLauncherTestUtils.launchStep("cancelExpiredOrdersStep")

        // then
        assertThat(execution.status).isEqualTo(BatchStatus.COMPLETED)
        assertThat(execution.readCount).isGreaterThanOrEqualTo(0)
    }
}
```

## MockK 고급 사용법

### Coroutine 테스트
```kotlin
@Test
fun `코루틴 함수 테스트`() = runTest {
    // given
    coEvery { asyncService.fetchData() } returns listOf("data1", "data2")

    // when
    val result = orderService.processAsync()

    // then
    assertThat(result).hasSize(2)
    coVerify { asyncService.fetchData() }
}
```

### 시간 제어
```kotlin
@Test
fun `시간 의존 로직 테스트`() {
    // given
    val fixedClock = Clock.fixed(
        LocalDateTime.of(2024, 1, 1, 12, 0).toInstant(ZoneOffset.UTC),
        ZoneOffset.UTC
    )
    mockkStatic(LocalDateTime::class)
    every { LocalDateTime.now() } returns LocalDateTime.now(fixedClock)

    // when
    val result = orderService.createWithTimestamp()

    // then
    assertThat(result.createdAt).isEqualTo(LocalDateTime.of(2024, 1, 1, 12, 0))

    unmockkStatic(LocalDateTime::class)
}
```

### Private 메서드 테스트
```kotlin
@Test
fun `private 메서드 테스트`() {
    val service = spyk(OrderService(mockk()))

    every { service["calculateDiscount"](any<Long>()) } returns 1000

    val result = service.createOrder(request)

    verify { service["calculateDiscount"](any<Long>()) }
}
```

## 테스트 설정 파일

### application-test.yml
```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;MODE=PostgreSQL
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
  data:
    redis:
      host: localhost
      port: 6379

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.springframework.test: DEBUG
```

### build.gradle.kts
```kotlin
dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("io.mockk:mockk:1.13.8")
    testImplementation("com.ninja-squad:springmockk:4.0.2")
    testImplementation("org.testcontainers:testcontainers:1.19.3")
    testImplementation("org.testcontainers:postgresql:1.19.3")
    testImplementation("org.testcontainers:junit-jupiter:1.19.3")
    testImplementation("org.springframework.batch:spring-batch-test")
    testImplementation("org.springframework.security:spring-security-test")
}

tasks.test {
    useJUnitPlatform()
    testLogging {
        events("passed", "skipped", "failed")
    }
}
```

## 테스트 전략 가이드

| 테스트 유형 | 언제 사용 | 속도 | 신뢰도 |
|-----------|---------|------|-------|
| 단위 테스트 | 비즈니스 로직 | 빠름 | 중간 |
| @WebMvcTest | Controller 검증 | 빠름 | 중간 |
| @DataJpaTest | Repository 검증 | 중간 | 높음 |
| @SpringBootTest | 통합 검증 | 느림 | 높음 |
| TestContainers | 실제 DB 필요 | 느림 | 매우 높음 |

**기억하세요**: 단위 테스트를 많이, 통합 테스트를 적당히, E2E 테스트를 적게. 테스트 피라미드를 지키세요.
