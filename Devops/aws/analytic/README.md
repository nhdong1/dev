# 📊 AWS Analytics Services — Lộ Trình Học Toàn Diện

> Hướng dẫn đầy đủ về AWS Analytics Services — từ nền tảng Kinesis (Luồng Dữ Liệu Thời Gian Thực), Glue (ETL — Extract Transform Load), Athena (Truy Vấn Serverless) đến Redshift (Kho Dữ Liệu), EMR (Elastic MapReduce — Xử Lý Dữ Liệu Lớn), Lake Formation (Quản Lý Data Lake) và các chiến lược kiến trúc dữ liệu hiện đại.

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Các Dịch Vụ](#tổng-quan-các-dịch-vụ)
4. [Tổng Quan Các Chủ Đề](#tổng-quan-các-chủ-đề)
5. [Ma Trận Kỹ Năng](#ma-trận-kỹ-năng)
6. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🎯 Lộ Trình Học

### **Giai Đoạn 1: Nền Tảng — Fundamentals (Tuần 1-2)**

- [ ] Batch Processing (Xử Lý Theo Lô) vs Stream Processing (Xử Lý Luồng Thời Gian Thực)
- [ ] Data Lake (Hồ Dữ Liệu) vs Data Warehouse (Kho Dữ Liệu) — Khái niệm và sự khác biệt
- [ ] ETL (Extract Transform Load — Trích Xuất Chuyển Đổi Tải) vs ELT (Extract Load Transform)
- [ ] Data Pipeline (Đường Ống Dữ Liệu) — Kiến trúc và thành phần
- [ ] S3 (Simple Storage Service) làm nền tảng lưu trữ cho analytics

### **Giai Đoạn 2: Dịch Vụ Cốt Lõi — Core Services (Tuần 3-6)**

- [ ] Kinesis Data Streams (Luồng Dữ Liệu Kinesis) & Kinesis Data Firehose (Vòi Cứu Hỏa Kinesis)
- [ ] AWS Glue — Data Catalog (Danh Mục Dữ Liệu), Crawlers (Trình Thu Thập), ETL Jobs (Công Việc ETL)
- [ ] Amazon Athena — Serverless SQL Query (Truy Vấn SQL Không Máy Chủ) trên S3
- [ ] Amazon Redshift — MPP Data Warehouse (Kho Dữ Liệu Song Song Đại Trà)

### **Giai Đoạn 3: Nâng Cao — Advanced (Tuần 7-10)**

- [ ] Amazon EMR (Elastic MapReduce) — Spark (Xử Lý Phân Tán), Hive, Presto
- [ ] AWS Lake Formation (Quản Lý Data Lake) — Security, Permissions, Governance
- [ ] Amazon OpenSearch Service — Full-text Search (Tìm Kiếm Toàn Văn Bản), Log Analytics
- [ ] Amazon MSK (Managed Streaming for Apache Kafka — Kafka Được Quản Lý)

### **Giai Đoạn 4: Chuyên Sâu — Specialization (Tuần 11+)**

- [ ] Lambda Architecture (Kiến Trúc Lambda — Batch + Speed Layer) vs Kappa Architecture
- [ ] Data Mesh (Lưới Dữ Liệu) — Domain-oriented data ownership
- [ ] Medallion Architecture (Kiến Trúc Huy Chương) — Bronze, Silver, Gold layers
- [ ] Real-time Analytics (Phân Tích Thời Gian Thực) — End-to-end pipeline design

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực                                    | Ưu Tiên | Thời Gian | Trạng Thái |
| ------------------------------------------- | ------- | --------- | ---------- |
| **Kinesis — Real-time Streaming**           | ⭐⭐⭐  | 2 tuần    | -          |
| **Glue — ETL & Data Catalog**               | ⭐⭐⭐  | 2 tuần    | -          |
| **Athena — Serverless Query**               | ⭐⭐⭐  | 1 tuần    | -          |
| **Redshift — Data Warehouse**               | ⭐⭐⭐  | 2 tuần    | -          |
| **S3 Data Lake — Thiết Kế & Tổ Chức**      | ⭐⭐⭐  | 1 tuần    | -          |
| **EMR — Big Data Processing**               | ⭐⭐    | 2 tuần    | -          |
| **Lake Formation — Governance**             | ⭐⭐    | 1 tuần    | -          |
| **QuickSight — BI & Visualization**         | ⭐⭐    | 1 tuần    | -          |
| **OpenSearch — Search & Log Analytics**     | ⭐⭐    | 1 tuần    | -          |
| **MSK — Managed Kafka**                     | ⭐⭐    | 2 tuần    | -          |

---

## 🗂️ Tổng Quan Các Dịch Vụ

### So Sánh Nhanh — Chọn Dịch Vụ Phù Hợp

| Dịch Vụ                  | Loại                         | Tốt Nhất Cho                                     | Mô Hình Tính Phí     |
| ------------------------ | ----------------------------- | ------------------------------------------------ | -------------------- |
| **Kinesis Data Streams** | Real-time Streaming           | Xử lý sự kiện tức thời, custom consumer          | Shard giờ + dữ liệu  |
| **Kinesis Firehose**     | Managed Delivery              | Đưa dữ liệu vào S3/Redshift/OpenSearch           | Dung lượng dữ liệu   |
| **MSK**                  | Managed Kafka                 | Kafka ecosystem, high throughput, multi-consumer | Instance giờ         |
| **AWS Glue**             | Serverless ETL                | Chuyển đổi dữ liệu, Data Catalog                 | DPU giờ              |
| **Athena**               | Serverless SQL                | Ad-hoc query trên S3, không setup cluster        | Dữ liệu quét (TB)    |
| **Redshift**             | MPP Data Warehouse            | OLAP workload lớn, complex analytical queries    | Node giờ / Serverless|
| **EMR**                  | Managed Big Data              | Spark/Hive/Presto, custom Big Data processing    | Instance giờ         |
| **OpenSearch**           | Search & Analytics            | Log analytics, full-text search, dashboards      | Instance giờ         |
| **QuickSight**           | BI & Visualization            | Dashboard, báo cáo, embedded analytics           | Session/User         |
| **Lake Formation**       | Data Lake Governance          | Kiểm soát truy cập data lake tập trung           | Phí dịch vụ cơ bản   |

---

## 🗂️ Tổng Quan Các Chủ Đề

### 📁 **1. Fundamentals — Nền Tảng Analytics** (`01-fundamentals/`)

- Analytics Overview (Tổng Quan Phân Tích) — Các loại analytics: Descriptive, Diagnostic, Predictive, Prescriptive
- Data Pipeline Concepts (Khái Niệm Đường Ống Dữ Liệu) — Ingestion, Processing, Storage, Serving
- Batch vs Streaming (Xử Lý Theo Lô vs Luồng Thời Gian Thực) — Trade-offs và lựa chọn
- Data Lake (Hồ Dữ Liệu) vs Data Warehouse (Kho Dữ Liệu) vs Data Lakehouse (Kết Hợp)
- ETL (Extract Transform Load) vs ELT — Khi nào dùng cái nào

### 📁 **2. Kinesis — Real-time Data Streaming** (`02-kinesis/`)

- **Kinesis Data Streams — KDS** (Luồng Dữ Liệu Kinesis) — Shards (Mảnh), Consumers (Người Tiêu Thụ), Retention
- **Kinesis Data Firehose — KDF** (Vòi Dữ Liệu Kinesis) — Managed delivery đến S3, Redshift, OpenSearch
- **Kinesis Data Analytics — KDA** — SQL / Apache Flink trên streaming data
- Kinesis vs MSK (Apache Kafka) — So sánh và khi nào chọn cái nào

### 📁 **3. AWS Glue — ETL & Data Catalog** (`03-glue/`)

- **Glue Data Catalog** (Danh Mục Dữ Liệu) — Metadata repository, tích hợp Athena/Redshift/EMR
- **Glue ETL Jobs** (Công Việc ETL) — PySpark, Scala, Python Shell, Ray
- **Glue Crawlers** (Trình Thu Thập) — Tự động phát hiện schema và cập nhật Catalog
- **Glue Data Quality** — Data profiling (Phân Tích Hồ Sơ Dữ Liệu), validation rules

### 📁 **4. Amazon Athena — Serverless Query** (`04-athena/`)

- Athena Fundamentals (Nền Tảng Athena) — Presto engine, S3 integration, partitioning
- **Performance Optimization** (Tối Ưu Hiệu Suất) — Columnar formats (Parquet, ORC), partitioning, compression
- **Athena Federated Query** (Truy Vấn Liên Kết) — Query nhiều data source khác nhau
- Cost Optimization (Tối Ưu Chi Phí) — Giảm dữ liệu quét, workgroup budgets

### 📁 **5. Amazon Redshift — Data Warehouse** (`05-redshift/`)

- **Redshift Architecture** (Kiến Trúc Redshift) — Leader node, Compute nodes, MPP (Massively Parallel Processing)
- Distribution & Sort Keys (Khóa Phân Phối & Sắp Xếp) — DISTKEY, SORTKEY, DISTSTYLE
- **Redshift Spectrum** — Query S3 trực tiếp từ Redshift
- **Redshift Serverless** (Redshift Không Máy Chủ) — Auto capacity management

### 📁 **6. Amazon EMR — Big Data Processing** (`06-emr/`)

- **EMR Architecture** (Kiến Trúc EMR) — Master, Core, Task nodes; Cluster lifecycle
- **Apache Spark on EMR** (Spark Trên EMR) — Distributed processing (Xử Lý Phân Tán), optimization
- **EMR Serverless** (EMR Không Máy Chủ) — Không cần quản lý cluster
- Cost Optimization (Tối Ưu Chi Phí) — Spot Instances, Instance Fleets, Auto-termination

### 📁 **7. AWS Lake Formation — Data Lake Governance** (`07-lake-formation/`)

- **Data Lake Design** (Thiết Kế Hồ Dữ Liệu) — Zone architecture, naming conventions, folder structure
- **Lake Formation Security** (Bảo Mật Lake Formation) — Column-level, row-level, tag-based access control
- S3 as Data Lake (S3 Làm Data Lake) — Lifecycle policies, intelligent tiering, storage classes

### 📁 **8. Amazon QuickSight — BI & Visualization** (`08-quicksight/`)

- QuickSight Basics (Nền Tảng QuickSight) — Datasets, analyses, dashboards, stories
- **SPICE Engine** (Super-fast Parallel In-memory Calculation Engine — Công Cụ Tính Toán Bộ Nhớ Song Song Siêu Nhanh)
- Embedded Analytics (Phân Tích Nhúng) — Tích hợp vào ứng dụng của bạn

### 📁 **9. Amazon OpenSearch Service** (`09-opensearch/`)

- OpenSearch Fundamentals (Nền Tảng OpenSearch) — Clusters, Indices, Shards, Replicas
- **OpenSearch Ingestion** (Thu Nạp Dữ Liệu) — Kinesis, Logstash, Fluent Bit integration
- Security (Bảo Mật) — Fine-grained access control (Kiểm Soát Truy Cập Tinh Tế), encryption

### 📁 **10. Amazon MSK — Managed Kafka** (`10-msk/`)

- **MSK Architecture** (Kiến Trúc MSK) — Brokers, Zookeeper, Topics, Partitions, Consumer Groups
- MSK vs Kinesis Data Streams — So sánh chi tiết, trade-offs
- MSK Security (Bảo Mật MSK) — TLS, SASL/SCRAM, IAM authentication

### 📁 **11. Data Architecture — Kiến Trúc Dữ Liệu** (`11-data-architecture/`)

- **Lambda Architecture** (Kiến Trúc Lambda) — Batch Layer + Speed Layer + Serving Layer
- **Kappa Architecture** (Kiến Trúc Kappa) — Chỉ Stream Processing, đơn giản hóa Lambda
- **Data Mesh** (Lưới Dữ Liệu) — Domain ownership, self-serve data platform
- **Medallion Architecture** (Kiến Trúc Huy Chương) — Bronze (thô), Silver (đã làm sạch), Gold (đã tổng hợp)

### 📁 **12. Interview Prep — Chuẩn Bị Phỏng Vấn** (`12-interview-prep/`)

- Top 20 AWS Analytics Interview Questions (20 Câu Hỏi Phỏng Vấn Hàng Đầu)
- System Design Scenarios (Kịch Bản Thiết Kế Hệ Thống) — Real-time pipeline, data lake, BI platform
- STAR Stories (Câu Chuyện Theo Phương Pháp STAR) — Incident & project templates
- Kế Hoạch Học 90 Ngày có lộ trình chi tiết

---

## 🎓 Theo Dịch Vụ Analytics

### **Amazon Kinesis — Real-time Streaming**

```
Điểm Mạnh: Tích hợp native AWS, serverless Firehose, managed scaling
Phù Hợp Cho: Clickstream, IoT, log ingestion, real-time dashboards
Học Ở: 02-kinesis/, 11-data-architecture/
```

### **AWS Glue — ETL & Catalog**

```
Điểm Mạnh: Serverless, tích hợp toàn bộ AWS analytics, Data Catalog trung tâm
Phù Hợp Cho: ETL pipelines, data cataloging, schema management
Học Ở: 03-glue/, 07-lake-formation/
```

### **Amazon Athena — Serverless Query**

```
Điểm Mạnh: Không cần quản lý infrastructure, pay-per-query, tích hợp Glue Catalog
Phù Hợp Cho: Ad-hoc analysis, log queries, data exploration
Học Ở: 04-athena/, 01-fundamentals/
```

### **Amazon Redshift — Data Warehouse**

```
Điểm Mạnh: MPP engine, Redshift Spectrum, tích hợp BI tools tốt
Phù Hợp Cho: OLAP workloads, complex joins, large-scale reporting
Học Ở: 05-redshift/, 08-quicksight/
```

### **Amazon EMR — Big Data**

```
Điểm Mạnh: Toàn quyền tùy chỉnh, hỗ trợ Spark/Hive/Presto, EMR Serverless
Phù Hợp Cho: ML pipelines, large-scale ETL, custom Big Data frameworks
Học Ở: 06-emr/, 11-data-architecture/
```

---

## 🔗 Liên Kết Nhanh

| Chủ Đề                          | Thư Mục                                                      | Ưu Tiên         |
| ------------------------------- | ------------------------------------------------------------ | --------------- |
| Bắt đầu từ đây                  | [README.md](./README.md)                                     | Start here      |
| Nền tảng Analytics              | [01-fundamentals/](./01-fundamentals/)                       | Nền tảng        |
| Real-time Streaming             | [02-kinesis/](./02-kinesis/)                                 | Quan trọng      |
| ETL & Data Catalog              | [03-glue/](./03-glue/)                                       | Quan trọng      |
| Serverless SQL                  | [04-athena/](./04-athena/)                                   | Phổ biến        |
| Data Warehouse                  | [05-redshift/](./05-redshift/)                               | Quan trọng      |
| Big Data Processing             | [06-emr/](./06-emr/)                                        | Nâng cao        |
| Data Lake Governance            | [07-lake-formation/](./07-lake-formation/)                   | Nâng cao        |
| Kiến trúc dữ liệu               | [11-data-architecture/](./11-data-architecture/)             | Chuyên sâu      |
| Câu hỏi phỏng vấn               | [12-interview-prep/INTERVIEW_GUIDE.md](./12-interview-prep/) | Trước phỏng vấn |

---

## 📊 Ma Trận Kỹ Năng

### Người Mới — Beginner (0-1 năm kinh nghiệm)

- [ ] Giải thích sự khác biệt giữa Data Lake (Hồ Dữ Liệu) và Data Warehouse (Kho Dữ Liệu)
- [ ] Chạy câu truy vấn SQL đơn giản trên Athena với dữ liệu từ S3
- [ ] Hiểu Batch Processing (Xử Lý Theo Lô) là gì và Stream Processing (Xử Lý Luồng) là gì
- [ ] Tạo Glue Crawler (Trình Thu Thập Glue) để phát hiện schema tự động
- [ ] Phân biệt Kinesis Data Streams và Kinesis Data Firehose

### Trung Cấp — Intermediate (1-3 năm kinh nghiệm)

- [ ] Thiết kế ETL pipeline (Đường Ống ETL) với AWS Glue từ S3 sang Redshift
- [ ] Tối ưu Athena query với Parquet format và partitioning (Phân Vùng)
- [ ] Xây dựng Kinesis Data Streams consumer (Người Tiêu Thụ) với Lambda
- [ ] Cấu hình Redshift distribution key (Khóa Phân Phối) và sort key (Khóa Sắp Xếp)
- [ ] Thiết lập Lake Formation permissions (Quyền Hạn Lake Formation) theo data domain

### Nâng Cao — Advanced (3-5+ năm kinh nghiệm)

- [ ] Kiến trúc real-time analytics pipeline (Đường Ống Phân Tích Thời Gian Thực) end-to-end
- [ ] Thiết kế Medallion Architecture (Kiến Trúc Huy Chương) với Delta Lake / Apache Iceberg
- [ ] Tối ưu EMR Spark job performance (Hiệu Suất Spark) và chi phí với Spot Instances
- [ ] Triển khai Data Mesh (Lưới Dữ Liệu) với Lake Formation và Glue Catalog
- [ ] Thiết kế multi-tenant analytics platform (Nền Tảng Phân Tích Đa Khách Hàng)

---

## 🚀 Bắt Đầu

### Bước 1: Đặt Mục Tiêu Học Tập

```
Chọn hướng đi phù hợp:
- Data Engineer: Tập trung Kinesis + Glue + Athena + Redshift
- Analytics Engineer: Tập trung Athena + Redshift + QuickSight
- Platform Engineer: Tập trung Lake Formation + EMR + Data Architecture
- Full-stack Analytics: Nắm vững tất cả dịch vụ
```

### Bước 2: Chuẩn Bị Môi Trường Lab

```bash
# Cài đặt AWS CLI (Command Line Interface — Giao Diện Dòng Lệnh)
aws configure

# Kiểm tra kết nối
aws sts get-caller-identity

# Tạo S3 bucket (Thùng S3) làm data lake (hồ dữ liệu) thực hành
aws s3 mb s3://my-analytics-lab-$(aws sts get-caller-identity --query Account --output text)

# Bật S3 versioning (Phiên Bản Hóa S3) cho bucket
aws s3api put-bucket-versioning \
  --bucket my-analytics-lab-$(aws sts get-caller-identity --query Account --output text) \
  --versioning-configuration Status=Enabled
```

### Bước 3: Học + Thực Hành

```
1. Đọc module lý thuyết (30 phút)
2. Thực hành trên AWS Console (30-45 phút)
3. Thử dùng AWS CLI / SDK cho cùng task (15 phút)
4. Xem lại checklist cost — tắt resources không dùng (5 phút)
```

### Bước 4: Chuẩn Bị Câu Chuyện Phỏng Vấn

```
Với mỗi chủ đề, chuẩn bị câu chuyện STAR:
- Situation (Tình Huống): Context của bài toán dữ liệu
- Task (Nhiệm Vụ): Pipeline / architecture bạn cần xây dựng
- Action (Hành Động): Dịch vụ AWS và thiết kế bạn đã chọn
- Result (Kết Quả): Latency, cost, throughput đạt được & bài học
```

---

## 📖 Tài Liệu Tham Khảo

### Sách Nên Đọc

- **"Fundamentals of Data Engineering"** by Joe Reis & Matt Housley — Nền tảng Data Engineering
- **"Designing Data-Intensive Applications"** by Martin Kleppmann — Distributed systems cho dữ liệu
- **"Data Pipelines Pocket Reference"** by James Densmore — Thực hành pipeline
- **"Streaming Systems"** by Tyler Akidau et al. — Lý thuyết stream processing

### Tài Liệu Chính Thức AWS

- [Amazon Kinesis Developer Guide](https://docs.aws.amazon.com/kinesis/)
- [AWS Glue Developer Guide](https://docs.aws.amazon.com/glue/)
- [Amazon Athena User Guide](https://docs.aws.amazon.com/athena/)
- [Amazon Redshift Database Developer Guide](https://docs.aws.amazon.com/redshift/)
- [Amazon EMR Management Guide](https://docs.aws.amazon.com/emr/)

### Blog & Nguồn Học Thêm

- AWS Big Data Blog (blog.awsgeek.com)
- AWS re:Invent Data talks (YouTube)
- A Cloud Guru / Coursera AWS Data Analytics courses
- DataTalks.Club (cộng đồng data engineering)

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Hàng Đầu Theo Danh Mục

#### Streaming & Real-time

- [ ] Kinesis Data Streams vs Kinesis Firehose — khi nào dùng cái nào?
- [ ] Giải thích Shard (Mảnh) trong Kinesis — cách tính số shard cần thiết?
- [ ] Kinesis vs MSK (Kafka) — trade-offs và tiêu chí lựa chọn?
- [ ] Làm thế nào để đảm bảo exactly-once processing (Xử Lý Chính Xác Một Lần) với Kinesis?

#### ETL & Data Catalog

- [ ] AWS Glue ETL (Extract Transform Load) hoạt động thế nào dưới hood?
- [ ] Glue Crawler (Trình Thu Thập) vs manual schema definition — khi nào dùng cái nào?
- [ ] Tối ưu Glue Job (Công Việc Glue) để giảm chi phí DPU (Data Processing Unit)?
- [ ] Glue Data Catalog (Danh Mục Dữ Liệu) tích hợp với Athena, Redshift, EMR thế nào?

#### Query & Data Warehouse

- [ ] Tối ưu Athena query (Truy Vấn Athena) để giảm dữ liệu quét — các kỹ thuật?
- [ ] Redshift DISTKEY (Khóa Phân Phối) và SORTKEY (Khóa Sắp Xếp) — cách chọn?
- [ ] Redshift Spectrum vs Athena — khi nào dùng cái nào?
- [ ] Redshift Serverless (Không Máy Chủ) vs Provisioned Redshift — trade-offs?

#### Architecture & Design

- [ ] Thiết kế real-time analytics pipeline (Đường Ống Phân Tích Thời Gian Thực) cho 1M events/giây?
- [ ] Lambda Architecture (Kiến Trúc Lambda) vs Kappa Architecture — khác nhau thế nào?
- [ ] Data Lake (Hồ Dữ Liệu) với S3 — cách tổ chức folder structure và governance?
- [ ] Medallion Architecture (Kiến Trúc Huy Chương) — giải thích Bronze, Silver, Gold layers?

Xem `12-interview-prep/` để có hướng dẫn đầy đủ Q&A.

---

## ✅ Checklist Tự Đánh Giá

Trước khi phỏng vấn hoặc nhận vai trò mới, xác nhận:

- [ ] Có thể giải thích sự khác biệt giữa Batch và Stream processing
- [ ] Có thể thiết kế data pipeline end-to-end với ít nhất 2 dịch vụ AWS analytics
- [ ] Có thể chạy và tối ưu Athena query trên dữ liệu Parquet đã phân vùng
- [ ] Có thể giải thích Redshift distribution strategy (Chiến Lược Phân Phối Redshift)
- [ ] Có thể chọn giữa Kinesis và MSK dựa trên yêu cầu kỹ thuật
- [ ] Có thể mô tả Medallion Architecture và áp dụng vào bài toán thực tế
- [ ] Có thể tính chi phí xấp xỉ cho một analytics pipeline trên AWS
- [ ] Có thể giải thích Lake Formation và cách kiểm soát quyền truy cập dữ liệu
- [ ] Có thể thiết kế HA (High Availability — Tính Sẵn Sàng Cao) cho Redshift cluster
- [ ] Có thể xử lý sự cố phổ biến: Glue job timeout, Kinesis throttling, Athena slow query

---

## 📞 Hỗ Trợ & Tài Nguyên

### Học Tập

- [AWS Free Tier](https://aws.amazon.com/free/) — Thực hành miễn phí (Athena, Glue có free tier giới hạn)
- [AWS Skill Builder](https://skillbuilder.aws/) — Khóa học chính thức AWS Analytics
- [AWS Analytics Lens](https://docs.aws.amazon.com/wellarchitected/latest/analytics-lens/) — Well-Architected cho Analytics

### Công Cụ

- **AWS Glue Studio** — Giao diện đồ họa thiết kế ETL job
- **Amazon Athena Console** — Chạy SQL trực tiếp trên S3
- **AWS Data Wrangler (SDK)** — Pandas-like API cho AWS analytics services
- **Apache Airflow on MWAA** — Orchestration (Điều Phối) workflow analytics
- **dbt (data build tool)** — Transformation layer trong data warehouse

### Cộng Đồng

- r/dataengineering (Reddit)
- AWS re:Post — Analytics category
- DataTalks.Club Slack
- Apache Kafka / Spark communities

---

## 📋 Cách Sử Dụng Hướng Dẫn Này

### Cho Mục Đích Tự Học

1. Bắt đầu với [Lộ Trình Học](#lộ-trình-học)
2. Đi qua từng giai đoạn theo thứ tự
3. Thực hành hands-on trên AWS Console sau mỗi module
4. **Chú ý chi phí** — xóa resources sau khi thực hành để tránh phát sinh phí

### Cho Chuẩn Bị Phỏng Vấn

1. Tập trung vào [12-interview-prep/](./12-interview-prep/)
2. Nắm sâu Kinesis + Glue + Athena + Redshift (luôn được hỏi)
3. Chuẩn bị câu chuyện thiết kế pipeline theo phương pháp STAR
4. Luyện giải thích trade-offs giữa các dịch vụ (ví dụ: EMR vs Glue, Redshift vs Athena)

### Cho Công Việc Thực Tế

1. Xác định bài toán: Batch hay Streaming? Real-time hay Near-real-time?
2. Tham khảo [11-data-architecture/](./11-data-architecture/) để chọn kiến trúc phù hợp
3. Dùng [07-lake-formation/](./07-lake-formation/) để thiết lập governance từ đầu
4. Áp dụng [04-athena/](./04-athena/) và [05-redshift/](./05-redshift/) cho query layer

---

## 🗺️ Các Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Chọn lộ trình học (Beginner / Intermediate / Advanced)
├─ 3️⃣  Bắt đầu với 01-fundamentals/ để nắm vững nền tảng
├─ 4️⃣  Tạo AWS Free Tier account và thực hành Athena với public dataset
├─ 5️⃣  Xây dựng mini data pipeline: S3 → Glue → Athena → QuickSight
├─ 6️⃣  Nâng cấp với real-time: Kinesis → Lambda → S3 → Athena
└─ 7️⃣  Chuẩn bị phỏng vấn với 12-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
**Maintainer:** Backend Interview Prep
