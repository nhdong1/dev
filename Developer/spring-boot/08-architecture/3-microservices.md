# Microservices — Kiến Trúc Vi Dịch Vụ

> Microservices Architecture (Kiến Trúc Vi Dịch Vụ) — phân tách ứng dụng thành các service nhỏ, độc lập, có thể deploy riêng biệt. Mỗi service sở hữu một bounded context (ngữ cảnh bị giới hạn) và database riêng.

---

## 📋 Mục Lục

1. [Monolith vs Microservices](#monolith-vs-microservices)
2. [Domain-Driven Design (DDD)](#domain-driven-design-ddd)
3. [Service Decomposition Strategies](#service-decomposition-strategies)
4. [Inter-Service Communication](#inter-service-communication)
5. [Data Management Patterns](#data-management-patterns)
6. [Spring Boot Microservice Project](#spring-boot-microservice-project)
7. [Distributed Systems Challenges](#distributed-systems-challenges)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Monolith vs Microservices

### Monolith (Ứng Dụng Nguyên Khối)

```
┌──────────────────────────────────────────────┐
│               MONOLITH APPLICATION            │
│                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐ │
│  │  Order   │ │ Payment  │ │  Inventory   │ │
│  │  Module  │ │  Module  │ │   Module     │ │
│  └──────────┘ └──────────┘ └──────────────┘ │
│         ↓            ↓            ↓          │
│  ┌────────────────────────────────────────┐  │
│  │          Shared Database               │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘

Ưu điểm Monolith:
✅ Đơn giản để develop và debug
✅ In-process calls — không có network latency
✅ ACID transactions dễ dàng
✅ Ít infrastructure overhead

Nhược điểm Monolith:
❌ Một bug có thể crash toàn bộ app
❌ Scale toàn bộ khi chỉ một phần cần scale
❌ Technology lock-in (Bị Khóa Công Nghệ)
❌ Team lớn → merge conflicts liên tục
❌ Deployment risky — deploy = deploy tất cả
```

### Microservices (Vi Dịch Vụ)

```
        Client
           │
    ┌──────▼──────┐
    │ API Gateway  │
    └──────┬───────┘
    ┌──────┼──────────────┐
    │      │              │
┌───▼───┐ ┌▼──────┐ ┌────▼──────┐
│ Order │ │Payment│ │ Inventory │
│Service│ │Service│ │  Service  │
└───┬───┘ └───┬───┘ └─────┬─────┘
    │         │            │
┌───▼──┐ ┌───▼──┐ ┌────────▼───┐
│Order │ │Pay   │ │ Inventory  │
│  DB  │ │  DB  │ │    DB      │
└──────┘ └──────┘ └────────────┘

Ưu điểm Microservices:
✅ Scale độc lập từng service
✅ Fault isolation — một service fail không crash tất cả
✅ Technology freedom (Tự Do Công Nghệ) — mỗi service dùng tech phù hợp
✅ Team tự trị — Conway's Law (Luật Conway)
✅ Deployment độc lập — liên tục deploy không rủi ro

Nhược điểm Microservices:
❌ Distributed systems complexity (Phức Tạp Hệ Thống Phân Tán)
❌ Network latency và partial failures
❌ Distributed transactions khó
❌ Operational overhead (Chi Phí Vận Hành) cao
❌ Đòi hỏi DevOps mature (Vận Hành Trưởng Thành)
```

---

## Domain-Driven Design (DDD)

### Tại Sao DDD Quan Trọng Với Microservices?

DDD — Domain-Driven Design (Thiết Kế Hướng Miền) cung cấp công cụ để phân chia service đúng cách dựa trên business domain.

### Khái Niệm Cốt Lõi

#### Bounded Context (Ngữ Cảnh Bị Giới Hạn)

```
"Customer" trong các context khác nhau:

┌───────────────────┐    ┌───────────────────┐
│   Order Context   │    │  Payment Context  │
│                   │    │                   │
│  Customer:        │    │  Customer:        │
│  - id             │    │  - id             │
│  - name           │    │  - billingAddress │
│  - shippingAddress│    │  - creditScore    │
│  - orderHistory   │    │  - paymentMethods │
└───────────────────┘    └───────────────────┘

→ Cùng tên "Customer" nhưng khác nhau trong mỗi context
→ Đây chính là dấu hiệu của 2 Bounded Context khác nhau
→ = 2 Microservice khác nhau
```

#### Aggregate (Tập Hợp)

```java
// Order Aggregate Root (Gốc Tập Hợp Đơn Hàng)
// Toàn bộ Order (đơn hàng) + OrderItems được coi là một đơn vị nhất quán
public class Order {
    private OrderId id;          // Aggregate Root ID
    private List<OrderItem> items; // Aggregate member — chỉ truy cập qua Order
    private OrderStatus status;
    private Money totalAmount;

    // Mọi thay đổi phải qua Aggregate Root
    public void addItem(ProductId productId, int quantity, Money price) {
        validateStatus(); // business rule
        items.add(new OrderItem(productId, quantity, price));
        recalculateTotal();
    }

    public void submit() {
        if (items.isEmpty()) throw new BusinessException("Không thể đặt đơn hàng trống");
        this.status = OrderStatus.SUBMITTED;
        registerEvent(new OrderSubmittedEvent(this.id));
    }
}

// KHÔNG bao giờ truy cập OrderItem trực tiếp từ ngoài Order
// orderItemRepository.save(item) ← SAI
// order.addItem(...); orderRepository.save(order) ← ĐÚNG
```

#### Ubiquitous Language (Ngôn Ngữ Phổ Quát)

```
Toàn bộ team (dev, BA, PM, domain expert) dùng cùng một thuật ngữ:

❌ Không nên:
- Dev gọi là "User", BA gọi là "Customer", PM gọi là "Client"
- Code: userEntity.customerId vs businessRule: customer checkout

✅ Nên:
- Thống nhất: "Customer" là người mua hàng
- Code: Customer, customerRepository, CustomerService
- Conversations: "Customer places an Order" (Khách Hàng Đặt Đơn Hàng)
```

### Context Map (Bản Đồ Ngữ Cảnh)

```
Quan hệ giữa các Bounded Contexts:

Order Context ──────Shared Kernel──────► Shared Domain (Money, Address)
     │
     │ Customer/Supplier
     ▼
Inventory Context
     │
     │ Published Language (REST API)
     ▼
Fulfillment Context

Payment Context ────Anti-Corruption Layer────► External Payment Gateway
                    (Lớp Chống Ô Nhiễm)        (Stripe, PayPal)
```

---

## Service Decomposition Strategies

### 1. Decompose by Business Capability (Phân Tách Theo Khả Năng Nghiệp Vụ)

```
E-commerce System:

├── Order Management Service    — Quản Lý Đơn Hàng
├── Product Catalog Service     — Danh Mục Sản Phẩm
├── Inventory Service           — Quản Lý Kho
├── Payment Service             — Thanh Toán
├── Shipping Service            — Vận Chuyển
├── Notification Service        — Thông Báo
└── User Account Service        — Tài Khoản Người Dùng
```

### 2. Decompose by Subdomain (Phân Tách Theo Subdomain)

```
DDD Subdomains (Các Subdomain Trong DDD):

Core Domain (Miền Cốt Lõi — Lợi Thế Cạnh Tranh):
└── Order Processing, Recommendation Engine

Supporting Subdomain (Miền Hỗ Trợ):
└── Inventory, Shipping, Customer Support

Generic Subdomain (Miền Chung — Có Thể Mua Sẵn):
└── Authentication (Keycloak), Payment (Stripe), Email (SendGrid)
```

### 3. Strangler Fig Pattern (Mẫu Cây Sung Bóp Nghẹt)

```
Migrate từng bước từ Monolith sang Microservices:

Bước 1:
Client → Monolith (toàn bộ)

Bước 2:
Client → API Facade (Mặt Tiền API) → Monolith
                                   └→ Payment Service (tách ra)

Bước 3:
Client → API Gateway → Payment Service
                    └→ Order Service (tách ra)
                    └→ Monolith (phần còn lại)

Bước N:
Client → API Gateway → Service A
                    └→ Service B
                    └→ Service C (Monolith đã được loại bỏ hoàn toàn)
```

---

## Inter-Service Communication

### Synchronous (Đồng Bộ) — REST / gRPC

```java
// Cách 1: OpenFeign Client (Khuyến Nghị)
@FeignClient(name = "inventory-service", path = "/api/v1/inventory")
public interface InventoryClient {

    @GetMapping("/check")
    InventoryResponse checkStock(@RequestParam String productId,
                                 @RequestParam int quantity);
}

// OrderService gọi InventoryService
@Service
public class OrderService {

    private final InventoryClient inventoryClient;

    public void createOrder(CreateOrderCommand command) {
        // Synchronous call — đợi response
        InventoryResponse inventory = inventoryClient.checkStock(
            command.productId(), command.quantity()
        );

        if (!inventory.isAvailable()) {
            throw new InsufficientStockException("Sản phẩm hết hàng");
        }
        // tiếp tục tạo order...
    }
}

// Cấu hình với Circuit Breaker (Cầu Dao Mạch) — sẽ học ở file 7
@FeignClient(
    name = "inventory-service",
    fallback = InventoryClientFallback.class
)
public interface InventoryClient { ... }

@Component
public class InventoryClientFallback implements InventoryClient {
    @Override
    public InventoryResponse checkStock(String productId, int quantity) {
        // Fallback: giả sử có hàng và xử lý sau
        return InventoryResponse.assume(true);
    }
}
```

### Asynchronous (Bất Đồng Bộ) — Kafka / RabbitMQ

```java
// OrderService publish event (đăng sự kiện)
@Service
public class OrderService {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @Transactional
    public void submitOrder(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.submit();
        orderRepository.save(order);

        // Publish event — không đợi Inventory hay Payment response
        kafkaTemplate.send("order.submitted",
            order.getId().toString(),
            new OrderSubmittedEvent(order.getId(), order.getItems())
        );
    }
}

// InventoryService consume event (tiêu thụ sự kiện)
@Component
public class OrderEventConsumer {

    @KafkaListener(topics = "order.submitted", groupId = "inventory-service")
    public void handleOrderSubmitted(OrderSubmittedEvent event) {
        inventoryService.reserveStock(event.getItems());
        // Publish InventoryReservedEvent
    }
}
```

### Communication Pattern Comparison

| Pattern | Khi Nào Dùng | Trade-off |
|---------|-------------|-----------|
| **REST (Synchronous)** | Cần response ngay, simple query | Tight coupling, cascade failures |
| **gRPC (Synchronous)** | High-performance internal calls | Cần Proto definition, harder debugging |
| **Kafka (Async)** | Event-driven, high throughput | Eventual consistency, complex flow |
| **RabbitMQ (Async)** | Task queues, retry logic | Message ordering khó |

---

## Data Management Patterns

### Database Per Service (Mỗi Dịch Vụ Một Cơ Sở Dữ Liệu)

```
Order Service      → PostgreSQL (orders, order_items)
Payment Service    → PostgreSQL (payments, transactions)
Inventory Service  → PostgreSQL (products, stock_levels)
Session Service    → Redis (user sessions)
Search Service     → Elasticsearch (product search index)
Analytics Service  → ClickHouse (event logs)

Lý Do:
- Decoupling (Tách Rời): Schema thay đổi không ảnh hưởng services khác
- Technology fit: Mỗi service chọn database phù hợp với use case
- Independent scaling (Scale Độc Lập): Scale storage của từng service
```

### Shared Database Anti-pattern

```
❌ KHÔNG NÊN:
Order Service ────────────┐
Payment Service ──────────┤──► Shared PostgreSQL
Inventory Service ────────┘

Vấn Đề:
- Schema change ảnh hưởng tất cả services
- Database trở thành single point of failure
- Teams bị phụ thuộc nhau (coupling qua DB)
- Không thể scale DB của từng service độc lập
```

### Saga Pattern — Distributed Transaction (Giao Dịch Phân Tán)

Microservices không thể dùng ACID transaction truyền thống qua nhiều services. Saga pattern giải quyết vấn đề này.

#### Choreography Saga (Saga Phối Hợp Tự Do)

```
Services tự điều phối qua events — không có orchestrator trung tâm

Checkout Flow:

1. OrderService:    CREATE Order (PENDING)
                    PUBLISH OrderCreatedEvent
                           ↓
2. PaymentService:  RECEIVE OrderCreatedEvent
                    PROCESS Payment
                    PUBLISH PaymentSuccessEvent (or PaymentFailedEvent)
                           ↓
3. InventoryService: RECEIVE PaymentSuccessEvent
                     RESERVE Stock
                     PUBLISH StockReservedEvent (or StockReservationFailedEvent)
                           ↓
4. OrderService:    RECEIVE StockReservedEvent
                    UPDATE Order → CONFIRMED

Compensating Transactions (Giao Dịch Bù Trừ) khi lỗi:
PaymentFailedEvent → OrderService: UPDATE Order → CANCELLED
StockReservationFailedEvent → PaymentService: REFUND payment
                            → OrderService: UPDATE Order → CANCELLED
```

#### Orchestration Saga (Saga Điều Phối Tập Trung)

```java
// SagaOrchestrator điều phối toàn bộ flow
@Component
public class CheckoutSagaOrchestrator {

    private final PaymentClient paymentClient;
    private final InventoryClient inventoryClient;
    private final OrderRepository orderRepository;

    @Transactional
    public void execute(CheckoutSagaContext context) {
        try {
            // Bước 1: Tạo Order
            Order order = createOrder(context);

            // Bước 2: Xử lý Payment
            PaymentResult payment = paymentClient.charge(
                context.getCustomerId(),
                order.getTotalAmount()
            );

            // Bước 3: Reserve Inventory (Dự Trữ Kho)
            try {
                inventoryClient.reserve(context.getItems());
            } catch (Exception e) {
                // Compensate: Hoàn tiền nếu reserve inventory thất bại
                paymentClient.refund(payment.getPaymentId());
                throw e;
            }

            // Bước 4: Confirm Order
            order.confirm();
            orderRepository.save(order);

        } catch (Exception e) {
            // Compensate: Cancel order
            cancelOrder(context.getOrderId());
            throw new SagaExecutionException("Checkout thất bại", e);
        }
    }
}
```

---

## Spring Boot Microservice Project

### pom.xml cho Order Service

```xml
<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Spring Cloud OpenFeign — HTTP Client -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>

    <!-- Spring Cloud Netflix Eureka Client — Service Discovery -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>

    <!-- Resilience4j — Circuit Breaker -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
    </dependency>

    <!-- Spring Kafka — Messaging -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>

    <!-- Spring Data JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- Spring Boot Actuator — Health & Metrics -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.3</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### application.yml cho Order Service

```yaml
spring:
  application:
    name: order-service    # Tên đăng ký với Eureka
  datasource:
    url: jdbc:postgresql://localhost:5432/orderdb
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      group-id: order-service
      auto-offset-reset: earliest

server:
  port: 8081

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/

# Resilience4j Circuit Breaker config
resilience4j:
  circuitbreaker:
    instances:
      inventory-service:
        failure-rate-threshold: 50          # Mở CB khi 50% requests fail
        wait-duration-in-open-state: 10s    # Giữ CB open 10 giây
        sliding-window-size: 10             # Evaluate 10 requests gần nhất

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,info,circuitbreakers
```

---

## Distributed Systems Challenges

### Network Reliability (Độ Tin Cậy Mạng)

```java
// Fallback cho trường hợp network không ổn định
@CircuitBreaker(name = "inventory-service", fallbackMethod = "checkStockFallback")
@Retry(name = "inventory-service")
public InventoryResponse checkStock(String productId, int quantity) {
    return inventoryClient.checkStock(productId, quantity);
}

public InventoryResponse checkStockFallback(String productId, int quantity, Exception e) {
    log.warn("Inventory service không khả dụng, dùng fallback: {}", e.getMessage());
    // Cho phép order tiến hành, kiểm tra inventory sau (async)
    return InventoryResponse.optimistic(true);
}
```

### Distributed Tracing (Theo Dõi Phân Tán)

```yaml
# Micrometer Tracing với Zipkin — theo dõi request qua nhiều services
management:
  tracing:
    sampling:
      probability: 1.0   # Trace 100% requests (chỉ cho dev)

spring:
  zipkin:
    base-url: http://localhost:9411
```

```
Request: POST /orders
    │
    ├─ Trace ID: abc123
    │
    ├─ [Order Service] span: create-order           100ms
    │        │
    │        ├─ [Inventory Service] span: check-stock  30ms
    │        │
    │        └─ [Payment Service] span: process-payment  60ms
    │
    └─ Total: 190ms
```

### Eventual Consistency (Nhất Quán Cuối Cùng)

```
Vấn đề: Không thể đảm bảo mọi services đồng bộ ngay lập tức

Giải pháp: Thiết kế chấp nhận "Eventually Consistent"

Ví dụ: Sau khi order tạo xong (Order Service),
inventory chưa được update ngay (Inventory Service đang xử lý Kafka message)

→ UI hiển thị "Đang xử lý" thay vì "Hoàn thành" ngay
→ Retry mechanism khi message processing fail
→ Idempotency (Lũy Đẳng) — xử lý cùng message nhiều lần không có tác dụng phụ
```

---

## Câu Hỏi Phỏng Vấn

**Q: Khi nào nên migrate từ Monolith sang Microservices?**

> Khi gặp các dấu hiệu: (1) Release cycle bị chậm do conflict giữa nhiều team; (2) Cần scale từng phần (ví dụ Order processing cần nhiều resource hơn User profile); (3) Domain rõ ràng, team có thể sở hữu từng domain độc lập; (4) Có DevOps capability để vận hành nhiều services. KHÔNG nên migrate chỉ vì nó "trendy" — microservices phức tạp hơn rất nhiều.

**Q: Bounded Context là gì, tại sao quan trọng?**

> Bounded Context là ranh giới ngữ nghĩa trong đó một model domain cụ thể được áp dụng nhất quán. Ví dụ "Product" trong Order context có giá và số lượng, nhưng "Product" trong Catalog context có description và SEO metadata. Việc nhận ra và tôn trọng Bounded Contexts giúp phân chia service đúng nơi — tránh "chatty microservices" giao tiếp quá nhiều với nhau.

**Q: Distributed Transaction trong Microservices giải quyết như thế nào?**

> Không thể dùng ACID transaction truyền thống qua nhiều services. Giải pháp phổ biến: (1) Saga Pattern — choreography (dùng events) hoặc orchestration (có coordinator). (2) Two-Phase Commit (2PC — Xác Nhận Hai Giai Đoạn) — hiếm khi dùng do blocking. (3) Eventual Consistency với compensating transactions (giao dịch bù trừ) khi có lỗi.

**Q: Database per Service ảnh hưởng thế nào đến querying?**

> Join query qua nhiều databases không thể làm trực tiếp. Giải pháp: (1) API Composition — gọi nhiều services và join ở application layer; (2) CQRS với Query Model tổng hợp dữ liệu từ nhiều services; (3) Event-driven Data Replication — copy dữ liệu cần thiết vào service cần dùng.

---

## ✅ Checklist

- [ ] Mỗi microservice có database riêng — không share database
- [ ] Service boundaries dựa trên business capabilities, không phải kỹ thuật
- [ ] Áp dụng Ubiquitous Language (Ngôn Ngữ Phổ Quát) trong code và documentation
- [ ] Cấu hình Circuit Breaker cho inter-service calls
- [ ] Distributed tracing với Trace ID qua toàn bộ request chain
- [ ] Idempotent message handlers để xử lý duplicate messages
- [ ] Saga pattern cho multi-step business transactions

---

**Xem tiếp:** [4-api-gateway.md](4-api-gateway.md) — Spring Cloud Gateway — Cổng API Trung Tâm
