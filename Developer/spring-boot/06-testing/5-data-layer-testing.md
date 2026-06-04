# Data Layer Testing với @DataJpaTest

> `@DataJpaTest` (Kiểm Thử Tầng Dữ Liệu) chỉ load JPA infrastructure —
> `@Entity`, `@Repository`, `EntityManager`, DataSource và Flyway/Liquibase migration —
> mà không load toàn bộ Application Context.
> Mặc định chạy trên in-memory H2 database (Cơ Sở Dữ Liệu Trong Bộ Nhớ).

---

## 1. Tại Sao Cần @DataJpaTest?

```
@SpringBootTest:    Load toàn bộ context (Controller, Service, ...)  → Chậm
@DataJpaTest:       Chỉ load JPA layer                              → Nhanh

@DataJpaTest load:
  ✅ @Entity classes
  ✅ @Repository (Spring Data JPA)
  ✅ EntityManager, EntityManagerFactory
  ✅ DataSource (H2 hoặc Testcontainers)
  ✅ Spring Data JPA configuration
  ✅ Flyway / Liquibase (nếu cấu hình)
  ✅ @Transactional (auto rollback sau mỗi test)

@DataJpaTest KHÔNG load:
  ❌ @Controller, @Service, @Component
  ❌ Spring Security
  ❌ @Async, @Scheduled
```

---

## 2. Cài Đặt Cơ Bản

### Repository Cần Test

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, UUID> {

    List<Order> findByStatus(OrderStatus status);

    List<Order> findByUserIdOrderByCreatedAtDesc(UUID userId);

    @Query("SELECT o FROM Order o WHERE o.totalPrice > :minPrice AND o.status = :status")
    List<Order> findExpensiveOrdersByStatus(
        @Param("minPrice") long minPrice,
        @Param("status") OrderStatus status
    );

    @Query(value = """
        SELECT * FROM orders
        WHERE created_at >= :startDate
        AND created_at <= :endDate
        """, nativeQuery = true)
    List<Order> findOrdersInDateRange(
        @Param("startDate") LocalDateTime startDate,
        @Param("endDate") LocalDateTime endDate
    );

    @Modifying
    @Query("UPDATE Order o SET o.status = :newStatus WHERE o.id = :id")
    int updateStatus(@Param("id") UUID id, @Param("newStatus") OrderStatus newStatus);

    @Query("SELECT COUNT(o) FROM Order o WHERE o.status = :status")
    long countByStatus(@Param("status") OrderStatus status);

    boolean existsByIdAndUserId(UUID id, UUID userId);
}
```

### Test Class Cơ Bản

```java
@DataJpaTest  // ← Load chỉ JPA layer, dùng H2 in-memory mặc định
@ActiveProfiles("test")
class OrderRepositoryTest {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private TestEntityManager entityManager;  // ← Tiện ích test — persist, flush, find

    private UUID testUserId;

    @BeforeEach
    void setUp() {
        testUserId = UUID.randomUUID();
    }

    // @Transactional + rollback tự động sau mỗi test — DB luôn sạch
}
```

---

## 3. TestEntityManager — Chuẩn Bị Dữ Liệu Test

`TestEntityManager` (Trình Quản Lý Thực Thể Test) cung cấp các method tiện lợi hơn `EntityManager` thông thường:

```java
@Test
void findByStatus_withMultipleOrders_returnsOnlyMatchingStatus() {
    // Dùng TestEntityManager để persist trực tiếp — bỏ qua Repository logic
    Order pendingOrder = entityManager.persistAndFlush(
        Order.builder()
            .userId(testUserId)
            .productId("p-001")
            .status(OrderStatus.PENDING)
            .totalPrice(100_000L)
            .createdAt(LocalDateTime.now())
            .build()
    );

    Order paidOrder = entityManager.persistAndFlush(
        Order.builder()
            .userId(testUserId)
            .productId("p-002")
            .status(OrderStatus.PAID)
            .totalPrice(200_000L)
            .createdAt(LocalDateTime.now())
            .build()
    );

    // Clear persistence context để đảm bảo query thực sự truy vấn DB
    entityManager.clear();

    // Test Repository method
    List<Order> pendingOrders = orderRepository.findByStatus(OrderStatus.PENDING);

    assertThat(pendingOrders).hasSize(1);
    assertThat(pendingOrders.get(0).getId()).isEqualTo(pendingOrder.getId());
    assertThat(pendingOrders.get(0).getStatus()).isEqualTo(OrderStatus.PENDING);
}
```

### Các Phương Thức TestEntityManager

```java
// persist() + flush() — lưu vào DB ngay lập tức
Order saved = entityManager.persistAndFlush(order);

// persist() — lưu nhưng chưa flush (chưa ghi DB)
Order persisted = entityManager.persist(order);
entityManager.flush();  // Flush thủ công khi cần

// find() — tìm theo primary key
Order found = entityManager.find(Order.class, orderId);

// clear() — xóa persistence context cache
// → Buộc query tiếp theo thực sự đọc từ DB (không phải từ first-level cache)
entityManager.clear();

// merge() — cập nhật entity đã detached
Order merged = entityManager.merge(detachedOrder);

// remove() — xóa entity
entityManager.remove(entityManager.find(Order.class, orderId));
entityManager.flush();
```

---

## 4. Test Custom @Query Methods

### JPQL Query

```java
@Test
void findExpensiveOrdersByStatus_returnsOrdersAboveThreshold() {
    // Chuẩn bị: 3 orders với giá khác nhau
    entityManager.persistAndFlush(createOrder("p-001", OrderStatus.PENDING, 50_000L));
    entityManager.persistAndFlush(createOrder("p-002", OrderStatus.PENDING, 150_000L));
    entityManager.persistAndFlush(createOrder("p-003", OrderStatus.PENDING, 250_000L));
    entityManager.persistAndFlush(createOrder("p-004", OrderStatus.PAID, 300_000L));  // PAID
    entityManager.clear();

    long minPrice = 100_000L;
    List<Order> result = orderRepository.findExpensiveOrdersByStatus(minPrice, OrderStatus.PENDING);

    assertThat(result).hasSize(2);  // Chỉ 150_000 và 250_000 — PENDING và >= 100_000
    assertThat(result)
        .extracting(Order::getTotalPrice)
        .containsExactlyInAnyOrder(150_000L, 250_000L);
    assertThat(result)
        .extracting(Order::getStatus)
        .containsOnly(OrderStatus.PENDING);
}
```

### Native Query

```java
@Test
void findOrdersInDateRange_returnsCorrectOrders() {
    LocalDateTime startDate = LocalDateTime.of(2026, 1, 1, 0, 0);
    LocalDateTime endDate = LocalDateTime.of(2026, 6, 30, 23, 59);
    LocalDateTime withinRange = LocalDateTime.of(2026, 3, 15, 12, 0);
    LocalDateTime outsideRange = LocalDateTime.of(2025, 12, 31, 23, 59);

    Order withinRangeOrder = Order.builder()
        .userId(testUserId).productId("p-001").status(OrderStatus.PENDING)
        .totalPrice(100_000L).createdAt(withinRange).build();
    Order outsideRangeOrder = Order.builder()
        .userId(testUserId).productId("p-002").status(OrderStatus.PENDING)
        .totalPrice(100_000L).createdAt(outsideRange).build();

    entityManager.persistAndFlush(withinRangeOrder);
    entityManager.persistAndFlush(outsideRangeOrder);
    entityManager.clear();

    List<Order> result = orderRepository.findOrdersInDateRange(startDate, endDate);

    assertThat(result).hasSize(1);
    assertThat(result.get(0).getCreatedAt()).isEqualTo(withinRange);
}
```

### @Modifying Query

```java
@Test
void updateStatus_withExistingOrder_updatesAndReturnsCount() {
    Order order = entityManager.persistAndFlush(
        createOrder("p-001", OrderStatus.PENDING, 100_000L)
    );
    entityManager.clear();

    int updatedCount = orderRepository.updateStatus(order.getId(), OrderStatus.PAID);

    assertThat(updatedCount).isEqualTo(1);

    // Xác minh thay đổi thực sự được lưu
    entityManager.clear();  // ← Clear cache để đọc từ DB
    Order updated = entityManager.find(Order.class, order.getId());
    assertThat(updated.getStatus()).isEqualTo(OrderStatus.PAID);
}

@Test
void updateStatus_withNonExistentId_returnsZero() {
    int updatedCount = orderRepository.updateStatus(UUID.randomUUID(), OrderStatus.PAID);
    assertThat(updatedCount).isEqualTo(0);
}
```

---

## 5. Test Relationship Loading (Tải Quan Hệ)

### Kiểm Tra Lazy Loading

```java
@Test
void findById_lazyLoadedOrderItems_shouldNotBeLoadedImmediately() {
    Order order = entityManager.persistAndFlush(
        Order.builder()
            .userId(testUserId)
            .status(OrderStatus.PENDING)
            .build()
    );

    // Persist order items
    entityManager.persistAndFlush(OrderItem.builder()
        .order(order).productId("p-001").quantity(2).price(50_000L).build());
    entityManager.persistAndFlush(OrderItem.builder()
        .order(order).productId("p-002").quantity(1).price(100_000L).build());
    entityManager.clear();

    Order found = orderRepository.findById(order.getId()).orElseThrow();

    // Kiểm tra items chưa được load (lazy)
    // Hibernate.isInitialized() trả về false nếu proxy chưa khởi tạo
    assertThat(Hibernate.isInitialized(found.getItems())).isFalse();

    // Trigger lazy loading
    int itemCount = found.getItems().size();
    assertThat(itemCount).isEqualTo(2);
    assertThat(Hibernate.isInitialized(found.getItems())).isTrue();
}
```

### Test @EntityGraph hoặc JOIN FETCH

```java
// Repository method với EntityGraph
@EntityGraph(attributePaths = {"items", "items.product"})
@Query("SELECT o FROM Order o WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") UUID id);

// Test
@Test
void findByIdWithItems_shouldEagerLoadItems() {
    // ... persist order with items
    entityManager.clear();

    Order found = orderRepository.findByIdWithItems(orderId).orElseThrow();

    // Items đã được load sẵn — không cần thêm query
    assertThat(Hibernate.isInitialized(found.getItems())).isTrue();
    assertThat(found.getItems()).hasSize(2);
}
```

---

## 6. Dùng Testcontainers Với @DataJpaTest

H2 không hỗ trợ một số tính năng PostgreSQL. Dùng Testcontainers khi cần SQL PostgreSQL-specific:

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
// ↑ QUAN TRỌNG: Không dùng H2 — dùng datasource từ @DynamicPropertySource
@Testcontainers
class OrderRepositoryWithPostgresTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void nativePostgresFeature_jsonbQuery_works() {
        // Test với PostgreSQL JSONB, array types, window functions, v.v.
        // Không thể test với H2
    }
}
```

---

## 7. Test Với Flyway Migration

```java
// Khi dùng Flyway, @DataJpaTest tự động chạy migration scripts
@DataJpaTest
class OrderRepositoryWithFlywayTest {

    // Flyway tự động chạy:
    // V1__create_tables.sql
    // V2__add_indexes.sql
    // V3__seed_initial_data.sql
    // → Schema đúng như production

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void existingDataFromMigration_shouldBeAccessible() {
        // Nếu có seed data trong migration scripts
        List<Order> seededOrders = orderRepository.findAll();
        // ... assertions
    }
}
```

---

## 8. Test Constraints và Validation

### Unique Constraint

```java
@Test
void save_withDuplicateOrderNumber_throwsConstraintViolation() {
    String orderNumber = "ORD-2026-001";
    entityManager.persistAndFlush(
        Order.builder().orderNumber(orderNumber).userId(testUserId).build()
    );

    // Attempt duplicate
    assertThatThrownBy(() -> {
        entityManager.persistAndFlush(
            Order.builder().orderNumber(orderNumber).userId(testUserId).build()
        );
    })
    .isInstanceOf(DataIntegrityViolationException.class);
}
```

### Not Null Constraint

```java
@Test
void save_withNullRequiredField_throwsConstraintViolation() {
    Order orderWithoutStatus = Order.builder()
        .userId(testUserId)
        .productId("p-001")
        // status bị null — NOT NULL constraint
        .build();

    assertThatThrownBy(() -> entityManager.persistAndFlush(orderWithoutStatus))
        .isInstanceOf(ConstraintViolationException.class);
}
```

---

## 9. Tổ Chức Test Data — Builder Helper

```java
// Helper method tránh lặp code
@DataJpaTest
class OrderRepositoryTest {

    private Order buildOrder(OrderStatus status, long price) {
        return Order.builder()
            .userId(testUserId)
            .productId("test-product-" + UUID.randomUUID())
            .status(status)
            .totalPrice(price)
            .createdAt(LocalDateTime.now())
            .build();
    }

    private Order persistOrder(OrderStatus status, long price) {
        return entityManager.persistAndFlush(buildOrder(status, price));
    }

    @Test
    void countByStatus_correctlyCounts() {
        persistOrder(OrderStatus.PENDING, 100_000L);
        persistOrder(OrderStatus.PENDING, 200_000L);
        persistOrder(OrderStatus.PAID, 150_000L);
        entityManager.clear();

        assertThat(orderRepository.countByStatus(OrderStatus.PENDING)).isEqualTo(2);
        assertThat(orderRepository.countByStatus(OrderStatus.PAID)).isEqualTo(1);
        assertThat(orderRepository.countByStatus(OrderStatus.CANCELLED)).isEqualTo(0);
    }
}
```

---

## 10. @DataJpaTest vs Các Test Slice Khác

| Test Slice | Load | Không Load | Khi Dùng |
|-----------|------|------------|----------|
| `@DataJpaTest` | JPA, DataSource, Flyway | Controller, Service, Security | Test Repository |
| `@WebMvcTest` | Controller, Filter, Security | Service, Repository, DB | Test Controller |
| `@DataRedisTest` | Redis, RedisTemplate | Controller, Service, JPA | Test Redis Repository |
| `@DataMongoTest` | MongoTemplate, MongoRepository | Controller, Service, JPA | Test MongoDB Repository |
| `@JsonTest` | Jackson, GSON | Tất cả | Test JSON serialization |
| `@SpringBootTest` | Tất cả | Không | Integration test đầy đủ |

---

## 11. Checklist Data Layer Testing

- [ ] Dùng `TestEntityManager` để persist dữ liệu test — tách biệt với Repository đang test
- [ ] Gọi `entityManager.clear()` trước khi gọi Repository method để tránh cache L1
- [ ] Test cả happy path và edge cases (empty result, not found, constraints violated)
- [ ] Dùng `@AutoConfigureTestDatabase(replace = NONE)` + Testcontainers khi cần PostgreSQL-specific SQL
- [ ] Test `@Modifying` query: kiểm tra return count và actual DB state sau khi update
- [ ] Test lazy/eager loading nếu quan trọng với application behavior

---

## 📚 Tài Liệu Tham Khảo

- [Spring Data JPA Testing](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing.spring-boot-applications.autoconfigured-spring-data-jpa)
- [TestEntityManager Javadoc](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/test/autoconfigure/orm/jpa/TestEntityManager.html)
- [Hibernate.isInitialized()](https://docs.jboss.org/hibernate/orm/current/javadocs/org/hibernate/Hibernate.html)

---

## 🎯 Câu Hỏi Phỏng Vấn

- [ ] `@DataJpaTest` vs `@SpringBootTest` khi test Repository — khi nào dùng cái nào?
- [ ] Tại sao cần gọi `entityManager.clear()` trước khi test Repository query?
- [ ] Khi nào nên dùng Testcontainers thay vì H2 in-memory cho data layer test?
- [ ] `@Transactional` trong `@DataJpaTest` có tác dụng gì?
- [ ] Làm thế nào test `@Modifying` query đúng cách?
- [ ] `AutoConfigureTestDatabase.Replace.NONE` có nghĩa là gì?
