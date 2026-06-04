# Query Optimization — Tối Ưu Truy Vấn Database

> Slow queries (Truy Vấn Chậm) là nguyên nhân hàng đầu gây performance degradation (Suy
> Giảm Hiệu Năng) trong ứng dụng. Một query không có index có thể chậm hơn 100–1000 lần
> so với query có index. File này đề cập chiến lược toàn diện để tối ưu truy vấn trong
> PostgreSQL kết hợp với Spring Boot / JPA.

---

## 📋 Mục Tiêu

- [ ] Đọc và hiểu **EXPLAIN ANALYZE** (Kế Hoạch Thực Thi Truy Vấn)
- [ ] Thiết kế và chọn đúng loại **index** (Chỉ Mục) cho mỗi trường hợp
- [ ] Tối ưu **batch operations** (Thao Tác Hàng Loạt) với JPA/JDBC
- [ ] Dùng **pagination** (Phân Trang) đúng cách để tránh full table scan
- [ ] Phát hiện và tắt **slow query log** (Nhật Ký Truy Vấn Chậm)
- [ ] Tránh các **anti-patterns** (Mẫu Chống Chỉ Định) phổ biến

---

## 1. EXPLAIN ANALYZE — Đọc Execution Plan

### Cú Pháp Cơ Bản

```sql
-- Phân tích query mà KHÔNG chạy thực sự
EXPLAIN SELECT * FROM products WHERE category = 'Electronics';

-- Phân tích VÀ chạy thực sự (độ chính xác cao hơn)
EXPLAIN ANALYZE SELECT * FROM products WHERE category = 'Electronics';

-- Với thông tin chi tiết hơn (Buffers, Format JSON)
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.status = 'PAID' AND o.created_at > NOW() - INTERVAL '7 days';
```

### Đọc EXPLAIN Output

```
Kết quả:
Seq Scan on products  (cost=0.00..1543.00 rows=1000 width=200) (actual time=0.012..8.543 rows=1234 loops=1)
  Filter: (category = 'Electronics')
  Rows Removed by Filter: 49766

Seq Scan on products  ← FULL TABLE SCAN — đây là vấn đề! Không có index
cost=0.00..1543.00    ← Ước tính cost (chi phí ước tính)
rows=1000             ← Ước tính số rows trả về
actual time=0.012..8.543  ← Thời gian thực tế: start..end (ms)
rows=1234             ← Số rows thực tế trả về
Rows Removed: 49766   ← Phải đọc 49766 rows để tìm 1234 rows → CẦN INDEX!
```

### Các Loại Scan Node (Nút Quét)

```
Seq Scan (Sequential Scan — Quét Tuần Tự):
  → Đọc toàn bộ bảng từ đầu đến cuối
  → Dùng khi table nhỏ hoặc query trả về nhiều rows (> 20-30% bảng)
  ⚠ Nếu bảng lớn + filter chặt → cần index!

Index Scan (Quét Chỉ Mục):
  → Dùng index để tìm rows, sau đó fetch data từ heap (bảng chính)
  → Hiệu quả cho selective queries (< 5–10% rows)

Index Only Scan (Quét Chỉ Mục Thuần):
  → Tất cả dữ liệu cần thiết có trong index — không cần đọc heap
  → Nhanh nhất! Covering index (Chỉ Mục Bao Phủ)

Bitmap Index Scan (Quét Chỉ Mục Bitmap):
  → Dùng nhiều indexes kết hợp với nhau
  → Tốt cho queries với nhiều conditions
```

---

## 2. Index Design (Thiết Kế Chỉ Mục)

### 2.1 B-Tree Index — Mặc Định, Đa Năng Nhất

```sql
-- Index cơ bản trên single column
CREATE INDEX idx_products_category ON products(category);

-- Composite index (Chỉ Mục Tổng Hợp) — thứ tự cột RẤT quan trọng
-- Rule: Cột có tính phân biệt cao nhất (highest cardinality) đặt đầu tiên
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
-- Query này DÙNG được index: WHERE user_id = 1 AND status = 'PAID'
-- Query này DÙNG được index: WHERE user_id = 1
-- Query này KHÔNG dùng được: WHERE status = 'PAID' (không có user_id ở trước)

-- Index trên expression (Chỉ Mục Biểu Thức)
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
-- Dùng cho: WHERE LOWER(email) = 'user@example.com'

-- Partial index (Chỉ Mục Một Phần) — chỉ index một subset rows
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'PENDING';
-- Nhỏ hơn full index, chỉ dùng cho: WHERE status = 'PENDING' AND ...

-- Unique index (Chỉ Mục Duy Nhất)
CREATE UNIQUE INDEX idx_users_email ON users(email);
```

### 2.2 Covering Index (Chỉ Mục Bao Phủ)

```sql
-- Bao gồm tất cả columns mà query cần — tránh fetch từ heap
-- Dùng INCLUDE để thêm columns vào index (PostgreSQL 11+)
CREATE INDEX idx_orders_covering ON orders(user_id, status)
INCLUDE (total_amount, created_at);

-- Query này chỉ cần đọc index, không cần đọc bảng chính!
SELECT total_amount, created_at
FROM orders
WHERE user_id = 1 AND status = 'PAID';
-- → Index Only Scan thay vì Index Scan → NHANH HƠN ~3-5x
```

### 2.3 Khi Nào KHÔNG Nên Tạo Index

```
❌ Tránh index khi:
  - Cột có ít giá trị duy nhất (low cardinality) như: is_active (true/false)
  - Bảng rất nhỏ (< 1000 rows) — Seq Scan thường nhanh hơn
  - Cột được update thường xuyên — index overhead cho write operations
  - Quá nhiều indexes trên 1 bảng — mỗi INSERT/UPDATE/DELETE phải cập nhật tất cả indexes

Mỗi index:
  + Tăng tốc SELECT
  - Chậm hơn INSERT/UPDATE/DELETE
  - Chiếm thêm disk space (Không Gian Lưu Trữ)
```

---

## 3. Slow Query Log (Nhật Ký Truy Vấn Chậm)

### Bật Slow Query Log Trong Spring Boot

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        # Log SQL queries
        show_sql: false          # Tắt trong production — quá nhiều noise
        format_sql: true         # Format SQL cho dễ đọc
        # Log queries chậm hơn threshold (Ngưỡng)
        session.events.log.LOG_QUERIES_SLOWER_THAN_MS: 100  # Log query > 100ms

logging:
  level:
    org.hibernate.SQL: DEBUG         # Log SQL statements
    org.hibernate.type: TRACE        # Log parameter values (CẨN THẬN: sensitive data)
    org.hibernate.stat: DEBUG        # Log Hibernate statistics
```

### Hibernate Statistics

```java
@Configuration
public class HibernateConfig {

    @Bean
    public HibernateJpaVendorAdapter hibernateJpaVendorAdapter() {
        HibernateJpaVendorAdapter adapter = new HibernateJpaVendorAdapter();
        return adapter;
    }
}

// Bật statistics
// application.yml:
// spring.jpa.properties.hibernate.generate_statistics: true
// logging.level.org.hibernate.stat: DEBUG

// Output trong logs:
// HHH000117: HQL: select o from Order o, time: 1234ms, rows: 5678
// Session Metrics:
//   12345 nanoseconds spent acquiring 1 JDBC connections
//   56789 nanoseconds spent executing 15 JDBC statements
```

### P6Spy — Log SQL Với Parameters

```xml
<!-- pom.xml -->
<dependency>
    <groupId>p6spy</groupId>
    <artifactId>p6spy</artifactId>
    <version>3.9.1</version>
    <scope>runtime</scope>
</dependency>
```

```yaml
# application.yml
spring:
  datasource:
    driver-class-name: com.p6spy.engine.spy.P6SpyDriver
    url: jdbc:p6spy:postgresql://localhost:5432/mydb
```

```properties
# spy.properties
logMessageFormat=com.p6spy.engine.spy.appender.MultiLineFormat
appender=com.p6spy.engine.spy.appender.Slf4JLogger
# Log chỉ queries chậm hơn 100ms
outagedetection=true
outagedetectioninterval=100
```

---

## 4. Batch Operations (Thao Tác Hàng Loạt)

### Vấn Đề Với Single-Row Operations

```java
// ❌ VẤN ĐỀ: Insert 10,000 rows = 10,000 queries!
List<Product> products = generateProducts(10_000);
for (Product p : products) {
    repository.save(p);  // Mỗi save = 1 INSERT query
}
// Logs: 10,000 INSERT statements
// Thời gian: ~30–60 giây!
```

### Batch Insert Với JPA

```yaml
# application.yml — bật batch cho JPA
spring:
  jpa:
    properties:
      hibernate:
        jdbc.batch_size: 50           # Batch 50 inserts cùng 1 lần
        order_inserts: true           # Sắp xếp inserts theo loại entity
        order_updates: true           # Sắp xếp updates theo loại entity
        batch_versioned_data: true    # Cho phép batch với versioned entities
```

```java
// ✅ ĐÚNG: Batch insert hiệu quả với JPA
@Service
@Transactional
public class ProductBatchService {

    @PersistenceContext
    private EntityManager em;

    public void batchInsert(List<Product> products) {
        int batchSize = 50;

        for (int i = 0; i < products.size(); i++) {
            em.persist(products.get(i));

            // Flush (Đẩy Xuống DB) và clear cache theo từng batch
            if (i % batchSize == 0 && i > 0) {
                em.flush();   // Gửi batch SQL xuống DB
                em.clear();   // Giải phóng first-level cache để tránh OOM
            }
        }

        em.flush();  // Flush batch cuối
    }
}
```

### Batch Insert Với Spring Data JPA `saveAll`

```java
// Spring Data JPA's saveAll đã hỗ trợ batch khi hibernate.jdbc.batch_size được cấu hình
@Transactional
public void batchSave(List<Product> products) {
    // saveAll() sẽ batch nếu batch_size được set
    // QUAN TRỌNG: Không dùng @GeneratedValue(strategy = IDENTITY) khi batch!
    // IDENTITY strategy disable batch — dùng SEQUENCE thay thế
    repository.saveAll(products);
}
```

### Chú Ý Về `@GeneratedValue`

```java
// ❌ IDENTITY strategy — KHÔNG hỗ trợ batch!
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // ← Tắt batch!
    private Long id;
}

// ✅ SEQUENCE strategy — Hỗ trợ batch
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
    @SequenceGenerator(name = "product_seq", sequenceName = "product_sequence",
                       allocationSize = 50)  // Pre-allocate 50 IDs
    private Long id;
}
```

### Bulk Update/Delete Với JPQL

```java
// Thay vì load entities rồi update từng cái (N queries):
// List<Order> orders = repository.findByStatus("PENDING");
// orders.forEach(o -> o.setStatus("PROCESSING")); // N updates

// ĐÚNG: 1 câu JPQL UPDATE cho tất cả
@Modifying
@Transactional
@Query("UPDATE Order o SET o.status = 'PROCESSING' WHERE o.status = 'PENDING' AND o.createdAt < :cutoff")
int bulkUpdateStatus(@Param("cutoff") LocalDateTime cutoff);

// ĐÚNG: Bulk DELETE
@Modifying
@Transactional
@Query("DELETE FROM Order o WHERE o.status = 'CANCELLED' AND o.createdAt < :cutoff")
int bulkDeleteOld(@Param("cutoff") LocalDateTime cutoff);
```

---

## 5. Pagination Best Practices (Thực Tiễn Phân Trang)

### Offset Pagination — Phổ Biến Nhưng Có Giới Hạn

```java
// Offset Pagination (Phân Trang Dựa Trên Vị Trí)
@GetMapping("/products")
public Page<ProductResponse> list(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "id") String sortBy) {

    Pageable pageable = PageRequest.of(page, size, Sort.by(sortBy));
    return repository.findAll(pageable)
                     .map(mapper::toResponse);
}
```

```sql
-- SQL tương ứng:
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 0;   -- Trang 1: OK
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 980; -- Trang 50: CHẬM!

-- Vấn đề OFFSET lớn:
-- PostgreSQL phải đọc và bỏ 980 rows trước khi trả về 20 rows
-- OFFSET = 10,000 → Phải đọc 10,020 rows để trả về 20 rows → Rất chậm!
```

### Keyset Pagination (Phân Trang Dựa Trên Khóa) — Hiệu Năng Tốt Hơn

```java
// Cursor-based / Keyset Pagination
// Thay vì OFFSET, dùng WHERE id > lastSeenId
public List<Product> getPage(Long lastId, int size) {
    if (lastId == null) {
        return repository.findTop20ByOrderByIdAsc();
    }
    return repository.findByIdGreaterThanOrderByIdAsc(lastId, PageRequest.of(0, size));
}
```

```sql
-- SQL tương ứng — luôn nhanh bất kể trang nào
SELECT * FROM products WHERE id > 980 ORDER BY id LIMIT 20;

-- Dùng index trên id → không cần đọc rows trước
-- Thời gian: constant O(log n) thay vì O(n)
```

### Tránh `count(*)` Không Cần Thiết

```java
// Page<T> luôn chạy thêm 1 COUNT(*) query
Page<Product> page = repository.findAll(pageable);
// → 2 queries: SELECT + COUNT(*) — COUNT có thể chậm trên bảng lớn!

// Dùng Slice<T> khi không cần tổng số trang
Slice<Product> slice = repository.findAll(pageable);
// → 1 query: chỉ SELECT
// Trade-off: Không biết tổng số trang/kết quả

// Hoặc count riêng với cache
@Cacheable("productCount")
public long countProducts() {
    return repository.count();
}
```

---

## 6. Query Optimization Patterns

### Projection — Chỉ Lấy Dữ Liệu Cần Thiết

```java
// ❌ SAI: Lấy toàn bộ entity nhưng chỉ dùng 2 fields
List<Product> products = repository.findAll();
return products.stream()
    .map(p -> new ProductSummary(p.getId(), p.getName()))
    .toList();
// → SELECT tất cả columns, transfer nhiều data không cần

// ✅ ĐÚNG: Dùng DTO projection (Chiếu DTO)
public interface ProductSummaryProjection {
    Long getId();
    String getName();
}

List<ProductSummaryProjection> projections = repository.findAllProjectedBy();
// → SELECT id, name FROM products (chỉ 2 columns)

// Hoặc JPQL với DTO constructor
@Query("SELECT new com.example.dto.ProductSummary(p.id, p.name) FROM Product p")
List<ProductSummary> findAllSummaries();
```

### Read-Only Queries

```java
// @Transactional(readOnly = true) cho SELECT queries:
// 1. Hibernate không tạo dirty checking (Kiểm Tra Thay Đổi) snapshot
// 2. Có thể route sang read replica (Bản Sao Chỉ Đọc)
// 3. Giảm overhead ~15-20%
@Transactional(readOnly = true)
public List<ProductResponse> findAll() {
    return repository.findAll().stream()
        .map(mapper::toResponse)
        .toList();
}
```

### Query Hints (Gợi Ý Truy Vấn)

```java
// Hint cho Hibernate
@QueryHints({
    @QueryHint(name = "org.hibernate.readOnly", value = "true"),
    @QueryHint(name = "org.hibernate.fetchSize", value = "50"),
    @QueryHint(name = "jakarta.persistence.query.timeout", value = "5000")
})
@Query("SELECT p FROM Product p WHERE p.category = :category")
List<Product> findByCategory(@Param("category") String category);
```

---

## 7. PostgreSQL-Specific Optimizations

### VACUUM và ANALYZE

```sql
-- VACUUM: Thu hồi không gian từ deleted rows (Hàng Đã Xóa)
VACUUM products;

-- VACUUM ANALYZE: Thu hồi không gian VÀ cập nhật statistics (Thống Kê)
VACUUM ANALYZE products;

-- Autovacuum (Tự Động Vacuum) — PostgreSQL tự động chạy
-- Kiểm tra autovacuum status:
SELECT schemaname, tablename, last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
WHERE tablename = 'products';
```

### Partial Index Cho Hot Paths (Đường Dẫn Nóng)

```sql
-- Chỉ index orders đang active (không index historical data)
CREATE INDEX idx_orders_active ON orders(user_id, created_at)
WHERE status IN ('PENDING', 'PROCESSING');

-- Index nhỏ hơn nhiều → query nhanh hơn cho active orders
-- Historical orders không vào index → không tốn overhead
```

### Index Trên JSON/JSONB Fields

```sql
-- GIN index (Generalized Inverted Index) cho JSONB
CREATE INDEX idx_products_metadata ON products USING GIN(metadata);

-- Query tận dụng GIN index:
SELECT * FROM products WHERE metadata @> '{"color": "red"}';
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: Khi nào dùng composite index (Chỉ Mục Tổng Hợp) thay vì multiple single indexes?**

```
Composite index khi:
  - Query thường dùng cả 2 columns trong WHERE
  - Cần covering index (tất cả columns trong SELECT+WHERE đều trong index)
  - ORDER BY trên nhiều columns

Single indexes khi:
  - Mỗi column được query độc lập thường xuyên
  - Columns có tính phân biệt cao (high cardinality)

Thứ tự trong composite index:
  - Cột đầu tiên: Hay xuất hiện trong equality conditions (WHERE col = ?)
  - Cột tiếp theo: Range conditions (WHERE col > ?)
  - Covering columns (INCLUDE): Chỉ để tránh heap fetch
```

**Q: Tại sao `SELECT *` là bad practice trong production?**

```
1. Network transfer: Truyền data không cần thiết qua mạng
2. Memory: Hibernate tạo object lớn hơn mức cần
3. Index: Không thể dùng covering index → phải fetch từ heap
4. Evolution: Khi thêm column, SELECT * thay đổi behavior

→ Luôn specify columns cần thiết hoặc dùng DTO projection
```

---

## ✅ Checklist

- [ ] Tất cả foreign key columns đã có index
- [ ] Slow query log được bật (threshold 100–200ms)
- [ ] Không có `SELECT *` trong production queries
- [ ] Batch operations dùng `jdbc.batch_size` và SEQUENCE generator
- [ ] Pagination dùng `Slice<T>` thay vì `Page<T>` khi không cần count
- [ ] `@Transactional(readOnly = true)` cho tất cả SELECT methods
- [ ] EXPLAIN ANALYZE đã được chạy cho critical queries
- [ ] Không có Seq Scan trên bảng lớn với selective WHERE conditions

---

**Xem tiếp:** [5-profiling.md](5-profiling.md) — Profiling với async-profiler & JFR
