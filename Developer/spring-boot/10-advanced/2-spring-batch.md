# Spring Batch — Xử Lý Dữ Liệu Hàng Loạt

> **Spring Batch** là framework mạnh mẽ để xử lý batch data (dữ liệu hàng loạt) — ETL (Extract, Transform, Load — Trích Xuất, Chuyển Đổi, Nạp), migration (di chuyển dữ liệu), báo cáo định kỳ, và xử lý file lớn hàng triệu records. Spring Batch cung cấp các cơ chế built-in cho retry (thử lại), skip (bỏ qua lỗi), restart (tiếp tục từ điểm dừng) và partitioning (phân vùng song song).

---

## 📋 Mục Lục

1. [Khi Nào Dùng Spring Batch?](#khi-nào-dùng-spring-batch)
2. [Kiến Trúc Spring Batch](#kiến-trúc-spring-batch)
3. [Cấu Hình Cơ Bản](#cấu-hình-cơ-bản)
4. [ItemReader — Đọc Dữ Liệu](#itemreader--đọc-dữ-liệu)
5. [ItemProcessor — Xử Lý Dữ Liệu](#itemprocessor--xử-lý-dữ-liệu)
6. [ItemWriter — Ghi Dữ Liệu](#itemwriter--ghi-dữ-liệu)
7. [Chunk-Oriented Processing](#chunk-oriented-processing)
8. [Tasklet — Tác Vụ Đơn Giản](#tasklet--tác-vụ-đơn-giản)
9. [Job Flow — Điều Khiển Luồng Job](#job-flow--điều-khiển-luồng-job)
10. [Skip & Retry — Chịu Lỗi](#skip--retry--chịu-lỗi)
11. [Partitioning — Xử Lý Song Song](#partitioning--xử-lý-song-song)
12. [JobParameters & JobLauncher](#jobparameters--joblauncher)
13. [Monitoring & Actuator](#monitoring--actuator)
14. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khi Nào Dùng Spring Batch?

```
PHÙ HỢP VỚI SPRING BATCH:
✅ Xử lý file CSV/XML/JSON hàng triệu dòng
✅ ETL từ database cũ sang database mới
✅ Tính toán báo cáo cuối ngày/tháng (end-of-day processing)
✅ Gửi email hàng loạt (bulk email)
✅ Cập nhật lương, điểm tín dụng, trạng thái
✅ Data migration khi upgrade hệ thống
✅ Import/Export dữ liệu giữa hệ thống

KHÔNG PHÙ HỢP:
❌ Real-time event processing (dùng Kafka Streams, Flink)
❌ Tác vụ đơn giản chạy theo lịch (dùng @Scheduled)
❌ Request/Response API
```

---

## Kiến Trúc Spring Batch

```
SPRING BATCH DOMAIN MODEL:

┌─────────────────────────────────────────────────────────┐
│                      JobRepository                       │
│  (Lưu trạng thái Job/Step vào database — metadata DB)   │
└─────────────────────────────────────────────────────────┘
                           ▲
                           │
┌──────────────────────────────────────────────────────┐
│                         JOB                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │  Step 1  │─►│  Step 2  │─►│    Step 3        │   │
│  │          │  │          │  │  (Chunk-oriented) │   │
│  │ Tasklet  │  │ Tasklet  │  │  Reader          │   │
│  │          │  │          │  │  Processor       │   │
│  └──────────┘  └──────────┘  │  Writer          │   │
│                               └──────────────────┘   │
└──────────────────────────────────────────────────────┘
                           ▲
                           │
                      JobLauncher
                      (Khởi động Job)
                           ▲
                           │
              @Scheduled / REST API / CLI

KEY CONCEPTS (Khái Niệm Chính):
- Job: Toàn bộ batch process
- Step: Một giai đoạn trong Job
- JobInstance: Một lần chạy Job với specific parameters
- JobExecution: Một lần thực thi JobInstance (có thể fail và retry)
- StepExecution: Một lần thực thi Step
- ExecutionContext: Map key-value lưu state giữa các bước
- JobRepository: Lưu trữ metadata (status, progress, counts)
```

---

## Cấu Hình Cơ Bản

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-batch</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  batch:
    jdbc:
      initialize-schema: always   # Tạo bảng metadata tự động
    job:
      enabled: false              # Không auto-run job khi start app
  datasource:
    url: jdbc:postgresql://localhost:5432/batchdb
```

```java
@Configuration
@EnableBatchProcessing  // Kích hoạt Spring Batch
public class BatchConfig {

    @Bean
    public Job importUserJob(JobRepository jobRepository,
                             Step importStep,
                             JobCompletionNotificationListener listener) {
        return new JobBuilder("importUserJob", jobRepository)
            .listener(listener)
            .start(importStep)
            .build();
    }

    @Bean
    public Step importStep(JobRepository jobRepository,
                           PlatformTransactionManager transactionManager,
                           ItemReader<UserCsvRecord> reader,
                           ItemProcessor<UserCsvRecord, User> processor,
                           ItemWriter<User> writer) {
        return new StepBuilder("importStep", jobRepository)
            .<UserCsvRecord, User>chunk(100, transactionManager) // Chunk size = 100
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .build();
    }
}
```

---

## ItemReader — Đọc Dữ Liệu

### FlatFileItemReader — Đọc File CSV/Text

```java
@Bean
@StepScope  // Tạo mới bean cho mỗi Step execution (quan trọng!)
public FlatFileItemReader<UserCsvRecord> csvReader(
        @Value("#{jobParameters['inputFile']}") String inputFile) {

    return new FlatFileItemReaderBuilder<UserCsvRecord>()
        .name("userCsvReader")
        .resource(new FileSystemResource(inputFile))
        .delimited()
        .delimiter(",")
        .names("id", "name", "email", "birthDate")  // Tên cột
        .fieldSetMapper(new BeanWrapperFieldSetMapper<>() {{
            setTargetType(UserCsvRecord.class);
        }})
        .linesToSkip(1)  // Bỏ qua dòng header
        .build();
}

// Model cho CSV
public record UserCsvRecord(String id, String name, String email, String birthDate) {}
```

### JdbcCursorItemReader — Đọc từ Database

```java
@Bean
@StepScope
public JdbcCursorItemReader<OldUser> databaseReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<OldUser>()
        .name("databaseReader")
        .dataSource(dataSource)
        .sql("SELECT id, name, email FROM old_users WHERE migrated = false ORDER BY id")
        .rowMapper(new BeanPropertyRowMapper<>(OldUser.class))
        .fetchSize(100)  // Lấy 100 rows mỗi lần từ DB cursor
        .build();
}
```

### JdbcPagingItemReader — Đọc theo Trang (An Toàn Hơn)

```java
@Bean
@StepScope
public JdbcPagingItemReader<OldUser> pagingReader(DataSource dataSource) {
    Map<String, Order> sortKeys = Map.of("id", Order.ASCENDING);

    return new JdbcPagingItemReaderBuilder<OldUser>()
        .name("pagingReader")
        .dataSource(dataSource)
        .selectClause("SELECT id, name, email")
        .fromClause("FROM old_users")
        .whereClause("WHERE migrated = false")
        .sortKeys(sortKeys)
        .rowMapper(new BeanPropertyRowMapper<>(OldUser.class))
        .pageSize(100)
        .build();
}
```

### Custom ItemReader

```java
@Component
@StepScope
public class ApiItemReader implements ItemReader<ProductDto> {

    private final ProductApiClient apiClient;
    private List<ProductDto> currentBatch;
    private int cursor = 0;
    private int page = 0;

    @Override
    public ProductDto read() throws Exception {
        if (currentBatch == null || cursor >= currentBatch.size()) {
            currentBatch = apiClient.fetchPage(page++, 100);
            cursor = 0;
            if (currentBatch.isEmpty()) {
                return null; // null = hết dữ liệu
            }
        }
        return currentBatch.get(cursor++);
    }
}
```

---

## ItemProcessor — Xử Lý Dữ Liệu

```java
@Component
public class UserProcessor implements ItemProcessor<UserCsvRecord, User> {

    private final PasswordEncoder passwordEncoder;

    @Override
    public User process(UserCsvRecord record) throws Exception {
        // Validate
        if (record.email() == null || !record.email().contains("@")) {
            return null; // null = bỏ qua record này (filtered out)
        }

        // Transform
        return User.builder()
            .name(record.name().trim())
            .email(record.email().toLowerCase())
            .birthDate(LocalDate.parse(record.birthDate()))
            .passwordHash(passwordEncoder.encode("changeme123"))
            .active(true)
            .createdAt(Instant.now())
            .build();
    }
}

// Kết hợp nhiều Processor
@Bean
public CompositeItemProcessor<UserCsvRecord, UserDto> compositeProcessor() {
    return new CompositeItemProcessorBuilder<UserCsvRecord, UserDto>()
        .delegates(
            new ValidateProcessor(),
            new EnrichProcessor(),
            new TransformProcessor()
        )
        .build();
}
```

---

## ItemWriter — Ghi Dữ Liệu

### JpaItemWriter

```java
@Bean
public JpaItemWriter<User> jpaWriter(EntityManagerFactory emf) {
    JpaItemWriter<User> writer = new JpaItemWriter<>();
    writer.setEntityManagerFactory(emf);
    return writer;
}
```

### JdbcBatchItemWriter — Hiệu Năng Cao Hơn JPA

```java
@Bean
public JdbcBatchItemWriter<User> jdbcWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<User>()
        .dataSource(dataSource)
        .sql("""
            INSERT INTO users (name, email, password_hash, created_at)
            VALUES (:name, :email, :passwordHash, :createdAt)
            ON CONFLICT (email) DO UPDATE
            SET name = EXCLUDED.name
            """)
        .beanMapped()  // Map từ bean properties
        .build();
}
```

### Custom ItemWriter

```java
@Component
public class EmailNotificationWriter implements ItemWriter<User> {

    private final EmailService emailService;

    @Override
    public void write(Chunk<? extends User> chunk) throws Exception {
        List<String> emails = chunk.getItems().stream()
            .map(User::getEmail)
            .toList();

        // Gửi batch email — không gửi từng cái một
        emailService.sendWelcomeBatch(emails);
        log.info("Đã gửi {} welcome emails", emails.size());
    }
}

// CompositeItemWriter — ghi vào nhiều nơi
@Bean
public CompositeItemWriter<User> compositeWriter() {
    return new CompositeItemWriterBuilder<User>()
        .delegates(jpaWriter(), auditLogWriter(), cacheInvalidationWriter())
        .build();
}
```

---

## Chunk-Oriented Processing

**Chunk-oriented processing** (Xử Lý Hướng Khối) là pattern cốt lõi — đọc N items, xử lý từng item, ghi tất cả N items trong 1 transaction.

```
CHUNK PROCESSING FLOW:

Chunk size = 3

  read()  read()  read()   ──► [item1, item2, item3]
    ▼       ▼       ▼
 process() process() process() ──► [proc1, proc2, proc3]
                                              ▼
                                    BEGIN TRANSACTION
                                         write([proc1, proc2, proc3])
                                    COMMIT TRANSACTION
                                              ▼
                                    read()  read()  read() ...

Nếu write() lỗi: ROLLBACK toàn bộ chunk
Nếu process() trả null: item đó bị bỏ qua
```

```java
@Bean
public Step processStep(JobRepository jobRepository,
                        PlatformTransactionManager txManager) {
    return new StepBuilder("processStep", jobRepository)
        .<InputDto, OutputDto>chunk(500, txManager) // Ghi mỗi 500 records
        .reader(reader())
        .processor(processor())
        .writer(writer())
        // Listener để log tiến độ
        .listener(new ChunkListener() {
            @Override
            public void afterChunk(ChunkContext context) {
                long count = context.getStepContext()
                    .getStepExecution().getWriteCount();
                log.info("Đã xử lý {} records", count);
            }
        })
        .build();
}
```

---

## Tasklet — Tác Vụ Đơn Giản

**Tasklet** phù hợp cho các bước không phải chunk — xóa file, gửi notification, chạy stored procedure.

```java
@Component
public class CleanupTasklet implements Tasklet {

    @Override
    public RepeatStatus execute(StepContribution contribution,
                                ChunkContext chunkContext) throws Exception {
        // Lấy jobParameters
        String outputDir = chunkContext.getStepContext()
            .getJobParameters().get("outputDir").toString();

        // Thực hiện công việc
        Files.deleteIfExists(Path.of(outputDir + "/temp.csv"));
        log.info("Đã xóa file tạm thời");

        return RepeatStatus.FINISHED; // CONTINUABLE = chạy lại Tasklet này
    }
}

@Bean
public Step cleanupStep(JobRepository jobRepository,
                        PlatformTransactionManager txManager,
                        CleanupTasklet cleanupTasklet) {
    return new StepBuilder("cleanupStep", jobRepository)
        .tasklet(cleanupTasklet, txManager)
        .build();
}
```

---

## Job Flow — Điều Khiển Luồng Job

```java
@Bean
public Job etlJob(JobRepository jobRepository,
                  Step extractStep, Step transformStep,
                  Step loadStep, Step cleanupStep,
                  Step errorNotifyStep) {
    return new JobBuilder("etlJob", jobRepository)
        .start(extractStep)
            // Nếu extractStep SUCCESS → transformStep
            .on("COMPLETED").to(transformStep)
            // Nếu extractStep FAILED → errorNotifyStep → kết thúc với FAILED
            .on("FAILED").to(errorNotifyStep)
        .from(transformStep)
            .on("COMPLETED").to(loadStep)
            .on("FAILED").to(errorNotifyStep)
        .from(loadStep)
            .on("COMPLETED").to(cleanupStep)
            .on("FAILED").fail()            // Dừng Job với trạng thái FAILED
        .from(errorNotifyStep)
            .on("*").end("STOPPED")         // Kết thúc với custom ExitStatus
        .end()
        .build();
}
```

### Step Parallelism — Chạy Song Song

```java
@Bean
public Job parallelJob(JobRepository jobRepository) {
    // Split — chạy các flow song song
    Flow flow1 = new FlowBuilder<Flow>("flow1")
        .start(processUsersStep())
        .build();

    Flow flow2 = new FlowBuilder<Flow>("flow2")
        .start(processProductsStep())
        .build();

    return new JobBuilder("parallelJob", jobRepository)
        .start(flow1)
        .split(new SimpleAsyncTaskExecutor()) // Thread pool
        .add(flow2)
        .end()
        .next(aggregateStep())  // Chạy sau khi cả 2 flows hoàn thành
        .build();
}
```

---

## Skip & Retry — Chịu Lỗi

```java
@Bean
public Step robustStep(JobRepository jobRepository,
                       PlatformTransactionManager txManager) {
    return new StepBuilder("robustStep", jobRepository)
        .<InputDto, OutputDto>chunk(100, txManager)
        .reader(reader())
        .processor(processor())
        .writer(writer())

        // Skip — bỏ qua lỗi, tiếp tục xử lý
        .faultTolerant()
        .skip(ValidationException.class)    // Bỏ qua lỗi validation
        .skip(ParseException.class)
        .skipLimit(100)                     // Tối đa 100 lần skip (vượt qua → Step FAILED)
        .noSkip(DatabaseException.class)    // Không bỏ qua lỗi DB

        // Retry — thử lại trước khi skip
        .retry(TransientDataAccessException.class)  // Lỗi tạm thời (network glitch)
        .retryLimit(3)                              // Thử lại tối đa 3 lần
        .noRetry(ValidationException.class)         // Không retry lỗi validation

        // Listener để log skip events
        .listener(new SkipListener<InputDto, OutputDto>() {
            @Override
            public void onSkipInRead(Throwable t) {
                log.warn("Bỏ qua khi đọc: {}", t.getMessage());
            }
            @Override
            public void onSkipInProcess(InputDto item, Throwable t) {
                log.warn("Bỏ qua khi xử lý item {}: {}", item, t.getMessage());
            }
            @Override
            public void onSkipInWrite(OutputDto item, Throwable t) {
                log.warn("Bỏ qua khi ghi item {}: {}", item, t.getMessage());
            }
        })
        .build();
}
```

---

## Partitioning — Xử Lý Song Song

**Partitioning** (Phân Vùng) chia dữ liệu thành nhiều partitions (phân vùng) và xử lý song song — cực kỳ hiệu quả cho dữ liệu lớn.

```
PARTITIONING:

Manager Step (Điều Phối)
    │
    ├── Partition 1: ID 1 – 10,000     → Worker Step → Thread 1
    ├── Partition 2: ID 10,001 – 20,000 → Worker Step → Thread 2
    ├── Partition 3: ID 20,001 – 30,000 → Worker Step → Thread 3
    └── Partition 4: ID 30,001 – 40,000 → Worker Step → Thread 4
```

```java
// Partitioner — quyết định cách chia
@Component
public class RangePartitioner implements Partitioner {

    private final JdbcTemplate jdbcTemplate;

    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        Long minId = jdbcTemplate.queryForObject(
            "SELECT MIN(id) FROM users WHERE migrated = false", Long.class);
        Long maxId = jdbcTemplate.queryForObject(
            "SELECT MAX(id) FROM users WHERE migrated = false", Long.class);

        long range = (maxId - minId) / gridSize + 1;
        Map<String, ExecutionContext> result = new HashMap<>();

        for (int i = 0; i < gridSize; i++) {
            ExecutionContext context = new ExecutionContext();
            long start = minId + i * range;
            long end = Math.min(start + range - 1, maxId);
            context.putLong("minId", start);
            context.putLong("maxId", end);
            result.put("partition-" + i, context);
        }
        return result;
    }
}

// Worker Step — một partition
@Bean
@StepScope
public JdbcPagingItemReader<User> partitionReader(
        @Value("#{stepExecutionContext['minId']}") Long minId,
        @Value("#{stepExecutionContext['maxId']}") Long maxId,
        DataSource dataSource) {

    return new JdbcPagingItemReaderBuilder<User>()
        .name("partitionReader")
        .dataSource(dataSource)
        .selectClause("SELECT *")
        .fromClause("FROM users")
        .whereClause("WHERE id BETWEEN " + minId + " AND " + maxId)
        .sortKeys(Map.of("id", Order.ASCENDING))
        .rowMapper(new BeanPropertyRowMapper<>(User.class))
        .pageSize(500)
        .build();
}

// Manager Step — điều phối partitions
@Bean
public Step managerStep(JobRepository jobRepository,
                        Step workerStep,
                        RangePartitioner partitioner) {
    return new StepBuilder("managerStep", jobRepository)
        .partitioner("workerStep", partitioner)
        .step(workerStep)
        .gridSize(8)  // 8 partitions chạy song song
        .taskExecutor(new SimpleAsyncTaskExecutor())
        .build();
}
```

---

## JobParameters & JobLauncher

```java
// Khởi động Job từ REST API
@RestController
@RequestMapping("/api/batch")
public class BatchController {

    private final JobLauncher jobLauncher;
    private final Job importUserJob;

    @PostMapping("/import")
    public ResponseEntity<String> startImport(
            @RequestParam String inputFile) throws Exception {

        JobParameters params = new JobParametersBuilder()
            .addString("inputFile", inputFile)
            .addLong("timestamp", System.currentTimeMillis()) // Ensure unique run
            .toJobParameters();

        JobExecution execution = jobLauncher.run(importUserJob, params);
        return ResponseEntity.ok("Job ID: " + execution.getId()
            + " | Status: " + execution.getStatus());
    }

    @GetMapping("/status/{jobId}")
    public JobExecutionDto getStatus(@PathVariable Long jobId) {
        JobExecution execution = jobExplorer.getJobExecution(jobId);
        return JobExecutionDto.from(execution);
    }
}

// Async JobLauncher — không block HTTP thread
@Bean
public JobLauncher asyncJobLauncher(JobRepository jobRepository) {
    TaskExecutorJobLauncher launcher = new TaskExecutorJobLauncher();
    launcher.setJobRepository(jobRepository);
    launcher.setTaskExecutor(new SimpleAsyncTaskExecutor());
    return launcher;
}
```

---

## Monitoring & Actuator

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,batch
```

```java
// JobExecutionListener — log khi Job hoàn thành
@Component
public class JobCompletionNotificationListener
        implements JobExecutionListener {

    @Override
    public void afterJob(JobExecution jobExecution) {
        if (jobExecution.getStatus() == BatchStatus.COMPLETED) {
            log.info("Job '{}' hoàn thành. Đọc: {}, Ghi: {}, Bỏ qua: {}",
                jobExecution.getJobInstance().getJobName(),
                jobExecution.getStepExecutions().stream()
                    .mapToLong(StepExecution::getReadCount).sum(),
                jobExecution.getStepExecutions().stream()
                    .mapToLong(StepExecution::getWriteCount).sum(),
                jobExecution.getStepExecutions().stream()
                    .mapToLong(StepExecution::getSkipCount).sum()
            );
        } else if (jobExecution.getStatus() == BatchStatus.FAILED) {
            log.error("Job '{}' THẤT BẠI. Lỗi: {}",
                jobExecution.getJobInstance().getJobName(),
                jobExecution.getAllFailureExceptions()
            );
            // Gửi alert
            alertService.sendBatchFailureAlert(jobExecution);
        }
    }
}
```

---

## Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa Chunk-oriented Step và Tasklet Step?**

A: Chunk-oriented Step đọc-xử lý-ghi theo từng chunk trong transaction — phù hợp khi xử lý tập dữ liệu lớn với I/O. Tasklet Step là unit of work đơn giản thực thi trong 1 transaction — phù hợp cho các bước không lặp lại như gửi notification, xóa file, gọi stored procedure.

---

**Q: JobInstance và JobExecution khác nhau như thế nào?**

A: JobInstance là khái niệm logic — một lần chạy Job với bộ JobParameters cụ thể (ví dụ: job "import" với `date=2026-06-01`). JobExecution là một lần thực thi vật lý — nếu JobInstance fail và restart, sẽ có JobExecution mới nhưng cùng JobInstance.

---

**Q: Tại sao cần `@StepScope` cho ItemReader?**

A: `@StepScope` tạo bean mới cho mỗi Step execution thay vì singleton. Quan trọng khi cần inject `#{jobParameters['...']}` hoặc `#{stepExecutionContext['...']}` — các giá trị này chỉ có trong context của một Step execution cụ thể. Không có `@StepScope`, Spring không thể inject late-binding expressions.

---

**Q: Partitioning trong Spring Batch giải quyết vấn đề gì?**

A: Partitioning chia dataset thành nhiều phần nhỏ và xử lý song song trên nhiều threads/nodes, giảm thời gian xử lý tuyến tính. Ví dụ: 10 triệu records trong 1 thread mất 2 giờ, chia 8 partitions → ~15 phút. Manager Step điều phối, Worker Steps thực thi độc lập.

---

**Q: Cơ chế Restart của Spring Batch hoạt động thế nào?**

A: Spring Batch lưu trạng thái trong JobRepository (bảng metadata). Nếu Job fail, restart với cùng JobParameters sẽ tiếp tục từ Step cuối cùng chưa hoàn thành — không chạy lại các Step đã COMPLETED. Với Chunk-oriented step, restart từ chunk tiếp theo chưa được commit (nhờ `saveState = true` và `ExecutionContext`).

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
