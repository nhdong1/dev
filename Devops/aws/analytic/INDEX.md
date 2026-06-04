# AWS Analytics Services — Chỉ Mục Đầy Đủ

> Hướng dẫn toàn diện về AWS Analytics Services — từ nền tảng đến kiến trúc dữ liệu nâng cao

## 📁 Cấu Trúc Thư Mục

```
Devops/aws/analytic/
├── README.md                               [BẮT ĐẦU TẠI ĐÂY] Lộ trình học & tổng quan
├── INDEX.md                                Chỉ mục đầy đủ (file này)
│
├── 01-fundamentals/
│   ├── README.md                           Nền tảng analytics, khái niệm cốt lõi
│   ├── 1-analytics-overview.md            (Cần tạo) Các loại analytics: Descriptive/Diagnostic/Predictive
│   ├── 2-data-pipeline-concepts.md        (Cần tạo) Ingestion, Processing, Storage, Serving
│   ├── 3-batch-vs-streaming.md            (Cần tạo) Xử lý theo lô vs luồng thời gian thực
│   ├── 4-data-lake-vs-warehouse.md        (Cần tạo) Hồ dữ liệu vs kho dữ liệu vs lakehouse
│   └── 5-etl-elt-concepts.md             (Cần tạo) ETL vs ELT — khi nào dùng cái nào
│
├── 02-kinesis/
│   ├── README.md                           Tổng quan Kinesis — real-time streaming
│   ├── 1-kinesis-data-streams.md          (Cần tạo) KDS — Shards, Consumers, Retention
│   ├── 2-kinesis-firehose.md              (Cần tạo) KDF — Delivery đến S3/Redshift/OpenSearch
│   ├── 3-kinesis-analytics.md             (Cần tạo) KDA — SQL/Flink trên streaming data
│   └── 4-kinesis-vs-kafka.md             (Cần tạo) So sánh Kinesis và MSK/Kafka
│
├── 03-glue/
│   ├── README.md                           Tổng quan Glue — ETL & Data Catalog
│   ├── 1-glue-catalog.md                  Data Catalog — metadata repository, tích hợp Athena/Redshift/EMR
│   ├── 2-glue-etl-jobs.md                 ETL Jobs — PySpark, Python Shell, Ray, DPU, tối ưu chi phí
│   ├── 3-glue-crawlers.md                 Crawlers — tự động phát hiện schema, partition discovery
│   └── 4-glue-data-quality.md            Data Quality — DQDL, profiling, validation, quarantine pattern
│
├── 04-athena/
│   ├── README.md                           Tổng quan Athena — serverless SQL query
│   ├── 1-athena-fundamentals.md           (Cần tạo) Presto engine, S3 integration, setup
│   ├── 2-athena-performance.md            (Cần tạo) Parquet, ORC, partitioning, compression
│   ├── 3-athena-federation.md             (Cần tạo) Federated Query — query nhiều data source
│   └── 4-athena-cost-optimization.md     (Cần tạo) Giảm dữ liệu quét, workgroup budgets
│
├── 05-redshift/
│   ├── README.md                           Tổng quan Redshift — MPP data warehouse
│   ├── 1-redshift-architecture.md         (Cần tạo) Leader node, Compute nodes, MPP
│   ├── 2-redshift-performance.md          (Cần tạo) DISTKEY, SORTKEY, DISTSTYLE, WLM
│   ├── 3-redshift-spectrum.md             (Cần tạo) Query S3 trực tiếp từ Redshift
│   └── 4-redshift-serverless.md          (Cần tạo) Auto capacity, RPU, use cases
│
├── 06-emr/
│   ├── README.md                           Tổng quan EMR — big data processing, deploy modes
│   ├── 1-emr-architecture.md              Master/Core/Task nodes, cluster lifecycle, HA, storage
│   ├── 2-emr-spark.md                     Apache Spark trên EMR, AQE, memory tuning, Glue Catalog
│   ├── 3-emr-serverless.md                EMR Serverless — không quản lý cluster, pre-initialized capacity
│   └── 4-emr-cost-optimization.md        Spot Instances, Instance Fleets, transient cluster, sizing
│
├── 07-lake-formation/
│   ├── README.md                           Tổng quan Lake Formation — data lake governance
│   ├── 1-data-lake-design.md              Zone architecture, folder structure, partitioning strategy
│   ├── 2-lake-formation-security.md       Column/row-level, LF-Tags, cross-account sharing
│   └── 3-s3-data-lake.md                 S3 lifecycle, Intelligent-Tiering, storage classes, tối ưu chi phí
│
├── 08-quicksight/
│   ├── README.md                           Tổng quan QuickSight — BI & visualization
│   ├── 1-quicksight-basics.md             (Cần tạo) Datasets, analyses, dashboards
│   ├── 2-spice-engine.md                  (Cần tạo) SPICE — in-memory calculation engine
│   └── 3-embedded-analytics.md           (Cần tạo) Tích hợp analytics vào ứng dụng
│
├── 09-opensearch/
│   ├── README.md                           Tổng quan OpenSearch — search & log analytics
│   ├── 1-opensearch-fundamentals.md       Clusters, Indices, Shards, Replicas, Query DSL
│   ├── 2-opensearch-ingestion.md          Kinesis Firehose, Lambda, Logstash, Fluent Bit, OSI
│   └── 3-opensearch-security.md          Fine-grained access control, Encryption, SAML, Audit
│
├── 10-msk/
│   ├── README.md                           Tổng quan MSK — Managed Kafka
│   ├── 1-msk-architecture.md              (Cần tạo) Brokers, Topics, Partitions, Consumer Groups
│   ├── 2-msk-vs-kinesis.md                (Cần tạo) So sánh chi tiết, khi nào chọn gì
│   └── 3-msk-security.md                 (Cần tạo) TLS, SASL/SCRAM, IAM authentication
│
├── 11-data-architecture/
│   ├── README.md                           Kiến trúc dữ liệu hiện đại
│   ├── 1-lambda-architecture.md           (Cần tạo) Batch + Speed + Serving layer
│   ├── 2-kappa-architecture.md            (Cần tạo) Stream-only, đơn giản hóa Lambda
│   ├── 3-data-mesh.md                     (Cần tạo) Domain ownership, self-serve platform
│   └── 4-medallion-architecture.md       (Cần tạo) Bronze, Silver, Gold layers
│
├── 12-interview-prep/
│   ├── README.md                           Tổng quan chuẩn bị phỏng vấn, lộ trình, checklist
│   ├── INTERVIEW_GUIDE.md                  Top 20 câu hỏi phỏng vấn + gợi ý trả lời chi tiết
│   └── system-design-scenarios.md         5 kịch bản thiết kế hệ thống: real-time, data lake, BI, log, multi-tenant
│
├── ROADMAP.md                              (Cần tạo) Lộ trình học chi tiết 90 ngày
├── GLOSSARY.md                             (Cần tạo) Thuật ngữ Analytics & AWS
└── RESOURCES.md                            (Cần tạo) Sách, blog, khóa học
```

---

## ✅ Đã Tạo

| Chủ Đề                                   | File                                           | Trạng Thái | Chất Lượng    |
| ---------------------------------------- | ---------------------------------------------- | ---------- | ------------- |
| **Tổng Quan & Lộ Trình**                 | README.md                                      | ✅         | Toàn diện     |
| **Chỉ Mục Đầy Đủ**                       | INDEX.md                                       | ✅         | Toàn diện     |
| **Nền Tảng — Mục Lục Module**            | 01-fundamentals/README.md                      | ✅         | Toàn diện     |
| **Tổng Quan Các Loại Analytics**         | 01-fundamentals/1-analytics-overview.md        | ✅         | Toàn diện     |
| **Khái Niệm Data Pipeline**              | 01-fundamentals/2-data-pipeline-concepts.md    | ✅         | Toàn diện     |
| **Batch vs Stream Processing**           | 01-fundamentals/3-batch-vs-streaming.md        | ✅         | Toàn diện     |
| **Data Lake vs Warehouse vs Lakehouse**  | 01-fundamentals/4-data-lake-vs-warehouse.md    | ✅         | Toàn diện     |
| **ETL vs ELT**                           | 01-fundamentals/5-etl-elt-concepts.md          | ✅         | Toàn diện     |
| **Kinesis — Tổng Quan Module**           | 02-kinesis/README.md                           | ✅         | Toàn diện     |
| **Kinesis Data Streams — KDS**           | 02-kinesis/1-kinesis-data-streams.md           | ✅         | Toàn diện     |
| **Kinesis Data Firehose — KDF**          | 02-kinesis/2-kinesis-firehose.md               | ✅         | Toàn diện     |
| **Kinesis Data Analytics — KDA/Flink**  | 02-kinesis/3-kinesis-analytics.md              | ✅         | Toàn diện     |
| **Kinesis vs Kafka / MSK**               | 02-kinesis/4-kinesis-vs-kafka.md               | ✅         | Toàn diện     |
| **Glue — Tổng Quan Module**              | 03-glue/README.md                              | ✅         | Toàn diện     |
| **Glue Data Catalog**                    | 03-glue/1-glue-catalog.md                      | ✅         | Toàn diện     |
| **Glue ETL Jobs**                        | 03-glue/2-glue-etl-jobs.md                     | ✅         | Toàn diện     |
| **Glue Crawlers**                        | 03-glue/3-glue-crawlers.md                     | ✅         | Toàn diện     |
| **Glue Data Quality**                    | 03-glue/4-glue-data-quality.md                 | ✅         | Toàn diện     |
| **Athena — Tổng Quan Module**            | 04-athena/README.md                            | ✅         | Toàn diện     |
| **Athena Fundamentals — Nền Tảng**       | 04-athena/1-athena-fundamentals.md             | ✅         | Toàn diện     |
| **Athena Performance Optimization**      | 04-athena/2-athena-performance.md              | ✅         | Toàn diện     |
| **Athena Federated Query**               | 04-athena/3-athena-federation.md               | ✅         | Toàn diện     |
| **Athena Cost Optimization**             | 04-athena/4-athena-cost-optimization.md        | ✅         | Toàn diện     |
| **Redshift — Tổng Quan Module**          | 05-redshift/README.md                          | ✅         | Toàn diện     |
| **Redshift Architecture — Kiến Trúc**   | 05-redshift/1-redshift-architecture.md         | ✅         | Toàn diện     |
| **Redshift Performance Optimization**    | 05-redshift/2-redshift-performance.md          | ✅         | Toàn diện     |
| **Redshift Spectrum — Query S3**         | 05-redshift/3-redshift-spectrum.md             | ✅         | Toàn diện     |
| **Redshift Serverless**                  | 05-redshift/4-redshift-serverless.md           | ✅         | Toàn diện     |
| **EMR — Tổng Quan Module**               | 06-emr/README.md                               | ✅         | Toàn diện     |
| **EMR Architecture — Kiến Trúc**        | 06-emr/1-emr-architecture.md                   | ✅         | Toàn diện     |
| **Apache Spark trên EMR**               | 06-emr/2-emr-spark.md                          | ✅         | Toàn diện     |
| **EMR Serverless**                      | 06-emr/3-emr-serverless.md                     | ✅         | Toàn diện     |
| **EMR Cost Optimization**               | 06-emr/4-emr-cost-optimization.md              | ✅         | Toàn diện     |
| **Lake Formation — Tổng Quan Module**   | 07-lake-formation/README.md                    | ✅         | Toàn diện     |
| **Data Lake Design — Thiết Kế**         | 07-lake-formation/1-data-lake-design.md        | ✅         | Toàn diện     |
| **Lake Formation Security**             | 07-lake-formation/2-lake-formation-security.md | ✅         | Toàn diện     |
| **S3 Data Lake**                        | 07-lake-formation/3-s3-data-lake.md            | ✅         | Toàn diện     |
| **QuickSight — Tổng Quan Module**       | 08-quicksight/README.md                        | ✅         | Toàn diện     |
| **QuickSight Basics — Dataset/Analysis**| 08-quicksight/1-quicksight-basics.md           | ✅         | Toàn diện     |
| **SPICE Engine — In-memory Cache**      | 08-quicksight/2-spice-engine.md                | ✅         | Toàn diện     |
| **Embedded Analytics**                  | 08-quicksight/3-embedded-analytics.md          | ✅         | Toàn diện     |
| **OpenSearch — Tổng Quan Module**       | 09-opensearch/README.md                        | ✅         | Toàn diện     |
| **OpenSearch Fundamentals — Nền Tảng** | 09-opensearch/1-opensearch-fundamentals.md     | ✅         | Toàn diện     |
| **OpenSearch Ingestion — Thu Nạp**      | 09-opensearch/2-opensearch-ingestion.md        | ✅         | Toàn diện     |
| **OpenSearch Security — Bảo Mật**      | 09-opensearch/3-opensearch-security.md         | ✅         | Toàn diện     |
| **MSK — Tổng Quan Module**              | 10-msk/README.md                               | ✅         | Toàn diện     |
| **MSK Architecture — Kiến Trúc**       | 10-msk/1-msk-architecture.md                  | ✅         | Toàn diện     |
| **MSK vs Kinesis — So Sánh**           | 10-msk/2-msk-vs-kinesis.md                    | ✅         | Toàn diện     |
| **MSK Security — Bảo Mật**            | 10-msk/3-msk-security.md                      | ✅         | Toàn diện     |
| **Data Architecture — Tổng Quan Module** | 11-data-architecture/README.md              | ✅         | Toàn diện     |
| **Lambda Architecture — Batch + Speed** | 11-data-architecture/1-lambda-architecture.md | ✅         | Toàn diện     |
| **Kappa Architecture — Stream-only**   | 11-data-architecture/2-kappa-architecture.md  | ✅         | Toàn diện     |
| **Data Mesh — Domain Ownership**       | 11-data-architecture/3-data-mesh.md           | ✅         | Toàn diện     |
| **Medallion Architecture — Bronze/Silver/Gold** | 11-data-architecture/4-medallion-architecture.md | ✅  | Toàn diện     |
| **Interview Prep — Tổng Quan Module**           | 12-interview-prep/README.md                       | ✅  | Toàn diện     |
| **Top 20 Câu Hỏi Phỏng Vấn AWS Analytics**     | 12-interview-prep/INTERVIEW_GUIDE.md              | ✅  | Toàn diện     |
| **System Design Scenarios — 5 Kịch Bản**        | 12-interview-prep/system-design-scenarios.md      | ✅  | Toàn diện     |

---

## 🎯 Cần Tạo Theo Thứ Tự Ưu Tiên

### Ưu Tiên Cao — Core Analytics Services

- [x] `01-fundamentals/README.md` — Nền tảng: Batch vs Streaming, Data Lake vs Warehouse ✅
- [x] `02-kinesis/README.md` — Kinesis Data Streams, Firehose, Analytics ✅
- [x] `03-glue/README.md` — Glue ETL, Data Catalog, Crawlers ✅
- [x] `04-athena/README.md` — Athena serverless query, tối ưu performance ✅
- [x] `05-redshift/README.md` — Redshift MPP, distribution strategies ✅
- [x] `12-interview-prep/INTERVIEW_GUIDE.md` — Top 20 câu hỏi phỏng vấn ✅

### Ưu Tiên Trung Bình — Advanced Services

- [x] `06-emr/README.md` — EMR, Spark, EMR Serverless ✅
- [x] `07-lake-formation/README.md` — Data Lake governance, security ✅
- [x] `11-data-architecture/README.md` — Lambda, Kappa, Medallion, Data Mesh ✅
- [x] `09-opensearch/README.md` — Search & log analytics ✅
- [ ] `ROADMAP.md` — Kế hoạch học 90 ngày chi tiết

### Ưu Tiên Thấp — Reference Materials

- [x] `08-quicksight/README.md` — BI & visualization ✅
- [x] `10-msk/README.md` — Managed Kafka ✅
- [x] `12-interview-prep/system-design-scenarios.md` — Kịch bản thiết kế hệ thống ✅
- [ ] `GLOSSARY.md` — Thuật ngữ
- [ ] `RESOURCES.md` — Tài liệu tham khảo

---

## 🚀 Cách Sử Dụng Knowledge Base Này

### Cho Tự Học

```
1. Đọc README.md để nắm tổng quan
2. Chọn lộ trình (Beginner / Intermediate / Advanced)
3. Học từng module theo thứ tự 01 → 12
4. Thực hành hands-on trên AWS (nhớ xóa resources sau khi xong)
5. Xây dựng mini project: end-to-end analytics pipeline
```

### Cho Chuẩn Bị Phỏng Vấn

```
1. Đọc 12-interview-prep/INTERVIEW_GUIDE.md trước
2. Tập trung vào dịch vụ target role yêu cầu:
   - Data Engineer: 02-kinesis, 03-glue, 04-athena, 05-redshift
   - Analytics Engineer: 04-athena, 05-redshift, 08-quicksight
   - Platform Engineer: 06-emr, 07-lake-formation, 11-data-architecture
3. Chuẩn bị câu chuyện pipeline design (STAR method)
4. Luyện giải thích trade-offs giữa các dịch vụ
```

### Cho Công Việc Thực Tế

```
Dùng như tài liệu tham khảo:
- Thiết kế pipeline mới: Xem 11-data-architecture/
- Streaming problem: Xem 02-kinesis/ hoặc 10-msk/
- ETL issue: Xem 03-glue/
- Query optimization: Xem 04-athena/ hoặc 05-redshift/
- Governance setup: Xem 07-lake-formation/
```

### Cho System Design

```
1. Xác định yêu cầu: Batch hay Streaming? Latency? Volume?
2. Đọc 11-data-architecture/ để chọn kiến trúc phù hợp
3. Chọn dịch vụ theo So Sánh Nhanh trong README.md
4. Thiết kế với nguyên tắc Well-Architected (bảo mật, reliability, cost)
```

---

## 📊 Ước Tính Thời Gian Học

| Module                          | Thời Gian   | Độ Khó | Ưu Tiên |
| ------------------------------- | ----------- | ------ | ------- |
| Fundamentals — Nền Tảng         | 3-4 giờ     | ⭐     | Phải học |
| Kinesis — Real-time Streaming   | 6-8 giờ     | ⭐⭐   | Phải học |
| Glue — ETL & Catalog            | 6-8 giờ     | ⭐⭐   | Phải học |
| Athena — Serverless Query       | 4-6 giờ     | ⭐⭐   | Phải học |
| Redshift — Data Warehouse       | 8-10 giờ    | ⭐⭐⭐ | Phải học |
| EMR — Big Data                  | 8-10 giờ    | ⭐⭐⭐ | Nên học  |
| Lake Formation — Governance     | 4-6 giờ     | ⭐⭐   | Nên học  |
| QuickSight — Visualization      | 3-4 giờ     | ⭐     | Nên học  |
| OpenSearch — Search Analytics   | 4-6 giờ     | ⭐⭐   | Nên học  |
| MSK — Managed Kafka             | 6-8 giờ     | ⭐⭐⭐ | Tùy chọn |
| Data Architecture Patterns      | 6-8 giờ     | ⭐⭐⭐ | Nên học  |

**Tổng: 60-90 giờ cho kiến thức AWS Analytics toàn diện**

---

## 🎓 Mức Độ Kỹ Năng Được Hỗ Trợ

### Người Mới — Beginner (0-1 năm kinh nghiệm)

- [ ] Phân biệt Data Lake và Data Warehouse
- [ ] Chạy query Athena cơ bản trên S3
- [ ] Hiểu ETL (Extract Transform Load) là gì
- [ ] Tạo Glue Crawler tự động phát hiện schema
- [ ] Biết Kinesis Firehose dùng để làm gì

**Thời gian đạt mức này:** 2-4 tuần

### Trung Cấp — Intermediate (1-3 năm kinh nghiệm)

- [ ] Thiết kế ETL pipeline Glue từ S3 sang Redshift
- [ ] Tối ưu Athena với Parquet và partition projection
- [ ] Xây dựng Kinesis consumer với Lambda
- [ ] Cấu hình Redshift DISTKEY và SORTKEY phù hợp
- [ ] Thiết lập Lake Formation column-level security

**Thời gian đạt mức này:** 2-3 tháng thực hành

### Nâng Cao — Advanced (3-5+ năm kinh nghiệm)

- [ ] Kiến trúc real-time analytics pipeline end-to-end
- [ ] Thiết kế Medallion Architecture với Apache Iceberg
- [ ] Tối ưu EMR Spark job với dynamic allocation và Spot
- [ ] Triển khai Data Mesh với Lake Formation
- [ ] Thiết kế multi-tenant analytics platform chi phí hiệu quả

**Thời gian đạt mức này:** Học liên tục, 6-12 tháng kinh nghiệm thực chiến

---

## 🔗 Điều Hướng Nhanh

| Cần Gì                             | Vị Trí                                                        |
| ---------------------------------- | ------------------------------------------------------------- |
| Tổng quan nhanh                    | [README.md](README.md)                                        |
| Nền tảng analytics                 | [01-fundamentals/README.md](01-fundamentals/README.md)        |
| Real-time streaming                | [02-kinesis/README.md](02-kinesis/README.md)                  |
| ETL & Data Catalog                 | [03-glue/README.md](03-glue/README.md)                        |
| Serverless SQL query               | [04-athena/README.md](04-athena/README.md)                    |
| Data Warehouse                     | [05-redshift/README.md](05-redshift/README.md)                |
| Big Data processing                | [06-emr/README.md](06-emr/README.md)                         |
| Data Lake governance               | [07-lake-formation/README.md](07-lake-formation/README.md)    |
| BI & Visualization                 | [08-quicksight/README.md](08-quicksight/README.md)            |
| Search & Log Analytics             | [09-opensearch/README.md](09-opensearch/README.md)            |
| Managed Kafka                      | [10-msk/README.md](10-msk/README.md)                         |
| Kiến trúc dữ liệu hiện đại         | [11-data-architecture/README.md](11-data-architecture/README.md) |
| Câu hỏi phỏng vấn                  | [12-interview-prep/INTERVIEW_GUIDE.md](12-interview-prep/INTERVIEW_GUIDE.md) |

---

## 📈 Theo Dõi Tiến Độ Học Tập

Sao chép và theo dõi tiến độ của bạn:

```markdown
## AWS Analytics — Tiến Độ Của Tôi

### Giai Đoạn 1: Nền Tảng (Tuần 1-2)
- [ ] Batch vs Streaming processing
- [ ] Data Lake vs Data Warehouse vs Lakehouse
- [ ] ETL vs ELT
- [ ] Data pipeline concepts
- [ ] S3 là nền tảng cho analytics AWS

### Giai Đoạn 2: Dịch Vụ Cốt Lõi (Tuần 3-6)
- [ ] Kinesis Data Streams — KDS
- [ ] Kinesis Data Firehose — KDF
- [ ] AWS Glue Data Catalog
- [ ] AWS Glue ETL Jobs
- [ ] Amazon Athena — serverless query
- [ ] Amazon Redshift — data warehouse

### Giai Đoạn 3: Nâng Cao (Tuần 7-10)
- [ ] Amazon EMR + Spark
- [ ] AWS Lake Formation governance
- [ ] Amazon OpenSearch
- [ ] Amazon MSK (Kafka)
- [ ] Kiến trúc Lambda / Kappa / Medallion

### Giai Đoạn 4: Chuyên Sâu (Tuần 11+)
- [ ] Data Mesh implementation
- [ ] End-to-end real-time pipeline
- [ ] Cost optimization strategies
- [ ] Mock system design interviews
- [ ] Hands-on portfolio project
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này, bạn phải có khả năng:

### ✅ Năng Lực Nền Tảng

- [ ] Giải thích Data Lake vs Data Warehouse mà không cần nhìn tài liệu
- [ ] Thiết kế data pipeline đơn giản cho bài toán thực tế
- [ ] Chọn đúng dịch vụ AWS analytics cho từng use case
- [ ] Ước tính chi phí cho một analytics workload

### ✅ Năng Lực Vận Hành

- [ ] Xử lý sự cố Kinesis throttling (Hạn Chế Tốc Độ)
- [ ] Tối ưu Glue Job bị chậm hoặc tốn nhiều DPU
- [ ] Debug Athena query trả về kết quả sai hoặc chậm
- [ ] Giải quyết Redshift query performance issues

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời Top 20 câu hỏi phỏng vấn AWS Analytics tự tin
- [ ] Thiết kế real-time pipeline khi được hỏi trong vòng 30 phút
- [ ] Giải thích trade-offs giữa các dịch vụ (Kinesis vs MSK, Athena vs Redshift)
- [ ] Biết ít nhất 2 câu chuyện dự án thực tế (STAR format)

---

## 🚀 Bước Tiếp Theo

### Ngay Lập Tức (Tuần Này)

1. Đọc README.md đầy đủ để nắm toàn cảnh
2. Chọn lộ trình học theo vai trò mục tiêu
3. Tạo AWS Free Tier account nếu chưa có
4. Thử chạy Athena query với AWS public datasets

### Ngắn Hạn (2-4 Tuần Tới)

1. Hoàn thành 01-fundamentals/ để vững nền tảng
2. Học 02-kinesis/ và thực hành gửi/nhận event
3. Học 03-glue/ và tạo Crawler + ETL job đơn giản
4. Học 04-athena/ và tối ưu query với Parquet

### Trung Hạn (1-3 Tháng)

1. Hoàn thành tất cả core modules (01-07)
2. Xây dựng portfolio project: mini data pipeline end-to-end
3. Học kiến trúc patterns trong 11-data-architecture/
4. Bắt đầu luyện câu hỏi phỏng vấn

### Dài Hạn (3-6 Tháng)

1. Nắm vững toàn bộ AWS Analytics ecosystem
2. Thực hành trên dữ liệu thực tế (tham gia project thực tế)
3. Lấy chứng chỉ AWS Certified Data Analytics — Specialty
4. Đóng góp vào open-source data tools (Airflow, dbt, Spark)

---

## 💡 Mẹo Thực Hành

1. **Dùng AWS Free Tier thông minh:** Athena tính phí theo dữ liệu quét — dùng Parquet và partition để giảm chi phí
2. **Xóa resources sau thực hành:** EMR cluster và Redshift cluster tính phí theo giờ — tắt khi không dùng
3. **Dùng Public Datasets:** AWS cung cấp nhiều public dataset miễn phí (NOAA, Wikipedia, NYC Taxi) để thực hành Athena
4. **Học bằng cách làm:** Đừng chỉ đọc — hãy thực sự chạy pipeline
5. **Hiểu chi phí:** Mỗi dịch vụ có mô hình tính phí khác nhau — nắm rõ để không bị "shock bill"
6. **Kết hợp dịch vụ:** AWS analytics services mạnh nhất khi kết hợp với nhau (S3 + Glue + Athena là bộ ba kinh điển)
7. **Kiểm tra Well-Architected:** Dùng AWS Analytics Lens để đánh giá kiến trúc của bạn

---

## 📞 Đóng Góp

Phát hiện lỗi? Muốn bổ sung nội dung?

Knowledge base này là tài liệu sống. Đóng góp được chào đón:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Thêm section cho chủ đề chưa được đề cập
- [ ] Ví dụ thực tế từ kinh nghiệm của bạn
- [ ] Giải thích rõ hơn các khái niệm phức tạp
- [ ] Bổ sung hands-on exercises (bài tập thực hành)

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 2.2
**Trạng Thái:** ✅ README.md & INDEX.md hoàn thành | ✅ 01-fundamentals hoàn thành (6 files) | ✅ 02-kinesis hoàn thành (5 files) | ✅ 03-glue hoàn thành (5 files) | ✅ 04-athena hoàn thành (5 files) | ✅ 05-redshift hoàn thành (5 files) | ✅ 06-emr hoàn thành (5 files) | ✅ 07-lake-formation hoàn thành (4 files) | ✅ 08-quicksight hoàn thành (4 files) | ✅ 09-opensearch hoàn thành (4 files) | ✅ 10-msk hoàn thành (4 files) | ✅ 11-data-architecture hoàn thành (5 files) | ✅ 12-interview-prep hoàn thành (3 files) — **KNOWLEDGE BASE HOÀN CHỈNH 100%**
