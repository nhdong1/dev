# Amazon EMR — Elastic MapReduce — Xử Lý Dữ Liệu Lớn

> EMR (Elastic MapReduce — Dịch Vụ MapReduce Co Giãn) là nền tảng quản lý cluster (cụm máy chủ) cho Big Data (Dữ Liệu Lớn) trên AWS, hỗ trợ Apache Spark, Hive, Presto, HBase và hàng chục framework mã nguồn mở khác.

## 📚 Nội Dung Module

| File | Chủ Đề | Độ Khó |
|------|--------|--------|
| [1-emr-architecture.md](1-emr-architecture.md) | Kiến trúc EMR — Node types, Cluster lifecycle | ⭐⭐ |
| [2-emr-spark.md](2-emr-spark.md) | Apache Spark trên EMR — Tối ưu & Tuning | ⭐⭐⭐ |
| [3-emr-serverless.md](3-emr-serverless.md) | EMR Serverless — Không quản lý cluster | ⭐⭐ |
| [4-emr-cost-optimization.md](4-emr-cost-optimization.md) | Tối ưu chi phí — Spot, Instance Fleets | ⭐⭐⭐ |

---

## 🎯 EMR Là Gì?

**Amazon EMR** (Elastic MapReduce — Dịch Vụ Xử Lý Dữ Liệu Co Giãn) là dịch vụ cloud-managed (được quản lý trên đám mây) cho phép chạy các framework Big Data như:

- **Apache Spark** — Xử lý dữ liệu phân tán tốc độ cao
- **Apache Hive** — SQL-like queries trên Hadoop (Hệ Thống Phân Tán Hadoop)
- **Apache HBase** — NoSQL database trên HDFS (Hệ Thống File Phân Tán Hadoop)
- **Presto / Trino** — SQL query engine tốc độ cao
- **Apache Flink** — Stream processing (Xử Lý Luồng Thời Gian Thực)
- **Apache Hudi / Iceberg / Delta Lake** — Table formats (Định Dạng Bảng) cho Data Lakehouse

---

## 🏗️ Các Mô Hình Triển Khai EMR

### 1. EMR on EC2 (Truyền Thống)

```
┌─────────────────────────────────────────────┐
│              EMR Cluster (Cụm EMR)          │
│                                             │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐     │
│  │ Primary │  │  Core   │  │  Task   │     │
│  │  Node   │  │  Node   │  │  Node   │     │
│  │(Master) │  │(Worker) │  │(Worker) │     │
│  └─────────┘  └─────────┘  └─────────┘     │
└─────────────────────────────────────────────┘
```

- **Toàn quyền kiểm soát** — Cấu hình instance type, disk, network
- **Chi phí dự đoán được** — Biết chính xác bao nhiêu instance đang chạy
- **Phù hợp cho:** Workload ổn định, cần cấu hình đặc biệt

### 2. EMR Serverless (Không Máy Chủ)

```
Developer → Submit Job → EMR Serverless → Auto-provision → Run → Auto-teardown
```

- **Không quản lý cluster** — AWS lo toàn bộ infrastructure (cơ sở hạ tầng)
- **Pay-per-use** — Chỉ trả tiền khi job đang chạy
- **Phù hợp cho:** Workload không đều, bắt đầu nhanh

### 3. EMR on EKS (Kubernetes)

```
EMR Job → Kubernetes (K8s) Cluster → EKS Nodes → Run Spark
```

- **Tận dụng Kubernetes** (Hệ Thống Điều Phối Container) hiện có
- **Chia sẻ infrastructure** với các workload khác
- **Phù hợp cho:** Tổ chức đã dùng EKS rộng rãi

---

## 📊 So Sánh Các Lựa Chọn EMR

| Tiêu Chí | EMR on EC2 | EMR Serverless | EMR on EKS |
|----------|-----------|----------------|------------|
| **Quản lý** | Cluster tự quản | Không cần quản lý | Qua Kubernetes |
| **Khởi động** | 5–15 phút | 1–2 phút | 1–3 phút |
| **Chi phí** | Instance giờ | vCPU + Memory giờ | EC2 + EKS |
| **Tùy chỉnh** | Cao nhất | Giới hạn | Trung bình |
| **Scaling** | Manual / Auto | Tự động hoàn toàn | Kubernetes HPA |
| **Phù hợp** | Long-running jobs | Intermittent jobs | K8s ecosystem |

---

## 🔗 Vị Trí EMR Trong Hệ Sinh Thái AWS Analytics

```
                    Data Sources (Nguồn Dữ Liệu)
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
           S3 Data      Kinesis      DynamoDB
           Lake         Stream       / RDS
              │            │            │
              └────────────┼────────────┘
                           │
                     ┌─────▼──────┐
                     │  Amazon    │
                     │    EMR     │
                     │(Processing)│
                     └─────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
           S3 (Results) Redshift    OpenSearch
           Parquet/ORC  (DW Load)   (Indexing)
```

**EMR thường đóng vai trò:**
- **Transformation Layer** (Tầng Chuyển Đổi) — Xử lý dữ liệu thô từ S3 hoặc Kinesis
- **Feature Engineering** (Kỹ Thuật Đặc Trưng) — Chuẩn bị dữ liệu cho ML
- **Large-scale ETL** — ETL cần Spark vì Glue không đủ mạnh hoặc quá tốn DPU

---

## 🆚 EMR vs AWS Glue — Khi Nào Dùng Gì?

| Tình Huống | Chọn EMR | Chọn Glue |
|-----------|---------|---------|
| Dataset > 1 TB, cần tối ưu Spark sâu | ✅ | |
| ETL đơn giản, ít code | | ✅ |
| Cần Hive, HBase, Presto | ✅ | |
| Serverless ETL không cần quản lý | | ✅ |
| ML pipeline với Spark MLlib | ✅ | |
| Tích hợp sẵn Glue Data Catalog | | ✅ |
| Custom Spark library / JAR | ✅ | |
| Chạy notebook Jupyter/Zeppelin | ✅ | |

> **Nguyên tắc:** Glue cho ETL phổ thông — EMR khi cần kiểm soát sâu hơn hoặc dùng framework ngoài Spark/Python.

---

## 🎓 Lộ Trình Học Module Này

```
Bước 1: Hiểu kiến trúc (1-emr-architecture.md)
   ↓
Bước 2: Học Spark trên EMR (2-emr-spark.md)
   ↓
Bước 3: EMR Serverless cho use case hiện đại (3-emr-serverless.md)
   ↓
Bước 4: Tối ưu chi phí thực chiến (4-emr-cost-optimization.md)
```

---

## ✅ Câu Hỏi Phỏng Vấn Thường Gặp

1. EMR Primary node, Core node và Task node — mỗi loại làm gì?
2. Khi nào dùng EMR Serverless thay vì EMR on EC2?
3. Cách tối ưu Spark job trên EMR để giảm thời gian và chi phí?
4. Spot Instance (Instance Giá Rẻ Tạm Thời) trong EMR — rủi ro và cách giảm thiểu?
5. EMR vs Glue — khi nào chọn cái nào?

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
