# 5 — ETL vs ELT: Định Nghĩa, So Sánh, Quyết Định

> ETL (Extract Transform Load — Trích Xuất Chuyển Đổi Tải) và ELT (Extract Load Transform — Trích Xuất Tải Chuyển Đổi) là hai pattern cơ bản để di chuyển và biến đổi dữ liệu. Cloud data warehouses đã làm thay đổi hoàn toàn cách chúng ta tiếp cận lựa chọn này.

## 📚 Mục Lục

1. [Định Nghĩa ETL](#định-nghĩa-etl)
2. [Định Nghĩa ELT](#định-nghĩa-elt)
3. [So Sánh Trực Tiếp](#so-sánh-trực-tiếp)
4. [Khi Nào Dùng ETL](#khi-nào-dùng-etl)
5. [Khi Nào Dùng ELT](#khi-nào-dùng-elt)
6. [AWS Implementation](#aws-implementation)
7. [dbt — Công Cụ ELT Hiện Đại](#dbt--công-cụ-elt-hiện-đại)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Định Nghĩa ETL

**ETL** = **E**xtract **T**ransform **L**oad — Trích Xuất, Chuyển Đổi, Tải

### Flow (Luồng Xử Lý)

```
Source          Staging Area           Target
  │              (Transform)            │
  │  Extract     ┌─────────────┐  Load  │
  ├─────────────→│             │───────→│
  │              │  Transform  │        │
  │              │  - Clean    │        │
  │              │  - Validate │        │
  │              │  - Enrich   │        │
  │              │  - Aggregate│        │
  │              └─────────────┘        │
Database/API  ETL Server/Tool      Data Warehouse

→ Transform xảy ra TRƯỚC khi Load vào target
```

### Ví Dụ ETL Thực Tế

```python
# AWS Glue ETL Job (PySpark)

# Step 1: EXTRACT — đọc từ nguồn
raw_orders = glueContext.create_dynamic_frame.from_catalog(
    database="raw_db",
    table_name="orders_raw"
)

# Step 2: TRANSFORM — trên Glue server (TRƯỚC khi load vào Redshift)
from awsglue.transforms import *

# Clean
clean_orders = Filter.apply(
    frame=raw_orders,
    f=lambda x: x["amount"] > 0 and x["status"] != "test"
)

# Enrich
enriched_orders = clean_orders.map(lambda row: {
    **row,
    "revenue_usd": row["amount_vnd"] / 23000,
    "order_month": row["order_date"][:7],
    "customer_tier": "gold" if row["lifetime_value"] > 1000 else "standard"
})

# Step 3: LOAD — ghi vào Redshift (chỉ clean data)
glueContext.write_dynamic_frame.from_jdbc_conf(
    frame=enriched_orders,
    catalog_connection="redshift-connection",
    connection_options={"dbtable": "analytics.orders_clean"}
)
```

### Đặc Điểm ETL

- **Transform trước khi Load** — target chỉ nhận clean data
- **Staging area riêng** — ETL server/tool xử lý transform
- **Data không vào target ở dạng raw** — bảo vệ data warehouse
- **Phù hợp khi target không đủ mạnh** để transform

---

## Định Nghĩa ELT

**ELT** = **E**xtract **L**oad **T**ransform — Trích Xuất, Tải, Chuyển Đổi

### Flow (Luồng Xử Lý)

```
Source              Target                  Analytics
  │                (Raw + Transform)            │
  │  Extract   Load  ┌─────────────────┐  Use   │
  ├──────────────────→│                 │───────→│
  │                  │  Raw Data Layer │        │
  │                  │       +         │        │
  │                  │  Transform      │        │
  │                  │  (SQL INSIDE    │        │
  │                  │   TARGET DB)    │        │
  │                  └─────────────────┘        │
Database/API     Data Warehouse/Data Lake   BI Tools

→ Load raw data vào target TRƯỚC, Transform xảy ra BÊN TRONG target
```

### Ví Dụ ELT Thực Tế

```sql
-- ELT với dbt (data build tool) trên Amazon Redshift

-- Step 1: EXTRACT + LOAD (Fivetran/Airbyte copy raw data vào Redshift)
-- Table: raw.orders (dữ liệu thô, chưa transform)

-- Step 2: TRANSFORM — xảy ra trong Redshift bằng SQL
-- File: models/staging/stg_orders.sql
SELECT
    order_id,
    customer_id,
    CAST(amount_vnd AS DECIMAL(15,2)) / 23000 AS amount_usd,
    CASE
        WHEN status = 'completed' THEN 'success'
        WHEN status IN ('cancelled', 'refunded') THEN 'failed'
        ELSE 'pending'
    END AS normalized_status,
    DATE(created_at AT TIME ZONE 'UTC' AT TIME ZONE 'Asia/Ho_Chi_Minh') AS order_date
FROM raw.orders
WHERE amount_vnd > 0
  AND status != 'test';

-- File: models/mart/daily_revenue.sql
SELECT
    order_date,
    SUM(amount_usd) AS total_revenue_usd,
    COUNT(*) AS order_count,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM {{ ref('stg_orders') }}
WHERE normalized_status = 'success'
GROUP BY order_date;
```

### Đặc Điểm ELT

- **Load raw trước, Transform sau** — raw data luôn có trong target
- **Transform dùng SQL** — không cần Spark hay ETL server
- **Tận dụng compute của data warehouse** — Redshift MPP engine
- **Linh hoạt** — có thể thêm transformations mới mà không re-ingest

---

## So Sánh Trực Tiếp

| Tiêu Chí                  | ETL                                     | ELT                                     |
| ------------------------- | --------------------------------------- | --------------------------------------- |
| **Thứ tự**                | Extract → **Transform** → Load          | Extract → Load → **Transform**          |
| **Nơi transform**         | ETL server (ngoài target)               | Bên trong target (DW/Data Lake)         |
| **Raw data trong DW?**    | Không — chỉ clean data                  | Có — raw data luôn tồn tại             |
| **Compute requirement**   | ETL server phải mạnh                    | Target DB phải mạnh (MPP)               |
| **Latency**               | Cao hơn (2 bước: transform + load)      | Thấp hơn (load nhanh, transform sau)    |
| **Flexibility**           | Thấp — thay đổi cần modify ETL job     | Cao — thêm SQL transform là xong        |
| **Data lineage**          | Phức tạp hơn                            | Dễ hơn (raw data + SQL = full lineage) |
| **Tooling**               | Glue, Informatica, Talend, Spark        | dbt, Dataform, Redshift SQL             |
| **Khi target DW yếu**     | ✅ Phù hợp                              | ❌ Không phù hợp                        |
| **Khi target DW mạnh**    | ❌ Lãng phí DW compute                  | ✅ Tận dụng tối đa DW                   |
| **PII/Sensitive data**    | ✅ Có thể mask trước khi load           | ⚠️ Raw data chứa PII trong DW          |
| **On-premises legacy DW** | ✅ Phổ biến với DW cũ                   | ❌ Không phù hợp                        |
| **Cloud DW (Redshift)**   | Vẫn dùng được                           | ✅ Phổ biến và hiệu quả hơn            |

---

## Khi Nào Dùng ETL

### Tình Huống ETL Phù Hợp

**1. Target database không đủ mạnh để transform**
```
On-premises Oracle Data Warehouse (10 nodes cũ)
→ Không có compute spare cho transformation
→ ETL server riêng xử lý transform trước khi load
→ Load vào DW chỉ clean, aggregated data
```

**2. Dữ liệu có PII (Personally Identifiable Information — Thông Tin Nhận Dạng Cá Nhân) cần mask trước**
```
Raw data: { "ssn": "123-45-6789", "credit_card": "4111..." }
→ ETL server mask/tokenize PII trước khi load vào DW
→ DW không bao giờ chứa raw PII
→ Compliance: GDPR, PCI-DSS, HIPAA
```

**3. Complex transformations không thể viết bằng SQL**
```
- Custom ML feature engineering
- PDF/image processing
- Complex business logic trong code (Python/Java)
- Third-party API calls để enrich data
```

**4. Dữ liệu từ nhiều nguồn với định dạng rất khác nhau**
```
XML from SAP + CSV from legacy + JSON from API
→ ETL tool có connectors và parsers cho mọi format
→ Normalize về common schema rồi load vào DW
```

**5. Legacy systems và on-premises environments**
```
On-premises infrastructure với limited DW resources
→ ETL là pattern truyền thống, được support tốt
→ Team đã quen với ETL tools (Informatica, SSIS)
```

### AWS ETL Stack
```
AWS Glue (primary ETL engine):
- PySpark jobs cho transformations phức tạp
- Python Shell cho lightweight scripts
- Visual ETL với Glue Studio (kéo thả)

Workflow:
RDS/S3/API → Glue ETL Job (transform) → Redshift/S3
```

---

## Khi Nào Dùng ELT

### Tình Huống ELT Phù Hợp

**1. Target là cloud data warehouse mạnh (Redshift, Snowflake, BigQuery)**
```
Amazon Redshift với 20 nodes:
→ Có compute dư để chạy SQL transforms
→ Tại sao phải dùng ETL server riêng?
→ Load raw vào Redshift, transform bằng SQL/dbt
→ MPP engine của Redshift nhanh hơn ETL server riêng
```

**2. Team data nhỏ, muốn đơn giản**
```
Team 2 Data Analysts + 1 Data Engineer
→ Không ai biết PySpark
→ Tất cả biết SQL
→ Dùng ELT + dbt → mọi người contribute được
→ Git version control cho SQL models → collaboration
```

**3. Cần flexibility — requirements thay đổi thường xuyên**
```
Business team thường xuyên thay đổi định nghĩa metrics:
"Revenue" = gross hay net? Có include tax không?

ETL: Phải modify PySpark job, deploy lại, test
ELT: Sửa SQL trong dbt model, chạy lại → xong trong 30 phút
```

**4. Muốn giữ raw data để reprocess**
```
Có bug trong transformation logic:
ETL: Raw data đã bị transform trước khi load → mất raw
ELT: Raw data vẫn trong DW → rerun SQL transform là xong
```

**5. Data exploration và iterative development**
```
Data scientist muốn thử nhiều transformation variants:
ELT: Viết SQL khác nhau trên raw data → iterate nhanh
ETL: Phải modify ETL pipeline → chậm hơn nhiều
```

### AWS ELT Stack

```
Modern ELT on AWS:

Source DB → Fivetran/Airbyte (Extract + Load) → Redshift Raw Schema
                                                        │
                                               dbt (Transform in Redshift SQL)
                                                        │
                                               QuickSight (BI)

Hoặc:

Source → AWS Glue (Extract + Load raw) → S3/Redshift Raw
                                              │
                                     dbt (Transform SQL)
                                              │
                                        BI / Analytics
```

---

## AWS Implementation

### ETL: AWS Glue

```python
# Glue ETL Job hoàn chỉnh
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.sql.functions import col, when, to_date, lit
from pyspark.sql.types import DecimalType

args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# EXTRACT
orders_raw = glueContext.create_dynamic_frame.from_catalog(
    database="raw_db",
    table_name="orders"
).toDF()

# TRANSFORM (trên Glue server — trước khi load)
orders_clean = (orders_raw
    # Clean
    .filter(col("amount") > 0)
    .filter(col("status").isin(["completed", "processing", "cancelled"]))
    .dropDuplicates(["order_id"])
    # Transform types
    .withColumn("amount_usd", col("amount_vnd").cast(DecimalType(15,2)) / lit(23000))
    .withColumn("order_date", to_date(col("created_at")))
    # Enrich
    .withColumn("customer_tier",
        when(col("lifetime_value") > 1000, "gold")
        .when(col("lifetime_value") > 100, "silver")
        .otherwise("bronze"))
    # Select only needed columns
    .select("order_id", "customer_id", "amount_usd", "order_date",
            "status", "customer_tier")
)

# LOAD (chỉ clean data vào Redshift)
glueContext.write_dynamic_frame.from_options(
    frame=DynamicFrame.fromDF(orders_clean, glueContext, "orders_clean"),
    connection_type="redshift",
    connection_options={
        "redshiftTmpDir": "s3://my-temp/redshift/",
        "useConnectionProperties": "true",
        "dbtable": "analytics.orders_clean",
        "connectionName": "redshift-prod"
    }
)

job.commit()
```

### ELT: dbt + Redshift

```yaml
# dbt project structure
models/
├── staging/          # 1-to-1 mapping từ raw source, minimal transform
│   ├── stg_orders.sql
│   └── stg_customers.sql
├── intermediate/     # Business logic, joins
│   └── int_orders_enriched.sql
└── mart/            # Final models cho BI
    ├── fct_orders.sql        # Fact table
    └── dim_customers.sql     # Dimension table
```

```sql
-- models/staging/stg_orders.sql
-- Chạy trong Redshift (ELT — Transform bên trong target)
WITH source AS (
    SELECT * FROM {{ source('raw', 'orders') }}
    WHERE _fivetran_deleted = FALSE  -- Fivetran CDC column
),

cleaned AS (
    SELECT
        order_id::BIGINT AS order_id,
        customer_id::BIGINT AS customer_id,
        amount_vnd::DECIMAL(15,2) / 23000 AS amount_usd,
        status::VARCHAR AS order_status,
        created_at::TIMESTAMP AS created_at,
        DATE(CONVERT_TIMEZONE('UTC', 'Asia/Ho_Chi_Minh', created_at)) AS order_date
    FROM source
    WHERE amount_vnd > 0
      AND status != 'test'
)

SELECT * FROM cleaned
```

```sql
-- models/mart/fct_orders.sql
SELECT
    o.order_id,
    o.customer_id,
    o.amount_usd,
    o.order_status,
    o.order_date,
    c.customer_tier,
    c.country,
    p.product_category
FROM {{ ref('stg_orders') }} o
LEFT JOIN {{ ref('dim_customers') }} c USING (customer_id)
LEFT JOIN {{ ref('stg_order_items') }} oi USING (order_id)
LEFT JOIN {{ ref('dim_products') }} p USING (product_id)
```

---

## dbt — Công Cụ ELT Hiện Đại

**dbt** (data build tool — Công Cụ Xây Dựng Dữ Liệu) là framework ELT phổ biến nhất hiện nay, cho phép Data Analysts viết transformations bằng SQL với engineering best practices.

### Tại Sao dbt Phổ Biến?

```
Traditional SQL scripts:                 dbt models:
- Không có version control               - Git version control
- Copy-paste SQL, không reuse            - {{ ref() }} để reference models
- Không test                             - Built-in testing framework
- Không documentation                    - Auto-generate docs
- Không lineage                          - Automatic data lineage graph
- Manual execution order                 - Dependency resolution tự động
```

### dbt Core Features

```yaml
# dbt tests — tự động validate data quality
# models/schema.yml
version: 2
models:
  - name: stg_orders
    columns:
      - name: order_id
        tests:
          - unique          # Không có duplicate order_id
          - not_null        # order_id không được null
      - name: order_status
        tests:
          - accepted_values:
              values: ['completed', 'processing', 'cancelled', 'pending']
      - name: amount_usd
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: "amount_usd > 0"
```

```bash
# dbt commands
dbt run        # Chạy tất cả models (CREATE TABLE/VIEW trong DW)
dbt test       # Chạy tất cả tests
dbt docs generate && dbt docs serve  # Generate & serve docs
dbt run --select stg_orders+  # Chỉ chạy stg_orders và downstream
```

### dbt trên AWS

```
AWS Stack với dbt:
Airbyte/Fivetran → Redshift (raw schema) → dbt → Redshift (analytics schema)
                                                        ↓
                                                  QuickSight
```

---

## Patterns Kết Hợp ETL + ELT

Thực tế nhiều hệ thống dùng cả hai:

```
                    HYBRID APPROACH
                    
External API ──→ Glue ETL (Complex transform, PII mask) ──→ S3 processed
                                                               │
MySQL CDC ──→ Fivetran (Load raw) ──→ Redshift raw schema ──→ dbt (ELT)
                                                               │
IoT Streams ──→ Kinesis → Lambda (lightweight ETL) ──→ DynamoDB
                                                               │
                                              ┌────────────────┘
                                              ▼
                                        Redshift analytics schema
                                              │
                                         QuickSight Dashboards
```

---

## Câu Hỏi Phỏng Vấn

**Q1: ETL khác ELT thế nào? Bạn sẽ chọn cái nào cho project mới?**

> **Trả lời:** ETL transform data trước khi load vào target, ELT load raw vào target rồi transform bên trong. Với một project mới trên AWS năm 2026, tôi thường default ELT với Redshift + dbt vì: (1) Redshift MPP đủ mạnh để transform, (2) SQL dễ maintain hơn Spark, (3) Raw data luôn có sẵn để reprocess nếu cần, (4) dbt cung cấp testing, documentation, lineage. Tôi sẽ chuyển sang ETL nếu cần mask PII trước khi load hoặc cần complex Python/Java transformations không thể viết bằng SQL.

**Q2: PII (Personally Identifiable Information) cần được xử lý thế nào trong ETL/ELT?**

> **Trả lời:** PII như email, số điện thoại, CMND cần được bảo vệ. Với ETL: mask/tokenize tại ETL layer trước khi load → DW không bao giờ chứa raw PII, an toàn nhất cho compliance (GDPR, PDPA Vietnam). Với ELT: load raw vào một raw schema có strict access control (chỉ data engineers), transform và mask trong staging layer, analytics models không có PII. Trên AWS: (1) Glue DataBrew có built-in PII detection và masking; (2) Amazon Macie phát hiện PII trong S3; (3) Redshift column-level access control ẩn PII columns với business users.

**Q3: dbt là gì và khi nào dùng nó?**

> **Trả lời:** dbt (data build tool) là framework ELT cho phép transform data trong data warehouse bằng SQL với software engineering best practices như version control (git), testing, documentation và lineage. Dùng dbt khi: (1) Team analytics biết SQL nhưng không biết Spark; (2) Cần data quality testing tích hợp trong pipeline; (3) Muốn documentation tự động và data lineage; (4) Sử dụng cloud DW như Redshift, Snowflake, BigQuery. Trên AWS, dbt thường kết hợp với Fivetran/Airbyte (Extract + Load) + Redshift (Transform + Store) + QuickSight (Serve).

**Q4: Scenario: Team 3 người Data Analyst, dùng Redshift, requirements thay đổi thường xuyên. ETL hay ELT?**

> **Trả lời:** ELT với dbt là lựa chọn rõ ràng ở đây. Ba lý do: (1) **Team size nhỏ:** 3 Data Analysts biết SQL không cần học Spark/PySpark; (2) **Redshift MPP:** đủ mạnh để chạy SQL transforms, không cần ETL server riêng; (3) **Changing requirements:** với dbt, thay đổi business logic chỉ cần sửa SQL file, `dbt run`, không phải deploy Glue job. Stack cụ thể: Fivetran (load raw) + Redshift (compute) + dbt (transform) + dbt tests (quality) + QuickSight (serve). Setup mất 1-2 tuần, maintainability cao.

---

## 💡 Key Takeaways

1. **ETL = Transform trước khi Load** — phù hợp với legacy systems, PII masking, complex non-SQL transforms
2. **ELT = Load raw, Transform sau** — phù hợp với cloud DW mạnh, SQL-first teams, linh hoạt
3. **Xu hướng 2026: ELT đang thắng** — Cloud DW ngày càng mạnh, SQL đủ cho hầu hết transforms
4. **dbt là game-changer** — version control + testing + docs cho SQL transformations
5. **Không phải either/or** — nhiều hệ thống dùng cả hai tùy use case
6. **AWS tools:** Glue = ETL powerhouse; dbt + Redshift = modern ELT stack

---

**← [4-data-lake-vs-warehouse.md](./4-data-lake-vs-warehouse.md)** | **→ [../02-kinesis/README.md](../02-kinesis/README.md)**
