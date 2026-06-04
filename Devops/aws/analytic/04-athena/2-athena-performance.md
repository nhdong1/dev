# Athena Performance Optimization — Tối Ưu Hiệu Suất Athena

> Bí quyết để Athena chạy nhanh hơn và tốn ít tiền hơn: chọn đúng định dạng file, thiết kế partition hợp lý, nén dữ liệu, và dùng các tính năng nâng cao như Partition Projection (Chiếu Partition) và Bucketing (Chia Bucket).

---

## 📚 Mục Lục

1. [Tầm Quan Trọng Của Tối Ưu](#1-tầm-quan-trọng-của-tối-ưu)
2. [Columnar Formats — Parquet và ORC](#2-columnar-formats--parquet-và-orc)
3. [Compression — Nén Dữ Liệu](#3-compression--nén-dữ-liệu)
4. [Partitioning — Phân Vùng Dữ Liệu](#4-partitioning--phân-vùng-dữ-liệu)
5. [Partition Projection — Chiếu Partition](#5-partition-projection--chiếu-partition)
6. [Bucketing — Chia Nhỏ Dữ Liệu](#6-bucketing--chia-nhỏ-dữ-liệu)
7. [File Size Optimization — Tối Ưu Kích Thước File](#7-file-size-optimization--tối-ưu-kích-thước-file)
8. [Query Tuning — Tối Ưu Câu Truy Vấn](#8-query-tuning--tối-ưu-câu-truy-vấn)
9. [Statistics và CBO](#9-statistics-và-cbo)
10. [Monitoring và Diagnostics](#10-monitoring-và-diagnostics)

---

## 1. Tầm Quan Trọng Của Tối Ưu

### Chi Phí Thực Tế — Trước và Sau Tối Ưu

```
Ví dụ: 10 TB dữ liệu CSV, query chạy 100 lần/ngày

Trước tối ưu (CSV, không partition):
  - Mỗi query quét toàn bộ: 10 TB × $5/TB = $50/query
  - 100 queries/ngày: $5,000/ngày = $150,000/tháng

Sau tối ưu (Parquet + partition + ZSTD):
  - Columnar: chỉ đọc columns cần = giảm 70%
  - Partition pruning: chỉ đọc date range cần = giảm 90% thêm
  - Compression: giảm kích thước file 60-80%
  - Kết quả: quét ~0.3 TB × $5/TB = $1.5/query
  - 100 queries/ngày: $150/ngày = $4,500/tháng
  → Tiết kiệm 97% chi phí
```

### Ba Nguyên Tắc Vàng

```
1. SCAN LESS  — Quét ít dữ liệu hơn (partition + columnar)
2. COMPRESS   — Nén dữ liệu (giảm bytes đọc từ S3)
3. QUERY SMART — Viết query thông minh (tránh SELECT *, push predicates)
```

---

## 2. Columnar Formats — Parquet và ORC

### Parquet — Định Dạng Cột Apache

#### Cấu Trúc File Parquet

```
┌──────────────────────────────────────────────┐
│                 Parquet File                  │
├──────────────────────────────────────────────┤
│  File Header (Magic bytes: PAR1)              │
├──────────────────────────────────────────────┤
│  Row Group 1 (128 MB mặc định)               │
│  ├── Column Chunk: id       [1,2,3,...]       │
│  │   ├── Page 1 (encoded + compressed)        │
│  │   ├── Page 2                               │
│  │   └── ...                                  │
│  ├── Column Chunk: name     [Alice, Bob, ...]  │
│  ├── Column Chunk: amount   [100.5, 200.0,...] │
│  └── Column Chunk: date     [2024-01-01, ...]  │
├──────────────────────────────────────────────┤
│  Row Group 2 ...                              │
├──────────────────────────────────────────────┤
│  File Footer (Schema + Row Group Metadata)    │
│  ├── Schema definition                        │
│  ├── Row group offsets                        │
│  └── Column statistics (min/max/null count)   │
└──────────────────────────────────────────────┘
```

#### Tại Sao Footer Statistics Quan Trọng?

```sql
-- Query: WHERE amount > 500
-- Parquet footer chứa: min=50, max=1000 cho mỗi row group
-- Athena đọc footer TRƯỚC → bỏ qua row groups không có dữ liệu thỏa điều kiện
-- → "Predicate pushdown" (Đẩy Điều Kiện Xuống) tự động
```

#### Tạo Parquet Từ CSV Bằng CTAS

```sql
CREATE TABLE sales_db.orders_parquet
WITH (
    format = 'PARQUET',
    parquet_compression = 'SNAPPY',
    partitioned_by = ARRAY['year', 'month'],
    external_location = 's3://my-datalake/sales/orders-parquet/',
    write_compression = 'SNAPPY'
)
AS
SELECT
    order_id,
    customer_id,
    amount,
    status,
    order_date,
    year(order_date) AS year,
    month(order_date) AS month
FROM sales_db.orders_csv;
```

### ORC — Optimized Row Columnar (Dạng Cột Hàng Được Tối Ưu)

ORC là định dạng cột khác, phổ biến trong hệ sinh thái Hive/Spark.

```
ORC vs Parquet — So Sánh:

Parquet:
✅ Tốt hơn cho complex nested types (STRUCT, ARRAY, MAP)
✅ Tích hợp tốt với Spark và Flink
✅ Được dùng phổ biến hơn trong AWS ecosystem

ORC:
✅ Compression tốt hơn một chút
✅ Tốt hơn cho Hive workloads
✅ Bloom filters tốt hơn cho point lookups

→ Với Athena: Dùng Parquet SNAPPY cho hầu hết trường hợp
→ Với Hive-heavy pipelines: ORC ZLIB có thể tốt hơn
```

---

## 3. Compression — Nén Dữ Liệu

### So Sánh Các Thuật Toán Nén

| Thuật Toán | Tốc Độ Nén | Tốc Độ Giải Nén | Tỉ Lệ Nén | Splittable | Khuyến Nghị |
|-----------|-----------|----------------|-----------|------------|-------------|
| **SNAPPY** | Nhanh | Rất nhanh | Trung bình (~50%) | Không | ✅ Mặc định cho Parquet |
| **ZSTD** | Trung bình | Nhanh | Tốt (~60%) | Không | ✅ Tốt nhất cho Parquet |
| **GZIP** | Chậm | Chậm | Rất tốt (~70%) | Không | ⚠️ Cho CSV khi space quan trọng |
| **LZO** | Nhanh | Nhanh | Trung bình | Có | ⚠️ Dùng khi cần splittable |
| **BZIP2** | Rất chậm | Chậm | Rất tốt | Có | ❌ Quá chậm |
| **Không nén** | - | Rất nhanh | 0% | Có | ❌ Lãng phí chi phí |

### Splittable vs Non-Splittable

```
GZIP (non-splittable):
File 1 GB GZIP → 1 worker đọc toàn bộ file
→ Không thể parallelize → chậm

LZO / BZIP2 (splittable):
File 1 GB LZO → nhiều workers đọc song song
→ Parallelize tốt → nhanh hơn

Parquet/ORC với SNAPPY/ZSTD:
Mỗi row group là đơn vị độc lập → implicit splittable
→ Đây là lý do Parquet+SNAPPY là combo tốt nhất
```

### Ví Dụ Chọn Compression

```sql
-- Athena Parquet với ZSTD (tốt nhất về balance)
CREATE TABLE optimized_table
WITH (
    format = 'PARQUET',
    parquet_compression = 'ZSTD'  -- hoặc 'SNAPPY'
)
AS SELECT ...;

-- ORC với ZLIB
CREATE TABLE orc_table
WITH (
    format = 'ORC',
    orc_compression = 'ZLIB'
)
AS SELECT ...;
```

---

## 4. Partitioning — Phân Vùng Dữ Liệu

### Partition Là Gì?

```
Không partition:
s3://bucket/orders/
├── 2024_jan_orders.parquet   ← Query WHERE month=3 vẫn đọc file này
├── 2024_feb_orders.parquet
└── 2024_mar_orders.parquet

Có partition theo year/month:
s3://bucket/orders/year=2024/month=1/data.parquet
s3://bucket/orders/year=2024/month=2/data.parquet
s3://bucket/orders/year=2024/month=3/data.parquet   ← Chỉ đọc file này

Query: WHERE year=2024 AND month=3
→ Athena đọc METADATA (từ Glue Catalog) → biết ngay chỉ cần đọc tháng 3
→ 33% files thay vì 100%
```

### Chọn Partition Key Hợp Lý

#### Nguyên Tắc Chọn Partition Key

```
✅ Tốt:
- Cột thường xuất hiện trong WHERE clause
- Cardinality (lực lượng) vừa phải: date, region, status
- Ví dụ tốt: year, month, day, region, customer_tier

❌ Tránh:
- Quá nhiều partition (high cardinality): user_id (hàng triệu partitions!)
- Quá ít partition: gender (chỉ 2-3 giá trị — không có lợi nhiều)
- Cột không bao giờ dùng trong filter
```

#### Ví Dụ Partition Strategy

```sql
-- E-commerce orders — partition tốt
CREATE EXTERNAL TABLE orders (
    order_id    STRING,
    customer_id STRING,
    amount      DOUBLE,
    status      STRING
)
PARTITIONED BY (
    order_year  INT,      -- Lọc theo năm
    order_month INT,      -- Lọc theo tháng
    region      STRING    -- Lọc theo khu vực
)
STORED AS PARQUET
LOCATION 's3://datalake/orders/';

-- Ví dụ partition path:
-- s3://datalake/orders/order_year=2024/order_month=3/region=SEA/
```

### Hive-Style vs Non-Hive Partition

```
Hive-style (khuyến nghị với Athena):
s3://bucket/data/year=2024/month=01/day=15/file.parquet
→ Athena tự hiểu partition keys và values từ path

Non-Hive-style:
s3://bucket/data/2024/01/15/file.parquet
→ Cần khai báo rõ hơn trong table definition
```

---

## 5. Partition Projection — Chiếu Partition

### Vấn Đề Với Partition Metadata Trong Glue Catalog

```
1 năm dữ liệu phân vùng theo ngày:
→ 365 partitions phải lưu trong Glue Catalog
→ MSCK REPAIR TABLE phải scan tất cả S3 paths
→ Query phải đọc partition metadata từ Catalog trước khi đọc data
→ Chậm khi có hàng nghìn partitions
```

### Partition Projection Giải Quyết Vấn Đề Này

Partition Projection cho phép Athena **tính toán partition values trực tiếp** từ khai báo, không cần tra Glue Catalog.

```sql
CREATE EXTERNAL TABLE logs (
    request_id  STRING,
    user_id     STRING,
    action      STRING,
    status_code INT,
    latency_ms  INT
)
PARTITIONED BY (
    dt          STRING,   -- partition key kiểu date
    region      STRING    -- partition key kiểu enum
)
STORED AS PARQUET
LOCATION 's3://my-logs/app-logs/'
TBLPROPERTIES (
    -- Bật Partition Projection
    'projection.enabled' = 'true',
    
    -- Định nghĩa partition 'dt' kiểu date
    'projection.dt.type'          = 'date',
    'projection.dt.range'         = '2023-01-01,NOW',    -- Từ đầu 2023 đến hiện tại
    'projection.dt.format'        = 'yyyy-MM-dd',
    'projection.dt.interval'      = '1',
    'projection.dt.interval.unit' = 'DAYS',
    
    -- Định nghĩa partition 'region' kiểu enum
    'projection.region.type'   = 'enum',
    'projection.region.values' = 'us-east-1,us-west-2,ap-southeast-1,eu-west-1',
    
    -- Template path
    'storage.location.template' = 's3://my-logs/app-logs/dt=${dt}/region=${region}/'
);
```

### Các Loại Partition Projection

| Loại | Ví Dụ | Dùng Khi |
|------|-------|---------|
| **date** | `2024-01-01` đến `NOW` | Partition theo thời gian |
| **integer** | `0` đến `999` | Partition theo số |
| **enum** | `us-east-1,eu-west-1` | Partition theo danh sách cố định |
| **injected** | Truyền từ query | Giá trị partition dynamic |

### Lợi Ích Của Partition Projection

```
Không có Partition Projection:
  Query → Glue Catalog (đọc metadata) → S3 (đọc data)
  Thêm ~100-500ms overhead với catalog lookup

Có Partition Projection:
  Query → Athena tính toán paths nội bộ → S3 (đọc data)
  → Không cần Glue Catalog lookup
  → Nhanh hơn đặc biệt với nhiều partitions
  → Không cần chạy MSCK REPAIR TABLE
```

---

## 6. Bucketing — Chia Nhỏ Dữ Liệu

Bucketing chia data thành các bucket cố định dựa trên hash của một column. Hữu ích khi thường xuyên join hai bảng lớn theo cùng một key.

```sql
-- Tạo bảng có bucketing (thường kết hợp với Glue ETL để tạo data)
CREATE EXTERNAL TABLE orders_bucketed (
    order_id    STRING,
    customer_id STRING,
    amount      DOUBLE
)
CLUSTERED BY (customer_id) INTO 256 BUCKETS
STORED AS PARQUET
LOCATION 's3://datalake/orders-bucketed/';

-- Tương tự, bảng customers cũng bucketed theo customer_id
CREATE EXTERNAL TABLE customers_bucketed (
    customer_id STRING,
    name        STRING,
    region      STRING
)
CLUSTERED BY (customer_id) INTO 256 BUCKETS
STORED AS PARQUET
LOCATION 's3://datalake/customers-bucketed/';

-- Join sẽ nhanh hơn vì cùng customer_id sẽ ở cùng bucket
SELECT o.order_id, c.name, o.amount
FROM orders_bucketed o
JOIN customers_bucketed c ON o.customer_id = c.customer_id;
```

**Lưu ý:** Bucketing phức tạp hơn partitioning và ít được dùng hơn trong Athena. Chỉ áp dụng khi thực sự cần tối ưu join performance.

---

## 7. File Size Optimization — Tối Ưu Kích Thước File

### Vấn Đề "Small Files" (File Nhỏ)

```
Anti-pattern — Small Files:
s3://bucket/data/year=2024/month=01/
├── part-00001.parquet  (1 KB)   ← Quá nhỏ!
├── part-00002.parquet  (2 KB)   ← Quá nhỏ!
├── part-00003.parquet  (1.5 KB) ← Quá nhỏ!
... (hàng ngàn files)

Vấn đề:
- Mỗi S3 GET request tốn thời gian (overhead cố định ~10-50ms)
- Athena phải mở/đóng hàng ngàn file connections
- Tổng chi phí S3 API tăng cao
- Query chậm dù data nhỏ
```

### Kích Thước File Lý Tưởng

```
Target file size: 128 MB - 512 MB mỗi file Parquet

Quá nhỏ (< 10 MB): Nhiều overhead, chậm
Tối ưu (128-512 MB): Balance tốt giữa parallelism và overhead
Quá lớn (> 1 GB): Ít parallelism, khó đọc song song
```

### Giải Pháp Compact Small Files

```sql
-- Dùng CTAS để compact small files thành files lớn hơn
CREATE TABLE sales_db.orders_compacted
WITH (
    format = 'PARQUET',
    parquet_compression = 'SNAPPY',
    partitioned_by = ARRAY['year', 'month'],
    external_location = 's3://datalake/orders-compacted/'
)
AS SELECT * FROM sales_db.orders_small_files;

-- Sau đó drop table cũ và rename
DROP TABLE sales_db.orders_small_files;
-- Cập nhật Glue Catalog để trỏ vào location mới
```

---

## 8. Query Tuning — Tối Ưu Câu Truy Vấn

### Những Điều Nên Làm

```sql
-- ✅ Chỉ SELECT columns cần thiết
SELECT order_id, amount FROM orders WHERE year=2024;

-- ✅ Luôn filter theo partition key (để partition pruning hoạt động)
SELECT * FROM logs WHERE dt >= '2024-01-01' AND dt < '2024-02-01';

-- ✅ Dùng LIMIT để preview dữ liệu
SELECT * FROM large_table LIMIT 100;

-- ✅ Dùng WITH (CTE) thay vì subquery lồng nhau nhiều cấp
WITH filtered AS (
    SELECT * FROM orders WHERE status = 'COMPLETED'
),
aggregated AS (
    SELECT customer_id, SUM(amount) AS total FROM filtered GROUP BY 1
)
SELECT * FROM aggregated WHERE total > 1000;

-- ✅ Dùng APPROXIMATE COUNT khi không cần chính xác tuyệt đối
SELECT approx_distinct(user_id) AS approx_unique_users FROM events;
```

### Những Điều Cần Tránh

```sql
-- ❌ SELECT * (đọc tất cả columns)
SELECT * FROM orders;  -- Với Parquet, đây là lãng phí

-- ❌ Query không có partition filter
SELECT * FROM logs WHERE user_id = '12345';  -- Scan toàn bộ!

-- ❌ LIKE với wildcard đầu chuỗi (không dùng được index)
SELECT * FROM products WHERE name LIKE '%laptop%';

-- ❌ Hàm trên partition column (ngăn partition pruning)
SELECT * FROM orders WHERE year(order_date) = 2024;  -- Sai!
-- ✅ Nên dùng:
SELECT * FROM orders WHERE order_year = 2024;  -- Dùng partition column trực tiếp
```

### JOIN Optimization

```sql
-- ✅ Broadcast join (join bảng nhỏ): Athena tự phát hiện
-- Bảng nhỏ (< vài trăm MB) sẽ được broadcast đến tất cả workers

-- ✅ Filter trước khi join
SELECT o.order_id, c.name
FROM (SELECT * FROM orders WHERE year = 2024) o
JOIN customers c ON o.customer_id = c.customer_id;

-- ❌ Tránh cross join không có điều kiện
SELECT * FROM table_a, table_b;  -- Cartesian product (tích Đề-các) = thảm họa!
```

---

## 9. Statistics và CBO

### CBO — Cost-Based Optimizer (Trình Tối Ưu Dựa Trên Chi Phí)

Athena v3 (Trino) có CBO có thể dùng **table statistics** (thống kê bảng) để chọn **join order** (thứ tự join) và **join strategy** (chiến lược join) tối ưu.

### Thu Thập Statistics

```sql
-- Athena hỗ trợ ANALYZE để thu thập statistics
ANALYZE sales_db.orders;

-- Thu thập statistics cho partition cụ thể
ANALYZE sales_db.orders PARTITION (year=2024, month=1);
```

### Glue Statistics

```python
import boto3

glue = boto3.client('glue')

# Kích hoạt thu thập statistics tự động trong Glue Catalog
glue.update_table(
    DatabaseName='sales_db',
    TableInput={
        'Name': 'orders',
        'Parameters': {
            'enable-statistics': 'true'
        }
    }
)
```

---

## 10. Monitoring và Diagnostics

### Query Execution Statistics

```python
import boto3

athena = boto3.client('athena')

# Lấy thông tin chi tiết về một query
response = athena.get_query_execution(QueryExecutionId='your-query-id')

execution = response['QueryExecution']
stats = execution['Statistics']

print(f"Data scanned: {stats['DataScannedInBytes'] / (1024**3):.2f} GB")
print(f"Execution time: {stats['TotalExecutionTimeInMillis']} ms")
print(f"Engine exec time: {stats['EngineExecutionTimeInMillis']} ms")
print(f"Queue time: {stats['QueryQueueTimeInMillis']} ms")
print(f"Est cost: ${stats['DataScannedInBytes'] / (1024**4) * 5:.4f}")
```

### CloudWatch Metrics Quan Trọng

```
Namespace: AWS/Athena

Metrics:
- DataScannedInBytes    → Lượng dữ liệu quét (giảm = tốt)
- QueryExecutionTime    → Thời gian thực thi
- EngineExecutionTime   → Thời gian engine chạy thực sự
- QueryQueueTime        → Thời gian chờ trong hàng đợi
- TotalQueryCount       → Số lượng query
- SuccessfulQueryCount  → Query thành công
- FailedQueryCount      → Query thất bại
```

### Query Plan Analysis

```sql
-- Xem execution plan (kế hoạch thực thi) trong Athena console
EXPLAIN
SELECT customer_id, SUM(amount)
FROM orders
WHERE year = 2024
GROUP BY customer_id;

-- Output sẽ hiển thị:
-- - TableScan với các filters được áp dụng
-- - Aggregation operators
-- - Exchange operators (shuffle giữa workers)
```

---

## 🔑 Checklist Tối Ưu Athena

```
File Format:
☑ Chuyển CSV/JSON sang Parquet hoặc ORC
☑ Dùng SNAPPY hoặc ZSTD compression cho Parquet

Partitioning:
☑ Xác định columns thường dùng trong WHERE
☑ Thiết kế partition scheme (year/month/day cho time-series)
☑ Dùng Partition Projection để tránh catalog overhead
☑ Tránh tạo quá nhiều micro-partitions

File Size:
☑ Compact small files thành 128-512 MB per file
☑ Dùng CTAS để compact định kỳ

Query:
☑ Không SELECT *
☑ Luôn filter theo partition key
☑ Dùng APPROXIMATE functions khi phù hợp
☑ Tránh hàm trên partition column trong WHERE

Monitoring:
☑ Track DataScannedInBytes mỗi query
☑ Set workgroup data limit để cảnh báo chi phí
☑ Dùng EXPLAIN để analyze slow queries
```

---

**Tiếp Theo:** [3-athena-federation.md](./3-athena-federation.md) — Federated Query: truy vấn đa nguồn dữ liệu (RDS, DynamoDB, Redis, ...)
