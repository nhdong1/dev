# 10. Advanced Topics — Chủ Đề Nâng Cao

> Chủ đề nâng cao về AWS Database Services: Redshift (Data Warehouse — Kho Dữ Liệu), Aurora Global Database (Cơ Sở Dữ Liệu Toàn Cầu), DynamoDB Global Tables (Bảng Toàn Cầu), Neptune (Cơ Sở Dữ Liệu Đồ Thị), DocumentDB (Cơ Sở Dữ Liệu Tài Liệu), và Timestream (Cơ Sở Dữ Liệu Chuỗi Thời Gian).

## 📚 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Các Dịch Vụ Trong Section Này](#các-dịch-vụ-trong-section-này)
3. [Kiến Trúc Multi-Region](#kiến-trúc-multi-region)
4. [Lộ Trình Học](#lộ-trình-học)
5. [Khi Nào Dùng Dịch Vụ Nào](#khi-nào-dùng-dịch-vụ-nào)

---

## Tổng Quan

Section này bao gồm các dịch vụ AWS Database chuyên biệt và các kỹ thuật nâng cao dành cho kỹ sư senior. Mỗi dịch vụ giải quyết một bài toán khác nhau mà các database thông thường (RDS, DynamoDB) không xử lý hiệu quả.

### Tại Sao Cần Học Advanced Topics?

- **Phỏng vấn senior roles:** Câu hỏi về Redshift, Neptune, Global Tables xuất hiện thường xuyên
- **System design:** Biết chọn đúng database cho đúng bài toán là điểm mấu chốt
- **Production readiness:** Multi-region, conflict resolution, eventual consistency là kỹ năng cần thiết cho hệ thống lớn

---

## Các Dịch Vụ Trong Section Này

| File | Dịch Vụ | Bài Toán Giải Quyết | Độ Khó |
|------|---------|----------------------|--------|
| `1-redshift.md` | Amazon Redshift | OLAP (Online Analytical Processing — Xử Lý Phân Tích Trực Tuyến), Data Warehouse | ⭐⭐⭐ |
| `2-aurora-global-advanced.md` | Aurora Global Database | Multi-region active-active (Đa Vùng Chủ-Chủ) | ⭐⭐⭐ |
| `3-dynamodb-global-tables.md` | DynamoDB Global Tables | NoSQL multi-region replication (Sao Chép Đa Vùng) | ⭐⭐⭐ |
| `4-neptune.md` | Amazon Neptune | Graph database (Cơ Sở Dữ Liệu Đồ Thị), relationships | ⭐⭐⭐ |
| `5-documentdb.md` | Amazon DocumentDB | MongoDB-compatible (Tương Thích MongoDB), document store | ⭐⭐ |
| `6-timestream.md` | Amazon Timestream | Time-series (Chuỗi Thời Gian), IoT metrics | ⭐⭐ |

---

## Kiến Trúc Multi-Region

Multi-region architecture (kiến trúc đa vùng) là chủ đề xuyên suốt trong section này. Hai dịch vụ chính hỗ trợ mô hình này:

### Aurora Global Database vs DynamoDB Global Tables

```
┌─────────────────────────────────────────────────────────────────┐
│                    MULTI-REGION COMPARISON                       │
├─────────────────────┬───────────────────────────────────────────┤
│  Aurora Global DB   │       DynamoDB Global Tables              │
├─────────────────────┼───────────────────────────────────────────┤
│  SQL (Quan hệ)      │  NoSQL (Key-value / Document)             │
│  1 writer region    │  Multi-master (tất cả region đều ghi)     │
│  Cross-region lag   │  Eventual consistency (≤1 giây)           │
│  ~1 giây RPO        │  Last-write-wins conflict resolution       │
│  RTO < 1 phút       │  Global replication ~1 giây               │
└─────────────────────┴───────────────────────────────────────────┘
```

### Khi Nào Dùng Active-Active vs Active-Passive?

| Mô Hình | Mô Tả | Use Case |
|---------|-------|----------|
| **Active-Passive** (Chủ-Dự Phòng) | Chỉ 1 region nhận ghi, region còn lại chờ failover | DR (Disaster Recovery — Khôi Phục Thảm Họa) với Aurora Global DB |
| **Active-Active** (Chủ-Chủ) | Tất cả region đều ghi, conflict resolution cần thiết | DynamoDB Global Tables, ứng dụng toàn cầu |

---

## Lộ Trình Học

### Beginner → Intermediate

```
1. Đọc về Redshift — hiểu OLAP vs OLTP (Xử Lý Giao Dịch Trực Tuyến)
2. Học DocumentDB — nếu team đang dùng MongoDB
3. Học Timestream — nếu có IoT hoặc metrics use case
```

### Intermediate → Senior

```
1. Aurora Global Database advanced patterns
2. DynamoDB Global Tables — conflict resolution
3. Neptune — graph traversal queries
```

### Thứ Tự Khuyên Dùng

```
Redshift → DocumentDB → Timestream → Aurora Global → DynamoDB Global → Neptune
   (phổ biến nhất)                                                   (chuyên biệt nhất)
```

---

## Khi Nào Dùng Dịch Vụ Nào

### Ma Trận Quyết Định

```
Câu hỏi 1: Dữ liệu có cấu trúc quan hệ (bảng, foreign key)?
  → Có → Redshift (nếu OLAP) hoặc Aurora Global (nếu OLTP đa vùng)
  → Không → Tiếp tục

Câu hỏi 2: Dữ liệu là document (JSON, BSON)?
  → Có → DocumentDB (đặc biệt nếu migrate từ MongoDB)
  → Không → Tiếp tục

Câu hỏi 3: Dữ liệu là time-series (timestamp + metrics)?
  → Có → Timestream
  → Không → Tiếp tục

Câu hỏi 4: Dữ liệu là đồ thị (nodes, edges, relationships)?
  → Có → Neptune
  → Không → DynamoDB Global Tables (key-value đa vùng)
```

### Bảng Tóm Tắt Nhanh

| Bài Toán | Dịch Vụ Phù Hợp |
|----------|-----------------|
| Phân tích dữ liệu lớn (terabyte+) | **Redshift** |
| SQL đa vùng với HA tốt | **Aurora Global Database** |
| NoSQL đa vùng, write ở mọi region | **DynamoDB Global Tables** |
| Quan hệ phức tạp (social graph, fraud detection) | **Neptune** |
| Document store, migrate từ MongoDB | **DocumentDB** |
| IoT sensor data, metrics theo thời gian | **Timestream** |

---

## Điều Hướng Nhanh

| Chủ Đề | File |
|--------|------|
| Redshift — OLAP & Distribution | [1-redshift.md](./1-redshift.md) |
| Aurora Global — Active-Active Advanced | [2-aurora-global-advanced.md](./2-aurora-global-advanced.md) |
| DynamoDB Global Tables | [3-dynamodb-global-tables.md](./3-dynamodb-global-tables.md) |
| Neptune — Graph Database | [4-neptune.md](./4-neptune.md) |
| DocumentDB — MongoDB Compatible | [5-documentdb.md](./5-documentdb.md) |
| Timestream — IoT & Time-Series | [6-timestream.md](./6-timestream.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
