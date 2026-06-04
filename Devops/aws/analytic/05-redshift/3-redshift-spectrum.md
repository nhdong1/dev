# Redshift Spectrum — Query S3 Trực Tiếp Từ Redshift

> Redshift Spectrum cho phép query dữ liệu lưu trên Amazon S3 trực tiếp từ Redshift mà không cần load vào cluster. Đây là nền tảng để xây dựng kiến trúc **Data Lakehouse** — kết hợp tính linh hoạt của Data Lake và hiệu suất của Data Warehouse.

---

## 📚 Mục Lục

1. [Redshift Spectrum Là Gì?](#1-redshift-spectrum-là-gì)
2. [Kiến Trúc Spectrum](#2-kiến-trúc-spectrum)
3. [Thiết Lập Spectrum](#3-thiết-lập-spectrum)
4. [External Schema và External Table](#4-external-schema-và-external-table)
5. [Các Định Dạng Dữ Liệu Được Hỗ Trợ](#5-các-định-dạng-dữ-liệu-được-hỗ-trợ)
6. [Partitioning — Phân Vùng Dữ Liệu S3](#6-partitioning--phân-vùng-dữ-liệu-s3)
7. [Query Optimization Cho Spectrum](#7-query-optimization-cho-spectrum)
8. [Data Lakehouse Pattern](#8-data-lakehouse-pattern)
9. [Spectrum vs Athena — Khi Nào Dùng Gì](#9-spectrum-vs-athena--khi-nào-dùng-gì)
10. [Chi Phí Spectrum](#10-chi-phí-spectrum)
11. [Hands-on: Query S3 Với Spectrum](#11-hands-on-query-s3-với-spectrum)

---

## 1. Redshift Spectrum Là Gì?

Redshift Spectrum là tính năng của Redshift cho phép:

```
Trước Spectrum (không có Spectrum):
  S3 (cold data) → COPY → Redshift cluster (hot data) → Query
  → Phải load toàn bộ dữ liệu vào Redshift trước khi query
  → Tốn storage Redshift, tốn thời gian COPY

Với Spectrum:
  S3 (cold data) ←→ Redshift Spectrum ←→ Redshift cluster
  → Query trực tiếp dữ liệu trên S3 từ Redshift SQL
  → JOIN dữ liệu S3 với dữ liệu trong Redshift cluster
  → Không cần COPY, tiết kiệm storage Redshift
```

### Use Cases Chính

```
1. Historical Data Query (Truy Vấn Dữ Liệu Lịch Sử):
   → Dữ liệu 2+ năm lưu trên S3 (rẻ), dữ liệu gần đây trong Redshift
   → Query JOIN dữ liệu cũ và mới trong cùng SQL

2. Data Lakehouse (Kho Dữ Liệu Hồ):
   → S3 làm data lake, Redshift làm query engine cho toàn bộ data lake

3. ETL Staging (Dàn Dựng ETL):
   → Query dữ liệu thô trên S3 → Transform → Load vào Redshift

4. Cost Optimization (Tối Ưu Chi Phí):
   → Chuyển dữ liệu lạnh từ Redshift sang S3 → tiết kiệm storage cost
   → Vẫn query được qua Spectrum khi cần
```

---

## 2. Kiến Trúc Spectrum

```
                    ┌──────────────────────────────┐
                    │        SQL Client             │
                    └──────────────┬───────────────┘
                                   │
                    ┌──────────────▼───────────────┐
                    │     Redshift Leader Node      │
                    │                               │
                    │  Phân tích query:             │
                    │  - Phần nào cần data từ       │
                    │    Redshift storage           │
                    │  - Phần nào cần data từ S3    │
                    └──────┬──────────────┬─────────┘
                           │              │
              ┌────────────▼──┐    ┌──────▼────────────────────────────┐
              │ Redshift      │    │       Redshift Spectrum            │
              │ Compute Nodes │    │       (Fleet Riêng Biệt)          │
              │               │    │                                   │
              │ Data trong    │    │  Hàng ngàn Spectrum nodes         │
              │ Redshift SSD  │    │  song song đọc dữ liệu từ S3      │
              │ hoặc RMS      │    │                                   │
              └────────┬──────┘    └──────────────────┬────────────────┘
                       │                              │
                       │           ┌──────────────────▼────────────────┐
                       │           │           Amazon S3               │
                       │           │                                   │
                       │           │  s3://datalake/orders/            │
                       │           │  ├── year=2023/month=01/          │
                       │           │  ├── year=2023/month=02/          │
                       │           │  └── ...                          │
                       │           └───────────────────────────────────┘
                       │                              │
                    ┌──▼──────────────────────────────▼────┐
                    │          Leader Node                  │
                    │    Merge results, return to client    │
                    └───────────────────────────────────────┘
```

### Điểm Quan Trọng Về Kiến Trúc

- **Spectrum nodes là fleet riêng biệt** — không dùng Compute Nodes của cluster
- Spectrum tự động scale theo lượng dữ liệu cần query
- Leader Node điều phối cả Redshift data và Spectrum data
- Phần tính toán nặng được **pushdown** (đẩy xuống) đến Spectrum nodes — giảm data transfer

---

## 3. Thiết Lập Spectrum

### Bước 1: Tạo IAM Role Cho Redshift Truy Cập S3

```json
// redshift-spectrum-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::my-datalake",
        "arn:aws:s3:::my-datalake/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "glue:CreateDatabase",
        "glue:DeleteDatabase",
        "glue:GetDatabase",
        "glue:GetDatabases",
        "glue:UpdateDatabase",
        "glue:CreateTable",
        "glue:DeleteTable",
        "glue:BatchDeleteTable",
        "glue:UpdateTable",
        "glue:GetTable",
        "glue:GetTables",
        "glue:BatchCreatePartition",
        "glue:CreatePartition",
        "glue:DeletePartition",
        "glue:BatchDeletePartition",
        "glue:UpdatePartition",
        "glue:GetPartition",
        "glue:GetPartitions",
        "glue:BatchGetPartition"
      ],
      "Resource": "*"
    }
  ]
}
```

```bash
# Tạo IAM Role
aws iam create-role \
  --role-name RedshiftSpectrumRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "redshift.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam put-role-policy \
  --role-name RedshiftSpectrumRole \
  --policy-name SpectrumS3GluePolicy \
  --policy-document file://redshift-spectrum-policy.json

# Gắn role vào Redshift cluster
aws redshift modify-cluster-iam-roles \
  --cluster-identifier my-analytics-cluster \
  --add-iam-roles arn:aws:iam::123456789:role/RedshiftSpectrumRole
```

### Bước 2: Tạo External Schema Trong Redshift

```sql
-- Tạo External Schema (Lược Đồ Ngoài) trỏ đến Glue Catalog
CREATE EXTERNAL SCHEMA spectrum_schema
FROM DATA CATALOG
DATABASE 'spectrum_db'        -- Database trong Glue Catalog (tạo nếu chưa có)
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftSpectrumRole'
CREATE EXTERNAL DATABASE IF NOT EXISTS;
-- CREATE EXTERNAL DATABASE IF NOT EXISTS → tạo database trong Glue nếu chưa có
```

---

## 4. External Schema và External Table

### External Schema

External Schema trong Redshift ánh xạ đến một **database trong Glue Data Catalog**:

```sql
-- Tạo external schema từ Glue Catalog
CREATE EXTERNAL SCHEMA spectrum
FROM DATA CATALOG
DATABASE 'my_datalake_db'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftSpectrumRole';

-- Tạo external schema từ Hive Metastore (cho EMR)
CREATE EXTERNAL SCHEMA hive_schema
FROM HIVE METASTORE
DATABASE 'hive_db'
URI 'thrift://hive-metastore-host:9083'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftSpectrumRole';
```

### External Table — Bảng Ngoài

```sql
-- Tạo external table trỏ đến S3
CREATE EXTERNAL TABLE spectrum.orders (
    order_id     BIGINT,
    customer_id  BIGINT,
    product_id   INTEGER,
    amount       DECIMAL(12, 2),
    status       VARCHAR(20),
    region       VARCHAR(50),
    order_date   DATE
)
PARTITIONED BY (year INT, month INT)   -- Partition columns (cột phân vùng)
STORED AS PARQUET
LOCATION 's3://my-datalake/orders/';

-- Thêm partitions thủ công
ALTER TABLE spectrum.orders
ADD PARTITION (year=2024, month=1)
LOCATION 's3://my-datalake/orders/year=2024/month=1/';

-- Hoặc để Glue Crawler tự phát hiện partitions
-- (Crawl xong → partitions tự động xuất hiện trong Spectrum)
```

### Xem Existing External Tables (Từ Glue Catalog)

```sql
-- Nếu đã có tables trong Glue Catalog, Spectrum tự thấy ngay
-- sau khi CREATE EXTERNAL SCHEMA trỏ đến đúng database

-- Kiểm tra external tables
SELECT schemaname, tablename, location, input_format
FROM svv_external_tables
ORDER BY schemaname, tablename;

-- Kiểm tra external partitions
SELECT schemaname, tablename, values, location
FROM svv_external_partitions
WHERE tablename = 'orders'
ORDER BY values;
```

---

## 5. Các Định Dạng Dữ Liệu Được Hỗ Trợ

| Định Dạng | Nén Hỗ Trợ | Hiệu Suất | Ghi Chú |
|-----------|------------|-----------|---------|
| **Parquet** | SNAPPY, GZIP, ZSTD | ⭐⭐⭐⭐⭐ | Tốt nhất — columnar |
| **ORC** | ZLIB, SNAPPY | ⭐⭐⭐⭐⭐ | Tốt — columnar |
| **Avro** | DEFLATE, SNAPPY | ⭐⭐⭐ | Phù hợp streaming |
| **JSON** | GZIP | ⭐⭐ | Chậm — text format |
| **CSV/TSV** | GZIP, BZIP2 | ⭐⭐ | Chậm — text format |
| **Ion** | GZIP | ⭐⭐⭐ | Amazon Ion format |
| **RegexSerDe** | GZIP | ⭐⭐ | Log files với regex |

```
Khuyến nghị:
  → Luôn dùng Parquet hoặc ORC cho data lake
  → Nén SNAPPY cho cân bằng giữa tốc độ và nén
  → Nén ZSTD khi ưu tiên tỷ lệ nén cao (lưu trữ lâu dài)
  → File size lý tưởng: 128 MB - 1 GB mỗi file
```

---

## 6. Partitioning — Phân Vùng Dữ Liệu S3

### Tại Sao Partitioning Quan Trọng Với Spectrum?

```
Không có partitioning:
  s3://datalake/orders/
    ├── file1.parquet  (1 GB — toàn bộ năm 2024)
    ├── file2.parquet  (1 GB)
    └── ...

  Query: WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31'
  → Spectrum phải đọc TẤT CẢ files → scan toàn bộ S3 → CHẬM và TỐN TIỀN

Có partitioning:
  s3://datalake/orders/year=2024/month=1/
    ├── file1.parquet  (100 MB — chỉ tháng 1/2024)
    └── file2.parquet

  Query: WHERE year=2024 AND month=1
  → Spectrum chỉ đọc thư mục year=2024/month=1/ → NHANH và RẺ
```

### Partition Pruning — Loại Bỏ Partition

```sql
-- Query SẼ dùng partition pruning (loại bỏ partition không cần thiết)
SELECT *
FROM spectrum.orders
WHERE year = 2024        -- ✅ Partition column → chỉ đọc year=2024/
  AND month = 1          -- ✅ Partition column → chỉ đọc month=1/

-- Query KHÔNG dùng partition pruning
SELECT *
FROM spectrum.orders
WHERE EXTRACT(YEAR FROM order_date) = 2024  -- ❌ Tính toán trên date column
                                             --    không phải partition column year
-- → Spectrum phải scan toàn bộ dữ liệu
```

### Partition Discovery Tự Động Với Glue Crawler

```bash
# Tạo Glue Crawler để tự phát hiện partitions mới trong S3
aws glue create-crawler \
  --name spectrum-orders-crawler \
  --role arn:aws:iam::123456789:role/GlueServiceRole \
  --database-name spectrum_db \
  --targets '{"S3Targets": [{"Path": "s3://my-datalake/orders/"}]}' \
  --schedule 'cron(0 6 * * ? *)'  # Chạy lúc 6 giờ sáng mỗi ngày

# Hoặc cập nhật partitions thủ công trong SQL
ALTER TABLE spectrum.orders
ADD PARTITION (year=2024, month=2)
LOCATION 's3://my-datalake/orders/year=2024/month=2/';
```

---

## 7. Query Optimization Cho Spectrum

### Predicate Pushdown — Đẩy Filter Xuống Spectrum

Redshift tự động đẩy WHERE clauses xuống Spectrum nodes để lọc dữ liệu trước khi gửi về Leader Node:

```sql
-- Tốt: Filter được pushdown đến Spectrum
SELECT customer_id, SUM(amount)
FROM spectrum.orders
WHERE year = 2024             -- ✅ Partition pruning
  AND status = 'COMPLETED'    -- ✅ Pushdown filter đến Spectrum nodes
  AND amount > 100            -- ✅ Pushdown filter
GROUP BY customer_id;

-- Xem Spectrum execution stats
SELECT s.query, s.is_rrscan, s.rows, s.bytes,
       s.files, s.files_scanned, s.splits
FROM svl_s3query_summary s
WHERE s.query = <query_id>;
-- files vs files_scanned → nếu files_scanned << files → partition pruning hoạt động tốt
```

### Tối Ưu File Layout Trên S3

```
Nguyên tắc:
  1. File size: 128 MB - 1 GB (quá nhỏ → overhead; quá lớn → ít parallelism)
  2. Số file trong một partition: 10-100 files (cân bằng parallelism)
  3. Tránh nhiều file nhỏ < 10 MB → "small files problem"

Small Files Problem (Vấn Đề Tệp Nhỏ):
  1000 file x 1 MB = 1 GB data
  → Spectrum tạo 1000 tasks, overhead khởi động > thời gian đọc thực tế

Giải pháp: Compact nhỏ files trong Glue ETL:
```

```python
# Glue ETL: Compact nhiều file nhỏ thành file lớn hơn
import awswrangler as wr

# Đọc partition nhỏ
df = wr.s3.read_parquet(
    path='s3://my-datalake/orders/year=2024/month=1/',
    dataset=True
)

# Ghi lại với ít file hơn (repartition)
wr.s3.to_parquet(
    df=df,
    path='s3://my-datalake/orders-compacted/year=2024/month=1/',
    max_rows_by_file=2_000_000,  # ~2M rows mỗi file
    dataset=True,
    partition_cols=['year', 'month']
)
```

---

## 8. Data Lakehouse Pattern

### Kiến Trúc Medallion Với Spectrum

```
                    Amazon S3 — Data Lake
┌───────────────────────────────────────────────────────────────┐
│  Bronze Layer (Lớp Thô — Raw):                                │
│  s3://datalake/bronze/orders/                                 │
│  → Dữ liệu gốc từ nguồn, không chỉnh sửa                     │
│  → Format: JSON, CSV gốc hoặc Parquet                         │
│                                                               │
│  Silver Layer (Lớp Sạch — Cleaned):                           │
│  s3://datalake/silver/orders/                                 │
│  → Đã làm sạch, chuẩn hóa, loại bỏ duplicate                 │
│  → Format: Parquet + SNAPPY, partitioned                      │
│                                                               │
│  Gold Layer (Lớp Vàng — Aggregated):                          │
│  s3://datalake/gold/orders_daily_summary/                     │
│  → Đã aggregate, sẵn sàng cho BI                              │
│  → Format: Parquet, ít partitions hơn                         │
└───────────────────────────────────────────────────────────────┘
                              │
                     Redshift Spectrum
                              │
              ┌───────────────┴───────────────┐
              │    Redshift Cluster           │
              │    Hot Data (dữ liệu nóng):   │
              │    - 90 ngày gần nhất         │
              │    - Đã load vào Redshift     │
              │    - Query < 1 giây           │
              └───────────────────────────────┘

SQL Query — Kết hợp hot và cold data:
  SELECT date_trunc('month', order_date) AS month,
         SUM(amount) AS revenue
  FROM (
      -- Cold data (> 90 ngày): từ S3 qua Spectrum
      SELECT order_date, amount
      FROM spectrum.orders
      WHERE year < 2024

      UNION ALL

      -- Hot data (90 ngày gần nhất): từ Redshift storage
      SELECT order_date, amount
      FROM redshift_schema.orders
      WHERE order_date >= CURRENT_DATE - 90
  ) combined
  GROUP BY 1
  ORDER BY 1;
```

### Unloading Dữ Liệu Cũ Từ Redshift Sang S3

```sql
-- Unload (xuất) dữ liệu cũ từ Redshift sang S3 (Spectrum) để tiết kiệm storage
UNLOAD (
    'SELECT * FROM orders WHERE order_date < ''2023-01-01'''
)
TO 's3://my-datalake/cold-orders/year=2022/'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftSpectrumRole'
FORMAT AS PARQUET
PARTITION BY (order_date)
ALLOWOVERWRITE;

-- Sau khi verify dữ liệu đã có trên S3, xóa khỏi Redshift để tiết kiệm storage
DELETE FROM orders WHERE order_date < '2023-01-01';
VACUUM DELETE ONLY orders;

-- Thêm partition vào Spectrum để vẫn query được
ALTER TABLE spectrum.cold_orders
ADD PARTITION (year=2022, month=1) LOCATION 's3://my-datalake/cold-orders/year=2022/month=1/'
ADD PARTITION (year=2022, month=2) LOCATION 's3://my-datalake/cold-orders/year=2022/month=2/';
-- ... thêm tất cả months
```

---

## 9. Spectrum vs Athena — Khi Nào Dùng Gì

| Tiêu Chí | Redshift Spectrum | Amazon Athena |
|----------|------------------|---------------|
| **Tích hợp** | Tích hợp sâu với Redshift cluster | Độc lập, serverless hoàn toàn |
| **JOIN với data warehouse** | Dễ dàng — cùng SQL, cùng cluster | Khó hơn — cần Federated Query |
| **Chi phí** | $5/TB quét (giống Athena) + chi phí Redshift cluster | $5/TB quét, không cần cluster |
| **Hiệu suất JOIN** | Nhanh hơn khi JOIN với Redshift tables | Chỉ query S3, không join được với Redshift native |
| **Quản lý** | Cần Redshift cluster đang chạy | Serverless, không cần quản lý |
| **Tốt nhất cho** | Kết hợp hot (Redshift) và cold (S3) data | Ad-hoc query thuần S3 |

```
Khi nào dùng Spectrum:
  ✅ Đang có Redshift cluster
  ✅ Cần JOIN dữ liệu S3 với dữ liệu trong Redshift
  ✅ Muốn single query engine (Redshift SQL) cho cả hot và cold data
  ✅ ETL staging: query S3 → transform → INSERT INTO Redshift

Khi nào dùng Athena:
  ✅ Không có Redshift cluster (hoặc không muốn chi phí cluster)
  ✅ Chỉ cần query S3 thuần, không join với data warehouse
  ✅ Ad-hoc exploration, không thường xuyên
  ✅ Lambda/API backend cần query S3 (serverless stack)
```

---

## 10. Chi Phí Spectrum

### Mô Hình Tính Phí

```
$5.00 / TB dữ liệu quét (scanned) từ S3

Ví dụ:
  Query quét 10 TB từ S3 → $50
  Query quét 100 GB → $0.50
  Query quét 1 GB Parquet (thay vì 10 GB CSV) → $0.005

Miễn phí:
  - Data transferred trong cùng Region
  - Requests đến S3 (S3 API calls)
  - Spectrum nodes (không tính phí riêng)

Lưu ý:
  - Tính phí theo dữ liệu thực sự quét sau khi partition pruning
  - Parquet/ORC + SNAPPY nén = scan ít hơn → tiết kiệm chi phí
  - Đây là chi phí Spectrum, không bao gồm chi phí Redshift cluster
```

### Tính Toán Chi Phí

```
Kịch bản: Daily report từ 2 năm dữ liệu
  - 2 năm dữ liệu Parquet trên S3: 500 GB
  - Query filter xuống 1 tháng: 20 GB
  - Chi phí: 20 GB × $0.005/GB = $0.10 mỗi query

So sánh nếu load toàn bộ vào Redshift:
  - 500 GB trong Redshift RA3: ~$0.024/GB/tháng = $12/tháng storage
  - + Chi phí cluster nodes

→ Spectrum tiết kiệm: Chỉ tính phí khi query, không phải storage trong cluster
```

---

## 11. Hands-on: Query S3 Với Spectrum

### Setup Hoàn Chỉnh

```sql
-- 1. Tạo external schema (chạy trong Redshift SQL)
CREATE EXTERNAL SCHEMA spectrum
FROM DATA CATALOG
DATABASE 'analytics_lake'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftSpectrumRole'
CREATE EXTERNAL DATABASE IF NOT EXISTS;

-- 2. Tạo external table cho orders lịch sử trên S3
CREATE EXTERNAL TABLE spectrum.orders_history (
    order_id     BIGINT,
    customer_id  BIGINT,
    product_id   INTEGER,
    amount       DECIMAL(12, 2),
    status       VARCHAR(20),
    region       VARCHAR(50),
    order_date   DATE
)
PARTITIONED BY (year INT, month INT)
STORED AS PARQUET
LOCATION 's3://my-datalake/orders-history/';

-- 3. Load partitions từ S3
ALTER TABLE spectrum.orders_history
ADD PARTITION (year=2023, month=1) LOCATION 's3://my-datalake/orders-history/year=2023/month=1/'
ADD PARTITION (year=2023, month=2) LOCATION 's3://my-datalake/orders-history/year=2023/month=2/'
ADD PARTITION (year=2023, month=12) LOCATION 's3://my-datalake/orders-history/year=2023/month=12/';

-- 4. Query kết hợp Spectrum (S3) và Redshift native (hot data)
SELECT
    date_trunc('quarter', order_date) AS quarter,
    SUM(amount) AS total_revenue,
    COUNT(DISTINCT customer_id) AS unique_customers,
    ROUND(AVG(amount), 2) AS avg_order_value
FROM (
    -- Hot data: 2024 trong Redshift
    SELECT order_date, amount, customer_id
    FROM analytics.fact_orders
    WHERE EXTRACT(YEAR FROM order_date) = 2024

    UNION ALL

    -- Cold data: 2023 qua Spectrum
    SELECT order_date, amount, customer_id
    FROM spectrum.orders_history
    WHERE year = 2023
) combined_orders
GROUP BY 1
ORDER BY 1;

-- 5. Kiểm tra query vừa chạy có dùng Spectrum không
SELECT query, is_rrscan, rows, bytes / 1024 / 1024 AS mb_scanned,
       files, files_scanned,
       ROUND(100.0 * files_scanned / NULLIF(files, 0), 1) AS pct_files_scanned
FROM svl_s3query_summary
ORDER BY query DESC
LIMIT 5;
```

### Troubleshooting Spectrum

```sql
-- Xem lỗi Spectrum query
SELECT query, external_query_id, error_message
FROM svl_s3log
WHERE error_message IS NOT NULL
ORDER BY starttime DESC
LIMIT 10;

-- Kiểm tra external tables có accessible không
SELECT * FROM svv_external_schemas;
SELECT * FROM svv_external_tables LIMIT 20;

-- Nếu query bị lỗi "location not found":
-- → Kiểm tra S3 path trong partition definition
-- → Kiểm tra IAM role có quyền s3:GetObject trên path đó không
```

---

## 🔑 Tóm Tắt Key Points

```
1. Spectrum = query S3 trực tiếp từ Redshift → không cần COPY, tiết kiệm storage
2. Kiến trúc: Spectrum nodes là fleet riêng biệt, không chiếm Compute Nodes của cluster
3. Partition pruning là tối ưu quan trọng nhất → chỉ scan partitions cần thiết
4. Parquet/ORC + partition → giảm chi phí Spectrum 90%+ so với CSV không partition
5. Data Lakehouse: Redshift hot data + Spectrum cold data = single query engine
6. $5/TB quét — tính theo dữ liệu THỰC SỰ scan sau partition pruning
7. UNLOAD từ Redshift → S3 → thêm Spectrum partition = tiết kiệm storage cost
8. Spectrum vs Athena: Dùng Spectrum khi cần JOIN với Redshift; Athena khi serverless thuần
9. Small files problem: cần compact files về 128MB-1GB để Spectrum hiệu quả
10. External Schema + Glue Catalog = chia sẻ metadata với Athena, EMR (một catalog cho tất cả)
```

---

**Tiếp Theo:** [4-redshift-serverless.md](./4-redshift-serverless.md) — Redshift Serverless — Auto capacity, RPU, và khi nào chọn Serverless thay vì Provisioned
