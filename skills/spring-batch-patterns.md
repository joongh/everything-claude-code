# Spring Batch 패턴

Spring Batch를 사용한 대용량 데이터 처리 패턴을 정의합니다.

## 기본 설정

### build.gradle.kts
```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-batch")
    implementation("org.springframework.batch:spring-batch-integration")
    testImplementation("org.springframework.batch:spring-batch-test")
}
```

### 배치 설정
```kotlin
@Configuration
@EnableBatchProcessing
class BatchConfig(
    private val jobRepository: JobRepository,
    private val transactionManager: PlatformTransactionManager
) {
    // 공통 설정
}
```

## Job/Step 구조

### 기본 Job 구성
```kotlin
@Configuration
class UserSyncJobConfig(
    private val jobRepository: JobRepository,
    private val transactionManager: PlatformTransactionManager,
    private val userReader: ItemReader<UserEntity>,
    private val userProcessor: ItemProcessor<UserEntity, UserDto>,
    private val userWriter: ItemWriter<UserDto>
) {
    @Bean
    fun userSyncJob(): Job {
        return JobBuilder("userSyncJob", jobRepository)
            .incrementer(RunIdIncrementer())
            .listener(jobExecutionListener())
            .start(userSyncStep())
            .build()
    }

    @Bean
    fun userSyncStep(): Step {
        return StepBuilder("userSyncStep", jobRepository)
            .chunk<UserEntity, UserDto>(100, transactionManager)
            .reader(userReader)
            .processor(userProcessor)
            .writer(userWriter)
            .faultTolerant()
            .skipLimit(10)
            .skip(DataIntegrityViolationException::class.java)
            .retryLimit(3)
            .retry(TransientDataAccessException::class.java)
            .listener(stepExecutionListener())
            .build()
    }

    @Bean
    fun jobExecutionListener() = object : JobExecutionListener {
        private val log = LoggerFactory.getLogger(javaClass)

        override fun beforeJob(jobExecution: JobExecution) {
            log.info("Job 시작: ${jobExecution.jobInstance.jobName}")
        }

        override fun afterJob(jobExecution: JobExecution) {
            log.info("Job 완료: ${jobExecution.status}")
            if (jobExecution.status == BatchStatus.FAILED) {
                log.error("Job 실패", jobExecution.allFailureExceptions.firstOrNull())
            }
        }
    }

    @Bean
    fun stepExecutionListener() = object : StepExecutionListener {
        private val log = LoggerFactory.getLogger(javaClass)

        override fun afterStep(stepExecution: StepExecution): ExitStatus {
            log.info("""
                Step 완료: ${stepExecution.stepName}
                - 읽기: ${stepExecution.readCount}
                - 처리: ${stepExecution.writeCount}
                - 스킵: ${stepExecution.skipCount}
            """.trimIndent())
            return stepExecution.exitStatus
        }
    }
}
```

## Reader 패턴

### JPA Reader
```kotlin
@Bean
@StepScope
fun userJpaReader(
    entityManagerFactory: EntityManagerFactory,
    @Value("#{jobParameters['status']}") status: String?
): JpaPagingItemReader<UserEntity> {
    return JpaPagingItemReaderBuilder<UserEntity>()
        .name("userJpaReader")
        .entityManagerFactory(entityManagerFactory)
        .queryString("SELECT u FROM UserEntity u WHERE u.status = :status")
        .parameterValues(mapOf("status" to status))
        .pageSize(100)
        .build()
}
```

### JDBC Reader
```kotlin
@Bean
@StepScope
fun userJdbcReader(
    dataSource: DataSource,
    @Value("#{jobParameters['createdAfter']}") createdAfter: String?
): JdbcCursorItemReader<UserEntity> {
    return JdbcCursorItemReaderBuilder<UserEntity>()
        .name("userJdbcReader")
        .dataSource(dataSource)
        .sql("""
            SELECT id, email, name, status, created_at
            FROM users
            WHERE created_at >= ?
            ORDER BY id
        """)
        .preparedStatementSetter { ps ->
            ps.setTimestamp(1, Timestamp.valueOf(LocalDateTime.parse(createdAfter)))
        }
        .rowMapper { rs, _ ->
            UserEntity(
                id = rs.getLong("id"),
                email = rs.getString("email"),
                name = rs.getString("name"),
                status = UserStatus.valueOf(rs.getString("status")),
                createdAt = rs.getTimestamp("created_at").toLocalDateTime()
            )
        }
        .build()
}
```

### 커스텀 Reader (API 호출)
```kotlin
@Component
@StepScope
class ExternalApiReader(
    private val apiClient: ExternalApiClient,
    @Value("#{jobParameters['pageSize']}") private val pageSize: Int = 100
) : ItemReader<ExternalUserDto> {

    private var currentPage = 0
    private var currentItems: MutableList<ExternalUserDto> = mutableListOf()
    private var hasMore = true

    override fun read(): ExternalUserDto? {
        if (currentItems.isEmpty() && hasMore) {
            fetchNextPage()
        }
        return if (currentItems.isNotEmpty()) {
            currentItems.removeAt(0)
        } else {
            null
        }
    }

    private fun fetchNextPage() {
        val response = apiClient.getUsers(page = currentPage, size = pageSize)
        currentItems.addAll(response.data)
        hasMore = response.hasNext
        currentPage++
    }
}
```

## Processor 패턴

### 기본 Processor
```kotlin
@Component
class UserProcessor : ItemProcessor<UserEntity, UserDto> {

    private val log = LoggerFactory.getLogger(javaClass)

    override fun process(item: UserEntity): UserDto? {
        // null 반환 시 해당 아이템 스킵
        if (!isValid(item)) {
            log.warn("Invalid user skipped: ${item.id}")
            return null
        }

        return UserDto(
            id = item.id!!,
            email = item.email,
            name = item.name.uppercase(),
            processedAt = LocalDateTime.now()
        )
    }

    private fun isValid(user: UserEntity): Boolean {
        return user.email.isNotBlank() && user.name.isNotBlank()
    }
}
```

### 복합 Processor
```kotlin
@Bean
fun compositeProcessor(
    validationProcessor: ValidationProcessor,
    transformProcessor: TransformProcessor,
    enrichmentProcessor: EnrichmentProcessor
): CompositeItemProcessor<UserEntity, UserDto> {
    return CompositeItemProcessorBuilder<UserEntity, UserDto>()
        .delegates(listOf(
            validationProcessor,
            transformProcessor,
            enrichmentProcessor
        ))
        .build()
}
```

### 조건부 처리
```kotlin
@Component
class ConditionalProcessor(
    private val premiumProcessor: ItemProcessor<UserEntity, UserDto>,
    private val standardProcessor: ItemProcessor<UserEntity, UserDto>
) : ItemProcessor<UserEntity, UserDto> {

    override fun process(item: UserEntity): UserDto? {
        return if (item.isPremium) {
            premiumProcessor.process(item)
        } else {
            standardProcessor.process(item)
        }
    }
}
```

## Writer 패턴

### JPA Writer
```kotlin
@Bean
fun userJpaWriter(entityManagerFactory: EntityManagerFactory): JpaItemWriter<UserEntity> {
    return JpaItemWriterBuilder<UserEntity>()
        .entityManagerFactory(entityManagerFactory)
        .build()
}
```

### JDBC Writer
```kotlin
@Bean
fun userJdbcWriter(dataSource: DataSource): JdbcBatchItemWriter<UserDto> {
    return JdbcBatchItemWriterBuilder<UserDto>()
        .dataSource(dataSource)
        .sql("""
            INSERT INTO user_sync_results (user_id, email, name, processed_at)
            VALUES (:id, :email, :name, :processedAt)
            ON CONFLICT (user_id) DO UPDATE SET
                email = EXCLUDED.email,
                name = EXCLUDED.name,
                processed_at = EXCLUDED.processed_at
        """)
        .beanMapped()
        .build()
}
```

### 복합 Writer
```kotlin
@Bean
fun compositeWriter(
    databaseWriter: JpaItemWriter<UserEntity>,
    fileWriter: FlatFileItemWriter<UserEntity>,
    notificationWriter: NotificationWriter
): CompositeItemWriter<UserEntity> {
    return CompositeItemWriterBuilder<UserEntity>()
        .delegates(listOf(databaseWriter, fileWriter, notificationWriter))
        .build()
}
```

### 커스텀 Writer
```kotlin
@Component
class KafkaWriter(
    private val kafkaTemplate: KafkaTemplate<String, UserEvent>
) : ItemWriter<UserDto> {

    override fun write(chunk: Chunk<out UserDto>) {
        chunk.items.forEach { user ->
            val event = UserEvent(
                type = "USER_SYNCED",
                userId = user.id,
                timestamp = LocalDateTime.now()
            )
            kafkaTemplate.send("user-events", user.id.toString(), event)
        }
    }
}
```

## 흐름 제어

### 조건부 Step 실행
```kotlin
@Bean
fun conditionalJob(): Job {
    return JobBuilder("conditionalJob", jobRepository)
        .start(validateStep())
        .on("FAILED").to(errorHandlingStep())
        .from(validateStep())
        .on("*").to(processStep())
        .from(processStep())
        .on("*").to(reportStep())
        .end()
        .build()
}
```

### 병렬 Step 실행
```kotlin
@Bean
fun parallelJob(): Job {
    return JobBuilder("parallelJob", jobRepository)
        .start(splitFlow())
        .build()
        .build()
}

@Bean
fun splitFlow(): Flow {
    return FlowBuilder<SimpleFlow>("splitFlow")
        .split(taskExecutor())
        .add(
            flow1(),
            flow2(),
            flow3()
        )
        .build()
}

@Bean
fun taskExecutor(): TaskExecutor {
    return ThreadPoolTaskExecutor().apply {
        corePoolSize = 4
        maxPoolSize = 8
        setThreadNamePrefix("batch-parallel-")
        initialize()
    }
}
```

## 파티셔닝

### Master Step
```kotlin
@Bean
fun partitionedStep(): Step {
    return StepBuilder("partitionedStep", jobRepository)
        .partitioner("workerStep", rangePartitioner())
        .step(workerStep())
        .gridSize(10)
        .taskExecutor(taskExecutor())
        .build()
}

@Bean
fun rangePartitioner(): Partitioner {
    return Partitioner { gridSize ->
        val result = mutableMapOf<String, ExecutionContext>()
        val totalCount = userRepository.count()
        val rangeSize = totalCount / gridSize + 1

        for (i in 0 until gridSize) {
            val context = ExecutionContext()
            context.putLong("minId", i * rangeSize)
            context.putLong("maxId", (i + 1) * rangeSize - 1)
            result["partition$i"] = context
        }
        result
    }
}

@Bean
@StepScope
fun partitionedReader(
    @Value("#{stepExecutionContext['minId']}") minId: Long,
    @Value("#{stepExecutionContext['maxId']}") maxId: Long
): JpaPagingItemReader<UserEntity> {
    return JpaPagingItemReaderBuilder<UserEntity>()
        .name("partitionedReader")
        .entityManagerFactory(entityManagerFactory)
        .queryString("SELECT u FROM UserEntity u WHERE u.id BETWEEN :minId AND :maxId")
        .parameterValues(mapOf("minId" to minId, "maxId" to maxId))
        .pageSize(100)
        .build()
}
```

## 스케줄링

### @Scheduled 사용
```kotlin
@Component
class BatchScheduler(
    private val jobLauncher: JobLauncher,
    private val userSyncJob: Job
) {
    private val log = LoggerFactory.getLogger(javaClass)

    @Scheduled(cron = "0 0 2 * * *")  // 매일 새벽 2시
    fun runUserSyncJob() {
        try {
            val params = JobParametersBuilder()
                .addLocalDateTime("executionTime", LocalDateTime.now())
                .addString("status", "ACTIVE")
                .toJobParameters()

            val execution = jobLauncher.run(userSyncJob, params)
            log.info("Job 실행 완료: ${execution.status}")
        } catch (e: Exception) {
            log.error("Job 실행 실패", e)
        }
    }
}
```

## 테스트

```kotlin
@SpringBatchTest
@SpringBootTest
class UserSyncJobTest(
    @Autowired private val jobLauncherTestUtils: JobLauncherTestUtils,
    @Autowired private val jobRepositoryTestUtils: JobRepositoryTestUtils
) {
    @BeforeEach
    fun setup() {
        jobRepositoryTestUtils.removeJobExecutions()
    }

    @Test
    fun `userSyncJob should complete successfully`() {
        // given
        val params = JobParametersBuilder()
            .addString("status", "ACTIVE")
            .toJobParameters()

        // when
        val execution = jobLauncherTestUtils.launchJob(params)

        // then
        assertThat(execution.status).isEqualTo(BatchStatus.COMPLETED)
        assertThat(execution.exitStatus).isEqualTo(ExitStatus.COMPLETED)
    }

    @Test
    fun `userSyncStep should process items correctly`() {
        // when
        val execution = jobLauncherTestUtils.launchStep("userSyncStep")

        // then
        assertThat(execution.status).isEqualTo(BatchStatus.COMPLETED)
        assertThat(execution.readCount).isGreaterThan(0)
        assertThat(execution.writeCount).isEqualTo(execution.readCount)
    }
}
```

## 모범 사례

1. **청크 크기 최적화**: 일반적으로 100-1000 사이, 데이터 크기와 트랜잭션 시간 고려
2. **멱등성 보장**: 동일 파라미터로 재실행해도 동일 결과
3. **재시작 가능성**: `restartable(true)` 설정으로 실패 지점부터 재시작
4. **스킵 정책**: 예상 가능한 예외는 스킵, 치명적 예외는 실패
5. **모니터링**: JobExecutionListener, StepExecutionListener 활용
6. **파라미터 관리**: JobParameters로 실행 조건 명시
