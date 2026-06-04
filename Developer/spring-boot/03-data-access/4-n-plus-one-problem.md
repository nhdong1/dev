# N+1 Problem — Phát Hiện và Giải Quyết

> N+1 Problem là vấn đề hiệu năng phổ biến nhất và nguy hiểm nhất khi làm việc với ORM.
> Một query vô hại trên môi trường development có thể gây timeout và sập hệ thống ở production
> khi dữ liệu tăng lên. Đây là chủ đề được hỏi nhiều nhất trong phỏng vấn Spring Boot.

---

## 📋 Mục Tiêu

- [ ] Giải thích **N+1 Problem** và tại sao nó nguy hiểm
- [ ] Phát hiện N+1 bằng **Hibernate Statistics** và **query logging**
- [ ] Giải quyết với **`JOIN FETCH`** trong JPQL
- [ ] Dùng **`@EntityGraph`** để khai báo fetch plan
- [ ] Áp dụng **`@BatchSize`** cho lazy collections
- [ ] Dùng **DTO Projection** để chỉ lấy đúng dữ liệu cần thiết
- [ ] Hiểu **MultipleBagFetchException** và cách tránh

---

## 1. N+1 Problem Là Gì?

### Ví Dụ Điển Hình

```java
// Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByStatus(OrderStatus status);
}

// Service
@Transactional(readOnly = true)
public List<OrderResponse> getPaidOrders() {
    List<Order> orders = orderRepository.findByStatus(OrderStatus.PAID); // Query 1

    return orders.stream()
        .map(order -> {
            // Mỗi order.getUser() → Trigger 1 SELECT thêm vì user lazy-loaded!
            String userName = order.getUser().getFullName();  // +1 query
            // Mỗi order.getItems().size() → Trigger 1 SELECT thêm!
            int itemCount = order.getItems().size();           // +1 query nữa!
            return new OrderResponse(order.getId(), userName, itemCount);
        })
        .toList();
}
```

```
Nếu có 100 orders → Database nhận:
  1 query: SELECT * FROM orders WHERE status = 'PAID'
  100 queries: SELECT * FROM users WHERE id = ?       (1 per order)
  100 queries: SELECT * FROM order_items WHERE order_id = ? (1 per order)
  ─────────────────────────────────────────────────────────────
  TỔNG: 201 queries thay vì 1 query!

Nếu có 1.000 orders → 2.001 queries
Nếu có 10.000 orders → 20.001 queries  ← Sập hệ thống!
```

### Tại Sao Xảy Ra?

```
Hibernate dùng LAZY loading theo mặc định:
- @OneToMany → LAZY (không load ngay)
- @ManyToMany → LAZY (không load ngay)

Khi truy cập lazy field trong vòng lặp:
  for (Order order : orders) {
      order.getUser()  ← Hibernate thấy: "User chưa load → SELECT ngay lập tức"
  }
  → Mỗi iteration = 1 SELECT = N queries
  → Tổng = 1 (query chính) + N (lazy load queries) = N+1
```

---

## 2. Phát Hiện N+1 Problem

### Cách 1 — Enable Hibernate Statistics

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true  # Bật thống kê Hibernate

logging:
  level:
    org.hibernate.stat: DEBUG         # Log statistics
    org.hibernate.SQL: DEBUG          # Log tất cả SQL
    org.hibernate.orm.jdbc.bind: TRACE # Log bind parameters
```

```
Output khi có N+1:
  Hibernate: select o1_0.id,o1_0.status,o1_0.user_id from orders o1_0 where o1_0.status=?
  Hibernate: select u1_0.id,u1_0.full_name from users u1_0 where u1_0.id=?
  Hibernate: select u1_0.id,u1_0.full_name from users u1_0 where u1_0.id=?
  Hibernate: select u1_0.id,u1_0.full_name from users u1_0 where u1_0.id=?
  ... (lặp 100 lần)
  
  Session Metrics:
    queries executed to database: 101
    ← Dấu hiệu rõ ràng của N+1!
```

### Cách 2 — datasource-proxy / p6spy

```xml
<!-- pom.xml — thêm vào dev dependencies -->
<dependency>
    <groupId>com.github.gavlyukovskiy</groupId>
    <artifactId>p6spy-spring-boot-starter</artifactId>
    <version>1.9.0</version>
    <scope>runtime</scope>
</dependency>
```

```
Output của p6spy — rõ ràng hơn:
  p6spy - 2 ms: select * from orders where status = 'PAID'
  p6spy - 1 ms: select * from users where id = 1
  p6spy - 1 ms: select * from users where id = 2
  p6spy - 1 ms: select * from users where id = 3
  ...
```

### Cách 3 — Test Phát Hiện N+1 Tự Động

```java
// Dùng thư viện hibernate-query-count hoặc tự đếm
@DataJpaTest
class OrderRepositoryTest {

    @Test
    void findPaidOrders_shouldNotHaveNPlusOne() {
        // Giả sử đã setup 10 orders với users khác nhau

        // Bật đếm query
        Statistics stats = entityManager.unwrap(Session.class)
                                        .getSessionFactory()
                                        .getStatistics();
        stats.clear();

        List<Order> orders = orderRepository.findByStatus(OrderStatus.PAID);
        orders.forEach(o -> o.getUser().getFullName()); // Trigger lazy load

        // Chỉ chấp nhận tối đa 2 queries (1 order + 1 user JOIN)
        assertThat(stats.getPrepareStatementCount()).isLessThanOrEqualTo(2);
    }
}
```

---

## 3. Giải Pháp 1 — JOIN FETCH trong JPQL

### Fetch Join Cơ Bản

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // ❌ Gây N+1 — user và items lazy loaded
    @Query("SELECT o FROM Order o WHERE o.status = :status")
    List<Order> findByStatus(@Param("status") OrderStatus status);

    // ✅ JOIN FETCH — load user cùng lúc với order
    @Query("SELECT o FROM Order o JOIN FETCH o.user u WHERE o.status = :status")
    List<Order> findByStatusWithUser(@Param("status") OrderStatus status);

    // ✅ JOIN FETCH nhiều relationship cùng lúc
    // ⚠️ Chỉ được FETCH 1 collection (items) — xem MultipleBagFetchException
    @Query("""
           SELECT DISTINCT o FROM Order o
           JOIN FETCH o.user u
           JOIN FETCH o.items i
           WHERE o.status = :status
           """)
    List<Order> findByStatusWithUserAndItems(@Param("status") OrderStatus status);

    // ✅ Fetch một entity cụ thể
    @Query("SELECT o FROM Order o JOIN FETCH o.user JOIN FETCH o.items WHERE o.id = :id")
    Optional<Order> findByIdWithDetails(@Param("id") Long id);
}
```

```
SQL được generate (1 query thay vì N+1):
  SELECT DISTINCT o.id, o.status, u.id, u.full_name, i.id, i.product_id, i.quantity
  FROM orders o
  INNER JOIN users u ON o.user_id = u.id
  INNER JOIN order_items i ON i.order_id = o.id
  WHERE o.status = 'PAID'
```

### JOIN FETCH với Pagination — Vấn Đề Đặc Biệt

```java
// ❌ NGUY HIỂM — JOIN FETCH với collection + Pagination!
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
Page<Order> findByStatusWithItems(@Param("status") OrderStatus status, Pageable pageable);
// Hibernate cảnh báo: "HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory!"
// → Hibernate load TOÀN BỘ bảng vào memory rồi phân trang ở Java → thảm họa!

// ✅ Giải pháp: 2-step approach
@Query(value = "SELECT o FROM Order o WHERE o.status = :status",
       countQuery = "SELECT COUNT(o) FROM Order o WHERE o.status = :status")
Page<Order> findByStatus(@Param("status") OrderStatus status, Pageable pageable);

// Service: Lấy IDs trước, rồi fetch với details
@Transactional(readOnly = true)
public Page<OrderResponse> getPaidOrdersPaged(Pageable pageable) {
    // Step 1: Phân trang chỉ với orders (không fetch items)
    Page<Order> ordersPage = orderRepository.findByStatus(OrderStatus.PAID, pageable);

    // Step 2: Fetch chi tiết cho đúng orders trong trang hiện tại
    List<Long> orderIds = ordersPage.getContent().stream()
                                    .map(Order::getId)
                                    .toList();
    List<Order> ordersWithDetails = orderRepository.findByIdsWithDetails(orderIds);

    // Step 3: Map và trả về
    return ordersPage.map(order -> {
        Order detailed = ordersWithDetails.stream()
            .filter(o -> o.getId().equals(order.getId()))
            .findFirst()
            .orElse(order);
        return mapper.toResponse(detailed);
    });
}

@Query("SELECT o FROM Order o JOIN FETCH o.user JOIN FETCH o.items WHERE o.id IN :ids")
List<Order> findByIdsWithDetails(@Param("ids") List<Long> ids);
```

---

## 4. Giải Pháp 2 — @EntityGraph

`@EntityGraph` cho phép khai báo fetch plan tại Repository mà không cần viết JPQL.

### Named EntityGraph — Khai Báo Trên Entity

```java
// Khai báo EntityGraph trên Entity
@Entity
@Table(name = "orders")
@NamedEntityGraph(
    name = "order-with-user-and-items",           // Tên để tham chiếu
    attributeNodes = {
        @NamedAttributeNode("user"),               // Fetch user
        @NamedAttributeNode(value = "items",       // Fetch items
                            subgraph = "item-product"), // Kèm theo subgraph
    },
    subgraphs = {
        @NamedSubgraph(
            name = "item-product",
            attributeNodes = @NamedAttributeNode("product") // Fetch product của mỗi item
        )
    }
)
public class Order {
    @Id private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    private User user;

    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;
}
```

```java
// Repository dùng @EntityGraph
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Dùng named graph đã khai báo trên Entity
    @EntityGraph(value = "order-with-user-and-items", type = EntityGraph.EntityGraphType.FETCH)
    Optional<Order> findById(Long id);

    // Inline EntityGraph — không cần khai báo trên Entity
    @EntityGraph(attributePaths = {"user", "items"})
    List<Order> findByStatus(OrderStatus status);

    // EntityGraph kết hợp với @Query
    @EntityGraph(attributePaths = {"user", "items", "items.product"})
    @Query("SELECT o FROM Order o WHERE o.createdAt >= :fromDate")
    List<Order> findRecentOrdersWithDetails(@Param("fromDate") LocalDateTime fromDate);
}
```

### EntityGraph.EntityGraphType — Hai Loại

```
FETCH (Phổ biến hơn):
    Các field trong graph → EAGER
    Các field KHÔNG trong graph → LAZY (override annotation trên Entity)
    → Chỉ load đúng những gì khai báo

LOAD:
    Các field trong graph → EAGER
    Các field KHÔNG trong graph → Giữ nguyên FetchType gốc trên annotation
    → Ít kiểm soát hơn, ít dùng
```

---

## 5. Giải Pháp 3 — @BatchSize (Hibernate-Specific)

`@BatchSize` không giải quyết N+1 hoàn toàn nhưng giảm số query từ N xuống còn N/batchSize.

```java
@Entity
public class User {

    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    @BatchSize(size = 50)  // Load 50 orders một lần thay vì 1 lần
    private List<Order> orders;
}

// Hoặc cấu hình global trong application.yml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 50
```

```
Nếu có 100 users, mỗi user lazy-load orders:
  Không BatchSize:  100 queries SELECT orders WHERE user_id = ?
  @BatchSize(50):   2 queries SELECT orders WHERE user_id IN (1,2,...,50)
                              SELECT orders WHERE user_id IN (51,52,...,100)
  
  N+1 → N/50 + 1
```

```java
// ✅ Kết hợp BatchSize với Pagination — hiệu quả cao
@Transactional(readOnly = true)
public Page<UserResponse> getUsersWithOrderCount(Pageable pageable) {
    Page<User> users = userRepository.findAll(pageable);
    // Khi map users → access orders → BatchSize gom 50 users một batch
    return users.map(user -> new UserResponse(
        user.getId(),
        user.getEmail(),
        user.getOrders().size()  // Lazy + BatchSize → không N+1
    ));
}
```

---

## 6. Giải Pháp 4 — DTO Projection (Tốt Nhất Cho Performance)

Khi chỉ cần một số field, DTO Projection tránh load toàn bộ Entity và các relationship.

```java
// Interface Projection — Spring Data tạo proxy
public interface OrderSummaryProjection {
    Long getId();
    BigDecimal getTotal();
    String getStatus();
    String getUserEmail();      // Field từ joined table — dùng alias
    Integer getItemCount();     // Computed từ subquery
}

// DTO Record — type-safe hơn
public record OrderSummaryDto(
    Long id,
    BigDecimal total,
    String status,
    String userEmail,
    Long itemCount
) {}

// Repository với DTO projection query — KHÔNG cần load Entity
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Interface projection
    @Query("""
           SELECT o.id AS id,
                  o.total AS total,
                  o.status AS status,
                  u.email AS userEmail,
                  COUNT(i.id) AS itemCount
           FROM Order o
           JOIN o.user u
           LEFT JOIN o.items i
           WHERE o.status = :status
           GROUP BY o.id, o.total, o.status, u.email
           """)
    List<OrderSummaryProjection> findSummaryByStatus(@Param("status") OrderStatus status);

    // DTO Record projection — constructor expression trong JPQL
    @Query("""
           SELECT new com.example.dto.OrderSummaryDto(
               o.id, o.total, CAST(o.status AS string), u.email, COUNT(i)
           )
           FROM Order o
           JOIN o.user u
           LEFT JOIN o.items i
           WHERE o.status = :status
           GROUP BY o.id, o.total, o.status, u.email
           """)
    List<OrderSummaryDto> findDtoByStatus(@Param("status") OrderStatus status);

    // Native query với DTO mapping
    @Query(value = """
           SELECT o.id, o.total, o.status,
                  u.email AS user_email,
                  COUNT(i.id) AS item_count
           FROM orders o
           JOIN users u ON o.user_id = u.id
           LEFT JOIN order_items i ON i.order_id = o.id
           WHERE o.status = :status
           GROUP BY o.id, o.total, o.status, u.email
           """,
           nativeQuery = true)
    List<OrderSummaryProjection> findSummaryNative(@Param("status") String status);
}
```

---

## 7. MultipleBagFetchException — Không Thể FETCH 2 Collection

```java
// ❌ LỖI — Không thể fetch 2 Bag (List) cùng lúc!
@Query("""
       SELECT o FROM Order o
       JOIN FETCH o.items     ← Bag 1
       JOIN FETCH o.tags      ← Bag 2
       WHERE o.status = :status
       """)
List<Order> findWithItemsAndTags(@Param("status") OrderStatus status);
// → org.hibernate.loader.MultipleBagFetchException: cannot simultaneously fetch multiple bags

// ✅ Giải pháp 1: Dùng Set thay vì List (không có thứ tự, không dup)
@OneToMany(mappedBy = "order")
private Set<OrderItem> items;  // Set thay List

// ✅ Giải pháp 2: 2 queries riêng biệt
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id IN :ids")
List<Order> findWithItems(@Param("ids") List<Long> ids);

@Query("SELECT o FROM Order o JOIN FETCH o.tags WHERE o.id IN :ids")
List<Order> findWithTags(@Param("ids") List<Long> ids);

// Service: Merge kết quả
@Transactional(readOnly = true)
public List<Order> findOrdersWithAllDetails(List<Long> ids) {
    List<Order> ordersWithItems = orderRepository.findWithItems(ids);
    List<Order> ordersWithTags = orderRepository.findWithTags(ids);

    // Hibernate first-level cache đảm bảo cùng instance
    // → ordersWithItems[0] và ordersWithTags[0] là CÙNG object trong memory
    // → Sau 2 queries, object đã có cả items lẫn tags
    return ordersWithItems; // items và tags đều đã được populate
}

// ✅ Giải pháp 3: @BatchSize cho từng collection
@OneToMany(mappedBy = "order")
@BatchSize(size = 50)
private List<OrderItem> items;

@ManyToMany
@BatchSize(size = 50)
private List<Tag> tags;
```

---

## 8. Tổng Hợp — Khi Nào Dùng Giải Pháp Nào

| Tình Huống | Giải Pháp | Lý Do |
|-----------|-----------|-------|
| API trả về 1 entity với đầy đủ details | `JOIN FETCH` / `@EntityGraph` | Load đúng, đủ dữ liệu cần |
| List entity + phân trang | 2-step (page IDs → fetch details) | Tránh in-memory pagination |
| Chỉ cần vài field từ nhiều bảng | **DTO Projection** | Nhanh nhất — không load entity |
| Existing code, nhiều lazy relationships | `@BatchSize` global | Giảm N xuống N/batchSize dễ dàng |
| Report / analytics | Native Query hoặc DTO | Tối ưu SQL thủ công |
| 2+ collections cần fetch | Set + JOIN FETCH / 2 queries | Tránh MultipleBagFetchException |

---

## 9. Cấu Hình Phát Hiện N+1 Trong Development

```yaml
# application-dev.yml — chỉ bật trong môi trường dev
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        generate_statistics: true

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.stat: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE  # Log bind params — rất verbose
```

```java
// Custom log filter để đếm số query trong test
@Component
@Profile("test")
public class QueryCountInterceptor implements StatementInspector {

    private final ThreadLocal<AtomicInteger> queryCount = ThreadLocal.withInitial(AtomicInteger::new);

    @Override
    public String inspect(String sql) {
        queryCount.get().incrementAndGet();
        return sql;
    }

    public int getQueryCount() {
        return queryCount.get().get();
    }

    public void reset() {
        queryCount.get().set(0);
    }
}
```

---

## ✅ Checklist N+1 Problem

- [ ] Bật `show-sql: true` và kiểm tra số queries trong development
- [ ] Viết test kiểm tra số lượng queries (không được tăng theo data)
- [ ] Dùng `JOIN FETCH` khi biết trước cần load relationship
- [ ] Dùng `@EntityGraph` khi cần fetch plan linh hoạt hơn
- [ ] Dùng DTO Projection khi chỉ cần một phần dữ liệu
- [ ] Cấu hình `default_batch_fetch_size` để giảm thiểu N+1 fallback
- [ ] Không dùng `FetchType.EAGER` trên collection

---

## 💡 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: N+1 là gì? Mô tả ví dụ thực tế?**
> Nếu có N orders, khi lazy-load user của mỗi order ta có 1 query chính + N queries user = N+1 queries. Với 1.000 orders → 1.001 queries thay vì 1 query.

**Q: Làm sao phát hiện N+1 trong production?**
> Bật `generate_statistics`, dùng p6spy hoặc slow query log của DB. Dấu hiệu: nhiều queries giống hệt nhau chỉ khác WHERE id = ? liên tiếp.

**Q: JOIN FETCH khác @EntityGraph thế nào?**
> JOIN FETCH dùng trong JPQL string, @EntityGraph dùng annotation. @EntityGraph linh hoạt hơn vì có thể kết hợp với bất kỳ derived query method, JOIN FETCH phải viết toàn bộ JPQL.

**Q: Tại sao không dùng EAGER loading cho tất cả?**
> EAGER luôn load relationship dù có cần không — gây load dữ liệu thừa, tăng memory và thời gian query. Cũng có thể gây CartesianProduct khi nhiều EAGER collections.

---

## 🔗 Liên Kết

- [2-orm-mapping.md](2-orm-mapping.md) — FetchType, Lazy/Eager
- [3-transactions.md](3-transactions.md) — Session scope với lazy loading
- [Vlad Mihalcea — N+1 Query Problem](https://vladmihalcea.com/n-plus-1-query-problem/)
- [Baeldung — Spring Data JPA @EntityGraph](https://www.baeldung.com/spring-data-jpa-named-entity-graphs)

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
