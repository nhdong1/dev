# Spring Events — ApplicationEvent & @EventListener

> Spring Events (Sự Kiện Spring) là cơ chế publish-subscribe (xuất bản - đăng ký) nội bộ trong Spring,
> cho phép các component giao tiếp với nhau mà không cần phụ thuộc trực tiếp —
> thực hiện nguyên tắc loose coupling (kết nối lỏng lẻo) trong ứng dụng.

---

## 1. Tại Sao Cần Spring Events?

### Vấn Đề Với Tight Coupling (Kết Nối Chặt Chẽ)

```java
// ❌ OrderService biết quá nhiều về các service khác
@Service
public class OrderService {

    private final EmailService emailService;
    private final InventoryService inventoryService;
    private final LoyaltyService loyaltyService;
    private final AuditService auditService;

    public Order createOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));

        emailService.sendConfirmation(order);         // Phụ thuộc trực tiếp
        inventoryService.deductStock(order);          // Phụ thuộc trực tiếp
        loyaltyService.addPoints(order);              // Phụ thuộc trực tiếp
        auditService.logOrderCreated(order);          // Phụ thuộc trực tiếp

        return order;
    }
}
```

**Vấn đề:** Thêm một hành động mới (vd: gửi SMS) → phải sửa `OrderService`. Vi phạm Open/Closed Principle (Nguyên Tắc Mở/Đóng).

### Giải Pháp Với Spring Events

```java
// ✅ OrderService chỉ publish event — không biết ai nghe
@Service
public class OrderService {

    private final ApplicationEventPublisher eventPublisher;

    public Order createOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));

        eventPublisher.publishEvent(new OrderCreatedEvent(this, order)); // ← Publish

        return order;
    }
}

// Mỗi concern xử lý riêng — thêm listener mới mà không cần sửa OrderService
@Component public class EmailListener     { @EventListener void on(OrderCreatedEvent e) { ... } }
@Component public class InventoryListener { @EventListener void on(OrderCreatedEvent e) { ... } }
@Component public class LoyaltyListener   { @EventListener void on(OrderCreatedEvent e) { ... } }
```

---

## 2. Tạo Custom Event (Sự Kiện Tùy Chỉnh)

### Cách 1 — Kế Thừa ApplicationEvent (Cũ — Spring < 4.2)

```java
public class OrderCreatedEvent extends ApplicationEvent {

    private final Order order;

    public OrderCreatedEvent(Object source, Order order) {
        super(source); // source = bean publish event
        this.order = order;
    }

    public Order getOrder() {
        return order;
    }
}
```

### Cách 2 — POJO Event (Hiện Đại — Spring 4.2+)

```java
// Không cần kế thừa gì — bất kỳ object nào đều có thể là event
public record OrderCreatedEvent(Long orderId, String customerEmail, BigDecimal amount) {
}

// Hoặc dùng class thông thường
public class UserRegisteredEvent {
    private final Long userId;
    private final String email;
    private final Instant registeredAt;

    // constructor, getters...
}
```

---

## 3. Publish Event (Xuất Bản Sự Kiện)

### Dùng ApplicationEventPublisher

```java
@Service
public class UserService {

    private final ApplicationEventPublisher eventPublisher;
    private final UserRepository userRepository;

    public UserService(ApplicationEventPublisher eventPublisher,
                       UserRepository userRepository) {
        this.eventPublisher = eventPublisher;
        this.userRepository = userRepository;
    }

    @Transactional
    public User registerUser(RegisterRequest request) {
        User user = userRepository.save(User.from(request));

        // Publish sau khi lưu DB thành công
        eventPublisher.publishEvent(new UserRegisteredEvent(user.getId(), user.getEmail()));

        return user;
    }
}
```

### Dùng ApplicationContext (Ít Phổ Biến)

```java
@Service
public class OrderService implements ApplicationContextAware {

    private ApplicationContext context;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.context = ctx;
    }

    public void processOrder(Order order) {
        context.publishEvent(new OrderCreatedEvent(order.getId(), ...));
    }
}
```

---

## 4. @EventListener — Nhận Sự Kiện

### 4.1 Cơ Bản

```java
@Component
public class EmailNotificationListener {

    private final EmailService emailService;

    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Spring tự động match event type với parameter type
        emailService.sendOrderConfirmation(event.customerEmail(), event.orderId());
    }
}
```

### 4.2 Conditional Listener (Listener Có Điều Kiện)

```java
@Component
public class PremiumNotificationListener {

    @EventListener(condition = "#event.amount > 1000000") // SpEL expression
    public void handleLargeOrder(OrderCreatedEvent event) {
        // Chỉ xử lý đơn hàng > 1 triệu
        notifyVipTeam(event.orderId());
    }
}
```

### 4.3 Nhận Nhiều Event Types

```java
@Component
public class AuditListener {

    @EventListener({OrderCreatedEvent.class, OrderCancelledEvent.class})
    public void handleOrderEvents(Object event) {
        if (event instanceof OrderCreatedEvent e) {
            auditService.log("ORDER_CREATED", e.orderId());
        } else if (event instanceof OrderCancelledEvent e) {
            auditService.log("ORDER_CANCELLED", e.orderId());
        }
    }
}
```

### 4.4 Return Value — Publish Event Tiếp Theo

```java
@Component
public class OrderProcessingListener {

    @EventListener
    public ShipmentCreatedEvent handleOrderPaid(OrderPaidEvent event) {
        Shipment shipment = shipmentService.create(event.orderId());

        // Return value tự động publish thành event mới
        return new ShipmentCreatedEvent(shipment.getId(), event.orderId());
    }
}
```

---

## 5. @TransactionalEventListener — Tích Hợp Với Transaction

### Vấn Đề Nghiêm Trọng Với @EventListener + @Transactional

```java
// ❌ Nguy hiểm — race condition
@Transactional
public User registerUser(RegisterRequest request) {
    User user = userRepository.save(request);
    eventPublisher.publishEvent(new UserRegisteredEvent(user.getId()));
    // ← Event được publish NGAY LẬP TỨC, trước khi transaction commit
    // Email listener chạy, query DB nhưng user chưa được commit!
    return user;
}

@EventListener
public void sendWelcomeEmail(UserRegisteredEvent event) {
    User user = userRepository.findById(event.userId()); // ← Có thể trả về null!
    emailService.send(user.getEmail(), "Welcome!");
}
```

### Giải Pháp — @TransactionalEventListener

```java
@Component
public class UserEventListener {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    // Chỉ chạy SAU KHI transaction commit thành công
    public void sendWelcomeEmail(UserRegisteredEvent event) {
        User user = userRepository.findById(event.userId()).orElseThrow();
        emailService.send(user.getEmail(), "Welcome!");
        // ← User chắc chắn đã có trong DB
    }
}
```

### Các Phase (Giai Đoạn) Của TransactionPhase

```java
// Chạy TRƯỚC KHI commit
@TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
public void beforeCommit(UserRegisteredEvent event) { ... }

// Chạy SAU KHI commit thành công (phổ biến nhất)
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void afterCommit(UserRegisteredEvent event) { ... }

// Chạy SAU KHI rollback
@TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
public void afterRollback(UserRegisteredEvent event) {
    // Cleanup, alert, compensating action
}

// Chạy SAU KHI transaction hoàn tất (commit hoặc rollback)
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)
public void afterCompletion(UserRegisteredEvent event) { ... }
```

### Fallback — Khi Không Có Transaction

```java
@TransactionalEventListener(
    phase = TransactionPhase.AFTER_COMMIT,
    fallbackExecution = true  // Chạy ngay cả khi không có transaction active
)
public void handleEvent(SomeEvent event) { ... }
```

---

## 6. Async Event Listener (Listener Bất Đồng Bộ)

### Kết Hợp @Async + @EventListener

```java
@Configuration
@EnableAsync
public class AsyncConfig { }

@Component
public class AsyncOrderListener {

    @Async               // ← Chạy trên thread pool riêng
    @EventListener
    public void sendEmailAsync(OrderCreatedEvent event) {
        // Không chặn thread publish event
        emailService.sendConfirmation(event.customerEmail());
    }
}
```

### Kết Hợp @Async + @TransactionalEventListener

```java
@Component
public class AsyncTransactionalListener {

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void processAfterCommit(UserRegisteredEvent event) {
        // Chạy bất đồng bộ, sau khi commit
        // Best of both worlds!
        integrationService.syncToExternalCRM(event.userId());
    }
}
```

> **Lưu ý:** Method trả về `void` — `@Async` + `@TransactionalEventListener` không hỗ trợ return value.

---

## 7. Application Lifecycle Events (Sự Kiện Vòng Đời Ứng Dụng)

Spring Boot publish sẵn các event về vòng đời ứng dụng:

```java
@Component
public class AppLifecycleListener {

    // Publish khi ApplicationContext được tạo (trước khi beans được init)
    @EventListener
    public void onStarting(ApplicationStartingEvent event) { }

    // ApplicationContext đã được cấu hình nhưng chưa refresh
    @EventListener
    public void onEnvironmentPrepared(ApplicationEnvironmentPreparedEvent event) { }

    // Context đã refresh, beans sẵn sàng, nhưng runner chưa gọi
    @EventListener
    public void onContextRefreshed(ContextRefreshedEvent event) { }

    // Thường dùng nhất — App đã fully started, sẵn sàng nhận request
    @EventListener
    public void onApplicationReady(ApplicationReadyEvent event) {
        log.info("Application started successfully");
        // Warm-up cache, check external dependencies, etc.
    }

    // App đang shutdown
    @EventListener
    public void onContextClosed(ContextClosedEvent event) {
        log.info("Application shutting down — releasing resources");
    }
}
```

---

## 8. Event Ordering (Thứ Tự Xử Lý)

```java
@Component
public class OrderedListeners {

    @Order(1)  // Xử lý đầu tiên
    @EventListener
    public void firstHandler(OrderCreatedEvent event) {
        inventoryService.deductStock(event.orderId()); // Phải làm trước
    }

    @Order(2)  // Xử lý thứ hai
    @EventListener
    public void secondHandler(OrderCreatedEvent event) {
        emailService.sendConfirmation(event.customerEmail());
    }

    @Order(3)  // Xử lý cuối
    @EventListener
    public void lastHandler(OrderCreatedEvent event) {
        analyticsService.trackOrder(event.orderId());
    }
}
```

---

## 9. Testing Spring Events

### Test Event Publishing

```java
@SpringBootTest
class OrderServiceTest {

    @Autowired
    private OrderService orderService;

    @MockBean
    private ApplicationEventPublisher eventPublisher;

    @Test
    void shouldPublishOrderCreatedEvent() {
        // Given
        OrderRequest request = new OrderRequest("item-1", 2);

        // When
        orderService.createOrder(request);

        // Then
        verify(eventPublisher).publishEvent(
            argThat(event -> event instanceof OrderCreatedEvent)
        );
    }
}
```

### Test Event Listener

```java
@SpringBootTest
class EmailListenerTest {

    @Autowired
    private ApplicationEventPublisher eventPublisher;

    @MockBean
    private EmailService emailService;

    @Test
    void shouldSendEmailWhenOrderCreated() {
        // When
        eventPublisher.publishEvent(new OrderCreatedEvent(1L, "user@test.com", BigDecimal.TEN));

        // Then
        verify(emailService).sendOrderConfirmation("user@test.com", 1L);
    }
}
```

---

## 10. Giới Hạn Của Spring Events

| Giới Hạn | Mô Tả | Giải Pháp |
|----------|--------|-----------|
| **Chỉ trong JVM** | Events không vượt qua process boundary | Dùng Kafka/RabbitMQ |
| **Mất khi app crash** | Events không persistent | Message broker với persistence |
| **Synchronous mặc định** | Publisher chờ tất cả listeners xong | `@Async` trên listener |
| **Không retry** | Listener fail → không retry tự động | Implement retry logic thủ công |
| **Không ordering đảm bảo** | Listeners chạy theo order đăng ký | `@Order` annotation |

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa `@EventListener` và `@TransactionalEventListener`?**
> `@EventListener` xử lý sự kiện ngay khi publish, kể cả khi transaction chưa commit → nguy cơ race condition khi listener đọc DB. `@TransactionalEventListener(phase = AFTER_COMMIT)` chỉ xử lý sau khi transaction commit thành công → đảm bảo data nhất quán.

**Q: Khi nào dùng Spring Events thay vì Kafka/RabbitMQ?**
> Spring Events phù hợp khi: (1) Events trong cùng JVM/monolith, (2) Không cần persistence hay retry, (3) Muốn loose coupling đơn giản. Kafka/RabbitMQ khi: (1) Events cross-service, (2) Cần durable/reliable delivery, (3) Cần replay events.

**Q: `@Async` + `@TransactionalEventListener` — cần lưu ý gì?**
> Listener chạy trên thread mới nên không có transaction active. Nếu listener cần transaction → phải annotate `@Transactional` trên listener method. Spring sẽ tạo transaction mới trong thread đó.

**Q: Spring Events có thread-safe không?**
> Mặc định không. Nếu nhiều threads publish cùng lúc và listener dùng shared state, cần đồng bộ hóa. Dùng `@Async` để tách listener sang thread pool riêng và tránh block publisher.
