# Event-Driven Architecture — CQRS, Event Sourcing & Saga

> Event-Driven Architecture (Kiến Trúc Hướng Sự Kiện) — hệ thống giao tiếp thông qua việc phát và tiêu thụ sự kiện (events). CQRS (Command Query Responsibility Segregation — Tách Biệt Trách Nhiệm Lệnh và Truy Vấn), Event Sourcing (Nguồn Sự Kiện), và Saga Pattern (Mẫu Saga) là ba patterns cốt lõi.

---

## 📋 Mục Lục

1. [Event-Driven Architecture Tổng Quan](#event-driven-architecture-tổng-quan)
2. [CQRS — Tách Biệt Lệnh và Truy Vấn](#cqrs--tách-biệt-lệnh-và-truy-vấn)
3. [Event Sourcing — Nguồn Sự Kiện](#event-sourcing--nguồn-sự-kiện)
4. [Saga Pattern — Giao Dịch Phân Tán](#saga-pattern--giao-dịch-phân-tán)
5. [Triển Khai Với Spring Boot & Kafka](#triển-khai-với-spring-boot--kafka)
6. [Outbox Pattern](#outbox-pattern)
7. [Trade-offs & Khi Nào Dùng](#trade-offs--khi-nào-dùng)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Event-Driven Architecture Tổng Quan

### Từ Synchronous Sang Event-Driven

```
SYNCHRONOUS (Đồng Bộ) — Request/Response:

Client ──POST /orders──► Order Service ──HTTP──► Inventory Service
                                        ◄──────── (response)
                         ◄──────────────────────── (response)

Vấn Đề:
❌ Tight coupling — Order Service phải biết về Inventory Service
❌ Cascade failures — Inventory down → Order không tạo được
❌ Blocking — Thread bị giữ trong khi đợi response
❌ Khó thêm consumer mới (ví dụ: Email Service cũng cần biết order tạo)
```

```
EVENT-DRIVEN (Hướng Sự Kiện):

Client ──POST /orders──► Order Service
                         │ (lưu vào DB)
                         │ PUBLISH → OrderCreatedEvent → Kafka Topic
                         │
                         ◄──── Response ngay lập tức (order đã lưu)

Sau đó, async (bất đồng bộ):
Kafka → Inventory Service (consume OrderCreatedEvent → reserve stock)
     → Email Service     (send confirmation email)
     → Analytics Service (update metrics)

Ưu Điểm:
✅ Loose coupling — services không biết nhau
✅ Resilience — một consumer fail không ảnh hưởng producer
✅ Scalability — dễ thêm consumer mới
✅ Non-blocking — producer không đợi
✅ Natural audit log (nhật ký kiểm toán tự nhiên)
```

### Các Loại Event

```java
// Domain Event (Sự Kiện Miền) — phản ánh thay đổi trong business domain
public record OrderCreatedEvent(
    String eventId,         // Unique ID của event
    String orderId,
    String userId,
    List<OrderItem> items,
    BigDecimal totalAmount,
    Instant occurredAt      // Thời điểm xảy ra sự kiện
) implements DomainEvent {}

// Integration Event (Sự Kiện Tích Hợp) — giao tiếp giữa các bounded contexts
public record PaymentProcessedEvent(
    String eventId,
    String orderId,
    String paymentId,
    PaymentStatus status,
    BigDecimal amount,
    Instant processedAt
) {}

// Event Notification (Thông Báo Sự Kiện) — nhẹ, chứa ít data
// Consumer cần query thêm nếu cần chi tiết
public record OrderStatusChangedNotification(
    String orderId,
    OrderStatus newStatus,
    Instant changedAt
) {}
```

---

## CQRS — Tách Biệt Lệnh và Truy Vấn

### CQRS Là Gì?

CQRS — Command Query Responsibility Segregation (Tách Biệt Trách Nhiệm Lệnh và Truy Vấn) chia hệ thống thành hai phần:

- **Command (Lệnh):** Thay đổi state — Create, Update, Delete. Không trả về data (chỉ trả kết quả success/fail).
- **Query (Truy Vấn):** Đọc state — không thay đổi gì. Trả về data.

```
TRADITIONAL (Truyền Thống):
     ┌─────────────┐
     │   Service   │ ← Cùng model, cùng DB cho cả read và write
     └──────┬──────┘
            │
     ┌──────▼──────┐
     │  Database   │
     └─────────────┘

CQRS:
Write Side (Phía Ghi):                Read Side (Phía Đọc):
┌──────────────┐                      ┌──────────────────────┐
│  Command     │                      │   Query Service       │
│  Handler     │                      │   (Tối Ưu Cho Đọc)   │
└──────┬───────┘                      └──────────┬───────────┘
       │ writes                                   │ reads
┌──────▼───────┐      Sync Events      ┌──────────▼───────────┐
│ Write DB     │ ────────────────────► │ Read DB              │
│ (normalized) │                       │ (denormalized /       │
│ PostgreSQL   │                       │  read-optimized)      │
└──────────────┘                       │ PostgreSQL View /     │
                                       │ Elasticsearch / Redis │
                                       └──────────────────────┘
```

### Triển Khai CQRS Cơ Bản

```java
// === COMMAND SIDE ===

// Command Object (Đối Tượng Lệnh) — immutable, mô tả ý định
public record CreateOrderCommand(
    String userId,
    List<OrderItemCommand> items,
    String shippingAddress
) {}

// Command Handler (Bộ Xử Lý Lệnh) — xử lý command, thay đổi state
@Service
@Transactional
public class CreateOrderCommandHandler {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    public OrderId handle(CreateOrderCommand command) {
        // Validate business rules
        validateOrder(command);

        // Create domain object
        Order order = Order.create(
            UserId.of(command.userId()),
            mapItems(command.items()),
            command.shippingAddress()
        );

        // Persist
        orderRepository.save(order);

        // Publish domain events
        order.getDomainEvents().forEach(eventPublisher::publishEvent);

        return order.getId();
    }
}

// === QUERY SIDE ===

// Query Object (Đối Tượng Truy Vấn)
public record GetOrdersByUserQuery(String userId, int page, int size) {}

// Query Handler (Bộ Xử Lý Truy Vấn) — chỉ đọc, không thay đổi state
@Service
@Transactional(readOnly = true)
public class GetOrdersByUserQueryHandler {

    // Có thể dùng read-optimized repository, hoặc thậm chí native SQL
    private final OrderReadRepository readRepository;

    public Page<OrderSummaryView> handle(GetOrdersByUserQuery query) {
        Pageable pageable = PageRequest.of(query.page(), query.size());
        return readRepository.findSummariesByUserId(query.userId(), pageable);
    }
}

// Read Model (Mô Hình Đọc) — denormalized, tối ưu cho query
// Không phải domain entity — chỉ là projection
public class OrderSummaryView {
    private String orderId;
    private String userId;
    private String userEmail;       // denormalized từ User service
    private String status;
    private BigDecimal totalAmount;
    private int itemCount;
    private String createdAt;
}

// Controller dùng Command và Query handlers riêng biệt
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    private final CreateOrderCommandHandler createHandler;
    private final GetOrdersByUserQueryHandler queryHandler;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Map<String, String> createOrder(@Valid @RequestBody CreateOrderRequest request) {
        CreateOrderCommand command = new CreateOrderCommand(
            request.userId(), request.items(), request.shippingAddress()
        );
        OrderId orderId = createHandler.handle(command);
        return Map.of("orderId", orderId.value().toString());
    }

    @GetMapping
    public Page<OrderSummaryView> getUserOrders(
            @RequestParam String userId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return queryHandler.handle(new GetOrdersByUserQuery(userId, page, size));
    }
}
```

### CQRS Với Separate Read Store

```java
// Khi Read Model được lưu ở Elasticsearch (tối ưu cho search)
@Component
public class OrderReadModelProjection {

    private final OrderSearchRepository elasticsearchRepo; // Elasticsearch

    @EventListener
    public void on(OrderCreatedEvent event) {
        // Update Read Model khi có Domain Event
        OrderDocument doc = OrderDocument.builder()
            .orderId(event.orderId())
            .userId(event.userId())
            .status("PENDING")
            .totalAmount(event.totalAmount())
            .items(event.items())
            .createdAt(event.occurredAt())
            .build();
        elasticsearchRepo.save(doc);
    }

    @EventListener
    public void on(OrderStatusChangedEvent event) {
        elasticsearchRepo.findById(event.orderId())
            .ifPresent(doc -> {
                doc.setStatus(event.newStatus().name());
                elasticsearchRepo.save(doc);
            });
    }
}
```

---

## Event Sourcing — Nguồn Sự Kiện

### Event Sourcing Là Gì?

Thay vì lưu trạng thái hiện tại (current state), lưu toàn bộ sequence of events (chuỗi sự kiện) dẫn đến trạng thái đó.

```
TRADITIONAL STATE STORAGE (Lưu Trạng Thái Truyền Thống):
┌─────────────────────────────────────┐
│ orders table                        │
├──────────┬────────────┬─────────────┤
│ order_id │ status     │ total_amount │
├──────────┼────────────┼─────────────┤
│ 001      │ DELIVERED  │ 500,000     │  ← Chỉ biết trạng thái cuối
└──────────┴────────────┴─────────────┘

EVENT SOURCING (Nguồn Sự Kiện):
┌─────────────────────────────────────────────────────────┐
│ order_events table (Event Store — Kho Sự Kiện)         │
├──────────┬────────────────────────┬─────────────────────┤
│ order_id │ event_type             │ event_data          │
├──────────┼────────────────────────┼─────────────────────┤
│ 001      │ OrderCreated           │ {items, userId}     │
│ 001      │ PaymentProcessed       │ {paymentId, amount} │
│ 001      │ StockReserved          │ {warehouseId}       │
│ 001      │ OrderShipped           │ {trackingNumber}    │
│ 001      │ OrderDelivered         │ {deliveredAt}       │
└──────────┴────────────────────────┴─────────────────────┘

→ Trạng thái hiện tại = Replay tất cả events
→ Full audit trail (Nhật Ký Kiểm Toán Đầy Đủ)
→ Có thể rewind về bất kỳ thời điểm nào
```

### Event Store Implementation

```java
// Domain Event interface
public interface DomainEvent {
    String getEventId();
    String getAggregateId();
    String getAggregateType();
    Instant getOccurredAt();
    int getVersion();
}

// Event Store (Kho Sự Kiện)
@Entity
@Table(name = "domain_events")
public class EventStoreEntry {
    @Id
    private String eventId;
    private String aggregateId;
    private String aggregateType;
    private String eventType;
    @Column(columnDefinition = "jsonb")
    private String payload;         // JSON serialized event
    private int version;            // Version trong aggregate
    private Instant occurredAt;
}

@Repository
public interface EventStoreRepository extends JpaRepository<EventStoreEntry, String> {
    List<EventStoreEntry> findByAggregateIdOrderByVersionAsc(String aggregateId);
    List<EventStoreEntry> findByAggregateIdAndVersionGreaterThanOrderByVersionAsc(
        String aggregateId, int fromVersion);
}

// Aggregate với Event Sourcing
public class Order extends AggregateRoot {

    private OrderId id;
    private OrderStatus status;
    private List<OrderItem> items = new ArrayList<>();
    private int version;

    // Khởi tạo từ events (Reconstruction — Tái Tạo Trạng Thái)
    public static Order reconstitute(List<DomainEvent> events) {
        Order order = new Order();
        events.forEach(order::apply);
        return order;
    }

    // Command methods — phát event, không thay đổi state trực tiếp
    public void create(UserId userId, List<OrderItem> items) {
        applyAndRecord(new OrderCreatedEvent(
            EventId.generate(),
            this.id.value().toString(),
            userId.value(),
            items,
            Instant.now(),
            this.version + 1
        ));
    }

    public void processPayment(PaymentId paymentId) {
        if (this.status != OrderStatus.PENDING) {
            throw new IllegalStateException("Chỉ xử lý thanh toán cho đơn PENDING");
        }
        applyAndRecord(new PaymentProcessedEvent(
            EventId.generate(),
            this.id.value().toString(),
            paymentId.value(),
            Instant.now(),
            this.version + 1
        ));
    }

    // Apply methods — thay đổi state dựa trên event (idempotent)
    private void apply(DomainEvent event) {
        if (event instanceof OrderCreatedEvent e) {
            this.id = OrderId.of(e.getAggregateId());
            this.status = OrderStatus.PENDING;
            this.items = new ArrayList<>(e.getItems());
            this.version = e.getVersion();
        } else if (event instanceof PaymentProcessedEvent e) {
            this.status = OrderStatus.PAID;
            this.version = e.getVersion();
        } else if (event instanceof OrderDeliveredEvent e) {
            this.status = OrderStatus.DELIVERED;
            this.version = e.getVersion();
        }
    }

    private void applyAndRecord(DomainEvent event) {
        apply(event);
        recordEvent(event); // Thêm vào danh sách pending events
    }
}
```

### Snapshot Pattern (Mẫu Ảnh Chụp)

```java
// Vấn Đề: Aggregate có 10,000 events → replay rất chậm
// Giải Pháp: Snapshot (Ảnh Chụp) trạng thái tại một version nhất định

@Entity
@Table(name = "aggregate_snapshots")
public class AggregateSnapshot {
    @Id
    private String aggregateId;
    private String aggregateType;
    @Column(columnDefinition = "jsonb")
    private String state;           // Serialized state
    private int version;            // Version tại thời điểm snapshot
    private Instant snapshotAt;
}

@Service
public class EventSourcedOrderRepository {

    private final EventStoreRepository eventStore;
    private final SnapshotRepository snapshotRepo;
    private final EventSerializer serializer;

    public Order findById(OrderId id) {
        // 1. Load snapshot mới nhất
        Optional<AggregateSnapshot> snapshot = snapshotRepo.findById(id.value().toString());

        int fromVersion = 0;
        Order order;

        if (snapshot.isPresent()) {
            // Khôi phục từ snapshot
            order = serializer.deserialize(snapshot.get().getState(), Order.class);
            fromVersion = snapshot.get().getVersion();
        } else {
            order = new Order();
        }

        // 2. Load events sau snapshot version
        List<EventStoreEntry> entries = eventStore
            .findByAggregateIdAndVersionGreaterThanOrderByVersionAsc(
                id.value().toString(), fromVersion);

        // 3. Replay events
        List<DomainEvent> events = entries.stream()
            .map(entry -> serializer.deserialize(entry.getPayload(), DomainEvent.class))
            .toList();

        events.forEach(order::apply);
        return order;
    }

    @Transactional
    public void save(Order order) {
        // Lưu pending events vào Event Store
        List<EventStoreEntry> entries = order.getPendingEvents().stream()
            .map(event -> new EventStoreEntry(event))
            .toList();
        eventStore.saveAll(entries);

        // Tạo snapshot nếu version vượt ngưỡng (ví dụ: 100 events)
        if (order.getVersion() % 100 == 0) {
            createSnapshot(order);
        }

        order.clearPendingEvents();
    }
}
```

---

## Saga Pattern — Giao Dịch Phân Tán

### Choreography Saga (Saga Phối Hợp Tự Do)

```
Checkout Saga — không có orchestrator, services tự điều phối:

OrderService        PaymentService      InventoryService    ShippingService
     │                   │                    │                   │
     │──OrderCreated─────►│                    │                   │
     │   Event            │                    │                   │
     │                   │──PaymentDone───────►│                   │
     │                   │   Event             │                   │
     │                   │                    │──StockReserved─────►│
     │                   │                    │   Event             │
     │                   │                    │               │──OrderShipped
     │◄──────────────────────────────────────────────────────────
     │   OrderShippedEvent

Compensating Transactions (Giao Dịch Bù Trừ) khi lỗi:
     │                   │──PaymentFailed─────►│
     │◄──OrderCancelled──────────────────────── (Inventory không cần reserve)
```

```java
// Choreography Saga: mỗi service lắng nghe và phát event

@Component
public class PaymentSagaParticipant {

    private final PaymentService paymentService;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @KafkaListener(topics = "order.created", groupId = "payment-saga")
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            PaymentResult result = paymentService.charge(
                event.getUserId(),
                event.getTotalAmount()
            );

            // Thành công → phát event cho bước tiếp theo
            kafkaTemplate.send("payment.processed", new PaymentProcessedEvent(
                event.getOrderId(), result.getPaymentId(), PaymentStatus.SUCCESS
            ));

        } catch (PaymentException e) {
            // Thất bại → phát compensating event
            kafkaTemplate.send("payment.failed", new PaymentFailedEvent(
                event.getOrderId(), e.getMessage()
            ));
        }
    }
}

@Component
public class OrderSagaParticipant {

    private final OrderService orderService;

    // Lắng nghe compensating events để rollback
    @KafkaListener(topics = "payment.failed", groupId = "order-saga")
    public void handlePaymentFailed(PaymentFailedEvent event) {
        orderService.cancelOrder(event.getOrderId(), "Thanh toán thất bại: " + event.getReason());
    }

    @KafkaListener(topics = "stock.reservation.failed", groupId = "order-saga")
    public void handleStockReservationFailed(StockReservationFailedEvent event) {
        // Compensate: refund payment đã xử lý
        orderService.cancelOrderWithRefund(event.getOrderId());
    }
}
```

### Orchestration Saga (Saga Điều Phối Tập Trung)

```java
// Saga Orchestrator quản lý toàn bộ workflow
@Service
@Slf4j
public class CheckoutSagaOrchestrator {

    private final SagaStateRepository sagaStateRepo;
    private final PaymentClient paymentClient;
    private final InventoryClient inventoryClient;
    private final ShippingClient shippingClient;
    private final OrderService orderService;

    @Transactional
    public SagaResult execute(CheckoutSagaRequest request) {
        // Tạo và lưu saga state (để resume nếu bị crash)
        SagaState saga = sagaStateRepo.save(new SagaState(
            UUID.randomUUID().toString(),
            request.getOrderId(),
            SagaStatus.STARTED
        ));

        try {
            // Step 1: Validate Order
            saga.setStep("VALIDATE_ORDER");
            orderService.validateOrder(request.getOrderId());
            saga.setStep("ORDER_VALIDATED");

            // Step 2: Process Payment
            saga.setStep("PROCESS_PAYMENT");
            PaymentResult payment = paymentClient.charge(
                request.getUserId(), request.getAmount()
            );
            saga.setPaymentId(payment.getPaymentId());
            saga.setStep("PAYMENT_DONE");

            // Step 3: Reserve Stock
            saga.setStep("RESERVE_STOCK");
            try {
                inventoryClient.reserve(request.getOrderId(), request.getItems());
                saga.setStep("STOCK_RESERVED");
            } catch (Exception e) {
                // Compensate: Refund payment
                log.error("Stock reservation failed, rolling back payment");
                paymentClient.refund(payment.getPaymentId());
                saga.setStatus(SagaStatus.COMPENSATED);
                sagaStateRepo.save(saga);
                throw new SagaCompensatedException("Hết hàng, đã hoàn tiền");
            }

            // Step 4: Schedule Shipping
            saga.setStep("SCHEDULE_SHIPPING");
            shippingClient.scheduleDelivery(request.getOrderId(), request.getAddress());

            // Complete
            saga.setStatus(SagaStatus.COMPLETED);
            sagaStateRepo.save(saga);

            orderService.confirmOrder(request.getOrderId());
            return SagaResult.success(request.getOrderId());

        } catch (SagaCompensatedException e) {
            orderService.cancelOrder(request.getOrderId(), e.getMessage());
            return SagaResult.failed(e.getMessage());
        }
    }
}
```

---

## Triển Khai Với Spring Boot & Kafka

### Kafka Topic Design (Thiết Kế Topic)

```
# Đặt tên topic theo convention (quy ước): {domain}.{event-type}

order.created             # OrderCreatedEvent
order.confirmed           # OrderConfirmedEvent
order.cancelled           # OrderCancelledEvent
payment.processed         # PaymentProcessedEvent
payment.failed            # PaymentFailedEvent
inventory.reserved        # InventoryReservedEvent
inventory.reservation-failed  # InventoryReservationFailedEvent
notification.requested    # NotificationRequestedEvent

# Dead Letter Topic (Topic Thư Không Giao Được) — xử lý messages thất bại
order.created.DLT
payment.processed.DLT
```

### Idempotent Consumer (Người Tiêu Thụ Lũy Đẳng)

```java
// Quan trọng: Kafka có thể deliver message nhiều lần (at-least-once)
// Consumer phải xử lý duplicate safely (idempotent)

@Component
public class InventoryEventConsumer {

    private final InventoryService inventoryService;
    private final ProcessedEventRepository processedEvents;

    @KafkaListener(topics = "order.created", groupId = "inventory-service")
    @Transactional
    public void handleOrderCreated(
            @Payload OrderCreatedEvent event,
            @Header(KafkaHeaders.RECEIVED_KEY) String key) {

        String eventId = event.getEventId();

        // Idempotency check (Kiểm Tra Lũy Đẳng)
        if (processedEvents.existsByEventId(eventId)) {
            log.info("Event {} đã được xử lý, bỏ qua", eventId);
            return; // Bỏ qua duplicate
        }

        try {
            inventoryService.reserveStock(event.getOrderId(), event.getItems());

            // Đánh dấu đã xử lý (trong cùng transaction)
            processedEvents.save(new ProcessedEvent(eventId, Instant.now()));

        } catch (Exception e) {
            log.error("Xử lý event {} thất bại: {}", eventId, e.getMessage());
            throw e; // Re-throw để Kafka retry
        }
    }
}

@Entity
@Table(name = "processed_events")
public class ProcessedEvent {
    @Id
    private String eventId;
    private Instant processedAt;
}
```

---

## Outbox Pattern

### Vấn Đề Dual-write (Ghi Kép)

```
Vấn Đề: Làm sao đảm bảo DB và Kafka luôn nhất quán?

❌ Sai:
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order);    // Bước 1: Lưu vào DB
    kafkaTemplate.send("order.created", event); // Bước 2: Publish Kafka
    // NẾU Kafka fail → DB đã commit → Inconsistency!
}
```

### Outbox Pattern Giải Quyết

```java
// Outbox Table — lưu events vào DB trong cùng transaction với business data
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {
    @Id
    private String eventId;
    private String aggregateType;
    private String aggregateId;
    private String eventType;
    @Column(columnDefinition = "jsonb")
    private String payload;
    private Instant createdAt;
    private boolean published = false;
}

@Service
@Transactional
public class OrderService {

    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;
    private final ObjectMapper objectMapper;

    public OrderId createOrder(CreateOrderCommand command) {
        // Bước 1: Lưu Order
        Order order = Order.create(command);
        orderRepository.save(order);

        // Bước 2: Lưu event vào Outbox (cùng transaction!)
        OutboxEvent outboxEvent = new OutboxEvent(
            UUID.randomUUID().toString(),
            "Order",
            order.getId().toString(),
            "OrderCreated",
            objectMapper.writeValueAsString(new OrderCreatedEvent(order)),
            Instant.now()
        );
        outboxRepository.save(outboxEvent);

        // Nếu transaction commit → cả Order và OutboxEvent được lưu
        // Nếu transaction rollback → cả hai đều không lưu
        return order.getId();
    }
}

// Outbox Publisher — đọc outbox và publish lên Kafka
@Component
@Slf4j
public class OutboxEventPublisher {

    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;

    @Scheduled(fixedDelay = 1000) // Chạy mỗi 1 giây
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> pending = outboxRepository.findTop100ByPublishedFalseOrderByCreatedAtAsc();

        for (OutboxEvent event : pending) {
            try {
                kafkaTemplate.send(
                    resolveTopicName(event.getEventType()),
                    event.getAggregateId(),
                    event.getPayload()
                ).get(5, TimeUnit.SECONDS); // Đợi ACK từ Kafka

                event.setPublished(true);
                outboxRepository.save(event);

            } catch (Exception e) {
                log.error("Không thể publish event {}: {}", event.getEventId(), e.getMessage());
                // Không throw → để retry ở lần chạy tiếp theo
            }
        }
    }

    private String resolveTopicName(String eventType) {
        return switch (eventType) {
            case "OrderCreated" -> "order.created";
            case "OrderCancelled" -> "order.cancelled";
            default -> "events.unknown";
        };
    }
}
```

---

## Trade-offs & Khi Nào Dùng

### CQRS — Khi Nào Phù Hợp?

```
✅ Dùng CQRS khi:
- Read/Write loads rất khác nhau (ví dụ: 100 reads : 1 write)
- Query phức tạp, cần denormalized data
- Domain phức tạp, cần tách biệt read/write models rõ ràng
- Scaling read và write độc lập

❌ Overkill khi:
- Simple CRUD với read/write tương đương
- Team nhỏ, không có bandwidth để maintain 2 models
```

### Event Sourcing — Khi Nào Phù Hợp?

```
✅ Dùng Event Sourcing khi:
- Cần full audit trail (fintech, healthcare, legal)
- Cần temporal queries ("Order ở trạng thái nào vào 10:30 hôm qua?")
- Cần event replay để build projections mới
- Domain tự nhiên là event-based

❌ Overkill khi:
- Simple CRUD — overhead quá lớn
- Team không quen với pattern
- Query phức tạp mà không có read model tốt
```

---

## Câu Hỏi Phỏng Vấn

**Q: CQRS là gì? Tại sao không phải project nào cũng cần?**

> CQRS tách biệt read và write operations thành hai model riêng. Phù hợp khi write model phức tạp (cần enforce business rules) và read model cần denormalized data cho performance. Không phải mọi project đều cần vì: CQRS tăng complexity đáng kể — cần sync read model khi write model thay đổi, đội ngũ cần hiểu eventual consistency, và với CRUD đơn giản, overhead không đáng có.

**Q: Event Sourcing khác gì với traditional state storage?**

> Traditional: lưu trạng thái hiện tại, thay đổi trực tiếp trong database. Event Sourcing: lưu toàn bộ sequence of events, state hiện tại là kết quả replay tất cả events. Ưu điểm: full audit trail, temporal queries, không mất data. Nhược điểm: replay chậm với nhiều events (giải quyết bằng Snapshot), phức tạp hơn, event schema evolution (tiến hóa schema) cần quản lý cẩn thận.

**Q: Saga Pattern giải quyết vấn đề gì? Choreography vs Orchestration?**

> Saga giải quyết distributed transaction — khi một business operation span qua nhiều services không thể dùng ACID transaction. Choreography (mỗi service lắng nghe events và tự hành động) phù hợp khi flow đơn giản — không có single point of failure nhưng khó debug và trace. Orchestration (có coordinator điều phối) phù hợp khi flow phức tạp — dễ monitor, dễ handle compensation nhưng orchestrator có thể trở thành bottleneck.

**Q: Outbox Pattern giải quyết vấn đề gì?**

> Đảm bảo atomic (nguyên tử) giữa database write và message publish. Không thể dùng XA Transaction (phân tán) cho Kafka và DB. Outbox lưu event vào DB trong cùng transaction với business data. Một background process riêng đọc và publish lên Kafka. Nếu publish fail, process retry. Nếu app crash, messages vẫn an toàn trong DB.

---

## ✅ Checklist

- [ ] Command handlers và Query handlers tách biệt
- [ ] Read models tối ưu cho queries — không dùng write model để query
- [ ] Idempotent event consumers — xử lý duplicate messages an toàn
- [ ] Outbox Pattern cho at-least-once delivery đáng tin cậy
- [ ] Compensating transactions được thiết kế cho mọi saga step
- [ ] Event versioning strategy — quản lý schema evolution
- [ ] Dead Letter Queue (Hàng Đợi Thư Chết) cho messages không xử lý được

---

**Xem tiếp:** [7-circuit-breaker.md](7-circuit-breaker.md) — Resilience4j — Xử Lý Lỗi Hệ Thống Phân Tán
