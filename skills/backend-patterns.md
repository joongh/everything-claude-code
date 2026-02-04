---
name: backend-patterns
description: Spring Boot 백엔드 아키텍처 패턴, API 설계, 데이터베이스 최적화 및 서버 사이드 모범 사례.
---

# 백엔드 개발 패턴

확장 가능한 Spring Boot 서버 사이드 애플리케이션을 위한 백엔드 아키텍처 패턴과 모범 사례.

## API 설계 패턴

### RESTful API 구조

```kotlin
// ✅ 리소스 기반 URL
GET    /api/v1/markets              # 리소스 목록
GET    /api/v1/markets/{id}         # 단일 리소스 조회
POST   /api/v1/markets              # 리소스 생성
PUT    /api/v1/markets/{id}         # 리소스 전체 교체
PATCH  /api/v1/markets/{id}         # 리소스 부분 수정
DELETE /api/v1/markets/{id}         # 리소스 삭제

// ✅ 쿼리 파라미터로 필터링, 정렬, 페이징
GET /api/v1/markets?status=active&sort=volume,desc&page=0&size=20
```

### Controller 계층

```kotlin
@RestController
@RequestMapping("/api/v1/markets")
class MarketController(
    private val marketService: MarketService
) {
    @GetMapping
    fun findAll(
        @RequestParam(required = false) status: MarketStatus?,
        @PageableDefault(size = 20) pageable: Pageable
    ): ResponseEntity<Page<MarketResponse>> {
        return ResponseEntity.ok(marketService.findAll(status, pageable))
    }

    @GetMapping("/{id}")
    fun findById(@PathVariable id: Long): ResponseEntity<MarketResponse> {
        return ResponseEntity.ok(marketService.findById(id))
    }

    @PostMapping
    fun create(
        @Valid @RequestBody request: CreateMarketRequest
    ): ResponseEntity<MarketResponse> {
        val market = marketService.create(request)
        val location = URI.create("/api/v1/markets/${market.id}")
        return ResponseEntity.created(location).body(market)
    }

    @PutMapping("/{id}")
    fun update(
        @PathVariable id: Long,
        @Valid @RequestBody request: UpdateMarketRequest
    ): ResponseEntity<MarketResponse> {
        return ResponseEntity.ok(marketService.update(id, request))
    }

    @DeleteMapping("/{id}")
    fun delete(@PathVariable id: Long): ResponseEntity<Unit> {
        marketService.delete(id)
        return ResponseEntity.noContent().build()
    }
}
```

### Service 계층

```kotlin
@Service
@Transactional(readOnly = true)
class MarketService(
    private val marketRepository: MarketRepository,
    private val eventPublisher: ApplicationEventPublisher
) {
    fun findAll(status: MarketStatus?, pageable: Pageable): Page<MarketResponse> {
        val markets = if (status != null) {
            marketRepository.findByStatus(status, pageable)
        } else {
            marketRepository.findAll(pageable)
        }
        return markets.map { MarketResponse.from(it) }
    }

    fun findById(id: Long): MarketResponse {
        val market = marketRepository.findByIdOrNull(id)
            ?: throw MarketNotFoundException(id)
        return MarketResponse.from(market)
    }

    @Transactional
    fun create(request: CreateMarketRequest): MarketResponse {
        val market = Market(
            name = request.name,
            description = request.description,
            endDate = request.endDate
        )
        val saved = marketRepository.save(market)
        eventPublisher.publishEvent(MarketCreatedEvent(saved.id!!))
        return MarketResponse.from(saved)
    }

    @Transactional
    fun update(id: Long, request: UpdateMarketRequest): MarketResponse {
        val market = marketRepository.findByIdOrNull(id)
            ?: throw MarketNotFoundException(id)

        market.update(request.name, request.description)
        return MarketResponse.from(market)
    }

    @Transactional
    fun delete(id: Long) {
        if (!marketRepository.existsById(id)) {
            throw MarketNotFoundException(id)
        }
        marketRepository.deleteById(id)
    }
}
```

### Repository 계층

```kotlin
interface MarketRepository : JpaRepository<Market, Long> {
    fun findByStatus(status: MarketStatus, pageable: Pageable): Page<Market>
    fun findByNameContainingIgnoreCase(name: String): List<Market>
    fun existsBySlug(slug: String): Boolean

    @Query("""
        SELECT m FROM Market m
        WHERE m.status = :status
        AND m.endDate > :now
        ORDER BY m.volume DESC
    """)
    fun findActiveMarkets(
        @Param("status") status: MarketStatus,
        @Param("now") now: LocalDateTime,
        pageable: Pageable
    ): Page<Market>
}
```

## 데이터베이스 패턴

### N+1 쿼리 방지

```kotlin
// ❌ 잘못된 예: N+1 쿼리 문제
val markets = marketRepository.findAll()
markets.forEach { market ->
    market.creator  // 각 마켓마다 추가 쿼리
}

// ✅ 올바른 예: Fetch Join
@Query("SELECT m FROM Market m JOIN FETCH m.creator")
fun findAllWithCreator(): List<Market>

// ✅ 또는: @EntityGraph
@EntityGraph(attributePaths = ["creator"])
fun findAll(): List<Market>
```

### QueryDSL 동적 쿼리

```kotlin
@Repository
class MarketQueryRepository(
    private val queryFactory: JPAQueryFactory
) {
    private val market = QMarket.market

    fun search(criteria: MarketSearchCriteria): Page<Market> {
        val content = queryFactory
            .selectFrom(market)
            .where(
                statusEquals(criteria.status),
                nameContains(criteria.name),
                createdAfter(criteria.createdAfter)
            )
            .orderBy(market.createdAt.desc())
            .offset(criteria.pageable.offset)
            .limit(criteria.pageable.pageSize.toLong())
            .fetch()

        val total = queryFactory
            .select(market.count())
            .from(market)
            .where(
                statusEquals(criteria.status),
                nameContains(criteria.name),
                createdAfter(criteria.createdAfter)
            )
            .fetchOne() ?: 0L

        return PageImpl(content, criteria.pageable, total)
    }

    private fun statusEquals(status: MarketStatus?) =
        status?.let { market.status.eq(it) }

    private fun nameContains(name: String?) =
        name?.let { market.name.containsIgnoreCase(it) }

    private fun createdAfter(date: LocalDateTime?) =
        date?.let { market.createdAt.goe(it) }
}
```

### 트랜잭션 관리

```kotlin
@Service
@Transactional(readOnly = true)
class OrderService(
    private val orderRepository: OrderRepository,
    private val productService: ProductService
) {
    @Transactional
    fun createOrder(request: CreateOrderRequest): OrderResponse {
        // 재고 확인 및 감소
        val product = productService.decreaseStock(
            request.productId,
            request.quantity
        )

        // 주문 생성
        val order = Order(
            productId = request.productId,
            quantity = request.quantity,
            totalPrice = product.price * request.quantity
        )

        return OrderResponse.from(orderRepository.save(order))
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun processPayment(orderId: Long): PaymentResult {
        // 별도 트랜잭션에서 결제 처리
    }
}
```

## 캐싱 전략

### Spring Cache

```kotlin
@Configuration
@EnableCaching
class CacheConfig {
    @Bean
    fun cacheManager(): CacheManager {
        return CaffeineCacheManager().apply {
            setCaffeine(
                Caffeine.newBuilder()
                    .maximumSize(1000)
                    .expireAfterWrite(Duration.ofMinutes(10))
            )
        }
    }
}

@Service
class MarketService(
    private val marketRepository: MarketRepository
) {
    @Cacheable(value = ["markets"], key = "#id")
    fun findById(id: Long): MarketResponse {
        val market = marketRepository.findByIdOrNull(id)
            ?: throw MarketNotFoundException(id)
        return MarketResponse.from(market)
    }

    @CacheEvict(value = ["markets"], key = "#id")
    @Transactional
    fun update(id: Long, request: UpdateMarketRequest): MarketResponse {
        // 업데이트 로직
    }

    @CacheEvict(value = ["markets"], allEntries = true)
    @Transactional
    fun refreshAllMarkets() {
        // 전체 캐시 무효화
    }
}
```

### Redis 캐시

```kotlin
@Configuration
class RedisConfig {
    @Bean
    fun redisTemplate(connectionFactory: RedisConnectionFactory): RedisTemplate<String, Any> {
        return RedisTemplate<String, Any>().apply {
            setConnectionFactory(connectionFactory)
            keySerializer = StringRedisSerializer()
            valueSerializer = GenericJackson2JsonRedisSerializer()
        }
    }
}

@Service
class CachedMarketService(
    private val marketRepository: MarketRepository,
    private val redisTemplate: RedisTemplate<String, Any>
) {
    companion object {
        private const val CACHE_KEY_PREFIX = "market:"
        private val CACHE_TTL = Duration.ofMinutes(10)
    }

    fun findById(id: Long): MarketResponse {
        val cacheKey = "$CACHE_KEY_PREFIX$id"

        // 캐시 확인
        val cached = redisTemplate.opsForValue().get(cacheKey)
        if (cached != null) {
            return cached as MarketResponse
        }

        // DB 조회
        val market = marketRepository.findByIdOrNull(id)
            ?: throw MarketNotFoundException(id)
        val response = MarketResponse.from(market)

        // 캐시 저장
        redisTemplate.opsForValue().set(cacheKey, response, CACHE_TTL)

        return response
    }
}
```

## 에러 처리 패턴

### 전역 예외 핸들러

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    private val log = LoggerFactory.getLogger(javaClass)

    @ExceptionHandler(BusinessException::class)
    fun handleBusinessException(e: BusinessException): ResponseEntity<ErrorResponse> {
        log.warn("비즈니스 예외: ${e.message}")
        return ResponseEntity
            .status(e.errorCode.status)
            .body(ErrorResponse(e.errorCode.name, e.message))
    }

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidationException(
        e: MethodArgumentNotValidException
    ): ResponseEntity<ErrorResponse> {
        val errors = e.bindingResult.fieldErrors
            .associate { it.field to (it.defaultMessage ?: "Invalid") }

        return ResponseEntity
            .badRequest()
            .body(ErrorResponse("VALIDATION_ERROR", "입력값 검증 실패", errors))
    }

    @ExceptionHandler(Exception::class)
    fun handleException(e: Exception): ResponseEntity<ErrorResponse> {
        log.error("예상치 못한 에러", e)
        return ResponseEntity
            .internalServerError()
            .body(ErrorResponse("INTERNAL_ERROR", "서버 오류가 발생했습니다"))
    }
}
```

## 비동기 처리

### 이벤트 기반 처리

```kotlin
// 이벤트 정의
data class OrderCreatedEvent(
    val orderId: Long,
    val userId: Long
)

// 이벤트 발행
@Service
class OrderService(
    private val eventPublisher: ApplicationEventPublisher
) {
    @Transactional
    fun create(request: CreateOrderRequest): OrderResponse {
        val order = orderRepository.save(Order(...))

        // 트랜잭션 커밋 후 이벤트 발행
        TransactionSynchronizationManager.registerSynchronization(
            object : TransactionSynchronization {
                override fun afterCommit() {
                    eventPublisher.publishEvent(
                        OrderCreatedEvent(order.id!!, order.userId)
                    )
                }
            }
        )

        return OrderResponse.from(order)
    }
}

// 이벤트 리스너
@Component
class OrderEventListener(
    private val notificationService: NotificationService
) {
    @Async
    @EventListener
    fun handleOrderCreated(event: OrderCreatedEvent) {
        notificationService.sendOrderConfirmation(event.orderId)
    }
}
```

**기억하세요**: 백엔드 패턴은 확장 가능하고 유지보수하기 쉬운 서버 사이드 애플리케이션을 가능하게 합니다. 복잡도 수준에 맞는 패턴을 선택하세요.
