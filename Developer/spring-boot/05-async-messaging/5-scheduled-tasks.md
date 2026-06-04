# Scheduled Tasks — @Scheduled, Quartz & Cron Expressions

> Lập lịch tác vụ định kỳ (scheduled tasks) trong Spring Boot —
> từ `@Scheduled` đơn giản, biểu thức cron (cron expressions),
> đến Quartz Scheduler (Trình Lập Lịch Quartz) mạnh mẽ cho môi trường clustered (nhiều instance)
> và ShedLock (Khóa Lịch Biểu) để tránh chạy trùng lặp.

---

## 1. Tại Sao Cần Scheduled Tasks?

### Use Cases (Tình Huống Sử Dụng) Phổ Biến

```
Gửi email digest hàng ngày lúc 8:00 sáng
Tính toán báo cáo doanh thu cuối tháng
Dọn dẹp temporary files mỗi đêm
Đồng bộ dữ liệu với external API mỗi 5 phút
Kiểm tra và hủy đơn hàng pending quá 30 phút
Gửi reminder cho user chưa hoàn tất profile
Xóa expired tokens/sessions định kỳ
```

---

## 2. @Scheduled — Cơ Bản

### 2.1 Kích Hoạt Scheduling

```java
@SpringBootApplication
@EnableScheduling  // ← Bắt buộc — kích hoạt cơ chế @Scheduled
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// Hoặc trong @Configuration riêng
@Configuration
@EnableScheduling
public class SchedulingConfig { }
```

### 2.2 Các Loại Lịch Biểu

```java
@Component
public class ScheduledTasks {

    private static final Logger log = LoggerFactory.getLogger(ScheduledTasks.class);

    // ── fixedRate: chạy mỗi N ms tính từ lúc bắt đầu lần trước
    // Không quan tâm task mất bao lâu — có thể overlap nếu chậm
    @Scheduled(fixedRate = 5000)
    public void runEvery5Seconds() {
        log.info("Fixed rate task: {}", LocalDateTime.now());
    }

    // ── fixedDelay: chờ N ms SAU KHI task trước hoàn thành rồi mới chạy lại
    // Đảm bảo không overlap — phù hợp cho tasks có side effects
    @Scheduled(fixedDelay = 3000)
    public void runWithDelay() {
        log.info("Fixed delay task — starts 3s after previous finishes");
        processQueue(); // Task xử lý tuần tự, an toàn
    }

    // ── initialDelay: chờ N ms sau khi app start mới bắt đầu schedule
    @Scheduled(fixedRate = 60000, initialDelay = 30000)
    public void runAfterStartup() {
        log.info("Starts 30s after app startup, then every 60s");
    }

    // ── cron: lịch biểu linh hoạt theo cú pháp cron
    @Scheduled(cron = "0 0 8 * * MON-FRI")   // 8:00 sáng, thứ 2-6
    public void sendDailyReport() {
        reportService.generateAndSend();
    }

    // ── Đọc từ application.properties (Linh Hoạt Hơn)
    @Scheduled(cron = "${app.schedule.cleanup-cron:0 0 2 * * *}")
    public void cleanupExpiredData() {
        cleanupService.deleteExpired();
    }
}
```

---

## 3. Cron Expressions (Biểu Thức Cron)

### Cấu Trúc

```
┌────────────── giây (0-59)
│ ┌──────────── phút (0-59)
│ │ ┌────────── giờ (0-23)
│ │ │ ┌──────── ngày trong tháng (1-31)
│ │ │ │ ┌────── tháng (1-12 hoặc JAN-DEC)
│ │ │ │ │ ┌──── ngày trong tuần (0-7 hoặc SUN-SAT, 0 và 7 đều là Chủ Nhật)
│ │ │ │ │ │
* * * * * *
```

### Ký Tự Đặc Biệt

| Ký Tự | Ý Nghĩa | Ví Dụ |
|-------|---------|-------|
| `*` | Mọi giá trị | `* * * * * *` = mỗi giây |
| `,` | Danh sách | `0 0 8,12,18 * * *` = 8h, 12h, 18h |
| `-` | Khoảng | `0 0 9-17 * * *` = mỗi giờ từ 9h đến 17h |
| `/` | Bước nhảy | `0 */15 * * * *` = mỗi 15 phút |
| `?` | Không quan tâm | Dùng cho ngày/tuần khi đã chỉ định cái kia |
| `L` | Cuối | `0 0 0 L * *` = ngày cuối tháng |
| `#` | N-th | `0 0 10 * * 2#1` = thứ 2 đầu tiên trong tháng |

### Ví Dụ Thực Tế

```
# Mỗi phút
0 * * * * *

# Mỗi 5 phút
0 */5 * * * *

# Mỗi 30 phút
0 0,30 * * * *

# Hàng ngày lúc 8:00 sáng
0 0 8 * * *

# Thứ 2 đến thứ 6, lúc 9:00 sáng
0 0 9 * * MON-FRI

# Đầu mỗi giờ (giờ làm việc)
0 0 9-17 * * MON-FRI

# Ngày đầu tiên mỗi tháng lúc 0:00
0 0 0 1 * *

# Cuối tháng lúc 23:59
0 59 23 L * *

# Thứ 2 đầu tiên của tháng lúc 10:00
0 0 10 * * MON#1

# Mỗi ngày lúc 2:30 sáng — dọn dẹp database
0 30 2 * * *

# Mỗi 10 phút trong giờ làm việc
0 */10 8-17 * * MON-FRI
```

### Công Cụ Tạo Cron

> Dùng [crontab.guru](https://crontab.guru/) hoặc [cronhub.io](https://crontab.io/) để validate cron expression.

### Timezone (Múi Giờ)

```java
// Mặc định dùng timezone của server — nguy hiểm khi deploy đa vùng!
@Scheduled(cron = "0 0 8 * * *", zone = "Asia/Ho_Chi_Minh")
public void sendVietnamReport() {
    reportService.send();
}
```

---

## 4. Thread Pool Cho Scheduled Tasks

### Vấn Đề Mặc Định

Mặc định Spring dùng **1 single thread** cho tất cả `@Scheduled` tasks. Nếu một task chậm, các task khác phải chờ!

```java
// ❌ Nếu task A mất 10 phút, task B phải chờ
@Scheduled(fixedRate = 1000)  // Task A — chạy mỗi giây
public void taskA() { Thread.sleep(600000); }  // Mất 10 phút!

@Scheduled(fixedRate = 5000)  // Task B — sẽ bị block bởi Task A
public void taskB() { ... }
```

### Giải Pháp — Cấu Hình Thread Pool

```java
@Configuration
@EnableScheduling
public class SchedulingConfig implements SchedulingConfigurer {

    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();

        // Số thread cho scheduled tasks
        scheduler.setPoolSize(10);
        scheduler.setThreadNamePrefix("scheduled-task-");
        scheduler.setWaitForTasksToCompleteOnShutdown(true);
        scheduler.setAwaitTerminationSeconds(30);
        scheduler.initialize();

        registrar.setTaskScheduler(scheduler);
    }
}
```

```properties
# Hoặc qua application.properties
spring.task.scheduling.pool.size=10
spring.task.scheduling.thread-name-prefix=scheduled-
spring.task.scheduling.shutdown.await-termination=true
spring.task.scheduling.shutdown.await-termination-period=30s
```

---

## 5. Dynamic Scheduling (Lịch Biểu Động)

### Thay Đổi Lịch Biểu Lúc Runtime

```java
@Component
public class DynamicScheduler {

    private ScheduledFuture<?> scheduledFuture;
    private final ThreadPoolTaskScheduler taskScheduler;
    private final ReportService reportService;

    public DynamicScheduler(ThreadPoolTaskScheduler taskScheduler,
                            ReportService reportService) {
        this.taskScheduler = taskScheduler;
        this.reportService = reportService;
    }

    // Lập lịch với cron expression động
    public void scheduleTask(String cronExpression) {
        // Hủy task cũ nếu đang chạy
        if (scheduledFuture != null) {
            scheduledFuture.cancel(false); // false = không interrupt nếu đang chạy
        }

        scheduledFuture = taskScheduler.schedule(
            () -> reportService.generateDailyReport(),
            new CronTrigger(cronExpression)
        );

        log.info("Task rescheduled with cron: {}", cronExpression);
    }

    // Dừng task
    public void stopTask() {
        if (scheduledFuture != null && !scheduledFuture.isCancelled()) {
            scheduledFuture.cancel(false);
        }
    }
}

// REST API để thay đổi lịch biểu lúc runtime
@RestController
@RequestMapping("/admin/scheduler")
public class SchedulerController {

    @PutMapping("/reschedule")
    public void reschedule(@RequestParam String cron) {
        dynamicScheduler.scheduleTask(cron);
    }

    @DeleteMapping("/stop")
    public void stop() {
        dynamicScheduler.stopTask();
    }
}
```

---

## 6. Quartz Scheduler — Cho Môi Trường Production

### Khi Nào Dùng Quartz?

```
Dùng @Scheduled khi:
├── Task đơn giản
├── Chạy 1 instance app
└── Không cần job history

Dùng Quartz khi:
├── Nhiều instance app (clustered)
├── Cần chạy job chỉ 1 lần dù nhiều instance
├── Cần job history, dashboard
├── Cần trigger phức tạp (calendar triggers)
└── Cần persistent jobs (tồn tại sau restart)
```

### 6.1 Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

### 6.2 application.properties — Quartz Clustered

```properties
# Job store type: memory hoặc jdbc (cho clustered)
spring.quartz.job-store-type=jdbc

# Auto tạo Quartz tables trong DB
spring.quartz.jdbc.initialize-schema=always

# Clustered mode — nhiều instance chia sẻ jobs qua DB
spring.quartz.properties.org.quartz.jobStore.isClustered=true
spring.quartz.properties.org.quartz.jobStore.clusterCheckinInterval=10000

# Instance ID tự động — mỗi instance có ID riêng
spring.quartz.properties.org.quartz.scheduler.instanceId=AUTO
spring.quartz.properties.org.quartz.scheduler.instanceName=MyClusteredScheduler

# Thread pool
spring.quartz.properties.org.quartz.threadPool.threadCount=10
```

### 6.3 Tạo Quartz Job

```java
// Job class — phải stateless, Spring inject qua JobFactory
@Component
public class ReportGenerationJob implements Job {

    // Spring tự inject — nhờ SpringBeanJobFactory
    @Autowired
    private ReportService reportService;

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        // Lấy data từ JobDataMap
        JobDataMap dataMap = context.getMergedJobDataMap();
        String reportType = dataMap.getString("reportType");
        Long userId = dataMap.getLong("userId");

        log.info("Executing report job: type={}, userId={}", reportType, userId);

        try {
            reportService.generate(reportType, userId);
        } catch (Exception ex) {
            throw new JobExecutionException(ex, true); // true = refireImmediately
        }
    }
}
```

### 6.4 Đăng Ký Job và Trigger

```java
@Configuration
public class QuartzConfig {

    // JobDetail — định nghĩa job
    @Bean
    public JobDetail reportJobDetail() {
        return JobBuilder.newJob(ReportGenerationJob.class)
            .withIdentity("reportJob", "reports")     // name, group
            .withDescription("Generate monthly report")
            .usingJobData("reportType", "MONTHLY")
            .storeDurably()   // Giữ job kể cả không có trigger
            .build();
    }

    // Trigger — khi nào chạy job
    @Bean
    public Trigger reportTrigger(JobDetail reportJobDetail) {
        return TriggerBuilder.newTrigger()
            .forJob(reportJobDetail)
            .withIdentity("reportTrigger", "reports")
            .withDescription("Run monthly at 1st day, 2:00 AM")
            .withSchedule(CronScheduleBuilder
                .cronSchedule("0 0 2 1 * ?")          // Ngày 1 mỗi tháng, 2:00 AM
                .withMisfireHandlingInstructionDoNothing()) // Bỏ qua nếu miss
            .build();
    }

    // SimpleTrigger — chạy ngay sau N giây, lặp M lần
    @Bean
    public Trigger immediateCleanupTrigger(JobDetail cleanupJobDetail) {
        return TriggerBuilder.newTrigger()
            .forJob(cleanupJobDetail)
            .withIdentity("cleanupTrigger")
            .startAt(DateBuilder.futureDate(30, DateBuilder.IntervalUnit.SECOND))
            .withSchedule(SimpleScheduleBuilder
                .simpleSchedule()
                .withIntervalInHours(24)
                .repeatForever())
            .build();
    }
}
```

### 6.5 Quản Lý Job Lúc Runtime

```java
@Service
public class JobManagementService {

    private final Scheduler scheduler;

    public void scheduleJob(String userId, String reportType, String cronExpr) throws SchedulerException {
        JobKey jobKey = new JobKey("report-" + userId, "user-reports");

        JobDetail job = JobBuilder.newJob(ReportGenerationJob.class)
            .withIdentity(jobKey)
            .usingJobData("userId", userId)
            .usingJobData("reportType", reportType)
            .build();

        Trigger trigger = TriggerBuilder.newTrigger()
            .withIdentity("trigger-" + userId, "user-reports")
            .withSchedule(CronScheduleBuilder.cronSchedule(cronExpr))
            .build();

        scheduler.scheduleJob(job, trigger);
    }

    public void pauseJob(String userId) throws SchedulerException {
        scheduler.pauseJob(new JobKey("report-" + userId, "user-reports"));
    }

    public void resumeJob(String userId) throws SchedulerException {
        scheduler.resumeJob(new JobKey("report-" + userId, "user-reports"));
    }

    public void deleteJob(String userId) throws SchedulerException {
        scheduler.deleteJob(new JobKey("report-" + userId, "user-reports"));
    }

    public void triggerJobNow(String userId) throws SchedulerException {
        scheduler.triggerJob(new JobKey("report-" + userId, "user-reports"));
    }
}
```

---

## 7. ShedLock — Distributed Lock Cho @Scheduled

### Vấn Đề Khi Nhiều Instance

```
Instance 1: @Scheduled → sendDailyEmail()  ─────► User nhận 2 email!
Instance 2: @Scheduled → sendDailyEmail()  ─────►
```

### Giải Pháp Với ShedLock

ShedLock (Khóa Lịch Biểu Phân Tán) đảm bảo chỉ 1 instance chạy task tại một thời điểm, dùng DB hoặc Redis làm lock store.

```xml
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
    <version>5.10.0</version>
</dependency>
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-jdbc-template</artifactId>
    <version>5.10.0</version>
</dependency>
```

```sql
-- Tạo bảng lock trong DB
CREATE TABLE shedlock (
    name       VARCHAR(64)  NOT NULL,
    lock_until TIMESTAMP    NOT NULL,
    locked_at  TIMESTAMP    NOT NULL,
    locked_by  VARCHAR(255) NOT NULL,
    PRIMARY KEY (name)
);
```

```java
@Configuration
@EnableScheduling
@EnableSchedulerLock(defaultLockAtMostFor = "PT30M")  // Lock tối đa 30 phút
public class SchedulingConfig {

    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(
            JdbcTemplateLockProvider.Configuration.builder()
                .withJdbcTemplate(new JdbcTemplate(dataSource))
                .usingDbTime()  // Dùng DB time — nhất quán hơn
                .build()
        );
    }
}

@Component
public class DistributedScheduledTasks {

    @Scheduled(cron = "0 0 8 * * *")
    @SchedulerLock(
        name = "sendDailyEmail",          // Tên lock — unique trong cluster
        lockAtMostFor = "PT10M",          // Lock tối đa 10 phút (phòng crash)
        lockAtLeastFor = "PT5M"           // Lock tối thiểu 5 phút (tránh 2 instance chạy liên tiếp)
    )
    public void sendDailyEmail() {
        // Chỉ 1 instance trong cluster chạy method này
        emailService.sendDailyDigest();
    }
}
```

---

## 8. Xử Lý Lỗi Và Monitoring

### Exception Handling Trong @Scheduled

```java
@Component
public class SafeScheduledTasks {

    @Scheduled(cron = "0 0 3 * * *")
    public void cleanupTask() {
        try {
            cleanupService.deleteExpiredData();
            log.info("Cleanup completed successfully");
        } catch (Exception ex) {
            // @Scheduled KHÔNG propagate exception — task tiếp tục schedule
            // Log và alert là bắt buộc!
            log.error("Cleanup task failed", ex);
            alertService.sendAlert("Scheduled cleanup failed", ex.getMessage());
        }
    }
}
```

> **Quan trọng:** `@Scheduled` bắt tất cả exception — task không crash app nhưng cũng không retry tự động. Phải tự handle retry nếu cần.

### Monitoring Scheduled Tasks

```java
@Component
public class MonitoredScheduledTask {

    private final MeterRegistry meterRegistry;
    private final Counter successCounter;
    private final Counter failureCounter;
    private final Timer executionTimer;

    public MonitoredScheduledTask(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.successCounter = Counter.builder("scheduled.task.success")
            .tag("name", "cleanup").register(meterRegistry);
        this.failureCounter = Counter.builder("scheduled.task.failure")
            .tag("name", "cleanup").register(meterRegistry);
        this.executionTimer = Timer.builder("scheduled.task.duration")
            .tag("name", "cleanup").register(meterRegistry);
    }

    @Scheduled(cron = "0 0 2 * * *")
    public void cleanupWithMetrics() {
        Timer.Sample sample = Timer.start(meterRegistry);
        try {
            cleanupService.run();
            successCounter.increment();
        } catch (Exception ex) {
            failureCounter.increment();
            log.error("Cleanup failed", ex);
        } finally {
            sample.stop(executionTimer);
        }
    }
}
```

---

## 9. Testing Scheduled Tasks

```java
@SpringBootTest
class ScheduledTasksTest {

    @Autowired
    private CleanupService cleanupService;

    @MockBean
    private UserRepository userRepository;

    // Test logic của task — không test scheduling
    @Test
    void shouldDeleteExpiredUsers() {
        // Given
        when(userRepository.findExpiredBefore(any())).thenReturn(List.of(expiredUser));

        // When — gọi trực tiếp method, không chờ schedule
        cleanupService.deleteExpiredUsers();

        // Then
        verify(userRepository).deleteAll(anyList());
    }
}

// Test trigger bằng awaitility
@SpringBootTest
class ScheduledTriggerTest {

    @Autowired
    private ReportService reportService;

    @Test
    void shouldRunEvery5Seconds() {
        // Dùng Awaitility để chờ task chạy
        await().atMost(10, SECONDS)
               .untilAsserted(() -> verify(reportService, atLeast(1)).generateReport());
    }
}
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: `fixedRate` vs `fixedDelay` — sự khác biệt quan trọng?**
> `fixedRate`: bắt đầu task mỗi N ms tính từ lần bắt đầu trước → có thể chạy overlap nếu task chậm hơn interval. `fixedDelay`: chờ N ms sau khi task **hoàn thành** → đảm bảo không overlap, phù hợp cho tasks có side effects hoặc cần xử lý tuần tự.

**Q: Cách đảm bảo @Scheduled chỉ chạy 1 lần trong môi trường nhiều instance (cluster)?**
> Dùng **ShedLock** với DB hoặc Redis làm distributed lock — đảm bảo chỉ 1 instance trong cluster giành được lock và thực thi task. Hoặc dùng **Quartz Scheduler** với `isClustered=true` và JDBC job store — Quartz tự quản lý clustering.

**Q: Khi nào dùng Quartz thay vì @Scheduled?**
> Quartz phù hợp khi: (1) Cần clustered scheduling — nhiều instance chỉ chạy 1 lần, (2) Cần persistent jobs — jobs tồn tại sau restart, (3) Cần quản lý jobs lúc runtime — thêm/sửa/xóa qua API, (4) Cần job history và monitoring dashboard, (5) Cần complex triggers — calendar, interval với giới hạn.

**Q: @Scheduled có retry tự động không khi task fail?**
> Không. `@Scheduled` bắt tất cả exception và task tiếp tục schedule bình thường — không có retry. Phải tự implement retry (vd: Spring Retry `@Retryable`) hoặc dùng message queue với retry mechanism.
