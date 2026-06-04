# Bean Lifecycle & Scopes — Vòng Đời và Phạm Vi Bean

> Hiểu Bean Lifecycle giúp bạn debug vấn đề khởi tạo, giải phóng tài nguyên đúng cách,
> và tránh các lỗi tinh tế liên quan đến Scope mismatch (sai phạm vi).

---

## 📋 Mục Tiêu

- [ ] Mô tả đầy đủ **Bean Lifecycle** (Vòng Đời Bean) qua các giai đoạn
- [ ] Dùng `@PostConstruct` và `@PreDestroy` để hook vào lifecycle
- [ ] Phân biệt tất cả **Bean Scopes** (Phạm Vi Bean)
- [ ] Tránh lỗi **Scope Mismatch** (Sai Phạm Vi) khi inject Prototype vào Singleton
- [ ] Giải thích **Eager** vs **Lazy** initialization (Khởi Tạo Sớm vs Trễ)

---

## 1. Bean Lifecycle Tổng Quan

```
Khởi động ứng dụng
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. INSTANTIATION (Khởi Tạo)                                     │
│    Spring tạo instance của Bean bằng constructor                │
│    → new UserService(userRepo, emailService)                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. DEPENDENCY INJECTION (Tiêm Phụ Thuộc)                        │
│    Spring inject các dependencies vào Bean vừa tạo              │
│    → setUserRepository(repo), setEmailService(svc)              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. AWARE INTERFACES (Giao Diện Nhận Thức)                       │
│    Nếu Bean implement BeanNameAware, ApplicationContextAware...  │
│    Spring gọi các setter methods tương ứng                      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. BeanPostProcessor — BEFORE (Trước Khởi Tạo)                  │
│    postProcessBeforeInitialization() được gọi                   │
│    AOP Proxy được tạo ở đây (cho @Transactional, @Async...)     │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. INITIALIZATION (Khởi Tạo Nghiệp Vụ)                          │
│    @PostConstruct method được gọi                               │
│    → Kiểm tra config, kết nối cache, warm-up data               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. BeanPostProcessor — AFTER (Sau Khởi Tạo)                     │
│    postProcessAfterInitialization() được gọi                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
        Bean sẵn sàng sử dụng
        (tồn tại trong suốt vòng đời ứng dụng — Singleton)
                         │
                         ▼ (khi ứng dụng tắt)
┌─────────────────────────────────────────────────────────────────┐
│ 7. DESTRUCTION (Hủy)                                            │
│    @PreDestroy method được gọi                                  │
│    → Đóng kết nối, flush cache, cleanup resources               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. @PostConstruct và @PreDestroy — Hook Vào Lifecycle

### @PostConstruct — Chạy Sau Khi DI Hoàn Thành

```java
@Service
public class CacheWarmupService {

    private final ProductRepository productRepository;
    private final CacheManager cacheManager;
    private Cache productCache;

    public CacheWarmupService(ProductRepository repo, CacheManager cacheManager) {
        this.productRepository = repo;
        this.cacheManager = cacheManager;
        // Lưu ý: KHÔNG thể dùng productCache ở đây vì chưa inject xong
    }

    @PostConstruct
    public void init() {
        // Chạy SAU KHI tất cả dependencies đã được inject
        // An toàn để dùng bất kỳ dependency nào
        productCache = cacheManager.getCache("products");

        // Warm-up cache — load sản phẩm phổ biến vào bộ nhớ khi khởi động
        List<Product> popularProducts = productRepository.findTop100ByOrderBySalesDesc();
        popularProducts.forEach(p -> productCache.put(p.getId(), p));

        log.info("Đã warm-up cache với {} sản phẩm", popularProducts.size());
    }
}
```

### @PreDestroy — Dọn Dẹp Trước Khi Bean Bị Hủy

```java
@Service
public class MessageQueueConsumer {

    private final KafkaConsumer<String, String> consumer;
    private volatile boolean running = true;

    @PostConstruct
    public void startConsuming() {
        // Bắt đầu consumer thread khi Bean khởi tạo
        Thread consumerThread = new Thread(this::consumeMessages, "kafka-consumer");
        consumerThread.start();
        log.info("Kafka consumer đã bắt đầu");
    }

    @PreDestroy
    public void shutdown() {
        // Graceful shutdown — dừng consumer trước khi app tắt
        running = false;
        consumer.wakeup(); // interrupt consumer loop
        log.info("Kafka consumer đã dừng gracefully");
    }

    private void consumeMessages() {
        while (running) {
            // xử lý messages...
        }
    }
}
```

### Ví Dụ Kiểm Tra Cấu Hình

```java
@Service
public class ExternalApiService {

    @Value("${external.api.key}")
    private String apiKey;

    @Value("${external.api.url}")
    private String apiUrl;

    @PostConstruct
    public void validateConfig() {
        // Fail-fast — kiểm tra config ngay khi khởi động
        // thay vì phát hiện lỗi khi gọi API lúc runtime
        if (!StringUtils.hasText(apiKey)) {
            throw new IllegalStateException(
                "external.api.key không được để trống. Kiểm tra application.properties"
            );
        }
        if (!apiUrl.startsWith("https://")) {
            throw new IllegalStateException(
                "external.api.url phải dùng HTTPS: " + apiUrl
            );
        }
        log.info("ExternalApiService đã sẵn sàng kết nối đến: {}", apiUrl);
    }
}
```

---

## 3. Bean Scopes — Phạm Vi Bean

### Singleton Scope (Phạm Vi Đơn Lẻ) — Mặc Định

```java
@Component
// Tương đương @Scope("singleton") hoặc @Scope(ConfigurableBeanFactory.SCOPE_SINGLETON)
public class UserService {
    // Chỉ tạo MỘT instance duy nhất cho toàn bộ ApplicationContext
    // Mọi nơi inject UserService đều nhận cùng một instance
}
```

```
ApplicationContext
    ├── UserController ──────→ UserService (instance #1)
    ├── AdminController ─────→ UserService (instance #1)  ← Cùng một instance!
    └── ReportService ───────→ UserService (instance #1)
```

**Khi nào dùng Singleton:**
- Stateless beans (Bean không có trạng thái) — Service, Repository, Utility
- Cache, Connection Pool — chia sẻ tài nguyên
- Mặc định cho hầu hết các Spring Bean

**Cảnh báo với Singleton:**
```java
@Service // Singleton
public class CounterService {
    private int count = 0; // ❌ NGUY HIỂM — Singleton chia sẻ state giữa các request

    public void increment() {
        count++; // race condition — nhiều thread cùng truy cập
    }
}

// ✅ Đúng: Nếu cần state, dùng thread-safe hoặc Prototype
@Service
public class CounterService {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet(); // thread-safe
    }
}
```

### Prototype Scope (Phạm Vi Nguyên Mẫu)

```java
@Component
@Scope("prototype")
// Hoặc: @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class PdfReportGenerator {
    // Tạo instance MỚI mỗi khi được inject hoặc yêu cầu
    private final List<ReportSection> sections = new ArrayList<>(); // state riêng biệt

    public void addSection(ReportSection section) {
        sections.add(section);
    }

    public byte[] generate() {
        // Tạo PDF từ sections
        return PdfRenderer.render(sections);
    }
}
```

```
ApplicationContext
    ├── ReportService A ──→ PdfReportGenerator (instance #1 — mới tạo)
    └── ReportService B ──→ PdfReportGenerator (instance #2 — mới tạo)
```

**Lưu ý quan trọng:** Spring **không** quản lý `@PreDestroy` cho Prototype bean — bạn phải tự dọn dẹp.

### Web Scopes — Chỉ Dùng Trong Web Application

```java
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {
    // Tạo mới cho MỖI HTTP request, tự động hủy khi request kết thúc
    private String requestId;
    private String userAgent;
    // Rất hữu ích để truyền thông tin xuyên suốt một request
}

@Component
@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserSessionData {
    // Tạo mới cho MỖI HTTP session, tồn tại trong suốt session
    private ShoppingCart cart = new ShoppingCart();
    private UserPreferences preferences;
}

@Component
@Scope(value = WebApplicationContext.SCOPE_APPLICATION, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class AppMetrics {
    // Tương tự Singleton nhưng gắn với ServletContext
    // Tồn tại trong suốt vòng đời của web application
}
```

### Bảng Tóm Tắt Scopes

| Scope | Số Lần Tạo | Thời Gian Tồn Tại | Trường Hợp Dùng |
|-------|-----------|-------------------|-----------------|
| `singleton` | 1 lần | Suốt vòng đời App | Service, Repository, Utility |
| `prototype` | Mỗi lần inject | Đến khi GC thu hồi | Object có state, không chia sẻ |
| `request` | Mỗi HTTP request | Đến hết request | Request tracking, per-request state |
| `session` | Mỗi HTTP session | Đến hết session | Shopping cart, user preferences |
| `application` | 1 lần / ServletContext | Đến khi web app dừng | App-level shared data |

---

## 4. Scope Mismatch — Sai Phạm Vi (Lỗi Tinh Tế)

Đây là một trong những lỗi tinh tế nhất trong Spring!

### Vấn Đề: Inject Prototype Bean vào Singleton Bean

```java
@Component
@Scope("prototype")
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();
    // Prototype — mỗi user nên có cart riêng
}

@Service
// Singleton — chỉ tạo một lần
public class CartService {

    @Autowired
    private final ShoppingCart cart; // ❌ NGUY HIỂM!
    // Cart được inject một lần khi CartService khởi tạo
    // Mặc dù Cart là Prototype, nhưng CartService Singleton giữ reference đến
    // MỘT instance Cart duy nhất → Tất cả user dùng chung một Cart!
}
```

```
CartService (Singleton)
    └── cart → ShoppingCart instance #1  ← cả 1000 user cùng dùng instance này!
                                           (KHÔNG có instance #2, #3...)
```

### Giải Pháp 1: ObjectProvider (Khuyến Nghị)

```java
@Service
public class CartService {

    private final ObjectProvider<ShoppingCart> cartProvider;

    public CartService(ObjectProvider<ShoppingCart> cartProvider) {
        this.cartProvider = cartProvider;
    }

    public void addToCart(Long userId, Item item) {
        // getObject() tạo NEW instance mỗi lần gọi
        ShoppingCart cart = cartProvider.getObject();
        cart.addItem(item);
        // Bây giờ mỗi lần gọi addToCart tạo cart mới — vẫn không đúng nghiệp vụ
        // nhưng ít nhất không share state giữa các call
    }
}
```

### Giải Pháp 2: ApplicationContext.getBean() (Ít dùng)

```java
@Service
public class CartService implements ApplicationContextAware {

    private ApplicationContext context;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.context = ctx;
    }

    public ShoppingCart getNewCart() {
        return context.getBean(ShoppingCart.class); // tạo Prototype mới
    }
}
```

### Giải Pháp 3: ScopedProxy (Cho Web Scopes)

```java
// proxyMode cần thiết khi inject Web-scoped Bean vào Singleton
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {
    private String requestId = UUID.randomUUID().toString();
    // ...
}

@Service // Singleton
public class AuditService {

    private final RequestContext requestContext; // ✅ An toàn nhờ ScopedProxy

    public AuditService(RequestContext requestContext) {
        // Spring inject một PROXY, không phải actual RequestContext
        // Mỗi khi gọi method trên proxy, nó delegate đến RequestContext của request hiện tại
        this.requestContext = requestContext;
    }

    public void log(String action) {
        // requestContext.getRequestId() trả về ID của request đang chạy
        auditLog.record(requestContext.getRequestId(), action);
    }
}
```

---

## 5. Lazy Initialization — Khởi Tạo Trễ

Mặc định, tất cả Singleton Bean được khởi tạo khi ApplicationContext khởi động (**eager initialization** — khởi tạo sớm).

```java
// Lazy Bean — chỉ tạo khi lần đầu được yêu cầu
@Service
@Lazy
public class HeavyReportService {
    // Tốn tài nguyên để khởi tạo — chỉ cần khi có request thực sự
    private final ReportEngine reportEngine;

    @PostConstruct
    public void init() {
        reportEngine = new ReportEngine(); // khởi tạo nặng
    }
}
```

### Lazy Cho Toàn Bộ Application

```yaml
# application.yml
spring:
  main:
    lazy-initialization: true  # Tất cả Bean đều lazy — khởi động nhanh hơn
```

**Cân nhắc:**
- **Eager** (mặc định): Phát hiện lỗi cấu hình sớm (fail-fast), nhưng khởi động chậm hơn
- **Lazy**: Khởi động nhanh (tốt cho development/testing), nhưng lỗi cấu hình chỉ lộ ra khi Bean lần đầu được dùng

**Khuyến nghị:** Dùng Eager cho production, Lazy có thể dùng để tăng tốc development/test startup.

---

## 6. Aware Interfaces — Nhận Thức Về Container

Đôi khi Bean cần biết về Spring Container hoặc chính nó:

```java
@Service
public class AuditService implements
    BeanNameAware,         // Biết tên Bean của mình
    ApplicationContextAware // Biết ApplicationContext
{
    private String beanName;
    private ApplicationContext context;

    @Override
    public void setBeanName(String name) {
        // Spring gọi method này trong lifecycle step 3
        this.beanName = name;
        log.debug("Bean name: {}", name);
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.context = ctx;
        // Có thể dùng context.getBean() để lấy bean khác dynamically
    }
}
```

**Lưu ý:** Dùng Aware Interfaces tạo coupling với Spring framework. Chỉ dùng khi thực sự cần.

---

## 7. BeanPostProcessor — Xử Lý Sau Khi Tạo Bean

`BeanPostProcessor` cho phép can thiệp vào quá trình tạo **mọi** Bean trong Container — đây là cơ chế Spring dùng để implement AOP, @Transactional, @Async.

```java
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // Gọi TRƯỚC @PostConstruct
        log.trace("Đang khởi tạo bean: {}", beanName);
        return bean; // phải trả về bean (hoặc wrapper)
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // Gọi SAU @PostConstruct
        // AOP Proxy được tạo ở đây!
        log.trace("Bean đã sẵn sàng: {}", beanName);
        return bean;
    }
}
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

### Q: Mô tả Bean Lifecycle trong Spring

**Trả lời ngắn gọn (cho phỏng vấn):**
1. **Instantiation** — Spring tạo instance qua constructor
2. **Dependency Injection** — Inject tất cả dependencies
3. **Aware interfaces** — Gọi setBeanName(), setApplicationContext() nếu có
4. **BeanPostProcessor before** — Xử lý trước init (AOP proxy được tạo)
5. **@PostConstruct** — Business initialization
6. **BeanPostProcessor after** — Xử lý sau init
7. **Sẵn sàng** — Bean tồn tại và phục vụ requests
8. **@PreDestroy** — Cleanup khi app shutdown

### Q: Singleton Scope có thread-safe không?

**Trả lời:** Không tự động. Singleton nghĩa là một instance dùng chung, nhưng nhiều thread có thể đồng thời gọi methods trên instance đó. Bean an toàn khi:
1. **Stateless** — Không có instance fields (hoặc chỉ có final fields)
2. **Immutable state** — Fields chỉ đọc, không ghi
3. **Thread-local state** — Dùng `ThreadLocal` cho state per-thread
4. **Synchronized** — Dùng `synchronized`, `AtomicInteger`, concurrent collections

### Q: Prototype Scope khác Singleton thế nào? Khi nào dùng Prototype?

**Trả lời:**
- Singleton: một instance chia sẻ toàn App. Phù hợp cho stateless components
- Prototype: tạo mới mỗi lần inject/getBean. Phù hợp khi cần:
  - Object có **mutable state** không chia sẻ được giữa các caller
  - Object không thread-safe cần mỗi thread có instance riêng
  - Ví dụ: Report generator, builder pattern object, per-request context

---

## ✅ Checklist Tự Kiểm Tra

- [ ] Viết Bean dùng @PostConstruct để warm-up cache hoặc validate config
- [ ] Viết Bean dùng @PreDestroy để đóng resource an toàn
- [ ] Giải thích Scope Mismatch và cách tránh
- [ ] Dùng `@Scope("prototype")` đúng cách với ObjectProvider
- [ ] Phân biệt Eager vs Lazy initialization — khi nào dùng cái nào

---

**Tiếp Theo:** [5-configuration.md](5-configuration.md) — @Value, @ConfigurationProperties, Profiles
