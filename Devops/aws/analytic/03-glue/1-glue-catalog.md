# 📚 AWS Glue Data Catalog — Danh Mục Dữ Liệu Trung Tâm

> Glue Data Catalog là **metadata repository** (kho siêu dữ liệu) trung tâm của AWS — lưu trữ thông tin về cấu trúc, vị trí và định dạng dữ liệu, cho phép Athena, Redshift Spectrum, EMR và Glue ETL cùng chia sẻ một nguồn metadata duy nhất.

## 📚 Mục Lục

1. [Data Catalog Là Gì](#data-catalog-là-gì)
2. [Cấu Trúc Phân Cấp](#cấu-trúc-phân-cấp)
3. [Các Thành Phần Chi Tiết](#các-thành-phần-chi-tiết)
4. [Tích Hợp Với Dịch Vụ AWS](#tích-hợp-với-dịch-vụ-aws)
5. [Quản Lý Data Catalog](#quản-lý-data-catalog)
6. [Schema Evolution](#schema-evolution)
7. [Bảo Mật và Kiểm Soát Truy Cập](#bảo-mật-và-kiểm-soát-truy-cập)
8. [Thực Hành Tốt Nhất](#thực-hành-tốt-nhất)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Data Catalog Là Gì

**Glue Data Catalog** hoạt động như **Apache Hive Metastore** tương thích — nhưng là dịch vụ managed, highly available (sẵn sàng cao), serverless của AWS.

### Vấn Đề Trước Khi Có Data Catalog

```
Trước đây (không có Data Catalog):
─────────────────────────────────
Athena phải tự biết schema → định nghĩa thủ công trong DDL
Redshift phải tự biết schema → tạo external table thủ công
EMR phải tự biết schema → cấu hình Hive Metastore riêng
Glue Job phải tự biết schema → hardcode trong script

→ Mỗi dịch vụ duy trì schema riêng → KHÔNG ĐỒNG BỘ, dễ sai
```

```
Sau khi có Data Catalog:
────────────────────────
Glue Data Catalog ← Crawler tự động phát hiện schema
        │
        ├──► Athena đọc schema từ Catalog
        ├──► Redshift Spectrum đọc schema từ Catalog
        ├──► EMR/Spark đọc schema từ Catalog
        └──► Glue ETL Job đọc schema từ Catalog

→ Một nguồn sự thật duy nhất (Single Source of Truth)
```

### Phân Biệt Metadata vs Data

| Khái Niệm      | Ý Nghĩa                                    | Ví Dụ                                      |
| --------------- | ------------------------------------------ | ------------------------------------------ |
| **Data**        | Dữ liệu thực tế                           | File Parquet chứa hàng triệu records       |
| **Metadata**    | Thông tin về dữ liệu                      | Schema, location, format, partition keys   |
| **Data Catalog**| Hệ thống lưu và quản lý metadata          | Glue Catalog lưu schema của file Parquet   |

---

## 🏗️ Cấu Trúc Phân Cấp

```
AWS Account
└── Glue Data Catalog (1 catalog per region per account)
    │
    ├── Database: raw_data
    │   ├── Table: events_json
    │   │   ├── Schema: event_id STRING, user_id STRING, ts TIMESTAMP, ...
    │   │   ├── Location: s3://my-bucket/raw/events/
    │   │   ├── SerDe: org.openx.data.jsonserde.JsonSerDe
    │   │   └── Partitions:
    │   │       ├── year=2024/month=01/day=01  → s3://my-bucket/raw/events/year=2024/month=01/day=01/
    │   │       └── year=2024/month=01/day=02  → s3://my-bucket/raw/events/year=2024/month=01/day=02/
    │   └── Table: user_profiles_csv
    │
    ├── Database: processed_data
    │   ├── Table: events_parquet
    │   └── Table: user_sessions
    │
    └── Database: aggregated_data
        └── Table: daily_metrics
```

---

## 🔍 Các Thành Phần Chi Tiết

### 1. Database (Cơ Sở Dữ Liệu)

**Là gì:** Nhóm logic của các bảng liên quan — không phải database thực sự mà chỉ là namespace (không gian tên).

```python
# Tạo database qua AWS SDK (boto3)
import boto3

glue = boto3.client('glue', region_name='ap-southeast-1')

glue.create_database(
    DatabaseInput={
        'Name': 'analytics_raw',
        'Description': 'Raw data layer — dữ liệu thô chưa xử lý',
        'LocationUri': 's3://my-data-lake/raw/',
        'Parameters': {
            'layer': 'raw',
            'team': 'data-engineering'
        }
    }
)
```

**Quy ước đặt tên tốt:**
```
{layer}_{domain}_{env}

Ví dụ:
  raw_ecommerce_prod
  processed_ecommerce_prod
  aggregated_ecommerce_prod
  raw_iot_sensors_dev
```

---

### 2. Table (Bảng)

**Là gì:** Định nghĩa schema và metadata của một dataset. Bảng Glue là **external table** (bảng ngoài) — dữ liệu nằm ở S3, Glue chỉ lưu metadata.

**Thành phần của một Table:**

```
Table: orders_parquet
├── StorageDescriptor (Mô Tả Lưu Trữ)
│   ├── Columns (Danh Sách Cột):
│   │   ├── order_id    STRING
│   │   ├── user_id     STRING
│   │   ├── amount      DOUBLE
│   │   ├── status      STRING
│   │   └── created_at  TIMESTAMP
│   ├── Location: s3://my-bucket/processed/orders/
│   ├── InputFormat:  org.apache.hadoop.mapred.TextInputFormat
│   ├── OutputFormat: org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat
│   └── SerDeInfo (Serializer/Deserializer — Bộ Tuần Tự Hóa/Giải Tuần Tự Hóa):
│       └── SerializationLibrary: org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe
├── PartitionKeys (Khóa Phân Vùng):
│   ├── year   STRING
│   ├── month  STRING
│   └── day    STRING
├── TableType: EXTERNAL_TABLE
└── Parameters:
    ├── classification: parquet
    ├── compressionType: snappy
    └── EXTERNAL: TRUE
```

**SerDe — Serializer/Deserializer (Bộ Tuần Tự Hóa/Giải Tuần Tự Hóa):**

| Format         | SerDe Library                                              |
| --------------- | ----------------------------------------------------------- |
| JSON           | `org.openx.data.jsonserde.JsonSerDe`                       |
| CSV            | `org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe`       |
| Parquet        | `org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe` |
| ORC            | `org.apache.hadoop.hive.ql.io.orc.OrcSerde`               |
| Avro           | `org.apache.hadoop.hive.serde2.avro.AvroSerDe`            |

---

### 3. Partitions (Phân Vùng)

**Là gì:** Cách tổ chức dữ liệu theo một hoặc nhiều cột — cho phép query engine chỉ đọc phần dữ liệu cần thiết.

```
S3 Layout với Partitioning:
s3://my-bucket/events/
    year=2024/
        month=01/
            day=01/
                file1.parquet
                file2.parquet
            day=02/
                file1.parquet
        month=02/
            day=01/
                ...
    year=2025/
        ...
```

**Partition Pruning (Cắt Tỉa Phân Vùng):**

```sql
-- Query KHÔNG dùng partition → quét TOÀN BỘ dữ liệu (tốn tiền, chậm)
SELECT * FROM events WHERE DAY(event_time) = 15;

-- Query CÓ dùng partition → chỉ quét 1 ngày (nhanh, rẻ)
SELECT * FROM events WHERE year='2024' AND month='01' AND day='15';
```

**Lưu Ý Quan Trọng:** Partition metadata phải được đăng ký vào Glue Catalog, không chỉ tạo folder trên S3.

```sql
-- Đăng ký partition thủ công trong Athena
ALTER TABLE events ADD PARTITION (year='2024', month='01', day='15')
LOCATION 's3://my-bucket/events/year=2024/month=01/day=15/';

-- Hoặc dùng MSCK REPAIR TABLE để tự động tìm partitions mới
MSCK REPAIR TABLE events;
```

---

### 4. Connections (Kết Nối)

**Là gì:** Thông tin kết nối đến data source bên ngoài S3 — như JDBC/ODBC databases.

```
Connection: prod-mysql-connection
├── Connection Type: JDBC
├── JDBC URL: jdbc:mysql://prod-db.cluster-xyz.ap-southeast-1.rds.amazonaws.com:3306/ecommerce
├── Username: glue_user (lưu trong Secrets Manager)
├── Password: *** (lưu trong Secrets Manager)
└── VPC: vpc-0abc123 (kết nối qua VPC cho database nội bộ)
```

---

## 🔗 Tích Hợp Với Dịch Vụ AWS

### Với Amazon Athena

```sql
-- Athena tự động đọc schema từ Glue Catalog
-- Database "analytics" trong Glue → namespace trong Athena
SELECT user_id, COUNT(*) as event_count
FROM analytics.events_parquet
WHERE year = '2024' AND month = '01'
GROUP BY user_id;
-- Athena biết: dữ liệu ở s3://..., format Parquet, partition là year/month/day
```

### Với Amazon Redshift Spectrum

```sql
-- Tạo external schema từ Glue Catalog
CREATE EXTERNAL SCHEMA glue_schema
FROM DATA CATALOG
DATABASE 'analytics'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftGlueRole'
REGION 'ap-southeast-1';

-- Query S3 data qua Redshift Spectrum
SELECT * FROM glue_schema.events_parquet LIMIT 100;
```

### Với AWS EMR

```python
# Spark trên EMR đọc schema từ Glue Catalog
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .config("hive.metastore.client.factory.class",
            "com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory") \
    .enableHiveSupport() \
    .getOrCreate()

# Đọc bảng từ Glue Catalog
df = spark.sql("SELECT * FROM analytics.events_parquet WHERE year='2024'")
```

### Với AWS Glue ETL Jobs

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext

args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glueContext = GlueContext(sc)

# Đọc từ Glue Catalog bằng DynamicFrame (Khung Dữ Liệu Động)
datasource = glueContext.create_dynamic_frame.from_catalog(
    database="analytics_raw",
    table_name="events_json",
    transformation_ctx="datasource"
)

print(f"Tổng số records: {datasource.count()}")
print("Schema:")
datasource.printSchema()
```

---

## 🛠️ Quản Lý Data Catalog

### Tạo Table Thủ Công (DDL trong Athena)

```sql
CREATE EXTERNAL TABLE analytics.orders (
    order_id    STRING,
    user_id     STRING,
    product_id  STRING,
    quantity    INT,
    price       DOUBLE,
    status      STRING,
    created_at  TIMESTAMP
)
PARTITIONED BY (
    year  STRING,
    month STRING,
    day   STRING
)
STORED AS PARQUET
LOCATION 's3://my-data-lake/processed/orders/'
TBLPROPERTIES (
    'parquet.compression'='SNAPPY',
    'classification'='parquet'
);
```

### Quản Lý Qua AWS CLI (Command Line Interface — Giao Diện Dòng Lệnh)

```bash
# Liệt kê databases
aws glue get-databases --region ap-southeast-1

# Liệt kê tables trong database
aws glue get-tables --database-name analytics_raw

# Xem chi tiết một table
aws glue get-table --database-name analytics_raw --name events_json

# Xem danh sách partitions
aws glue get-partitions \
    --database-name analytics_raw \
    --table-name events_json \
    --expression "year='2024' and month='01'"
```

### Table Properties Quan Trọng

```python
# Cập nhật table properties để tối ưu Athena query
glue.update_table(
    DatabaseName='analytics_processed',
    TableInput={
        'Name': 'events_parquet',
        'Parameters': {
            'classification': 'parquet',
            'compressionType': 'snappy',
            # Partition projection — Athena tự tính partitions, không cần query Catalog
            'projection.enabled': 'true',
            'projection.year.type': 'integer',
            'projection.year.range': '2023,2026',
            'projection.month.type': 'integer',
            'projection.month.range': '1,12',
            'projection.month.digits': '2',
            'projection.day.type': 'integer',
            'projection.day.range': '1,31',
            'projection.day.digits': '2',
            'storage.location.template':
                's3://my-bucket/events/year=${year}/month=${month}/day=${day}/'
        }
    }
)
```

---

## 🔄 Schema Evolution (Tiến Hóa Schema)

**Schema Evolution** là khả năng xử lý khi cấu trúc dữ liệu thay đổi theo thời gian — thêm cột mới, đổi kiểu dữ liệu, xóa cột cũ.

### Các Loại Thay Đổi Schema

```
Thêm cột mới (Backward Compatible — Tương Thích Ngược):
  Trước: order_id, user_id, amount
  Sau:   order_id, user_id, amount, discount_code  ← thêm cột mới
  → Glue Crawler cập nhật table với cột mới
  → Query cũ vẫn chạy (cột mới = NULL cho dữ liệu cũ)

Đổi kiểu dữ liệu (Không Tương Thích):
  Trước: price DOUBLE
  Sau:   price STRING
  → Nguy hiểm! Query có thể lỗi
  → Cần xử lý cẩn thận

Xóa cột (Không Tương Thích):
  → Query dùng cột bị xóa sẽ lỗi
  → Cần migration plan
```

### Crawler Schema Change Policy (Chính Sách Thay Đổi Schema)

Khi chạy lại Crawler, có thể cấu hình:

| Chính Sách                  | Hành Động Khi Schema Thay Đổi                      |
| ---------------------------- | --------------------------------------------------- |
| **Update in database**      | Cập nhật table definition — thêm column mới        |
| **Mark as deprecated**      | Giữ table cũ, đánh dấu deprecated (Đã Lỗi Thời)   |
| **Create new table**         | Tạo table mới với timestamp — giữ cả hai           |

### Xử Lý Schema Evolution Trong ETL Job

```python
# DynamicFrame (Khung Dữ Liệu Động) xử lý schema linh hoạt hơn DataFrame
from awsglue.transforms import ResolveChoice, DropNullFields

# ResolveChoice — xử lý khi một cột có nhiều kiểu dữ liệu
# Ví dụ: price có thể là DOUBLE hoặc STRING trong các file khác nhau
resolved = ResolveChoice.apply(
    frame=datasource,
    choice="cast:double",      # Ép kiểu về DOUBLE
    specs=[("price", "cast:double")]
)

# DropNullFields — xóa các cột toàn NULL
cleaned = DropNullFields.apply(frame=resolved)
```

---

## 🔒 Bảo Mật và Kiểm Soát Truy Cập

### IAM Permissions (Quyền IAM — Identity and Access Management)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "glue:GetDatabase",
                "glue:GetDatabases",
                "glue:GetTable",
                "glue:GetTables",
                "glue:GetPartition",
                "glue:GetPartitions",
                "glue:BatchGetPartition"
            ],
            "Resource": [
                "arn:aws:glue:ap-southeast-1:123456789:catalog",
                "arn:aws:glue:ap-southeast-1:123456789:database/analytics_*",
                "arn:aws:glue:ap-southeast-1:123456789:table/analytics_*/*"
            ]
        }
    ]
}
```

### Lake Formation Integration (Tích Hợp Lake Formation)

**Khi bật Lake Formation:** Quyền truy cập được kiểm soát ở cấp độ column và row, không chỉ table.

```
Không có Lake Formation:
  IAM policy → Table access → Toàn bộ data

Có Lake Formation:
  IAM policy + Lake Formation permissions
  → Column-level: chỉ cho phép SELECT trên cột không nhạy cảm
  → Row-level: chỉ cho phép xem data của department mình quản lý
```

### Resource Policy (Chính Sách Tài Nguyên) — Cross-Account Access

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::987654321:root"
            },
            "Action": [
                "glue:GetDatabase",
                "glue:GetTable",
                "glue:GetPartitions"
            ],
            "Resource": [
                "arn:aws:glue:ap-southeast-1:123456789:catalog",
                "arn:aws:glue:ap-southeast-1:123456789:database/shared_analytics",
                "arn:aws:glue:ap-southeast-1:123456789:table/shared_analytics/*"
            ]
        }
    ]
}
```

---

## ✅ Thực Hành Tốt Nhất

### Naming Convention (Quy Ước Đặt Tên)

```
Database:
  {layer}_{domain}_{env}
  → raw_ecommerce_prod
  → processed_ecommerce_prod
  → aggregated_finance_dev

Table:
  {entity}_{format}_{version_nếu_cần}
  → orders_parquet
  → events_json_v2
  → daily_sales_summary_parquet

Column (snake_case):
  → order_id, user_id, created_at, total_amount
```

### Tagging Strategy (Chiến Lược Gắn Thẻ)

```python
# Gắn tags để quản lý và tính phí
glue.tag_resource(
    ResourceArn='arn:aws:glue:ap-southeast-1:123456789:database/raw_ecommerce_prod',
    TagsToAdd={
        'Environment': 'prod',
        'Team': 'data-engineering',
        'Domain': 'ecommerce',
        'DataLayer': 'raw',
        'CostCenter': 'DE-001'
    }
)
```

### Partition Strategy (Chiến Lược Phân Vùng)

```
Nguyên tắc:
1. Partition theo cột hay được dùng trong WHERE clause
2. Cardinality (Lực Lượng Phần Tử) vừa phải — không quá nhiều, không quá ít
3. Kích thước file mỗi partition: 128 MB – 1 GB (optimal cho Parquet)

Ví dụ tốt:
  PARTITIONED BY (year STRING, month STRING, day STRING)
  → ~365 partitions/năm, dễ query theo ngày

Tránh partition theo user_id nếu có hàng triệu users:
  → Hàng triệu partitions nhỏ → S3 ListObjects chậm → query chậm
```

---

## ❓ Câu Hỏi Phỏng Vấn

**Q: Tại sao Glue Data Catalog quan trọng trong data lake architecture?**
> Giải quyết vấn đề "data swamp" (đầm lầy dữ liệu) — khi data lake không có metadata thì không ai biết dữ liệu ở đâu, có nghĩa gì, dùng thế nào. Catalog cung cấp single source of truth (nguồn sự thật duy nhất) về schema, location, format — giúp Athena, Redshift, EMR query mà không cần biết chi tiết lưu trữ.

**Q: Partition trong Glue Catalog ảnh hưởng thế nào đến Athena performance?**
> Partition metadata trong Catalog giúp Athena biết chính xác folder nào cần đọc mà không cần list toàn bộ S3 prefix. Khi query `WHERE year='2024' AND month='01'`, Athena chỉ đọc 31 partitions thay vì toàn bộ dataset — giảm cả chi phí (tính phí theo TB quét) lẫn thời gian chạy.

**Q: Partition Projection là gì và khi nào dùng?**
> Partition Projection (Chiếu Phân Vùng) là tính năng Athena tự tính toán partition locations từ công thức, không cần query Glue Catalog cho từng partition. Hữu ích khi có hàng nghìn partitions (ví dụ: hourly partitions cho 3 năm = ~26,000 partitions) — giúp tránh bottleneck khi Athena phải liệt kê partitions từ Catalog.

**Q: Cross-account Glue Catalog sharing hoạt động thế nào?**
> Dùng Glue Resource Policy cho phép AWS account khác truy cập Catalog của mình. Kết hợp với S3 bucket policy để cho phép account đó đọc dữ liệu thực sự. Lake Formation cung cấp cách quản lý tinh tế hơn cho cross-account data sharing.

**Q: Sự khác biệt giữa Glue Table và Athena Table?**
> Không có sự khác biệt — khi tạo table trong Athena bằng DDL (ví dụ `CREATE EXTERNAL TABLE`), thực ra Athena đang tạo table trong Glue Data Catalog. Đây là cùng một table, hai giao diện khác nhau để quản lý.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**File:** `03-glue/1-glue-catalog.md`
**Xem Tiếp:** `2-glue-etl-jobs.md` — ETL Jobs và tối ưu chi phí DPU
