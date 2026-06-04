# Medallion Architecture — Kiến Trúc Huy Chương: Bronze, Silver, Gold

> Medallion Architecture (Kiến Trúc Huy Chương) là pattern tổ chức dữ liệu trong data lake thành các lớp chất lượng tăng dần — Bronze (Đồng/Thô), Silver (Bạc/Đã Làm Sạch) và Gold (Vàng/Đã Tổng Hợp). Được phổ biến bởi Databricks và áp dụng rộng rãi trên AWS. Đây là pattern đơn giản nhất, dễ implement nhất trong các kiến trúc dữ liệu hiện đại.

## 📚 Mục Lục

1. [Tổng Quan Medallion Architecture](#tổng-quan)
2. [Bronze Layer — Lớp Đồng (Dữ Liệu Thô)](#bronze-layer)
3. [Silver Layer — Lớp Bạc (Dữ Liệu Sạch)](#silver-layer)
4. [Gold Layer — Lớp Vàng (Dữ Liệu Tổng Hợp)](#gold-layer)
5. [Triển Khai trên AWS](#triển-khai-trên-aws)
6. [Medallion với Apache Iceberg và Delta Lake](#iceberg-delta)
7. [Ưu Điểm và Nhược Điểm](#ưu-điểm-và-nhược-điểm)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🏅 Tổng Quan Medallion Architecture

### Luồng Dữ Liệu

```
  NGUỒN DỮ LIỆU                                           TIÊU THỤ
  (Data Sources)                                         (Consumers)

  Databases        ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
  (MySQL, Oracle)  │             │    │             │    │             │
  → CDC Events ──→ │   BRONZE    │──→ │   SILVER    │──→ │    GOLD     │──→ BI Dashboards
                   │  (Thô/Raw) │    │  (Sạch)     │    │ (Tổng Hợp) │    (QuickSight)
  Files            │             │    │             │    │             │──→ Data Science
  (CSV, JSON, XML) │ Dữ liệu gốc │    │ Đã validate │    │ Đã aggregate│    (ML Models)
  → S3 Upload ──→  │ Bất biến    │    │ Có schema   │    │ Business    │──→ Ad-hoc SQL
                   │ Lưu mãi mãi │    │ chuẩn hóa   │    │ metrics     │    (Athena)
  Streams          │             │    │             │    │             │──→ APIs
  (Kinesis, Kafka) └─────────────┘    └─────────────┘    └─────────────┘
  → Real-time ──→
                       Glue ETL          Glue ETL
                    (Bronze→Silver)   (Silver→Gold)
```

### Triết Lý Cốt Lõi

**Câu hỏi:** Tại sao cần tới 3 lớp? Không thể đưa dữ liệu thẳng từ nguồn vào layer sẵn dùng?

**Trả lời — Ba lý do:**

1. **Dữ liệu gốc phải được bảo toàn (Bronze):** Khi phát hiện lỗi trong transformation, cần quay lại dữ liệu gốc để tính toán lại. Không có Bronze, không có khả năng reprocess.

2. **Không thể dùng dữ liệu thô trực tiếp (cần Silver):** Dữ liệu thực tế thường có lỗi — null values không mong muốn, format không nhất quán, trùng lặp. Silver layer chuẩn hóa dữ liệu.

3. **Không thể query dữ liệu chi tiết cho mọi use case (cần Gold):** Analyst không cần query toàn bộ transaction history mỗi lần. Gold layer tổng hợp sẵn các aggregates phổ biến.

---

## 🥉 Bronze Layer — Lớp Đồng (Dữ Liệu Thô)

### Đặc Điểm

| Đặc Điểm | Mô Tả |
|---------|-------|
| **Nguồn gốc** | Dữ liệu gốc từ hệ thống nguồn, không thay đổi |
| **Tính bất biến** | Append-only (chỉ thêm, không sửa, không xóa) |
| **Format** | Giữ nguyên format gốc hoặc convert sang Parquet/JSON |
| **Schema validation** | Tối thiểu hoặc không có |
| **Retention** | Lưu trữ lâu dài — thường là vĩnh viễn hoặc nhiều năm |
| **Truy cập** | Hạn chế — chỉ data engineers, không phải analysts |

### Cấu Trúc Thư Mục Bronze trên S3

```
s3://my-data-lake/bronze/
├── orders/                              # Từ hệ thống đơn hàng
│   ├── year=2024/month=01/day=15/       # Partitioned by ingestion time
│   │   ├── orders_20240115_001.parquet
│   │   └── orders_20240115_002.parquet
│   └── year=2024/month=01/day=16/
│       └── orders_20240116_001.parquet
├── customers/                           # Từ CRM (Customer Relationship Management)
│   ├── full_load/                       # Batch full dump (tải đầy đủ)
│   │   └── 2024-01-01/customers.parquet
│   └── cdc/                            # CDC (Change Data Capture — Bắt Thay Đổi Dữ Liệu)
│       ├── 2024-01-15/changes.parquet
│       └── 2024-01-16/changes.parquet
└── clickstream/                         # Từ web analytics
    ├── year=2024/month=01/day=15/hour=00/
    │   └── events_00001.parquet
    └── year=2024/month=01/day=15/hour=01/
        └── events_00001.parquet
```

### Best Practices cho Bronze Layer

```python
# AWS Glue Job: Ingest raw data vào Bronze Layer
import sys
from awsglue.context import GlueContext
from pyspark.context import SparkContext

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session

# Đọc từ source (ví dụ: RDS via JDBC)
source_df = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:mysql://rds-host/ecommerce") \
    .option("dbtable", "orders") \
    .option("user", "readonly_user") \
    .option("password", rds_password) \
    .load()

# Thêm metadata columns (không sửa dữ liệu gốc)
from pyspark.sql import functions as F
bronze_df = source_df \
    .withColumn("_ingestion_timestamp", F.current_timestamp()) \
    .withColumn("_source_system", F.lit("rds_ecommerce")) \
    .withColumn("_batch_id", F.lit("batch_20240115_001"))

# Ghi vào Bronze — partitioned by ingestion date
bronze_df.write \
    .mode("append") \
    .partitionBy("_ingestion_date") \
    .parquet("s3://my-data-lake/bronze/orders/")
```

---

## 🥈 Silver Layer — Lớp Bạc (Dữ Liệu Sạch)

### Đặc Điểm

| Đặc Điểm | Mô Tả |
|---------|-------|
| **Nguồn** | Đọc từ Bronze Layer |
| **Transformations** | Cleansing, deduplication (loại trùng lặp), type casting, normalization |
| **Schema** | Schema rõ ràng, chuẩn hóa, có Glue Catalog |
| **Data Quality** | Validated — có log lỗi cho records không hợp lệ |
| **Format** | Parquet hoặc ORC, partitioned theo business keys |
| **Truy cập** | Data engineers và advanced analysts |

### Transformations Điển Hình Bronze → Silver

```python
# AWS Glue Job: Bronze → Silver transformation

from pyspark.sql import functions as F
from pyspark.sql.types import *

# 1. Đọc từ Bronze
bronze_orders = spark.read.parquet("s3://my-data-lake/bronze/orders/")

# 2. CLEANSING (Làm Sạch) — xử lý null values
silver_orders = bronze_orders \
    .filter(F.col("order_id").isNotNull()) \
    .filter(F.col("customer_id").isNotNull()) \
    .filter(F.col("amount") > 0)

# 3. TYPE CASTING (Chuyển Kiểu) — chuẩn hóa kiểu dữ liệu
silver_orders = silver_orders \
    .withColumn("order_date",
        F.to_date(F.col("order_date_str"), "yyyy-MM-dd")) \
    .withColumn("amount",
        F.col("amount").cast(DoubleType())) \
    .withColumn("customer_id",
        F.col("customer_id").cast(LongType()))

# 4. DEDUPLICATION (Loại Trùng Lặp)
# Giữ record mới nhất nếu có nhiều bản cùng order_id
from pyspark.sql.window import Window
window_spec = Window.partitionBy("order_id").orderBy(F.col("_ingestion_timestamp").desc())
silver_orders = silver_orders \
    .withColumn("row_num", F.row_number().over(window_spec)) \
    .filter(F.col("row_num") == 1) \
    .drop("row_num")

# 5. NORMALIZATION (Chuẩn Hóa)
silver_orders = silver_orders \
    .withColumn("status", F.upper(F.trim(F.col("status")))) \
    .withColumn("country_code", F.upper(F.col("country_code")))

# 6. DERIVED FIELDS (Trường Phái Sinh) — tính toán thêm từ dữ liệu gốc
silver_orders = silver_orders \
    .withColumn("order_year", F.year(F.col("order_date"))) \
    .withColumn("order_month", F.month(F.col("order_date"))) \
    .withColumn("is_high_value", F.col("amount") > 1000)

# 7. DATA QUALITY LOG — ghi lại records không hợp lệ
invalid_records = bronze_orders \
    .filter(F.col("order_id").isNull() | (F.col("amount") <= 0))

invalid_records.write \
    .mode("append") \
    .parquet("s3://my-data-lake/quarantine/orders/")

# 8. Ghi vào Silver — partitioned theo business key
silver_orders.write \
    .mode("overwrite") \
    .partitionBy("order_year", "order_month") \
    .parquet("s3://my-data-lake/silver/orders/")

print(f"Bronze records: {bronze_orders.count()}")
print(f"Silver records: {silver_orders.count()}")
print(f"Quarantined: {invalid_records.count()}")
```

---

## 🥇 Gold Layer — Lớp Vàng (Dữ Liệu Tổng Hợp)

### Đặc Điểm

| Đặc Điểm | Mô Tả |
|---------|-------|
| **Nguồn** | Đọc từ Silver Layer (đôi khi join nhiều Silver tables) |
| **Transformations** | Aggregations (tổng hợp), joins, business metrics |
| **Schema** | Tối ưu cho consumption — wide tables hoặc star schema |
| **Format** | Parquet, thường với nhiều aggregations pre-computed |
| **Truy cập** | Tất cả: analysts, BI tools, data scientists, APIs |
| **Update pattern** | Full refresh hoặc incremental (tùy kích thước) |

### Ví Dụ Gold Layer Tables

```python
# AWS Glue Job: Silver → Gold — tạo business metrics

# ===== GOLD TABLE 1: Daily Revenue Summary =====
# Tổng hợp doanh thu hàng ngày theo category

daily_revenue = spark.sql("""
    SELECT
        o.order_date,
        p.category_name,
        p.brand,
        COUNT(DISTINCT o.order_id)    AS total_orders,
        COUNT(DISTINCT o.customer_id) AS unique_customers,
        SUM(oi.quantity)              AS total_units_sold,
        SUM(oi.quantity * oi.unit_price) AS gross_revenue,
        SUM(oi.quantity * oi.unit_price * (1 - oi.discount_pct))
                                      AS net_revenue,
        AVG(o.amount)                 AS avg_order_value
    FROM silver.orders o
    JOIN silver.order_items oi ON o.order_id = oi.order_id
    JOIN silver.products p    ON oi.product_id = p.product_id
    WHERE o.status != 'CANCELLED'
    GROUP BY o.order_date, p.category_name, p.brand
""")

daily_revenue.write \
    .mode("overwrite") \
    .partitionBy("order_date") \
    .saveAsTable("gold.daily_revenue_by_category")

# ===== GOLD TABLE 2: Customer 360 =====
# Toàn bộ thông tin khách hàng trong một wide table

customer_360 = spark.sql("""
    SELECT
        c.customer_id,
        c.customer_name,
        c.email,
        c.country_code,
        c.registration_date,
        DATEDIFF(CURRENT_DATE, c.registration_date) AS days_since_registration,

        -- Order metrics
        COUNT(o.order_id)          AS lifetime_orders,
        SUM(o.amount)              AS lifetime_value,
        AVG(o.amount)              AS avg_order_value,
        MAX(o.order_date)          AS last_order_date,
        DATEDIFF(CURRENT_DATE, MAX(o.order_date))
                                   AS days_since_last_order,

        -- RFM Score (Recency, Frequency, Monetary — Gần Đây, Tần Suất, Giá Trị)
        CASE
            WHEN DATEDIFF(CURRENT_DATE, MAX(o.order_date)) <= 30  THEN 'Active'
            WHEN DATEDIFF(CURRENT_DATE, MAX(o.order_date)) <= 90  THEN 'At Risk'
            WHEN DATEDIFF(CURRENT_DATE, MAX(o.order_date)) <= 180 THEN 'Lapsed'
            ELSE 'Churned'
        END AS customer_segment
    FROM silver.customers c
    LEFT JOIN silver.orders o ON c.customer_id = o.customer_id
    GROUP BY c.customer_id, c.customer_name, c.email,
             c.country_code, c.registration_date
""")

customer_360.write \
    .mode("overwrite") \
    .saveAsTable("gold.customer_360")
```

---

## ☁️ Triển Khai trên AWS

### Kiến Trúc Hoàn Chỉnh

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    MEDALLION ARCHITECTURE TRÊN AWS                        │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  INGESTION         BRONZE              SILVER              GOLD            │
│  ─────────         ──────              ──────              ────            │
│                                                                            │
│  RDS/Aurora  ──→   S3 (raw)    ──→    S3 (cleaned) ──→   S3 (aggregated)  │
│  Kinesis     ──→   Parquet/JSON        Parquet             Parquet         │
│  DMS (CDC)   ──→   No schema           Glue Catalog        Glue Catalog    │
│                    check               + validation        + indexes       │
│                                                                            │
│                     Glue Crawler        Glue ETL            Glue ETL       │
│                     (auto-discover)     (Bronze→Silver)     (Silver→Gold)  │
│                                                                            │
│  SERVING LAYER:                                                            │
│  ─────────────                                                             │
│  Amazon Athena ──────────────────────── Query Bronze/Silver/Gold           │
│  Redshift Spectrum ─────────────────── Query Gold (complex analytics)      │
│  AWS Glue DataBrew ─────────────────── Visual data prep                    │
│  Amazon QuickSight ─────────────────── BI Dashboards từ Gold               │
│                                                                            │
│  ORCHESTRATION (Điều Phối):                                                │
│  ─────────────                                                             │
│  AWS Glue Workflows ─────── Batch pipeline orchestration                   │
│  Amazon EventBridge ──────── Trigger pipeline khi có dữ liệu mới           │
│  Amazon MWAA (Airflow) ────── Complex workflow management                  │
│                                                                            │
└──────────────────────────────────────────────────────────────────────────┘
```

### Glue Catalog Setup

```python
# Tạo databases trong Glue Catalog cho từng layer
import boto3

glue = boto3.client('glue', region_name='us-east-1')

# Tạo database cho từng layer
for layer in ['bronze', 'silver', 'gold', 'quarantine']:
    glue.create_database(
        DatabaseInput={
            'Name': f'data_lake_{layer}',
            'Description': f'Medallion Architecture — {layer.capitalize()} Layer',
            'Parameters': {
                'layer': layer,
                'created_by': 'data-platform-team'
            }
        }
    )

# Ví dụ tạo table cho Silver layer
glue.create_table(
    DatabaseName='data_lake_silver',
    TableInput={
        'Name': 'orders',
        'Description': 'Cleaned and validated orders from Bronze layer',
        'Parameters': {
            'classification': 'parquet',
            'source_layer': 'bronze',
            'sla_freshness': 'hourly'
        },
        'StorageDescriptor': {
            'Location': 's3://my-data-lake/silver/orders/',
            'InputFormat': 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat',
            'OutputFormat': 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat',
            'SerdeInfo': {
                'SerializationLibrary': 'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'
            },
            'Columns': [
                {'Name': 'order_id', 'Type': 'string', 'Comment': 'Unique order identifier'},
                {'Name': 'customer_id', 'Type': 'bigint'},
                {'Name': 'order_date', 'Type': 'date'},
                {'Name': 'amount', 'Type': 'double'},
                {'Name': 'status', 'Type': 'string'},
                {'Name': 'country_code', 'Type': 'string'},
                {'Name': 'is_high_value', 'Type': 'boolean'}
            ]
        },
        'PartitionKeys': [
            {'Name': 'order_year', 'Type': 'int'},
            {'Name': 'order_month', 'Type': 'int'}
        ]
    }
)
```

### EventBridge Trigger — Tự Động Kích Hoạt Pipeline

```json
{
  "Comment": "Trigger Bronze→Silver khi S3 có file mới",
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {
      "name": ["my-data-lake"]
    },
    "object": {
      "key": [{
        "prefix": "bronze/orders/"
      }]
    }
  },
  "Targets": [{
    "Id": "TriggerGlueJobBronzeToSilver",
    "Arn": "arn:aws:glue:us-east-1:123:job/bronze-to-silver-orders",
    "RoleArn": "arn:aws:iam::123:role/EventBridgeGlueRole"
  }]
}
```

---

## 🧊 Medallion với Apache Iceberg và Delta Lake

### Tại Sao Dùng Table Formats Hiện Đại?

Parquet thuần trên S3 có giới hạn:
- Không hỗ trợ **ACID transactions** (giao dịch nguyên tử) — update/delete phức tạp
- Không có **schema evolution** (phát triển schema) an toàn
- Không có **time travel** (du hành thời gian — query dữ liệu tại thời điểm cụ thể)

Apache Iceberg và Delta Lake giải quyết những vấn đề này.

### Apache Iceberg trên AWS

```python
# Glue Job với Iceberg format — cho Silver và Gold layers
spark.conf.set("spark.sql.extensions",
               "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions")
spark.conf.set("spark.sql.catalog.glue_catalog",
               "org.apache.iceberg.spark.SparkCatalog")
spark.conf.set("spark.sql.catalog.glue_catalog.warehouse",
               "s3://my-data-lake/")
spark.conf.set("spark.sql.catalog.glue_catalog.catalog-impl",
               "org.apache.iceberg.aws.glue.GlueCatalog")

# Tạo Iceberg table cho Silver layer
spark.sql("""
    CREATE TABLE IF NOT EXISTS glue_catalog.data_lake_silver.orders (
        order_id     STRING,
        customer_id  BIGINT,
        order_date   DATE,
        amount       DOUBLE,
        status       STRING
    )
    USING iceberg
    PARTITIONED BY (months(order_date))
    LOCATION 's3://my-data-lake/silver/orders/'
""")

# Upsert (MERGE) — không thể làm với plain Parquet
spark.sql("""
    MERGE INTO glue_catalog.data_lake_silver.orders AS target
    USING incoming_orders AS source
    ON target.order_id = source.order_id
    WHEN MATCHED THEN
        UPDATE SET target.status = source.status,
                   target.amount = source.amount
    WHEN NOT MATCHED THEN
        INSERT *
""")

# Time travel — query dữ liệu tại thời điểm cụ thể
spark.sql("""
    SELECT * FROM glue_catalog.data_lake_silver.orders
    FOR SYSTEM_TIME AS OF '2024-01-15 10:00:00'
""")

# Schema evolution — thêm column mà không cần rewrite toàn bộ data
spark.sql("""
    ALTER TABLE glue_catalog.data_lake_silver.orders
    ADD COLUMN is_international BOOLEAN
""")
```

### So Sánh Iceberg vs Delta Lake vs Plain Parquet

| Tính Năng | Plain Parquet | Apache Iceberg | Delta Lake |
|-----------|---------------|----------------|------------|
| **ACID transactions** | ❌ | ✅ | ✅ |
| **Time travel** | ❌ | ✅ | ✅ |
| **Schema evolution** | Hạn chế | ✅ | ✅ |
| **MERGE/UPSERT** | ❌ | ✅ | ✅ |
| **Partition evolution** | ❌ | ✅ | Hạn chế |
| **AWS Glue native** | ✅ | ✅ (từ Glue 3.0) | ✅ (từ Glue 3.0) |
| **Athena support** | ✅ | ✅ | ✅ |
| **EMR support** | ✅ | ✅ | ✅ |
| **Vendor** | Open standard | Open standard | Databricks + open |

---

## ⚖️ Ưu Điểm và Nhược Điểm

### ✅ Ưu Điểm

| Ưu Điểm | Giải Thích |
|---------|-----------|
| **Đơn giản, dễ hiểu** | Ba lớp Bronze/Silver/Gold rõ ràng, dễ onboard team mới |
| **Dữ liệu gốc được bảo toàn** | Bronze layer là safety net — luôn có thể reprocess |
| **Audit trail** (Dấu Vết Kiểm Toán) | Biết dữ liệu đến từ đâu, qua những bước transform nào |
| **Phân tách concerns** | Data Quality ở Silver, Business Logic ở Gold |
| **Linh hoạt** | Có thể thêm lớp intermediate nếu cần (Bronze → Bronze+ → Silver) |
| **Chi phí tối ưu** | Bronze dùng S3 rẻ; Gold dùng Redshift/Athena cho query nhanh |

### ❌ Nhược Điểm

| Nhược Điểm | Giải Thích |
|-----------|-----------|
| **Latency cao** | Batch pipeline Bronze→Silver→Gold có thể mất hàng giờ |
| **Storage duplication** | Cùng dữ liệu tồn tại ở 3 lớp — tốn S3 storage |
| **Pipeline management** | Cần orchestrate và monitor nhiều Glue jobs |
| **Gold table explosion** | Nếu không kiểm soát, số lượng Gold tables tăng nhanh |
| **Not real-time** | Pattern thuần batch — cần kết hợp với Kinesis nếu cần real-time |

---

## 🎓 Câu Hỏi Phỏng Vấn

### Câu 1: "Giải thích Medallion Architecture. Tại sao cần 3 lớp thay vì 2?"

**Gợi ý trả lời:**

Medallion Architecture tổ chức data lake thành ba lớp chất lượng tăng dần:

- **Bronze (Thô):** Dữ liệu gốc bất biến — không transform, không xóa. Đây là nguồn sự thật cuối cùng cho reprocessing.

- **Silver (Sạch):** Dữ liệu đã validate, deduplicate, standardize. Sẵn sàng cho analytics nhưng chưa tổng hợp — vẫn ở mức granular (chi tiết).

- **Gold (Tổng Hợp):** Aggregated, business metrics sẵn sàng cho BI tools, ML models, và APIs.

Tại sao cần cả 3? Vì 2 lớp không đủ:
- Bronze + Gold (bỏ Silver): Khi Gold job lỗi, không có intermediate đã validate để debug. Business logic và data quality lẫn lộn.
- Silver + Gold (bỏ Bronze): Mất khả năng reprocess khi phát hiện lỗi trong Silver transformation.
- Bronze + Silver (bỏ Gold): Analyst phải viết aggregation queries mỗi lần → chậm, không nhất quán.

---

### Câu 2: "Làm thế nào để implement incremental processing (xử lý tăng dần) trong Medallion?"

**Gợi ý trả lời:**

Thay vì full refresh (làm mới toàn bộ) mỗi lần, incremental processing chỉ xử lý dữ liệu mới:

**Approach 1 — Watermark (Dấu Nước) based:**
```python
# Đọc high watermark từ lần chạy trước
last_processed = get_last_watermark("bronze_to_silver_orders")

# Chỉ xử lý records mới hơn watermark
new_data = spark.read.parquet("s3://bronze/orders/") \
    .filter(F.col("_ingestion_timestamp") > last_processed)

# Process và ghi vào Silver
process_and_write(new_data, "s3://silver/orders/")

# Cập nhật watermark
update_watermark("bronze_to_silver_orders", current_time)
```

**Approach 2 — Apache Iceberg MERGE:**
```sql
-- Upsert với Iceberg — efficient incremental load
MERGE INTO silver.orders AS target
USING new_bronze_data AS source ON target.order_id = source.order_id
WHEN MATCHED AND source.updated_at > target.updated_at
    THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *
```

**Approach 3 — S3 partition-based:**
Nếu data partitioned by date, chỉ re-process partitions có dữ liệu mới.

---

### Câu 3: "So sánh Medallion Architecture với Lambda Architecture?"

**Gợi ý trả lời:**

| | Medallion | Lambda |
|--|-----------|--------|
| **Focus** | Data quality layers | Batch + Stream processing |
| **Complexity** | Thấp | Cao |
| **Real-time** | Không (batch) | Có (Speed Layer) |
| **Use case** | Data lake organization | Yêu cầu cả batch lẫn real-time |

Medallion và Lambda không loại trừ nhau — có thể kết hợp:
- Dùng Medallion để tổ chức data lake (Bronze/Silver/Gold)
- Áp dụng Lambda principle để có cả batch accuracy và real-time freshness

Ví dụ: Gold layer có thể được feed từ cả Glue batch job (accurate, hourly) lẫn Kinesis streaming job (near-real-time, vài phút) — đây là Medallion + Lambda kết hợp.

---

## 📊 Tổng Kết

```
Medallion Architecture = Bronze (Thô) + Silver (Sạch) + Gold (Tổng Hợp)

Bronze: Lưu dữ liệu gốc bất biến — safety net cho reprocessing
Silver: Validate, deduplicate, standardize — trusted, granular data
Gold: Aggregate, business metrics — sẵn sàng cho BI/ML/API

Trên AWS:
  → S3 cho tất cả 3 lớp (storage rẻ)
  → AWS Glue ETL cho transformations
  → AWS Glue Data Catalog cho metadata management
  → Amazon Athena cho ad-hoc queries
  → Amazon QuickSight kết nối với Gold layer
  → Apache Iceberg cho ACID + time travel khi cần

Pattern phổ biến nhất để bắt đầu:
  S3 (Bronze) → Glue ETL → S3 (Silver) → Glue ETL → S3 (Gold) → Athena/QuickSight
```

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Trạng Thái:** ✅ Hoàn thành
**Module Hoàn Thành:** [README.md](./README.md) | [1-lambda-architecture.md](./1-lambda-architecture.md) | [2-kappa-architecture.md](./2-kappa-architecture.md) | [3-data-mesh.md](./3-data-mesh.md) | **4-medallion-architecture.md** ✅
