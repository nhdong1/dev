# Connection Pooling — HikariCP

> HikariCP là connection pool (bể kết nối) mặc định của Spring Boot — nhanh nhất,
> nhẹ nhất, và đáng tin cậy nhất. Cấu hình sai HikariCP là nguyên nhân phổ biến
> gây ra production outage (sự cố production) không cần thiết.

---

## 📋 Mục Tiêu

- [ ] Hiểu **Connection Pool** hoạt động như thế nào
- [ ] Tính toán **`maximumPoolSize`** phù hợp cho workload
- [ ] Cấu hình đầy đủ các thuộc tính quan trọng của **HikariCP**
- [ ] Phát hiện **connection leak** (Rò Rỉ Kết Nối) với `leakDetectionThreshold`
- [ ] Monitor (Giám Sát) pool health với Micrometer metrics
- [ ] Xử lý **pool exhaustion** (Cạn Kiệt Pool) và timeout đúng cách

---

## 1. Connection Pool Là Gì?

### Vấn Đề Khi Không Có Pool

```
Mỗi HTTP request tạo kết nối DB mới:

Request → [Tạo TCP connection] → [TCP handshake] → [Auth DB] → [Execute query] → [Đóng connection]
           ~50–200ms overhead mỗi lần!

Với 100 concurrent requests → 100 kết nối DB mở/đóng liên tục
→ DB server quá tải
→ Response time tệ
```

### Connection Pool Giải Quyết Vấn Đề

```
Khởi động app → Pool tạo sẵn N kết nối DB

Request đến → [Lấy kết nối từ pool] → [Execute query] → [Trả kết nối về pool]
               ~microseconds!

Kết nối được tái sử dụng — không tạo mới mỗi lần
```

```
HikariCP Pool:

  ┌─────────────────────────────────────────┐
  │              HikariCP Pool              │
  │                                         │
  │  [Conn1] [Conn2] [Conn3] ... [ConnN]    │
  │    idle    busy    idle        idle      │
  │                                         │
  │  minimumIdle=5   maximumPoolSize=10     │
  └─────────────────────────────────────────┘
         ↑                    ↑
    Idle connections     Connections đang dùng
    (Kết nối nhàn rỗi)   (Connections in use)
```

---

## 2. Cấu Hình HikariCP

### `application.yml` — Cấu Hình Đầy Đủ

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver

    hikari:
      # ── Kích Thước Pool ──────────────────────────────────────
      maximum-pool-size: 10        # Số kết nối tối đa trong pool
      minimum-idle: 5              # Số kết nối idle tối thiểu duy trì

      # ── Timeout Settings (Cấu Hình Thời Gian Chờ) ───────────
      connection-timeout: 30000    # Thời gian chờ lấy connection (ms) — mặc định 30s
      idle-timeout: 600000         # Kết nối idle bị đóng sau (ms) — mặc định 10 phút
      max-lifetime: 1800000        # Thời gian sống tối đa của connection (ms) — 30 phút
      keepalive-time: 300000       # Ping DB để giữ connection sống (ms) — 5 phút

      # ── Leak Detection (Phát Hiện Rò Rỉ) ────────────────────
      leak-detection-threshold: 2000  # Log warning nếu connection giữ > 2 giây (ms)

      # ── Pool Metadata ────────────────────────────────────────
      pool-name: MainPool          # Tên pool — xuất hiện trong logs và metrics
      connection-test-query: SELECT 1  # Query kiểm tra connection còn sống (optional với JDBC4+)

      # ── Advanced Settings ────────────────────────────────────
      auto-commit: true            # Auto-commit (Tự Động Commit) — mặc định true
      transaction-isolation: TRANSACTION_READ_COMMITTED  # Isolation level mặc định
```

### Cấu Hình Theo Code (Java Config)

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();

        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setUsername(username);
        config.setPassword(password);

        // Pool size
        config.setMaximumPoolSize(10);
        config.setMinimumIdle(5);

        // Timeouts
        config.setConnectionTimeout(30_000);   // 30 giây
        config.setIdleTimeout(600_000);        // 10 phút
        config.setMaxLifetime(1_800_000);      // 30 phút

        // Leak detection
        config.setLeakDetectionThreshold(2_000);  // 2 giây

        config.setPoolName("MainPool");

        // Performance properties
        config.addDataSourceProperty("cachePrepStmts", "true");          // Cache prepared statements
        config.addDataSourceProperty("prepStmtCacheSize", "250");        // Cache size
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");   // Max SQL length to cache

        return new HikariDataSource(config);
    }
}
```

---

## 3. Tính Toán `maximumPoolSize`

### Công Thức HikariCP

```
maximumPoolSize = connections_needed_per_second × average_query_time_in_seconds

Ví dụ thực tế:
  - App nhận: 500 requests/second
  - Mỗi request cần: 2 DB queries
  - Average query time: 10ms = 0.01s

  connections_needed = 500 × 2 × 0.01 = 10 connections

maximumPoolSize = 10 (+ một ít buffer)
```

### Quy Tắc Kinh Nghiệm (Rule of Thumb)

```
Pool size lý tưởng theo PostgreSQL documentation:
  connections = (core_count × 2) + effective_spindle_count

Ví dụ server 4 core, 1 SSD:
  connections = (4 × 2) + 1 = 9 → dùng 10

Ví dụ server 8 core, SSD RAID:
  connections = (8 × 2) + 1 = 17 → dùng 20
```

### Cạm Bẫy Phổ Biến

```
❌ SAI: maximumPoolSize = 100 (quá nhiều)

Nhiều connections KHÔNG có nghĩa là nhanh hơn!
  - Mỗi connection chiếm ~5–10MB RAM trên PostgreSQL
  - Too many connections → context switching overhead (Chi Phí Chuyển Ngữ Cảnh)
  - PostgreSQL default max_connections = 100
  - Nếu 5 service instances × 100 connections = 500 > 100 → lỗi ngay!

✅ ĐÚNG: Tính theo công thức, thường là 10–20 connections
```

### Multiple Service Instances (Nhiều Instance Dịch Vụ)

```
Nếu chạy 3 instances trong K8s:
  Total DB connections = instances × maximumPoolSize
  3 instances × 10 connections = 30 connections total

PostgreSQL default: max_connections = 100
PgBouncer (Connection Pooler): có thể handle nhiều hơn

Quy tắc: total_connections ≤ DB_max_connections × 0.8 (giữ 20% buffer)
```

---

## 4. Phát Hiện Connection Leak

### Connection Leak Là Gì?

```
Connection leak (Rò Rỉ Kết Nối) xảy ra khi:
  - Code lấy connection từ pool
  - Nhưng KHÔNG trả về (do exception không được handle, hoặc lỗi logic)
  - Connection bị "giữ mãi mãi"
  - Pool cạn kiệt → các requests mới phải chờ → timeout

Ví dụ lỗi:
  Connection conn = dataSource.getConnection();
  // ... exception xảy ra ...
  conn.close();  // ← KHÔNG BAO GIỜ ĐƯỢC GỌI!
```

### Cấu Hình Leak Detection

```yaml
hikari:
  leak-detection-threshold: 2000  # ms — log warning nếu connection held > 2s
```

```
Khi phát hiện leak, HikariCP log:
  WARN  c.z.hikari.pool.ProxyLeaseTask - Connection leak detection triggered for
  com.example.service.ProductService.getAll(), stack trace follows
  java.lang.Exception: Apparent connection leak detected
      at com.example.service.ProductService.getAll(ProductService.java:45)
      ...

→ Tìm ngay ProductService.java line 45 và fix
```

### Nguyên Nhân Leak Phổ Biến

```java
// ❌ ANTI-PATTERN 1: Không đóng ResultSet/Statement
public void badMethod() {
    Connection conn = dataSource.getConnection();
    Statement stmt = conn.createStatement();
    ResultSet rs = stmt.executeQuery("SELECT ...");
    // Exception xảy ra → conn không được đóng!
}

// ✅ ĐÚNG: Dùng try-with-resources (Quản Lý Tài Nguyên Tự Động)
public void goodMethod() {
    try (Connection conn = dataSource.getConnection();
         Statement stmt = conn.createStatement();
         ResultSet rs = stmt.executeQuery("SELECT ...")) {
        // Auto-close ngay cả khi exception
    }
}

// ❌ ANTI-PATTERN 2: Giữ connection trong @Async method dài
@Async
@Transactional
public CompletableFuture<Void> longRunningTask() {
    // Transaction (và connection) bị giữ suốt thời gian task chạy
    // Nếu task chạy 30 giây → connection bị giữ 30 giây!
    Thread.sleep(30_000);
    return CompletableFuture.completedFuture(null);
}

// ✅ ĐÚNG: Fetch data trước, xử lý sau khi release connection
@Async
public CompletableFuture<Void> longRunningTask() {
    List<Item> items = fetchItems();  // @Transactional ngắn, giải phóng connection ngay
    // Xử lý items mà không giữ connection
    processItems(items);
    return CompletableFuture.completedFuture(null);
}
```

---

## 5. Pool Exhaustion (Cạn Kiệt Pool)

### Triệu Chứng

```
Logs xuất hiện:
  ERROR HikariPool - MainPool - Connection is not available, request timed out after 30000ms

Symptoms (Triệu Chứng):
  - Response time đột ngột tăng cao
  - Lỗi timeout 503
  - Số active connections = maximumPoolSize
  - Requests đang pending trong queue
```

### Nguyên Nhân và Giải Pháp

```
Nguyên nhân 1: maximumPoolSize quá nhỏ so với traffic
→ Giải pháp: Tăng pool size (dựa trên công thức)

Nguyên nhân 2: Queries chạy quá chậm → connection bị giữ lâu
→ Giải pháp: Optimize slow queries, thêm indexes

Nguyên nhân 3: Connection leak
→ Giải pháp: Bật leak detection, fix code

Nguyên nhân 4: Deadlock (Bế Tắc) trong DB
→ Giải pháp: Review transaction isolation, query order

Nguyên nhân 5: External service slow → @Transactional giữ connection chờ
→ Giải pháp: Tách external service call ra ngoài @Transactional
```

### Pattern Tránh Giữ Connection Quá Lâu

```java
// ❌ SAI: Gọi external API trong transaction
@Transactional
public Order createOrder(CreateOrderRequest request) {
    Order order = orderRepository.save(new Order(request));

    // External API call trong transaction!
    // Connection bị giữ suốt thời gian gọi external service
    paymentService.charge(order);   // Có thể mất 1–5 giây!
    notificationService.send(order); // Thêm 1 giây!

    return order;
}

// ✅ ĐÚNG: Tách external calls ra ngoài transaction
public Order createOrder(CreateOrderRequest request) {
    // Transaction ngắn — chỉ DB operations
    Order order = createOrderInDb(request);

    // External calls sau khi transaction đã commit và connection được trả về pool
    paymentService.charge(order);
    notificationService.send(order);

    return order;
}

@Transactional
private Order createOrderInDb(CreateOrderRequest request) {
    return orderRepository.save(new Order(request));
}
```

---

## 6. Monitoring HikariCP

### Tích Hợp Micrometer

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: metrics
  metrics:
    tags:
      application: ${spring.application.name}
```

```java
// Metrics tự động được export khi có HikariCP + Micrometer
// Kiểm tra tại: /actuator/metrics/hikaricp.connections
```

### Metrics Quan Trọng

```
hikaricp.connections.active       — Số connections đang được dùng
hikaricp.connections.idle         — Số connections nhàn rỗi
hikaricp.connections.pending      — Số requests đang chờ connection
hikaricp.connections.total        — Tổng số connections trong pool
hikaricp.connections.timeout.total — Tổng số lần timeout

hikaricp.connections.acquire      — Thời gian chờ để lấy connection
hikaricp.connections.usage        — Thời gian connection được dùng
hikaricp.connections.creation     — Thời gian tạo connection mới
```

### Grafana Dashboard Query

```promql
# Active connections ratio (Tỷ Lệ Connections Đang Dùng)
hikaricp_connections_active / hikaricp_connections_max

# Connection wait time P99 (Thời Gian Chờ P99)
histogram_quantile(0.99, rate(hikaricp_connections_acquire_seconds_bucket[5m]))

# Pool utilization alert (Cảnh Báo Khi Pool > 80%)
hikaricp_connections_active / hikaricp_connections_max > 0.8
```

### Alert Rules (Quy Tắc Cảnh Báo)

```yaml
# Prometheus alert rules
groups:
  - name: hikaricp
    rules:
      - alert: HikariCPPoolExhausted
        expr: hikaricp_connections_pending > 0
        for: 1m
        annotations:
          summary: "Connection pool has pending requests"

      - alert: HikariCPHighUtilization
        expr: hikaricp_connections_active / hikaricp_connections_max > 0.8
        for: 5m
        annotations:
          summary: "Connection pool utilization > 80%"
```

---

## 7. Read Replica Configuration (Cấu Hình Bản Sao Chỉ Đọc)

```java
// Dùng 2 data sources: primary (ghi) và replica (chỉ đọc)
@Configuration
public class DataSourceRoutingConfig {

    @Bean
    @Primary
    public DataSource routingDataSource(
            @Qualifier("primaryDataSource") DataSource primary,
            @Qualifier("replicaDataSource") DataSource replica) {

        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("primary", primary);
        targetDataSources.put("replica", replica);

        AbstractRoutingDataSource routing = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                // Nếu transaction là readOnly → dùng replica
                return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
                    ? "replica"
                    : "primary";
            }
        };

        routing.setTargetDataSources(targetDataSources);
        routing.setDefaultTargetDataSource(primary);
        return routing;
    }
}

// Service dùng read replica tự động
@Transactional(readOnly = true)  // ← readOnly=true → route sang replica
public List<Product> findAll() {
    return repository.findAll();
}

@Transactional  // ← Mặc định readOnly=false → route sang primary
public Product create(CreateProductRequest request) {
    return repository.save(new Product(request));
}
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: Tại sao `maximumPoolSize = 100` lại có thể gây vấn đề?**

```
Nhiều connections hơn không có nghĩa là hiệu năng tốt hơn:
  - Mỗi PostgreSQL connection = 5–10MB RAM
  - DB có max_connections giới hạn (thường 100–200)
  - Quá nhiều connections → context switching overhead
  - Công thức đúng: (CPU cores × 2) + effective_spindle_count
```

**Q: Sự khác biệt giữa `connectionTimeout` và `idleTimeout`?**

```
connectionTimeout: Thời gian tối đa chờ LẤY connection từ pool
  → Nếu vượt quá → SQLTimeoutException

idleTimeout: Thời gian tối đa connection nhàn rỗi trong pool
  → Nếu vượt quá → HikariCP đóng connection đó (giải phóng tài nguyên)
```

**Q: Làm thế nào để debug connection leak?**

```
1. Bật leak-detection-threshold: 2000
2. Kiểm tra logs cho "Apparent connection leak detected"
3. Xem stack trace → tìm file/line gây ra leak
4. Đảm bảo dùng try-with-resources hoặc @Transactional đúng cách
5. Kiểm tra external service calls bên trong @Transactional
```

---

## ✅ Checklist

- [ ] `maximumPoolSize` được tính toán dựa trên tải thực tế (không đặt tùy tiện)
- [ ] `minimumIdle` ≤ `maximumPoolSize` và đủ để handle baseline load
- [ ] `connectionTimeout` được set (không dùng giá trị mặc định 30s nếu cần tighter)
- [ ] `maxLifetime` < giá trị `wait_timeout` của DB server
- [ ] `leakDetectionThreshold` được bật trong staging/production
- [ ] HikariCP metrics được export qua Micrometer
- [ ] Alert được set khi pool utilization > 80%
- [ ] External API calls không nằm trong `@Transactional`

---

**Xem tiếp:** [3-jvm-tuning.md](3-jvm-tuning.md) — JVM & Garbage Collector Tuning
