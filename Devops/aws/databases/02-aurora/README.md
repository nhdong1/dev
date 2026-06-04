# 02 — Aurora — Cơ Sở Dữ Liệu Đám Mây Hiệu Năng Cao

> Amazon Aurora là engine cơ sở dữ liệu quan hệ được AWS xây dựng từ đầu cho môi trường cloud (điện toán đám mây), tương thích với MySQL và PostgreSQL nhưng đạt hiệu năng cao hơn đáng kể nhờ kiến trúc shared storage (lưu trữ dùng chung) phân tán độc đáo.

## 📚 Mục Lục

1. [Tại Sao Aurora Ra Đời?](#tại-sao-aurora-ra-đời)
2. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
3. [Các Thành Phần Cốt Lõi](#các-thành-phần-cốt-lõi)
4. [Aurora vs RDS — So Sánh Nhanh](#aurora-vs-rds--so-sánh-nhanh)
5. [Nội Dung Chi Tiết](#nội-dung-chi-tiết)
6. [Khi Nào Nên Dùng Aurora](#khi-nào-nên-dùng-aurora)
7. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Aurora Ra Đời?

RDS (Relational Database Service — Dịch Vụ Cơ Sở Dữ Liệu Quan Hệ) truyền thống có một giới hạn cơ bản: nó lấy MySQL/PostgreSQL nguyên bản và đặt vào cloud mà không thay đổi kiến trúc storage (lưu trữ). Điều này dẫn đến các bottleneck (điểm nghẽn cổ chai):

- **Replication lag** (độ trễ sao chép): Standby và Read Replicas phải replay transaction logs
- **Storage bound by one AZ** (lưu trữ bị giới hạn một vùng): EBS volume chỉ trong một AZ
- **Failover chậm**: 60–120 giây để standby tiếp quản

Aurora ra đời năm 2014 với triết lý: **log is the database** — thay vì ghi dữ liệu xong rồi ghi log, chỉ cần ghi log và để storage layer tự dựng lại dữ liệu.

### Kết Quả Đo Lường Thực Tế

| Chỉ Số                                          | RDS MySQL          | Aurora MySQL       | Cải Thiện |
| ----------------------------------------------- | ------------------ | ------------------ | --------- |
| **Throughput** (Thông Lượng)                    | ~30K writes/giây   | ~200K writes/giây  | ~5–6×     |
| **Failover time** (Thời Gian Chuyển Đổi Dự Phòng) | 60–120 giây      | 15–30 giây         | ~4×       |
| **Read Replica lag** (Độ Trễ Bản Sao Đọc)      | Giây đến phút      | Milli-giây         | ~100×     |
| **Storage limit** (Giới Hạn Lưu Trữ)           | 64 TB              | 128 TB             | 2×        |
| **Storage replication** (Sao Chép Lưu Trữ)     | 1 bản (EBS)        | 6 bản (3 AZ)       | 6×        |

---

## Kiến Trúc Tổng Quan

### Sự Khác Biệt Nền Tảng

```
RDS Architecture (Kiến Trúc RDS):
┌─────────────┐     binlog       ┌─────────────┐
│   Primary   │ ─────────────→  │  Read Rep.  │
│  (EBS Vol)  │                  │  (EBS Vol)  │
└─────────────┘
    Mỗi instance có storage riêng, replication qua binlog

Aurora Architecture (Kiến Trúc Aurora):
┌──────────────────────────────────────────────────┐
│              Shared Storage Volume               │
│         (Redo Log — Nhật Ký Ghi Lại)            │
│   6 copies across 3 AZs (6 bản trên 3 AZ)       │
│  AZ-a: [copy1] [copy2]                           │
│  AZ-b: [copy3] [copy4]                           │
│  AZ-c: [copy5] [copy6]                           │
└───┬──────────────┬───────────────┬───────────────┘
    │              │               │
┌───▼───┐     ┌───▼───┐      ┌───▼───┐
│Writer │     │Reader │      │Reader │
│(1 cái)│     │(tối đa│      │15 cái)│
└───────┘     └───────┘      └───────┘
    Tất cả instances chia sẻ cùng storage — không cần replication
```

### Nguyên Lý "Log Is The Database"

Aurora chỉ truyền **redo log records** (bản ghi nhật ký ghi lại) từ Writer đến storage layer — không truyền data pages. Storage layer tự áp dụng log và dựng lại data pages khi cần đọc:

```
Truyền thống (RDS):
  Writer → ghi data page + ghi redo log → gửi cả hai đến Standby

Aurora:
  Writer → chỉ ghi redo log → gửi log đến 6 storage nodes
  Storage nodes tự áp dụng log để tạo data pages
  → Lượng network I/O giảm ~7.7 lần
```

---

## Các Thành Phần Cốt Lõi

### Aurora Cluster (Cụm Aurora)

Một Aurora cluster bao gồm:

| Thành Phần                                          | Số Lượng        | Vai Trò                                    |
| --------------------------------------------------- | --------------- | ------------------------------------------ |
| **Writer instance** (Máy Chủ Ghi)                  | 1               | Xử lý tất cả write operations              |
| **Reader instances** (Máy Chủ Đọc)                 | 0–15            | Phục vụ read-only queries                  |
| **Shared cluster volume** (Ổ Lưu Trữ Dùng Chung)  | 1 (6 bản sao)   | Dữ liệu thực tế, chia sẻ bởi tất cả       |
| **Cluster endpoint** (Điểm Truy Cập Cụm)           | 1               | Trỏ đến Writer — dùng cho write/read-write |
| **Reader endpoint** (Điểm Truy Cập Đọc)            | 1               | Load balance (Cân Bằng Tải) đến Readers   |
| **Instance endpoints** (Điểm Truy Cập Máy Chủ)    | 1 per instance  | Kết nối trực tiếp đến từng instance        |

### Aurora Storage (Lưu Trữ Aurora)

- **Tự động tăng:** Bắt đầu từ 10 GB, tự động tăng theo 10 GB increments (bước tăng), tối đa 128 TB
- **6-way replication** (sao chép 6 chiều): 2 copies per AZ × 3 AZs
- **Quorum writes** (ghi theo đa số): Cần 4/6 thành công để xác nhận write
- **Quorum reads** (đọc theo đa số): Cần 3/6 để đảm bảo đọc đúng nhất
- **Self-healing** (tự chữa lành): Phát hiện và sửa lỗi data corruption tự động

---

## Aurora vs RDS — So Sánh Nhanh

| Tính Năng                                              | RDS                         | Aurora                      |
| ------------------------------------------------------ | --------------------------- | --------------------------- |
| **Engines tương thích**                                | MySQL, PostgreSQL (nguyên bản) | MySQL, PostgreSQL (compatible) |
| **Storage architecture**                               | EBS per instance            | Shared distributed storage  |
| **Max Read Replicas**                                  | 5                           | 15                          |
| **Replica lag**                                        | Giây đến phút               | Mili-giây                   |
| **Failover time**                                      | 60–120 giây                 | 15–30 giây                  |
| **Storage scaling**                                    | Thủ công hoặc auto          | Hoàn toàn tự động           |
| **Backtrack** (Quay Lại Theo Thời Gian)               | Không có                    | Có (MySQL-compatible)       |
| **Global Database** (Cơ Sở Dữ Liệu Toàn Cầu)         | Không có                    | Có                          |
| **Serverless** (Không Máy Chủ)                        | Không có                    | Aurora Serverless v2        |
| **Chi phí storage**                                    | Trả theo GB được provision  | Trả theo GB thực sự dùng    |
| **Chi phí instance**                                   | ~Tương đương                | ~20% đắt hơn RDS            |

---

## Nội Dung Chi Tiết

### Các File Trong Thư Mục Này

| File                        | Chủ Đề                                           | Trạng Thái |
| --------------------------- | ------------------------------------------------ | ---------- |
| `README.md`                 | Tổng quan Aurora (file này)                      | ✅          |
| `1-aurora-architecture.md`  | Shared storage, cluster endpoints, storage engine| ✅          |
| `2-aurora-serverless.md`    | Aurora Serverless v2, auto-scaling ACU           | ✅          |
| `3-aurora-global.md`        | Global Database, cross-region replication        | ✅          |
| `4-aurora-vs-rds.md`        | Trade-offs chi tiết, cost comparison             | ✅          |
| `5-aurora-ha-failover.md`   | High availability, failover mechanisms           | ✅          |

---

## Khi Nào Nên Dùng Aurora

### Dùng Aurora Khi

```
✅ OLTP (Online Transaction Processing — Xử Lý Giao Dịch Trực Tuyến) quy mô cao
✅ Cần High Availability (Tính Sẵn Sàng Cao) với failover < 30 giây
✅ Nhiều read-heavy workloads cần nhiều Read Replicas
✅ Cần global replication (sao chép toàn cầu) với độ trễ thấp
✅ Workload có peak traffic (lưu lượng đỉnh) không đều — dùng Serverless v2
✅ Đang dùng MySQL/PostgreSQL và muốn nâng cấp không thay đổi code
```

### Dùng RDS Thay Khi

```
❌ Budget (Ngân Sách) hạn chế — RDS rẻ hơn Aurora ~20–30%
❌ Cần Oracle hoặc SQL Server — Aurora không hỗ trợ
❌ Dev/test environment không cần HA — RDS Single-AZ đủ dùng
❌ Workload nhỏ < 10 GB storage và < 100 connections
❌ Cần custom storage engine đặc biệt của MySQL/PostgreSQL
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Q1: Aurora khác RDS ở điểm gì căn bản nhất?

**Trả lời:** Sự khác biệt căn bản là kiến trúc storage. RDS dùng EBS volume riêng cho từng instance và sao chép dữ liệu qua binlog (binary log — nhật ký nhị phân). Aurora dùng shared distributed storage (lưu trữ phân tán dùng chung) với 6 bản sao trên 3 AZ — tất cả instances trong cluster đọc từ cùng một volume. Nhờ đó Aurora không cần replication giữa Writer và Readers, giảm lag xuống còn mili-giây và failover chỉ mất 15–30 giây thay vì 60–120 giây.

### Q2: Aurora "5x faster than MySQL" có thực sự chính xác không?

**Trả lời:** Con số này từ benchmark (điểm chuẩn hiệu năng) của AWS trong điều kiện write-heavy OLTP. Trong thực tế, cải thiện phụ thuộc rất nhiều vào workload: read-heavy workloads ít thấy cải thiện hơn, còn write-intensive workloads có thể thấy 3–5× do Aurora chỉ truyền redo logs thay vì full data pages.

### Q3: Tại sao Aurora Read Replica có lag thấp hơn RDS Read Replica?

**Trả lời:** RDS Read Replicas phải nhận binlog từ Primary, parse và replay từng transaction — quá trình này tốn thời gian và CPU, dẫn đến lag từ giây đến phút. Aurora Readers truy cập trực tiếp vào shared storage volume đã có dữ liệu mới nhất — họ chỉ cần đọc data pages từ storage, không cần replay bất kỳ transaction nào. Lag thường chỉ là mili-giây.

### Q4: Aurora có phù hợp cho mọi workload không?

**Trả lời:** Không. Aurora tốt nhất cho OLTP quy mô cao, high-traffic applications cần HA. Không phù hợp cho: analytics workloads (dùng Redshift), applications cần Oracle/SQL Server, workloads nhỏ ở giai đoạn dev/test (chi phí cao hơn không cần thiết), hoặc khi team chưa quen với Aurora-specific features như cluster endpoints, backtrack, và Global Database.

---

## 🔗 Điều Hướng

| Trước                                                 | Tiếp Theo                                            |
| ----------------------------------------------------- | ---------------------------------------------------- |
| [01-rds-fundamentals/](../01-rds-fundamentals/README.md) | [1-aurora-architecture.md](./1-aurora-architecture.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
