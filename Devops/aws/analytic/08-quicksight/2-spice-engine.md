# SPICE Engine — Super-fast Parallel In-memory Calculation Engine

> SPICE (Super-fast Parallel In-memory Calculation Engine — Công Cụ Tính Toán Bộ Nhớ Song Song Siêu Nhanh) là in-memory database (Cơ Sở Dữ Liệu Bộ Nhớ Trong) của QuickSight, cho phép query tốc độ cao mà không phụ thuộc vào hiệu suất của nguồn dữ liệu gốc.

---

## 📚 Mục Lục

1. [SPICE Là Gì?](#1-spice-là-gì)
2. [Kiến Trúc SPICE](#2-kiến-trúc-spice)
3. [SPICE vs Direct Query — So Sánh](#3-spice-vs-direct-query--so-sánh)
4. [Import Data vào SPICE](#4-import-data-vào-spice)
5. [Incremental Refresh — Làm Mới Tăng Dần](#5-incremental-refresh--làm-mới-tăng-dần)
6. [Quản Lý SPICE Capacity](#6-quản-lý-spice-capacity)
7. [Tối Ưu SPICE](#7-tối-ưu-spice)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. SPICE Là Gì?

SPICE là **in-memory columnar storage engine** (Bộ Lưu Trữ Cột Bộ Nhớ Trong) được AWS xây dựng riêng cho QuickSight. Khi dữ liệu được import vào SPICE:

- Dữ liệu được lưu trữ ở dạng **columnar format** (Định Dạng Cột) — tương tự Parquet
- Nén dữ liệu ở mức cao bằng các thuật toán tối ưu cho analytics
- Phân tán dữ liệu trên nhiều node để xử lý song song (Parallel Processing)
- Query được thực hiện hoàn toàn trong RAM — không cần đọc từ disk

### Tên Viết Tắt

| Chữ Cái | Tiếng Anh | Tiếng Việt |
|---------|-----------|------------|
| **S** | Super-fast | Siêu Nhanh |
| **P** | Parallel | Song Song |
| **I** | In-memory | Bộ Nhớ Trong |
| **C** | Calculation | Tính Toán |
| **E** | Engine | Công Cụ |

---

## 2. Kiến Trúc SPICE

```
┌─────────────────────────────────────────────────────────────┐
│                      SPICE Cluster                           │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Node 1  │  │  Node 2  │  │  Node 3  │  │  Node N  │   │
│  │ (RAM)    │  │ (RAM)    │  │ (RAM)    │  │ (RAM)    │   │
│  │          │  │          │  │          │  │          │   │
│  │[Col: id] │  │[Col: id] │  │[Col: id] │  │[Col: id] │   │
│  │[Col: rev]│  │[Col: rev]│  │[Col: rev]│  │[Col: rev]│   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│         │            │            │            │             │
│         └────────────┴────────────┴────────────┘            │
│                    Parallel Query Coordinator                │
│                    (Điều Phối Query Song Song)               │
└─────────────────────────────────────────────────────────────┘
              ▲                          │
              │ Import (Nhập)            │ Query Result
              │                          ▼
         Data Sources              QuickSight Visual
    (S3, Redshift, Athena...)      (Kết Quả Hiển Thị)
```

### Cách SPICE Xử Lý Query

```
User click visual
    │
    ▼
QuickSight Query Engine
    │
    ▼
SPICE Parallel Scan (Quét Song Song)
    ├── Node 1: Scan column "revenue" rows 1-1M
    ├── Node 2: Scan column "revenue" rows 1M-2M
    └── Node 3: Scan column "revenue" rows 2M-3M
    │
    ▼
Aggregate & Merge Results
    │
    ▼
Return to Visual (<1 giây cho hàng triệu rows)
```

---

## 3. SPICE vs Direct Query — So Sánh

| Tiêu Chí | SPICE | Direct Query |
|----------|-------|--------------|
| **Tốc độ query** | Rất nhanh (ms-giây) | Phụ thuộc nguồn |
| **Dữ liệu real-time** | Không — refresh theo lịch | Có — mỗi query là live |
| **Chi phí** | Phí SPICE capacity ($0.25/GB/tháng) | Phí từ nguồn (Athena, Redshift) |
| **Giới hạn dữ liệu** | Quota SPICE (mặc định 10 GB/author) | Không giới hạn |
| **Concurrent users** | Tốt — không ảnh hưởng nguồn gốc | Có thể gây tải cho nguồn |
| **Nguồn offline** | Vẫn truy vấn được | Không truy vấn được |
| **Độ phức tạp** | Cần lên lịch refresh | Đơn giản hơn |

### Khi Nào Chọn SPICE?

```
✅ Chọn SPICE khi:
├── Nhiều người dùng đồng thời (>10 concurrent users)
├── Nguồn dữ liệu chậm (RDS, on-premises database)
├── Dữ liệu không thay đổi theo giờ (daily refresh đủ)
├── Muốn cô lập tải từ production database
└── Cần query performance nhất quán

✅ Chọn Direct Query khi:
├── Cần real-time data (thay đổi từng giây)
├── Dữ liệu quá lớn cho SPICE quota
├── Nguồn là Redshift/Athena (đã đủ nhanh)
└── Chi phí SPICE quá cao so với chi phí nguồn
```

---

## 4. Import Data vào SPICE

### 4.1 Full Refresh — Làm Mới Toàn Bộ

Import toàn bộ dữ liệu từ nguồn vào SPICE, xóa dữ liệu cũ.

```
Full Refresh Process:
1. QuickSight kết nối đến nguồn
2. Đọc toàn bộ dữ liệu (SELECT * FROM table)
3. Transform sang SPICE columnar format
4. Xóa dữ liệu SPICE cũ
5. Ghi dữ liệu mới vào SPICE
6. Swap (Chuyển Đổi) sang dataset mới (atomic operation)
```

**Ưu điểm:** Đơn giản, dữ liệu luôn nhất quán.
**Nhược điểm:** Tốn thời gian và băng thông với dữ liệu lớn.

### 4.2 Incremental Refresh — Làm Mới Tăng Dần

Chỉ import dữ liệu mới hoặc đã thay đổi trong một khoảng thời gian.

```
Incremental Refresh Process:
1. Xác định window_column (cột timestamp/date)
2. Xác định window_size (ví dụ: 3 days)
3. QuickSight query: WHERE date >= (NOW() - 3 days)
4. Import chỉ dữ liệu trong window
5. Merge vào SPICE (replace dữ liệu trong window, giữ phần còn lại)
```

**Ví dụ cấu hình:**

```
Dataset: orders_dataset
Incremental refresh settings:
    Time column: order_date
    Window size: 7 days

→ Mỗi lần refresh, chỉ import 7 ngày gần nhất
→ Nhanh hơn full refresh 10-100x với dữ liệu lịch sử lớn
```

**Lưu ý quan trọng với Incremental Refresh:**

```
⚠️ Cảnh báo:
- Dữ liệu bên ngoài window KHÔNG bao giờ được refresh
- Nếu cần sửa lỗi dữ liệu cũ → phải full refresh
- Cột date/timestamp phải có index ở nguồn để nhanh
- Không hỗ trợ delete (xóa record cũ ra khỏi SPICE)
```

---

## 5. Incremental Refresh — Làm Mới Tăng Dần

### 5.1 Cấu Hình Qua Console

```
Dataset → Edit dataset
    → Schedule refresh
    → Incremental refresh
    → Select time column: order_date
    → Window size: 3 days
    → Start refresh time: 02:00 UTC daily
```

### 5.2 Cấu Hình Qua API (AWS CLI)

```bash
# Tạo incremental refresh schedule
aws quicksight create-refresh-schedule \
  --aws-account-id 123456789012 \
  --data-set-id my-dataset-id \
  --schedule '{
    "ScheduleId": "daily-incremental",
    "ScheduleFrequency": {
      "Interval": "DAILY",
      "TimeOfTheDay": "02:00"
    },
    "RefreshType": "INCREMENTAL_REFRESH",
    "IncrementalRefresh": {
      "LookbackWindow": {
        "ColumnName": "order_date",
        "Size": 7,
        "SizeUnit": "DAY"
      }
    }
  }'
```

### 5.3 Trigger Refresh Thủ Công

```bash
# Trigger full refresh ngay lập tức
aws quicksight create-ingestion \
  --aws-account-id 123456789012 \
  --data-set-id my-dataset-id \
  --ingestion-id manual-refresh-$(date +%s) \
  --ingestion-type FULL_REFRESH

# Kiểm tra trạng thái
aws quicksight describe-ingestion \
  --aws-account-id 123456789012 \
  --data-set-id my-dataset-id \
  --ingestion-id manual-refresh-1716000000
```

**Trạng thái ingestion (Trạng Thái Nhập Dữ Liệu):**

```
INITIALIZED → QUEUED → RUNNING → COMPLETED
                                   └── (hoặc) FAILED
```

---

## 6. Quản Lý SPICE Capacity

### 6.1 Quota Mặc Định

| Phiên Bản | Dung Lượng Mặc Định |
|-----------|---------------------|
| **Standard** | 10 GB / Author |
| **Enterprise** | 10 GB / Author |
| **Tối đa mua thêm** | Không giới hạn (trả phí) |

**Lưu ý:** SPICE capacity được chia sẻ trong cùng một AWS account và AWS Region (Vùng AWS). Tất cả Authors dùng chung pool.

### 6.2 Xem Lượng SPICE Đã Dùng

```
Console: QuickSight → Manage QuickSight → SPICE capacity
```

```bash
# CLI
aws quicksight describe-account-subscription \
  --aws-account-id 123456789012

# Xem capacity usage
aws quicksight list-ingestions \
  --aws-account-id 123456789012 \
  --data-set-id my-dataset-id
```

### 6.3 Mua Thêm SPICE Capacity

```
QuickSight Console
    → Manage QuickSight
    → SPICE capacity
    → Purchase additional capacity
    → Chọn số GB cần thêm
    → Xác nhận ($0.25/GB/tháng)
```

---

## 7. Tối Ưu SPICE

### 7.1 Giảm Dung Lượng SPICE

**Chiến lược 1: Chỉ import cột cần thiết**

```sql
-- Thay vì import toàn bộ bảng:
SELECT * FROM orders   -- ❌ Lãng phí SPICE

-- Chỉ import cột sẽ dùng trong dashboard:
SELECT order_id, order_date, region, product_id, revenue, quantity
FROM orders             -- ✅ Tiết kiệm SPICE
```

**Chiến lược 2: Filter trước khi import**

```sql
-- Chỉ import dữ liệu 2 năm gần nhất
SELECT *
FROM orders
WHERE order_date >= DATEADD(year, -2, GETDATE())  -- ✅
```

**Chiến lược 3: Pre-aggregate ở nguồn**

```sql
-- Thay vì import raw transactions (hàng triệu rows):
SELECT order_date, region, product_id,
       SUM(revenue) AS total_revenue,
       COUNT(*) AS order_count
FROM orders
GROUP BY order_date, region, product_id
-- → Giảm 100x số rows nhưng vẫn đủ cho dashboard cấp ngày
```

### 7.2 Tối Ưu Refresh Performance

```
Tips để refresh nhanh hơn:
├── Dùng Incremental Refresh thay Full Refresh khi có thể
├── Refresh vào giờ thấp điểm (02:00-05:00 UTC)
├── Đảm bảo cột timestamp có index ở nguồn
├── Tránh refresh nhiều dataset cùng lúc
└── Dùng S3 manifest file cho large flat files
```

### 7.3 Monitor Refresh Failures — Giám Sát Lỗi Làm Mới

```bash
# Liệt kê tất cả ingestion và trạng thái
aws quicksight list-ingestions \
  --aws-account-id 123456789012 \
  --data-set-id my-dataset-id \
  --query 'Ingestions[*].{Status:IngestionStatus,Time:CreatedTime,Rows:RowInfo}' \
  --output table
```

**Các lỗi thường gặp khi refresh:**

| Lỗi | Nguyên Nhân | Giải Pháp |
|-----|-------------|-----------|
| `SOURCE_ERROR` | Nguồn dữ liệu không phản hồi | Kiểm tra kết nối, VPC settings |
| `SPICE_TABLE_NOT_FOUND` | Dataset bị xóa khỏi SPICE | Trigger full refresh |
| `QUOTA_EXCEEDED` | Vượt SPICE quota | Mua thêm capacity hoặc xóa dataset cũ |
| `IAM_PERMISSION_ERROR` | Thiếu quyền đọc nguồn | Kiểm tra IAM role của QuickSight |

---

## 8. Câu Hỏi Phỏng Vấn

### Q1: SPICE khác gì so với việc query trực tiếp Redshift hay Athena?

**Trả lời:**
- **SPICE** import dữ liệu vào in-memory engine của QuickSight — query không đụng đến Redshift/Athena. Cực nhanh, không tốn chi phí Redshift/Athena, cô lập tải khỏi production.
- **Direct Query** gửi query đến Redshift/Athena mỗi khi user tương tác với visual. Dữ liệu real-time nhưng tốn compute cost và có thể làm chậm nguồn nếu nhiều user đồng thời.
- **Chọn SPICE** khi cần hiệu suất cao và nhiều concurrent user. **Chọn Direct Query** khi cần real-time và nguồn đã đủ nhanh.

### Q2: Incremental Refresh giải quyết vấn đề gì?

**Trả lời:** Với dữ liệu lớn (ví dụ: 500 GB lịch sử 5 năm), Full Refresh tốn nhiều thời gian và băng thông mỗi lần. Incremental Refresh chỉ import dữ liệu trong "lookback window" (ví dụ: 7 ngày gần nhất) — nhanh hơn 10-100x. Trade-off: dữ liệu ngoài window không bao giờ tự cập nhật; nếu cần sửa dữ liệu cũ phải chạy full refresh thủ công.

### Q3: Tại sao SPICE có giới hạn dung lượng và làm thế nào để quản lý?

**Trả lời:** SPICE dùng in-memory storage trên cluster được AWS quản lý — có chi phí thực. Mặc định mỗi Author có 10 GB, chia sẻ trong account. Để quản lý:
1. Pre-aggregate dữ liệu trước khi import
2. Chỉ import cột thực sự dùng trong dashboard
3. Dùng date filter để chỉ giữ dữ liệu N năm gần nhất
4. Mua thêm capacity ($0.25/GB/tháng) nếu cần
5. Xóa dataset không còn dùng để giải phóng SPICE

### Q4: Khi SPICE refresh bị lỗi, dashboard có bị ảnh hưởng không?

**Trả lời:** Không — SPICE dùng cơ chế **atomic swap** (Hoán Đổi Nguyên Tử). Refresh tạo ra bản mới hoàn chỉnh trước, rồi mới swap sang. Nếu refresh thất bại, SPICE giữ nguyên dữ liệu cũ — dashboard vẫn hoạt động bình thường với dữ liệu của lần refresh trước. Người dùng cần setup CloudWatch alert để biết khi refresh thất bại.

---

## 🔗 Liên Kết Tiếp Theo

- [QuickSight Basics — Dataset, Analysis, Dashboard](./1-quicksight-basics.md)
- [Embedded Analytics — Nhúng vào ứng dụng](./3-embedded-analytics.md)

---

**Cập Nhật Lần Cuối:** 2026-05-17
