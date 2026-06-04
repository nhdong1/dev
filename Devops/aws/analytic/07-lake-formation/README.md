# 07 — AWS Lake Formation — Quản Trị Data Lake

> AWS Lake Formation — Dịch Vụ Quản Trị Hồ Dữ Liệu — là dịch vụ giúp bạn xây dựng, bảo mật và quản lý data lake (hồ dữ liệu) trên AWS một cách tập trung, an toàn và hiệu quả.

## 📚 Mục Lục Module

| File | Nội Dung | Trạng Thái |
| ---- | -------- | ---------- |
| [README.md](./README.md) | Tổng quan Lake Formation, kiến trúc, use cases | ✅ |
| [1-data-lake-design.md](./1-data-lake-design.md) | Zone architecture, folder structure, naming conventions | ✅ |
| [2-lake-formation-security.md](./2-lake-formation-security.md) | Column/row-level security, LF-Tags, permissions | ✅ |
| [3-s3-data-lake.md](./3-s3-data-lake.md) | S3 lifecycle, Intelligent-Tiering, storage classes | ✅ |

---

## 🎯 Lake Formation Là Gì?

**AWS Lake Formation** (Hình Thành Hồ Dữ Liệu) là dịch vụ giúp:

- **Xây dựng** data lake an toàn trong vài ngày thay vì vài tháng
- **Quản lý tập trung** quyền truy cập dữ liệu ở cấp độ cột, hàng, và bảng
- **Tích hợp** với Glue, Athena, Redshift Spectrum, EMR, QuickSight
- **Kiểm toán** (Audit) mọi truy cập dữ liệu qua CloudTrail

```
┌─────────────────────────────────────────────────────────────────────┐
│                      AWS Lake Formation                             │
│                                                                     │
│  ┌──────────────────┐     ┌──────────────────────────────────────┐  │
│  │   Data Sources   │     │         Data Lake (S3)               │  │
│  │                  │     │  ┌──────────┐  ┌──────────────────┐  │  │
│  │  • RDS           │────▶│  │  Raw     │  │   Processed      │  │  │
│  │  • DynamoDB      │     │  │  Zone    │  │   Zone           │  │  │
│  │  • S3 (existing) │     │  └──────────┘  └──────────────────┘  │  │
│  │  • On-premises   │     │  ┌──────────────────────────────────┐  │  │
│  └──────────────────┘     │  │         Curated Zone             │  │  │
│                           │  └──────────────────────────────────┘  │  │
│  ┌──────────────────┐     └──────────────────────────────────────┘  │
│  │  Access Control  │                      │                        │
│  │                  │     ┌────────────────▼─────────────────────┐  │
│  │  • Column-level  │     │         Consumers (Người Dùng)       │  │
│  │  • Row-level     │────▶│  • Athena  • Redshift  • EMR         │  │
│  │  • Table-level   │     │  • Glue    • QuickSight               │  │
│  │  • LF-Tags       │     └──────────────────────────────────────┘  │
│  └──────────────────┘                                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ Kiến Trúc Tổng Quan

### Các Thành Phần Chính

#### 1. Data Catalog (Danh Mục Dữ Liệu)
Lake Formation tích hợp chặt với **Glue Data Catalog** — kho lưu trữ metadata trung tâm:
- **Databases** (Cơ Sở Dữ Liệu) — nhóm các bảng liên quan
- **Tables** (Bảng) — metadata của dataset (schema, location, format)
- **Partitions** (Phân Vùng) — phân chia dữ liệu theo cột để tối ưu query

#### 2. Data Lake Administrator (Quản Trị Viên Hồ Dữ Liệu)
Người có toàn quyền quản lý Lake Formation:
- Cấp phép cho người dùng và role khác
- Đăng ký S3 location làm data lake storage
- Tạo LF-Tags (Nhãn Lake Formation) để phân loại dữ liệu

#### 3. Permissions Model (Mô Hình Phân Quyền)
Lake Formation hỗ trợ hai cơ chế:
- **Lake Formation permissions** — cấp phép ở cấp độ Catalog (bảng, cột, hàng)
- **IAM permissions** — cấp phép ở cấp độ S3 (vẫn cần, nhưng Lake Formation là lớp trên)

```
IAM Policy (Cho phép truy cập Glue/Athena)
           +
Lake Formation Permission (Cho phép đọc bảng cụ thể)
           =
Người dùng có thể truy cập dữ liệu
```

---

## 🔑 Tính Năng Nổi Bật

### Fine-Grained Access Control (Kiểm Soát Truy Cập Tinh Tế)

| Cấp Độ | Mô Tả | Ví Dụ |
| ------ | ----- | ----- |
| **Database** | Quyền trên toàn bộ database | Nhóm Analyst chỉ thấy database `analytics` |
| **Table** | Quyền trên một bảng cụ thể | Nhóm Finance chỉ thấy bảng `transactions` |
| **Column** | Ẩn/hiện cột nhất định | Ẩn cột `ssn`, `credit_card_number` |
| **Row** | Lọc hàng theo điều kiện | Mỗi region chỉ thấy dữ liệu region mình |
| **Cell** | Kết hợp cột + hàng | Ẩn giá trị lương của cấp cao hơn mình |

### LF-Tags (Nhãn Lake Formation — Tag-Based Access Control)
Thay vì cấp quyền từng bảng, dùng tag để quản lý theo nhóm:
```
LF-Tag: classification=confidential → Chỉ nhóm DataSteward được truy cập
LF-Tag: environment=production → Chỉ nhóm Production được truy cập
LF-Tag: domain=finance → Nhóm Finance team được truy cập
```

### Cross-Account Data Sharing (Chia Sẻ Dữ Liệu Giữa Các Tài Khoản)
Chia sẻ bảng trong Catalog với AWS account khác mà không cần copy dữ liệu:
- Account A có dữ liệu, Account B query qua Athena
- Lake Formation kiểm soát quyền ở Account A
- Dữ liệu không bao giờ rời khỏi Account A

---

## 📊 Use Cases Phổ Biến

### 1. Enterprise Data Lake Governance (Quản Trị Data Lake Doanh Nghiệp)
```
Bài toán: 50 team, 500 dataset, cần kiểm soát ai được đọc gì
Giải pháp: Lake Formation + LF-Tags theo domain và classification
Lợi ích: Quản lý tập trung, audit trail đầy đủ
```

### 2. PII Data Protection (Bảo Vệ Dữ Liệu Nhận Dạng Cá Nhân)
```
Bài toán: Dataset chứa thông tin cá nhân (email, phone, CMND)
Giải pháp: Column-level masking/restriction qua Lake Formation
Lợi ích: Analyst làm việc với dữ liệu mà không thấy PII
```

### 3. Multi-Tenant Analytics (Phân Tích Đa Khách Hàng)
```
Bài toán: SaaS platform, mỗi tenant chỉ thấy dữ liệu của mình
Giải pháp: Row-level filters theo tenant_id
Lợi ích: Một data lake, nhiều tenant, bảo mật hoàn toàn
```

### 4. Data Mesh Implementation (Triển Khai Lưới Dữ Liệu)
```
Bài toán: Mỗi domain team tự quản lý dữ liệu của mình
Giải pháp: Cross-account sharing + LF-Tags theo domain
Lợi ích: Ownership phân tán, governance tập trung
```

---

## 🔗 Tích Hợp Với Các Dịch Vụ AWS

```
┌─────────────────────────────────────────────────────────┐
│               Lake Formation Integrations               │
│                                                         │
│  Ingestion    │  Processing   │  Query        │  BI     │
│  ────────     │  ──────────   │  ──────       │  ──     │
│  Glue ETL     │  EMR Spark    │  Athena       │  Quick  │
│  DMS          │  Glue Jobs    │  Redshift     │  Sight  │
│  AppFlow      │  Lambda       │  Spectrum     │         │
│               │               │  SageMaker    │         │
└─────────────────────────────────────────────────────────┘
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

1. **Lake Formation khác gì với IAM S3 permissions thuần túy?**
   → Lake Formation cung cấp kiểm soát ở cấp độ bảng/cột/hàng trong Glue Catalog, trong khi IAM chỉ kiểm soát ở cấp độ S3 prefix. Lake Formation là lớp bảo mật bổ sung phía trên IAM.

2. **Giải thích LF-Tags và lợi ích so với resource-based permissions?**
   → LF-Tags cho phép cấp quyền theo thuộc tính (attribute-based access control) thay vì từng resource riêng lẻ. Khi dataset mới được tạo với tag phù hợp, quyền truy cập tự động áp dụng — không cần cấu hình lại.

3. **Row-level security trong Lake Formation hoạt động thế nào?**
   → Lake Formation tạo một data filter (bộ lọc dữ liệu) với biểu thức SQL, sau đó gán filter đó cho principal (người dùng/role). Khi query chạy qua Athena/Redshift Spectrum, filter tự động được áp dụng.

4. **Cross-account data sharing trong Lake Formation làm thế nào?**
   → Dùng AWS RAM (Resource Access Manager) để share Glue Catalog resource với account khác. Account nhận chỉ có quyền query, không có quyền admin hay copy dữ liệu.

---

## 📖 Trong Module Này

- **[1-data-lake-design.md](./1-data-lake-design.md)** — Thiết kế data lake: zone architecture, folder naming, partitioning strategy
- **[2-lake-formation-security.md](./2-lake-formation-security.md)** — Fine-grained security: column, row, cell-level; LF-Tags; cross-account
- **[3-s3-data-lake.md](./3-s3-data-lake.md)** — S3 làm nền tảng data lake: lifecycle, Intelligent-Tiering, storage classes

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
