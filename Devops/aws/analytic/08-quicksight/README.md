# 📊 Amazon QuickSight — BI & Visualization (Trí Tuệ Kinh Doanh & Trực Quan Hóa)

> Amazon QuickSight là dịch vụ BI (Business Intelligence — Trí Tuệ Kinh Doanh) serverless (Không Máy Chủ) trên đám mây, cho phép tạo dashboard (Bảng Điều Khiển), báo cáo và nhúng analytics vào ứng dụng với chi phí tính theo người dùng.

---

## 📚 Mục Lục Module

| File | Chủ Đề | Trạng Thái |
|------|--------|------------|
| [1-quicksight-basics.md](./1-quicksight-basics.md) | Datasets, Analyses, Dashboards, Stories | ✅ |
| [2-spice-engine.md](./2-spice-engine.md) | SPICE — In-memory Calculation Engine | ✅ |
| [3-embedded-analytics.md](./3-embedded-analytics.md) | Embedded Analytics — Nhúng vào ứng dụng | ✅ |

---

## 🎯 QuickSight Là Gì?

**Amazon QuickSight** là dịch vụ BI (Business Intelligence) được quản lý hoàn toàn bởi AWS, không cần cài đặt server hay phần mềm. Người dùng có thể kết nối trực tiếp đến các nguồn dữ liệu AWS (S3, Redshift, Athena, RDS...) hoặc nguồn bên ngoài để tạo dashboard tương tác.

### Đặc Điểm Nổi Bật

| Đặc Điểm | Chi Tiết |
|----------|----------|
| **Serverless** | Không cần quản lý infrastructure (Hạ Tầng) |
| **SPICE** | Super-fast Parallel In-memory Calculation Engine — in-memory cache siêu nhanh |
| **ML Insights** | Tích hợp sẵn ML (Machine Learning) để phát hiện anomaly (Bất Thường) và dự báo |
| **Embedded Analytics** | Nhúng dashboard vào ứng dụng web/mobile |
| **Pay-per-session** | Tính phí theo phiên sử dụng — phù hợp với số lượng user lớn |

---

## 🏗️ Kiến Trúc Tổng Quan

```
Nguồn Dữ Liệu (Data Sources)
        │
        ▼
┌───────────────────────────────────┐
│         Dataset (Bộ Dữ Liệu)     │
│  ┌─────────────┐ ┌─────────────┐ │
│  │  SPICE Mode │ │ Direct Query│ │
│  │  (In-memory)│ │ (Live data) │ │
│  └─────────────┘ └─────────────┘ │
└───────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────┐
│         Analysis (Phân Tích)      │
│   Visuals + Calculated Fields     │
│   Filters + Parameters            │
└───────────────────────────────────┘
        │
        ├─── Publish ──▶  Dashboard (Bảng Điều Khiển)
        │                  (Chỉ đọc, chia sẻ được)
        │
        └─── Embed ──▶   Embedded Dashboard
                          (Trong ứng dụng của bạn)
```

---

## 🔌 Nguồn Dữ Liệu Được Hỗ Trợ

### AWS Native Sources (Nguồn AWS Gốc)

| Nguồn | Ghi Chú |
|-------|---------|
| **Amazon S3** | CSV, JSON, Parquet, Excel |
| **Amazon Athena** | Truy vấn trực tiếp S3 data lake |
| **Amazon Redshift** | MPP Data Warehouse |
| **Amazon RDS / Aurora** | MySQL, PostgreSQL, SQL Server |
| **AWS IoT Analytics** | Dữ liệu IoT (Internet of Things) |

### External Sources (Nguồn Bên Ngoài)

| Nguồn | Ghi Chú |
|-------|---------|
| **Salesforce** | CRM data |
| **Snowflake** | Cloud Data Warehouse |
| **Jira** | Project tracking |
| **Adobe Analytics** | Web analytics |
| **Custom JDBC/ODBC** | Bất kỳ database nào hỗ trợ chuẩn này |

---

## 📦 Các Thành Phần Chính

### 1. Dataset — Bộ Dữ Liệu

Kết nối đến nguồn dữ liệu, định nghĩa schema (Lược Đồ), tạo calculated fields (Trường Tính Toán) và filters (Bộ Lọc).

```
Data Source ──▶ Dataset ──▶ Analysis ──▶ Dashboard
```

**Hai chế độ lưu trữ:**
- **SPICE mode:** Import dữ liệu vào in-memory engine — nhanh hơn, giới hạn dung lượng
- **Direct Query mode:** Truy vấn trực tiếp nguồn dữ liệu — real-time (Thời Gian Thực) nhưng phụ thuộc vào tốc độ nguồn

### 2. Analysis — Phân Tích

Workspace (Không Gian Làm Việc) để xây dựng visual (Biểu Đồ), áp dụng filter (Bộ Lọc), tạo parameter (Tham Số) và calculated field (Trường Tính Toán).

### 3. Dashboard — Bảng Điều Khiển

Bản xuất bản (published version) của Analysis — chỉ đọc, chia sẻ với người dùng cuối. Có thể đặt lịch refresh (Làm Mới) tự động.

### 4. Story — Câu Chuyện Dữ Liệu

Narrative (Tường Thuật) dạng slideshow kết hợp visual và text để kể câu chuyện từ dữ liệu.

---

## 💰 Mô Hình Tính Phí

### Standard Edition (Phiên Bản Chuẩn)

| Thành Phần | Phí |
|-----------|-----|
| **Author** (Tác Giả) | ~$18/tháng/người — tạo và sửa dashboard |
| **Reader** (Người Đọc) | ~$0.30/phiên, tối đa $5/tháng/người |
| **SPICE capacity** | ~$0.25/GB/tháng |

### Enterprise Edition (Phiên Bản Doanh Nghiệp)

Thêm: encryption at rest (Mã Hóa Khi Lưu Trữ), VPC connectivity (Kết Nối VPC), Active Directory, row-level security (Bảo Mật Cấp Hàng).

---

## 🆚 So Sánh Nhanh

| Tiêu Chí | QuickSight | Tableau | Power BI |
|----------|-----------|---------|----------|
| **Hạ tầng** | Serverless | Server/Cloud | Cloud |
| **Chi phí** | Pay-per-session | Per-user license | Per-user license |
| **Tích hợp AWS** | Rất mạnh (native) | Plugin | Plugin |
| **Embedded** | Có (SDK) | Có (Server) | Có (Embedded) |
| **Offline** | Không | Có | Có |
| **Learning curve** | Thấp | Cao | Trung bình |

---

## 🔗 Điều Hướng Module

| Chủ Đề | File |
|--------|------|
| Datasets, Analyses, Dashboards | [1-quicksight-basics.md](./1-quicksight-basics.md) |
| SPICE Engine — In-memory cache | [2-spice-engine.md](./2-spice-engine.md) |
| Embedded Analytics — Nhúng vào app | [3-embedded-analytics.md](./3-embedded-analytics.md) |
| Module trước: Lake Formation | [../07-lake-formation/README.md](../07-lake-formation/README.md) |
| Module sau: OpenSearch | [../09-opensearch/README.md](../09-opensearch/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
