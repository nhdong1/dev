# 1 — Thiết Kế Data Lake — Zone Architecture & Folder Structure

> Thiết kế data lake (hồ dữ liệu) tốt là nền tảng cho mọi kiến trúc analytics. Phần này tập trung vào Zone Architecture (Kiến Trúc Vùng), naming conventions (quy ước đặt tên), partitioning strategy (chiến lược phân vùng), và các nguyên tắc tổ chức dữ liệu trên S3.

---

## 🏗️ Zone Architecture — Kiến Trúc Vùng Dữ Liệu

### Mô Hình 3 Vùng Cơ Bản (Tương Đương Medallion Architecture)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Data Lake on S3                              │
│                                                                     │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐  │
│  │   Raw Zone      │    │  Processed Zone  │    │  Curated Zone   │  │
│  │  (Vùng Thô)     │───▶│  (Vùng Đã Xử    │───▶│  (Vùng Đã Tinh  │  │
│  │                 │    │   Lý)           │    │   Chỉnh)        │  │
│  │  • Dữ liệu gốc  │    │  • Đã làm sạch  │    │  • Sẵn dùng BI  │  │
│  │  • Định dạng    │    │  • Parquet/ORC  │    │  • Aggregated   │  │
│  │    gốc (JSON,   │    │  • Partitioned  │    │  • Joined       │  │
│  │    CSV, logs)   │    │  • Deduplicated │    │  • Business     │  │
│  │  • Không xóa    │    │  • Schema valid │    │    metrics      │  │
│  │    bao giờ      │    │                 │    │                 │  │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘  │
│       Bronze                  Silver                  Gold           │
│  (Theo Medallion Architecture — Kiến Trúc Huy Chương)               │
└─────────────────────────────────────────────────────────────────────┘
```

### So Sánh Tên Gọi Các Zone

| Tên Thông Dụng | Medallion | Mô Tả |
| -------------- | --------- | ----- |
| **Raw / Landing** | Bronze | Dữ liệu gốc, nguyên vẹn như lúc nhận |
| **Processed / Cleansed** | Silver | Đã làm sạch, validate, convert sang columnar format |
| **Curated / Serving** | Gold | Đã aggregate, join — sẵn sàng cho BI và analytics |
| **Sandbox** | — | Vùng thử nghiệm — data scientists khám phá |
| **Archive** | — | Dữ liệu cũ, lưu trữ dài hạn với cost thấp |

---

## 📁 Folder Structure — Cấu Trúc Thư Mục

### Template Chuẩn

```
s3://company-data-lake-{account-id}/
│
├── raw/                              ← Raw Zone (Vùng Thô)
│   ├── source=crm/
│   │   ├── entity=customers/
│   │   │   ├── year=2024/month=01/day=15/
│   │   │   │   └── customers_20240115_143022.json.gz
│   │   │   └── year=2024/month=01/day=16/
│   │   └── entity=orders/
│   │       └── year=2024/month=01/day=15/
│   ├── source=erp/
│   │   └── entity=products/
│   └── source=clickstream/
│       └── event=page_view/
│           └── year=2024/month=01/day=15/hour=14/
│
├── processed/                        ← Processed Zone (Vùng Đã Xử Lý)
│   ├── domain=customer/
│   │   ├── table=dim_customer/
│   │   │   ├── year=2024/month=01/
│   │   │   │   └── part-00000.parquet
│   │   └── table=fact_orders/
│   │       └── year=2024/month=01/day=15/
│   └── domain=product/
│       └── table=dim_product/
│
├── curated/                          ← Curated Zone (Vùng Đã Tinh Chỉnh)
│   ├── domain=finance/
│   │   ├── report=monthly_revenue/
│   │   │   └── year=2024/month=01/
│   │   └── report=customer_ltv/
│   └── domain=marketing/
│       └── report=campaign_performance/
│
├── sandbox/                          ← Sandbox Zone (Vùng Thử Nghiệm)
│   ├── user=alice/
│   └── user=bob/
│
└── archive/                          ← Archive Zone (Vùng Lưu Trữ)
    ├── raw/year=2020/
    └── raw/year=2021/
```

### Nguyên Tắc Đặt Tên (Naming Conventions)

#### ✅ Nên Làm

```
# Dùng lowercase (chữ thường) và dấu gạch ngang
s3://my-data-lake/raw/source=crm/entity=customers/

# Dùng hive-style partition (phân vùng kiểu Hive) cho Athena/Glue tự nhận diện
year=2024/month=01/day=15/

# Thêm timestamp vào filename để tránh xung đột
customers_20240115_143022.parquet

# Dùng tên có nghĩa, rõ ràng
source=payment-service/entity=transactions/
```

#### ❌ Tránh Làm

```
# Không dùng ký tự đặc biệt hoặc uppercase trong prefix
s3://MyDataLake/RAW/CRM Data/Customers 2024/

# Không để quá nhiều file nhỏ trong một partition
# (gây "small file problem" — vấn đề file nhỏ)
year=2024/month=01/day=15/file_001.parquet  ← 1KB mỗi file là vấn đề

# Không đặt partition có cardinality quá cao (nhiều giá trị riêng biệt)
# Ví dụ: partition theo user_id → hàng triệu partition
user_id=123456/   ← Tránh điều này
```

---

## 🗂️ Partitioning Strategy — Chiến Lược Phân Vùng

### Partitioning Là Gì?

Partitioning (Phân Vùng) là cách tổ chức dữ liệu vào các thư mục con trên S3, giúp:
- **Pruning** (Cắt Tỉa) — Athena/Redshift Spectrum chỉ đọc partition cần thiết
- **Giảm chi phí** — Ít dữ liệu quét = ít tiền trả cho Athena
- **Tăng tốc query** — Nhỏ hơn dữ liệu đọc = kết quả nhanh hơn

### Các Chiến Lược Phân Vùng

#### 1. Time-based Partitioning (Phân Vùng Theo Thời Gian) — Phổ Biến Nhất

```sql
-- Partition theo year/month/day — phù hợp cho log, events, transactions
year=2024/month=01/day=15/

-- Query chỉ đọc dữ liệu của ngày 15/01/2024
SELECT * FROM orders
WHERE year='2024' AND month='01' AND day='15'
-- Athena chỉ đọc dữ liệu trong thư mục year=2024/month=01/day=15/
```

**Khi nào dùng:** Events, logs, transactions — bất kỳ dữ liệu nào có tính chất time-series.

#### 2. Category-based Partitioning (Phân Vùng Theo Danh Mục)

```
-- Partition theo region và product_category
region=us-east-1/category=electronics/
region=us-west-2/category=books/
```

**Khi nào dùng:** Khi query thường xuyên filter theo một category cụ thể.

#### 3. Hybrid Partitioning (Phân Vùng Kết Hợp)

```
-- Kết hợp time + category
year=2024/month=01/region=us-east-1/
```

**Khi nào dùng:** Khi query pattern kết hợp cả thời gian và category.

### Partition Projection — Bỏ Qua Cần Tạo Partition Thủ Công

Thay vì chạy `MSCK REPAIR TABLE` mỗi khi có partition mới, dùng **Partition Projection** (Chiếu Phân Vùng) trong Athena:

```sql
-- Tạo bảng với partition projection
CREATE EXTERNAL TABLE logs (
  timestamp string,
  message string
)
PARTITIONED BY (year string, month string, day string)
LOCATION 's3://my-bucket/logs/'
TBLPROPERTIES (
  'projection.enabled' = 'true',
  'projection.year.type' = 'integer',
  'projection.year.range' = '2020,2030',
  'projection.month.type' = 'integer',
  'projection.month.range' = '1,12',
  'projection.month.digits' = '2',
  'projection.day.type' = 'integer',
  'projection.day.range' = '1,31',
  'projection.day.digits' = '2',
  'storage.location.template' =
    's3://my-bucket/logs/year=${year}/month=${month}/day=${day}/'
);
-- Athena tự tính toán partition locations mà không cần metadata catalog
```

---

## 📐 Data Lake Design Principles — Nguyên Tắc Thiết Kế

### 1. Immutability (Tính Bất Biến) trong Raw Zone

```
Nguyên tắc: Dữ liệu trong Raw Zone KHÔNG BAO GIỜ bị sửa đổi hay xóa.

Lý do:
- Cho phép re-process (xử lý lại) khi logic ETL thay đổi
- Audit trail (dấu vết kiểm toán) đầy đủ
- Recovery (khôi phục) khi pipeline bị lỗi
- Compliance (tuân thủ) với yêu cầu lưu trữ dữ liệu gốc

Cách thực hiện:
- Bật S3 Object Lock (Khóa Đối Tượng S3) chế độ COMPLIANCE
- Dùng S3 Versioning (Phiên Bản Hóa)
- IAM policy: PutObject được phép, DeleteObject bị cấm với Raw Zone
```

### 2. Schema Evolution (Tiến Hóa Schema)

Dữ liệu thực tế sẽ thay đổi schema theo thời gian. Dùng format hỗ trợ schema evolution:

| Format | Schema Evolution | Hiệu Suất | Dùng Khi |
| ------ | ---------------- | --------- | -------- |
| **Apache Parquet** | Thêm cột được, xóa/đổi tên khó | Rất tốt | Batch analytics |
| **Apache ORC** | Tương tự Parquet | Rất tốt | Hive/Redshift |
| **Apache Avro** | Tốt (forward + backward compat) | Trung bình | Streaming, Kafka |
| **Apache Iceberg** | Xuất sắc (full schema evolution) | Rất tốt | Modern lakehouse |
| **Delta Lake** | Xuất sắc | Rất tốt | Databricks/EMR |

### 3. Data Catalog First — Catalog Trước Tiên

```
Mọi dataset trong data lake đều phải được đăng ký trong Glue Data Catalog:
- Tên bảng, schema, location rõ ràng
- Phân loại format đúng (parquet/json/csv)
- Tags đầy đủ (owner, domain, classification, pii=true/false)
- Description cho từng cột quan trọng
```

### 4. Cost-Aware Design (Thiết Kế Ý Thức Chi Phí)

```
Chi phí S3 = Storage + Requests + Data Transfer

Tối ưu:
- Compress dữ liệu (Snappy/GZIP cho Parquet)
- Dùng đúng storage class (Intelligent-Tiering cho dữ liệu không rõ access pattern)
- Lifecycle policies: Raw Zone → Glacier sau 90 ngày
- Compact small files thành file lớn hơn (giảm request overhead)
```

---

## 🎯 Sizing Guidelines — Hướng Dẫn Kích Thước

### File Size Recommendations (Khuyến Nghị Kích Thước File)

```
Target file size: 128MB - 1GB (tối ưu cho Athena/Spark)

Quá nhỏ (< 10MB):
- Nhiều file → nhiều S3 GET requests → chậm và tốn tiền
- Giải pháp: Glue Job compact file hoặc dùng OPTIMIZE trong Iceberg

Quá lớn (> 5GB):
- Khó đọc song song (parallelism thấp)
- Giải pháp: Chia nhỏ hoặc partition tốt hơn
```

### Compaction Pattern (Mẫu Nén File)

```python
# Glue Job nén file nhỏ thành file lớn
import boto3
from awsglue.context import GlueContext
from pyspark.context import SparkContext

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session

# Đọc partition chứa nhiều file nhỏ
df = spark.read.parquet("s3://my-lake/processed/domain=events/year=2024/month=01/day=15/")

# Repartition (Phân Chia Lại) thành số file phù hợp
# Rule of thumb: 1 file per 128MB data
df.repartition(4).write \
  .mode("overwrite") \
  .parquet("s3://my-lake/processed/domain=events/year=2024/month=01/day=15/")
```

---

## 🏛️ Data Lake vs Data Lakehouse

Xu hướng hiện đại là **Data Lakehouse** (Kho-Hồ Dữ Liệu Kết Hợp) — kết hợp tính linh hoạt của data lake với khả năng ACID transaction của data warehouse:

```
Data Lake (Hồ Dữ Liệu Truyền Thống):
  + Lưu trữ mọi loại dữ liệu
  + Chi phí thấp
  - Không có ACID transactions
  - Khó quản lý schema
  - Không hỗ trợ UPDATE/DELETE hiệu quả

Data Lakehouse với Apache Iceberg / Delta Lake:
  + Lưu trữ mọi loại dữ liệu
  + Chi phí thấp
  + ACID transactions ✅
  + Schema evolution tốt ✅
  + Time travel (xem dữ liệu tại thời điểm cụ thể) ✅
  + Efficient UPDATE/DELETE ✅
```

### Apache Iceberg trên AWS

```sql
-- Tạo Iceberg table qua Athena
CREATE TABLE my_iceberg_table (
  id bigint,
  name string,
  email string,
  created_at timestamp
)
PARTITIONED BY (year(created_at))
LOCATION 's3://my-lake/processed/domain=customer/table=customers/'
TBLPROPERTIES (
  'table_type' = 'ICEBERG',
  'format' = 'parquet'
);

-- UPDATE — không thể với Parquet thông thường!
UPDATE my_iceberg_table
SET email = 'new@example.com'
WHERE id = 12345;

-- Time Travel (Du Hành Thời Gian) — xem dữ liệu lúc trước
SELECT * FROM my_iceberg_table
FOR TIMESTAMP AS OF TIMESTAMP '2024-01-01 00:00:00';
```

---

## ❓ Câu Hỏi Phỏng Vấn

**Q: Tại sao phải tổ chức data lake thành nhiều zone thay vì lưu tất cả một nơi?**

> Zone architecture giải quyết ba vấn đề chính: (1) **Data quality** — raw zone giữ nguyên dữ liệu gốc, processed zone đảm bảo chất lượng; (2) **Access control** — mỗi zone có quyền truy cập khác nhau, data scientist có thể sandbox trong khi BI chỉ thấy curated; (3) **Cost optimization** — có thể áp dụng lifecycle policies khác nhau cho từng zone.

**Q: Small file problem (vấn đề file nhỏ) trong data lake là gì và cách giải quyết?**

> Khi có hàng nghìn file nhỏ (dưới 10MB), mỗi Athena query phải tạo hàng nghìn S3 GET request, làm chậm query và tăng chi phí. Giải pháp: (1) Dùng Glue Job compaction chạy định kỳ; (2) Dùng Kinesis Firehose với buffer size lớn hơn khi ingest; (3) Dùng Apache Iceberg với lệnh `OPTIMIZE` để tự động compact.

**Q: Partition projection (chiếu phân vùng) trong Athena giải quyết vấn đề gì?**

> Bình thường, Athena phải lookup Glue Catalog để biết partition nào tồn tại (gọi là partition metadata request). Với dữ liệu mới được thêm vào S3 nhưng chưa chạy `MSCK REPAIR TABLE`, Athena không thấy partition mới. Partition Projection cho phép Athena tự tính toán partition locations dựa trên rule (ví dụ: year từ 2020-2030, month từ 1-12) — không cần metadata catalog, không bao giờ bị "missing partition".

---

**Tiếp Theo:** [2-lake-formation-security.md](./2-lake-formation-security.md) — Bảo mật và kiểm soát truy cập trong Lake Formation

**Cập Nhật Lần Cuối:** 2026-05-17
