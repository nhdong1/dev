# DMS CDC — Change Data Capture: Đồng Bộ Dữ Liệu Liên Tục

> **CDC** — Change Data Capture (Bắt Thay Đổi Dữ Liệu) — là kỹ thuật theo dõi và capture mọi thay đổi (INSERT, UPDATE, DELETE) trong database nguồn và áp dụng chúng lên database đích theo thời gian thực. CDC là nền tảng của **zero-downtime database migration** — cho phép source DB tiếp tục phục vụ ứng dụng trong suốt quá trình migration.

## 📚 Mục Lục

1. [CDC Là Gì? Tại Sao Cần?](#cdc-là-gì)
2. [Cơ Chế CDC Theo Từng DB Engine](#cơ-chế-cdc)
3. [CDC Trong DMS: Full Load + CDC Workflow](#cdc-trong-dms)
4. [Giám Sát CDC Lag](#giám-sát-cdc-lag)
5. [Xử Lý Sự Cố CDC](#xử-lý-sự-cố)
6. [CDC-Only Task: Khi Nào Dùng?](#cdc-only-task)
7. [Zero-Downtime Cutover Với CDC](#zero-downtime-cutover)
8. [CDC Limitations — Hạn Chế](#cdc-limitations)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 CDC Là Gì? Tại Sao Cần?

### Vấn Đề Khi Không Có CDC

```
Migration không có CDC (dump và restore):

Thời điểm T1: Bắt đầu dump database (100 GB)
│   → Ứng dụng vẫn đang ghi dữ liệu mới
│
Thời điểm T2: Dump hoàn thành sau 4 giờ
│   → Có 40,000 bản ghi mới được tạo trong 4 giờ
│   → Dump file bị "stale" (không có 40,000 bản ghi mới này)
│
Thời điểm T3: Import dump vào target DB (thêm 2 giờ)
│   → Target thiếu 40,000 + thêm 20,000 bản ghi mới nữa
│
Kết quả: Target DB thiếu ~60,000 records → không thể cutover!
Giải pháp truyền thống: Tắt ứng dụng → dump → import → bật lại
Downtime: 4 giờ dump + 2 giờ import = 6 giờ!
```

### CDC Giải Quyết Như Thế Nào?

```
Migration với CDC (Full Load + CDC):

Thời điểm T1: Bắt đầu Full Load
│   → DMS tải dữ liệu và đồng thời ghi lại starting CDC position
│   → Ứng dụng vẫn chạy bình thường
│
Thời điểm T1 → T2: Full Load đang chạy
│   → CDC tự động bắt đầu track mọi thay đổi từ T1
│   → 40,000 bản ghi mới được ghi vào CDC buffer
│
Thời điểm T2: Full Load hoàn thành
│   → DMS chuyển sang CDC phase
│   → Apply 40,000 thay đổi đã buffer vào target
│
Thời điểm T2+: CDC ongoing
│   → Mọi thay đổi mới tiếp tục được apply real-time
│   → Target "bắt kịp" source trong vài phút
│
Kết quả: Target luôn gần đồng bộ với source
Downtime khi cutover: < 5 phút (chỉ drain remaining changes)
```

---

## ⚙️ Cơ Chế CDC Theo Từng DB Engine

### MySQL / MariaDB: Binary Log (Binlog)

```
Binary Log (Binlog) — Nhật Ký Nhị Phân:
├── MySQL ghi mọi thay đổi data vào binary log file
├── DMS đọc binlog như một slave replication
├── Format: phải là ROW (không phải STATEMENT hay MIXED)
└── Binlog_row_image: phải là FULL

Cách DMS đọc binlog:
├── DMS kết nối MySQL như một "replica" bằng replication credentials
├── Yêu cầu server ID duy nhất (server_id không trùng với servers khác)
└── DMS đọc từ một binlog position cụ thể (lưu trong task state)

Cấu hình MySQL cho CDC:
```

```sql
-- my.cnf:
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog-format = ROW
binlog-row-image = FULL
expire-logs-days = 7   -- giữ binlog 7 ngày (quan trọng!)

-- Restart MySQL sau khi thay đổi my.cnf

-- Kiểm tra:
SHOW MASTER STATUS;
-- Hiển thị: File, Position, Binlog_Do_DB, Binlog_Ignore_DB
-- DMS lưu File + Position này làm starting point cho CDC
```

### Oracle: LogMiner / Binary Reader

```
Oracle có hai cơ chế CDC cho DMS:

Cơ chế 1: Oracle LogMiner (mặc định)
├── LogMiner là công cụ built-in của Oracle để đọc redo logs
├── DMS dùng LogMiner API để parse transaction log
├── Ưu: Không cần cài thêm gì, phù hợp với hầu hết cấu hình
├── Nhược: Có thể chậm với volume lớn
└── Yêu cầu:
    ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
    GRANT LOGMINING TO dms_user;
    GRANT SELECT ON V_$LOGMNR_CONTENTS TO dms_user;

Cơ chế 2: Oracle Binary Reader
├── DMS đọc Oracle redo log files trực tiếp (bypass LogMiner)
├── Ưu: Nhanh hơn LogMiner (2-3x) với volume lớn
├── Nhược: Cần quyền truy cập filesystem vào redo log files
└── Dùng khi: Production Oracle với volume CDC cao
```

### PostgreSQL: Logical Replication Slot

```
PostgreSQL dùng Logical Replication — Logical Replication Slot:
├── PostgreSQL ghi WAL — Write-Ahead Log (Nhật Ký Ghi Trước) cho mọi thay đổi
├── Logical Replication decode WAL thành row-level changes
└── DMS tạo một replication slot và consume thay đổi từ đó

Cấu hình PostgreSQL cho CDC:
```

```sql
-- postgresql.conf:
wal_level = logical          -- phải là 'logical', không phải 'replica'
max_replication_slots = 5    -- đủ cho DMS và monitoring tools
max_wal_senders = 5
wal_sender_timeout = 0       -- không timeout connection

-- Restart PostgreSQL

-- DMS tự tạo replication slot khi khởi động CDC task
-- Slot name mặc định: awsdms_<task-arn>

-- Quan trọng: Replication slot giữ lại WAL files
-- Nếu DMS bị dừng lâu → WAL tích lũy → disk full!
SELECT slot_name, active, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots;
-- Xóa slot nếu không còn cần:
SELECT pg_drop_replication_slot('awsdms_slot_name');
```

### SQL Server: MS-CDC / Transaction Log

```
SQL Server hỗ trợ hai cơ chế:

Cơ chế 1: MS-CDC (SQL Server Change Data Capture)
├── Tính năng built-in của SQL Server 2008+
├── Phải enable ở cấp database và table
└── Setup:
```

```sql
-- Bật MS-CDC ở cấp database:
USE retail_db;
EXEC sys.sp_cdc_enable_db;

-- Bật cho từng table cần track:
EXEC sys.sp_cdc_enable_table
  @source_schema = N'dbo',
  @source_name = N'orders',
  @role_name = NULL,            -- NULL: mọi user đều đọc được
  @supports_net_changes = 1;    -- track net changes (chỉ kết quả cuối)

-- Kiểm tra:
SELECT * FROM sys.change_tracking_tables;
SELECT name, is_cdc_enabled FROM sys.databases WHERE name = 'retail_db';
```

```
Cơ chế 2: Backup mode (sử dụng Transaction Log)
├── DMS đọc từ transaction log backup
├── Phù hợp khi không thể bật MS-CDC
└── Hạn chế: CDC chỉ hoạt động khi có transaction log backup
```

---

## 🔄 CDC Trong DMS: Full Load + CDC Workflow

### Chi Tiết Các Phase

```
PHASE 1: Full Load (Tải Toàn Bộ)
──────────────────────────────────
Thời điểm bắt đầu: DMS ghi lại "CDC start position":
├── MySQL: binlog file name + position
├── Oracle: SCN — System Change Number (Số Thay Đổi Hệ Thống)
├── PostgreSQL: LSN — Log Sequence Number (Số Thứ Tự Log)
└── SQL Server: Log Sequence Number

Quá trình:
├── DMS SELECT * từng table (theo batch)
├── INSERT hàng loạt vào target
└── Không có locking trên source (InnoDB: MVCC — Multi-Version Concurrency Control)

PHASE 2: Transition (Chuyển Tiếp)
───────────────────────────────────
├── Full Load hoàn thành cho tất cả tables
├── DMS đọc lại CDC từ start position (đã lưu ở đầu)
└── Apply tất cả changes xảy ra TRONG THỜI GIAN full load

PHASE 3: Ongoing CDC (Đồng Bộ Liên Tục)
──────────────────────────────────────────
├── DMS đọc changes real-time từ transaction log
├── Apply từng INSERT/UPDATE/DELETE lên target
├── Throughput: thường < 1 giây lag cho low-to-medium load
└── Tiếp tục vô hạn cho đến khi cutover
```

### CDC Event Processing — Xử Lý Sự Kiện CDC

```
DMS xử lý CDC events theo thứ tự:

1. Read: Đọc từ transaction log
   ├── Đọc theo batch (không phải từng event một)
   └── Default batch size: 10,000 changes

2. Decode: Giải mã sự kiện
   ├── Binlog/WAL/redo log → row-level changes
   └── INSERT: before=NULL, after=new row
       UPDATE: before=old row, after=new row
       DELETE: before=old row, after=NULL

3. Transform: Áp dụng Table Mapping rules
   └── Filter, rename nếu có cấu hình

4. Apply: Ghi vào target
   ├── INSERT: INSERT INTO target VALUES (...)
   ├── UPDATE: UPDATE target SET ... WHERE pk = ...
   └── DELETE: DELETE FROM target WHERE pk = ...

5. Commit: Confirm với source rằng đã apply xong
   └── Advance CDC position
```

---

## 📈 Giám Sát CDC Lag

### Các Metrics Quan Trọng

```
CloudWatch DMS Metrics cần theo dõi:

CDCLatencySource (Độ Trễ Nguồn):
├── Định nghĩa: Thời gian từ khi event xảy ra trên source đến khi DMS đọc được
├── Đơn vị: giây
├── Ngưỡng bình thường: < 5 giây
├── Ngưỡng cảnh báo: > 30 giây
└── Nguyên nhân cao: network latency, source DB quá tải, transaction log chậm

CDCLatencyTarget (Độ Trễ Đích):
├── Định nghĩa: Thời gian từ khi DMS đọc event đến khi apply vào target
├── Đơn vị: giây
├── Ngưỡng bình thường: < 10 giây
├── Ngưỡng cảnh báo: > 60 giây
└── Nguyên nhân cao: target DB quá tải, long-running transactions, lock contention

CDCIncomingChanges (Thay Đổi Đang Chờ):
├── Số changes đang buffer trong DMS replication instance
├── Bình thường: < 1,000
└── Nếu tăng liên tục → DMS không theo kịp source throughput
```

### Dashboard Giám Sát CDC

```
Tạo CloudWatch Dashboard với các metrics:

Widget 1: CDC Latency
├── CDCLatencySource (màu xanh)
└── CDCLatencyTarget (màu đỏ)
→ Mục tiêu: cả hai < 10 giây

Widget 2: Throughput
├── CDCIncomingChanges
└── FullLoadThroughputRowsTarget (trong phase full load)

Widget 3: Errors
└── TableError count
→ Phải = 0

Alarm:
├── CDCLatencyTarget > 300 seconds → SNS notification
└── TableError > 0 → SNS notification
```

### Phân Tích Khi CDC Lag Tăng

```
Checklist khi CDCLatencyTarget tăng bất thường:

1. Kiểm tra target DB:
   ├── CPU: RDS CloudWatch → CPUUtilization > 80%?
   ├── IOPS: WriteIOPS gần max provisioned?
   └── FreeStorageSpace: Disk gần đầy?

2. Kiểm tra long-running transactions:
   -- PostgreSQL:
   SELECT pid, now() - pg_stat_activity.query_start AS duration, query
   FROM pg_stat_activity
   WHERE state != 'idle' AND query_start < now() - interval '5 minutes'
   ORDER BY duration DESC;

   -- MySQL:
   SHOW PROCESSLIST;
   SELECT * FROM information_schema.INNODB_TRX ORDER BY trx_started;

3. Kiểm tra DMS Replication Instance:
   ├── FreeMemory đang giảm → nâng cấp instance class
   └── CPU spikes → quá nhiều transforms đang chạy

4. Giải pháp:
   ├── Scale up target DB (tạm thời)
   ├── Nâng cấp Replication Instance
   └── Giảm parallel CDC apply threads
```

---

## 🔧 Xử Lý Sự Cố CDC

### Sự Cố 1: CDC Dừng Do Binlog Bị Xóa (MySQL)

```
Triệu chứng: Task error "Could not find log file name in binary log"

Nguyên nhân:
└── MySQL rotate và xóa binlog files sau expire_logs_days
    DMS chưa kịp đọc → binlog position không còn tồn tại

Giải pháp:
1. Tăng expire_logs_days trên MySQL:
   SET GLOBAL expire_logs_days = 7;

2. Restart DMS task từ đầu (nếu không thể dùng binlog cũ):
   ├── Stop task
   ├── Reload target tables (truncate)
   └── Restart task: Full Load + CDC

Phòng ngừa:
└── Luôn đặt expire_logs_days >= 3 (khuyến nghị 7)
    Thời gian đủ để DMS đọc ngay cả khi bị delay
```

### Sự Cố 2: Oracle CDC Lỗi "ORA-01291"

```
Triệu chứng: "ORA-01291: missing logfile"

Nguyên nhân:
└── Oracle redo log đã được archived và xóa khỏi archive log directory

Giải pháp:
1. Giữ archive logs đủ lâu:
   RMAN> CONFIGURE ARCHIVELOG DELETION POLICY TO APPLIED ON STANDBY;
   RMAN> DELETE ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE - 3';

2. Dùng Oracle Binary Reader thay LogMiner:
   ├── Trong DMS Source Endpoint settings
   └── "Use Binary Reader": Yes

3. Restart DMS task từ checkpoint gần nhất:
   └── DMS tự động resume từ last applied SCN
```

### Sự Cố 3: PostgreSQL Replication Slot Tích Lũy WAL

```
Triệu chứng: Disk space của PostgreSQL tăng nhanh

Nguyên nhân:
└── DMS tạo replication slot → PostgreSQL giữ WAL files
    Nếu DMS dừng lâu → WAL tích lũy không được xóa

Giải pháp tức thì:
-- Kiểm tra slot và lag:
SELECT slot_name, active, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag
FROM pg_replication_slots;

-- Nếu slot inactive và lag lớn → xóa slot:
SELECT pg_drop_replication_slot('slot_name');

-- Sau khi xóa slot, restart DMS task:
└── Task sẽ tạo slot mới và resume từ đầu (hoặc từ checkpoint)

Phòng ngừa:
├── Giám sát pg_replication_slots thường xuyên
└── Đặt max_slot_wal_keep_size = 10GB trong postgresql.conf (PostgreSQL 13+)
    → Tự động xóa slot nếu WAL tích lũy quá ngưỡng
```

---

## 📌 CDC-Only Task: Khi Nào Dùng?

### Use Cases Cho CDC-Only

```
CDC-Only task: Chỉ sync changes, không tải lại full data

Khi nào dùng:

1. Sau khi dùng native tools để load data ban đầu:
   ├── Dùng mysqldump hoặc pg_dump để load bulk data nhanh hơn DMS
   ├── Load xong → Dùng DMS CDC-Only để sync changes tiếp theo
   └── Lợi ích: Tốc độ load nhanh hơn + CDC của DMS

2. Restart task sau sự cố:
   ├── Full Load đã hoàn thành trước đó
   ├── Task bị fail trong CDC phase
   └── Restart CDC-Only từ last known good position

3. Replicate changes cho use cases khác:
   ├── Event sourcing: đẩy DB changes vào Kafka/Kinesis
   ├── Analytics: sync OLTP database sang Redshift real-time
   └── Cache invalidation: invalidate Redis cache khi data thay đổi

4. Blue-Green database switch:
   ├── Giữ target DB luôn sync với source bằng CDC
   └── Switch traffic bất cứ lúc nào cần
```

### Thiết Lập CDC-Only Task

```
Khi tạo task, chọn:
├── Migration type: CDC only
└── CDC start position (optional):
    ├── MySQL: binlog file + position
    │   Ví dụ: mysql-bin.000123;154
    ├── Oracle: SCN value
    │   Ví dụ: 1234567
    └── PostgreSQL: LSN value
        Ví dụ: 0/1234567

Nếu không chỉ định start position:
└── DMS bắt đầu từ "now" — chỉ capture changes từ thời điểm task start
```

---

## ⚡ Zero-Downtime Cutover Với CDC

### Quy Trình Cutover Tối Ưu

```
Giai đoạn chuẩn bị (T-1 tuần đến T-1 ngày):
├── DMS Full Load + CDC đang chạy
├── CDCLatencyTarget ổn định < 10 giây
├── Data Validation: Validated ✅
└── Team đã thử cutover trên staging environment

Ngày cutover (maintenance window, ví dụ 2 AM Chủ Nhật):

T-30 phút: Pre-cutover checklist
├── ✅ CDCLatencyTarget < 10 giây
├── ✅ Không có table errors
├── ✅ Target DB health checks: CPU < 50%, disk < 70%
└── ✅ Rollback plan đã được confirm với team

T-5 phút: Begin cutover
1. Enable maintenance mode trên load balancer
   → Trả 503 Service Unavailable cho mọi request
2. Kiểm tra in-flight transactions đã hoàn thành
   → SHOW PROCESSLIST; (MySQL) — đợi đến khi hết active queries
3. Kiểm tra CDCLatencyTarget lần cuối

T-2 phút: Final sync
4. Đợi CDCLatencyTarget → 0 (hoặc < 1 giây)
5. Chạy final validation:
   SELECT COUNT(*) FROM orders;  -- so sánh source vs target
6. Confirm với team: "Source = Target, ready to cutover"

T-0: Cutover
7. Cập nhật connection string ứng dụng → target DB
8. Restart application servers
9. Disable maintenance mode trên load balancer

T+5 phút: Smoke test
10. Test các endpoint quan trọng: login, create order, read data
11. Check application logs: không có DB connection errors
12. Monitor target DB metrics

T+30 phút: Cutover complete
13. Stop DMS task (không cần CDC nữa)
14. Delete Replication Instance (tiết kiệm chi phí)
15. Lên kế hoạch decommission source DB (sau 30 ngày giữ lại để rollback)
```

### Rollback Plan

```
Nếu phát hiện vấn đề sau cutover:

Trong vòng 30 phút đầu (rollback đơn giản):
├── Source DB vẫn còn nguyên vẹn (chưa tắt, chưa xóa)
├── Cập nhật connection string → source DB
├── Restart application servers
└── Thông báo: rollback thành công, điều tra vấn đề

Sau 30 phút (rollback phức tạp hơn):
├── Data mới đã được ghi vào target DB (Aurora) nhưng không có trên source
└── Cần quyết định: accept data loss (không khuyến nghị) hoặc sync lại
    → Giải pháp: DMS CDC-Only từ target (Aurora) → source (on-premises)
      (reverse CDC — nếu DMS hỗ trợ source type này)

Phòng ngừa tốt nhất:
└── Thực hiện test cutover nhiều lần trên staging trước
    Test rollback procedure trên staging
    Cutover trong off-peak hours với team đủ người
```

---

## ⚠️ CDC Limitations — Hạn Chế

| Hạn Chế | Chi Tiết | Giải Pháp |
| ------- | -------- | --------- |
| **DDL changes không được replicate** | ALTER TABLE trên source trong CDC gây lỗi | Không thay đổi schema trong quá trình CDC |
| **Truncate không được capture** | TRUNCATE TABLE không phải DML → DMS bỏ qua | Tránh TRUNCATE trong quá trình migration |
| **Large transactions làm tăng lag** | Transaction 1 triệu rows → buffer lớn | Chia nhỏ batch writes trên source |
| **Replication slot giữ WAL** | PostgreSQL WAL tích lũy nếu DMS dừng | Monitor slot thường xuyên, xóa nếu không dùng |
| **Sequences cần sync riêng** | Sequence values không được replicate qua CDC | Sync sequence value sau cutover |
| **LOB columns trong CDC** | BLOB/CLOB update trong CDC cần cấu hình riêng | Bật InlineLobMaxSize trong task settings |
| **Multi-table transaction ordering** | Transaction span nhiều table có thể apply sai thứ tự | DMS cố gắng maintain, nhưng cần test kỹ |

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: CDC là gì và tại sao nó quan trọng trong database migration?**

> CDC — Change Data Capture (Bắt Thay Đổi Dữ Liệu) — là kỹ thuật theo dõi và capture mọi thay đổi (INSERT, UPDATE, DELETE) trong database và áp dụng chúng lên database khác theo thời gian thực. Trong migration, CDC quan trọng vì nó cho phép zero-downtime: source DB vẫn phục vụ ứng dụng trong khi DMS đang tải dữ liệu, và sau khi full load xong, CDC tự động bắt kịp mọi thay đổi xảy ra trong thời gian đó. Khi target đã đồng bộ, cutover chỉ cần 2-5 phút thay vì hàng giờ.

**Q: Cơ chế CDC khác nhau như thế nào giữa MySQL, Oracle và PostgreSQL?**

> MySQL dùng Binary Log (binlog) với format ROW — DMS kết nối như một slave replica và đọc binlog từ một position cụ thể. Oracle dùng Redo Log thông qua LogMiner API hoặc Binary Reader trực tiếp — cần bật Supplemental Logging để log đủ thông tin row-level changes. PostgreSQL dùng Logical Replication Slot — một slot lưu vị trí trong WAL (Write-Ahead Log) và decode thành row-level events. Điểm chung: tất cả đều dựa trên transaction log của database, và DMS cần credentials đủ quyền để đọc log đó.

**Q: CDCLatencyTarget liên tục tăng — nguyên nhân và cách xử lý?**

> CDCLatencyTarget tăng có nghĩa là DMS đang không theo kịp tốc độ ghi của source DB. Nguyên nhân phổ biến: (1) Target DB quá tải — CPU cao, IOPS đầy, lock contention; (2) DMS Replication Instance hết RAM — buffer overflow buộc ghi chậm lại; (3) Replication Instance CPU cao do nhiều transformation rules. Cách xử lý: kiểm tra target DB metrics trên CloudWatch, xem slow queries, scale up target DB tạm thời, nâng cấp Replication Instance class, giảm parallel apply threads.

**Q: Tại sao phải cẩn thận với PostgreSQL replication slot khi dùng DMS CDC?**

> Khi DMS tạo một logical replication slot trên PostgreSQL, database sẽ giữ lại tất cả WAL files từ vị trí của slot — không bao giờ xóa chúng cho đến khi slot được advance (consumed) hoặc drop. Nếu DMS task bị dừng lâu (sự cố, maintennce), WAL tích lũy không giới hạn và có thể làm đầy disk của PostgreSQL — gây downtime cho production database. Phòng ngừa: giám sát `pg_replication_slots` thường xuyên, drop slot nếu không còn dùng, và cấu hình `max_slot_wal_keep_size` trong PostgreSQL 13+.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
