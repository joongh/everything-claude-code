# jOOQ/QueryDSL 패턴

타입 안전한 SQL 쿼리를 위한 jOOQ와 QueryDSL 사용 패턴을 정의합니다.

## jOOQ

### 기본 설정

#### build.gradle.kts
```kotlin
plugins {
    id("nu.studer.jooq") version "8.2"
}

dependencies {
    implementation("org.jooq:jooq:3.18.6")
    jooqGenerator("org.postgresql:postgresql:42.6.0")
}

jooq {
    configurations {
        create("main") {
            jooqConfiguration.apply {
                jdbc.apply {
                    driver = "org.postgresql.Driver"
                    url = "jdbc:postgresql://localhost:5432/myapp"
                    user = "postgres"
                    password = "password"
                }
                generator.apply {
                    name = "org.jooq.codegen.KotlinGenerator"
                    database.apply {
                        name = "org.jooq.meta.postgres.PostgresDatabase"
                        inputSchema = "public"
                    }
                    target.apply {
                        packageName = "com.example.jooq"
                        directory = "src/main/kotlin"
                    }
                }
            }
        }
    }
}
```

### 기본 CRUD

```kotlin
@Repository
class UserJooqRepository(
    private val dsl: DSLContext
) {
    private val u = USERS

    fun findById(id: Long): User? {
        return dsl.selectFrom(u)
            .where(u.ID.eq(id))
            .fetchOneInto(User::class.java)
    }

    fun findAll(): List<User> {
        return dsl.selectFrom(u)
            .fetchInto(User::class.java)
    }

    fun insert(user: User): User {
        return dsl.insertInto(u)
            .set(u.EMAIL, user.email)
            .set(u.NAME, user.name)
            .set(u.STATUS, user.status.name)
            .returning()
            .fetchOneInto(User::class.java)!!
    }

    fun update(user: User): Int {
        return dsl.update(u)
            .set(u.NAME, user.name)
            .set(u.EMAIL, user.email)
            .where(u.ID.eq(user.id))
            .execute()
    }

    fun deleteById(id: Long): Int {
        return dsl.deleteFrom(u)
            .where(u.ID.eq(id))
            .execute()
    }
}
```

### 조건부 쿼리

```kotlin
fun search(criteria: UserSearchCriteria): List<User> {
    return dsl.selectFrom(u)
        .where(buildConditions(criteria))
        .orderBy(u.CREATED_AT.desc())
        .limit(criteria.limit)
        .offset(criteria.offset)
        .fetchInto(User::class.java)
}

private fun buildConditions(criteria: UserSearchCriteria): Condition {
    var condition = DSL.trueCondition()

    criteria.email?.let {
        condition = condition.and(u.EMAIL.containsIgnoreCase(it))
    }
    criteria.name?.let {
        condition = condition.and(u.NAME.containsIgnoreCase(it))
    }
    criteria.status?.let {
        condition = condition.and(u.STATUS.eq(it.name))
    }
    criteria.createdAfter?.let {
        condition = condition.and(u.CREATED_AT.ge(it))
    }

    return condition
}
```

### 조인 쿼리

```kotlin
fun findUserWithOrders(userId: Long): UserWithOrders? {
    val u = USERS
    val o = ORDERS

    val result = dsl.select(
            u.ID, u.EMAIL, u.NAME,
            o.ID, o.AMOUNT, o.STATUS
        )
        .from(u)
        .leftJoin(o).on(o.USER_ID.eq(u.ID))
        .where(u.ID.eq(userId))
        .fetch()

    if (result.isEmpty()) return null

    val user = result.first().let {
        User(
            id = it[u.ID]!!,
            email = it[u.EMAIL]!!,
            name = it[u.NAME]!!
        )
    }

    val orders = result
        .filter { it[o.ID] != null }
        .map {
            Order(
                id = it[o.ID]!!,
                amount = it[o.AMOUNT]!!,
                status = OrderStatus.valueOf(it[o.STATUS]!!)
            )
        }

    return UserWithOrders(user, orders)
}
```

### 집계 쿼리

```kotlin
fun getOrderStatsByUser(): List<UserOrderStats> {
    val u = USERS
    val o = ORDERS

    return dsl.select(
            u.ID,
            u.NAME,
            DSL.count(o.ID).`as`("order_count"),
            DSL.sum(o.AMOUNT).`as`("total_amount"),
            DSL.avg(o.AMOUNT).`as`("avg_amount")
        )
        .from(u)
        .leftJoin(o).on(o.USER_ID.eq(u.ID))
        .groupBy(u.ID, u.NAME)
        .having(DSL.count(o.ID).gt(0))
        .orderBy(DSL.field("total_amount").desc())
        .fetchInto(UserOrderStats::class.java)
}
```

### 배치 처리

```kotlin
fun batchInsert(users: List<User>): IntArray {
    return dsl.batch(
        users.map { user ->
            dsl.insertInto(u)
                .set(u.EMAIL, user.email)
                .set(u.NAME, user.name)
                .set(u.STATUS, user.status.name)
        }
    ).execute()
}

// 대량 삽입 (더 효율적)
fun bulkInsert(users: List<User>) {
    dsl.batchInsert(
        users.map { user ->
            UsersRecord().apply {
                email = user.email
                name = user.name
                status = user.status.name
            }
        }
    ).execute()
}
```

---

## QueryDSL

### 기본 설정

#### build.gradle.kts
```kotlin
plugins {
    kotlin("kapt") version "1.9.10"
}

dependencies {
    implementation("com.querydsl:querydsl-jpa:5.0.0:jakarta")
    kapt("com.querydsl:querydsl-apt:5.0.0:jakarta")
}

// Q 클래스 생성 경로
kotlin {
    sourceSets.main {
        kotlin.srcDir("build/generated/source/kapt/main")
    }
}
```

### 기본 CRUD

```kotlin
@Repository
class UserQueryRepository(
    private val queryFactory: JPAQueryFactory
) {
    private val user = QUser.user

    fun findById(id: Long): User? {
        return queryFactory
            .selectFrom(user)
            .where(user.id.eq(id))
            .fetchOne()
    }

    fun findAll(): List<User> {
        return queryFactory
            .selectFrom(user)
            .fetch()
    }

    fun findByEmail(email: String): User? {
        return queryFactory
            .selectFrom(user)
            .where(user.email.eq(email))
            .fetchOne()
    }
}
```

### 동적 쿼리

```kotlin
fun search(criteria: UserSearchCriteria): List<User> {
    return queryFactory
        .selectFrom(user)
        .where(
            emailContains(criteria.email),
            nameContains(criteria.name),
            statusEquals(criteria.status),
            createdAfter(criteria.createdAfter)
        )
        .orderBy(user.createdAt.desc())
        .offset(criteria.offset.toLong())
        .limit(criteria.limit.toLong())
        .fetch()
}

private fun emailContains(email: String?): BooleanExpression? {
    return email?.let { user.email.containsIgnoreCase(it) }
}

private fun nameContains(name: String?): BooleanExpression? {
    return name?.let { user.name.containsIgnoreCase(it) }
}

private fun statusEquals(status: UserStatus?): BooleanExpression? {
    return status?.let { user.status.eq(it) }
}

private fun createdAfter(date: LocalDateTime?): BooleanExpression? {
    return date?.let { user.createdAt.goe(it) }
}
```

### BooleanBuilder 활용

```kotlin
fun searchWithBuilder(criteria: UserSearchCriteria): List<User> {
    val builder = BooleanBuilder()

    criteria.email?.let {
        builder.and(user.email.containsIgnoreCase(it))
    }
    criteria.name?.let {
        builder.and(user.name.containsIgnoreCase(it))
    }
    criteria.statuses?.let { statuses ->
        builder.and(user.status.`in`(statuses))
    }
    criteria.createdAfter?.let {
        builder.and(user.createdAt.goe(it))
    }
    criteria.createdBefore?.let {
        builder.and(user.createdAt.loe(it))
    }

    return queryFactory
        .selectFrom(user)
        .where(builder)
        .fetch()
}
```

### 조인 쿼리

```kotlin
fun findUserWithOrders(userId: Long): UserWithOrdersDto? {
    val order = QOrder.order

    return queryFactory
        .select(
            Projections.constructor(
                UserWithOrdersDto::class.java,
                user.id,
                user.name,
                user.email,
                Expressions.asSimple(
                    JPAExpressions
                        .select(order.count())
                        .from(order)
                        .where(order.user.id.eq(user.id))
                )
            )
        )
        .from(user)
        .where(user.id.eq(userId))
        .fetchOne()
}

// 페치 조인
fun findUserWithOrdersFetch(userId: Long): User? {
    val order = QOrder.order

    return queryFactory
        .selectFrom(user)
        .leftJoin(user.orders, order).fetchJoin()
        .where(user.id.eq(userId))
        .fetchOne()
}
```

### DTO 프로젝션

```kotlin
// Projections.constructor 사용
fun findUserSummaries(): List<UserSummaryDto> {
    return queryFactory
        .select(
            Projections.constructor(
                UserSummaryDto::class.java,
                user.id,
                user.name,
                user.email,
                user.status
            )
        )
        .from(user)
        .fetch()
}

// @QueryProjection 사용 (Q 클래스 생성 필요)
data class UserSummaryDto @QueryProjection constructor(
    val id: Long,
    val name: String,
    val email: String,
    val status: UserStatus
)

fun findUserSummariesWithQDto(): List<UserSummaryDto> {
    return queryFactory
        .select(QUserSummaryDto(user.id, user.name, user.email, user.status))
        .from(user)
        .fetch()
}
```

### 집계 쿼리

```kotlin
fun getOrderStatsByStatus(): List<OrderStatusStats> {
    val order = QOrder.order

    return queryFactory
        .select(
            Projections.constructor(
                OrderStatusStats::class.java,
                order.status,
                order.count(),
                order.amount.sum(),
                order.amount.avg()
            )
        )
        .from(order)
        .groupBy(order.status)
        .having(order.count().gt(0))
        .orderBy(order.amount.sum().desc())
        .fetch()
}
```

### 서브쿼리

```kotlin
fun findUsersWithRecentOrders(days: Int): List<User> {
    val order = QOrder.order
    val cutoffDate = LocalDateTime.now().minusDays(days.toLong())

    return queryFactory
        .selectFrom(user)
        .where(
            user.id.`in`(
                JPAExpressions
                    .select(order.user.id)
                    .from(order)
                    .where(order.createdAt.goe(cutoffDate))
            )
        )
        .fetch()
}

// exists 서브쿼리
fun findUsersWithOrders(): List<User> {
    val order = QOrder.order

    return queryFactory
        .selectFrom(user)
        .where(
            JPAExpressions
                .selectOne()
                .from(order)
                .where(order.user.id.eq(user.id))
                .exists()
        )
        .fetch()
}
```

### 페이징

```kotlin
fun findUsersWithPaging(pageable: Pageable): Page<User> {
    val content = queryFactory
        .selectFrom(user)
        .offset(pageable.offset)
        .limit(pageable.pageSize.toLong())
        .orderBy(*getOrderSpecifiers(pageable.sort))
        .fetch()

    val total = queryFactory
        .select(user.count())
        .from(user)
        .fetchOne() ?: 0L

    return PageImpl(content, pageable, total)
}

private fun getOrderSpecifiers(sort: Sort): Array<OrderSpecifier<*>> {
    return sort.map { order ->
        val path = when (order.property) {
            "name" -> user.name
            "email" -> user.email
            "createdAt" -> user.createdAt
            else -> user.id
        }
        if (order.isAscending) path.asc() else path.desc()
    }.toTypedArray()
}
```

### 벌크 연산

```kotlin
fun updateStatusByIds(ids: List<Long>, status: UserStatus): Long {
    return queryFactory
        .update(user)
        .set(user.status, status)
        .where(user.id.`in`(ids))
        .execute()
}

fun deleteInactiveUsers(): Long {
    return queryFactory
        .delete(user)
        .where(user.status.eq(UserStatus.INACTIVE))
        .execute()
}
```

---

## jOOQ vs QueryDSL 선택 가이드

| 상황 | 권장 |
|------|------|
| 복잡한 Native SQL 필요 | jOOQ |
| JPA Entity 기반 프로젝트 | QueryDSL |
| DB 스키마 우선 설계 | jOOQ |
| 도메인 모델 우선 설계 | QueryDSL |
| 다양한 DB 벤더 지원 필요 | jOOQ |
| Spring Data JPA와 통합 | QueryDSL |
