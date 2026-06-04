# Redshift Performance Optimization — Tối Ưu Hiệu Suất

> Nắm vững DISTKEY (Distribution Key — Khóa Phân Phối), SORTKEY (Sort Key — Khóa Sắp Xếp), DISTSTYLE (Distribution Style — Kiểu Phân Phối) và WLM — Workload Management (Quản Lý Khối Lượng Công Việc) để biến Redshift cluster từ chậm thành nhanh.

---

## 📚 Mục Lục

1. [DISTSTYLE — Kiểu Phân Phối Dữ Liệu](#1-diststyle--kiểu-phân-phối-dữ-liệu)
2. [DISTKEY — Khóa Phân Phối](#2-distkey--khóa-phân-phối)
3. [SORTKEY — Khóa Sắp Xếp](#3-sortkey--khóa-sắp-xếp)
4. [Compound vs Interleaved SORTKEY](#4-compound-vs-interleaved-sortkey)
5. [Table Design Best Practices](#5-table-design-best-practices)
6. [WLM — Workload Management](#6-wlm--workload-management)
7. [Concurrency Scaling — Mở Rộng Đồng Thời](#7-concurrency-scaling--mở-rộng-đồng-thời)
8. [VACUUM và ANALYZE](#8-vacuum-và-analyze)
9. [Query Optimization Techniques](#9-query-optimization-techniques)
10. [Monitoring và Diagnosis](#10-monitoring-và-diagnosis)

---

## 1. DISTSTYLE — Kiểu Phân Phối Dữ Liệu

**DISTSTYLE** quyết định cách Redshift phân phối rows của bảng ra các Slices. Đây là quyết định quan trọng nhất ảnh hưởng đến hiệu suất JOIN và query.

### Các Loại DISTSTYLE

#### AUTO — Tự Động (Mặc Định)

```sql
CREATE TABLE orders (...) DISTSTYLE AUTO;
-- Redshift tự chọn EVEN hoặc KEY dựa trên kích thước bảng
-- Bảng nhỏ → EVEN; Bảng lớn → KEY (nếu có DISTKEY)
-- Khuyến nghị cho người mới bắt đầu
```

#### EVEN — Phân Phối Đều

```sql
CREATE TABLE dim_date (...) DISTSTYLE EVEN;
-- Rows phân phối đều theo round-robin (vòng tròn) ra tất cả Slices
-- Đảm bảo không Slice nào bị "nóng" (hot spot)
-- Dùng khi: không có JOIN với bảng khác, hoặc bảng nhỏ
```

```
Ví dụ DISTSTYLE EVEN — 6 rows, 3 Slices:
  Slice 1: [row1, row4]
  Slice 2: [row2, row5]
  Slice 3: [row3, row6]
→ Mỗi Slice có đúng 2 rows — cân bằng hoàn hảo
```

#### KEY — Phân Phối Theo Khóa

```sql
CREATE TABLE orders (
    order_id    BIGINT,
    customer_id BIGINT,   -- DISTKEY
    amount      DECIMAL(12,2),
    order_date  DATE
) DISTKEY (customer_id);
-- Rows có cùng customer_id sẽ ở cùng Slice
-- Dùng khi: thường xuyên JOIN với bảng khác theo cột này
```

```
Ví dụ DISTKEY (customer_id):
  Slice 1: [customer_id=101, customer_id=105, ...]  (hash(customer_id) mod 3 = 0)
  Slice 2: [customer_id=102, customer_id=106, ...]  (hash(customer_id) mod 3 = 1)
  Slice 3: [customer_id=103, customer_id=107, ...]  (hash(customer_id) mod 3 = 2)

JOIN: orders.customer_id = customers.customer_id
  → Nếu customers cũng DISTKEY(customer_id):
     Cùng customer_id ở cùng Slice → JOIN không cần network redistribution
     → DS_DIST_NONE → NHANH NHẤT
```

#### ALL — Sao Chép Tất Cả Nodes

```sql
CREATE TABLE dim_country (
    country_code VARCHAR(3),
    country_name VARCHAR(100)
) DISTSTYLE ALL;
-- Toàn bộ bảng được sao chép đến MỌI node
-- Dùng khi: bảng nhỏ (<1 triệu rows) và thường JOIN với nhiều bảng lớn
-- Không dùng cho bảng lớn → tốn storage và làm chậm UPDATE/INSERT
```

### So Sánh DISTSTYLE

| DISTSTYLE | Phân Phối | Tốt Cho | Tránh Khi |
|-----------|-----------|---------|-----------|
| **AUTO** | Redshift tự quyết định | Mặc định, bảng mới | — |
| **EVEN** | Round-robin đều | Bảng không JOIN nhiều, DIM tables vừa | Bảng JOIN thường xuyên |
| **KEY** | Hash theo cột | FACT tables, bảng lớn JOIN thường xuyên | Cột DISTKEY có ít giá trị (skew) |
| **ALL** | Copy đến tất cả nodes | Bảng nhỏ (<1M rows) JOIN với nhiều bảng lớn | Bảng lớn, bảng hay UPDATE |

---

## 2. DISTKEY — Khóa Phân Phối

### Cách Chọn DISTKEY Tốt

```
Tiêu chí chọn DISTKEY:
  1. Cột thường dùng để JOIN với bảng fact hoặc dimension lớn nhất
  2. Cardinality (số lượng giá trị duy nhất) cao — tránh data skew
  3. Không phải cột thường filter (WHERE clause) — đó là SORTKEY
```

### Data Skew — Phân Phối Không Đều

```
VẤN ĐỀ: Chọn DISTKEY có ít giá trị duy nhất

Ví dụ: DISTKEY (status) với status = ['active', 'inactive', 'pending']

  Slice 1: [hash('active') → 10M rows]   ← Slice nóng (hot), chậm
  Slice 2: [hash('inactive') → 8M rows]  ← Nặng
  Slice 3: [hash('pending') → 0.5M rows] ← Rảnh rang

→ Query phải chờ Slice 1 hoàn thành → bottleneck (nút thắt cổ chai)
```

### Phát Hiện Data Skew

```sql
-- Kiểm tra phân phối rows trên các slices
SELECT
    TRIM(name) AS table_name,
    slice,
    num_values AS row_count,
    ROUND(100.0 * num_values / SUM(num_values) OVER (PARTITION BY name), 2) AS pct
FROM svv_diskusage
WHERE name = 'orders' AND col = 0
ORDER BY slice;

-- Nếu một slice có >120% trung bình → cân nhắc đổi DISTKEY

-- Kiểm tra data skew bằng query
SELECT
    customer_id,
    COUNT(*) AS row_count
FROM orders
GROUP BY customer_id
ORDER BY row_count DESC
LIMIT 10;
-- Nếu top customer có 50%+ tổng rows → DISTKEY(customer_id) sẽ bị skew
```

---

## 3. SORTKEY — Khóa Sắp Xếp

**SORTKEY** xác định thứ tự vật lý của dữ liệu trên đĩa. Khi dữ liệu được sắp xếp, Redshift dùng **Zone Maps** (Bản Đồ Vùng) để bỏ qua các khối dữ liệu không cần đọc.

### Zone Maps — Bản Đồ Vùng

```
Bảng orders với SORTKEY(order_date) — dữ liệu sắp xếp theo ngày:

  Block 1: [2024-01-01 .. 2024-01-31]   ← min=Jan 1, max=Jan 31
  Block 2: [2024-02-01 .. 2024-02-29]   ← min=Feb 1, max=Feb 29
  Block 3: [2024-03-01 .. 2024-03-31]   ← min=Mar 1, max=Mar 31

Query: WHERE order_date = '2024-02-15'
  → Zone Map: Feb 15 nằm trong khoảng Feb 1 - Feb 29 → chỉ đọc Block 2
  → Block 1 và Block 3 bị BỎ QUA hoàn toàn (Block Skipping — Bỏ Qua Khối)
  → Giảm I/O 66% trong ví dụ này!
```

### Khi Nào SORTKEY Hiệu Quả

```
✅ SORTKEY hiệu quả khi:
   - Cột thường xuyên xuất hiện trong WHERE clause
   - Cột dùng cho range query (BETWEEN, >=, <=)
   - Cột join (kết hợp với DISTKEY cùng cột → tốt nhất)
   - Cột ORDER BY thường xuyên

❌ SORTKEY ít hiệu quả khi:
   - Dữ liệu không còn được sắp xếp (sau nhiều INSERT) → cần VACUUM SORT
   - Cột có quá nhiều giá trị duy nhất mà không dùng range query
   - Bảng rất nhỏ (< vài MB)
```

---

## 4. Compound vs Interleaved SORTKEY

### Compound SORTKEY — Khóa Sắp Xếp Ghép

```sql
-- Compound: ưu tiên theo thứ tự cột từ trái sang phải
CREATE TABLE orders (...)
COMPOUND SORTKEY (order_date, customer_id, product_id);
```

```
Dữ liệu được sắp xếp:
  Đầu tiên theo order_date
  → Trong cùng ngày, theo customer_id
     → Trong cùng customer, theo product_id

Hiệu quả khi query:
  ✅ WHERE order_date = '2024-01'                        → Zone map hiệu quả
  ✅ WHERE order_date = '2024-01' AND customer_id = 100  → Rất hiệu quả
  ⚠️ WHERE customer_id = 100                            → Ít hiệu quả (cột thứ 2)
  ❌ WHERE product_id = 500                             → Không hiệu quả (cột thứ 3)
```

### Interleaved SORTKEY — Khóa Sắp Xếp Xen Kẽ

```sql
-- Interleaved: cân bằng trọng số cho tất cả các cột
CREATE TABLE orders (...)
INTERLEAVED SORTKEY (order_date, customer_id, product_id);
```

```
Dữ liệu được sắp xếp với Z-order curve (đường cong Z):
  Mỗi cột có trọng số ngang nhau

Hiệu quả khi query với bất kỳ cột nào:
  ✅ WHERE order_date = '2024-01'  → Tốt
  ✅ WHERE customer_id = 100       → Tốt (khác với Compound)
  ✅ WHERE product_id = 500        → Tốt (khác với Compound)

Nhưng:
  ⚠️ VACUUM REINDEX tốn kém hơn nhiều
  ⚠️ INSERT mới làm giảm hiệu quả nhanh hơn
  ⚠️ AWS khuyến nghị dùng Compound trong hầu hết trường hợp
```

### Khuyến Nghị Thực Tế

```
Dùng Compound SORTKEY (mặc định) khi:
  - Có 1-3 cột thường dùng trong WHERE
  - Workload có thể dự đoán (luôn filter theo ngày, sau đó theo region)
  - Cần VACUUM nhanh

Cân nhắc Interleaved SORTKEY khi:
  - Nhiều loại query khác nhau, filter theo nhiều cột khác nhau
  - Không thể dự đoán query pattern
  - Bảng ít khi bị INSERT rows mới (mostly read-only)
```

---

## 5. Table Design Best Practices

### Schema Design Cho Redshift

```sql
-- FACT TABLE (Bảng Sự Kiện) — bảng lớn nhất, nhiều rows
CREATE TABLE sales.fact_orders (
    order_id      BIGINT    NOT NULL ENCODE AZ64,
    customer_id   BIGINT    NOT NULL ENCODE AZ64,  -- DISTKEY
    product_id    INTEGER   NOT NULL ENCODE AZ64,
    date_id       INTEGER   NOT NULL ENCODE AZ64,  -- SORTKEY
    amount        DECIMAL(12, 2) ENCODE AZ64,
    quantity      SMALLINT ENCODE AZ64,
    discount      DECIMAL(5, 2) ENCODE AZ64
)
DISTKEY (customer_id)       -- JOIN thường xuyên với dim_customer
SORTKEY (date_id);          -- Filter range theo ngày thường xuyên

-- DIMENSION TABLE lớn (Bảng Chiều Lớn) — JOIN với fact table
CREATE TABLE sales.dim_customer (
    customer_id   BIGINT    NOT NULL ENCODE AZ64,
    customer_name VARCHAR(200) ENCODE ZSTD,
    email         VARCHAR(200) ENCODE ZSTD,
    country_code  VARCHAR(3)   ENCODE ZSTD,
    segment       VARCHAR(50)  ENCODE ZSTD,
    created_date  DATE     ENCODE AZ64
)
DISTKEY (customer_id)  -- Khớp với fact_orders.customer_id → không cần redistribute khi JOIN
SORTKEY (customer_id);

-- DIMENSION TABLE nhỏ (Bảng Chiều Nhỏ) — copy đến tất cả nodes
CREATE TABLE sales.dim_date (
    date_id       INTEGER    NOT NULL,
    full_date     DATE       NOT NULL,
    year          SMALLINT,
    quarter       SMALLINT,
    month         SMALLINT,
    week          SMALLINT,
    day_of_week   SMALLINT,
    is_weekend    BOOLEAN
)
DISTSTYLE ALL           -- Bảng nhỏ, copy đến tất cả nodes
SORTKEY (date_id);
```

### Encoding Tốt Nhất Theo Data Type

```sql
-- Sử dụng ENCODE phù hợp để tối ưu nén và tốc độ đọc
column_int        BIGINT       ENCODE AZ64,     -- Số nguyên
column_decimal    DECIMAL      ENCODE AZ64,     -- Số thực
column_timestamp  TIMESTAMP    ENCODE AZ64,     -- Thời gian
column_date       DATE         ENCODE AZ64,     -- Ngày
column_varchar    VARCHAR(500) ENCODE ZSTD,     -- Text ngắn/dài
column_boolean    BOOLEAN      ENCODE RAW,      -- Boolean
column_status     VARCHAR(20)  ENCODE BYTEDICT, -- Giá trị lặp nhiều (low cardinality)
```

---

## 6. WLM — Workload Management

**WLM** (Workload Management — Quản Lý Khối Lượng Công Việc) kiểm soát cách Redshift phân bổ memory (bộ nhớ) và concurrency (số query chạy đồng thời) cho các loại query khác nhau.

### Automatic WLM vs Manual WLM

```
Automatic WLM (Khuyến Nghị):
  → Redshift tự động phân bổ memory cho từng query
  → Tự scale từ 1 đến ~50 concurrent queries tùy workload
  → Phù hợp cho hầu hết use cases

Manual WLM:
  → Người dùng định nghĩa queue (hàng đợi) và memory allocation
  → Kiểm soát chính xác hơn nhưng phức tạp hơn
  → Dùng khi có SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ) nghiêm ngặt
```

### Manual WLM Configuration

```json
// Ví dụ cấu hình WLM với 3 queue (hàng đợi)
[
  {
    "query_group": ["etl_jobs"],         // Nhóm query ETL
    "query_group_wild_card": 0,
    "user_group": ["etl_user"],          // User chạy ETL
    "memory_percent_to_use": 40,         // 40% tổng memory
    "query_concurrency": 3,              // Tối đa 3 ETL queries cùng lúc
    "max_execution_time": 3600000        // Timeout 1 giờ (ms)
  },
  {
    "query_group": ["reports"],          // Nhóm query báo cáo
    "memory_percent_to_use": 40,         // 40% tổng memory
    "query_concurrency": 10,             // 10 query báo cáo cùng lúc
    "max_execution_time": 300000         // Timeout 5 phút
  },
  {
    "query_group": [],                   // Default queue — tất cả query còn lại
    "memory_percent_to_use": 20,         // 20% tổng memory
    "query_concurrency": 5
  }
]
```

### Query Group Assignment — Gán Query Vào Hàng Đợi

```sql
-- Gán query vào WLM queue cụ thể
SET query_group TO 'etl_jobs';
-- Chạy ETL queries ở đây
INSERT INTO fact_table SELECT ...;

SET query_group TO '';  -- Reset về default

-- Hoặc dùng SET trong session
SET query_group TO 'reports';
SELECT * FROM complex_report_view;
```

### Short Query Acceleration — Tăng Tốc Query Ngắn

**SQA** (Short Query Acceleration — Tăng Tốc Query Ngắn) tự động phát hiện query ngắn và ưu tiên thực thi chúng:

```
Không có SQA:
  Queue: [Long ETL query 10 phút] [Short dashboard query 2 giây] [...]
  → Dashboard query phải chờ ETL xong mới được chạy → 10 phút chờ!

Với SQA:
  Short query được tách ra → chạy trong dedicated short query queue
  → Dashboard query hoàn thành trong 2 giây dù ETL đang chạy
```

```sql
-- Bật SQA trong Parameter Group (Nhóm Tham Số) của cluster
-- (Bật trong AWS Console → Cluster → Parameter Groups → enable_short_query_acceleration = 1)

-- Kiểm tra SQA có hoạt động không
SELECT * FROM stl_wlm_query
WHERE service_class = 14  -- Service class 14 = SQA queue
ORDER BY starttime DESC
LIMIT 10;
```

---

## 7. Concurrency Scaling — Mở Rộng Đồng Thời

**Concurrency Scaling** (Mở Rộng Đồng Thời) tự động thêm cluster capacity khi số lượng concurrent queries vượt quá giới hạn:

```
Bình thường:
  Main cluster xử lý 50 concurrent queries

Lúc cao điểm (peak time):
  Main cluster đầy (50/50 queries)
  Query 51 → Redshift tự động khởi động Concurrency Scaling cluster
  → Query 51 chạy trên scaling cluster (< 1 phút warm-up)
  → Khi traffic giảm → scaling cluster tự tắt

Chi phí:
  - 1 giờ credit miễn phí mỗi ngày cho mỗi main cluster
  - Sau đó tính phí theo giây (~$0.40/giờ cho ra3.xlplus)
```

```sql
-- Bật Concurrency Scaling cho WLM queue
-- Trong Parameter Group, cấu hình:
-- concurrency_scaling = auto (bật tự động)

-- Kiểm tra queries đang dùng Concurrency Scaling
SELECT query, service_class, queue_start_time, exec_start_time
FROM stl_wlm_query
WHERE service_class_name LIKE '%scaling%'
ORDER BY queue_start_time DESC
LIMIT 10;
```

---

## 8. VACUUM và ANALYZE

### VACUUM — Dọn Dẹp Và Sắp Xếp Lại

Sau nhiều DELETE và UPDATE, Redshift cần **VACUUM** để:
1. Lấy lại không gian đĩa từ rows đã xóa
2. Sắp xếp lại dữ liệu theo SORTKEY

```sql
-- VACUUM toàn bộ bảng (sắp xếp lại + thu hồi không gian)
VACUUM FULL orders;

-- VACUUM chỉ thu hồi không gian (không sort lại) — nhanh hơn
VACUUM DELETE ONLY orders;

-- VACUUM chỉ sắp xếp lại (không thu hồi không gian)
VACUUM SORT ONLY orders;

-- VACUUM REINDEX cho Interleaved SORTKEY
VACUUM REINDEX orders;

-- Kiểm tra bảng nào cần VACUUM
SELECT "table", rows, sorted_rows,
       ROUND(100.0 * sorted_rows / rows, 1) AS pct_sorted,
       size, skew_rows
FROM svv_table_info
WHERE rows > 0
ORDER BY pct_sorted ASC
LIMIT 20;
-- Nếu pct_sorted < 95% → cân nhắc VACUUM SORT
```

### Auto Vacuum — VACUUM Tự Động

```
Redshift tự động chạy VACUUM ở background khi cluster idle:
  - Auto SORT: Sắp xếp lại dữ liệu unsorted
  - Auto DELETE: Thu hồi không gian sau DELETE

Cấu hình (mặc định bật):
  enable_vacuum_boost = true   → Ưu tiên VACUUM khi cluster nhàn
  vacuum_cost_delay = 0        → Không delay giữa các vacuum operation
```

### ANALYZE — Cập Nhật Thống Kê

Query optimizer cần thống kê chính xác để chọn execution plan tốt:

```sql
-- ANALYZE toàn bộ bảng
ANALYZE orders;

-- ANALYZE chỉ một số cột
ANALYZE orders (customer_id, order_date, amount);

-- ANALYZE với tỷ lệ sampling (lấy mẫu) — nhanh hơn cho bảng lớn
ANALYZE orders PREDICATE COLUMNS;
-- Chỉ analyze các cột thường xuất hiện trong WHERE, JOIN, GROUP BY

-- Kiểm tra khi nào bảng được analyze lần cuối
SELECT "table", stats_off, size
FROM svv_table_info
WHERE stats_off > 20  -- stats_off > 20% nghĩa là cần ANALYZE
ORDER BY stats_off DESC;
```

---

## 9. Query Optimization Techniques

### 1. Giảm Data Movement (Di Chuyển Dữ Liệu Qua Mạng)

```sql
-- BAD: Hai bảng lớn với DISTKEY khác nhau → phải redistribute
-- orders DISTKEY(customer_id), transactions DISTKEY(transaction_id)
SELECT o.order_id, t.transaction_id
FROM orders o
JOIN transactions t ON o.order_id = t.order_id;
-- EXPLAIN sẽ hiện DS_DIST_BOTH → cả hai bảng đều redistribute → CHẬM

-- GOOD: Đảm bảo hai bảng JOIN có cùng DISTKEY
-- Đổi transactions sang DISTKEY(order_id)
CREATE TABLE transactions (...) DISTKEY(order_id);
-- Bây giờ JOIN không cần redistribute → DS_DIST_NONE → NHANH
```

### 2. Tối Ưu JOIN Order

```sql
-- Redshift optimizer thường tự chọn tốt, nhưng có thể hint thủ công
-- Bảng nhỏ nên ở bên RIGHT của JOIN để được broadcast

-- Thêm hint (gợi ý) trong query (Redshift SQL hint)
SELECT /*+ broadcast(dim_date) */
    o.order_id, d.full_date
FROM fact_orders o
JOIN dim_date d ON o.date_id = d.date_id;
-- broadcast(dim_date) → buộc Redshift broadcast dim_date đến tất cả nodes
```

### 3. Sử Dụng Late Binding Views

```sql
-- Late Binding View (Khung Nhìn Ràng Buộc Muộn) — không kiểm tra schema khi tạo view
-- Hữu ích khi underlying table có thể thay đổi
CREATE VIEW sales_summary
WITH NO SCHEMA BINDING AS
SELECT
    customer_id,
    SUM(amount) AS total_amount,
    COUNT(*) AS order_count
FROM orders
GROUP BY customer_id;
```

### 4. Dùng COPY Thay Vì INSERT

```sql
-- SLOW: INSERT từng row hoặc INSERT ... SELECT nhỏ
INSERT INTO orders VALUES (1, 100, 500.0, '2024-01-01');

-- FAST: COPY từ S3 — parallel load, nhanh hơn rất nhiều
COPY orders (order_id, customer_id, amount, order_date)
FROM 's3://my-datalake/orders/2024/'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftS3Role'
FORMAT AS PARQUET
COMPUPDATE OFF       -- Tắt nén tự động nếu đã biết encoding
STATUPDATE OFF;      -- Tắt tự động ANALYZE (chạy ANALYZE sau)

-- Sau khi COPY lớn:
ANALYZE orders;      -- Cập nhật thống kê
VACUUM SORT ONLY orders;  -- Sắp xếp lại (nếu cần)
```

### 5. Tránh SELECT *

```sql
-- BAD: Đọc toàn bộ cột
SELECT * FROM orders WHERE order_date = '2024-01-01';

-- GOOD: Chỉ đọc cột cần thiết
SELECT order_id, customer_id, amount
FROM orders
WHERE order_date = '2024-01-01';
-- Columnar storage → chỉ đọc 3 cột thay vì tất cả
```

---

## 10. Monitoring và Diagnosis

### Xem Query Performance

```sql
-- Top 10 query chậm nhất trong 7 ngày qua
SELECT
    TRIM(u.usename) AS username,
    q.query,
    ROUND(q.elapsed / 1000000.0, 2) AS elapsed_sec,
    q.rows,
    TRIM(SUBSTRING(qt.querytxt, 1, 100)) AS query_text
FROM stl_query q
JOIN pg_user u ON q.userid = u.usesysid
JOIN stl_querytext qt ON q.query = qt.query AND qt.sequence = 0
WHERE q.starttime > GETDATE() - INTERVAL '7 days'
  AND q.aborted = 0
ORDER BY q.elapsed DESC
LIMIT 10;

-- Xem query plans đã thực thi
SELECT query, nodeid, parentid, plannode, info
FROM stl_explain
WHERE query = <query_id>
ORDER BY nodeid;

-- Tìm queries bị reject vì queue WLM đầy
SELECT query, service_class, queue_start_time,
       DATEDIFF(seconds, queue_start_time, exec_start_time) AS queue_wait_sec
FROM stl_wlm_query
WHERE queue_start_time > GETDATE() - INTERVAL '1 day'
ORDER BY queue_wait_sec DESC
LIMIT 20;
```

### Phát Hiện Bottlenecks

```sql
-- Bước 1: Xem query execution steps và thời gian mỗi bước
SELECT query, step, label,
       SUM(rows) AS rows_processed,
       SUM(bytes) AS bytes_processed,
       MAX(elapsed_time) AS max_step_time_us
FROM svl_query_report
WHERE query = <query_id>
GROUP BY query, step, label
ORDER BY step;

-- Bước 2: Tìm bước tốn thời gian nhất
-- Thường là: Hash Join, Nested Loop, hoặc Network steps

-- Bước 3: Kiểm tra skew (mất cân bằng giữa các slices)
SELECT query, segment, step, label,
       MAX(rows) AS max_rows_slice,
       MIN(rows) AS min_rows_slice,
       ROUND(MAX(rows) * 1.0 / NULLIF(MIN(rows), 0), 2) AS skew_ratio
FROM svl_query_report
WHERE query = <query_id>
GROUP BY query, segment, step, label
HAVING MAX(rows) > 0
ORDER BY skew_ratio DESC;
-- skew_ratio > 5 → có vấn đề với DISTKEY → cân nhắc thay đổi
```

---

## 🔑 Tóm Tắt Key Points

```
1. DISTSTYLE KEY + matching DISTKEY trên cả hai bảng JOIN = không cần redistribute = nhanh nhất
2. SORTKEY theo cột filter range (date, timestamp) → Zone Map → bỏ qua blocks không cần đọc
3. DISTSTYLE ALL cho bảng nhỏ (<1M rows) JOIN thường xuyên với fact table lớn
4. Tránh DISTKEY có ít giá trị duy nhất (low cardinality) → data skew → bottleneck
5. Compound SORTKEY phổ biến hơn Interleaved trong thực tế
6. WLM queue: tách ETL (memory nhiều, concurrency ít) khỏi BI queries (concurrency cao)
7. COPY từ S3 > INSERT rows riêng lẻ — luôn dùng COPY cho bulk load
8. VACUUM định kỳ sau nhiều DELETE; ANALYZE sau khi load dữ liệu mới
9. Đọc EXPLAIN output để phát hiện DS_DIST_BOTH (mắc nhất) và tối ưu
10. SQA (Short Query Acceleration) giúp dashboard query không bị ETL block
```

---

**Tiếp Theo:** [3-redshift-spectrum.md](./3-redshift-spectrum.md) — Redshift Spectrum — Query S3 trực tiếp, data lakehouse pattern
