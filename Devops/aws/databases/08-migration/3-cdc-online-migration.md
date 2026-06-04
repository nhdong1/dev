# CDC & Online Migration — Di Chuyển Không Downtime

> CDC (Change Data Capture — Thu Thập Thay Đổi Dữ Liệu) là kỹ thuật theo dõi và ghi lại mọi thay đổi trong database theo thời gian thực. Kết hợp CDC với AWS DMS cho phép thực hiện online migration (di chuyển trực tuyến) — source database vẫn tiếp tục phục vụ production traffic trong khi DMS đồng bộ liên tục sang target, đạt được zero-downtime hoặc minimal-downtime migration.

## 📚 Mục Lục

1. [CDC Là Gì?](#cdc-là-gì)
2. [CDC Theo Từng Engine](#cdc-theo-từng-engine)
3. [Online Migration Flow](#online-migration-flow)
4. [Zero-Downtime Techniques](#zero-downtime-techniques)
5. [Validation & Data Consistency](#validation--data-consistency)
6. [Rủi Ro & Cách Giảm Thiểu](#rủi-ro--cách-giảm-thiểu)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 📡 CDC Là Gì?

### Khái Niệm

CDC (Change Data Capture — Thu Thập Thay Đổi Dữ Liệu) là kỹ thuật đọc **transaction log** (nhật ký giao dịch) của database để biết mọi INSERT, UPDATE, DELETE xảy ra theo thứ tự thời gian chính xác.

Transaction log là file mà mọi database engine đều duy trì để:
- Đảm bảo durability (tính bền vững) — D trong ACID
- Crash recovery (khôi phục sau sự cố)
- Replication (sao chép) giữa primary và replica

CDC đọc log này thay vì query trực tiếp vào tables, do đó:
- Không tạo thêm load lớn trên source database
- Thấy đúng thứ tự các thay đổi (đảm bảo consistency)
- Bắt được cả DELETE (query thông thường không thấy row đã xóa)

### Tại Sao CDC Quan Trọng Cho Migration?

```
Không có CDC (chỉ Full Load):
  T=0    T=6h (full load done)     T=6h+
  ──────────────────────────────────────
  [SOURCE writes] ─── DMS loads ──▶ TARGET
                  ↑                 ↑
                  Copy data         Done
                  
  Vấn đề: Trong 6 giờ full load, source có thêm hàng triệu thay đổi
  → Target bị stale (lỗi thời) ngay khi vừa load xong
  → Phải dừng source để đảm bảo consistency → downtime!

Có CDC (Full Load + CDC):
  T=0    T=6h (full load done)     T=6h+     Cutover
  ──────────────────────────────────────────────────
  [SOURCE writes] ─── DMS loads ──▶ TARGET
  [CDC captures changes] ─────────▶ [Apply to TARGET]
                                    ↑
                                    Lag giảm dần → ~0
  
  Kết quả: Target luôn bắt kịp source → cutover bất cứ lúc nào
```

---

## ⚙️ CDC Theo Từng Engine

### MySQL / MariaDB — Binary Log (Binlog)

**Cấu hình cần thiết trên source MySQL:**

```ini
# my.cnf / my.ini — cần restart MySQL sau khi thay đổi
[mysqld]
server-id = 1                    # Unique ID — ID duy nhất cho server
log_bin = /var/lib/mysql/binlog  # Bật binary log
binlog_format = ROW              # DMS yêu cầu ROW format
binlog_row_image = FULL          # Ghi đầy đủ before/after values
expire_logs_days = 7             # Giữ log 7 ngày (đủ để DMS catchup)
```

**Kiểm tra binary log đã bật:**
```sql
SHOW VARIABLES LIKE 'log_bin';          -- ON
SHOW VARIABLES LIKE 'binlog_format';    -- ROW
SHOW MASTER STATUS;                      -- File hiện tại và position
```

**Cách DMS dùng binlog:**
- DMS kết nối như một MySQL replica (bản sao)
- Dùng protocol MySQL replication để nhận binlog events
- Không cần account admin — chỉ cần `REPLICATION SLAVE` privilege (quyền)

### PostgreSQL — Logical Replication Slots

**Cấu hình cần thiết:**

```ini
# postgresql.conf
wal_level = logical              # Bật logical replication
max_replication_slots = 10       # Số replication slots tối đa
max_wal_senders = 10             # Số connections WAL sender
```

**Tạo replication slot cho DMS:**
```sql
-- DMS tự tạo slot, hoặc tạo thủ công:
SELECT pg_create_logical_replication_slot('dms_slot', 'pgoutput');

-- Kiểm tra:
SELECT slot_name, plugin, active, restart_lsn
FROM pg_replication_slots;
```

**Cảnh báo quan trọng với PostgreSQL:**

```
⚠️ WAL accumulation (Tích Lũy WAL Log):
  Replication slot giữ WAL files lại cho đến khi DMS đọc xong.
  Nếu DMS bị dừng lâu, WAL files tích tụ → disk full → PostgreSQL crash!
  
  Giải pháp:
  - Monitor replication slot lag: SELECT * FROM pg_replication_slots;
  - Set max_slot_wal_keep_size (PostgreSQL 13+) để giới hạn WAL giữ lại
  - Drop slot ngay khi không dùng: SELECT pg_drop_replication_slot('dms_slot');
```

### Oracle — LogMiner hoặc Binary Reader

DMS hỗ trợ 2 cách đọc Oracle redo logs:

```
LogMiner (mặc định):
  - Dùng Oracle LogMiner API để parse redo logs
  - Đơn giản hơn về cấu hình
  - Chậm hơn Binary Reader với database lớn

Binary Reader:
  - DMS đọc trực tiếp redo log files, parse binary format
  - Nhanh hơn 10-100x so với LogMiner cho database lớn
  - Cần thêm permissions và cấu hình
  - Recommended cho Oracle > 100 GB
```

**Cấu hình tối thiểu Oracle để dùng DMS CDC:**

```sql
-- Bật supplemental logging (nhật ký bổ sung)
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;

-- Bật supplemental logging cho tất cả columns (cần cho ROW level)
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;

-- Grant permissions cho DMS user
GRANT SELECT ANY TRANSACTION TO dms_user;
GRANT SELECT ON V_$ARCHIVED_LOG TO dms_user;
GRANT SELECT ON V_$LOG TO dms_user;
GRANT SELECT ON V_$LOGFILE TO dms_user;
GRANT LOGMINING TO dms_user;  -- Oracle 12c+
```

### SQL Server — MS-CDC

```sql
-- Bật CDC trên database
EXEC sys.sp_cdc_enable_db;

-- Bật CDC trên từng table
EXEC sys.sp_cdc_enable_table
  @source_schema = N'dbo',
  @source_name   = N'orders',
  @role_name     = NULL;

-- Kiểm tra CDC đã bật:
SELECT name, is_cdc_enabled FROM sys.databases WHERE name = 'myapp';
SELECT name, is_tracked_by_cdc FROM sys.tables WHERE name = 'orders';
```

---

## 🔄 Online Migration Flow

### Timeline Thực Tế

```
Ngày 1: Setup & Validation
├── Tạo target database
├── Chạy SCT (nếu heterogeneous)
├── Cấu hình CDC trên source
├── Tạo DMS replication instance và endpoints
└── Test connections

Ngày 2-5: Full Load
├── Start DMS task: Full Load + CDC
├── Monitor: FullLoadThroughputRowsSource
├── Verify row counts định kỳ
└── CDC đã bắt đầu record changes (song song với full load)

Ngày 6-7: CDC Catchup
├── Full load hoàn thành
├── CDC apply accumulated changes
├── Lag (độ trễ) giảm dần từ nhiều giờ → vài phút → vài giây
└── Monitor: CDCLatencySource, CDCLatencyTarget

Ngày 8: Steady State
├── Lag ổn định < 30 giây
├── Chạy application tests trên target
├── Performance benchmarking
└── Chuẩn bị cutover plan

Ngày 9-10: Cutover Window
└── (Xem chi tiết trong 5-cutover-runbook.md)
```

### Monitoring CDC Lag

**CloudWatch Metrics quan trọng:**

```
CDCLatencySource:
  Độ trễ từ khi change xảy ra trên source đến khi DMS đọc được
  Target: < 10 giây
  Cảnh báo: > 60 giây

CDCLatencyTarget:
  Độ trễ từ khi DMS đọc được đến khi apply vào target
  Target: < 10 giây  
  Cảnh báo: > 60 giây

CDCThroughputRowsSource:
  Số rows/giây đọc từ source
  
CDCThroughputRowsTarget:
  Số rows/giây apply vào target
  Nên gần bằng Source — nếu Target << Source → target là bottleneck
```

---

## 🚀 Zero-Downtime Techniques

### Kỹ Thuật 1: Blue-Green Deployment (Triển Khai Xanh-Lục)

```
Blue (Hiện tại — đang chạy)  Green (Mới — target DB)
────────────────────────────  ─────────────────────────
App → Blue DB                 App → (chưa dùng)
      ↓ DMS CDC                     ↑
      └──────────────────────────────┘ (đồng bộ liên tục)

Cutover:
1. Wait CDC lag ≈ 0
2. Switch App → Green DB (thay đổi connection string)
3. Test smoke tests
4. Done — downtime chỉ vài giây khi restart app
```

### Kỹ Thuật 2: DNS Cutover (Cắt Chuyển DNS)

```
Thay vì hardcode connection string, dùng DNS alias (bí danh DNS):

db.production.internal → Blue DB IP    (trước cutover)
db.production.internal → Green DB IP   (sau cutover)

Quy trình:
1. Đổi DNS record trỏ sang Green DB
2. TTL thấp (30-60 giây) → propagate nhanh
3. Connections cũ (dùng Blue IP) sẽ hết pool dần
4. Connections mới dùng Green IP

Ưu điểm: Không cần deploy lại ứng dụng
Nhược điểm: DNS caching ở client có thể gây vài connections còn dùng Blue
```

### Kỹ Thuật 3: Application-Level Feature Flag

```
Dùng feature flag (cờ tính năng) để chuyển đổi database:

config = {
  "database": {
    "use_new_db": false,    ← Đổi thành true khi ready
    "blue_dsn": "mysql://blue-db...",
    "green_dsn": "postgresql://green-db..."
  }
}

App code:
if config.use_new_db:
    db = connect(config.green_dsn)
else:
    db = connect(config.blue_dsn)

Ưu điểm: Rollback ngay lập tức bằng cách đổi flag
Nhược điểm: Cần code change, phức tạp nếu schema thay đổi
```

### Kỹ Thuật 4: Dual-Write (Ghi Đôi)

```
Trong giai đoạn transition (chuyển tiếp):
  App ghi vào CẢ HAI database
  App đọc từ Blue DB (source of truth)
  DMS vẫn chạy để sync
  
Sau khi validate Green DB:
  App đọc từ Green DB
  Dừng write vào Blue DB
  
Ưu điểm: Không cần CDC — app tự sync
Nhược điểm: Phức tạp về code, có thể có inconsistency nếu một write fail
            Chỉ phù hợp với hệ thống đơn giản
```

---

## ✅ Validation & Data Consistency

### Row Count Validation (Xác Minh Số Hàng)

```sql
-- Chạy trên source:
SELECT table_name, table_rows
FROM information_schema.tables
WHERE table_schema = 'myapp'
ORDER BY table_name;

-- Chạy trên target:
SELECT table_name, table_rows
FROM information_schema.tables  
WHERE table_schema = 'myapp'
ORDER BY table_name;

-- So sánh — sai số nhỏ là bình thường do approximate counting
-- Để chính xác hơn: SELECT COUNT(*) trực tiếp (tốn thời gian)
```

### Data Sampling (Lấy Mẫu Dữ Liệu)

```sql
-- Kiểm tra 1% dữ liệu ngẫu nhiên:
-- Source (MySQL):
SELECT * FROM orders 
WHERE (id % 100) = 0   -- Lấy mỗi 100 rows
ORDER BY id
LIMIT 1000;

-- Target (PostgreSQL):
SELECT * FROM orders
WHERE (id % 100) = 0
ORDER BY id
LIMIT 1000;

-- So sánh kết quả — dùng script tự động (Python/diff tool)
```

### Checksum Validation (Xác Minh Tổng Kiểm Tra)

```sql
-- MySQL source:
SELECT 
  COUNT(*) as row_count,
  SUM(total_amount) as sum_total,
  MAX(updated_at) as latest_update,
  MD5(GROUP_CONCAT(id ORDER BY id)) as id_checksum
FROM orders
WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31';

-- PostgreSQL target: (cú pháp khác nhưng logic tương tự)
SELECT
  COUNT(*) as row_count,
  SUM(total_amount) as sum_total,
  MAX(updated_at) as latest_update,
  MD5(string_agg(id::text, ',' ORDER BY id)) as id_checksum
FROM orders
WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31';
```

### AWS DMS Data Validation Task

DMS có tính năng data validation tích hợp:

```
Bật trong DMS task settings:
  "ValidationSettings": {
    "EnableValidation": true,
    "ValidationMode": "ROW_LEVEL",
    "ThreadCount": 5
  }

DMS tự động:
- So sánh row counts từng table
- So sánh row data theo hash
- Báo cáo các row bị mismatch (không khớp)
- Tự động retry cho mismatched rows
```

---

## ⚠️ Rủi Ro & Cách Giảm Thiểu

### Rủi Ro 1: Transaction Log Bị Xóa Trước Khi DMS Đọc

```
Tình huống:
  MySQL binlog được purge (xóa tự động) sau 1 ngày
  DMS bị dừng 2 ngày vì lý do nào đó
  → DMS không thể tiếp tục CDC từ điểm dừng
  → Phải restart full load lại từ đầu

Phòng ngừa:
  MySQL: SET GLOBAL expire_logs_days = 7;
         Hoặc: binlog_expire_logs_seconds = 604800  (7 ngày)
  PostgreSQL: monitor replication slot lag
  Oracle: tăng LOG_ARCHIVE_DEST retention
  
  Monitor CDCLatencySource — nếu tăng đột biến → DMS đang lag xa
```

### Rủi Ro 2: Large Transactions (Giao Dịch Lớn)

```
Tình huống:
  Ai đó chạy UPDATE orders SET status = 'processed' WHERE year = 2023;
  → 10 triệu rows bị update trong 1 transaction
  → DMS phải apply 10 triệu rows cùng lúc → lag tăng đột biến

Phòng ngừa:
  - Tránh bulk operations trên source trong giai đoạn CDC
  - Nếu cần: batch nhỏ (1000 rows/transaction) với LIMIT + loop
  - Thông báo team về migration window
```

### Rủi Ro 3: Schema Changes Trên Source

```
Tình huống:
  Developer thêm column mới vào table trong khi CDC đang chạy
  → DMS không biết về column mới → skip column đó
  → Target thiếu column → inconsistency

Phòng ngừa:
  - Freeze schema changes (không ALTER TABLE) trong migration window
  - Nếu bắt buộc: dừng DMS, apply schema change trên cả source và target, restart DMS
  - Dùng DMS task setting: handleSourceTableAltered = STOP_TASK
```

### Rủi Ro 4: Network Partitioning (Phân Vùng Mạng)

```
Tình huống:
  Network giữa source và DMS bị gián đoạn vài giờ
  → CDC bị ngắt quãng
  → DMS cần đọc lại từ checkpoint (điểm kiểm tra)

DMS xử lý:
  DMS tự động retry và resume từ last checkpoint khi network phục hồi
  Miễn là transaction log trên source vẫn còn
  → Monitor và đảm bảo log retention đủ dài
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: CDC (Change Data Capture) là gì và hoạt động như thế nào trong DMS?**

> CDC là kỹ thuật đọc transaction log của database để nắm bắt mọi INSERT, UPDATE, DELETE theo thứ tự thời gian chính xác mà không cần query trực tiếp vào tables. Với MySQL, DMS kết nối như một replica và đọc binary log. Với PostgreSQL, DMS dùng logical replication slot để đọc WAL (Write-Ahead Log — Nhật Ký Ghi Trước). Điều này cho phép DMS đồng bộ liên tục từ source sang target trong khi source vẫn phục vụ production traffic.

**Q: Làm thế nào để thực hiện zero-downtime database migration?**

> Dùng DMS với task type Full Load + CDC. Quy trình: (1) chạy full load để copy dữ liệu ban đầu, (2) CDC bắt đầu đồng bộ các thay đổi trong giai đoạn full load, (3) chờ CDC lag về gần 0, (4) trong maintenance window ngắn, dừng writes trên source, chờ lag = 0, validate row counts, sau đó chuyển application connection string sang target. Downtime thực tế chỉ là thời gian restart application — thường dưới 1 phút.

**Q: Nếu PostgreSQL replication slot không được drop khi migration xong, điều gì xảy ra?**

> Replication slot giữ WAL files lại không xóa, chờ consumer đọc. Nếu consumer (DMS) không đọc, WAL tích lũy liên tục. Khi disk đầy, PostgreSQL crash — đây là incident nghiêm trọng. Luôn drop replication slot ngay khi migration hoàn tất: `SELECT pg_drop_replication_slot('slot_name');`. Nên set `max_slot_wal_keep_size` trong PostgreSQL 13+ để giới hạn WAL giữ lại.

**Q: Tại sao không nên có large transactions trên source trong giai đoạn CDC?**

> Large transactions như UPDATE hàng triệu rows cùng lúc làm DMS lag tăng đột biến vì phải apply tất cả changes đó vào target. Nếu target không handle kịp (bottleneck trên target), CDC lag có thể tăng từ vài giây lên nhiều giờ. Nên tránh bulk operations, batch lớn, hoặc báo trước để team không chạy maintenance jobs trong migration window.

---

**Liên Kết:**
- [1-dms-overview.md](./1-dms-overview.md) — AWS DMS chi tiết
- [2-sct.md](./2-sct.md) — Schema Conversion Tool
- [5-cutover-runbook.md](./5-cutover-runbook.md) — Cutover planning chi tiết

**Cập Nhật Lần Cuối:** 2026-05-15
