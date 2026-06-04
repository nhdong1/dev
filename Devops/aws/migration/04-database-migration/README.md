# AWS Database Migration Service (DMS) & SCT — Tổng Quan

> Di chuyển cơ sở dữ liệu (database migration) là một trong những phần phức tạp và rủi ro nhất của bất kỳ dự án migration nào. AWS cung cấp hai công cụ chủ lực: **AWS DMS** — Database Migration Service (Dịch Vụ Di Chuyển Cơ Sở Dữ Liệu) và **AWS SCT** — Schema Conversion Tool (Công Cụ Chuyển Đổi Schema). Bộ đôi này giải quyết toàn bộ vòng đời di chuyển CSDL, từ chuyển đổi schema đến tải dữ liệu và đồng bộ liên tục.

## 📚 Mục Lục (Table of Contents)

1. [Tại Sao DB Migration Phức Tạp?](#tại-sao-db-migration-phức-tạp)
2. [Hai Công Cụ Cốt Lõi](#hai-công-cụ-cốt-lõi)
3. [Hai Loại DB Migration Chính](#hai-loại-db-migration-chính)
4. [Luồng Migration Điển Hình](#luồng-migration-điển-hình)
5. [So Sánh DMS và SCT](#so-sánh-dms-và-sct)
6. [Điều Hướng Tài Liệu](#điều-hướng-tài-liệu)

---

## 🎯 Tại Sao DB Migration Phức Tạp?

Di chuyển server (dùng MGN) tương đối đơn giản: sao chép block-by-block, launch EC2. Nhưng di chuyển database phức tạp hơn nhiều:

```
Thách thức khi di chuyển database:

1. Dữ liệu không thể dừng (24/7 availability):
   └── Không thể tắt DB của production để di chuyển → cần CDC liên tục

2. Schema không tương thích (heterogeneous migration):
   ├── Oracle dùng NUMBER, PostgreSQL dùng NUMERIC
   ├── SQL Server dùng NVARCHAR, MySQL dùng VARCHAR
   └── Stored procedures, triggers, views có cú pháp khác nhau

3. Volume dữ liệu lớn:
   ├── Full load có thể mất hàng giờ đến hàng ngày
   └── Trong thời gian đó, dữ liệu mới vẫn liên tục được ghi vào source

4. Tính toàn vẹn dữ liệu (Data Integrity):
   ├── Foreign keys, constraints, indexes phải được duy trì
   └── Kiểm tra record count, checksum sau migration

5. Performance sau migration:
   └── Query optimizer khác nhau → cần tune lại queries và indexes
```

---

## 🛠️ Hai Công Cụ Cốt Lõi

### 1. AWS DMS — Database Migration Service (Dịch Vụ Di Chuyển Cơ Sở Dữ Liệu)

**Vai trò:** Thực hiện di chuyển dữ liệu thực tế — kết nối source DB, đọc dữ liệu, ghi vào target DB.

```
DMS hoạt động theo nguyên lý:
├── Replication Instance (EC2 được quản lý bởi AWS) làm trung gian
├── Source Endpoint → kết nối vào DB nguồn (Oracle, MySQL, SQL Server...)
├── Target Endpoint → kết nối vào DB đích (Aurora, RDS, Redshift...)
└── Replication Task → định nghĩa dữ liệu nào cần di chuyển và cách nào

Ba loại task:
├── Full Load — Tải toàn bộ: tải hết data một lần (downtime)
├── Full Load + CDC — Tải toàn bộ + đồng bộ thay đổi: tải xong rồi bắt kịp
└── CDC Only — Chỉ đồng bộ thay đổi: dùng khi schema đã sẵn sàng
```

Tài liệu chi tiết: [1-dms-fundamentals.md](./1-dms-fundamentals.md)

---

### 2. AWS SCT — Schema Conversion Tool (Công Cụ Chuyển Đổi Schema)

**Vai trò:** Tự động phân tích và chuyển đổi schema, stored procedures, views, triggers từ engine này sang engine khác.

```
SCT hoạt động như thế nào:
├── Kết nối vào source DB → lấy schema đầy đủ
├── Phân tích từng object: table, view, stored procedure, function, trigger
├── Tự động chuyển đổi (conversion action report):
│   ├── Simple objects (tables, indexes) → thường 100% tự động
│   └── Complex objects (procedures, triggers) → đánh dấu cần review thủ công
└── Tạo script SQL để tạo schema trên target DB

Nguyên tắc: SCT chuyển đổi SCHEMA, DMS di chuyển DATA
```

Tài liệu chi tiết: [4-schema-conversion-tool.md](./4-schema-conversion-tool.md)

---

## 📂 Hai Loại DB Migration Chính

### Homogeneous Migration — Di Chuyển Đồng Nhất (Cùng Engine)

```
Ví dụ:
├── MySQL 5.7 (on-premises) → Amazon RDS MySQL 8.0
├── PostgreSQL (on-premises) → Amazon Aurora PostgreSQL
└── SQL Server (on-premises) → Amazon RDS SQL Server

Đặc điểm:
├── Schema tương thích (không cần SCT)
├── DMS có thể tải trực tiếp
├── Nhanh hơn và ít rủi ro hơn heterogeneous
└── Thường chỉ cần DMS (không cần SCT)
```

Tài liệu chi tiết: [2-dms-homogeneous.md](./2-dms-homogeneous.md)

---

### Heterogeneous Migration — Di Chuyển Dị Cấu Trúc (Khác Engine)

```
Ví dụ phổ biến nhất:
├── Oracle → Amazon Aurora PostgreSQL   (tiết kiệm license Oracle $$$)
├── SQL Server → Amazon Aurora MySQL
├── Oracle → Amazon Aurora MySQL
└── DB2 → Amazon RDS PostgreSQL

Đặc điểm:
├── Schema KHÔNG tương thích → PHẢI dùng SCT trước
├── Stored procedures, triggers cần chuyển đổi cú pháp
├── Phức tạp hơn, cần nhiều thời gian kiểm thử
└── Quy trình: SCT chuyển schema → DMS di chuyển data
```

Tài liệu chi tiết: [3-dms-heterogeneous.md](./3-dms-heterogeneous.md)

---

## 🔄 Luồng Migration Điển Hình

### Kịch Bản: Oracle → Aurora PostgreSQL (Zero-Downtime)

```
Giai đoạn 1: Chuẩn bị (1-2 tuần)
├── Cài AWS SCT → kết nối Oracle source
├── Tạo Assessment Report — Báo Cáo Đánh Giá
│   ├── Xem % object tự động convert được
│   └── Lập kế hoạch cho object cần sửa thủ công
├── Chuyển đổi schema (SCT) → tạo Aurora PostgreSQL schema
└── Test schema: tạo Aurora cluster, apply schema script

Giai đoạn 2: Full Load — Tải Toàn Bộ (vài giờ đến vài ngày)
├── Tạo DMS Replication Instance
├── Tạo Source Endpoint (Oracle) + Target Endpoint (Aurora)
├── Tạo Replication Task: Full Load + CDC
├── Khởi động task → DMS tải toàn bộ data từ Oracle sang Aurora
└── Theo dõi tiến trình trong DMS Console

Giai đoạn 3: CDC — Change Data Capture (song song với production)
├── Sau full load: DMS tự động chuyển sang CDC mode
├── Mọi INSERT/UPDATE/DELETE trên Oracle → replicated sang Aurora
├── Kiểm tra replication lag (độ trễ đồng bộ)
└── Aurora dần "bắt kịp" với Oracle

Giai đoạn 4: Validation — Kiểm Tra Tính Toàn Vẹn Dữ Liệu
├── So sánh record count trên từng table
├── Chạy data validation queries
└── Kiểm tra foreign keys, constraints

Giai đoạn 5: Cutover — Cắt Chuyển (downtime vài phút)
├── Maintenance window (ví dụ: 2 AM Chủ Nhật)
├── Tắt ứng dụng (application)
├── Đợi DMS sync lần cuối (drain remaining CDC events)
├── Kiểm tra lag = 0
├── Cập nhật connection string ứng dụng → Aurora
├── Khởi động lại ứng dụng
└── Kiểm tra hoạt động bình thường
```

---

## 📊 So Sánh DMS và SCT

| Tiêu Chí | AWS DMS | AWS SCT |
| -------- | ------- | ------- |
| **Chức năng** | Di chuyển DATA (dữ liệu) | Chuyển đổi SCHEMA (cấu trúc) |
| **Cài đặt** | Cloud service (không cài gì) | Phần mềm cài trên máy tính của bạn |
| **Khi nào dùng** | Mọi DB migration | Chỉ khi khác DB engine (heterogeneous) |
| **Phí** | Theo giờ replication instance | Miễn phí (tải xuống) |
| **CDC support** | ✅ Hỗ trợ đầy đủ | ❌ Không có (chỉ xử lý schema) |
| **Giao diện** | AWS Console / CLI / API | Ứng dụng desktop (GUI) |
| **Kết quả** | Data trong target DB | SQL scripts tạo schema |

> **Quy tắc nhớ nhanh:** Dùng SCT để "dịch" schema (như từ điển), dùng DMS để "vận chuyển" data (như xe tải).

---

## 🔗 Điều Hướng Tài Liệu

| File | Nội Dung |
| ---- | -------- |
| [1-dms-fundamentals.md](./1-dms-fundamentals.md) | DMS kiến trúc, replication instance, endpoints, task types, monitoring |
| [2-dms-homogeneous.md](./2-dms-homogeneous.md) | Di chuyển MySQL→MySQL, PostgreSQL→Aurora — quy trình và best practices |
| [3-dms-heterogeneous.md](./3-dms-heterogeneous.md) | Di chuyển Oracle→Aurora, SQL Server→MySQL — kết hợp SCT + DMS |
| [4-schema-conversion-tool.md](./4-schema-conversion-tool.md) | SCT cài đặt, assessment report, chuyển đổi schema, xử lý object phức tạp |
| [5-dms-cdc.md](./5-dms-cdc.md) | CDC chi tiết: binlog, LogMiner, replication slots, monitoring lag, zero-downtime |

---

## 🎓 Câu Hỏi Phỏng Vấn Điển Hình (Module Này)

1. DMS là gì? Nó khác gì với backup/restore thông thường?
2. Phân biệt homogeneous vs heterogeneous migration. Khi nào cần SCT?
3. Giải thích CDC — Change Data Capture và tại sao cần thiết cho zero-downtime migration.
4. DMS Replication Instance là gì? Chọn instance type dựa trên tiêu chí nào?
5. Chiến lược zero-downtime database migration với DMS là gì?
6. SCT có thể tự động convert 100% code không? Những phần nào thường phải sửa thủ công?
7. Sau khi DMS migration hoàn thành, làm thế nào kiểm tra tính toàn vẹn dữ liệu?

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
