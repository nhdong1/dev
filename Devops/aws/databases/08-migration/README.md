# Database Migration — Di Chuyển Cơ Sở Dữ Liệu trên AWS

> Hướng dẫn toàn diện về chiến lược và công cụ di chuyển cơ sở dữ liệu lên AWS — từ AWS DMS (Database Migration Service — Dịch Vụ Di Chuyển Cơ Sở Dữ Liệu) và SCT (Schema Conversion Tool — Công Cụ Chuyển Đổi Schema) đến CDC (Change Data Capture — Thu Thập Thay Đổi Dữ Liệu) và kế hoạch cutover (cắt chuyển) không có downtime (thời gian ngừng hoạt động).

## 📚 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Kiến Trúc Di Chuyển](#kiến-trúc-di-chuyển)
3. [Công Cụ Di Chuyển AWS](#công-cụ-di-chuyển-aws)
4. [Các Loại Di Chuyển](#các-loại-di-chuyển)
5. [Quy Trình Di Chuyển](#quy-trình-di-chuyển)
6. [Lộ Trình Học](#lộ-trình-học)

---

## 🎯 Tổng Quan

Database migration (di chuyển cơ sở dữ liệu) là một trong những thách thức phức tạp nhất trong kỹ thuật hạ tầng. Rủi ro cao: mất dữ liệu, downtime kéo dài, hoặc hệ thống mới không tương thích.

### Tại Sao Cần Di Chuyển Database?

| Lý Do                              | Mô Tả                                                          |
| ---------------------------------- | -------------------------------------------------------------- |
| **Cloud Migration**                | Chuyển từ on-premises lên AWS                                  |
| **Engine Upgrade**                 | Nâng cấp MySQL 5.7 → 8.0, PostgreSQL 13 → 16                  |
| **Engine Change**                  | Oracle → Aurora PostgreSQL (tiết kiệm chi phí license)        |
| **Modernization**                  | MySQL monolith → DynamoDB microservices                        |
| **Consolidation**                  | Nhiều DB nhỏ → một DB lớn được quản lý tốt hơn                |
| **Scaling**                        | Self-managed → fully managed để scale dễ hơn                  |

### Thách Thức Di Chuyển

```
Kỹ thuật:
  - Schema incompatibility (Không tương thích schema) giữa các engine
  - Data type mapping (Ánh xạ kiểu dữ liệu) khác nhau
  - Stored procedures, triggers không tương thích
  - Character encoding (Mã hóa ký tự) — utf8 vs utf8mb4

Vận hành:
  - Downtime (Thời gian ngừng) không thể chấp nhận
  - Data consistency (Tính nhất quán dữ liệu) trong quá trình di chuyển
  - Rollback (Quay lại) khi gặp sự cố
  - Validation (Xác minh) dữ liệu sau di chuyển

Tổ chức:
  - Nhiều team phụ thuộc vào cùng một DB
  - Application code cần cập nhật song song
  - Testing (Kiểm thử) toàn diện trước cutover
```

---

## 🏗️ Kiến Trúc Di Chuyển

### Luồng Di Chuyển Tổng Quát

```
┌─────────────────────────────────────────────────────────────────┐
│                    MIGRATION FLOW                                │
│                                                                  │
│  Source DB          AWS DMS             Target DB               │
│  ─────────          ───────             ─────────               │
│  │On-prem│          │Repl. │            │Aurora │               │
│  │MySQL  │──Full───▶│Task  │──────────▶│MySQL  │               │
│  │       │  Load    │      │            │       │               │
│  │       │──CDC ───▶│      │──ongoing──▶│       │               │
│  └───────┘  (real-  └──────┘  changes   └───────┘               │
│             time)                                                │
│                                                                  │
│  Phase 1: Full Load (Tải Đầy Đủ) — copy toàn bộ dữ liệu        │
│  Phase 2: CDC — đồng bộ thay đổi real-time                      │
│  Phase 3: Cutover — chuyển traffic sang target                  │
└─────────────────────────────────────────────────────────────────┘
```

### Homogeneous vs Heterogeneous Migration

```
Homogeneous Migration (Di Chuyển Cùng Loại):
  MySQL ──────────────────▶ MySQL on RDS
  PostgreSQL ─────────────▶ PostgreSQL on Aurora
  Oracle ─────────────────▶ Oracle on RDS
  
  Công cụ: AWS DMS (có thể không cần SCT)
  Độ phức tạp: Thấp–Trung

Heterogeneous Migration (Di Chuyển Khác Loại):
  Oracle ──────────────────▶ Aurora PostgreSQL
  SQL Server ──────────────▶ Aurora MySQL
  MySQL ───────────────────▶ DynamoDB
  
  Công cụ: SCT (chuyển schema) + DMS (chuyển data)
  Độ phức tạp: Cao
```

---

## 🔧 Công Cụ Di Chuyển AWS

### AWS DMS — Database Migration Service

DMS là dịch vụ managed giúp di chuyển database với minimal downtime. DMS chạy trên một **replication instance** (máy chủ sao chép) — một EC2 instance do AWS quản lý.

**Các loại task DMS:**

| Task Type                     | Mô Tả                                                    |
| ----------------------------- | -------------------------------------------------------- |
| **Full Load**                 | Copy toàn bộ dữ liệu từ source sang target một lần       |
| **CDC Only**                  | Chỉ thu thập thay đổi (dùng sau khi đã có full load)     |
| **Full Load + CDC**           | Full load rồi tiếp tục CDC — phổ biến nhất              |

Chi tiết: [1-dms-overview.md](./1-dms-overview.md)

### AWS SCT — Schema Conversion Tool

SCT là công cụ desktop giúp chuyển đổi database schema (cấu trúc cơ sở dữ liệu) từ một engine sang engine khác trong các heterogeneous migration.

- Phân tích mức độ tương thích tự động
- Chuyển đổi tables, views, indexes tự động
- Báo cáo các object cần chỉnh sửa thủ công (stored procedures, triggers)

Chi tiết: [2-sct.md](./2-sct.md)

### CDC — Change Data Capture

CDC (Change Data Capture — Thu Thập Thay Đổi Dữ Liệu) cho phép di chuyển **online** — tức là source database vẫn tiếp tục nhận write trong khi DMS đồng bộ liên tục sang target.

- MySQL/MariaDB: dùng binary log (binlog)
- PostgreSQL: dùng logical replication slots
- Oracle: dùng LogMiner hoặc Binary Reader

Chi tiết: [3-cdc-online-migration.md](./3-cdc-online-migration.md)

---

## 🗺️ Các Loại Di Chuyển

### 6R Migration Framework

Khi di chuyển lên AWS, thường áp dụng một trong các chiến lược 6R:

```
Retire    — Loại bỏ hệ thống không còn cần
Retain    — Giữ nguyên on-premises
Rehost    — Lift-and-shift: EC2 + tự quản lý DB
Replatform — Di chuyển lên managed service với ít thay đổi
Refactor  — Re-architect: thiết kế lại hoàn toàn
Repurchase — Thay thế bằng SaaS
```

Với database, 3 chiến lược quan trọng nhất:

| Chiến Lược        | Mô Tả                                               | Ví Dụ                           |
| ----------------- | --------------------------------------------------- | -------------------------------- |
| **Lift-and-Shift**| Chuyển nguyên xi lên EC2, tự quản lý               | MySQL on-prem → MySQL on EC2     |
| **Re-platform**   | Chuyển lên managed service, ít thay đổi            | MySQL on-prem → MySQL on RDS     |
| **Re-architect**  | Thiết kế lại để tận dụng cloud-native capabilities | Oracle → Aurora PostgreSQL        |

Chi tiết: [4-migration-strategies.md](./4-migration-strategies.md)

---

## 📋 Quy Trình Di Chuyển

### Giai Đoạn 1: Assessment (Đánh Giá)

```
1. Kiểm kê database: version, size, connection count, workload type
2. Chạy SCT (nếu heterogeneous) để biết % schema tự động chuyển được
3. Xác định RPO/RTO yêu cầu cho migration window
4. Lập danh sách dependencies (ứng dụng nào kết nối vào DB này)
```

### Giai Đoạn 2: Preparation (Chuẩn Bị)

```
1. Tạo target database trên AWS (RDS, Aurora, DynamoDB...)
2. Chuyển schema (dùng SCT hoặc native dump/restore)
3. Tạo DMS replication instance và endpoints
4. Test network connectivity từ DMS đến source & target
5. Bật binary logging trên source (nếu dùng CDC)
```

### Giai Đoạn 3: Full Load (Tải Đầy Đủ)

```
1. Bắt đầu DMS task: Full Load + CDC
2. Theo dõi progress trên DMS console
3. Verify row counts sau full load
4. CDC bắt đầu bắt kịp (lag giảm dần về 0)
```

### Giai Đoạn 4: Validation (Xác Minh)

```
1. Row count check: source vs target
2. Data sampling: so sánh mẫu ngẫu nhiên
3. Application testing: chạy integration tests trên target
4. Performance benchmarking trên target
```

### Giai Đoạn 5: Cutover (Cắt Chuyển)

```
1. Chọn maintenance window (cửa sổ bảo trì) — thường cuối tuần/đêm khuya
2. Thông báo người dùng về downtime dự kiến
3. Dừng writes trên source (đưa ứng dụng vào read-only hoặc maintenance mode)
4. Chờ DMS lag về 0 (tất cả thay đổi đã đồng bộ)
5. Verify final row counts
6. Cập nhật connection strings của ứng dụng
7. Smoke test trên target
8. Mở traffic cho người dùng
```

### Giai Đoạn 6: Post-Migration (Sau Di Chuyển)

```
1. Monitor application errors trong 24-48 giờ đầu
2. Theo dõi DB performance (slow queries, CPU, connections)
3. Giữ source database chạy thêm 1-2 tuần (sẵn sàng rollback)
4. Xóa DMS task và replication instance khi ổn định
```

Chi tiết cutover: [5-cutover-runbook.md](./5-cutover-runbook.md)

---

## 📁 Lộ Trình Học

```
08-migration/
├── README.md                   ← Bạn đang ở đây
├── 1-dms-overview.md           AWS DMS architecture, task types, endpoints
├── 2-sct.md                    Schema Conversion Tool, heterogeneous migration
├── 3-cdc-online-migration.md   CDC, online migration, zero-downtime techniques
├── 4-migration-strategies.md   Lift-and-shift, re-platform, re-architect
└── 5-cutover-runbook.md        Cutover planning, rollback, validation checklist
```

### Học Theo Thứ Tự

| Bước | File                         | Nội Dung                                         | Thời Gian |
| ---- | ---------------------------- | ------------------------------------------------ | --------- |
| 1    | README.md (file này)         | Tổng quan, kiến trúc, quy trình                  | 30 phút   |
| 2    | 1-dms-overview.md            | Hiểu DMS từ trong ra ngoài                       | 60 phút   |
| 3    | 2-sct.md                     | SCT cho heterogeneous migration                  | 45 phút   |
| 4    | 3-cdc-online-migration.md    | CDC — kỹ thuật zero-downtime                     | 60 phút   |
| 5    | 4-migration-strategies.md    | Chọn chiến lược phù hợp                          | 45 phút   |
| 6    | 5-cutover-runbook.md         | Thực hành runbook cutover                        | 60 phút   |

**Tổng thời gian: ~5 giờ để nắm vững DB migration trên AWS**

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

- DMS (Database Migration Service) là gì và khi nào dùng?
- Sự khác biệt giữa homogeneous và heterogeneous migration?
- CDC (Change Data Capture) hoạt động như thế nào?
- Làm sao để thực hiện zero-downtime database migration?
- Khi nào dùng SCT (Schema Conversion Tool)?
- Giải thích các bước trong một migration cutover?
- Sự khác biệt giữa lift-and-shift, re-platform, và re-architect?
- Làm sao để validate dữ liệu sau khi di chuyển?

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
