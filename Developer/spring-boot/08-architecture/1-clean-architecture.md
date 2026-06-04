# Clean Architecture — Kiến Trúc Sạch & Hexagonal Architecture

> Clean Architecture (Kiến Trúc Sạch) và Hexagonal Architecture (Kiến Trúc Lục Giác), còn gọi là Ports & Adapters (Cổng & Bộ Chuyển Đổi) — các mô hình thiết kế giúp codebase độc lập với framework, database và giao diện bên ngoài.

---

## 📋 Mục Lục

1. [Vấn Đề Với Kiến Trúc Truyền Thống](#vấn-đề-với-kiến-trúc-truyền-thống)
2. [Clean Architecture Là Gì?](#clean-architecture-là-gì)
3. [Hexagonal Architecture — Ports & Adapters](#hexagonal-architecture--ports--adapters)
4. [Triển Khai Với Spring Boot](#triển-khai-với-spring-boot)
5. [Cấu Trúc Package](#cấu-trúc-package)
6. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
7. [Trade-offs & Khi Nào Dùng](#trade-offs--khi-nào-dùng)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Vấn Đề Với Kiến Trúc Truyền Thống

### Tight Coupling (Ghép Nối Chặt) Là Vấn Đề

```
Kiến trúc phân tầng truyền thống:

Controller → Service → Repository → Database

Vấn đề:
- Domain logic bị "rò rỉ" vào Controller và Repository
- JPA Entity bị dùng trực tiếp làm DTO → coupling giữa DB schema và API
- Khó test vì phải mock toàn bộ stack
- Thay đổi database hoặc framework → phải sửa nhiều tầng
```

### Dependency Rule (Quy Tắc Phụ Thuộc) Bị Vi Phạm

```java
// Anti-pattern: Service phụ thuộc trực tiếp vào JPA Entity
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository; // phụ thuộc vào infrastructure

    public Order createOrder(OrderRequest request) {
        Order order = new Order(); // JPA Entity trực tiếp
        order.setStatus("PENDING");
        return orderRepository.save(order); // trả về JPA Entity cho Controller
    }
}
```

---

## Clean Architecture Là Gì?

Clean Architecture của Robert C. Martin (Uncle Bob) đặt ra nguyên tắc:

> **"Domain logic không được phụ thuộc vào bất kỳ thứ gì bên ngoài."**

### Các Vòng Tròn (Layers)

```
           ┌──────────────────────────────┐
           │      Frameworks & Drivers     │  ← Spring Boot, JPA, Kafka
           │  ┌────────────────────────┐  │
           │  │   Interface Adapters   │  │  ← Controllers, Presenters, Gateways
           │  │  ┌──────────────────┐  │  │
           │  │  │   Application    │  │  │  ← Use Cases (Trường Hợp Sử Dụng)
           │  │  │  ┌────────────┐  │  │  │
           │  │  │  │  Entities  │  │  │  │  ← Domain Objects, Business Rules
           │  │  │  └────────────┘  │  │  │
           │  │  └──────────────────┘  │  │
           │  └────────────────────────┘  │
           └──────────────────────────────┘

Dependency Rule (Quy Tắc Phụ Thuộc):
→ Mũi tên phụ thuộc chỉ hướng VÀO TRONG
→ Domain không biết gì về Spring, JPA, HTTP
```

### 4 Tầng Chính

| Tầng | Vai Trò | Ví Dụ |
|------|---------|-------|
| **Entities** (Thực Thể) | Core business rules (Quy Tắc Nghiệp Vụ Cốt Lõi) | `Order`, `User`, `Product` |
| **Use Cases** (Trường Hợp Sử Dụng) | Application-specific logic (Logic Ứng Dụng Cụ Thể) | `CreateOrderUseCase`, `PlaceOrderService` |
| **Interface Adapters** (Bộ Chuyển Đổi Giao Diện) | Chuyển đổi dữ liệu | `OrderController`, `JpaOrderRepository` |
| **Frameworks & Drivers** (Framework & Trình Điều Khiển) | Công cụ bên ngoài | Spring Boot, PostgreSQL, Kafka |

---

## Hexagonal Architecture — Ports & Adapters

Hexagonal Architecture của Alistair Cockburn tương đương Clean Architecture nhưng dùng thuật ngữ khác:

```
                    ┌─────────────────────────────┐
                    │         Hexagon             │
  REST Controller   │                             │   JPA Repository
  (Driving Adapter) │  ┌─────────────────────┐   │   (Driven Adapter)
        ────────────┼──►  Application Service ├───┼──►
                    │  │  (Use Case / Port)   │   │
  Kafka Consumer    │  │                      │   │   Kafka Producer
  (Driving Adapter) │  │   Domain Objects     │   │   (Driven Adapter)
        ────────────┼──►  (Business Logic)    ├───┼──►
                    │  └─────────────────────┘   │
  gRPC Server       │                             │   Redis Cache
  (Driving Adapter) │                             │   (Driven Adapter)
        ────────────┼─────────────────────────────┼──►
                    └─────────────────────────────┘

Port (Cổng): Interface được định nghĩa trong Domain
Adapter (Bộ Chuyển Đổi): Implement của Port, nằm ngoài Domain
```

### Hai Loại Port

**Driving Ports** (Cổng Chủ Động) — còn gọi là Primary/Inbound:

```java
// Port: interface trong application layer
public interface CreateOrderUseCase {
    OrderId createOrder(CreateOrderCommand command);
}

// Adapter (Driving): REST Controller implement port này
@RestController
public class OrderController {
    private final CreateOrderUseCase createOrderUseCase;

    @PostMapping("/orders")
    public ResponseEntity<OrderResponse> create(@RequestBody CreateOrderRequest request) {
        CreateOrderCommand command = new CreateOrderCommand(
            request.getUserId(),
            request.getItems()
        );
        OrderId orderId = createOrderUseCase.createOrder(command);
        return ResponseEntity.ok(new OrderResponse(orderId.value()));
    }
}
```

**Driven Ports** (Cổng Bị Động) — còn gọi là Secondary/Outbound:

```java
// Port: interface trong domain layer — domain KHÔNG biết JPA tồn tại
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByUserId(UserId userId);
}

// Adapter (Driven): JPA implementation nằm ở infrastructure layer
@Repository
public class JpaOrderRepositoryAdapter implements OrderRepository {
    private final JpaOrderJpaRepository jpaRepository;
    private final OrderMapper mapper;

    @Override
    public void save(Order order) {
        OrderJpaEntity entity = mapper.toEntity(order);
        jpaRepository.save(entity);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepository.findById(id.value())
            .map(mapper::toDomain);
    }
}
```

---

## Triển Khai Với Spring Boot

### Cấu Trúc Module

```
src/main/java/com/example/
├── domain/                          ← Tầng Domain (không phụ thuộc gì)
│   ├── model/
│   │   ├── Order.java               ← Domain Entity (Thực Thể Miền)
│   │   ├── OrderId.java             ← Value Object (Đối Tượng Giá Trị)
│   │   ├── OrderItem.java
│   │   └── OrderStatus.java         ← Domain Enum
│   ├── event/
│   │   └── OrderCreatedEvent.java   ← Domain Event (Sự Kiện Miền)
│   └── exception/
│       └── OrderNotFoundException.java
│
├── application/                     ← Tầng Application (chỉ phụ thuộc Domain)
│   ├── port/
│   │   ├── in/                      ← Driving Ports (Input Ports)
│   │   │   ├── CreateOrderUseCase.java
│   │   │   └── GetOrderUseCase.java
│   │   └── out/                     ← Driven Ports (Output Ports)
│   │       ├── OrderRepository.java
│   │       └── OrderEventPublisher.java
│   ├── service/
│   │   └── OrderService.java        ← Use Case Implementation
│   └── dto/
│       ├── CreateOrderCommand.java  ← Input DTO
│       └── OrderDto.java            ← Output DTO
│
└── infrastructure/                  ← Tầng Infrastructure (phụ thuộc Spring, JPA)
    ├── web/
    │   ├── OrderController.java     ← Driving Adapter: REST
    │   └── dto/
    │       ├── CreateOrderRequest.java
    │       └── OrderResponse.java
    ├── persistence/
    │   ├── JpaOrderRepositoryAdapter.java  ← Driven Adapter: JPA
    │   ├── OrderJpaEntity.java             ← JPA Entity (KHÔNG phải Domain Entity)
    │   ├── OrderJpaRepository.java         ← Spring Data JPA interface
    │   └── OrderMapper.java                ← MapStruct mapper
    └── messaging/
        └── KafkaOrderEventPublisher.java   ← Driven Adapter: Kafka
```

### Domain Entity — Thuần Túy, Không Phụ Thuộc Framework

```java
// domain/model/Order.java
// Không có @Entity, không có Spring annotation
public class Order {
    private final OrderId id;
    private final UserId userId;
    private OrderStatus status;
    private final List<OrderItem> items;
    private final List<OrderCreatedEvent> domainEvents = new ArrayList<>();

    // Factory method (Phương Thức Tạo) — enforce business rules
    public static Order create(UserId userId, List<OrderItem> items) {
        if (items == null || items.isEmpty()) {
            throw new IllegalArgumentException("Đơn hàng phải có ít nhất một sản phẩm");
        }
        Order order = new Order(OrderId.generate(), userId, OrderStatus.PENDING, items);
        order.domainEvents.add(new OrderCreatedEvent(order.id, userId));
        return order;
    }

    public void confirm() {
        if (this.status != OrderStatus.PENDING) {
            throw new IllegalStateException("Chỉ có thể xác nhận đơn hàng ở trạng thái PENDING");
        }
        this.status = OrderStatus.CONFIRMED;
    }

    public Money calculateTotal() {
        return items.stream()
            .map(OrderItem::subtotal)
            .reduce(Money.ZERO, Money::add);
    }

    public List<OrderCreatedEvent> pullDomainEvents() {
        List<OrderCreatedEvent> events = new ArrayList<>(domainEvents);
        domainEvents.clear();
        return events;
    }
    // ... getters
}
```

### Value Object (Đối Tượng Giá Trị)

```java
// domain/model/OrderId.java
public record OrderId(UUID value) {
    public OrderId {
        Objects.requireNonNull(value, "OrderId không được null");
    }

    public static OrderId generate() {
        return new OrderId(UUID.randomUUID());
    }

    public static OrderId of(String value) {
        return new OrderId(UUID.fromString(value));
    }
}

// domain/model/Money.java
public record Money(BigDecimal amount, Currency currency) {
    public static final Money ZERO = new Money(BigDecimal.ZERO, Currency.getInstance("VND"));

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Không thể cộng hai đơn vị tiền tệ khác nhau");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

### Use Case Implementation

```java
// application/service/OrderService.java
@Service
@Transactional
public class OrderService implements CreateOrderUseCase, GetOrderUseCase {

    private final OrderRepository orderRepository;     // Port — không phải JPA
    private final OrderEventPublisher eventPublisher;  // Port — không phải Kafka

    public OrderService(OrderRepository orderRepository,
                        OrderEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }

    @Override
    public OrderId createOrder(CreateOrderCommand command) {
        // Tạo domain object — logic nghiệp vụ nằm ở đây
        List<OrderItem> items = command.items().stream()
            .map(item -> new OrderItem(
                ProductId.of(item.productId()),
                item.quantity(),
                Money.of(item.price())
            ))
            .toList();

        Order order = Order.create(UserId.of(command.userId()), items);

        // Lưu qua port — không biết đằng sau là JPA hay gì
        orderRepository.save(order);

        // Publish domain events (Sự Kiện Miền)
        order.pullDomainEvents().forEach(eventPublisher::publish);

        return order.getId();
    }

    @Override
    @Transactional(readOnly = true)
    public OrderDto getOrder(OrderId orderId) {
        return orderRepository.findById(orderId)
            .map(OrderDto::fromDomain)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
}
```

### JPA Adapter — Chuyển Đổi Domain ↔ JPA Entity

```java
// infrastructure/persistence/JpaOrderRepositoryAdapter.java
@Repository
public class JpaOrderRepositoryAdapter implements OrderRepository {

    private final OrderJpaRepository jpaRepository;
    private final OrderMapper mapper;

    @Override
    public void save(Order order) {
        OrderJpaEntity entity = mapper.toEntity(order);
        jpaRepository.save(entity);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepository.findById(id.value())
            .map(mapper::toDomain);
    }
}

// infrastructure/persistence/OrderJpaEntity.java
@Entity
@Table(name = "orders")
public class OrderJpaEntity {
    @Id
    private UUID id;

    @Column(name = "user_id", nullable = false)
    private UUID userId;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItemJpaEntity> items;
    // getters/setters
}
```

---

## Cấu Trúc Package

### Package By Layer (Theo Tầng) — Không Khuyến Nghị

```
❌ Không nên:
com.example.
├── controller/    ← tất cả controllers của mọi feature
├── service/       ← tất cả services
└── repository/    ← tất cả repositories

Nhược điểm:
- Khó tìm code liên quan
- Dễ tạo coupling giữa các features
- Khó extract thành microservice sau này
```

### Package By Feature (Theo Tính Năng) — Khuyến Nghị

```
✅ Nên dùng:
com.example.
├── order/
│   ├── domain/
│   ├── application/
│   └── infrastructure/
├── payment/
│   ├── domain/
│   ├── application/
│   └── infrastructure/
└── user/
    ├── domain/
    ├── application/
    └── infrastructure/

Ưu điểm:
- Cohesion cao (mọi code liên quan đến Order ở cùng chỗ)
- Dễ tách thành microservice
- Ranh giới module rõ ràng
```

---

## Ví Dụ Thực Tế

### Luồng Xử Lý "Tạo Đơn Hàng"

```
1. HTTP POST /orders
        │
        ▼
2. OrderController (Infrastructure Layer)
   - Validate request DTO
   - Map sang CreateOrderCommand
        │
        ▼
3. CreateOrderUseCase.createOrder(command) [Application Port]
        │
        ▼
4. OrderService (Application Layer)
   - Tạo Order domain object
   - Áp dụng business rules (validate, tính toán)
   - Gọi OrderRepository.save(order)
   - Gọi EventPublisher.publish(events)
        │
     ┌──┴──┐
     │     │
     ▼     ▼
5a. JpaOrderRepositoryAdapter    5b. KafkaOrderEventPublisher
    (Infrastructure)                  (Infrastructure)
    - Map sang JPA Entity             - Serialize domain event
    - orderJpaRepository.save()       - Gửi Kafka message
```

### Testing Với Clean Architecture

```java
// Test Use Case mà không cần Spring context
class OrderServiceTest {

    // Mock ports — không cần H2 hay mock JPA
    private OrderRepository orderRepository = mock(OrderRepository.class);
    private OrderEventPublisher eventPublisher = mock(OrderEventPublisher.class);
    private OrderService orderService = new OrderService(orderRepository, eventPublisher);

    @Test
    void createOrder_ValidItems_ShouldSaveAndPublishEvent() {
        // Arrange
        CreateOrderCommand command = new CreateOrderCommand(
            "user-123",
            List.of(new OrderItemCommand("product-1", 2, new BigDecimal("50000")))
        );

        // Act
        OrderId orderId = orderService.createOrder(command);

        // Assert
        assertNotNull(orderId);
        verify(orderRepository).save(any(Order.class));
        verify(eventPublisher).publish(any(OrderCreatedEvent.class));
    }

    @Test
    void createOrder_EmptyItems_ShouldThrowException() {
        CreateOrderCommand command = new CreateOrderCommand("user-123", List.of());

        assertThrows(IllegalArgumentException.class,
            () -> orderService.createOrder(command));

        verifyNoInteractions(orderRepository);
    }
}
```

---

## Trade-offs & Khi Nào Dùng

### Ưu Điểm

| Ưu Điểm | Chi Tiết |
|---------|---------|
| **Testability** (Khả Năng Kiểm Thử) | Domain và Use Cases test được mà không cần Spring |
| **Framework Independence** (Độc Lập Framework) | Có thể đổi Spring sang Quarkus mà không sửa domain |
| **Database Independence** (Độc Lập Database) | Swap từ PostgreSQL sang MongoDB chỉ cần viết Adapter mới |
| **Long-term Maintainability** (Bảo Trì Lâu Dài) | Business rules tập trung, không bị phân tán |

### Nhược Điểm

| Nhược Điểm | Chi Tiết |
|-----------|---------|
| **Boilerplate** (Mã Lặp) | Nhiều interface, mapper, class hơn |
| **Learning Curve** (Đường Cong Học) | Team cần thời gian làm quen |
| **Overkill cho CRUD đơn giản** | App CRUD không cần complexity này |
| **Mapper overhead** (Chi Phí Chuyển Đổi) | Domain Entity ↔ JPA Entity ↔ DTO mapping |

### Khi Nào Dùng Clean Architecture?

```
✅ Phù Hợp Khi:
- Domain logic phức tạp (e-commerce, fintech, healthcare)
- Codebase dự kiến tồn tại > 3 năm
- Team áp dụng DDD
- Cần swap infrastructure components
- Muốn đạt coverage > 80% dễ dàng

❌ Overkill Khi:
- Simple CRUD API (ví dụ: blog đơn giản)
- Prototype / MVP cần ra nhanh
- Team < 3 người, deadline < 1 tháng
```

---

## Câu Hỏi Phỏng Vấn

**Q: Clean Architecture khác gì Layered Architecture?**

> Layered Architecture (Kiến Trúc Phân Tầng) cho phép tầng trên phụ thuộc tầng dưới (Controller → Service → Repository), thường dẫn đến domain logic bị coupled với database/framework. Clean Architecture đảo ngược phụ thuộc này: domain không phụ thuộc vào bất kỳ tầng nào, các outer layers (framework, DB) phụ thuộc vào domain thông qua interfaces (Dependency Inversion Principle).

**Q: Port là gì, Adapter là gì trong Hexagonal Architecture?**

> **Port** là một interface được định nghĩa trong domain/application layer, đại diện cho một khả năng mà domain cần (output port như `OrderRepository`) hoặc cung cấp (input port như `CreateOrderUseCase`). **Adapter** là implementation cụ thể của port, nằm trong infrastructure layer — ví dụ `JpaOrderRepositoryAdapter` implement `OrderRepository` port.

**Q: Làm sao test domain logic khi dùng Clean Architecture?**

> Vì domain không phụ thuộc vào Spring hay JPA, ta test bằng pure Java unit tests. Mock các output ports (như `OrderRepository`) bằng Mockito hoặc tự viết in-memory implementation. Không cần `@SpringBootTest` hay database cho domain tests.

**Q: Bạn đã áp dụng Clean Architecture trong dự án thực chưa? Gặp vấn đề gì?**

> Vấn đề thường gặp: (1) Mapping fatigue — quá nhiều mapper giữa các tầng, giải quyết bằng MapStruct; (2) Team không quen — cần training và code review nghiêm; (3) Ranh giới không rõ — phải pair programming để enforce boundaries.

---

## ✅ Checklist

- [ ] Domain Entity không import Spring, JPA annotation
- [ ] Use Case test được mà không cần Spring context
- [ ] JPA Entity tách biệt với Domain Entity
- [ ] Dependency chỉ hướng vào trong (Domain ← Application ← Infrastructure)
- [ ] Output Ports là interface trong application layer
- [ ] Mapper xử lý chuyển đổi giữa các tầng

---

**Xem tiếp:** [2-layered-architecture.md](2-layered-architecture.md) — Controller-Service-Repository truyền thống
