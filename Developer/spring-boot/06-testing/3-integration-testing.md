# Integration Testing với @SpringBootTest & Testcontainers

> Integration Test (Kiểm Thử Tích Hợp) kiểm tra nhiều component làm việc cùng nhau —
> từ Controller đến Service, Repository đến Database thực.
> Testcontainers cho phép chạy database, Kafka, Redis thực trong Docker container
> ngay trong quá trình test.

---

## 1. Tại Sao Cần Integration Test?

```
Unit Test kiểm tra: mỗi class hoạt động đúng khi cô lập
Integration Test kiểm tra: các class hoạt động đúng khi kết hợp

Những lỗi Unit Test KHÔNG phát hiện được:
  ✗ SQL query sai → chạy H2 thì OK nhưng PostgreSQL thì fail
  ✗ Transaction rollback không đúng do sai propagation
  ✗ Bean wiring sai → @Autowired inject nhầm implementation
  ✗ Kafka consumer deserialize message sai format
  ✗ Security configuration chặn sai endpoint
  ✗ N+1 query chỉ xuất hiện với dữ liệu thực
```

---

## 2. @SpringBootTest — Tải Toàn Bộ Application Context

### Cấu Hình Cơ Bản

```java
@SpringBootTest(
    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT
    // RANDOM_PORT: khởi động HTTP server với cổng ngẫu nhiên (tránh conflict)
    // DEFINED_PORT: dùng cổng trong application.properties (mặc định 8080)
    // MOCK: không khởi động HTTP server (dùng với MockMvc)
    // NONE: không khởi động HTTP server và không tạo WebApplicationContext
)
@ActiveProfiles("test")  // ← Dùng application-test.properties
class OrderIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;  // ← HTTP client cho integration test

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Test
    void createOrder_endToEnd_persistsAndReturnsOrder() {
        // Gọi API thực, qua Controller → Service → Repository → Database
        CreateOrderRequest request = new CreateOrderRequest("product-1", 2, 50_000L);

        ResponseEntity<OrderResponse> response = restTemplate.postForEntity(
            "/api/orders",
            request,
            OrderResponse.class
        );

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getId()).isNotNull();

        // Xác minh dữ liệu thực sự được lưu vào DB
        UUID orderId = response.getBody().getId();
        assertThat(orderRepository.findById(orderId)).isPresent();
    }
}
```

### application-test.properties

```properties
# src/test/resources/application-test.properties

# Override database (sẽ dùng H2 hoặc Testcontainers)
spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
spring.datasource.driver-class-name=org.h2.Driver
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.hibernate.ddl-auto=create-drop

# Tắt các tính năng không cần cho test
spring.mail.host=localhost
spring.mail.port=3025
management.endpoints.web.exposure.include=health

# Logging
logging.level.org.springframework.test=DEBUG
logging.level.org.hibernate.SQL=DEBUG
```

---

## 3. Testcontainers — Database & Services Thực Trong Docker

### Tại Sao Cần Testcontainers?

```
Vấn đề với H2 in-memory:
  ✗ Không hỗ trợ PostgreSQL-specific SQL (JSONB, arrays, window functions)
  ✗ Không phát hiện PostgreSQL-specific bugs
  ✗ Khác biệt về behavior (NULL handling, string functions)

Giải pháp với Testcontainers:
  ✅ Chạy PostgreSQL thực trong Docker container
  ✅ Giống hệt môi trường production
  ✅ Tự động cleanup sau khi test xong
  ✅ Hỗ trợ PostgreSQL, MySQL, MongoDB, Redis, Kafka, Elasticsearch, ...
```

### Cài Đặt

```xml
<!-- pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>testcontainers-bom</artifactId>
            <version>1.19.8</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>kafka</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Cách 1: Container Per Test Class

```java
@SpringBootTest
@Testcontainers  // ← Kích hoạt Testcontainers support trong JUnit 5
class OrderRepositoryTest {

    @Container  // ← @Container + static = dùng chung cho cả class (khởi động 1 lần)
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("testuser")
        .withPassword("testpass")
        .withInitScript("sql/schema.sql");  // ← Chạy script khi container start

    @DynamicPropertySource  // ← Override Spring properties với container URL
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void findByStatus_returnsMatchingOrders() {
        // Thao tác với PostgreSQL thực
        Order pendingOrder = orderRepository.save(
            Order.builder().status(OrderStatus.PENDING).productId("p-001").build()
        );

        List<Order> pendingOrders = orderRepository.findByStatus(OrderStatus.PENDING);

        assertThat(pendingOrders).contains(pendingOrder);
    }
}
```

### Cách 2: Shared Container — Tái Sử Dụng Cho Nhiều Test Class

```java
// Base class dùng chung — khởi động container một lần cho toàn bộ test suite
public abstract class AbstractIntegrationTest {

    static final PostgreSQLContainer<?> POSTGRES;
    static final KafkaContainer KAFKA;

    static {
        POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

        KAFKA = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"));

        // Khởi động song song để tiết kiệm thời gian
        Startables.deepStart(POSTGRES, KAFKA).join();
    }

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
        registry.add("spring.kafka.bootstrap-servers", KAFKA::getBootstrapServers);
    }
}

// Các test class kế thừa
@SpringBootTest
class OrderServiceIT extends AbstractIntegrationTest {
    // Dùng chung POSTGRES và KAFKA đã khởi động
}

@SpringBootTest
class PaymentServiceIT extends AbstractIntegrationTest {
    // Không cần khởi động lại container
}
```

### Cách 3: @ServiceConnection (Spring Boot 3.1+)

```java
// Spring Boot 3.1+ hỗ trợ tự động cấu hình từ container
@SpringBootTest
@Testcontainers
class ModernIntegrationTest {

    @Container
    @ServiceConnection  // ← Tự động override datasource properties — không cần @DynamicPropertySource
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Container
    @ServiceConnection
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.6.0")
    );

    @Container
    @ServiceConnection
    static RedisContainer redis = new RedisContainer("redis:7-alpine");
}
```

---

## 4. @DirtiesContext — Làm Mới Application Context

```java
// @DirtiesContext báo cho Spring biết context bị "bẩn" — cần tạo lại

@SpringBootTest
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
// Tạo lại context sau MỖI test method — tốn kém nhất

@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
// Tạo lại context sau KHI HẾT cả class — mặc định và tiết kiệm nhất

@DirtiesContext(methodMode = DirtiesContext.MethodMode.BEFORE_METHOD)
// Tạo lại context TRƯỚC test method cụ thể

class StatefulIntegrationTest {

    @Autowired
    private ApplicationContext context;

    @Test
    @DirtiesContext  // ← Áp dụng cho test method này
    void testThatModifiesApplicationState() {
        // Test này thay đổi state của context (VD: singleton bean state)
        // → Context sẽ bị tạo lại cho test tiếp theo
    }
}
```

> **Lưu Ý:** `@DirtiesContext` làm chậm test rất nhiều vì phải load lại Spring Context.
> Chỉ dùng khi thực sự cần thiết. Ưu tiên thiết kế test stateless.

---

## 5. Quản Lý Dữ Liệu Test

### @Transactional Trong Test

```java
@SpringBootTest
@Transactional  // ← Mỗi test method chạy trong transaction và rollback sau khi xong
class OrderServiceTransactionalTest {

    @Autowired private OrderService orderService;
    @Autowired private OrderRepository orderRepository;

    @Test
    void createOrder_shouldPersistInTransaction() {
        orderService.createOrder(request);

        // Dữ liệu TỒN TẠI trong test (cùng transaction)
        assertThat(orderRepository.count()).isEqualTo(1);

        // Sau khi test kết thúc → ROLLBACK → DB sạch sẽ
    }

    @Test
    @Rollback(false)  // ← Không rollback — dữ liệu giữ lại
    void createOrder_noRollback() {
        // Test này sẽ commit dữ liệu thực sự
        // CẢNH BÁO: Có thể ảnh hưởng đến các test khác!
    }
}
```

### Dùng @Sql và @SqlGroup

```java
@SpringBootTest
class OrderQueryTest {

    @Test
    @Sql(scripts = "/test-data/insert-orders.sql",
         executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
    @Sql(scripts = "/test-data/cleanup-orders.sql",
         executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
    void findPendingOrders_returnsCorrectData() {
        // Dữ liệu đã được insert bởi SQL script trước khi test chạy
        List<Order> pending = orderRepository.findByStatus(OrderStatus.PENDING);
        assertThat(pending).hasSize(3); // Khớp với dữ liệu trong SQL script
    }
}
```

---

## 6. Integration Test Với Kafka

```java
@SpringBootTest
@Testcontainers
class OrderEventIT extends AbstractIntegrationTest {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @Autowired
    private OrderEventConsumer orderEventConsumer;

    @Test
    void publishOrderEvent_consumerReceivesAndProcesses() throws InterruptedException {
        // Tạo countdown latch để chờ consumer xử lý
        CountDownLatch latch = new CountDownLatch(1);
        orderEventConsumer.setProcessedCallback(latch::countDown);

        // Publish event
        OrderEvent event = new OrderEvent(UUID.randomUUID(), "ORDER_CREATED", LocalDateTime.now());
        kafkaTemplate.send("order-events", event.getOrderId().toString(), event);

        // Chờ tối đa 10 giây cho consumer xử lý
        boolean processed = latch.await(10, TimeUnit.SECONDS);
        assertThat(processed).isTrue();

        // Verify consumer đã xử lý đúng
        assertThat(orderEventConsumer.getLastProcessedEvent()).isNotNull();
        assertThat(orderEventConsumer.getLastProcessedEvent().getType())
            .isEqualTo("ORDER_CREATED");
    }
}
```

---

## 7. Tối Ưu Tốc Độ Integration Test

### Vấn Đề: Context Loading Chậm

```
Mỗi lần @SpringBootTest → Spring phải:
  1. Scan tất cả @Component, @Service, @Repository  (~1–3 giây)
  2. Kết nối database, khởi động Kafka consumer, ... (~2–5 giây)
  3. Chạy test (vài milli giây đến vài giây)

Nếu có 50 test class với @SpringBootTest → 50 lần load context!
```

### Giải Pháp: Context Caching (Bộ Nhớ Đệm Context)

```java
// Spring tự động cache context nếu configuration GIỐNG NHAU
// → Các test class dùng cùng @SpringBootTest config sẽ DÙNG CHUNG context

// ✅ Cùng dùng @SpringBootTest(webEnvironment = RANDOM_PORT) + @ActiveProfiles("test")
// → Spring cache và tái sử dụng → chỉ load 1 lần

// ❌ Mỗi class dùng @MockBean khác nhau → context khác nhau → load lại
// → Gom tất cả @MockBean vào base class để tái sử dụng context
@SpringBootTest
abstract class BaseIntegrationTest {
    // Khai báo tất cả @MockBean cần thiết ở đây
    @MockBean EmailService emailService;
    @MockBean PaymentGateway paymentGateway;
}

class OrderIT extends BaseIntegrationTest { /* tái sử dụng context */ }
class ProductIT extends BaseIntegrationTest { /* tái sử dụng context */ }
```

### Tách Biệt Test Types

```java
// Dùng tag để chạy riêng biệt
@Tag("integration")
@SpringBootTest
class SlowIntegrationTest { }

@Tag("unit")
@ExtendWith(MockitoExtension.class)
class FastUnitTest { }
```

```xml
<!-- Chạy chỉ unit tests trong local development -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <groups>unit</groups>               <!-- Chỉ unit tests -->
        <excludedGroups>integration</excludedGroups>
    </configuration>
</plugin>

<!-- Chạy tất cả trong CI/CD -->
<profile>
    <id>ci</id>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <groups>unit,integration</groups>
                </configuration>
            </plugin>
        </plugins>
    </build>
</profile>
```

---

## 8. Ví Dụ Thực Tế — End-to-End Test Đặt Hàng

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@Testcontainers
class OrderEndToEndIT extends AbstractIntegrationTest {

    @LocalServerPort
    private int port;

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private ProductRepository productRepository;

    @BeforeEach
    void setUp() {
        // Chuẩn bị dữ liệu test
        productRepository.save(Product.builder()
            .id("laptop-pro")
            .name("Laptop Pro")
            .price(30_000_000L)
            .stockQuantity(5)
            .build());
    }

    @AfterEach
    void tearDown() {
        orderRepository.deleteAll();
        productRepository.deleteAll();
    }

    @Test
    void fullOrderFlow_createPayCancel() {
        // BƯỚC 1: Tạo đơn hàng
        CreateOrderRequest createRequest = new CreateOrderRequest("laptop-pro", 1);
        ResponseEntity<OrderResponse> createResponse = restTemplate.postForEntity(
            "/api/orders", createRequest, OrderResponse.class
        );
        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        UUID orderId = createResponse.getBody().getId();

        // BƯỚC 2: Xem chi tiết đơn hàng
        ResponseEntity<OrderResponse> getResponse = restTemplate.getForEntity(
            "/api/orders/" + orderId, OrderResponse.class
        );
        assertThat(getResponse.getBody().getStatus()).isEqualTo(OrderStatus.PENDING);
        assertThat(getResponse.getBody().getTotalPrice()).isEqualTo(30_000_000L);

        // BƯỚC 3: Thanh toán
        PaymentRequest paymentRequest = new PaymentRequest(orderId, "CREDIT_CARD");
        ResponseEntity<Void> payResponse = restTemplate.postForEntity(
            "/api/payments", paymentRequest, Void.class
        );
        assertThat(payResponse.getStatusCode()).isEqualTo(HttpStatus.OK);

        // BƯỚC 4: Xác minh trạng thái cập nhật
        OrderResponse paidOrder = restTemplate
            .getForObject("/api/orders/" + orderId, OrderResponse.class);
        assertThat(paidOrder.getStatus()).isEqualTo(OrderStatus.PAID);

        // BƯỚC 5: Xác minh tồn kho giảm
        Product updatedProduct = productRepository.findById("laptop-pro").orElseThrow();
        assertThat(updatedProduct.getStockQuantity()).isEqualTo(4); // Giảm từ 5 xuống 4
    }
}
```

---

## 9. Checklist Integration Testing

- [ ] Dùng Testcontainers thay vì H2 khi cần SQL PostgreSQL-specific
- [ ] Gom `@MockBean` vào base class để Spring có thể cache context
- [ ] Cleanup dữ liệu test trong `@AfterEach` hoặc dùng `@Transactional`
- [ ] Đánh tag `@Tag("integration")` để tách khỏi unit tests
- [ ] Dùng `@ActiveProfiles("test")` với `application-test.properties` riêng
- [ ] Tránh `@DirtiesContext` nếu không thực sự cần — rất tốn kém
- [ ] Shared container (static field) để khởi động Docker container một lần

---

## 📚 Tài Liệu Tham Khảo

- [Spring Boot Testing Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing)
- [Testcontainers Java Documentation](https://java.testcontainers.org/)
- [Testcontainers Spring Boot](https://java.testcontainers.org/frameworks/spring_boot/)

---

## 🎯 Câu Hỏi Phỏng Vấn

- [ ] Tại sao Testcontainers tốt hơn H2 cho integration test?
- [ ] `@DirtiesContext` làm gì? Khi nào nên và không nên dùng?
- [ ] Spring Context Caching hoạt động thế nào? Điều gì khiến cache bị invalidate?
- [ ] `webEnvironment = RANDOM_PORT` vs `MOCK` — khác biệt gì?
- [ ] Làm thế nào để integration test chạy nhanh hơn khi số lượng test tăng?
- [ ] `@Sql` và `@Transactional` trong test — chúng tương tác với nhau thế nào?
