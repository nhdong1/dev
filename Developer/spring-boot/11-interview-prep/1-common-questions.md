# 📖 Câu Hỏi Thường Gặp Theo Chủ Đề

> Bộ câu hỏi phỏng vấn được phân loại theo từng chủ đề kỹ thuật — dùng để học sâu từng area, không phải ôn tổng quát. Mỗi câu có gợi ý trả lời và những điểm quan trọng cần đề cập.

---

## 🔵 Chủ Đề 1: Spring Core & IoC/DI

### Câu hỏi nền tảng

**Q: Spring Container (ApplicationContext) khác BeanFactory như thế nào?**

Gợi ý trả lời:
- `BeanFactory` là interface cơ bản — lazy initialization (khởi tạo lười biếng), không có AOP, events
- `ApplicationContext` extends `BeanFactory` — eager initialization, có AOP, event publishing, i18n, resource loading
- Thực tế luôn dùng `ApplicationContext` (`AnnotationConfigApplicationContext`, `WebApplicationContext`)

Điểm cộng: Đề cập `ClassPathXmlApplicationContext` (cũ) vs `AnnotationConfigApplicationContext` (hiện đại)

---

**Q: @Autowired có thể inject collection không? Ví dụ?**

```java
// Spring inject TẤT CẢ beans implement PaymentProcessor
@Autowired
private List<PaymentProcessor> processors; // [VNPayProcessor, MomoProcessor, StripeProcessor]

// Hoặc inject theo tên
@Autowired
private Map<String, PaymentProcessor> processorMap;
// key = bean name, value = bean instance
```

Ứng dụng thực tế: Strategy Pattern — chọn processor theo loại thanh toán.

---

**Q: @Primary và @Qualifier khác nhau thế nào?**

- `@Primary` — đánh dấu bean ưu tiên khi có nhiều bean cùng type → áp dụng toàn global
- `@Qualifier("beanName")` — chỉ định chính xác bean nào cần inject → áp dụng tại điểm inject

```java
@Bean @Primary
public DataSource primaryDS() { ... }   // dùng mặc định

@Bean("readonlyDS")
public DataSource readonlyDS() { ... }  // chỉ dùng khi @Qualifier("readonlyDS")
```

---

**Q: @Value vs @ConfigurationProperties — Khi nào dùng cái nào?**

- `@Value("${property.key}")` — inject đơn lẻ từng property → đơn giản nhưng không type-safe
- `@ConfigurationProperties(prefix = "app")` — nhóm properties vào POJO → type-safe, IDE autocomplete, validation

```java
// @ConfigurationProperties — khuyến nghị cho nhóm properties liên quan
@ConfigurationProperties(prefix = "app.payment")
public record PaymentConfig(
    String apiUrl,
    int timeoutMs,
    boolean sandboxMode
) {}
```

---

**Q: Spring Profiles hoạt động thế nào? Cách activate trong các môi trường khác nhau?**

Gợi ý trả lời:
- Profiles cho phép tải config/bean khác nhau theo môi trường (dev, test, prod)
- Activate qua: `spring.profiles.active=prod`, environment variable `SPRING_PROFILES_ACTIVE`, hoặc `-Dspring.profiles.active=prod`
- `@Profile("prod")` trên @Bean hoặc @Component
- `application-{profile}.yml` tự động được load

---

### Câu hỏi trung cấp

**Q: Khi nào xảy ra BeanCreationException? Cách debug?**

Các nguyên nhân phổ biến:
1. Missing required bean — `@Autowired` không tìm thấy bean phù hợp
2. Circular dependency với constructor injection
3. @PostConstruct method throw exception
4. Cấu hình sai (ví dụ: datasource URL sai → kết nối fail khi startup)

Debug: Đọc root cause trong stack trace — thường là `Caused by:` ở cuối.

---

**Q: ApplicationContext Events — Khi nào dùng @EventListener?**

```java
// Publish event
@Service
public class OrderService {
    @Autowired ApplicationEventPublisher eventPublisher;

    @Transactional
    public void createOrder(Order order) {
        orderRepo.save(order);
        eventPublisher.publishEvent(new OrderCreatedEvent(order));
        // Event listener chạy TRONG cùng transaction
    }
}

// Listen to event
@EventListener
@Async // chạy bất đồng bộ trong thread khác
public void onOrderCreated(OrderCreatedEvent event) {
    emailService.sendConfirmation(event.getOrder());
}

// @TransactionalEventListener — chỉ chạy SAU KHI transaction commit thành công
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onOrderCreated(OrderCreatedEvent event) {
    // Chạy sau commit → đảm bảo order đã lưu DB trước khi gửi email
    emailService.sendConfirmation(event.getOrder());
}
```

---

## 🟢 Chủ Đề 2: Spring Data JPA & Database

### Câu hỏi nền tảng

**Q: @Entity, @Table, @Column — Khi nào cần @Column?**

Gợi ý: `@Column` optional — chỉ cần khi muốn custom column name, nullable, length, unique constraint:

```java
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    // Tên field = tên column (auto) → không cần @Column
    private String email;
    
    // Override khi cần customize
    @Column(name = "full_name", nullable = false, length = 100, unique = true)
    private String name;
}
```

---

**Q: @GeneratedValue strategies — IDENTITY vs SEQUENCE vs TABLE?**

| Strategy | Cách hoạt động | Dùng với |
| -------- | -------------- | -------- |
| `IDENTITY` | DB auto-increment | MySQL, PostgreSQL SERIAL |
| `SEQUENCE` | DB sequence object | PostgreSQL (hiệu quả hơn) |
| `TABLE` | Bảng đặc biệt lưu next ID | Portable nhưng chậm (locking) |
| `AUTO` | Hibernate chọn tự động | Không khuyến nghị production |

**Điểm cộng:** SEQUENCE tốt hơn IDENTITY vì cho phép batch insert — Hibernate có thể pre-fetch IDs.

---

**Q: Sự khác biệt giữa `save()` và `saveAndFlush()` trong JpaRepository?**

- `save()` — lưu vào persistence context (first-level cache — bộ nhớ cache cấp 1), chưa chắc đã flush xuống DB ngay
- `saveAndFlush()` — flush ngay lập tức xuống DB (thực thi SQL INSERT/UPDATE)
- `flush()` xảy ra tự động: khi commit transaction, hoặc khi query cần nhất quán

Dùng `saveAndFlush()` khi cần: test với `@DataJpaTest`, hoặc cần ID ngay sau khi save.

---

**Q: Cascade Types — khi nào dùng CascadeType.ALL?**

```java
// ✅ Phù hợp: Order sở hữu OrderItems — lifecycle gắn liền
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderItem> items;

// ❌ Không phù hợp: User và Role — independent entities
@ManyToMany(cascade = CascadeType.ALL) // NGUY HIỂM! Delete user → delete role!
private Set<Role> roles;
// → Chỉ dùng cascade = {} hoặc loại bỏ cascade hoàn toàn với ManyToMany
```

`orphanRemoval = true` — tự động xóa child entities khi bị remove khỏi parent collection.

---

### Câu hỏi trung cấp

**Q: First-level cache vs Second-level cache trong Hibernate?**

- **First-level cache (Session Cache):** Tự động, per-transaction — trong 1 transaction, entity load 2 lần chỉ query DB 1 lần
- **Second-level cache (L2 Cache):** Optional, shared across transactions — cần cấu hình (Ehcache, Redis)

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE) // bật L2 cache
public class Category { ... }
```

Điểm cộng: L2 cache gây vấn đề với concurrent updates — phải chọn strategy (READ_ONLY, READ_WRITE, NONSTRICT_READ_WRITE) phù hợp.

---

**Q: Optimistic Locking (Khóa Lạc Quan) vs Pessimistic Locking (Khóa Bi Quan)?**

```java
// Optimistic Locking — dùng @Version
@Entity
public class Product {
    @Version
    private int version; // Hibernate tự động quản lý

    // UPDATE product SET stock = ?, version = version+1
    // WHERE id = ? AND version = ? (version cũ)
    // Nếu version đã thay đổi → OptimisticLockException
}

// Pessimistic Locking — khóa DB thực sự (SELECT ... FOR UPDATE)
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Product> findById(Long id);
// → Các transaction khác BLOCK cho đến khi transaction này commit
```

**Khi nào dùng:**
- Optimistic: Ít conflict, read-heavy, UX tốt hơn
- Pessimistic: Nhiều conflict, critical sections (như inventory deduction)

---

**Q: Auditing tự động với Spring Data JPA — Cách implement?**

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class) // Spring Data Auditing
public abstract class BaseEntity {
    
    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updatedAt;
    
    @CreatedBy
    @Column(updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    private String updatedBy;
}

// @SpringBootApplication class:
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
```

---

## 🔴 Chủ Đề 3: Spring Security

### Câu hỏi nền tảng

**Q: Sự khác biệt giữa `permitAll()` và `anonymous()`?**

- `permitAll()` — cho phép tất cả truy cập (cả authenticated và anonymous)
- `anonymous()` — chỉ cho phép anonymous users (users chưa login)
- Thực tế: `permitAll()` được dùng cho public endpoints (login, register, public API)

---

**Q: Cách implement Role-based Authorization (Phân Quyền Dựa Trên Vai Trò)?**

```java
// Option 1: Config-based
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .requestMatchers("/api/manager/**").hasAnyRole("ADMIN", "MANAGER")
    .anyRequest().authenticated()
);

// Option 2: Method-level (@PreAuthorize)
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
public UserDTO getUser(Long userId) { ... }
// hasRole('ADMIN') OR user đang xem profile của chính mình

// Option 3: @PostAuthorize — kiểm tra SAU khi method chạy
@PostAuthorize("returnObject.ownerId == authentication.principal.id")
public Document getDocument(Long id) { ... }
```

---

**Q: Cách lưu trữ mật khẩu an toàn?**

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12); // strength = 12 (work factor)
    // BCrypt tự generate salt, chứa salt trong hash string
    // $2a$12$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
}

// Verify
boolean isValid = passwordEncoder.matches(rawPassword, encodedPassword);
```

**Điểm quan trọng:**
- Không dùng MD5/SHA1 (không có salt, dễ rainbow table attack)
- BCrypt/Argon2/scrypt — có work factor, slow by design để chống brute force
- Không bao giờ decrypt mật khẩu — chỉ verify bằng `matches()`

---

### Câu hỏi trung cấp

**Q: Cách implement Multi-tenancy Security (Bảo Mật Đa Người Thuê)?**

Gợi ý trả lời:
- Row-level security: mỗi query filter thêm `WHERE tenant_id = :currentTenant`
- Dùng Hibernate Filter hoặc Spring AOP để tự động inject tenant filter
- `@CurrentSecurityContext` để lấy tenant từ JWT claims

---

**Q: Cách bảo vệ endpoint khỏi Rate Limiting (Giới Hạn Tần Suất)?**

```java
// Với Spring Cloud Gateway
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("api_route", r -> r.path("/api/**")
            .filters(f -> f
                .requestRateLimiter(c -> c
                    .setRateLimiter(redisRateLimiter())
                    .setKeyResolver(userKeyResolver())))
            .uri("lb://api-service"))
        .build();
}

// Với Resilience4j RateLimiter
@RateLimiter(name = "loginEndpoint", fallbackMethod = "rateLimitFallback")
@PostMapping("/api/auth/login")
public ResponseEntity<TokenResponse> login(@RequestBody LoginRequest request) { ... }
```

---

**Q: Stateless vs Stateful Authentication — Trade-offs?**

| | Stateless (JWT) | Stateful (Session) |
|-| --------------- | ------------------ |
| **Scalability** | ✅ Dễ scale ngang | ❌ Cần sticky sessions hoặc shared session store |
| **Logout ngay lập tức** | ❌ Khó — token valid đến hết TTL | ✅ Xóa session là xong |
| **Token revocation** | ❌ Cần blocklist | ✅ Native support |
| **Payload size** | ❌ Mỗi request kèm token | ✅ Chỉ session ID nhỏ |
| **Microservices** | ✅ Mỗi service tự verify | ❌ Phải gọi session store |

---

## 🟡 Chủ Đề 4: Testing

### Câu hỏi nền tảng

**Q: Unit Test vs Integration Test vs End-to-End Test — Kim Tự Tháp Kiểm Thử?**

```
         /\
        /E2E\          ← Ít nhất (chậm, brittle — dễ gãy)
       /------\
      /  Integ \       ← Vừa phải
     /----------\
    /  Unit Tests \    ← Nhiều nhất (nhanh, isolated)
   /--------------\
```

- **Unit Test**: Test 1 class/function trong isolation — mock tất cả dependencies
- **Integration Test**: Test nhiều components cùng nhau — thường có DB, cache thật
- **E2E Test**: Test toàn bộ flow từ UI đến DB — Selenium, Cypress

---

**Q: @DirtiesContext là gì? Tại sao nên tránh?**

`@DirtiesContext` buộc Spring reload ApplicationContext sau mỗi test → chậm vì context creation tốn nhiều thời gian.

```java
// ❌ Tránh dùng — làm chậm test suite
@SpringBootTest
@DirtiesContext
class OrderTest { ... }

// ✅ Thay bằng: clean up data trong @BeforeEach/@AfterEach
@SpringBootTest
class OrderTest {
    @BeforeEach
    void cleanUp() {
        orderRepo.deleteAll(); // xóa data test, không reload context
    }
}
```

---

**Q: Cách test @Scheduled tasks?**

```java
// Option 1: Test logic trực tiếp — không cần @Scheduled
@Service
public class ReportService {
    public void generateDailyReport() { /* logic */ } // test method này
    
    @Scheduled(cron = "0 0 6 * * *")
    public void scheduledReport() {
        generateDailyReport(); // delegate sang method thuần
    }
}

// Option 2: Dùng ScheduledTaskHolder để verify task được register
@SpringBootTest
class ScheduledTaskTest {
    @Autowired ScheduledTaskHolder scheduledTaskHolder;
    
    @Test
    void verifyDailyReportTaskIsScheduled() {
        Set<ScheduledTask> tasks = scheduledTaskHolder.getScheduledTasks();
        assertThat(tasks).anyMatch(task -> 
            task.toString().contains("generateDailyReport"));
    }
}
```

---

### Câu hỏi trung cấp

**Q: Spy vs Mock trong Mockito — Khác nhau thế nào?**

```java
// Mock — fake hoàn toàn, không gọi real implementation
OrderService mockService = mock(OrderService.class);
when(mockService.create(any())).thenReturn(new Order()); // phải stub tất cả

// Spy — wrap real object, chỉ stub method cần thiết
OrderService realService = new OrderService(mockRepo);
OrderService spyService = spy(realService);
doReturn(new Order()).when(spyService).create(any()); // override 1 method
// Các method khác vẫn gọi real implementation
```

**Khi dùng Spy:** Khi muốn test partial behavior của 1 class — ví dụ: test exception handling trong private method.

---

**Q: Cách test exception handling trong @ControllerAdvice?**

```java
@WebMvcTest(ProductController.class)
class ProductControllerExceptionTest {
    @Autowired MockMvc mockMvc;
    @MockBean ProductService productService;

    @Test
    void getProduct_notFound_shouldReturn404() throws Exception {
        // Given
        given(productService.findById(999L))
            .willThrow(new ResourceNotFoundException("Product not found"));

        // When + Then
        mockMvc.perform(get("/api/products/999"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.title").value("Product not found"))
            .andExpect(jsonPath("$.status").value(404));
    }
}
```

---

## 🟠 Chủ Đề 5: Performance & Caching

### Câu hỏi nền tảng

**Q: @Cacheable hoạt động thế nào? Các gotchas (bẫy) thường gặp?**

```java
@Cacheable(
    value = "products",       // cache name
    key = "#id",             // SpEL expression
    condition = "#id > 0",   // chỉ cache khi condition true
    unless = "#result == null" // không cache nếu result null
)
public Product findById(Long id) { ... }

@CacheEvict(value = "products", key = "#product.id")
public void updateProduct(Product product) { ... }

@CachePut(value = "products", key = "#result.id") // luôn update cache
public Product createProduct(CreateProductRequest req) { ... }
```

**Gotchas (bẫy) thường gặp:**
1. **Self-invocation** — gọi @Cacheable method từ cùng class → cache không hoạt động (giống @Transactional)
2. **Complex cache key** — dùng `key = "#request.hashCode()"` nếu argument là object
3. **Cache deserialization** — object trong Redis phải serializable
4. **TTL (Time-to-Live — Thời Gian Sống)** — không set TTL → stale data vĩnh viễn

---

**Q: Cache Aside vs Write-Through vs Write-Behind — Khác nhau thế nào?**

```
Cache Aside (Lazy Loading — Tải Lười Biếng):
  Read: App check cache → miss → load DB → store cache → return
  Write: App update DB → invalidate cache
  Pros: Chỉ cache dữ liệu thực sự được đọc
  Cons: Cache miss đầu tiên chậm; race condition có thể xảy ra

Write-Through (Ghi Xuyên):
  Write: App write cache → cache sync write DB → return
  Pros: Cache luôn consistent với DB
  Cons: Write latency cao hơn

Write-Behind (Ghi Trễ):
  Write: App write cache → return → async flush to DB later
  Pros: Write rất nhanh
  Cons: Data loss risk nếu cache down trước khi flush
```

**Spring @Cacheable = Cache Aside pattern.**

---

### Câu hỏi trung cấp

**Q: Cách debug slow queries trong JPA?**

```yaml
# Bật SQL logging
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        generate_statistics: true  # in thống kê query count
```

```java
// Dùng EXPLAIN ANALYZE để phân tích query plan
@Query(value = "EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = :userId", nativeQuery = true)
List<String> explainQuery(@Param("userId") Long userId);
```

**Công cụ:**
- `p6spy` — log tất cả queries với execution time
- `datasource-proxy` — đếm queries, detect N+1
- `slow_query_log` trong MySQL/PostgreSQL config

---

**Q: Cách phòng tránh Memory Leak (Rò Rỉ Bộ Nhớ) trong Spring Boot?**

Nguyên nhân phổ biến:
1. **Static collections** — lưu data mãi mãi trong static field
2. **HttpSession** — lưu quá nhiều data vào session
3. **ThreadLocal không clean** — đặc biệt trong thread pool
4. **Event listener không unregister** — giữ reference đến bean
5. **Connection leak** — connection HikariCP không được release

```java
// Phát hiện leak với Spring Boot Actuator
management.metrics.enable.jvm=true
# → monitor /actuator/metrics/jvm.memory.used

// Phát hiện HikariCP connection leak
spring.datasource.hikari.leak-detection-threshold=60000
# → warning nếu connection bị giữ > 60s
```

---

## 🔵 Chủ Đề 6: Async & Messaging

### Câu hỏi nền tảng

**Q: @Async hoạt động thế nào? Các lỗi thường gặp?**

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean("taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new CallerRunsPolicy()); // fallback
        executor.initialize();
        return executor;
    }
}

@Service
public class EmailService {
    @Async("taskExecutor") // chạy trên thread pool riêng
    public CompletableFuture<Void> sendEmail(String to, String body) {
        // không block caller thread
        emailClient.send(to, body);
        return CompletableFuture.completedFuture(null);
    }
}
```

**Lỗi thường gặp:**
1. **Self-invocation** — `this.sendEmail()` không async (cùng class)
2. **@Transactional + @Async** — transaction không propagate sang thread mới
3. **Exception handling** — exception trong @Async method bị nuốt → dùng `AsyncUncaughtExceptionHandler`

---

**Q: Kafka Consumer Group (Nhóm Consumer) hoạt động thế nào?**

```
Topic "orders" với 3 partitions:
  Partition 0
  Partition 1
  Partition 2

Consumer Group "order-processors" (3 consumers):
  Consumer A → Partition 0 (exclusive assignment)
  Consumer B → Partition 1
  Consumer C → Partition 2

→ Message trong mỗi partition được xử lý đúng thứ tự
→ 3 consumers xử lý song song (parallel)
→ Nếu Consumer C down → rebalance → Consumer A hoặc B nhận Partition 2
```

**Lưu ý:** Số consumers trong 1 group không nên vượt số partitions — consumer thừa sẽ idle.

---

**Q: Idempotency (Tính Bất Biến) trong message consumer — Tại sao quan trọng?**

```java
// Kafka đảm bảo at-least-once delivery → cùng message có thể nhận 2 lần
// Consumer phải idempotent — xử lý 2 lần = xử lý 1 lần

@KafkaListener(topics = "payment.processed")
public void handlePayment(PaymentEvent event) {
    // ❌ Non-idempotent — trừ tiền 2 lần nếu consume 2 lần!
    wallet.deduct(event.getAmount());
    
    // ✅ Idempotent — kiểm tra đã xử lý chưa
    if (!processedEventRepo.existsById(event.getEventId())) {
        wallet.deduct(event.getAmount());
        processedEventRepo.save(new ProcessedEvent(event.getEventId()));
    }
}
```

---

## ⚫ Chủ Đề 7: Kiến Trúc & Microservices

### Câu hỏi nền tảng

**Q: Bounded Context (Ngữ Cảnh Ràng Buộc) trong DDD là gì?**

Gợi ý trả lời:
- Ranh giới logic nơi một domain model nhất quán và không mâu thuẫn
- Ví dụ: "User" trong bounded context Orders = order history + preferences; "User" trong bounded context Auth = credentials + roles — khác nhau!
- Microservice nên map 1-1 với bounded context

---

**Q: Saga Pattern (Kiểu Mẫu Saga) — Choreography vs Orchestration?**

```
Orchestration Saga (có central coordinator):
  Order Service → calls → Payment Service → calls → Inventory Service
  Order Service biết toàn bộ flow và coordinate

Choreography Saga (không có coordinator):
  Order Service emit "OrderCreated" event
  Payment Service lắng nghe → emit "PaymentProcessed" event
  Inventory Service lắng nghe → emit "InventoryReserved" event
  Mỗi service tự quyết định khi nào roll back
```

**Khi nào chọn:**
- Orchestration: Flow phức tạp, cần rollback rõ ràng, dễ debug
- Choreography: Services độc lập cao, ít coupling, scale tốt hơn

---

**Q: Service Mesh (Lưới Dịch Vụ) — Istio là gì? Khi nào cần?**

Gợi ý trả lời:
- Service Mesh: Infrastructure layer xử lý service-to-service communication
- Tính năng: Load balancing, mTLS (mutual TLS — xác thực hai chiều), circuit breaking, observability
- Khi cần: Nhiều microservices (10+), cần centralized traffic policy, zero-trust networking
- Spring Boot dùng Istio sidecar proxy (Envoy) — application code không cần thay đổi

---

**Q: API Gateway (Cổng API) Pattern — Những tính năng nào nên đặt ở Gateway?**

**NÊN đặt ở Gateway:**
- Rate limiting (giới hạn tần suất)
- Authentication (xác thực) — verify JWT
- SSL termination (kết thúc SSL)
- Request routing (định tuyến)
- Load balancing
- Request/response transformation (biến đổi)
- Logging & tracing (ghi log & theo dõi)

**KHÔNG nên đặt ở Gateway:**
- Business logic (logic nghiệp vụ) — thuộc về service
- Authorization (phân quyền) chi tiết — service tự handle
- Complex aggregation — dùng BFF (Backend for Frontend) pattern riêng

---

## 🟣 Chủ Đề 8: DevOps & Observability

### Câu hỏi nền tảng

**Q: Distributed Tracing (Theo Dõi Phân Tán) — Trace ID vs Span ID?**

```
Request đến từ browser:
  Trace ID: abc-123 (định danh toàn bộ request chain)
  
  API Gateway Span: abc-123 / span-001
    → Order Service Span: abc-123 / span-002 (parent: span-001)
      → DB Query Span: abc-123 / span-003 (parent: span-002)
      → Payment Service Span: abc-123 / span-004 (parent: span-002)
        → Stripe API Span: abc-123 / span-005 (parent: span-004)
```

**Spring Boot cấu hình:**

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 0.1  # sample 10% requests (production)
```

---

**Q: Liveness Probe vs Readiness Probe trong Kubernetes?**

| | Liveness Probe | Readiness Probe |
|-| -------------- | --------------- |
| **Mục đích** | "Pod còn sống không?" | "Pod sẵn sàng nhận traffic chưa?" |
| **Khi fail** | Kubernetes restart pod | Kubernetes tạm ngừng gửi traffic |
| **Endpoint** | `/actuator/health/liveness` | `/actuator/health/readiness` |
| **Fail nếu** | App bị deadlock, crash | DB chưa connect, init chưa xong |

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30    # chờ 30s sau startup trước khi check
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

---

**Q: Blue-Green Deployment (Triển Khai Xanh-Lam) vs Rolling Update (Cập Nhật Cuốn)?**

```
Blue-Green:
  Blue: version 1.0 (đang serve traffic)
  Green: version 1.1 (đã deploy, chưa có traffic)
  Switch: thay đổi load balancer → 100% traffic sang Green
  Rollback: switch lại Blue ngay lập tức
  Downside: Tốn gấp đôi tài nguyên

Rolling Update (Kubernetes default):
  1/3 pods → version 1.1, 2/3 pods vẫn version 1.0
  2/3 pods → version 1.1, 1/3 pods vẫn version 1.0
  3/3 pods → version 1.1
  Downside: Cả 2 versions chạy song song → cần backward compatible
```

---

## 📝 Tự Kiểm Tra — Bài Tập Thực Hành

### Bài 1: Viết REST API hoàn chỉnh (30 phút)

Yêu cầu: Tạo API quản lý Task (công việc) với:
- CRUD endpoints
- Bean Validation trên request body
- Global exception handler trả về ProblemDetail (RFC 7807)
- Unit test cho service layer với Mockito
- @WebMvcTest cho controller

### Bài 2: Debug N+1 Problem (20 phút)

Cho code sau — tìm và fix N+1:

```java
@GetMapping("/api/orders")
public List<OrderDTO> getAllOrders() {
    List<Order> orders = orderRepo.findAll();
    return orders.stream()
        .map(o -> new OrderDTO(
            o.getId(),
            o.getCustomer().getName(), // LAZY load
            o.getItems().size()        // LAZY load collection
        ))
        .toList();
}
```

### Bài 3: Design JWT Authentication Flow (15 phút)

Vẽ sequence diagram (sơ đồ tuần tự) cho:
- Login → nhận access token + refresh token
- Gửi authenticated request
- Access token hết hạn → refresh
- Logout → revoke tokens

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
