# 2 — Data Pipeline Concepts: Khái Niệm Đường Ống Dữ Liệu

> Data Pipeline (Đường Ống Dữ Liệu) là tập hợp các bước tự động hóa để di chuyển, biến đổi và phân phối dữ liệu từ nguồn đến đích. Hiểu kiến trúc pipeline là kỹ năng cốt lõi của Data Engineer.

## 📚 Mục Lục

1. [Data Pipeline Là Gì?](#data-pipeline-là-gì)
2. [4 Tầng Của Data Pipeline](#4-tầng-của-data-pipeline)
3. [Ingestion — Tầng Thu Nạp](#ingestion--tầng-thu-nạp)
4. [Processing — Tầng Xử Lý](#processing--tầng-xử-lý)
5. [Storage — Tầng Lưu Trữ](#storage--tầng-lưu-trữ)
6. [Serving — Tầng Phục Vụ](#serving--tầng-phục-vụ)
7. [Kiến Trúc Pipeline Thực Tế](#kiến-trúc-pipeline-thực-tế)
8. [Thành Phần Hỗ Trợ](#thành-phần-hỗ-trợ)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Data Pipeline Là Gì?

**Data Pipeline** (Đường Ống Dữ Liệu) là một chuỗi các bước xử lý tự động hóa để:
1. **Thu thập** dữ liệu từ nhiều nguồn khác nhau
2. **Biến đổi** dữ liệu thành định dạng phù hợp
3. **Lưu trữ** dữ liệu ở vị trí mong muốn
4. **Cung cấp** dữ liệu cho người dùng hoặc ứng dụng

```
Nguồn Dữ Liệu → [PIPELINE] → Đích Dữ Liệu

Cơ sở dữ liệu ─┐
API bên ngoài  ─┤
Log files      ─┤→ Ingest → Process → Store → Serve → Analytics / Apps
Thiết bị IoT   ─┤
Clickstream    ─┘
```

**Tại sao cần Data Pipeline?**
- Dữ liệu thô từ nhiều nguồn không thể dùng trực tiếp
- Cần tự động hóa để xử lý volume lớn
- Đảm bảo consistency (tính nhất quán) và quality (chất lượng)
- Tách biệt concerns: ingestion ≠ processing ≠ serving

---

## 4 Tầng Của Data Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│                     DATA PIPELINE ARCHITECTURE                      │
├──────────────┬──────────────────┬──────────────┬────────────────────┤
│  INGESTION   │    PROCESSING    │   STORAGE    │      SERVING       │
│  (Thu Nạp)   │    (Xử Lý)       │  (Lưu Trữ)  │    (Phục Vụ)       │
├──────────────┼──────────────────┼──────────────┼────────────────────┤
│ Collect raw  │ Clean, transform │ Persist data │ Expose to users    │
│ data from    │ validate, enrich │ in optimal   │ and applications   │
│ sources      │ aggregate data   │ formats      │ securely           │
├──────────────┼──────────────────┼──────────────┼────────────────────┤
│ • Kinesis    │ • AWS Glue       │ • S3         │ • Athena           │
│ • MSK        │ • EMR/Spark      │ • Redshift   │ • Redshift         │
│ • DMS        │ • Lambda         │ • DynamoDB   │ • OpenSearch       │
│ • Transfer   │ • Kinesis Anal.  │ • RDS        │ • QuickSight       │
└──────────────┴──────────────────┴──────────────┴────────────────────┘
```

---

## Ingestion — Tầng Thu Nạp

**Định nghĩa:** Tầng đầu tiên — thu thập dữ liệu thô từ các nguồn khác nhau và đưa vào pipeline.

### Các Loại Data Sources (Nguồn Dữ Liệu)

| Loại Nguồn              | Ví Dụ                                     | Đặc Điểm                           |
| ----------------------- | ----------------------------------------- | ---------------------------------- |
| Transactional DB        | MySQL, PostgreSQL, Oracle                 | Structured, ACID                   |
| NoSQL DB                | MongoDB, DynamoDB, Cassandra              | Flexible schema, high throughput   |
| APIs                    | REST, GraphQL, WebSocket                  | Pull-based, rate-limited           |
| Event streams           | Kafka topics, Kinesis streams             | Push-based, ordered, replay        |
| Files                   | CSV, JSON, Parquet, Avro trên S3/HDFS     | Batch-friendly                     |
| IoT devices             | MQTT, HTTP từ sensors                     | High volume, small messages        |
| Clickstream             | Web/mobile user events                    | High throughput, session-based     |
| SaaS applications       | Salesforce, HubSpot, Stripe               | API với pagination                 |

### Ingestion Patterns (Mẫu Thu Nạp)

#### 1. Full Load (Tải Toàn Bộ)
```
Lần đầu hoặc khi cần refresh hoàn toàn:
Source DB ──────────────────────────────→ Target
         (tất cả records)

Ưu điểm: Đơn giản, đảm bảo sync hoàn toàn
Nhược điểm: Tốn bandwidth, không phù hợp với large tables
```

#### 2. Incremental Load (Tải Gia Tăng)
```
Chỉ lấy dữ liệu mới/thay đổi:
Source DB ──────────────────────────────→ Target
         (WHERE updated_at > last_run)

Ưu điểm: Hiệu quả, nhanh hơn
Nhược điểm: Cần timestamp hoặc CDC, có thể miss deletes
```

#### 3. CDC — Change Data Capture (Bắt Thay Đổi Dữ Liệu)
```
Theo dõi thay đổi ở log level:
Source DB → Database Log (binlog) → CDC Tool → Target
                                  (INSERT/UPDATE/DELETE events)

Ưu điểm: Real-time, bắt được deletes, minimal load trên source
Nhược điểm: Phức tạp hơn, cần cấu hình log
```

### AWS Ingestion Services

| Service                  | Use Case                                         | Pattern          |
| ------------------------ | ------------------------------------------------ | ---------------- |
| **Kinesis Data Streams** | Real-time event streaming từ apps               | Streaming        |
| **Kinesis Firehose**     | Managed delivery vào S3/Redshift/OpenSearch     | Streaming (managed) |
| **MSK (Kafka)**          | High-throughput streaming, Kafka ecosystem       | Streaming        |
| **AWS DMS** (Database Migration Service) | Migrate database, CDC từ RDS     | Batch + CDC      |
| **AWS Transfer Family**  | SFTP/FTP/FTPS file transfer vào S3              | File-based       |
| **AWS DataSync**         | Sync data giữa on-premises và AWS               | Batch            |
| **AWS Glue**             | Pull từ JDBC sources, APIs                       | Batch            |

---

## Processing — Tầng Xử Lý

**Định nghĩa:** Biến đổi dữ liệu thô thành dữ liệu có giá trị — làm sạch, validate, enrich, aggregate.

### Các Loại Processing Operations (Thao Tác Xử Lý)

#### Data Cleaning (Làm Sạch Dữ Liệu)
```python
# Ví dụ: Remove duplicates, handle nulls, fix data types
df = df.dropDuplicates()
df = df.fillna({'email': 'unknown', 'age': 0})
df = df.withColumn('price', df['price'].cast('double'))
df = df.filter(df['amount'] > 0)  # Remove invalid records
```

#### Data Transformation (Biến Đổi Dữ Liệu)
```python
# Ví dụ: Normalize, reshape, compute derived fields
df = df.withColumn('revenue_usd', df['amount_vnd'] / 23000)
df = df.withColumn('date', to_date(df['timestamp']))
df = df.withColumn('user_tier',
    when(df['lifetime_value'] > 1000, 'gold')
    .when(df['lifetime_value'] > 100, 'silver')
    .otherwise('bronze'))
```

#### Data Enrichment (Làm Phong Phú Dữ Liệu)
```python
# Ví dụ: Join với reference data để thêm context
orders_df = orders_df.join(customers_df, 'customer_id', 'left')
orders_df = orders_df.join(products_df, 'product_id', 'left')
# Giờ orders có đầy đủ thông tin customer + product
```

#### Aggregation (Tổng Hợp)
```python
# Ví dụ: Summarize data cho analytics
daily_revenue = orders_df \
    .groupBy('date', 'region') \
    .agg(sum('revenue').alias('total_revenue'),
         count('*').alias('order_count'),
         avg('revenue').alias('avg_order_value'))
```

### AWS Processing Services

| Service                      | Mô Tả                                              | Khi Nào Dùng                          |
| ---------------------------- | -------------------------------------------------- | ------------------------------------- |
| **AWS Glue**                 | Serverless ETL với PySpark                         | ETL không cần quản lý cluster         |
| **Amazon EMR**               | Managed Hadoop/Spark cluster                       | Xử lý lớn, cần kiểm soát cluster     |
| **Amazon Kinesis Analytics** | SQL/Apache Flink trên streaming data               | Real-time stream processing           |
| **AWS Lambda**               | Serverless function, event-driven                  | Light transformations, triggers       |
| **Amazon SageMaker**         | ML pipelines, feature engineering                  | ML/AI workloads                       |

---

## Storage — Tầng Lưu Trữ

**Định nghĩa:** Lưu trữ dữ liệu đã xử lý ở định dạng và vị trí tối ưu cho downstream use cases.

### Storage Patterns (Mẫu Lưu Trữ)

#### 1. Raw Zone (Vùng Thô) — Bronze Layer
```
Đặc điểm:
- Dữ liệu gốc, không biến đổi
- Schema-on-read (đọc mới xác định schema)
- Immutable (bất biến) — không bao giờ xóa/sửa
- Lưu trữ mãi mãi để audit và reprocessing

AWS: S3/raw/ hoặc S3/bronze/
```

#### 2. Processed Zone (Vùng Đã Xử Lý) — Silver Layer
```
Đặc điểm:
- Dữ liệu đã làm sạch, chuẩn hóa
- Schema cố định, validated
- Columnar format (Parquet, ORC) cho performance
- Partitioned (phân vùng) theo time/category

AWS: S3/processed/ hoặc S3/silver/
```

#### 3. Curated Zone (Vùng Tinh Chế) — Gold Layer
```
Đặc điểm:
- Aggregated, business-ready data
- Optimized cho specific use cases
- Loaded vào Redshift hoặc DynamoDB cho serving
- Refreshed theo schedule

AWS: Redshift / S3/gold/ / DynamoDB
```

### Storage Formats (Định Dạng Lưu Trữ)

| Format       | Loại       | Tốt Cho                             | Tính Năng Nổi Bật                      |
| ------------ | ---------- | ----------------------------------- | --------------------------------------- |
| **CSV**      | Row-based  | Human-readable, export/import       | Đơn giản, phổ biến, không nén tốt      |
| **JSON**     | Row-based  | Semi-structured, APIs               | Flexible schema, nested data           |
| **Parquet**  | Columnar   | Analytics queries, Athena, Spark    | Nén tốt (~70%), schema embedded        |
| **ORC**      | Columnar   | Hive, analytics                     | Tốt hơn Parquet cho Hive workloads     |
| **Avro**     | Row-based  | Streaming, schema evolution         | Schema evolution tốt, Kafka-friendly   |
| **Delta/Iceberg** | Columnar | ACID transactions trên data lake | Time travel, ACID, upsert support     |

> **Quy tắc vàng:** Dùng **Parquet** cho analytics trên S3 — giảm ~70% storage cost và tăng ~10x query performance so với CSV.

### AWS Storage Services

| Service             | Loại                  | Use Case                                     |
| ------------------- | --------------------- | -------------------------------------------- |
| **Amazon S3**       | Object storage        | Data lake, raw/processed data, backup        |
| **Amazon Redshift** | MPP data warehouse    | OLAP analytics, complex queries              |
| **Amazon DynamoDB** | NoSQL key-value       | Serving layer, low-latency lookups           |
| **Amazon RDS**      | Relational database   | OLTP workloads, operational data             |
| **Amazon Aurora**   | Managed RDBMS         | High-performance OLTP, global databases      |
| **ElastiCache**     | In-memory cache       | Sub-millisecond lookup, session storage      |

---

## Serving — Tầng Phục Vụ

**Định nghĩa:** Cung cấp dữ liệu đã xử lý cho người dùng cuối, ứng dụng và các hệ thống analytics.

### Serving Patterns (Mẫu Phục Vụ)

#### 1. Batch Serving (Phục Vụ Theo Lô)
```
Use case: Báo cáo định kỳ, ETL downstream

Cơ chế:
- Data được precomputed và stored
- Query chạy trên pre-aggregated data
- Latency: giây đến phút chấp nhận được

AWS: Athena query S3, Redshift queries, QuickSight SPICE
```

#### 2. Online Serving (Phục Vụ Trực Tuyến)
```
Use case: Real-time features cho ML, API responses

Cơ chế:
- Low-latency lookups (< 10ms)
- Pre-computed features stored trong fast storage
- Single record lookups by key

AWS: DynamoDB, ElastiCache, Aurora
```

#### 3. Streaming Serving (Phục Vụ Luồng)
```
Use case: Real-time dashboards, alerts, event-driven apps

Cơ chế:
- Continuous data flow
- Near real-time updates (giây)
- Push notifications khi events xảy ra

AWS: Kinesis → Lambda → API Gateway, OpenSearch Dashboards
```

### AWS Serving Services

| Service                      | Latency       | Use Case                                   |
| ---------------------------- | ------------- | ------------------------------------------ |
| **Amazon Athena**            | Giây - phút  | Ad-hoc SQL, data exploration               |
| **Amazon Redshift**          | Milli - giây | Complex OLAP, BI tools                     |
| **Amazon DynamoDB**          | Milli giây   | Real-time apps, low-latency lookups        |
| **Amazon OpenSearch**        | Milli giây   | Full-text search, log analytics            |
| **Amazon QuickSight**        | Giây         | BI dashboards, reports                     |
| **Amazon SageMaker**         | Milli giây   | ML model inference                         |
| **API Gateway + Lambda**     | Milli giây   | Custom API endpoints                       |

---

## Kiến Trúc Pipeline Thực Tế

### Ví Dụ 1: E-commerce Analytics Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│                    E-COMMERCE ANALYTICS PIPELINE                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  INGESTION          PROCESSING         STORAGE          SERVING     │
│                                                                      │
│  RDS (orders) ─────→ Glue ETL Job ──→ S3/processed/ ──→ Athena     │
│                          │                               │           │
│  Clickstream ──→ Kinesis─┘               Redshift ←────→ QuickSight │
│  (user events)   Firehose                    ↑                       │
│                     │            Glue Job────┘                       │
│  Mobile App ────→ Kinesis                                            │
│  (real-time)     Analytics ──────────────────────────→ Lambda Alert │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Ví Dụ 2: Log Analytics Pipeline

```
Application Servers
        │
        ▼
  CloudWatch Logs / Kinesis Firehose
        │
        ▼ (raw logs)
  S3 /raw/logs/year=2026/month=05/day=17/
        │
        ▼ (Glue Crawler tự động detect schema)
  Glue Data Catalog
        │
        ├──→ Athena (ad-hoc investigation)
        │
        └──→ OpenSearch (real-time search & alerting)
```

---

## Thành Phần Hỗ Trợ

Ngoài 4 tầng chính, một data pipeline production cần các thành phần hỗ trợ:

### 1. Orchestration (Điều Phối)
**Vấn đề:** Làm sao chạy các steps đúng thứ tự, handle dependencies, retry on failure?

```
Airflow DAG (Directed Acyclic Graph — Đồ Thị Không Chu Trình Có Hướng):

Extract ──→ Validate ──→ Transform ──→ Load ──→ Notify
  │              │            │          │
  └── (fail) ───→ Alert     (fail)    (fail)
                           Retry x3   Rollback
```

**AWS Options:**
- **AWS Step Functions** — Visual workflow orchestration, serverless
- **Amazon MWAA** (Managed Workflows for Apache Airflow) — Managed Airflow
- **AWS Glue Workflows** — Orchestrate Glue jobs và crawlers
- **Amazon EventBridge** — Event-driven scheduling và triggers

### 2. Monitoring & Alerting (Giám Sát & Cảnh Báo)
**Các metrics cần theo dõi:**
- Pipeline latency (độ trễ): thời gian từ data source đến serving
- Data freshness (độ tươi dữ liệu): dữ liệu cũ nhất là bao lâu?
- Error rate (tỷ lệ lỗi): % records bị drop hoặc fail
- Throughput (thông lượng): records/second processed
- Cost (chi phí): DPU-hours, S3 storage, query costs

**AWS Options:** CloudWatch, CloudWatch Alarms, SNS notifications

### 3. Data Quality (Chất Lượng Dữ Liệu)
**Các kiểm tra cần thiết:**
```
Completeness check:   COUNT(*) WHERE field IS NULL < threshold?
Uniqueness check:     COUNT(DISTINCT id) = COUNT(*)?
Validity check:       email REGEXP valid pattern?
Freshness check:      MAX(created_at) > NOW() - 1 HOUR?
Range check:          amount BETWEEN 0 AND 1000000?
```

**AWS Options:** AWS Glue Data Quality, dbt tests, Great Expectations

### 4. Data Catalog (Danh Mục Dữ Liệu)
**Mục đích:** Biết dữ liệu nào tồn tại, ở đâu, schema như thế nào, ai sở hữu.

**AWS Glue Data Catalog** lưu:
- Table definitions (database, table name, columns, types)
- Partition information (phân vùng theo date, region)
- Data location (S3 path)
- Statistics (row count, size, last updated)

### 5. Security (Bảo Mật)
```
Encryption at rest:    S3 SSE-KMS, Redshift encryption
Encryption in transit: TLS/HTTPS cho mọi connections
Access control:        IAM roles, Lake Formation permissions
Audit logging:         CloudTrail, S3 access logs
PII handling:          Glue DataBrew masking, Macie detection
```

---

## Câu Hỏi Phỏng Vấn

**Q1: Vẽ data pipeline từ database production vào dashboard analytics.**

> **Trả lời:** 
> 1. **Ingestion:** AWS DMS (CDC mode) để capture changes từ RDS production → ghi vào S3 raw zone (tránh load trực tiếp lên DB production)
> 2. **Processing:** AWS Glue ETL job (chạy hourly) đọc S3 raw → clean/transform → ghi Parquet vào S3 processed zone
> 3. **Storage:** Load processed data vào Redshift với COPY command
> 4. **Serving:** QuickSight kết nối Redshift → dashboards cho business users
>
> *Giải thích thêm: Dùng CDC thay full load để giảm tải DB production. Glue serverless để không quản lý cluster. Redshift cho complex OLAP queries.*

**Q2: Làm sao đảm bảo data quality trong pipeline?**

> **Trả lời:** Đặt data quality checks tại nhiều điểm:
> - **Tại Ingestion:** Validate schema, reject malformed records
> - **Tại Processing:** Glue Data Quality rules (null checks, range checks, uniqueness)
> - **Tại Storage:** Constraint enforcement trong Redshift (NOT NULL, UNIQUE)
> - **Tại Serving:** dbt tests để validate business logic
> - **Alerting:** CloudWatch alarm khi error rate > threshold, notify team qua SNS

**Q3: Nếu pipeline bị chậm (latency cao), bạn debug như thế nào?**

> **Trả lời:** Kiểm tra từng tầng:
> 1. **Ingestion bottleneck?** — Kinesis throttling? Insufficient shards? → Tăng shard count
> 2. **Processing bottleneck?** — Glue job dùng quá ít DPU? EMR cluster không đủ nodes? → Scale up
> 3. **Storage bottleneck?** — S3 throttling? Large file writes? → Dùng multipart upload
> 4. **Serving bottleneck?** — Athena query full scan? → Thêm partitions, chuyển sang Parquet
> 5. **Kiểm tra CloudWatch metrics** tại mỗi stage để xác định bottleneck chính xác

---

## 💡 Key Takeaways

1. **4 tầng pipeline:** Ingestion → Processing → Storage → Serving — mỗi tầng có trách nhiệm riêng
2. **Dùng Parquet** cho S3 analytics — nén tốt hơn, query nhanh hơn, tiết kiệm chi phí
3. **Luôn giữ raw data** — immutable, không bao giờ xóa, dùng cho reprocessing
4. **Tách biệt concerns** — ingestion layer không nên làm transformation phức tạp
5. **Monitoring là bắt buộc** — pipeline không có monitoring = pipeline không đáng tin cậy
6. **Data Quality là non-negotiable** — garbage in, garbage out

---

**← [1-analytics-overview.md](./1-analytics-overview.md)** | **→ [3-batch-vs-streaming.md](./3-batch-vs-streaming.md)**
