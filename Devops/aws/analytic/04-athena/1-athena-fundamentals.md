# Athena Fundamentals — Nền Tảng Amazon Athena

> Hiểu sâu về kiến trúc bên trong, cách Athena xử lý query, tích hợp với S3 và Glue Catalog, và cách thiết lập môi trường làm việc hiệu quả.

---

## 📚 Mục Lục

1. [Kiến Trúc Engine](#1-kiến-trúc-engine)
2. [Presto và Trino — Nền Tảng Query](#2-presto-và-trino--nền-tảng-query)
3. [Athena v2 vs v3 (Athena SQL Engine)](#3-athena-v2-vs-v3)
4. [Tích Hợp S3](#4-tích-hợp-s3)
5. [Glue Data Catalog Integration](#5-glue-data-catalog-integration)
6. [Tạo Table và Database](#6-tạo-table-và-database)
7. [Định Dạng Dữ Liệu Được Hỗ Trợ](#7-định-dạng-dữ-liệu-được-hỗ-trợ)
8. [Thực Thi Query và Vòng Đời](#8-thực-thi-query-và-vòng-đời)
9. [Query Result Location](#9-query-result-location)
10. [Hands-on: Chạy Query Đầu Tiên](#10-hands-on-chạy-query-đầu-tiên)

---

## 1. Kiến Trúc Engine

### Luồng Xử Lý Query

```
Client (Console / JDBC / SDK)
        │
        │  SQL Text
        ▼
┌──────────────────────────────────────────────────────┐
│                  Athena Control Plane                  │
│                                                        │
│  ┌─────────┐   ┌──────────┐   ┌───────────────────┐  │
│  │ Parser  │──▶│ Analyzer │──▶│ Query Planner     │  │
│  │ (SQL→AST│   │(semantic │   │(tạo execution plan│  │
│  │  tree)  │   │ check)   │   │ tối ưu cost-based)│  │
│  └─────────┘   └──────────┘   └─────────┬─────────┘  │
│                                          │             │
│  ┌───────────────────────────────────────▼─────────┐  │
│  │              Distributed Executor                │  │
│  │  Worker 1 │ Worker 2 │ Worker 3 │ ... Worker N  │  │
│  │  (đọc S3  │ (đọc S3  │ (đọc S3  │               │  │
│  │   splits) │  splits) │  splits) │               │  │
│  └─────────────────────────┬───────────────────────┘  │
└────────────────────────────┼─────────────────────────┘
                             │ Kết quả
                             ▼
                    S3 Result Bucket
                    (output location)
```

### Các Thành Phần Chính

| Thành Phần | Vai Trò |
|-----------|---------|
| **Parser** | Phân tích cú pháp SQL thành AST (Abstract Syntax Tree — Cây Cú Pháp Trừu Tượng) |
| **Analyzer** | Kiểm tra semantic (ngữ nghĩa), resolve table/column names từ Catalog |
| **Query Planner** | Tạo execution plan (kế hoạch thực thi) tối ưu dựa trên thống kê |
| **Distributed Executor** | Phân phối công việc đến nhiều worker song song |
| **Connector** | Giao tiếp với data source (S3, JDBC, DynamoDB, ...) |

---

## 2. Presto và Trino — Nền Tảng Query

### Lịch Sử

```
2012: Facebook phát triển Presto (xử lý data warehouse nội bộ)
2015: Facebook open-source Presto
2018: Một số maintainer tách ra tạo PrestoSQL
2020: PrestoSQL đổi tên thành Trino
2022: AWS Athena v3 chuyển sang Trino engine
```

### Kiến Trúc Presto/Trino

```
┌──────────────────────────────────────────────┐
│              Coordinator Node                 │
│  - Nhận query từ client                      │
│  - Phân tích và lập kế hoạch                 │
│  - Phân phối tasks đến Worker nodes          │
│  - Thu thập và trả về kết quả               │
└──────────────────┬───────────────────────────┘
                   │ Task Assignment
        ┌──────────┴───────────┐
        ▼                      ▼
┌──────────────┐      ┌──────────────┐
│ Worker Node 1│      │ Worker Node 2│  ...
│ - Thực thi   │      │ - Thực thi   │
│   tasks       │      │   tasks       │
│ - Đọc data   │      │ - Đọc data   │
│   từ S3      │      │   từ S3      │
└──────────────┘      └──────────────┘
```

### Tính Năng Nổi Bật

- **MPP** (Massively Parallel Processing — Xử Lý Song Song Đại Trà): Phân tán query ra nhiều worker
- **Pipeline Execution** (Thực Thi Pipeline): Các stage chạy song song, không chờ nhau hoàn thành
- **Connector Architecture** (Kiến Trúc Connector): Pluggable system để kết nối nhiều data source
- **Cost-based Optimizer — CBO** (Trình Tối Ưu Dựa Trên Chi Phí): Sử dụng thống kê để chọn join strategy tốt nhất

---

## 3. Athena v2 vs v3

Athena hiện có hai phiên bản engine. **Nên dùng v3** cho tất cả workload mới.

| Tính Năng | Athena v2 (Presto) | Athena v3 (Trino) |
|-----------|-------------------|-------------------|
| **Engine** | Presto 0.217 | Trino (phiên bản mới hơn) |
| **Hiệu suất** | Baseline | Nhanh hơn ~20-30% |
| **SQL** | ANSI SQL cơ bản | ANSI SQL đầy đủ hơn |
| **Window functions** | Giới hạn | Đầy đủ |
| **UNNEST** | Hạn chế | Cải tiến |
| **JSON functions** | Cơ bản | Phong phú hơn |
| **Apache Iceberg** | Hạn chế | Hỗ trợ tốt hơn |
| **ACID transactions** | Không | Qua Iceberg/Hudi/Delta |

### Cách Chuyển Sang v3

```sql
-- Trong Workgroup settings hoặc query:
-- Chọn "Athena engine version 3" trong Workgroup configuration

-- Kiểm tra version đang dùng:
SELECT current_engine_version()
```

---

## 4. Tích Hợp S3

### Cách Athena Đọc Dữ Liệu Từ S3

```
S3 Bucket
└── database/
    └── table/
        ├── year=2024/month=01/day=01/
        │   ├── part-00000.parquet   ← Split 1
        │   ├── part-00001.parquet   ← Split 2
        │   └── part-00002.parquet   ← Split 3
        └── year=2024/month=01/day=02/
            └── part-00000.parquet   ← Split 4

Athena chia thành các "splits" (đơn vị đọc song song)
→ Mỗi worker đọc một split độc lập
→ Kết quả được merge ở coordinator
```

### S3 Prefix Convention (Quy Ước Đặt Tên Prefix)

```
Khuyến nghị:
s3://bucket/database_name/table_name/partition_key=value/

Ví dụ thực tế:
s3://my-datalake/sales/orders/year=2024/month=01/
s3://my-datalake/sales/orders/year=2024/month=02/
```

### IAM Permissions Cần Thiết

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-data-bucket",
        "arn:aws:s3:::my-data-bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::my-query-results/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "athena:StartQueryExecution",
        "athena:GetQueryResults",
        "athena:GetQueryExecution",
        "athena:StopQueryExecution"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "glue:GetDatabase",
        "glue:GetTable",
        "glue:GetPartitions"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 5. Glue Data Catalog Integration

### Glue Catalog Là Metadata Store Trung Tâm

```
┌─────────────────────────────────────────────────┐
│              AWS Glue Data Catalog               │
│                                                  │
│  Database: sales                                 │
│  └── Table: orders                              │
│       ├── Schema: order_id STRING,               │
│       │           amount DOUBLE,                 │
│       │           order_date DATE                │
│       ├── Location: s3://bucket/sales/orders/    │
│       ├── Format: PARQUET                        │
│       └── Partitions: year, month               │
└───────────────────┬──────────────────────────────┘
                    │  Metadata lookup
          ┌─────────┴──────────┐
          ▼                    ▼
    Amazon Athena        Amazon EMR / Redshift Spectrum
    (query engine)       (các engine khác cùng dùng Catalog)
```

### Tạo Database và Table Với Glue Catalog

```sql
-- Tạo database trong Glue Catalog
CREATE DATABASE IF NOT EXISTS sales_db
COMMENT 'Database chứa dữ liệu bán hàng'
LOCATION 's3://my-datalake/sales/';

-- Tạo external table (bảng ngoài — dữ liệu vẫn ở S3)
CREATE EXTERNAL TABLE IF NOT EXISTS sales_db.orders (
    order_id    STRING,
    customer_id STRING,
    amount      DOUBLE,
    status      STRING,
    order_date  DATE
)
PARTITIONED BY (year INT, month INT)
STORED AS PARQUET
LOCATION 's3://my-datalake/sales/orders/'
TBLPROPERTIES ('parquet.compress' = 'SNAPPY');
```

### Thêm Partition Thủ Công (Manual Partition Loading)

```sql
-- Cách 1: Thêm từng partition
ALTER TABLE sales_db.orders
ADD PARTITION (year=2024, month=1)
LOCATION 's3://my-datalake/sales/orders/year=2024/month=1/';

-- Cách 2: MSCK REPAIR TABLE — tự động phát hiện tất cả partitions
-- (chậm hơn với nhiều partitions, nên dùng ALTER TABLE hoặc Glue Crawler)
MSCK REPAIR TABLE sales_db.orders;

-- Cách 3: Partition Projection (tốt nhất — không cần lưu partition metadata)
-- Xem file 2-athena-performance.md để biết chi tiết
```

---

## 6. Tạo Table và Database

### Các Loại Table Trong Athena

#### External Table (Bảng Ngoài) — Phổ Biến Nhất

```sql
CREATE EXTERNAL TABLE cloudtrail_logs (
    eventVersion STRING,
    userIdentity STRUCT<
        type: STRING,
        principalId: STRING,
        arn: STRING,
        accountId: STRING,
        userName: STRING
    >,
    eventTime STRING,
    eventSource STRING,
    eventName STRING,
    awsRegion STRING,
    sourceIPAddress STRING,
    requestParameters STRING,
    responseElements STRING
)
ROW FORMAT SERDE 'com.amazon.emr.hive.serde.CloudTrailSerde'
STORED AS INPUTFORMAT 'com.amazon.emr.cloudtrail.CloudTrailInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://my-cloudtrail-bucket/AWSLogs/123456789/CloudTrail/ap-southeast-1/';
```

#### CTAS — Create Table As Select (Tạo Bảng Từ Kết Quả Query)

```sql
-- Tạo bảng mới từ kết quả query, lưu dưới dạng Parquet
CREATE TABLE sales_db.orders_2024
WITH (
    format = 'PARQUET',
    parquet_compression = 'SNAPPY',
    partitioned_by = ARRAY['month'],
    external_location = 's3://my-datalake/sales/orders_2024/'
)
AS SELECT *
FROM sales_db.orders
WHERE year = 2024;
```

#### View (Khung Nhìn)

```sql
-- View không lưu dữ liệu, chỉ lưu logic query
CREATE OR REPLACE VIEW sales_db.high_value_orders AS
SELECT
    order_id,
    customer_id,
    amount,
    order_date
FROM sales_db.orders
WHERE amount > 1000
  AND status = 'COMPLETED';
```

---

## 7. Định Dạng Dữ Liệu Được Hỗ Trợ

| Định Dạng | Loại | Nén | Hiệu Suất | Khuyến Nghị |
|-----------|------|-----|-----------|-------------|
| **Parquet** | Columnar (Dạng Cột) | SNAPPY, GZIP, ZSTD | ⭐⭐⭐⭐⭐ | Tốt nhất cho analytics |
| **ORC** | Columnar (Dạng Cột) | ZLIB, SNAPPY | ⭐⭐⭐⭐⭐ | Tốt, phổ biến với Hive |
| **Avro** | Row-based (Dạng Hàng) | DEFLATE, SNAPPY | ⭐⭐⭐ | Tốt cho streaming |
| **JSON** | Text (Văn Bản) | GZIP | ⭐⭐ | Linh hoạt, không tối ưu |
| **CSV** | Text (Văn Bản) | GZIP | ⭐⭐ | Đơn giản, không tối ưu |
| **TSV** | Text (Văn Bản) | GZIP | ⭐⭐ | Tương tự CSV |

### Tại Sao Parquet Tốt Hơn CSV?

```
CSV (Row-based — Lưu Theo Hàng):
┌─────┬──────────┬────────┬────────────┐
│ id  │ name     │ amount │ date       │
├─────┼──────────┼────────┼────────────┤
│ 1   │ Alice    │ 100.5  │ 2024-01-01 │  ← Cả hàng được đọc
│ 2   │ Bob      │ 200.0  │ 2024-01-02 │    dù chỉ cần 'amount'
│ 3   │ Carol    │ 150.75 │ 2024-01-03 │
└─────┴──────────┴────────┴────────────┘
Query: SELECT SUM(amount) → phải đọc toàn bộ file

Parquet (Columnar — Lưu Theo Cột):
Column: id     │ Column: name          │ Column: amount      │ Column: date
[1, 2, 3, ...] │ [Alice, Bob, Carol...]│ [100.5, 200.0, 150.]│ [2024-01-01, ...]

Query: SELECT SUM(amount) → chỉ đọc column 'amount'
→ Tiết kiệm 75% dữ liệu đọc → giảm 75% chi phí Athena
```

---

## 8. Thực Thi Query và Vòng Đời

### Vòng Đời Một Query

```
1. QUEUED     → Query đang chờ trong hàng đợi workgroup
2. RUNNING    → Athena đang thực thi query
3. SUCCEEDED  → Query hoàn thành thành công
4. FAILED     → Query thất bại (lỗi SQL, không tìm thấy file, ...)
5. CANCELLED  → User hoặc timeout đã hủy query
```

### Giới Hạn Quan Trọng

| Giới Hạn | Mặc Định | Ghi Chú |
|----------|----------|---------|
| **Query timeout** | 30 phút | Không thể tăng, phải tách query |
| **Concurrent queries** | 20 / account | Có thể yêu cầu tăng qua Support |
| **Query result size** | 1000 rows trong console | Dùng SDK để lấy toàn bộ |
| **Output file size** | ~1 GB per result file | Dùng CTAS để kiểm soát |
| **DDL timeout** | 10 phút | Cho CREATE/DROP operations |

### Xử Lý Query Kết Quả Lớn

```python
import boto3

athena = boto3.client('athena', region_name='ap-southeast-1')

# Bắt đầu query
response = athena.start_query_execution(
    QueryString='SELECT * FROM sales_db.orders WHERE year = 2024',
    QueryExecutionContext={'Database': 'sales_db'},
    ResultConfiguration={
        'OutputLocation': 's3://my-query-results/athena-output/'
    }
)

query_id = response['QueryExecutionId']

# Poll cho đến khi hoàn thành
import time
while True:
    status = athena.get_query_execution(QueryExecutionId=query_id)
    state = status['QueryExecution']['Status']['State']
    
    if state in ['SUCCEEDED', 'FAILED', 'CANCELLED']:
        break
    time.sleep(2)

# Lấy kết quả theo trang (paginated)
paginator = athena.get_paginator('get_query_results')
for page in paginator.paginate(QueryExecutionId=query_id):
    rows = page['ResultSet']['Rows']
    # Xử lý rows...
```

---

## 9. Query Result Location

### Cấu Hình Output Location

```
s3://my-query-results/
└── athena-output/
    ├── a1b2c3d4-e5f6-7890-abcd-ef1234567890.csv      ← Kết quả query
    ├── a1b2c3d4-e5f6-7890-abcd-ef1234567890.csv.metadata  ← Metadata
    └── ...

Lưu ý: Athena giữ kết quả 45 ngày (mặc định)
→ Nên thiết lập S3 lifecycle để tự xóa sau 7-30 ngày
```

### S3 Lifecycle Cho Query Results

```json
{
  "Rules": [
    {
      "ID": "delete-athena-results",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "athena-output/"
      },
      "Expiration": {
        "Days": 30
      }
    }
  ]
}
```

---

## 10. Hands-on: Chạy Query Đầu Tiên

### Dùng AWS Public Dataset (Không Tốn Phí Setup)

```sql
-- 1. Tạo database
CREATE DATABASE IF NOT EXISTS nyc_taxi_lab;

-- 2. Tạo table từ NYC Taxi public dataset
CREATE EXTERNAL TABLE nyc_taxi_lab.yellow_trips (
    vendor_id           STRING,
    pickup_datetime     TIMESTAMP,
    dropoff_datetime    TIMESTAMP,
    passenger_count     INT,
    trip_distance       DOUBLE,
    pickup_longitude    DOUBLE,
    pickup_latitude     DOUBLE,
    rate_code           STRING,
    store_and_fwd_flag  STRING,
    dropoff_longitude   DOUBLE,
    dropoff_latitude    DOUBLE,
    payment_type        STRING,
    fare_amount         DOUBLE,
    surcharge           DOUBLE,
    mta_tax             DOUBLE,
    tip_amount          DOUBLE,
    tolls_amount        DOUBLE,
    total_amount        DOUBLE
)
STORED AS PARQUET
LOCATION 's3://ursa-labs-taxi-data/';

-- 3. Thống kê cơ bản
SELECT
    COUNT(*) AS total_trips,
    ROUND(AVG(trip_distance), 2) AS avg_distance_miles,
    ROUND(AVG(total_amount), 2) AS avg_fare_usd,
    ROUND(SUM(total_amount) / 1000000, 2) AS total_revenue_millions
FROM nyc_taxi_lab.yellow_trips
WHERE year(pickup_datetime) = 2019;

-- 4. Top giờ cao điểm
SELECT
    hour(pickup_datetime) AS pickup_hour,
    COUNT(*) AS trip_count
FROM nyc_taxi_lab.yellow_trips
WHERE year(pickup_datetime) = 2019
GROUP BY hour(pickup_datetime)
ORDER BY trip_count DESC
LIMIT 5;
```

### AWS CLI

```bash
# Chạy query qua CLI
aws athena start-query-execution \
  --query-string "SELECT COUNT(*) FROM sales_db.orders WHERE year = 2024" \
  --query-execution-context "Database=sales_db" \
  --result-configuration "OutputLocation=s3://my-query-results/athena/" \
  --region ap-southeast-1

# Lấy kết quả
aws athena get-query-results \
  --query-execution-id <QUERY_ID> \
  --region ap-southeast-1
```

---

## 🔑 Tóm Tắt Key Points

```
1. Athena = Presto/Trino engine chạy trên S3, serverless hoàn toàn
2. Tính phí $5/TB dữ liệu quét → Parquet + partition = tiết kiệm chi phí
3. Glue Catalog là metadata store — chia sẻ với EMR, Redshift Spectrum
4. External Table = dữ liệu ở S3, DROP TABLE không xóa data
5. CTAS = tạo table mới từ query, lý tưởng để chuyển đổi CSV → Parquet
6. Timeout 30 phút là hard limit — tách query lớn thành nhỏ hơn
7. Query results lưu ở S3 output location — nhớ thiết lập lifecycle
```

---

**Tiếp Theo:** [2-athena-performance.md](./2-athena-performance.md) — Tối ưu hiệu suất với Parquet, ORC, partitioning và compression
