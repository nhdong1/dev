# Amazon Redshift — MPP Data Warehouse (Kho Dữ Liệu Song Song Đại Trà)

> Amazon Redshift là dịch vụ data warehouse (kho dữ liệu) được quản lý hoàn toàn, sử dụng kiến trúc MPP — Massively Parallel Processing (Xử Lý Song Song Đại Trà) để thực thi các OLAP — Online Analytical Processing (Xử Lý Phân Tích Trực Tuyến) workload với hiệu suất cao trên dữ liệu quy mô petabyte.

---

## 📚 Mục Lục Module

| File | Nội Dung | Trạng Thái |
|------|----------|------------|
| [1-redshift-architecture.md](./1-redshift-architecture.md) | Kiến trúc Redshift — Leader node, Compute nodes, MPP | ✅ |
| [2-redshift-performance.md](./2-redshift-performance.md) | Tối ưu hiệu suất — DISTKEY, SORTKEY, DISTSTYLE, WLM | ✅ |
| [3-redshift-spectrum.md](./3-redshift-spectrum.md) | Redshift Spectrum — Query S3 trực tiếp từ Redshift | ✅ |
| [4-redshift-serverless.md](./4-redshift-serverless.md) | Redshift Serverless — Auto capacity, RPU, use cases | ✅ |

---

## 🎯 Tổng Quan Nhanh

### Amazon Redshift Là Gì?

Amazon Redshift là dịch vụ **cloud data warehouse** (kho dữ liệu đám mây) được AWS quản lý hoàn toàn, tối ưu cho các **analytical workload** (khối lượng công việc phân tích) phức tạp. Redshift:

- Dùng kiến trúc **columnar storage** — Column-oriented Storage (Lưu Trữ Theo Cột) để tăng tốc query phân tích
- Phân tán dữ liệu và tính toán trên nhiều node — **MPP** (Massively Parallel Processing — Xử Lý Song Song Đại Trà)
- Tích hợp với **Redshift Spectrum** để query dữ liệu trực tiếp trên S3
- Cung cấp tùy chọn **Serverless** (Không Máy Chủ) cho workload không thường xuyên

### Khi Nào Dùng Redshift?

```
✅ Phù hợp:
   - OLAP workload: complex JOIN, GROUP BY, aggregate trên hàng tỉ rows
   - BI dashboards cần response time < 1 giây
   - Reporting định kỳ (daily, weekly, monthly)
   - Data warehouse tập trung cho toàn tổ chức
   - Workload chạy thường xuyên và có thể dự đoán

❌ Không phù hợp:
   - OLTP (Online Transaction Processing — Xử Lý Giao Dịch Trực Tuyến): nhiều INSERT/UPDATE/DELETE nhỏ → dùng RDS/Aurora
   - Ad-hoc query không thường xuyên trên S3 → dùng Athena
   - Key-value lookup → dùng DynamoDB
   - Search engine → dùng OpenSearch
```

---

## 🏗️ Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                              │
│  SQL Clients │ JDBC/ODBC │ BI Tools (QuickSight, Tableau, dbt)  │
└──────────────────────────┬──────────────────────────────────────┘
                           │ SQL Query
┌──────────────────────────▼──────────────────────────────────────┐
│                    Redshift Cluster                               │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    Leader Node (Nút Dẫn Đầu)                │ │
│  │  - Nhận query từ client                                      │ │
│  │  - Lập kế hoạch thực thi (Execution Plan)                   │ │
│  │  - Phân phối công việc cho Compute Nodes                    │ │
│  │  - Tổng hợp kết quả và trả về client                        │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                           │ Distribute Tasks                      │
│              ┌────────────┼────────────┐                          │
│              ▼            ▼            ▼                          │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                │
│  │Compute Node1│ │Compute Node2│ │Compute Node3│  ...            │
│  │             │ │             │ │             │                  │
│  │  Slice 1    │ │  Slice 3    │ │  Slice 5    │                  │
│  │  Slice 2    │ │  Slice 4    │ │  Slice 6    │                  │
│  │             │ │             │ │             │                  │
│  │  Local SSD  │ │  Local SSD  │ │  Local SSD  │                  │
│  └─────────────┘ └─────────────┘ └─────────────┘                │
└───────────────────────────────────────────────────────────────────┘
                           │
              ┌────────────┴────────────────┐
              ▼                             ▼
┌─────────────────────┐       ┌─────────────────────────┐
│  Amazon S3          │       │  Redshift Managed Storage│
│  (Redshift Spectrum)│       │  (RA3 nodes — tách biệt  │
│  query external data│       │   storage và compute)    │
└─────────────────────┘       └─────────────────────────┘
```

---

## ⚡ Điểm Mạnh và Hạn Chế

### Điểm Mạnh

| Đặc Điểm | Giải Thích |
|-----------|-----------|
| **MPP Engine** | Tự động phân tán query cho tất cả Compute Nodes |
| **Columnar Storage** | Nén tốt, đọc chỉ cột cần thiết → query analytics nhanh hơn |
| **Redshift Spectrum** | Query S3 trực tiếp — không cần load dữ liệu vào cluster |
| **SQL chuẩn** | Hỗ trợ đầy đủ PostgreSQL-compatible SQL |
| **RA3 Nodes** | Tách biệt storage và compute — scale độc lập |
| **Tích hợp AWS** | Glue, S3, IAM, VPC, CloudWatch, QuickSight, SageMaker |
| **Serverless option** | Không quản lý cluster, tự động scale theo workload |

### Hạn Chế Cần Biết

| Hạn Chế | Nguyên Nhân | Giải Pháp |
|---------|-------------|-----------|
| **Không phù hợp OLTP** | Columnar storage chậm cho row-level updates | Dùng RDS/Aurora |
| **Vacuum & Analyze** | Cần chạy định kỳ sau nhiều DELETE/UPDATE | Bật auto vacuum |
| **Cold start** | Cluster provisioned start mất vài phút | Dùng Serverless hoặc keep warm |
| **Chi phí cao khi idle** | Provisioned cluster tính phí 24/7 | Pause cluster hoặc dùng Serverless |
| **Concurrency limit** | Mặc định tối đa ~50 concurrent queries | Cấu hình WLM hoặc Concurrency Scaling |

---

## 🆚 Redshift vs Các Dịch Vụ Khác

### Redshift vs Athena

| Tiêu Chí | Redshift | Athena |
|----------|----------|--------|
| **Hạ tầng** | Provisioned hoặc Serverless cluster | Serverless hoàn toàn |
| **Chi phí** | Theo giờ (node) hoặc RPU-giây | Theo TB dữ liệu quét |
| **Tốc độ** | Mili-giây đến giây (với indexing tốt) | Giây đến phút |
| **Workload** | Thường xuyên, phức tạp | Ad-hoc, không thường xuyên |
| **Tốt nhất** | BI dashboards, complex OLAP | Data exploration, log queries |

### Redshift vs EMR

| Tiêu Chí | Redshift | EMR |
|----------|----------|-----|
| **Mục đích** | SQL analytics, data warehouse | Spark/Hive, ML pipelines, ETL |
| **Giao diện** | SQL | Spark API, HQL, Presto SQL |
| **Quản lý** | Fully managed | Semi-managed (cluster tự quản) |
| **Tốt nhất** | BI, reporting, complex SQL | Big data ETL, ML, custom processing |

---

## 📋 Use Cases Phổ Biến

```
1. Enterprise Data Warehouse (Kho Dữ Liệu Doanh Nghiệp)
   → Tập trung dữ liệu từ nhiều nguồn: CRM, ERP, ứng dụng → analytics

2. BI Dashboard Backend (Nền Tảng BI)
   → Cung cấp dữ liệu nhanh cho QuickSight, Tableau, Power BI

3. Customer Analytics (Phân Tích Khách Hàng)
   → Phân tích hành vi, churn prediction, segmentation

4. Financial Reporting (Báo Cáo Tài Chính)
   → Tổng hợp doanh thu, chi phí, P&L với complex aggregations

5. Operational Analytics (Phân Tích Vận Hành)
   → Monitoring KPIs, SLAs từ dữ liệu operational

6. Data Lakehouse (Kết Hợp Data Lake và Warehouse)
   → Redshift + Spectrum + S3 = query cả hot và cold data
```

---

## 🔗 Liên Kết Nội Bộ

| Chủ Đề | Module |
|--------|--------|
| Glue ETL load dữ liệu vào Redshift | [03-glue/2-glue-etl-jobs.md](../03-glue/2-glue-etl-jobs.md) |
| Athena vs Redshift — khi nào chọn gì | [04-athena/README.md](../04-athena/README.md) |
| Redshift Spectrum vs Athena | [3-redshift-spectrum.md](./3-redshift-spectrum.md) |
| QuickSight kết nối Redshift | [08-quicksight/README.md](../08-quicksight/README.md) |

---

## 🎓 Lộ Trình Học Module Này

```
Người mới:
  1-redshift-architecture.md → Nắm kiến trúc MPP, Leader/Compute nodes

Intermediate:
  2-redshift-performance.md → DISTKEY, SORTKEY, WLM — cốt lõi để optimize

Nâng cao:
  3-redshift-spectrum.md    → Mở rộng query ra S3, data lakehouse
  4-redshift-serverless.md  → Lựa chọn hiện đại cho workload linh hoạt
```

---

**Tiếp Theo:** [1-redshift-architecture.md](./1-redshift-architecture.md) — Kiến trúc MPP, Leader Node, Compute Nodes và cách Redshift thực thi query
