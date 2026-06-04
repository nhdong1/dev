# Spring Data JPA — Repository, Entity, JPQL, Criteria API

> Spring Data JPA (Java Persistence API — Giao Diện Lập Trình Dữ Liệu Java) là tầng trừu tượng trên Hibernate,
> cho phép truy cập cơ sở dữ liệu quan hệ mà không cần viết boilerplate code (code mẫu lặp lại).

---

## 📋 Mục Tiêu

- [ ] Định nghĩa **Entity** (Thực Thể) với `@Entity`, `@Table`, `@Id`, `@Column`
- [ ] Sử dụng **`JpaRepository`** — các method sẵn có và khi nào cần tùy chỉnh
- [ ] Viết **Derived Query Methods** (Phương Thức Truy Vấn Suy Diễn) từ tên method
- [ ] Dùng **`@Query`** với JPQL (Java Persistence Query Language — Ngôn Ngữ Truy Vấn JPA) và Native SQL
- [ ] Implement **Pagination** (Phân Trang) và **Sorting** (Sắp Xếp) với `Pageable`
- [ ] Hiểu **Criteria API** — truy vấn động, type-safe

---

## 1. Entity — Định Nghĩa Thực Thể

### Entity Cơ Bản

```java
import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity                          // Đánh dấu đây là JPA Entity — ánh xạ với bảng DB
@Table(name = "products",        // Tên bảng tùy chỉnh (mặc định: tên class viết thường)
       indexes = {
           @Index(name = "idx_product_sku", columnList = "sku", unique = true),
           @Index(name = "idx_product_category", columnList = "category_id")
       })
public class Product {

    @Id                                               // Khóa chính (Primary Key)
    @GeneratedValue(strategy = GenerationType.IDENTITY) // Auto-increment — DB tự tăng
    private Long id;

    @Column(name = "product_name",   // Tên cột tùy chỉnh
            nullable = false,        // NOT NULL constraint
            length = 200)            // VARCHAR(200)
    private String name;

    @Column(unique = true, nullable = false, length = 50)
    private String sku;              // Stock Keeping Unit — Mã Quản Lý Kho

    @Column(precision = 10, scale = 2)  // DECIMAL(10, 2) — cho giá tiền
    private BigDecimal price;

    @Column(name = "created_at",
            updatable = false)       // Không cho phép update sau khi tạo
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @PrePersist     // Lifecycle callback — gọi trước khi INSERT
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }

    @PreUpdate      // Lifecycle callback — gọi trước khi UPDATE
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }

    // Constructors, Getters, Setters...
}
```

### Các Chiến Lược GenerationType

```
GenerationType.IDENTITY   — DB tự tăng (MySQL AUTO_INCREMENT, PostgreSQL SERIAL)
                            ✅ Đơn giản, phổ biến nhất
                            ❌ Không hỗ trợ batch insert hiệu quả

GenerationType.SEQUENCE   — Dùng DB Sequence (Oracle, PostgreSQL)
                            ✅ Hỗ trợ batch insert tốt hơn
                            ✅ Phù hợp với allocationSize để giảm round-trip

GenerationType.TABLE      — Dùng bảng đặc biệt để quản lý ID
                            ❌ Chậm nhất — tránh dùng trong production

GenerationType.UUID       — Tự động tạo UUID (Java 17+, Spring Boot 3+)
                            ✅ Phù hợp cho distributed systems (hệ thống phân tán)
                            ❌ Index kém hiệu quả hơn số nguyên
```

```java
// ✅ Dùng SEQUENCE với allocationSize để tối ưu batch insert
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE,
                generator = "product_seq")
@SequenceGenerator(name = "product_seq",
                   sequenceName = "product_sequence",
                   allocationSize = 50)  // Lấy trước 50 ID từ DB mỗi lần — giảm round-trip
private Long id;

// ✅ UUID cho microservices
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id;
```

---

## 2. JpaRepository — Repository Interface

### Cây Phả Hệ Repository

```
Repository<T, ID>                                 ← Marker interface (rỗng)
└── CrudRepository<T, ID>                         ← 11 methods CRUD cơ bản
     ├── save(entity)
     ├── saveAll(entities)
     ├── findById(id) → Optional<T>
     ├── existsById(id)
     ├── findAll() → Iterable<T>
     ├── findAllById(ids) → Iterable<T>
     ├── count()
     ├── deleteById(id)
     ├── delete(entity)
     ├── deleteAllById(ids)
     └── deleteAll()
          │
          └── PagingAndSortingRepository<T, ID>   ← Thêm phân trang
               ├── findAll(Sort sort)
               └── findAll(Pageable pageable) → Page<T>
                    │
                    └── JpaRepository<T, ID>      ← JPA-specific methods
                         ├── flush()
                         ├── saveAndFlush(entity)
                         ├── deleteInBatch(entities)
                         ├── deleteAllInBatch()
                         └── getOne(id)           ← Deprecated, dùng getReferenceById()
```

### Tạo Repository

```java
// Cách đơn giản nhất — Spring Data tự generate implementation
public interface ProductRepository extends JpaRepository<Product, Long> {
    // Không cần viết gì — đã có đầy đủ CRUD
}

// Service sử dụng
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;

    public Product findById(Long id) {
        return productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
    }

    public Product save(Product product) {
        return productRepository.save(product);  // INSERT hoặc UPDATE tùy entity có ID chưa
    }

    public Page<Product> findAll(Pageable pageable) {
        return productRepository.findAll(pageable);
    }

    public void delete(Long id) {
        productRepository.deleteById(id);  // Ném exception nếu không tìm thấy
    }
}
```

---

## 3. Derived Query Methods — Phương Thức Truy Vấn Suy Diễn

Spring Data tự động tạo SQL từ tên method. Đây là tính năng "magic" nhất của Spring Data.

### Quy Tắc Đặt Tên

```
find   By   Email          And     Active
 ▲      ▲      ▲            ▲        ▲
Subject Keyword Condition  Connector Condition

find / read / get / query / search / count / exists / delete
```

### Ví Dụ Thực Tế

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // ==========  Tìm kiếm đơn giản  ==========

    Optional<User> findByEmail(String email);
    // SELECT * FROM users WHERE email = ?

    List<User> findByLastName(String lastName);
    // SELECT * FROM users WHERE last_name = ?

    boolean existsByEmail(String email);
    // SELECT COUNT(*) > 0 FROM users WHERE email = ?

    long countByActive(boolean active);
    // SELECT COUNT(*) FROM users WHERE active = ?

    // ==========  Kết hợp nhiều điều kiện  ==========

    List<User> findByFirstNameAndLastName(String firstName, String lastName);
    // WHERE first_name = ? AND last_name = ?

    List<User> findByEmailOrPhoneNumber(String email, String phone);
    // WHERE email = ? OR phone_number = ?

    // ==========  Comparison operators  ==========

    List<User> findByAgeBetween(int min, int max);
    // WHERE age BETWEEN ? AND ?

    List<User> findByAgeGreaterThan(int age);
    // WHERE age > ?

    List<User> findByAgeLessThanEqual(int age);
    // WHERE age <= ?

    List<User> findByCreatedAtAfter(LocalDateTime date);
    // WHERE created_at > ?

    // ==========  String matching  ==========

    List<User> findByFirstNameContaining(String keyword);
    // WHERE first_name LIKE '%keyword%'

    List<User> findByEmailStartingWith(String prefix);
    // WHERE email LIKE 'prefix%'

    List<User> findByLastNameEndingWith(String suffix);
    // WHERE last_name LIKE '%suffix'

    List<User> findByFirstNameContainingIgnoreCase(String keyword);
    // WHERE LOWER(first_name) LIKE '%keyword%'

    // ==========  Null checks  ==========

    List<User> findByPhoneNumberIsNull();
    // WHERE phone_number IS NULL

    List<User> findByDeletedAtIsNotNull();
    // WHERE deleted_at IS NOT NULL

    // ==========  Collection / IN  ==========

    List<User> findByRoleIn(Collection<Role> roles);
    // WHERE role IN (?, ?, ?)

    List<User> findByIdNotIn(Collection<Long> ids);
    // WHERE id NOT IN (?, ?, ?)

    // ==========  Ordering  ==========

    List<User> findByActiveOrderByCreatedAtDesc(boolean active);
    // WHERE active = ? ORDER BY created_at DESC

    List<User> findTop5ByActiveOrderByCreatedAtDesc(boolean active);
    // WHERE active = ? ORDER BY created_at DESC LIMIT 5

    // ==========  Delete  ==========

    void deleteByEmail(String email);
    // DELETE FROM users WHERE email = ?

    long deleteByActiveFalse();
    // DELETE FROM users WHERE active = false — trả về số dòng đã xóa
}
```

---

## 4. @Query — JPQL và Native SQL

Khi Derived Query Method không đủ mạnh, dùng `@Query`.

### JPQL (Java Persistence Query Language)

JPQL dùng **tên Entity và field Java** (không phải tên bảng và cột DB).

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // JPQL đơn giản — tìm theo trạng thái
    @Query("SELECT o FROM Order o WHERE o.status = :status")
    List<Order> findByStatus(@Param("status") OrderStatus status);

    // JPQL với JOIN — lấy Order cùng với User info
    @Query("""
            SELECT o FROM Order o
            JOIN FETCH o.user u
            WHERE u.email = :email
            AND o.createdAt >= :fromDate
            ORDER BY o.createdAt DESC
            """)
    List<Order> findByUserEmailAndDateRange(
        @Param("email") String email,
        @Param("fromDate") LocalDateTime fromDate
    );

    // Projection — chỉ lấy một số field (trả về interface projection)
    @Query("SELECT o.id AS id, o.total AS total, o.status AS status FROM Order o WHERE o.user.id = :userId")
    List<OrderSummary> findSummariesByUserId(@Param("userId") Long userId);

    // COUNT query với điều kiện phức tạp
    @Query("SELECT COUNT(o) FROM Order o WHERE o.status = :status AND o.user.active = true")
    long countActiveUserOrders(@Param("status") OrderStatus status);

    // UPDATE query — phải có @Modifying
    @Modifying
    @Query("UPDATE Order o SET o.status = :newStatus WHERE o.status = :oldStatus AND o.updatedAt < :cutoff")
    int bulkUpdateOrderStatus(
        @Param("newStatus") OrderStatus newStatus,
        @Param("oldStatus") OrderStatus oldStatus,
        @Param("cutoff") LocalDateTime cutoff
    );

    // Kết hợp @Query với Pageable — phải khai báo countQuery riêng khi có JOIN
    @Query(value = """
                   SELECT o FROM Order o
                   JOIN FETCH o.user u
                   WHERE o.status = :status
                   """,
           countQuery = "SELECT COUNT(o) FROM Order o WHERE o.status = :status")
    Page<Order> findByStatusWithPagination(
        @Param("status") OrderStatus status,
        Pageable pageable
    );
}
```

### Projection Interface — Lấy Một Phần Dữ Liệu

```java
// Interface Projection — Spring Data tự tạo proxy implement interface này
public interface OrderSummary {
    Long getId();
    BigDecimal getTotal();
    OrderStatus getStatus();

    // Computed field — dùng @Value với SpEL (Spring Expression Language)
    @Value("#{target.total.multiply(new java.math.BigDecimal('1.1'))}")
    BigDecimal getTotalWithTax();
}

// DTO Projection — type-safe hơn, performance tốt hơn
public record OrderSummaryDto(Long id, BigDecimal total, OrderStatus status) {}

// Dùng DTO constructor trong JPQL
@Query("SELECT new com.example.dto.OrderSummaryDto(o.id, o.total, o.status) FROM Order o WHERE o.user.id = :userId")
List<OrderSummaryDto> findDtosByUserId(@Param("userId") Long userId);
```

### Native SQL — Khi JPQL Không Đủ

```java
// Native query — dùng khi cần DB-specific features (window functions, CTE, etc.)
@Query(value = """
       SELECT p.*, c.name AS category_name
       FROM products p
       INNER JOIN categories c ON p.category_id = c.id
       WHERE p.price BETWEEN :minPrice AND :maxPrice
       ORDER BY p.price ASC
       LIMIT :limit OFFSET :offset
       """,
       nativeQuery = true)  // ← Báo Spring đây là SQL thuần, không phải JPQL
List<Map<String, Object>> findProductsInPriceRange(
    @Param("minPrice") BigDecimal minPrice,
    @Param("maxPrice") BigDecimal maxPrice,
    @Param("limit") int limit,
    @Param("offset") int offset
);
```

---

## 5. Pagination & Sorting — Phân Trang và Sắp Xếp

### Pageable — Interface Phân Trang

```java
// Controller — nhận Pageable từ request parameters
@GetMapping("/products")
public ResponseEntity<Page<ProductResponse>> getProducts(
    @RequestParam(defaultValue = "0") int page,    // Số trang (0-based)
    @RequestParam(defaultValue = "20") int size,   // Số phần tử mỗi trang
    @RequestParam(defaultValue = "createdAt") String sortBy,
    @RequestParam(defaultValue = "DESC") String sortDir
) {
    Sort sort = Sort.by(Sort.Direction.fromString(sortDir), sortBy);
    Pageable pageable = PageRequest.of(page, size, sort);

    Page<Product> products = productRepository.findAll(pageable);

    return ResponseEntity.ok(products.map(productMapper::toResponse));
}

// Hoặc dùng @PageableDefault — Spring tự inject Pageable
@GetMapping("/products")
public Page<ProductResponse> getProducts(
    @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC)
    Pageable pageable
) {
    return productRepository.findAll(pageable)
                            .map(productMapper::toResponse);
}
```

### Page vs Slice — Khi Nào Dùng Gì

```java
// Page<T> — đếm tổng số bản ghi (SELECT COUNT(*)) — phù hợp cho UI phân trang
Page<Product> page = productRepository.findAll(pageable);
page.getTotalElements();    // Tổng số bản ghi
page.getTotalPages();       // Tổng số trang
page.getNumber();           // Trang hiện tại (0-based)
page.getSize();             // Kích thước trang
page.getContent();          // List<Product> của trang hiện tại
page.hasNext();             // Có trang tiếp theo?

// Slice<T> — KHÔNG đếm tổng (không có COUNT query) — phù hợp cho "Load More" / infinite scroll
Slice<Product> slice = productRepository.findByActive(true, pageable);
slice.hasNext();            // Có phần tử tiếp theo?
slice.getContent();         // List<Product>
// slice.getTotalElements() ← KHÔNG CÓ method này!
```

### Sort — Sắp Xếp

```java
// Sắp xếp đơn giản
Sort sort = Sort.by("price").ascending();
Sort sort = Sort.by(Sort.Direction.DESC, "createdAt");

// Sắp xếp nhiều trường
Sort multiSort = Sort.by("category").ascending()
                     .and(Sort.by("price").descending())
                     .and(Sort.by("name").ascending());

// Null handling
Sort sort = Sort.by(Sort.Order.asc("price").nullsLast());
// NULLS LAST — giá null xuất hiện cuối cùng
```

---

## 6. Criteria API — Truy Vấn Động Type-Safe

Criteria API dùng khi điều kiện truy vấn thay đổi theo runtime (ví dụ: bộ lọc tìm kiếm nhiều trường tuỳ chọn).

```java
// Specification Pattern (Mẫu Specification) — tách logic query thành các class riêng
public class ProductSpecifications {

    // Mỗi method trả về một Specification<Product>
    public static Specification<Product> hasCategory(Long categoryId) {
        return (root, query, criteriaBuilder) ->
            categoryId == null ? null :
            criteriaBuilder.equal(root.get("category").get("id"), categoryId);
    }

    public static Specification<Product> priceBetween(BigDecimal min, BigDecimal max) {
        return (root, query, criteriaBuilder) -> {
            if (min == null && max == null) return null;
            if (min == null) return criteriaBuilder.lessThanOrEqualTo(root.get("price"), max);
            if (max == null) return criteriaBuilder.greaterThanOrEqualTo(root.get("price"), min);
            return criteriaBuilder.between(root.get("price"), min, max);
        };
    }

    public static Specification<Product> nameContains(String keyword) {
        return (root, query, criteriaBuilder) ->
            keyword == null || keyword.isBlank() ? null :
            criteriaBuilder.like(
                criteriaBuilder.lower(root.get("name")),
                "%" + keyword.toLowerCase() + "%"
            );
    }

    public static Specification<Product> isActive() {
        return (root, query, criteriaBuilder) ->
            criteriaBuilder.isTrue(root.get("active"));
    }
}

// Repository phải extend JpaSpecificationExecutor
public interface ProductRepository extends JpaRepository<Product, Long>,
                                           JpaSpecificationExecutor<Product> {
}

// Service — kết hợp các Specification linh hoạt
@Service
public class ProductSearchService {

    public Page<Product> search(ProductSearchRequest request, Pageable pageable) {
        Specification<Product> spec = Specification
            .where(ProductSpecifications.isActive())
            .and(ProductSpecifications.hasCategory(request.getCategoryId()))
            .and(ProductSpecifications.priceBetween(request.getMinPrice(), request.getMaxPrice()))
            .and(ProductSpecifications.nameContains(request.getKeyword()));

        return productRepository.findAll(spec, pageable);
    }
}
```

---

## 7. @Modifying — Update/Delete Queries

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // Bulk update — cần @Modifying và @Transactional
    @Modifying
    @Transactional
    @Query("UPDATE User u SET u.active = false WHERE u.lastLoginAt < :cutoff")
    int deactivateInactiveUsers(@Param("cutoff") LocalDateTime cutoff);

    // Bulk delete
    @Modifying
    @Transactional
    @Query("DELETE FROM User u WHERE u.deletedAt IS NOT NULL AND u.deletedAt < :cutoff")
    int purgeDeletedUsers(@Param("cutoff") LocalDateTime cutoff);

    // clearAutomatically = true — xóa first-level cache sau khi chạy
    // (tránh đọc data cũ từ cache sau bulk update)
    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Transactional
    @Query("UPDATE Product p SET p.price = p.price * :multiplier WHERE p.category.id = :categoryId")
    int adjustPriceByCategory(
        @Param("multiplier") BigDecimal multiplier,
        @Param("categoryId") Long categoryId
    );
}
```

---

## 8. Custom Repository — Repository Tùy Chỉnh

Khi cần logic phức tạp vượt quá khả năng của Derived Query và `@Query`:

```java
// Interface định nghĩa custom methods
public interface ProductRepositoryCustom {
    List<Product> searchWithComplexFilters(ProductFilter filter);
}

// Implementation — dùng EntityManager trực tiếp
@Repository
@RequiredArgsConstructor
public class ProductRepositoryCustomImpl implements ProductRepositoryCustom {

    private final EntityManager em;  // EntityManager (Quản Lý Thực Thể)

    @Override
    public List<Product> searchWithComplexFilters(ProductFilter filter) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Product> cq = cb.createQuery(Product.class);
        Root<Product> root = cq.from(Product.class);

        List<Predicate> predicates = new ArrayList<>();

        if (filter.getKeyword() != null) {
            predicates.add(cb.like(
                cb.lower(root.get("name")),
                "%" + filter.getKeyword().toLowerCase() + "%"
            ));
        }

        if (filter.getCategoryIds() != null && !filter.getCategoryIds().isEmpty()) {
            predicates.add(root.get("category").get("id").in(filter.getCategoryIds()));
        }

        cq.where(predicates.toArray(new Predicate[0]))
          .orderBy(cb.desc(root.get("createdAt")));

        return em.createQuery(cq)
                 .setMaxResults(filter.getLimit())
                 .setFirstResult(filter.getOffset())
                 .getResultList();
    }
}

// Repository kết hợp cả hai
public interface ProductRepository extends JpaRepository<Product, Long>,
                                           JpaSpecificationExecutor<Product>,
                                           ProductRepositoryCustom {  // ← Kế thừa custom interface
    // Derived queries và @Query annotations ở đây...
}
```

---

## 9. Entity Auditing — Tự Động Ghi Lại Thông Tin Kiểm Toán

```java
// Enable auditing trong main class hoặc config class
@SpringBootApplication
@EnableJpaAuditing
public class Application {}

// Base entity có thể tái sử dụng — @MappedSuperclass
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseAuditEntity {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(updatable = false, length = 100)
    private String createdBy;

    @LastModifiedBy
    @Column(length = 100)
    private String updatedBy;
}

// Cung cấp thông tin user hiện tại
@Component
public class SpringSecurityAuditorAware implements AuditorAware<String> {

    @Override
    public Optional<String> getCurrentAuditor() {
        return Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
            .filter(auth -> auth.isAuthenticated() && !(auth instanceof AnonymousAuthenticationToken))
            .map(Authentication::getName);
    }
}

// Entity kế thừa BaseAuditEntity
@Entity
@Table(name = "products")
public class Product extends BaseAuditEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    // createdAt, updatedAt, createdBy, updatedBy được kế thừa từ BaseAuditEntity
}
```

---

## 10. Cấu Hình JPA — application.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:secret}
    driver-class-name: org.postgresql.Driver

  jpa:
    hibernate:
      ddl-auto: validate       # validate — production (chỉ kiểm tra schema khớp)
                               # update — development (tự thêm cột mới, không xóa)
                               # create-drop — testing (tạo rồi xóa sau test)
                               # none — nếu dùng Flyway/Liquibase quản lý schema
    show-sql: false            # true chỉ khi debug — không dùng production
    properties:
      hibernate:
        format_sql: true       # Format SQL đẹp hơn khi show-sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect
        jdbc:
          batch_size: 50       # Batch insert — gom 50 câu INSERT vào một lần gọi DB
          order_inserts: true  # Sắp xếp INSERT để batch hiệu quả hơn
          order_updates: true
        default_schema: public # Schema mặc định
        generate_statistics: false  # true khi cần phát hiện N+1

logging:
  level:
    org.hibernate.SQL: DEBUG           # Log câu SQL
    org.hibernate.orm.jdbc.bind: TRACE # Log bind parameters — rất verbose!
```

---

## 11. Flyway — Database Migration (Di Chuyển Schema Cơ Sở Dữ Liệu)

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

```sql
-- src/main/resources/db/migration/V1__init_schema.sql
-- Naming convention: V{version}__{description}.sql (2 dấu gạch dưới)
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    first_name  VARCHAR(100) NOT NULL,
    last_name   VARCHAR(100) NOT NULL,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

-- src/main/resources/db/migration/V2__add_products_table.sql
CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    sku         VARCHAR(50) NOT NULL UNIQUE,
    price       DECIMAL(10, 2),
    created_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_product_sku ON products(sku);
```

---

## ✅ Checklist Thực Hành

- [ ] Tạo Entity với `@Entity`, `@Table`, `@Id`, `@Column` đúng constraints
- [ ] Viết Repository với ít nhất 5 Derived Query Methods
- [ ] Implement tìm kiếm với Specification Pattern
- [ ] Thêm Pagination vào API endpoint
- [ ] Viết `@Query` JPQL với JOIN và Projection
- [ ] Thiết lập Flyway với 2 migration scripts
- [ ] Cấu hình Entity Auditing với `@CreatedDate`, `@LastModifiedDate`

---

## 💡 Anti-Patterns (Mẫu Chống Tiêu Chuẩn) Phổ Biến

```java
// ❌ TRÁNH: Lấy tất cả rồi filter ở Java
List<User> allUsers = userRepository.findAll();
List<User> activeUsers = allUsers.stream()
                                  .filter(User::isActive)
                                  .collect(Collectors.toList());
// ✅ ĐÚNG: Filter ở DB
List<User> activeUsers = userRepository.findByActiveTrue();

// ❌ TRÁNH: Dùng findAll() không có phân trang
List<Order> orders = orderRepository.findAll(); // Có thể trả về triệu dòng!
// ✅ ĐÚNG: Luôn phân trang
Page<Order> orders = orderRepository.findAll(PageRequest.of(0, 100));

// ❌ TRÁNH: Dùng getOne() đã deprecated
Product product = productRepository.getOne(id); // Deprecated!
// ✅ ĐÚNG: Dùng getReferenceById() hoặc findById()
Product product = productRepository.getReferenceById(id); // Lazy proxy
Product product = productRepository.findById(id).orElseThrow(); // Load ngay
```

---

## 🔗 Liên Kết

- [2-orm-mapping.md](2-orm-mapping.md) — Relationships, Cascade, FetchType
- [3-transactions.md](3-transactions.md) — @Transactional, Propagation
- [4-n-plus-one-problem.md](4-n-plus-one-problem.md) — N+1 và JOIN FETCH
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Vlad Mihalcea Blog](https://vladmihalcea.com/) — JPA/Hibernate chuyên sâu

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
