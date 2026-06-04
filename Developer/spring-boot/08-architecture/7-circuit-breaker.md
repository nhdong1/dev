# Circuit Breaker — Resilience4j

> Resilience4j là thư viện khả năng chịu lỗi (fault tolerance) nhẹ nhàng nhất cho Java, lấy cảm hứng từ Netflix Hystrix. Cung cấp CircuitBreaker (Cầu Dao Mạch), Retry (Thử Lại), RateLimiter (Giới Hạn Tốc Độ), TimeLimiter (Giới Hạn Thời Gian) và Bulkhead (Vách Ngăn).

---

## 📋 Mục Lục

1. [Tại Sao Cần Circuit Breaker?](#tại-sao-cần-circuit-breaker)
2. [Circuit Breaker — Cầu Dao Mạch](#circuit-breaker--cầu-dao-mạch)
3. [Retry — Thử Lại](#retry--thử-lại)
4. [Bulkhead — Vách Ngăn](#bulkhead--vách-ngăn)
5. [TimeLimiter — Giới Hạn Thời Gian](#timelimiter--giới-hạn-thời-gian)
6. [RateLimiter — Giới Hạn Tốc Độ](#ratelimiter--giới-hạn-tốc-độ)
7. [Kết Hợp Nhiều Patterns](#kết-hợp-nhiều-patterns)
8. [Monitoring & Actuator](#monitoring--actuator)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Circuit Breaker?

### Cascade Failure (Lỗi Dây Chuyền)

```
Không Có Circuit Breaker:

Order Service ────HTTP────► Inventory Service (DOWN — Tắt)
     │
     │ Thread bị block 30 giây đợi timeout
     │
     ▼
Thread Pool của Order Service bị EXHAUSTED (Cạn Kiệt)
     │
     ▼
Order Service không xử lý được request nào khác
     │
     ▼
API Gateway timeout → Client nhận 503
     │
     ▼
Toàn bộ hệ thống bị ảnh hưởng dù chỉ Inventory Service down

→ Cascade Failure (Lỗi Dây Chuyền)
```

```
Có Circuit Breaker:

Order Service ────HTTP────► Inventory Service (DOWN)
     │
     │ CircuitBreaker phát hiện failures liên tục
     │
     ▼
CircuitBreaker OPEN (Mở)
     │
     ▼
Subsequent requests → FAIL FAST (Thất Bại Nhanh) — không đợi timeout
     │
     ▼
Fallback được gọi ngay lập tức → Order Service vẫn hoạt động
     │
     ▼
Inventory Service phục hồi → CircuitBreaker tự CLOSE (Đóng) lại
```

### Three States of Circuit Breaker (Ba Trạng Thái)

```
                    CLOSED (Đóng)
                   /            \
                  /              \
        Failure rate           Success
        > threshold            in half-open
                  \              /
                   \            /
                    OPEN (Mở)
                         │
                         │ Wait duration elapsed
                         ▼
                   HALF-OPEN (Nửa Mở)
                   (Cho phép N requests thử)

CLOSED:    Hoạt động bình thường, đếm failures
OPEN:      Fail fast tất cả requests, gọi fallback
HALF-OPEN: Thử nghiệm N requests để kiểm tra service đã phục hồi chưa
```

---

## Circuit Breaker — Cầu Dao Mạch

### Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```

### Cấu Hình YAML

```yaml
resilience4j:
  circuitbreaker:
    instances:
      inventory-service:
        # Loại sliding window: COUNT_BASED hoặc TIME_BASED
        sliding-window-type: COUNT_BASED
        # Số request để tính failure rate (khi COUNT_BASED)
        sliding-window-size: 10
        # Tỉ lệ lỗi để chuyển sang OPEN (%)
        failure-rate-threshold: 50
        # Thời gian giữ trạng thái OPEN trước khi chuyển sang HALF-OPEN
        wait-duration-in-open-state: 10s
        # Số request được phép trong HALF-OPEN
        permitted-number-of-calls-in-half-open-state: 3
        # Tỉ lệ chậm để trigger (% requests > slow-call-duration-threshold)
        slow-call-rate-threshold: 80
        # Request nào được coi là "chậm"
        slow-call-duration-threshold: 3s
        # Số request tối thiểu trước khi tính failure rate
        minimum-number-of-calls: 5
        # Tự động chuyển từ OPEN sang HALF-OPEN (không đợi request)
        automatic-transition-from-open-to-half-open-enabled: true

      payment-service:
        sliding-window-type: TIME_BASED  # Dựa theo thời gian
        sliding-window-size: 60          # 60 giây
        failure-rate-threshold: 30       # Thắt chặt hơn cho payment
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 5
```

### Sử Dụng Với Annotation

```java
@Service
@Slf4j
public class OrderService {

    private final InventoryClient inventoryClient;

    // Kết hợp CB + Retry + TimeLimiter
    @CircuitBreaker(name = "inventory-service", fallbackMethod = "checkStockFallback")
    @Retry(name = "inventory-service")
    @TimeLimiter(name = "inventory-service")
    public CompletableFuture<InventoryResponse> checkStock(String productId, int quantity) {
        return CompletableFuture.supplyAsync(() ->
            inventoryClient.checkStock(productId, quantity)
        );
    }

    // Fallback method — phải có cùng signature + Exception parameter
    public CompletableFuture<InventoryResponse> checkStockFallback(
            String productId, int quantity, Exception e) {
        log.warn("Inventory service không khả dụng ({}), dùng fallback", e.getMessage());

        // Chiến lược fallback: assume có hàng, kiểm tra sau
        return CompletableFuture.completedFuture(
            InventoryResponse.optimisticAvailability(productId, quantity)
        );
    }

    // Fallback cho trường hợp cụ thể hơn
    public CompletableFuture<InventoryResponse> checkStockFallback(
            String productId, int quantity, CallNotPermittedException e) {
        // Lỗi này xảy ra khi CB ở trạng thái OPEN
        log.error("Circuit Breaker OPEN cho inventory-service");
        throw new ServiceUnavailableException("Inventory service tạm thời không khả dụng");
    }
}
```

### Sử Dụng Programmatic (Lập Trình Thủ Công)

```java
@Service
public class PaymentService {

    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final ExternalPaymentGateway paymentGateway;

    public PaymentResult processPayment(PaymentRequest request) {
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("payment-gateway");

        // Decorating function với circuit breaker
        Supplier<PaymentResult> decoratedSupplier = CircuitBreaker.decorateSupplier(
            cb,
            () -> paymentGateway.charge(request)
        );

        // Thêm fallback
        Try<PaymentResult> result = Try.ofSupplier(decoratedSupplier)
            .recover(CallNotPermittedException.class, e -> {
                log.warn("Payment gateway CB OPEN, dùng fallback queue");
                return queuePaymentForRetry(request);
            })
            .recover(Exception.class, e -> {
                log.error("Payment failed: {}", e.getMessage());
                throw new PaymentException("Không thể xử lý thanh toán", e);
            });

        return result.get();
    }
}

// Lắng nghe State Transitions (Chuyển Đổi Trạng Thái)
@Component
public class CircuitBreakerEventListener {

    @Bean
    public RegistryEventConsumer<CircuitBreaker> circuitBreakerEventConsumer(
            MeterRegistry meterRegistry) {
        return new RegistryEventConsumer<>() {
            @Override
            public void onEntryAddedEvent(EntryAddedEvent<CircuitBreaker> event) {
                CircuitBreaker cb = event.getAddedEntry();

                cb.getEventPublisher()
                    .onStateTransition(transition -> {
                        log.warn("CircuitBreaker '{}' chuyển trạng thái: {} → {}",
                            cb.getName(),
                            transition.getStateTransition().getFromState(),
                            transition.getStateTransition().getToState()
                        );
                        // Alert team khi CB mở
                        if (transition.getStateTransition().getToState() == CircuitBreaker.State.OPEN) {
                            alertingService.sendAlert("Circuit Breaker OPEN: " + cb.getName());
                        }
                    })
                    .onFailureRateExceeded(event2 ->
                        log.warn("CB '{}' failure rate: {}%", cb.getName(), event2.getFailureRate())
                    );
            }
        };
    }
}
```

---

## Retry — Thử Lại

### Cấu Hình

```yaml
resilience4j:
  retry:
    instances:
      inventory-service:
        # Số lần thử lại tối đa (bao gồm lần đầu)
        max-attempts: 3
        # Thời gian chờ giữa các lần retry
        wait-duration: 500ms
        # Exponential backoff (Hồi Lui Lũy Thừa)
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2    # 500ms → 1000ms → 2000ms
        # Chỉ retry với các exception này
        retry-exceptions:
          - java.io.IOException
          - java.net.SocketTimeoutException
          - feign.RetryableException
        # KHÔNG retry với các exception này (business exceptions)
        ignore-exceptions:
          - com.example.exception.InsufficientStockException
          - com.example.exception.ProductNotFoundException
```

### Retry Với Jitter (Ngẫu Nhiên Hóa Thời Gian)

```java
// Jitter (Biến Động) tránh thundering herd — nhiều services retry cùng lúc
@Configuration
public class RetryConfig {

    @Bean
    public RetryRegistry retryRegistry() {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofMillis(500))
            .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
                Duration.ofMillis(500),    // Initial wait
                2.0,                       // Multiplier
                Duration.ofSeconds(10)     // Max wait
            ))
            .retryExceptions(IOException.class, SocketTimeoutException.class)
            .ignoreExceptions(BusinessException.class)
            .build();

        return RetryRegistry.of(config);
    }
}

@Service
public class InventoryService {

    @Retry(name = "inventory-service", fallbackMethod = "reserveStockFallback")
    public void reserveStock(String orderId, List<OrderItem> items) {
        inventoryClient.reserve(orderId, items);
    }

    // Fallback sau khi đã retry hết
    public void reserveStockFallback(String orderId, List<OrderItem> items, Exception e) {
        log.error("Hết lần retry cho inventory. OrderId: {}", orderId);
        // Publish event để xử lý async
        eventPublisher.publishEvent(new InventoryReservationFailedEvent(orderId, items));
    }
}
```

---

## Bulkhead — Vách Ngăn

### Thread Pool Bulkhead (Vách Ngăn Bể Thread)

Isolate thread pools để một service chậm không chiếm toàn bộ threads.

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      inventory-service:
        # Số threads trong pool riêng cho inventory-service
        max-thread-pool-size: 10
        # Queue capacity (hàng đợi)
        queue-capacity: 20
        # Thread nào không dùng sau 1 phút sẽ bị terminate
        keep-alive-duration: 1m
        # Core thread pool size
        core-thread-pool-size: 5

      payment-service:
        max-thread-pool-size: 5    # Ít hơn vì payment ít traffic hơn
        queue-capacity: 10
        core-thread-pool-size: 2
```

### Semaphore Bulkhead (Vách Ngăn Semaphore)

```yaml
resilience4j:
  bulkhead:
    instances:
      # Giới hạn số concurrent calls (lời gọi đồng thời)
      external-api:
        max-concurrent-calls: 25      # Tối đa 25 concurrent calls
        max-wait-duration: 0ms        # Không đợi — fail ngay nếu đầy
```

```java
@Service
public class ReportService {

    // Bulkhead giới hạn số concurrent report generations
    @Bulkhead(name = "report-generation", type = Bulkhead.Type.THREADPOOL,
              fallbackMethod = "generateReportFallback")
    public CompletableFuture<Report> generateReport(ReportRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            // Heavy computation — có thể mất nhiều thời gian
            return reportEngine.generate(request);
        });
    }

    public CompletableFuture<Report> generateReportFallback(
            ReportRequest request, BulkheadFullException e) {
        log.warn("Report queue đầy, trả về cached report nếu có");
        return CompletableFuture.completedFuture(reportCache.getLatest(request.getType()));
    }
}
```

---

## TimeLimiter — Giới Hạn Thời Gian

```yaml
resilience4j:
  timelimiter:
    instances:
      inventory-service:
        # Timeout sau 2 giây
        timeout-duration: 2s
        # Hủy CompletableFuture nếu timeout
        cancel-running-future: true
```

```java
@Service
public class OrderService {

    @TimeLimiter(name = "inventory-service", fallbackMethod = "checkStockTimeout")
    public CompletableFuture<InventoryResponse> checkStockAsync(String productId) {
        return CompletableFuture.supplyAsync(() -> inventoryClient.checkStock(productId));
    }

    // Fallback khi timeout
    public CompletableFuture<InventoryResponse> checkStockTimeout(
            String productId, TimeoutException e) {
        log.warn("Inventory check timeout cho product: {}", productId);
        // Trả về "probably available" khi timeout
        return CompletableFuture.completedFuture(InventoryResponse.unknown(productId));
    }
}
```

---

## RateLimiter — Giới Hạn Tốc Độ

```yaml
resilience4j:
  ratelimiter:
    instances:
      external-payment-api:
        # Số request được phép trong mỗi refresh period
        limit-for-period: 10
        # Refresh period (làm mới giới hạn)
        limit-refresh-period: 1s
        # Timeout chờ nếu không có slot (0 = fail ngay)
        timeout-duration: 0s
```

```java
@Service
public class NotificationService {

    // Giới hạn tốc độ gọi external email API
    @RateLimiter(name = "email-api", fallbackMethod = "sendEmailFallback")
    public void sendEmail(EmailRequest request) {
        emailApiClient.send(request);
    }

    public void sendEmailFallback(EmailRequest request, RequestNotPermitted e) {
        log.warn("Email rate limit exceeded, queuing: {}", request.getTo());
        emailQueue.add(request); // Đưa vào queue để gửi sau
    }
}
```

---

## Kết Hợp Nhiều Patterns

### Thứ Tự Áp Dụng (Decorator Order)

```
Request đến
     │
     ▼
[1] RateLimiter    ← Kiểm tra rate limit trước
     │
     ▼
[2] CircuitBreaker ← Kiểm tra CB state
     │
     ▼
[3] Bulkhead       ← Kiểm tra concurrent limit
     │
     ▼
[4] TimeLimiter    ← Set timeout
     │
     ▼
[5] Retry          ← Retry nếu fail (bao gồm cả CB open, timeout)
     │
     ▼
Actual Call
```

```yaml
# Cấu hình đầy đủ cho một service
resilience4j:
  circuitbreaker:
    instances:
      payment-service:
        failure-rate-threshold: 50
        sliding-window-size: 10
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 5

  retry:
    instances:
      payment-service:
        max-attempts: 2          # Ít retry hơn cho payment
        wait-duration: 1s
        retry-exceptions:
          - java.io.IOException
        ignore-exceptions:
          - com.example.PaymentDeclinedException

  timelimiter:
    instances:
      payment-service:
        timeout-duration: 5s     # Payment cần thời gian hơn

  bulkhead:
    instances:
      payment-service:
        max-concurrent-calls: 20
        max-wait-duration: 500ms
```

```java
@Service
public class PaymentService {

    @CircuitBreaker(name = "payment-service", fallbackMethod = "paymentFallback")
    @Retry(name = "payment-service")
    @TimeLimiter(name = "payment-service")
    @Bulkhead(name = "payment-service", type = Bulkhead.Type.SEMAPHORE)
    public CompletableFuture<PaymentResult> processPayment(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() ->
            externalPaymentGateway.charge(request)
        );
    }

    // Fallback theo exception type — từ cụ thể đến chung
    public CompletableFuture<PaymentResult> paymentFallback(
            PaymentRequest request, CallNotPermittedException e) {
        // CB đang OPEN
        return CompletableFuture.failedFuture(
            new ServiceUnavailableException("Payment service tạm không khả dụng")
        );
    }

    public CompletableFuture<PaymentResult> paymentFallback(
            PaymentRequest request, TimeoutException e) {
        // Timeout
        log.error("Payment timeout cho orderId: {}", request.getOrderId());
        return CompletableFuture.failedFuture(
            new PaymentTimeoutException("Kết nối payment gateway timeout")
        );
    }

    public CompletableFuture<PaymentResult> paymentFallback(
            PaymentRequest request, Exception e) {
        // Fallback chung
        log.error("Payment failed: {}", e.getMessage());
        return CompletableFuture.failedFuture(
            new PaymentException("Không thể xử lý thanh toán")
        );
    }
}
```

---

## Monitoring & Actuator

### Expose Resilience4j Metrics

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,circuitbreakers,retries,bulkheads
  endpoint:
    health:
      show-details: always
  health:
    circuitbreakers:
      enabled: true
    retryevents:
      enabled: true

# Tích hợp với Micrometer → Prometheus
resilience4j:
  circuitbreaker:
    instances:
      inventory-service:
        register-health-indicator: true    # Hiện trong /actuator/health
        event-consumer-buffer-size: 10     # Buffer cho events
```

### Prometheus Metrics

```
# Các metrics Resilience4j expose cho Prometheus

resilience4j_circuitbreaker_state{name="inventory-service"} 0  # 0=CLOSED, 1=OPEN, 2=HALF_OPEN
resilience4j_circuitbreaker_failure_rate{name="inventory-service"} 25.0
resilience4j_circuitbreaker_calls_total{name="inventory-service",kind="successful"} 1500
resilience4j_circuitbreaker_calls_total{name="inventory-service",kind="failed"} 200
resilience4j_circuitbreaker_calls_total{name="inventory-service",kind="not_permitted"} 50
resilience4j_circuitbreaker_slow_calls_total{name="inventory-service"} 30

resilience4j_retry_calls_total{name="inventory-service",kind="successful_with_retry"} 45
resilience4j_retry_calls_total{name="inventory-service",kind="failed_with_retry"} 10

resilience4j_bulkhead_available_concurrent_calls{name="payment-service"} 15
```

### Grafana Dashboard Queries

```
# Tỉ lệ lỗi của Circuit Breaker
rate(resilience4j_circuitbreaker_calls_total{kind="failed"}[5m]) /
rate(resilience4j_circuitbreaker_calls_total[5m]) * 100

# Trạng thái Circuit Breaker (alert khi > 0 = OPEN)
resilience4j_circuitbreaker_state{name=~".*"} > 0

# Số lần CB không cho phép (OPEN đang fail fast)
increase(resilience4j_circuitbreaker_calls_total{kind="not_permitted"}[5m])
```

---

## Câu Hỏi Phỏng Vấn

**Q: Circuit Breaker hoạt động thế nào? Khi nào nên dùng?**

> Circuit Breaker theo dõi tỉ lệ lỗi trong một sliding window (cửa sổ trượt). Khi failure rate vượt threshold, CB chuyển sang OPEN — reject tất cả requests ngay lập tức (fail fast) thay vì đợi timeout. Sau wait duration, CB chuyển HALF-OPEN cho một số requests thử nghiệm; nếu thành công thì CLOSE lại. Dùng khi: gọi external services có thể fail, muốn tránh cascade failures, cần fallback behavior.

**Q: Resilience4j có những patterns nào? Mỗi cái giải quyết vấn đề gì?**

> (1) **CircuitBreaker** — ngăn cascade failures khi downstream service fail; (2) **Retry** — tự động thử lại transient failures (lỗi thoáng qua) như network blips; (3) **Bulkhead** — isolate thread pools/semaphores tránh một service chậm làm chết service khác; (4) **TimeLimiter** — đặt timeout cho async calls, không để thread bị block vô thời hạn; (5) **RateLimiter** — giới hạn throughput gọi external APIs có rate limit.

**Q: Retry và Circuit Breaker dùng cùng nhau có vấn đề gì không?**

> Cần cẩn thận về thứ tự và interaction. Nếu CB ở OPEN, Retry cũng sẽ retry nhưng mỗi lần sẽ bị CB reject — gây tốn công retry vô ích. Giải pháp: (1) Không ignore `CallNotPermittedException` trong Retry config; (2) Đặt CB wraps Retry (CB ở ngoài, Retry ở trong) để khi CB OPEN, toàn bộ lần retry đều không thực hiện; (3) Hoặc set max-attempts của Retry thấp (2-3) để tránh delay quá lâu.

**Q: Bulkhead là gì, Thread Pool Bulkhead vs Semaphore Bulkhead?**

> Bulkhead (lấy từ thuật ngữ hàng hải — vách ngăn tàu) isolate các phần của system để lỗi không lan ra. Thread Pool Bulkhead: mỗi service có thread pool riêng — nếu service chậm, chỉ pool đó bị block, các pools khác không bị ảnh hưởng. Semaphore Bulkhead: giới hạn số concurrent calls bằng semaphore — nhẹ hơn nhưng không isolate thread. Thread Pool phù hợp hơn cho external calls blocking (HTTP), Semaphore phù hợp cho non-blocking code.

---

## ✅ Checklist

- [ ] CircuitBreaker cấu hình cho tất cả external service calls
- [ ] Fallback methods được implement cho mọi CB/Retry
- [ ] Retry chỉ áp dụng cho transient failures — không retry business exceptions
- [ ] Bulkhead tách thread pools cho các downstream services quan trọng
- [ ] Metrics expose cho Prometheus — setup alert khi CB OPEN
- [ ] TimeLimiter ngắn hơn timeout của upstream service để tránh cascade timeout
- [ ] Test failure scenarios — kiểm tra fallback hoạt động đúng

---

**Module hoàn thành!** Quay lại [README.md](README.md) để xem tổng quan hoặc tiếp tục với [09-cloud-deployment](../09-cloud-deployment/README.md).
