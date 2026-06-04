# Amazon Athena — Serverless SQL Query (Truy Vấn SQL Không Máy Chủ)

> Amazon Athena là dịch vụ truy vấn SQL serverless (không máy chủ) cho phép phân tích dữ liệu trực tiếp trên Amazon S3 mà không cần thiết lập hay quản lý hạ tầng. Dựa trên engine Presto/Trino, Athena hỗ trợ chuẩn ANSI SQL, định dạng dữ liệu mở và tính phí theo lượng dữ liệu quét.

---

## 📚 Mục Lục Module

| File | Nội Dung | Trạng Thái |
|------|----------|------------|
| [1-athena-fundamentals.md](./1-athena-fundamentals.md) | Nền tảng Athena — Presto engine, S3 integration, setup | ✅ |
| [2-athena-performance.md](./2-athena-performance.md) | Tối ưu hiệu suất — Parquet, ORC, partitioning, compression | ✅ |
| [3-athena-federation.md](./3-athena-federation.md) | Federated Query — truy vấn đa nguồn dữ liệu | ✅ |
| [4-athena-cost-optimization.md](./4-athena-cost-optimization.md) | Tối ưu chi phí — giảm dữ liệu quét, workgroup budgets | ✅ |

---

## 🎯 Tổng Quan Nhanh

### Amazon Athena Là Gì?

Amazon Athena là dịch vụ **interactive query service** (dịch vụ truy vấn tương tác) được xây dựng trên nền tảng **Presto** (từ phiên bản Athena v2 trở đi dùng **Trino** — phiên bản cải tiến của Presto). Athena cho phép:

- Chạy SQL trực tiếp trên dữ liệu lưu trong **S3** (Simple Storage Service)
- Không cần provisioning (cung cấp tài nguyên) hay quản lý server
- Tự động scale (mở rộng) theo nhu cầu
- Tích hợp sẵn với **AWS Glue Data Catalog** (Danh Mục Dữ Liệu Glue)

### Mô Hình Tính Phí

```
$5 / TB dữ liệu quét (scanned)
→ Dùng Parquet/ORC + partition → quét ít hơn → giảm chi phí 70-95%
→ Query thất bại không bị tính phí
→ DDL (CREATE/DROP/ALTER) không bị tính phí
```

---

## 🏗️ Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────┐
│                      Client Layer                           │
│  SQL Console │ JDBC/ODBC │ AWS SDK │ QuickSight │ dbt       │
└──────────────────────┬──────────────────────────────────────┘
                       │ SQL Query
┌──────────────────────▼──────────────────────────────────────┐
│                   Amazon Athena                              │
│  ┌────────────────┐    ┌──────────────────────────────────┐ │
│  │  Query Engine  │    │        Workgroup Manager          │ │
│  │ (Trino/Presto) │    │  (quản lý workload và chi phí)   │ │
│  └────────┬───────┘    └──────────────────────────────────┘ │
│           │                                                  │
│  ┌────────▼───────────────────────────────┐                 │
│  │         Query Planning & Execution      │                 │
│  │  Parser → Planner → Optimizer → Runner  │                 │
│  └────────┬──────────────┬────────────────┘                 │
└───────────┼──────────────┼─────────────────────────────────┘
            │              │
┌───────────▼──┐   ┌───────▼──────────────────────────────────┐
│ Glue Catalog │   │            Amazon S3                       │
│ (Metadata)   │   │  s3://bucket/prefix/year=2024/month=01/   │
│ - Databases  │   │  ├── file1.parquet                        │
│ - Tables     │   │  ├── file2.parquet                        │
│ - Schemas    │   │  └── ...                                  │
│ - Partitions │   └───────────────────────────────────────────┘
└──────────────┘
```

---

## ⚡ Điểm Mạnh và Hạn Chế

### Điểm Mạnh

| Đặc Điểm | Giải Thích |
|-----------|-----------|
| **Serverless** | Không quản lý infrastructure (cơ sở hạ tầng) |
| **Pay-per-query** | Trả tiền theo dữ liệu quét, không phải idle time |
| **Standard SQL** | Hỗ trợ ANSI SQL, window functions, CTEs |
| **Format đa dạng** | CSV, JSON, Parquet, ORC, Avro, Textfile |
| **Tích hợp native** | Glue Catalog, IAM, CloudTrail, Lake Formation |
| **Federated Query** | Kết nối RDS, DynamoDB, Redis và hơn thế nữa |

### Hạn Chế Cần Biết

| Hạn Chế | Nguyên Nhân | Giải Pháp |
|---------|-------------|-----------|
| **Không hỗ trợ DML** | Read-only analytics | Dùng Glue/Spark để write |
| **Timeout 30 phút** | Giới hạn query execution | Tách query hoặc dùng CTAS |
| **Concurrency giới hạn** | Mặc định 20 concurrent queries | Tăng limit hoặc dùng workgroup |
| **Không tối ưu OLTP** | Thiết kế cho analytics | Dùng RDS/Aurora cho OLTP |
| **Chi phí biến động** | Tính theo dữ liệu quét | Cần tối ưu format và partition |

---

## 🆚 Athena vs Các Dịch Vụ Khác

### Athena vs Redshift

| Tiêu Chí | Athena | Redshift |
|----------|--------|----------|
| **Hạ tầng** | Serverless hoàn toàn | Provisioned hoặc Serverless |
| **Chi phí** | Pay-per-query | Pay-per-hour (theo giờ) |
| **Tốc độ** | Giây đến phút | Mili-giây đến giây |
| **Workload** | Ad-hoc, không thường xuyên | Thường xuyên, phức tạp |
| **Lưu trữ** | Dữ liệu ở S3 | Copy vào Redshift storage |
| **Tốt nhất** | Data exploration, log query | Complex OLAP, BI dashboards |

### Athena vs EMR Presto/Trino

| Tiêu Chí | Athena | EMR Presto/Trino |
|----------|--------|-----------------|
| **Quản lý** | Fully managed | Tự quản lý cluster |
| **Linh hoạt** | Cố định config | Tùy chỉnh hoàn toàn |
| **Chi phí** | Theo dữ liệu quét | Theo EC2 instance giờ |
| **Warm-up** | Không | Cluster startup time |
| **Tốt nhất** | Đơn giản, nhanh khởi động | Tùy chỉnh sâu, throughput cao |

---

## 📋 Use Cases Phổ Biến

```
1. Ad-hoc Data Exploration (Khám Phá Dữ Liệu Tùy Nhu Cầu)
   → Analyst muốn query dữ liệu S3 ngay mà không cần setup

2. Log Analytics (Phân Tích Log)
   → CloudTrail, ALB access logs, VPC Flow Logs trên S3

3. ETL Validation (Kiểm Tra ETL)
   → Verify output của Glue job trước khi load vào Redshift

4. Cost-effective Reporting (Báo Cáo Tiết Kiệm Chi Phí)
   → Monthly/weekly reports không cần Redshift running 24/7

5. Data Lake Querying (Truy Vấn Data Lake)
   → Query Bronze/Silver/Gold layers trong Medallion Architecture

6. Federated Analytics (Phân Tích Liên Kết)
   → Join dữ liệu S3 với RDS và DynamoDB trong một query
```

---

## 🔗 Liên Kết Nội Bộ

| Chủ Đề | Module |
|--------|--------|
| Glue Data Catalog tích hợp | [03-glue/1-glue-catalog.md](../03-glue/1-glue-catalog.md) |
| Định dạng Parquet và ORC | [2-athena-performance.md](./2-athena-performance.md) |
| S3 Data Lake design | [07-lake-formation/3-s3-data-lake.md](../07-lake-formation/3-s3-data-lake.md) |
| Athena vs Redshift Spectrum | [05-redshift/3-redshift-spectrum.md](../05-redshift/3-redshift-spectrum.md) |

---

## 🎓 Lộ Trình Học Module Này

```
Người mới:
  1-athena-fundamentals.md → Nắm cơ bản, chạy query đầu tiên

Intermediate:
  2-athena-performance.md → Tối ưu với Parquet + partition

Nâng cao:
  3-athena-federation.md  → Query đa nguồn
  4-athena-cost-optimization.md → Kiểm soát chi phí và workload
```

---

**Tiếp Theo:** [1-athena-fundamentals.md](./1-athena-fundamentals.md) — Nền tảng Athena, Presto engine, setup môi trường
