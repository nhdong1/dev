# 🔧 AWS Glue — ETL & Data Catalog (Chuyển Đổi Dữ Liệu & Danh Mục Dữ Liệu)

> AWS Glue là dịch vụ ETL (Extract, Transform, Load — Trích Xuất, Chuyển Đổi, Tải) hoàn toàn serverless (không máy chủ) của AWS, tích hợp Data Catalog (Danh Mục Dữ Liệu) trung tâm, Crawlers (Trình Thu Thập Tự Động) và Data Quality (Chất Lượng Dữ Liệu) — là xương sống của mọi data lake (hồ dữ liệu) hiện đại trên AWS.

## 📚 Mục Lục Module

1. [Tổng Quan AWS Glue](#tổng-quan-aws-glue)
2. [Các Thành Phần Chính](#các-thành-phần-chính)
3. [Khi Nào Dùng Glue](#khi-nào-dùng-glue)
4. [Kiến Trúc Tổng Thể](#kiến-trúc-tổng-thể)
5. [Các File Trong Module](#các-file-trong-module)
6. [So Sánh Với Dịch Vụ Khác](#so-sánh-với-dịch-vụ-khác)
7. [Câu Hỏi Phỏng Vấn Nhanh](#câu-hỏi-phỏng-vấn-nhanh)

---

## 🎯 Tổng Quan AWS Glue

**AWS Glue** là dịch vụ **serverless data integration** (tích hợp dữ liệu không máy chủ) cho phép bạn:

- **Khám phá** (discover) dữ liệu phân tán qua Crawlers
- **Lưu trữ metadata** (siêu dữ liệu) tập trung trong Data Catalog
- **Chuyển đổi** (transform) dữ liệu với PySpark, Python Shell, hoặc Ray
- **Kiểm tra chất lượng** dữ liệu với Data Quality

### Tại Sao Glue Quan Trọng?

| Bài Toán Thực Tế                              | Giải Pháp Glue                                 |
| --------------------------------------------- | ----------------------------------------------- |
| Dữ liệu nằm rải rác S3, RDS, DynamoDB        | Crawler tự động phát hiện và catalog hóa        |
| Cần convert CSV → Parquet để Athena query nhanh | Glue ETL Job transform tự động                 |
| Athena, Redshift, EMR cần biết schema dữ liệu | Dùng chung Glue Data Catalog                   |
| Dữ liệu đến hàng ngày, schema thay đổi       | Crawler cập nhật schema tự động theo lịch       |
| Cần kiểm tra dữ liệu hợp lệ trước khi load   | Glue Data Quality validation rules              |

### Vị Trí Glue Trong Hệ Sinh Thái AWS Analytics

```
┌─────────────────────────────────────────────────────────────────┐
│                     DATA SOURCES (Nguồn Dữ Liệu)                │
│  S3 │ RDS │ DynamoDB │ Redshift │ JDBC (MySQL/PostgreSQL) │ ...  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      AWS GLUE                                    │
│                                                                  │
│  ┌─────────────────┐    ┌──────────────────┐                    │
│  │   CRAWLERS      │───▶│  DATA CATALOG    │◀──────────────┐   │
│  │ (Trình Thu Thập)│    │ (Danh Mục Dữ Liệu│               │   │
│  └─────────────────┘    │  Databases,      │               │   │
│                          │  Tables,         │               │   │
│  ┌─────────────────┐    │  Partitions,     │               │   │
│  │   ETL JOBS      │───▶│  Schema,         │               │   │
│  │ (Công Việc ETL) │    │  Statistics)     │               │   │
│  │  PySpark / Shell│    └──────────────────┘               │   │
│  └─────────────────┘                                        │   │
│                                                             │   │
│  ┌─────────────────┐    ┌──────────────────┐               │   │
│  │  DATA QUALITY   │    │  GLUE STUDIO     │───────────────┘   │
│  │ (Chất Lượng DL) │    │ (Visual ETL UI)  │                   │
│  └─────────────────┘    └──────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
                             │
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
       ┌─────────┐    ┌──────────┐    ┌───────────┐
       │ Athena  │    │ Redshift │    │    EMR    │
       │(Query)  │    │(Warehouse│    │ (Big Data)│
       └─────────┘    └──────────┘    └───────────┘
```

---

## 🗂️ Các Thành Phần Chính

### 1. AWS Glue Data Catalog — Danh Mục Dữ Liệu

**Là gì:** Kho lưu trữ metadata (siêu dữ liệu) trung tâm, tương tự Apache Hive Metastore.

**Bao gồm:**
- **Databases** (Cơ Sở Dữ Liệu): Nhóm logic các bảng liên quan
- **Tables** (Bảng): Định nghĩa schema, location (vị trí), format của dữ liệu
- **Partitions** (Phân Vùng): Thông tin phân vùng để tăng tốc query
- **Connections** (Kết Nối): Thông tin kết nối đến JDBC/ODBC sources

```
Glue Data Catalog
└── Database: analytics_db
    ├── Table: raw_events          → s3://bucket/raw/events/   (JSON)
    ├── Table: processed_events    → s3://bucket/proc/events/  (Parquet)
    └── Table: user_profiles       → s3://bucket/users/        (Parquet)
        └── Partitions: year=2024/month=01/day=01/...
```

**Tích hợp:** Athena, Redshift Spectrum, EMR đều đọc metadata từ Glue Catalog — một nơi quản lý schema cho toàn bộ hệ sinh thái.

---

### 2. AWS Glue Crawlers — Trình Thu Thập Tự Động

**Là gì:** Chương trình tự động kết nối đến data store, đọc dữ liệu và suy luận schema, sau đó cập nhật vào Glue Data Catalog.

```
Crawler chạy
    → Kết nối đến S3/RDS/DynamoDB/...
    → Đọc mẫu dữ liệu (sample files)
    → Suy luận schema (column names, data types)
    → Phát hiện partitions
    → Ghi vào Glue Data Catalog
```

**Lịch chạy:** Theo yêu cầu (on-demand), theo lịch (scheduled cron), hoặc theo sự kiện (event-driven).

---

### 3. AWS Glue ETL Jobs — Công Việc ETL

**Là gì:** Script chạy trên Apache Spark (hoặc Python Shell/Ray) để transform dữ liệu.

**Các loại job:**

| Loại Job       | Engine         | Tốt Cho                                        | DPU Tối Thiểu |
| --------------- | -------------- | ---------------------------------------------- | -------------- |
| **Spark**       | Apache Spark   | Xử lý dữ liệu lớn, transformation phức tạp    | 2 DPU          |
| **Python Shell**| Python 3       | Script nhỏ, API calls, file processing nhẹ     | 0.0625 DPU     |
| **Streaming**   | Spark Streaming| Xử lý dữ liệu từ Kinesis/Kafka liên tục        | 2 DPU          |
| **Ray**         | Ray framework  | ML preprocessing, distributed Python           | 2 DPU          |

**DPU — Data Processing Unit (Đơn Vị Xử Lý Dữ Liệu):** Đơn vị tính tài nguyên Glue (1 DPU = 4 vCPU + 16 GB RAM).

---

### 4. AWS Glue Data Quality — Chất Lượng Dữ Liệu

**Là gì:** Tính năng kiểm tra chất lượng dữ liệu với quy tắc (rules) tự động hoặc tùy chỉnh.

**Khả năng:**
- **Profiling** (Phân Tích Hồ Sơ): Thống kê tự động về dữ liệu (null %, unique count, min/max)
- **Rules** (Quy Tắc): Định nghĩa điều kiện dữ liệu phải thỏa mãn
- **Recommendations** (Gợi Ý): Glue tự đề xuất rules dựa trên phân tích dữ liệu

---

## 🎯 Khi Nào Dùng Glue

### ✅ Chọn Glue Khi

- Cần ETL serverless không muốn quản lý cluster
- Dùng S3 làm data lake — cần catalog hóa dữ liệu cho Athena/Redshift
- Cần tự động phát hiện schema khi dữ liệu thay đổi cấu trúc
- Team chủ yếu dùng Python/PySpark
- Workload ETL không liên tục (chạy vài lần/ngày)

### ❌ Xem Xét Thay Thế Khi

| Tình Huống                               | Thay Thế Tốt Hơn         |
| ----------------------------------------- | -------------------------- |
| ETL cực phức tạp, cần tùy chỉnh cluster | EMR + Spark               |
| Streaming ETL liên tục, độ trễ thấp      | Kinesis + Lambda           |
| Transformation đơn giản, không cần Spark | Lambda hoặc Python script  |
| Workflow orchestration (Điều Phối)        | MWAA (Apache Airflow)      |

---

## 🏗️ Kiến Trúc Tổng Thể

### Pattern 1: Batch ETL Pipeline (Đường Ống ETL Theo Lô)

```
S3 Raw Data
    │
    ▼
Glue Crawler (hàng đêm)
    │ → cập nhật schema vào Data Catalog
    ▼
Glue ETL Job (PySpark)
    │ → đọc JSON từ S3 raw
    │ → transform, clean, enrich
    │ → ghi Parquet vào S3 processed
    ▼
Glue Crawler (sau ETL)
    │ → catalog hóa dữ liệu processed
    ▼
Athena hoặc Redshift Spectrum
    → Analysts query dữ liệu sạch
```

### Pattern 2: CDC Pipeline (Change Data Capture — Thu Thập Thay Đổi Dữ Liệu)

```
RDS / Aurora (production DB)
    │
    ▼ (via DMS — Database Migration Service)
S3 (CDC files)
    │
    ▼
Glue ETL Job (Spark)
    │ → merge changes vào data lake
    │ → upsert (insert + update) records
    ▼
S3 Processed (Parquet / Iceberg)
```

### Pattern 3: Streaming ETL (ETL Dữ Liệu Luồng)

```
Kinesis Data Streams
    │
    ▼
Glue Streaming Job (Spark Streaming)
    │ → micro-batch processing
    │ → enrichment với lookup tables
    ▼
S3 / Redshift / OpenSearch
```

---

## 📁 Các File Trong Module

| File                     | Nội Dung                                                     |
| ------------------------- | ------------------------------------------------------------ |
| `README.md`              | File này — tổng quan module Glue                             |
| `1-glue-catalog.md`      | Data Catalog — cấu trúc, quản lý, tích hợp với Athena/Redshift/EMR |
| `2-glue-etl-jobs.md`     | ETL Jobs — PySpark, Python Shell, Ray, DPU, tối ưu chi phí   |
| `3-glue-crawlers.md`     | Crawlers — cấu hình, lịch chạy, xử lý schema evolution       |
| `4-glue-data-quality.md` | Data Quality — profiling, rules, validation, monitoring      |

---

## ⚖️ So Sánh Với Dịch Vụ Khác

### Glue vs EMR

| Tiêu Chí                  | AWS Glue                      | Amazon EMR                       |
| -------------------------- | ----------------------------- | -------------------------------- |
| **Quản lý cluster**        | Serverless, AWS quản lý       | Bạn quản lý cluster              |
| **Khởi động**              | Chậm hơn (cold start ~2 phút) | Nhanh hơn sau khi cluster sẵn   |
| **Tùy chỉnh**              | Giới hạn (chỉ Spark/Python)  | Đầy đủ (Spark, Hive, Presto...) |
| **Chi phí khi idle**       | $0 — trả theo use            | Trả cho cluster đang chạy       |
| **Data Catalog**           | Tích hợp sẵn                 | Dùng chung Glue Catalog          |
| **Tốt cho**                | ETL không liên tục            | Xử lý lớn liên tục, ML          |

### Glue vs Lambda (ETL nhẹ)

| Tiêu Chí              | AWS Glue                    | AWS Lambda                       |
| ---------------------- | --------------------------- | -------------------------------- |
| **Thời gian chạy max** | Không giới hạn              | 15 phút                          |
| **Memory max**         | Phụ thuộc DPU               | 10 GB                            |
| **Tốt cho**            | Xử lý GB → TB dữ liệu      | Xử lý MB, event-driven nhỏ      |
| **PySpark support**    | ✅ Native                  | ❌ Phải tự cài                  |

---

## ❓ Câu Hỏi Phỏng Vấn Nhanh

**Q: Glue Data Catalog là gì và tại sao quan trọng?**
> Là centralized metadata repository (kho siêu dữ liệu trung tâm) lưu schema, location, partition info của dữ liệu. Quan trọng vì Athena, Redshift Spectrum, EMR đều dùng chung Catalog — giúp định nghĩa schema một lần, dùng ở nhiều nơi.

**Q: Crawler khác gì với manual table definition?**
> Crawler tự động kết nối đến data source, suy luận schema và phát hiện partitions — không cần viết DDL thủ công. Hữu ích khi schema thường xuyên thay đổi hoặc có nhiều bảng cần catalog hóa. Manual definition phù hợp khi schema cố định và cần kiểm soát chính xác.

**Q: DPU là gì? Cách tối ưu chi phí Glue?**
> DPU (Data Processing Unit — Đơn Vị Xử Lý Dữ Liệu) = 4 vCPU + 16 GB RAM, tính phí theo giây. Tối ưu bằng: dùng Python Shell job cho task nhỏ (0.0625 DPU thay vì 2 DPU), bật Job Bookmarks để không xử lý lại dữ liệu cũ, dùng Glue Flex execution cho workload không khẩn cấp (giảm 34% chi phí).

**Q: Glue vs EMR — chọn cái nào?**
> Glue cho ETL serverless, không muốn quản lý infrastructure, workload không liên tục. EMR khi cần toàn quyền tùy chỉnh Spark, xử lý dữ liệu khổng lồ liên tục, hoặc dùng framework ngoài Spark (Hive, Presto, HBase).

**Q: Schema Evolution (Tiến Hóa Schema) trong Glue hoạt động thế nào?**
> Khi schema thay đổi (thêm/đổi column), Crawler có thể: (1) Cập nhật table definition với column mới, (2) Tạo table mới (nếu schema thay đổi nhiều), (3) Đánh dấu column bị xóa. ETL Job với DynamicFrame (Khung Dữ Liệu Động) xử lý linh hoạt hơn DataFrame vì không enforce schema cứng nhắc.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Module:** 03-glue | **Phụ Thuộc:** 01-fundamentals (ETL/ELT concepts)
