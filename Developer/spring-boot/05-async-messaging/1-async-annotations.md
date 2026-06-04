# @Async, CompletableFuture & ThreadPoolTaskExecutor

> Hiểu sâu về lập trình bất đồng bộ (asynchronous programming) trong Spring Boot —
> từ annotation `@Async` đơn giản đến `CompletableFuture` (Tương Lai Hoàn Thành) phức tạp,
> cấu hình `ThreadPoolTaskExecutor` (Trình Thực Thi Hồ Luồng) và xử lý lỗi trong môi trường bất đồng bộ.

---

## 1. Tại Sao Cần Lập Trình Bất Đồng Bộ?

### Vấn Đề Với Synchronous (Đồng Bộ)

```
Client ──► /api/orders ──► OrderService
                                │
                         Lưu DB (10ms)
                                │
                         Gửi email (2000ms)  ← Chặn luồng HTTP!
                                │
                         Cập nhật inventory (50ms)
                                │
                         ◄── Trả về response (tổng ~2060ms)
```

HTTP thread bị chặn 2 giây chỉ vì gửi email — lãng phí tài nguyên, giảm throughput (thông lượng).

### Giải Pháp Với @Async

```
Client ──► /api/orders ──► OrderService
                                │
                         Lưu DB (10ms)
                                │
                         ──► @Async: gửi email (background thread)
                                │
                         Cập nhật inventory (50ms)
                                │
                         ◄── Trả về response (~60ms) ✅
```

---

## 2. Kích Hoạt @Async

### Bước 1: Thêm @EnableAsync

```java
@Configuration
@EnableAsync  // ← Bắt buộc — kích hoạt cơ chế @Async
public class AsyncConfig {
}
```

> **Quan trọng:** Không có `@EnableAsync`, annotation `@Async` bị bỏ qua hoàn toàn — method vẫn chạy đồng bộ mà không có cảnh báo nào!

### Bước 2: Annotate Method

```java
@Service
public class NotificationService {

    @Async  // ← Method này sẽ chạy trong thread pool riêng
    public void sendWelcomeEmail(String email) {
        // Giả lập gửi email mất 2 giây
        Thread.sleep(2000);
        System.out.println("Email sent to: " + email
                + " | Thread: " + Thread.currentThread().getName());
    }
}
```

### Bước 3: Gọi Từ Bean Khác

```java
@Service
public class UserService {

    private final NotificationService notificationService;

    public UserService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    public User registerUser(RegisterRequest request) {
        User user = userRepository.save(new User(request));

        notificationService.sendWelcomeEmail(user.getEmail()); // Không chặn!

        return user; // Trả về ngay lập tức
    }
}
```

---

## 3. Bẫy Self-Invocation (Gọi Nội Bộ)

### ❌ Anti-Pattern — @Async Không Hoạt Động

```java
@Service
public class OrderService {

    public void processOrder(Order order) {
        saveOrder(order);
        sendConfirmation(order); // @Async KHÔNG hoạt động!
    }

    @Async
    public void sendConfirmation(Order order) {
        // Vẫn chạy đồng bộ vì được gọi từ chính class này
    }
}
```

**Tại sao?** Spring `@Async` hoạt động thông qua AOP proxy (Proxy AOP — Đại Diện AOP). Khi gọi `this.sendConfirmation()`, bạn bypass proxy, gọi thẳng method — không có async nào cả.

### ✅ Cách Sửa 1 — Tách Thành Class Riêng

```java
@Service
public class OrderService {

    private final OrderNotificationService notificationService;

    public void processOrder(Order order) {
        saveOrder(order);
        notificationService.sendConfirmation(order); // ✅ Qua proxy
    }
}

@Service
public class OrderNotificationService {

    @Async
    public void sendConfirmation(Order order) {
        // Chạy bất đồng bộ thực sự
    }
}
```

### ✅ Cách Sửa 2 — Self-Inject

```java
@Service
public class OrderService {

    @Lazy
    @Autowired
    private OrderService self; // Inject proxy của chính mình

    public void processOrder(Order order) {
        saveOrder(order);
        self.sendConfirmation(order); // ✅ Qua proxy
    }

    @Async
    public void sendConfirmation(Order order) {
        // Chạy bất đồng bộ
    }
}
```

---

## 4. CompletableFuture — Nhận Kết Quả Từ @Async

### 4.1 Return CompletableFuture

```java
@Service
public class ReportService {

    @Async
    public CompletableFuture<String> generateReport(Long userId) {
        // Tác vụ tốn thời gian
        String report = buildReport(userId);
        return CompletableFuture.completedFuture(report);
    }
}
```

### 4.2 Chạy Song Song (Parallel Execution)

```java
@Service
public class DashboardService {

    private final ReportService reportService;
    private final AnalyticsService analyticsService;
    private final UserService userService;

    public DashboardData buildDashboard(Long userId) throws Exception {
        // Khởi động 3 tác vụ cùng lúc
        CompletableFuture<String> reportFuture   = reportService.generateReport(userId);
        CompletableFuture<List<Event>> eventsFuture = analyticsService.getEvents(userId);
        CompletableFuture<User> userFuture        = userService.getUser(userId);

        // Chờ tất cả hoàn thành
        CompletableFuture.allOf(reportFuture, eventsFuture, userFuture).join();

        return new DashboardData(
            reportFuture.get(),
            eventsFuture.get(),
            userFuture.get()
        );
        // Thời gian = max(report, events, user) thay vì report + events + user
    }
}
```

### 4.3 Chaining (Chuỗi Xử Lý)

```java
@Async
public CompletableFuture<ProcessedData> fetchAndProcess(String url) {
    return CompletableFuture
        .supplyAsync(() -> httpClient.fetch(url))        // Bước 1: fetch
        .thenApply(raw -> dataParser.parse(raw))          // Bước 2: parse
        .thenApply(parsed -> enricher.enrich(parsed))     // Bước 3: enrich
        .exceptionally(ex -> {
            log.error("Pipeline failed", ex);
            return ProcessedData.empty();
        });
}
```

---

## 5. Cấu Hình ThreadPoolTaskExecutor

### 5.1 Vấn Đề Với Default Thread Pool

Mặc định, `@Async` dùng `SimpleAsyncTaskExecutor` — tạo thread mới cho **mỗi** lần gọi. Đây là anti-pattern trong production vì:
- Không giới hạn số thread
- Không tái sử dụng thread — tốn chi phí tạo/hủy

### 5.2 Cấu Hình Executor Tùy Chỉnh

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();

        // Core Pool Size (Kích Thước Hồ Cốt Lõi): số thread luôn tồn tại
        executor.setCorePoolSize(5);

        // Max Pool Size (Kích Thước Tối Đa): tạo thêm thread khi queue đầy
        executor.setMaxPoolSize(20);

        // Queue Capacity (Dung Lượng Hàng Đợi): buffer khi core threads bận
        executor.setQueueCapacity(100);

        // Thread Name Prefix (Tiền Tố Tên Luồng): dễ debug trong logs
        executor.setThreadNamePrefix("async-exec-");

        // Chờ task hoàn thành khi shutdown
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);

        executor.initialize();
        return executor;
    }

    @Override
    public Executor getAsyncExecutor() {
        return taskExecutor();
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new CustomAsyncExceptionHandler();
    }
}
```

### 5.3 Nhiều Executor Cho Các Mục Đích Khác Nhau

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    // Executor cho email — ít thread, ưu tiên thấp
    @Bean("emailExecutor")
    public Executor emailExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("email-");
        executor.initialize();
        return executor;
    }

    // Executor cho report — nhiều thread, CPU-intensive
    @Bean("reportExecutor")
    public Executor reportExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(30);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("report-");
        executor.initialize();
        return executor;
    }
}

@Service
public class NotificationService {

    @Async("emailExecutor")   // ← Chỉ định executor nào dùng
    public void sendEmail(String to, String content) { ... }
}

@Service
public class ReportService {

    @Async("reportExecutor")
    public CompletableFuture<byte[]> generatePdfReport(Long id) { ... }
}
```

### 5.4 Tính Toán Core Pool Size

```
CPU-bound tasks (Tác Vụ Bị Giới Hạn CPU):
  corePoolSize = số CPU core
  maxPoolSize  = số CPU core * 2

I/O-bound tasks (Tác Vụ Bị Giới Hạn I/O — network, DB, file):
  corePoolSize = số CPU core * (1 + wait_time / cpu_time)
  Ví dụ: 4 CPU, wait 90%, cpu 10% → corePoolSize = 4 * (1 + 9) = 40
```

---

## 6. Xử Lý Exception Trong @Async

### 6.1 Vấn Đề

Exception từ `@Async` method bị **nuốt im lặng** nếu return type là `void`:

```java
@Async
public void riskyTask() {
    throw new RuntimeException("Task failed!"); // ← Mất mát, không ai biết!
}
```

### 6.2 AsyncUncaughtExceptionHandler

```java
public class CustomAsyncExceptionHandler implements AsyncUncaughtExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(CustomAsyncExceptionHandler.class);

    @Override
    public void handleUncaughtException(Throwable ex, Method method, Object... params) {
        log.error("Async task failed: method={}, params={}, error={}",
                method.getName(), Arrays.toString(params), ex.getMessage(), ex);

        // Có thể: gửi alert, lưu vào DB, retry...
    }
}

@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new CustomAsyncExceptionHandler();
    }
}
```

### 6.3 Exception Với CompletableFuture

```java
@Async
public CompletableFuture<String> fetchData(String id) {
    try {
        String result = externalService.fetch(id);
        return CompletableFuture.completedFuture(result);
    } catch (Exception ex) {
        return CompletableFuture.failedFuture(ex); // ← Propagate exception
    }
}

// Phía gọi — xử lý exception
CompletableFuture<String> future = fetchData("123");
future.whenComplete((result, ex) -> {
    if (ex != null) {
        log.error("Fetch failed", ex);
    } else {
        processResult(result);
    }
});
```

---

## 7. Propagation Context (Truyền Ngữ Cảnh)

### Vấn Đề SecurityContext

```java
@Async
public void backgroundTask() {
    // SecurityContextHolder.getContext().getAuthentication() == null !
    // Vì @Async chạy trên thread mới, không có SecurityContext
}
```

### Giải Pháp — SecurityContextHolder Mode

```java
// Trong main thread, trước khi dùng @Async:
SecurityContextHolder.setStrategyName(
    SecurityContextHolder.MODE_INHERITABLETHREADLOCAL
);
// Hoặc trong application.properties:
// spring.security.strategy=MODE_INHERITABLETHREADLOCAL
```

### Truyền Thủ Công

```java
@Async
public void taskWithContext(Authentication auth) {
    SecurityContextHolder.getContext().setAuthentication(auth);
    try {
        // Thực hiện logic với security context
    } finally {
        SecurityContextHolder.clearContext(); // ← Bắt buộc clean up
    }
}

// Phía gọi:
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
backgroundService.taskWithContext(auth);
```

---

## 8. Cấu Hình application.properties

```properties
# Custom executor (nếu dùng Spring Boot auto-config)
spring.task.execution.pool.core-size=5
spring.task.execution.pool.max-size=20
spring.task.execution.pool.queue-capacity=100
spring.task.execution.pool.keep-alive=60s
spring.task.execution.thread-name-prefix=async-task-

# Shutdown behavior
spring.task.execution.shutdown.await-termination=true
spring.task.execution.shutdown.await-termination-period=60s
```

---

## 9. Monitoring (Giám Sát) với Actuator

```java
@Bean
public ThreadPoolTaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(5);
    executor.setMaxPoolSize(20);
    executor.setQueueCapacity(100);
    executor.setThreadNamePrefix("async-");

    // Bật metrics — tích hợp với Micrometer/Prometheus
    executor.setTaskDecorator(new MdcTaskDecorator());

    executor.initialize();
    return executor;
}
```

Xem metrics tại `/actuator/metrics/executor.*`:
- `executor.active` — số thread đang chạy
- `executor.queued` — số task trong hàng đợi
- `executor.pool.size` — kích thước pool hiện tại

---

## 10. Best Practices (Thực Hành Tốt Nhất)

### ✅ Nên Làm

```java
// 1. Luôn đặt tên executor và dùng đúng loại
@Async("emailExecutor")
public CompletableFuture<Void> sendEmail(...) { ... }

// 2. Trả về CompletableFuture thay vì void khi cần biết kết quả
@Async
public CompletableFuture<ReportResult> generateReport(Long id) { ... }

// 3. Handle exception đúng cách
@Override
public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
    return new CustomAsyncExceptionHandler();
}

// 4. Timeout cho CompletableFuture
CompletableFuture<String> result = service.asyncTask()
    .orTimeout(5, TimeUnit.SECONDS)     // Java 9+
    .exceptionally(ex -> "default");
```

### ❌ Tránh Làm

```java
// 1. Không dùng SimpleAsyncTaskExecutor trong production
// (Mặc định nếu không cấu hình)

// 2. Không gọi @Async từ cùng class (self-invocation)
public void method() {
    this.asyncMethod(); // ❌ Không hoạt động
}

// 3. Không bỏ qua exception từ void @Async
@Async
public void fire() {
    throw new RuntimeException(); // ❌ Mất mát nếu không có handler
}

// 4. Không truyền entity JPA vào @Async — session đã đóng
@Async
public void processOrder(Order order) {
    order.getItems(); // ❌ LazyInitializationException
    // Truyền ID, load lại trong background thread
}
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: `@Async` hoạt động thế nào bên dưới?**
> Spring tạo AOP proxy cho bean chứa `@Async`. Khi method được gọi qua proxy, Spring interceptor submit task vào `TaskExecutor` thay vì gọi trực tiếp. Đây là lý do self-invocation không hoạt động — bỏ qua proxy.

**Q: Khi nào nên dùng `CompletableFuture` thay vì `void`?**
> Dùng `CompletableFuture` khi: (1) cần biết task thành công/thất bại, (2) muốn chờ kết quả, (3) muốn chạy song song nhiều task rồi `allOf()`. Dùng `void` cho fire-and-forget (bắn và quên) như ghi log, gửi notification.

**Q: `corePoolSize` vs `maxPoolSize` vs `queueCapacity` — khi nào thread mới được tạo?**
> 1. Nếu số thread < `corePoolSize` → tạo thread mới ngay.
> 2. Nếu số thread >= `corePoolSize` → thêm vào queue.
> 3. Nếu queue đầy và số thread < `maxPoolSize` → tạo thêm thread.
> 4. Nếu queue đầy và số thread = `maxPoolSize` → áp dụng `RejectedExecutionHandler`.

**Q: Cách debug @Async task?**
> (1) Đặt `thread-name-prefix` rõ ràng → thấy trong logs. (2) Dùng MDC (Mapped Diagnostic Context — Ngữ Cảnh Chẩn Đoán Có Ánh Xạ) để truyền request ID sang thread mới. (3) Enable `/actuator/metrics/executor.*` để monitor pool health.
