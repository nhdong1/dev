# 4 — Data Lake vs Data Warehouse vs Data Lakehouse

> Ba kiến trúc lưu trữ dữ liệu quan trọng nhất trong analytics hiện đại. Hiểu rõ sự khác biệt và trade-off giúp bạn thiết kế storage strategy đúng cho từng use case và trả lời tự tin trong phỏng vấn.

## 📚 Mục Lục

1. [Tổng Quan Ba Kiến Trúc](#tổng-quan-ba-kiến-trúc)
2. [Data Lake — Hồ Dữ Liệu](#data-lake--hồ-dữ-liệu)
3. [Data Warehouse — Kho Dữ Liệu](#data-warehouse--kho-dữ-liệu)
4. [Data Lakehouse — Kết Hợp](#data-lakehouse--kết-hợp)
5. [So Sánh Chi Tiết](#so-sánh-chi-tiết)
6. [Quyết Định Kiến Trúc](#quyết-định-kiến-trúc)
7. [AWS Implementation](#aws-implementation)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Ba Kiến Trúc

```
┌─────────────────────────────────────────────────────────────────────┐
│                 EVOLUTION CỦA DATA STORAGE                         │
├─────────────────┬────────────────────┬────────────────────────────┤
│   DATA LAKE     │  DATA WAREHOUSE    │     DATA LAKEHOUSE         │
│   ~2010         │   ~1990s-2000s     │     ~2020+                 │
│                 │                    │                            │
│  "Lưu tất cả   │  "Tổ chức tốt,     │  "Tốt nhất của cả hai     │
│  trước, nghĩ   │  query nhanh,       │  thế giới — linh hoạt      │
│  sau"           │  reliable"          │  VÀ có cấu trúc"          │
│                 │                    │                            │
│  Amazon S3 +    │  Amazon Redshift   │  S3 + Delta Lake/Iceberg   │
│  Lake Formation │                    │  + Athena/Glue             │
└─────────────────┴────────────────────┴────────────────────────────┘
```

---

## Data Lake — Hồ Dữ Liệu

### Định Nghĩa
**Data Lake** (Hồ Dữ Liệu) là kho lưu trữ tập trung có thể chứa **mọi loại dữ liệu** ở **định dạng gốc** (raw format), bất kể cấu trúc, ở **quy mô bất kỳ**.

### Đặc Điểm Cốt Lõi

#### Schema-on-Read (Schema Khi Đọc)
```
Traditional DB (Schema-on-Write — Schema Khi Ghi):
→ Phải định nghĩa schema TRƯỚC khi ghi dữ liệu
→ INSERT INTO users (id, name, email) VALUES (...)
   ↑ Schema đã cố định

Data Lake (Schema-on-Read):
→ Ghi dữ liệu RAW không cần schema
→ Đọc và áp dụng schema khi query
→ SELECT * FROM s3.'s3://my-lake/users/' ← Athena xác định schema khi đọc
```

#### Kiến Trúc Zone (Vùng Dữ Liệu)
```
Data Lake Zones:

/raw/          ← Dữ liệu gốc, không biến đổi, immutable
  ├── orders/2026/05/17/orders_001.json
  ├── logs/2026/05/17/access.log.gz
  └── iot/sensor_data_*.avro

/processed/    ← Đã làm sạch, chuẩn hóa, Parquet format
  ├── orders/year=2026/month=05/day=17/part-0000.parquet
  └── users/country=VN/part-0000.parquet

/curated/      ← Business-ready, aggregated
  ├── daily_revenue/year=2026/month=05/
  └── user_segments/
```

### Use Cases Phù Hợp Với Data Lake

| Use Case                          | Lý Do Data Lake Phù Hợp                            |
| --------------------------------- | --------------------------------------------------- |
| ML/AI training data               | Cần raw data, schema linh hoạt                     |
| Log archival & compliance         | Volume lớn, ít query, cost-effective trên S3        |
| Data exploration & discovery      | Không biết trước schema và query patterns           |
| Multi-format data (JSON, XML, CSV)| Không cần định nghĩa schema trước                  |
| IoT data ingestion                | Volume cực lớn, semi-structured                    |
| Data science experimentation      | Linh hoạt, dễ thử nghiệm                           |

### Lợi Thế
- **Chi phí thấp:** S3 ~$0.023/GB/tháng vs Redshift ~$0.25/GB/tháng
- **Linh hoạt:** Bất kỳ format, schema nào
- **Scale vô hạn:** Exabytes trên S3
- **Mọi loại dữ liệu:** Structured, semi-structured, unstructured
- **Dùng cho nhiều mục đích:** Analytics, ML, archiving

### Hạn Chế
- **Query chậm hơn** nếu không tổ chức tốt (full scan thay vì index lookup)
- **Governance phức tạp** — dễ thành "Data Swamp" (Đầm Lầy Dữ Liệu) nếu không quản lý
- **Không có ACID transactions** — không thể UPDATE/DELETE dễ dàng
- **Cần expertise** — Data Engineers biết tối ưu partitioning, formats
- **Không phù hợp OLTP** — không phải database cho ứng dụng

> ⚠️ **Data Swamp** (Đầm Lầy Dữ Liệu): Khi Data Lake không có governance tốt → dữ liệu không có metadata, không ai biết dữ liệu nào có ý nghĩa gì, không ai dùng được → lãng phí hoàn toàn.

---

## Data Warehouse — Kho Dữ Liệu

### Định Nghĩa
**Data Warehouse** (Kho Dữ Liệu) là hệ thống lưu trữ dữ liệu **có cấu trúc**, được **tối ưu cho analytical queries** (OLAP — Online Analytical Processing), thường là **schema-on-write** và dữ liệu đã qua **ETL** (Extract Transform Load).

### Đặc Điểm Kỹ Thuật

#### OLAP vs OLTP (Phân Tích vs Giao Dịch)
```
OLTP (Online Transaction Processing — Xử Lý Giao Dịch Trực Tuyến):
- Ứng dụng production (web app, mobile app)
- INSERT/UPDATE/DELETE nhiều, nhỏ
- Response time < 10ms
- Ví dụ: RDS, DynamoDB, Aurora

OLAP (Online Analytical Processing — Xử Lý Phân Tích Trực Tuyến):
- Data Warehouse cho analytics
- SELECT với aggregations lớn, complex joins
- Query trên hàng triệu-tỷ rows
- Ví dụ: Redshift, BigQuery, Snowflake
```

#### Columnar Storage (Lưu Trữ Cột)
```
Row-based storage (cho OLTP):
Row 1: [id=1, name="Alice", age=25, salary=1000]
Row 2: [id=2, name="Bob",   age=30, salary=2000]

Columnar storage (cho OLAP — Data Warehouse):
id column:     [1, 2, 3, 4, ...]
name column:   ["Alice", "Bob", "Charlie", ...]
age column:    [25, 30, 28, ...]
salary column: [1000, 2000, 1500, ...]

Query "SELECT AVG(salary)" → chỉ đọc salary column → 100x nhanh hơn
```

#### Star Schema (Lược Đồ Ngôi Sao)
```
Fact Table (Bảng Sự Kiện — trung tâm):
orders_fact
├── order_id (PK)
├── customer_id (FK) ──→ customers_dim
├── product_id (FK)  ──→ products_dim
├── date_id (FK)     ──→ date_dim
├── amount
└── quantity

Dimension Tables (Bảng Chiều — xung quanh):
customers_dim: customer details
products_dim:  product details
date_dim:      year, month, day, quarter, weekday
```

### Redshift — AWS Data Warehouse

```
Architecture (Kiến Trúc):

┌──────────────────────────────────────────┐
│            LEADER NODE                   │
│  - Query planning & coordination         │
│  - Client connections (JDBC/ODBC)        │
└──────────────────┬───────────────────────┘
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ COMPUTE  │ │ COMPUTE  │ │ COMPUTE  │
│  NODE 1  │ │  NODE 2  │ │  NODE 3  │
│ (data +  │ │ (data +  │ │ (data +  │
│ compute) │ │ compute) │ │ compute) │
└──────────┘ └──────────┘ └──────────┘

MPP (Massively Parallel Processing — Xử Lý Song Song Đại Trà):
Query được chia nhỏ và chạy song song trên tất cả compute nodes
```

### Use Cases Phù Hợp Với Data Warehouse

| Use Case                          | Lý Do DW Phù Hợp                                   |
| --------------------------------- | --------------------------------------------------- |
| BI dashboards & reports           | Query nhanh, schema cố định, optimized             |
| Complex OLAP queries              | MPP engine, columnar storage                       |
| Historical reporting              | Lưu trữ structured, query predictable              |
| Business intelligence             | Tích hợp tốt với BI tools (Tableau, Power BI)      |
| Regulated industries              | ACID compliance, audit trail                       |
| Multi-join analytical queries     | Optimized query planner, statistics                |

### Lợi Thế
- **Query performance cao** — columnar storage, MPP, query optimization
- **Dễ dùng cho business users** — SQL familiar, BI tools integration
- **Reliable** — ACID compliance, data consistency
- **Governance tốt** — Schema enforcement, access control

### Hạn Chế
- **Chi phí cao hơn** — ~$0.25/GB/tháng vs S3 $0.023/GB
- **Chỉ structured data** — không xử lý tốt JSON, image, video
- **Schema rigid** — thay đổi schema tốn thời gian (ALTER TABLE)
- **Limited scale** — petabyte scale tốn kém
- **Không linh hoạt** — phải biết query patterns trước để tối ưu

---

## Data Lakehouse — Kết Hợp

### Định Nghĩa
**Data Lakehouse** (Kho Hồ Dữ Liệu) là kiến trúc kết hợp **tính linh hoạt và chi phí thấp của Data Lake** với **quản lý dữ liệu và hiệu suất query của Data Warehouse**.

### Các Công Nghệ Cốt Lõi

#### Open Table Formats (Định Dạng Bảng Mở)
Là lớp metadata trên S3 cung cấp ACID transactions, schema evolution, time travel:

| Format           | Nhà Phát Triển | Đặc Điểm Nổi Bật                               |
| ---------------- | -------------- | ----------------------------------------------- |
| **Delta Lake**   | Databricks     | ACID, time travel, MERGE, Z-ordering           |
| **Apache Iceberg** | Netflix/Apple | ACID, partition evolution, hidden partitioning |
| **Apache Hudi**  | Uber           | Upserts, incremental pulls, near real-time     |

#### Tính Năng Lakehouse

```
Data Lake (S3) + Open Table Format = Data Lakehouse

Bổ sung vào S3:
├── ACID Transactions (Giao Dịch ACID)
│   UPDATE user SET email = 'new@email.com' WHERE id = 123
│   (trước đây phải rewrite toàn bộ file Parquet)
│
├── Time Travel (Du Hành Thời Gian)
│   SELECT * FROM orders TIMESTAMP AS OF '2026-01-01'
│   (query dữ liệu tại một thời điểm trong quá khứ)
│
├── Schema Evolution (Tiến Hóa Schema)
│   ALTER TABLE orders ADD COLUMN discount DECIMAL(10,2)
│   (thêm cột mà không cần rewrite data)
│
└── Upserts (Cập Nhật và Chèn)
    MERGE INTO orders USING updates ON orders.id = updates.id
    WHEN MATCHED THEN UPDATE ...
    WHEN NOT MATCHED THEN INSERT ...
```

### Architecture Lakehouse Trên AWS

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS DATA LAKEHOUSE                           │
│                                                                 │
│  Sources → Kinesis/Glue → S3 (Parquet + Iceberg metadata)      │
│                                    │                           │
│                    ┌───────────────┼────────────────┐          │
│                    │               │                │          │
│               Athena           AWS Glue         Amazon EMR    │
│            (ad-hoc SQL)     (ETL jobs)      (Spark workloads) │
│                    │               │                │          │
│                    └───────────────┼────────────────┘          │
│                                    │                           │
│                              QuickSight                        │
│                            (BI & Reports)                      │
└─────────────────────────────────────────────────────────────────┘

S3 giữ role là storage layer, Iceberg/Delta là transaction layer
```

### Use Cases Phù Hợp Với Lakehouse

| Use Case                          | Lý Do Lakehouse Phù Hợp                            |
| --------------------------------- | --------------------------------------------------- |
| Upserts và deletes lên dữ liệu S3 | ACID transactions qua Iceberg/Delta                |
| ML + BI trên cùng data            | Single copy of data cho nhiều workloads            |
| GDPR right-to-be-forgotten        | DELETE specific records trong Data Lake            |
| Slowly Changing Dimensions (SCD)  | MERGE/UPDATE operations                            |
| Streaming + Batch cùng table      | Unified table cho cả hai                          |

---

## So Sánh Chi Tiết

| Tiêu Chí                  | Data Lake            | Data Warehouse       | Data Lakehouse       |
| ------------------------- | -------------------- | -------------------- | -------------------- |
| **Storage cost**          | Thấp (~$0.023/GB)    | Cao (~$0.25/GB)      | Thấp (S3-based)      |
| **Query performance**     | Trung bình           | Cao                  | Cao (với tuning)     |
| **Data types**            | Mọi loại             | Chỉ structured       | Mọi loại             |
| **Schema**                | Schema-on-read       | Schema-on-write      | Cả hai               |
| **ACID transactions**     | Không                | Có                   | Có (Iceberg/Delta)   |
| **Time travel**           | Không                | Hạn chế              | Có                   |
| **Governance**            | Phức tạp             | Tốt                  | Tốt (với LF)         |
| **ML support**            | Rất tốt              | Hạn chế              | Rất tốt              |
| **BI support**            | Cần ETL thêm         | Native               | Tốt                  |
| **Maturity**              | Mature               | Mature               | Đang phát triển      |
| **AWS service**           | S3 + Lake Formation  | Amazon Redshift      | S3 + Iceberg/Glue    |

---

## Quyết Định Kiến Trúc

### Decision Tree (Cây Quyết Định)

```
Câu hỏi 1: Dữ liệu có structured hoàn toàn không?
  ├──→ Không (JSON, log, image, mixed) → DATA LAKE hoặc LAKEHOUSE
  └──→ Có → Tiếp tục...

Câu hỏi 2: Có cần performance cao cho BI/reporting?
  ├──→ Có, query < 1 giây → DATA WAREHOUSE (Redshift)
  └──→ Chấp nhận vài giây → LAKE hoặc LAKEHOUSE đủ

Câu hỏi 3: Có cần ML/data science workloads không?
  ├──→ Có → DATA LAKE hoặc LAKEHOUSE (không muốn copy data)
  └──→ Không → DATA WAREHOUSE đơn giản hơn

Câu hỏi 4: Budget?
  ├──→ Nhỏ → DATA LAKE (S3 rẻ) + Athena (serverless)
  └──→ Đủ lớn → DATA WAREHOUSE hoặc LAKEHOUSE

Câu hỏi 5: Team có nhiều Data Engineers hay ít?
  ├──→ Ít → DATA WAREHOUSE (đơn giản hơn, ít tùy chỉnh)
  └──→ Có team mạnh → LAKEHOUSE (maximize value)
```

### Kịch Bản Thực Tế

| Công Ty          | Situation                                 | Khuyến Nghị           | Lý Do                                      |
| ---------------- | ----------------------------------------- | --------------------- | ------------------------------------------ |
| Startup Stage 1  | 5 người, data < 100GB, cần BI dashboard   | **Data Warehouse**    | Đơn giản, Redshift Serverless rẻ, nhanh setup |
| Mid-size SaaS    | Data Engineer team 3 người, mixed data    | **Data Lake** + Athena | Rẻ, linh hoạt, dùng được ngay             |
| Enterprise       | ML team + BI team + big data              | **Lakehouse**         | Unified platform cho mọi use case          |
| Regulated Bank   | Compliance, audit trails, structured data | **Data Warehouse**    | ACID, governance, schema control          |
| IoT Company      | PB of sensor data, varied ML workloads    | **Data Lake**         | Volume cực lớn, nhiều formats, ML          |

---

## AWS Implementation

### AWS Data Lake Stack

```bash
# 1. S3 là foundation
aws s3 mb s3://my-data-lake-prod

# 2. Lake Formation quản lý permissions
# → Column-level, row-level security
# → Tag-based access control (TBAC)

# 3. Glue Crawler tự động catalog
# → Phát hiện schema
# → Cập nhật Glue Data Catalog

# 4. Athena để query
SELECT date, SUM(revenue)
FROM "my_catalog"."processed"."orders"
WHERE year = '2026' AND month = '05'
GROUP BY date
-- Chỉ đọc relevant partitions, không scan toàn bộ S3
```

### AWS Data Warehouse Stack

```sql
-- Amazon Redshift
-- 1. Create table với distribution strategy
CREATE TABLE orders (
    order_id    BIGINT NOT NULL,
    customer_id BIGINT NOT NULL,
    amount      DECIMAL(10,2),
    order_date  DATE
)
DISTKEY(customer_id)   -- Phân phối data theo customer_id
SORTKEY(order_date);   -- Sort data theo ngày để range queries nhanh

-- 2. Load data từ S3 với COPY
COPY orders
FROM 's3://my-bucket/orders/'
IAM_ROLE 'arn:aws:iam::123:role/RedshiftRole'
FORMAT AS PARQUET;

-- 3. Query analytics nhanh
SELECT customer_id, SUM(amount) as total_revenue
FROM orders
WHERE order_date BETWEEN '2026-01-01' AND '2026-05-17'
GROUP BY customer_id
ORDER BY total_revenue DESC
LIMIT 100;
```

### AWS Lakehouse Stack (Apache Iceberg)

```python
# AWS Glue + Apache Iceberg + S3
# 1. Tạo Iceberg table qua Glue
spark.sql("""
CREATE TABLE glue_catalog.analytics.orders (
    order_id    BIGINT,
    customer_id BIGINT,
    amount      DECIMAL(10,2),
    order_date  DATE,
    status      STRING
)
USING iceberg
LOCATION 's3://my-lake/analytics/orders/'
PARTITIONED BY (days(order_date))
""")

# 2. UPSERT (Merge) — không làm được với plain Parquet
spark.sql("""
MERGE INTO glue_catalog.analytics.orders t
USING updates s ON t.order_id = s.order_id
WHEN MATCHED AND s.status = 'cancelled'
    THEN DELETE
WHEN MATCHED
    THEN UPDATE SET t.amount = s.amount, t.status = s.status
WHEN NOT MATCHED
    THEN INSERT *
""")

# 3. Time Travel — query dữ liệu tại thời điểm cũ
spark.sql("""
SELECT * FROM glue_catalog.analytics.orders
TIMESTAMP AS OF '2026-01-01 00:00:00'
""")
```

---

## Câu Hỏi Phỏng Vấn

**Q1: Giải thích sự khác biệt giữa Data Lake và Data Warehouse.**

> **Trả lời:** Data Lake (Hồ Dữ Liệu) lưu trữ mọi loại dữ liệu ở định dạng gốc (raw) với schema-on-read — nghĩa là schema chỉ được áp dụng khi đọc. Rẻ hơn nhiều nhưng cần xử lý thêm trước khi dùng cho analytics. Data Warehouse (Kho Dữ Liệu) lưu dữ liệu có cấu trúc đã qua ETL, schema-on-write, optimized cho OLAP queries với columnar storage và MPP. Query nhanh hơn và dễ cho business users nhưng kém linh hoạt và đắt hơn. Trên AWS: S3 + Lake Formation = Data Lake, Redshift = Data Warehouse.

**Q2: Data Lake dễ bị trở thành "Data Swamp" — làm sao tránh?**

> **Trả lời:** Data Swamp (Đầm Lầy Dữ Liệu) xảy ra khi không có governance tốt. Tránh bằng: (1) **AWS Lake Formation** — quản lý permissions tập trung, ai được đọc dữ liệu nào; (2) **Glue Data Catalog** — mọi dataset phải có metadata (owner, description, schema, quality score); (3) **Naming conventions** — folder structure chuẩn: /raw/, /processed/, /curated/ + partition theo date; (4) **Data Quality rules** — Glue Data Quality validate data trước khi ghi; (5) **Data lineage** — track dữ liệu đến từ đâu, đã transform gì; (6) **Regular cleanup** — lifecycle policies xóa/archive dữ liệu cũ.

**Q3: Khi nào nên dùng Apache Iceberg trên S3 thay vì Amazon Redshift?**

> **Trả lời:** Dùng Iceberg trên S3 khi: (1) Cần ACID transactions nhưng muốn giữ chi phí storage thấp; (2) Có cả ML workloads (SageMaker) lẫn BI workloads (Athena) trên cùng dataset — tránh copy data; (3) Cần upserts/deletes trong data lake (ví dụ: GDPR right-to-erasure); (4) Data volume rất lớn (PB) → S3 rẻ hơn nhiều so với Redshift nodes. Dùng Redshift khi: (1) Primary use case là BI với complex joins và cần sub-second query; (2) Team không có expertise về Spark/Iceberg; (3) Cần tích hợp chặt với BI tools (Tableau, QuickSight via JDBC).

**Q4: Thiết kế storage strategy cho một company với requirements: 10TB data/ngày, ML team + BI team, budget $50K/tháng.**

> **Trả lời:** Tôi sẽ thiết kế Data Lakehouse:
> - **Raw zone:** S3 Standard-IA lưu raw data 90 ngày → Glacier sau đó (~$2K/tháng)
> - **Processed zone:** S3 Standard với Parquet + Apache Iceberg tables (~$5K/tháng)
> - **Curated zone:** Redshift Serverless cho BI queries, chỉ load aggregated data (~$15K/tháng)
> - **ML data:** Notebook access thẳng vào S3 processed zone qua SageMaker (~$10K/tháng)
> - **Catalog:** Glue Data Catalog + Lake Formation (~$1K/tháng)
> - **Total:** ~$33K/tháng, trong budget. Lợi thế: ML team dùng S3 không phải qua Redshift, BI team dùng Redshift nhanh, không duplicate data lớn.

---

## 💡 Key Takeaways

1. **Data Lake = linh hoạt, rẻ** — phù hợp dữ liệu đa dạng, ML workloads, volume lớn
2. **Data Warehouse = nhanh, có cấu trúc** — phù hợp BI, reporting, business users
3. **Data Lakehouse = tốt nhất của cả hai** — ACID trên S3, một copy data cho nhiều use cases
4. **Không có "best" — chỉ có "best for your context"**
5. **AWS Stack:** S3 = Data Lake core; Redshift = Data Warehouse; S3 + Iceberg/Glue = Lakehouse
6. **Governance quan trọng như storage** — không có Lake Formation + Catalog → Data Swamp

---

**← [3-batch-vs-streaming.md](./3-batch-vs-streaming.md)** | **→ [5-etl-elt-concepts.md](./5-etl-elt-concepts.md)**
