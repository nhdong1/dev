# Transactions — Quản Lý Giao Dịch với @Transactional

> Transaction (Giao Dịch) đảm bảo tính toàn vẹn dữ liệu theo nguyên tắc ACID.
> `@Transactional` là annotation quan trọng nhất trong Spring Data — hiểu sai cơ chế của nó
> là nguyên nhân hàng đầu gây bug dữ liệu khó tìm trong production.

---

## 📋 Mục Tiêu

- [ ] Hiểu **ACID** (Atomicity, Consistency, Isolation, Durability — Nguyên Tử Tính, Nhất Quán Tính, Cô Lập Tính, Bền Vững Tính)
- [ ] Sử dụng đúng `@Transactional` và các thuộc tính của nó
- [ ] Phân biệt các **Propagation** (Lan Truyền) — `REQUIRED`, `REQUIRES_NEW`, `NESTED`
- [ ] Hiểu các **Isolation Level** (Mức Cô Lập) và khi nào dùng mức nào
- [ ] Tránh **Self-Invocation Problem** (Vấn Đề Tự Gọi) và các pitfalls phổ biến
- [ ] Cấu hình Rollback đúng cách

---

## 1. ACID — Bốn Tính Chất Cơ Bản Của Giao Dịch

```
A — Atomicity (Nguyên Tử Tính):
    "Tất cả hoặc không có gì"
    Ví dụ: Chuyển tiền = Trừ tài khoản A + Cộng tài khoản B
    → Nếu bước 2 thất bại, bước 1 phải rollback

C — Consistency (Nhất Quán Tính):
    "Dữ liệu luôn hợp lệ trước và sau transaction"
    Ví dụ: Tổng tiền trong hệ thống không đổi sau khi chuyển tiền

I — Isolation (Cô Lập Tính):
    "Các transaction song song không nhìn thấy dữ liệu trung gian của nhau"
    Ví dụ: User A đang chuyển tiền, User B đọc số dư → không thấy trạng thái giữa chừng

D — Durability (Bền Vững Tính):
    "Sau khi commit thành công, dữ liệu được lưu vĩnh viễn dù server crash"
    Ví dụ: Giao dịch đã commit → restart server → dữ liệu vẫn còn
```

---

## 2. @Transactional — Cơ Bản

### Cách Hoạt Động (Proxy-Based AOP)

```
Gọi service.transfer()
        │
        ▼
  TransactionInterceptor (Spring AOP Proxy)
        │
        ├── BEGIN TRANSACTION
        │         │
        │         ▼
        │   Code của transfer() thực thi
        │         │
        │    ┌────┴────┐
        │    │ Success?│
        │    └────┬────┘
        │    Yes  │  No (Exception)
        │    ▼    │    ▼
        │ COMMIT   │  ROLLBACK
        │         │
        └─────────┘
```

```java
@Service
@Transactional  // Áp dụng cho toàn bộ class — mọi method đều có transaction
public class TransferService {

    private final AccountRepository accountRepository;
    private final TransactionLogRepository logRepository;

    // Method này có transaction từ class-level @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId)
            .orElseThrow(() -> new AccountNotFoundException(fromId));
        Account to = accountRepository.findById(toId)
            .orElseThrow(() -> new AccountNotFoundException(toId));

        if (from.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException(fromId, amount);
        }

        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));

        // Nếu đây throw exception → toàn bộ transaction rollback
        // → Cả 2 cập nhật đều bị hủy
        logRepository.save(new TransactionLog(fromId, toId, amount));
    }

    // Method-level override class-level — chỉ đọc, không ghi
    @Transactional(readOnly = true)
    public Account findById(Long id) {
        return accountRepository.findById(id).orElseThrow();
    }
}
```

### readOnly = true — Tối Ưu Transaction Chỉ Đọc

```java
// ✅ readOnly = true cho query methods
@Transactional(readOnly = true)
public List<ProductResponse> findAllProducts() {
    return productRepository.findAll()
                            .stream()
                            .map(mapper::toResponse)
                            .toList();
}

/*
 readOnly = true mang lại:
 1. Hibernate tắt dirty checking (kiểm tra thay đổi) → nhanh hơn
 2. DB có thể optimize — route query đến read replica
 3. Không flush session → giảm overhead
 4. Giúp developer nhận ra method này chỉ đọc
*/
```

---

## 3. Propagation — Hành Vi Lan Truyền Giao Dịch

Propagation xác định: "Nếu đã có transaction đang chạy, method mới này sẽ làm gì?"

### Biểu Đồ Propagation

```
REQUIRED (Mặc Định):
    Caller có TX? → Dùng chung TX đó
    Caller không có TX? → Tạo TX mới
    [TX_A] → calls → [REQUIRED method] → cùng TX_A

REQUIRES_NEW:
    Luôn tạo TX mới — tạm dừng TX cha nếu có
    [TX_A] → calls → [REQUIRES_NEW method] → tạo [TX_B], tạm dừng TX_A
                   TX_B commit/rollback độc lập với TX_A

NESTED:
    Nếu có TX cha → tạo savepoint trong TX cha
    → Rollback NESTED chỉ rollback đến savepoint, TX cha tiếp tục
    [TX_A] → calls → [NESTED method] → savepoint S1 trong TX_A

SUPPORTS:
    Caller có TX? → Tham gia TX đó
    Caller không có TX? → Chạy KHÔNG có TX (không tạo mới)

NOT_SUPPORTED:
    Luôn chạy KHÔNG có TX — tạm dừng TX cha nếu có

NEVER:
    KHÔNG bao giờ chạy với TX — ném exception nếu có TX đang chạy

MANDATORY:
    BẮT BUỘC phải có TX đang chạy — ném exception nếu không có
```

### Ví Dụ REQUIRED — Dùng Chung Transaction

```java
@Service
public class OrderService {

    @Transactional  // TX_A bắt đầu
    public void createOrder(OrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);             // Trong TX_A

        inventoryService.reserveStock(request);  // Cũng trong TX_A (REQUIRED)
        paymentService.initiatePayment(request); // Cũng trong TX_A

        // Nếu bất kỳ step nào throw exception → toàn bộ TX_A rollback
    }
}

@Service
public class InventoryService {

    @Transactional(propagation = Propagation.REQUIRED) // Mặc định
    public void reserveStock(OrderRequest request) {
        // Không tạo TX mới — dùng TX_A của OrderService
        inventoryRepository.decreaseStock(request.getProductId(), request.getQuantity());
    }
}
```

### Ví Dụ REQUIRES_NEW — Ghi Log Audit Độc Lập

```java
@Service
public class OrderService {

    @Transactional  // TX_A
    public void processOrder(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();

        try {
            paymentService.charge(order);
            order.setStatus(OrderStatus.PAID);
        } catch (PaymentException e) {
            order.setStatus(OrderStatus.PAYMENT_FAILED);
            // TX_A sẽ rollback order status change
        } finally {
            // Ghi audit log DÙ thành công hay thất bại
            // REQUIRES_NEW → log được lưu dù TX_A rollback
            auditService.log("ORDER_PROCESSED", orderId, order.getStatus());
        }
    }
}

@Service
public class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(String event, Long entityId, Object data) {
        // TX_B mới — độc lập với TX_A
        // TX_A rollback cũng không ảnh hưởng đến audit log
        auditLogRepository.save(new AuditLog(event, entityId, data.toString()));
    }
}
```

### Ví Dụ NESTED — Savepoint Trong Transaction

```java
@Service
public class BatchImportService {

    @Transactional  // TX cha
    public ImportResult importProducts(List<ProductDto> products) {
        int success = 0;
        int failed = 0;

        for (ProductDto dto : products) {
            try {
                importSingleProduct(dto);  // NESTED — savepoint
                success++;
            } catch (Exception e) {
                // NESTED rollback về savepoint — TX cha tiếp tục
                // Product lỗi bị rollback, các product khác vẫn OK
                failed++;
                log.warn("Skip product {}: {}", dto.getSku(), e.getMessage());
            }
        }
        return new ImportResult(success, failed);
    }

    @Transactional(propagation = Propagation.NESTED)
    public void importSingleProduct(ProductDto dto) {
        // Tạo savepoint — nếu method này fail:
        // → Chỉ rollback đến savepoint
        // → TX cha không bị ảnh hưởng
        Product product = mapper.toEntity(dto);
        productRepository.save(product);
        // Các thao tác khác...
    }
}
```

---

## 4. Isolation Levels — Mức Cô Lập Giao Dịch

### Các Vấn Đề Cần Giải Quyết

```
Dirty Read (Đọc Bẩn):
    TX_A đọc dữ liệu TX_B đang ghi nhưng TX_B chưa commit
    → TX_B rollback → TX_A đọc phải dữ liệu "không có thật"

Non-Repeatable Read (Đọc Không Lặp Lại):
    TX_A đọc dòng 2 lần → kết quả khác nhau vì TX_B đã UPDATE ở giữa

Phantom Read (Đọc Ma):
    TX_A đếm số dòng 2 lần → kết quả khác nhau vì TX_B đã INSERT ở giữa

Lost Update (Mất Bản Ghi Cập Nhật):
    TX_A và TX_B cùng đọc rồi ghi → TX_A ghi sau, ghi đè TX_B
```

### Bảng So Sánh Isolation Levels

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Hiệu Năng |
|----------------|------------|---------------------|--------------|-----------|
| `READ_UNCOMMITTED` | ✅ Có thể | ✅ Có thể | ✅ Có thể | Nhanh nhất |
| `READ_COMMITTED` | ❌ Không | ✅ Có thể | ✅ Có thể | Nhanh |
| `REPEATABLE_READ` | ❌ Không | ❌ Không | ✅ Có thể | Trung bình |
| `SERIALIZABLE` | ❌ Không | ❌ Không | ❌ Không | Chậm nhất |

> PostgreSQL mặc định: `READ_COMMITTED`
> MySQL (InnoDB) mặc định: `REPEATABLE_READ`

### Ví Dụ Các Isolation Level

```java
// READ_COMMITTED (Mặc định của hầu hết DB)
// Chỉ đọc dữ liệu đã COMMIT — không đọc dirty data
@Transactional(isolation = Isolation.READ_COMMITTED)
public ProductReport generateReport() {
    // Phù hợp cho hầu hết use case
    return reportService.buildReport();
}

// REPEATABLE_READ — Đọc cùng dữ liệu trong suốt TX
// Dùng khi cần đảm bảo số liệu nhất quán trong quá trình tính toán
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void reconcileAccounts(Long accountId) {
    // Lần đọc 1 và lần đọc 2 phải trả về cùng kết quả
    BigDecimal balance = accountRepository.getBalance(accountId); // = 1000
    // ... xử lý mất thời gian ...
    BigDecimal balanceCheck = accountRepository.getBalance(accountId); // Vẫn = 1000
    // (dù TX khác đã thay đổi trong lúc này)
}

// SERIALIZABLE — TX chạy tuần tự như thể một mình
// Chỉ dùng khi absolutely necessary — rất chậm, nhiều deadlock
@Transactional(isolation = Isolation.SERIALIZABLE)
public void processFinancialSettlement() {
    // Tính toán tài chính quan trọng — không chấp nhận bất kỳ anomaly nào
}

// DEFAULT — Dùng isolation mặc định của DB (thường là READ_COMMITTED)
@Transactional(isolation = Isolation.DEFAULT)
public void normalOperation() { }
```

---

## 5. Rollback — Cấu Hình Rollback

### Mặc Định — Chỉ Rollback Với RuntimeException

```java
// Mặc định của @Transactional:
// ✅ Rollback với: RuntimeException và Error
// ❌ KHÔNG rollback với: Checked Exception (Exception không phải Runtime)

@Transactional
public void processOrder(Long orderId) throws OrderException { // OrderException là Checked
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.process();
    orderRepository.save(order);

    if (order.isInvalid()) {
        throw new OrderException("Invalid");  // ← KHÔNG rollback theo mặc định!
    }
}
```

### Cấu Hình Rollback Tùy Chỉnh

```java
// Rollback với cả Checked Exception
@Transactional(rollbackFor = {OrderException.class, PaymentException.class})
public void processOrder(Long orderId) throws OrderException {
    // Nếu OrderException hay PaymentException → rollback
}

// Rollback với tất cả Exception
@Transactional(rollbackFor = Exception.class)
public void criticalOperation() throws Exception {
    // Bất kỳ exception nào → rollback
}

// KHÔNG rollback với exception cụ thể
@Transactional(noRollbackFor = {BusinessValidationException.class})
public void validateAndProcess(Order order) {
    // BusinessValidationException → commit thay vì rollback
    // Dùng khi muốn ghi lại lỗi validation vào DB dù có exception
}
```

### Manual Rollback — Rollback Thủ Công

```java
@Transactional
public void processWithConditionalRollback(Order order) {
    orderRepository.save(order);

    if (order.requiresManualReview()) {
        // Đánh dấu rollback nhưng không throw exception
        // Transaction sẽ rollback khi method kết thúc
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
        return; // Không throw — nhưng transaction sẽ rollback
    }

    // Tiếp tục xử lý bình thường
    paymentService.charge(order);
}
```

---

## 6. Self-Invocation Problem — Vấn Đề Tự Gọi

Đây là pitfall nguy hiểm nhất của `@Transactional`. AOP proxy không hoạt động khi method gọi method khác trong cùng class!

```java
@Service
public class OrderService {

    // ❌ SAI — transaction của sendNotification() KHÔNG hoạt động!
    @Transactional
    public void createOrder(OrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);

        // Gọi method trong cùng class → gọi trực tiếp, bỏ qua proxy!
        // @Transactional(propagation = REQUIRES_NEW) của sendNotification KHÔNG có tác dụng
        this.sendNotification(order);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendNotification(Order order) {
        // Tưởng là TX mới nhưng thực ra vẫn chạy trong TX của createOrder!
        notificationRepository.save(new Notification(order));
    }
}
```

```
Lý do:
    Spring tạo Proxy bọc quanh OrderService bean.
    Client gọi proxy.createOrder() → proxy intercept → tạo transaction → gọi target.createOrder()
    Bên trong createOrder(), "this.sendNotification()" gọi TRỰC TIẾP target object,
    KHÔNG qua proxy → AOP không intercept → không có transaction mới!
```

### Giải Pháp 1 — Inject Chính Mình (Self-Injection)

```java
@Service
public class OrderService {

    @Autowired
    @Lazy  // Lazy để tránh circular dependency khi khởi động
    private OrderService self;

    @Transactional
    public void createOrder(OrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);

        // Gọi qua self (proxy) → AOP hoạt động đúng
        self.sendNotification(order);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendNotification(Order order) {
        notificationRepository.save(new Notification(order));
    }
}
```

### Giải Pháp 2 — Tách Thành Class Riêng (Khuyến Nghị)

```java
@Service
public class OrderService {

    private final NotificationService notificationService;

    @Transactional
    public void createOrder(OrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);

        // Gọi bean khác → qua proxy → hoạt động đúng
        notificationService.sendOrderNotification(order);
    }
}

@Service
public class NotificationService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendOrderNotification(Order order) {
        notificationRepository.save(new Notification(order));
    }
}
```

### Giải Pháp 3 — AopContext (Ít Dùng)

```java
// Cần @EnableAspectJAutoProxy(exposeProxy = true) trong config
@Transactional
public void createOrder(OrderRequest request) {
    Order order = new Order(request);
    orderRepository.save(order);

    // Lấy proxy hiện tại từ AopContext
    ((OrderService) AopContext.currentProxy()).sendNotification(order);
}
```

---

## 7. Transaction Timeout — Giới Hạn Thời Gian

```java
// Nếu transaction chạy quá 30 giây → ném TransactionTimedOutException → rollback
@Transactional(timeout = 30) // Đơn vị: giây
public void processLargeReport() {
    // Xử lý dữ liệu lớn — phải hoàn thành trong 30 giây
}

// Application-wide default — application.yml
// spring.transaction.default-timeout=30
```

---

## 8. @TransactionalEventListener — Sự Kiện Trong Transaction

```java
// Publisher — phát sự kiện trong transaction
@Service
public class OrderService {

    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public Order createOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));

        // Phát sự kiện — TRONG transaction
        eventPublisher.publishEvent(new OrderCreatedEvent(order.getId()));

        return order;
    }
}

// Listener — lắng nghe sự kiện SAU KHI transaction commit
@Component
public class OrderEventListener {

    // AFTER_COMMIT: Chỉ xử lý khi TX commit thành công
    // → Đảm bảo email chỉ gửi khi order thực sự được lưu vào DB
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderCreated(OrderCreatedEvent event) {
        emailService.sendOrderConfirmation(event.getOrderId());
    }

    // AFTER_ROLLBACK: Xử lý khi TX rollback — dùng để ghi log, alert
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void onOrderFailed(OrderCreatedEvent event) {
        log.error("Order creation failed for orderId: {}", event.getOrderId());
        alertService.notify("Order creation failed", event.getOrderId());
    }

    // AFTER_COMPLETION: Xử lý bất kể commit hay rollback
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)
    public void onOrderProcessed(OrderCreatedEvent event) {
        cacheService.invalidate("order:" + event.getOrderId());
    }

    // BEFORE_COMMIT: Trước khi commit — vẫn trong transaction
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    @Transactional(propagation = Propagation.MANDATORY) // Phải có TX đang chạy
    public void beforeOrderCommit(OrderCreatedEvent event) {
        auditRepository.save(new AuditEntry(event.getOrderId()));
    }
}
```

---

## 9. Distributed Transaction — Giao Dịch Phân Tán

Khi một use case phải thao tác trên nhiều database hoặc nhiều service, `@Transactional` thông thường không đủ.

### Saga Pattern — Giải Pháp Microservices

```
Choreography-based Saga (Saga Dựa Trên Dàn Hợp Xướng):
    OrderService → event → InventoryService → event → PaymentService → event → ...
    Mỗi service lắng nghe event và tự xử lý bước tiếp theo
    Nếu fail: phát compensating event (sự kiện bù trừ) để undo các bước trước

Orchestration-based Saga (Saga Dựa Trên Điều Phối):
    SagaOrchestrator → gọi tuần tự InventoryService, PaymentService, ShippingService
    Nếu bước nào fail → Orchestrator gọi compensating action cho các bước đã thành công
```

```java
// Ví dụ đơn giản: Outbox Pattern (Mẫu Hộp Thư Gửi)
// Đảm bảo event được gửi khi transaction commit
@Transactional
public Order createOrder(OrderRequest request) {
    Order order = orderRepository.save(new Order(request));

    // Lưu event vào outbox TRONG CÙNG transaction với order
    // → Đảm bảo atomic: hoặc cả order và event được lưu, hoặc cả hai không
    OutboxEvent event = OutboxEvent.builder()
        .aggregateType("ORDER")
        .aggregateId(order.getId().toString())
        .eventType("ORDER_CREATED")
        .payload(objectMapper.writeValueAsString(order))
        .build();
    outboxRepository.save(event);

    return order;
    // Sau khi commit: background job đọc outbox → gửi event đến Kafka
}
```

---

## 10. Transaction Best Practices — Thực Tiễn Tốt Nhất

```java
// ✅ 1. Để @Transactional ở Service layer, KHÔNG phải Controller hay Repository
@Service
@Transactional(readOnly = true) // Class-level: mặc định readOnly
public class ProductService {

    @Transactional // Override: ghi → không readOnly
    public Product create(ProductRequest request) { ... }

    // Inherited: readOnly = true
    public Product findById(Long id) { ... }
}

// ✅ 2. Không catch exception bên trong @Transactional (sẽ không rollback)
@Transactional
public void badPattern() {
    try {
        riskyOperation();
    } catch (Exception e) {
        log.error("Error", e);
        // ❌ Transaction KHÔNG rollback — exception bị nuốt!
    }
}

// ✅ 3. Giữ transaction ngắn — chỉ bao gồm DB operations
@Transactional
public void goodPattern(OrderRequest request) {
    // Không gọi external API trong transaction!
    // Không gửi email trong transaction!
    Order order = orderRepository.save(new Order(request));
    inventoryRepository.reserve(request.getItems());
    // Kết thúc transaction nhanh nhất có thể
}

// Gọi external service SAU khi transaction commit
public void processOrder(OrderRequest request) {
    Order order = createOrderTransactionally(request); // TX commit ở đây
    emailService.sendConfirmation(order);              // Sau TX — không block DB connection
}

// ✅ 4. Không dùng @Transactional trên private method
// Spring AOP không thể wrap private methods
private void privateMethod() { ... } // @Transactional không có tác dụng!

// ✅ 5. Cẩn thận với lazy loading sau khi ra ngoài transaction
@Transactional(readOnly = true)
public Order findOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
}

// ❌ SAI — LazyInitializationException!
Order order = orderService.findOrder(1L);
order.getItems().size(); // Session đã đóng → không thể lazy load!
```

---

## ✅ Checklist @Transactional

- [ ] `readOnly = true` cho tất cả query-only methods
- [ ] Không catch exception mà không rethrow trong `@Transactional`
- [ ] Không gọi external API (HTTP, email, SMS) bên trong transaction
- [ ] Hiểu Self-Invocation problem — tách method ra class riêng
- [ ] Cấu hình `rollbackFor` khi dùng Checked Exception
- [ ] Dùng `REQUIRES_NEW` cho audit log — phải độc lập với TX cha
- [ ] Đặt timeout hợp lý cho transactions dài

---

## 💡 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: `@Transactional` hoạt động thế nào?**
> Spring dùng AOP Proxy để bọc bean. Khi method được gọi, proxy intercept → bắt đầu transaction → gọi method thực → commit hoặc rollback dựa trên kết quả.

**Q: Khi nào nên dùng `REQUIRES_NEW`?**
> Khi cần operation độc lập với TX cha — ví dụ: audit log, ghi lỗi vào DB. TX_B commit/rollback không phụ thuộc TX_A.

**Q: Tại sao Checked Exception không rollback?**
> Thiết kế của Spring theo triết lý EJB cũ: Checked Exception = lỗi dự đoán được, đã được xử lý; RuntimeException = lỗi không mong đợi. Override bằng `rollbackFor = Exception.class`.

**Q: `@Transactional` trên `private` method có hoạt động không?**
> Không. Spring AOP chỉ wrap public methods. Giải pháp: chuyển thành public hoặc tách ra class riêng.

---

## 🔗 Liên Kết

- [2-orm-mapping.md](2-orm-mapping.md) — CascadeType với transaction
- [4-n-plus-one-problem.md](4-n-plus-one-problem.md) — LazyInitializationException sau TX
- [Spring Transaction Reference](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction)
- [Vlad Mihalcea — @Transactional Pitfalls](https://vladmihalcea.com/spring-transactional-annotation/)

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
