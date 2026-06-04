# AWS DMS — Database Migration Service: Nền Tảng Và Kiến Trúc

> **AWS DMS** — Database Migration Service (Dịch Vụ Di Chuyển Cơ Sở Dữ Liệu) — là dịch vụ cloud cho phép di chuyển cơ sở dữ liệu lên AWS một cách nhanh chóng và an toàn. Source DB vẫn tiếp tục hoạt động bình thường trong suốt quá trình migration, đảm bảo thời gian gián đoạn (downtime) tối thiểu cho ứng dụng.

## 📚 Mục Lục

1. [DMS Là Gì? Kiến Trúc Tổng Quan](#dms-là-gì)
2. [Replication Instance — Instance Sao Chép](#replication-instance)
3. [Endpoints — Điểm Cuối Kết Nối](#endpoints)
4. [Replication Tasks — Tác Vụ Sao Chép](#replication-tasks)
5. [Cơ Sở Dữ Liệu Được Hỗ Trợ](#cơ-sở-dữ-liệu-được-hỗ-trợ)
6. [Giám Sát Và Xử Lý Lỗi](#giám-sát-và-xử-lý-lỗi)
7. [Phí Dịch Vụ](#phí-dịch-vụ)
8. [Hạn Chế Cần Biết](#hạn-chế-cần-biết)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 DMS Là Gì? Kiến Trúc Tổng Quan

### So Sánh Với Các Phương Pháp Migration Truyền Thống

```
Phương pháp truyền thống (backup/restore):
├── Tắt ứng dụng → downtime bắt đầu
├── Export/dump toàn bộ database (hàng giờ đến hàng ngày)
├── Transfer file dump sang AWS
├── Import vào RDS/Aurora
└── Bật ứng dụng trở lại → downtime kết thúc
   Nhược điểm: Downtime dài, rủi ro cao, không linh hoạt

AWS DMS (online migration):
├── DMS kết nối source và target ĐỒNG THỜI
├── Tải dữ liệu trong khi source vẫn chạy bình thường (full load)
├── Sau full load: bắt kịp các thay đổi real-time (CDC)
└── Cutover chỉ cần vài phút khi đã đồng bộ xong
   Ưu điểm: Downtime tối thiểu, có thể test trước, rollback dễ dàng
```

### Sơ Đồ Kiến Trúc DMS

```
[SOURCE DATABASE — CSDL Nguồn]
    │ On-premises Oracle / MySQL / SQL Server / PostgreSQL...
    │
    │ (1) DMS kết nối qua network (VPN/Direct Connect/Internet)
    ▼
[DMS REPLICATION INSTANCE — Instance Sao Chép DMS]
    │ EC2 instance được quản lý bởi AWS (bạn chọn size)
    │ ├── Đọc dữ liệu từ source (qua Source Endpoint)
    │ ├── Xử lý và transform dữ liệu nếu cần (Table Mapping)
    │ └── Ghi dữ liệu vào target (qua Target Endpoint)
    │
    │ (2) DMS đẩy dữ liệu vào target
    ▼
[TARGET DATABASE — CSDL Đích]
    └── RDS / Aurora / Redshift / S3 / DynamoDB / DocumentDB...
```

### Ba Thành Phần Cốt Lõi

```
DMS = Replication Instance + Source Endpoint + Target Endpoint + Task

1. Replication Instance:
   └── "Máy chủ trung gian" — đọc từ source, ghi vào target

2. Endpoints (điểm cuối kết nối):
   ├── Source Endpoint: thông tin kết nối đến DB nguồn
   └── Target Endpoint: thông tin kết nối đến DB đích

3. Replication Task (tác vụ):
   ├── Ghép source endpoint + target endpoint + replication instance
   ├── Định nghĩa: migrate table nào? filter gì? transform gì?
   └── Loại task: Full Load / CDC / Full Load + CDC
```

---

## 🖥️ Replication Instance — Instance Sao Chép

### Replication Instance Là Gì?

**Replication Instance** là EC2 instance được AWS quản lý hoàn toàn — bạn chỉ cần chọn instance class. Đây là "trái tim" của DMS: tất cả dữ liệu đi qua instance này.

```
Replication Instance thực hiện:
├── Kết nối đến source DB và đọc dữ liệu
├── Áp dụng Table Mapping rules (lọc, đổi tên, transform)
├── Buffer dữ liệu tạm thời (dùng local storage)
└── Ghi vào target DB
```

### Chọn Instance Class Phù Hợp

| Instance Class | vCPU | RAM | Phù Hợp Với |
| ------------- | ---- | --- | ----------- |
| `dms.t3.micro` | 2 | 1 GB | Dev/Test nhỏ, demo |
| `dms.t3.small` | 2 | 2 GB | DB nhỏ < 100 GB, low traffic |
| `dms.t3.medium` | 2 | 4 GB | DB vừa, full load đơn giản |
| `dms.c5.large` | 2 | 4 GB | Production — CPU-intensive transformation |
| `dms.r5.large` | 2 | 16 GB | Production — RAM-intensive, large tables |
| `dms.r5.xlarge` | 4 | 32 GB | Enterprise — nhiều task song song, LOB data |
| `dms.r5.4xlarge` | 16 | 128 GB | Very large DB, phức tạp, nhiều table |

```
Quy tắc chọn instance class:
├── Small DB (< 100 GB): dms.t3.medium
├── Medium DB (100 GB - 1 TB): dms.r5.large
├── Large DB (> 1 TB): dms.r5.xlarge trở lên
├── Nhiều LOB columns (BLOB, CLOB, TEXT lớn): tăng RAM (r5 series)
└── Nhiều transformation rules: tăng CPU (c5 series)
```

### Multi-AZ Cho Replication Instance

```
Single-AZ (mặc định):
├── Phù hợp: dev, test, non-critical migrations
└── Rủi ro: nếu AZ có sự cố, task migration bị gián đoạn

Multi-AZ (khuyến nghị cho production):
├── AWS tự động tạo standby instance ở AZ khác
├── Failover tự động (< 1 phút)
└── Chi phí: ~2x so với Single-AZ
```

> **Thực tế:** Với migration production quan trọng, luôn dùng Multi-AZ để tránh rủi ro AZ failure làm gián đoạn migration kéo dài ngày.

### Replication Subnet Group — Nhóm Subnet Sao Chép

```
Cấu hình mạng cho Replication Instance:
├── Tạo Replication Subnet Group:
│   └── Chọn ít nhất 2 subnet ở 2 AZ khác nhau (trong cùng VPC)
├── DMS sẽ đặt Replication Instance vào một trong các subnet này
└── Instance cần kết nối được đến cả source và target DB:
    ├── Source on-premises: qua VPN/Direct Connect hoặc public IP
    └── Target RDS/Aurora: cùng VPC hoặc VPC peering
```

---

## 🔌 Endpoints — Điểm Cuối Kết Nối

### Endpoint Là Gì?

**Endpoint** lưu trữ thông tin kết nối đến một cơ sở dữ liệu (source hoặc target). Mỗi endpoint chứa:

```
Thông tin trong một Endpoint:
├── Engine type: mysql / oracle / postgres / sqlserver / aurora...
├── Server name (hostname hoặc IP)
├── Port: 3306 (MySQL), 1521 (Oracle), 5432 (PostgreSQL)...
├── Database name (schema name)
├── Username / Password (hoặc Secrets Manager ARN)
└── SSL mode: none / require / verify-ca / verify-full
```

### Test Connection — Kiểm Tra Kết Nối

```
Trước khi chạy task, luôn Test Connection:
├── DMS Replication Instance thử kết nối đến DB theo endpoint config
├── Kết quả: Successful hoặc Failed (với error message)
└── Nguyên nhân thất bại thường gặp:
    ├── Security group chặn port DB
    ├── Username/password sai
    ├── DB chưa mở binary logging (MySQL) hoặc supplemental logging (Oracle)
    └── Network connectivity giữa Replication Instance và DB
```

### Database Engines Hỗ Trợ Làm Source Endpoint

| Engine | Phiên Bản | Full Load | CDC | Ghi Chú |
| ------ | --------- | --------- | --- | -------- |
| **Oracle** | 10g, 11g, 12c, 19c, 21c | ✅ | ✅ | Cần Oracle LogMiner hoặc Binary Reader |
| **MySQL** | 5.5, 5.6, 5.7, 8.0 | ✅ | ✅ | Cần bật binary logging (binlog) |
| **PostgreSQL** | 9.4+ | ✅ | ✅ | Cần logical replication slot |
| **SQL Server** | 2008 R2 - 2022 | ✅ | ✅ | Cần MS-CDC hoặc backup mode |
| **MariaDB** | 10.0+ | ✅ | ✅ | Tương tự MySQL |
| **MongoDB** | 3.6+ | ✅ | ✅ | Change streams |
| **Aurora MySQL** | v2, v3 | ✅ | ✅ | |
| **Aurora PostgreSQL** | v1+ | ✅ | ✅ | |
| **IBM Db2** | 9.7+ | ✅ | ✅ | Cần đặc quyền admin |
| **SAP ASE** | 15.x, 16.x | ✅ | ✅ | |
| **Redis** | 6.x+ | ✅ | ❌ | Chỉ full load |
| **S3** | - | ✅ | ❌ | Nguồn file CSV/Parquet |

### Database Engines Hỗ Trợ Làm Target Endpoint

| Engine | Phiên Bản | Ghi Chú |
| ------ | --------- | -------- |
| **Aurora MySQL** | v2, v3 | Mục tiêu phổ biến nhất |
| **Aurora PostgreSQL** | v1+ | Thay thế Oracle/SQL Server |
| **RDS MySQL** | 5.6, 5.7, 8.0 | |
| **RDS PostgreSQL** | 9.6 - 15 | |
| **RDS SQL Server** | 2014 - 2022 | |
| **RDS Oracle** | 12c, 19c, 21c | |
| **Amazon Redshift** | - | Data warehouse migration |
| **Amazon S3** | - | Data lake, file output |
| **Amazon DynamoDB** | - | Migrate sang NoSQL |
| **Amazon DocumentDB** | - | Migrate từ MongoDB |
| **Amazon Kinesis** | - | Streaming pipeline |
| **Amazon OpenSearch** | - | Search/analytics |
| **Kafka / MSK** | - | Streaming via Kafka connector |

---

## 📋 Replication Tasks — Tác Vụ Sao Chép

### Ba Loại Task

```
1. Full Load (Tải Toàn Bộ):
   ├── Tải toàn bộ dữ liệu từ source sang target một lần
   ├── Source DB vẫn chạy bình thường (không lock table)
   ├── Thích hợp: DB nhỏ, có thể chấp nhận downtime ngắn
   └── Hạn chế: dữ liệu mới trong quá trình full load không được bắt
      → Cần dừng ứng dụng để cutover chính xác

2. Full Load + CDC (Tải Toàn Bộ + Bắt Thay Đổi Liên Tục):
   ├── Phase 1: Full Load — tải toàn bộ dữ liệu hiện có
   ├── Phase 2: CDC tự động bắt đầu sau full load
   │   └── Bắt kịp tất cả INSERT/UPDATE/DELETE xảy ra trong thời gian full load
   ├── Thích hợp: zero-downtime migration (phổ biến nhất)
   └── Cần: source DB bật CDC capability (binlog, LogMiner...)

3. CDC Only (Chỉ Bắt Thay Đổi):
   ├── Không tải dữ liệu lịch sử — chỉ apply changes từ một thời điểm
   ├── Thích hợp: schema đã sẵn sàng trên target, chỉ cần sync delta
   └── Use case: khởi động lại task sau sự cố, hoặc dùng cùng native dump
```

### Table Mapping — Ánh Xạ Bảng

**Table Mapping** — Ánh Xạ Bảng — cho phép bạn kiểm soát chính xác dữ liệu nào được migrate:

```json
// Ví dụ Table Mapping rules (JSON):
{
  "rules": [
    {
      "rule-type": "selection",
      "rule-id": "1",
      "rule-name": "include-orders",
      "object-locator": {
        "schema-name": "retail",
        "table-name": "%"
      },
      "rule-action": "include"
    },
    {
      "rule-type": "selection",
      "rule-id": "2",
      "rule-name": "exclude-temp",
      "object-locator": {
        "schema-name": "retail",
        "table-name": "temp_%"
      },
      "rule-action": "exclude"
    },
    {
      "rule-type": "transformation",
      "rule-id": "3",
      "rule-name": "lowercase-schema",
      "rule-action": "convert-lowercase",
      "rule-target": "schema",
      "object-locator": {
        "schema-name": "%"
      }
    }
  ]
}
```

```
Các loại rule trong Table Mapping:
├── selection: include/exclude table nào
├── transformation: đổi tên schema/table/column, chuyển chữ hoa/thường
├── column-filter: lọc column không muốn migrate
└── row-filter: lọc row theo điều kiện (WHERE clause)
```

### Task Settings — Cài Đặt Task

```
Các cài đặt quan trọng trong Task Settings:

LOB (Large Object) settings:
├── Limited LOB mode: giới hạn kích thước LOB (nhanh nhưng có thể mất data)
├── Full LOB mode: tải toàn bộ LOB (chậm hơn, an toàn)
└── Inline LOB mode: quyết định động theo kích thước thực tế (khuyến nghị)

Error handling:
├── STOP_TASK: dừng task khi có lỗi
├── LOG_ERROR: ghi log lỗi và tiếp tục (không an toàn cho production)
└── SUSPEND_TABLE: tạm dừng table bị lỗi, tiếp tục table khác

Parallel load (tải song song):
├── Tải nhiều table đồng thời
├── Số lượng thread: từ 1 đến 32 (phụ thuộc instance class)
└── Giảm thời gian full load đáng kể cho DB lớn

Transaction consistency:
└── Đảm bảo transaction order được giữ nguyên trong CDC phase
```

---

## 📊 Cơ Sở Dữ Liệu Được Hỗ Trợ

### Bật CDC Trên Source DB

Để DMS thực hiện CDC, source DB phải được cấu hình đặc biệt:

#### MySQL / MariaDB — Bật Binary Logging

```sql
-- Kiểm tra binary logging:
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';

-- Yêu cầu:
-- log_bin = ON
-- binlog_format = ROW (quan trọng — không dùng STATEMENT hoặc MIXED)
-- binlog_row_image = FULL (để capture toàn bộ row data)
```

```ini
# my.cnf (cấu hình MySQL):
[mysqld]
log-bin=mysql-bin
binlog-format=ROW
binlog-row-image=full
expire_logs_days=3
```

```sql
-- Tạo user DMS với quyền cần thiết:
CREATE USER 'dms_user'@'%' IDENTIFIED BY 'password';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'dms_user'@'%';
FLUSH PRIVILEGES;
```

#### Oracle — Bật Supplemental Logging

```sql
-- Bật supplemental logging ở cấp database:
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;

-- Bật cho từng table (hoặc ALL COLUMNS cho tất cả):
ALTER TABLE retail.orders ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;

-- Kiểm tra:
SELECT SUPPLEMENTAL_LOG_DATA_MIN, SUPPLEMENTAL_LOG_DATA_ALL
FROM V$DATABASE;

-- Tạo user DMS:
CREATE USER dms_user IDENTIFIED BY password;
GRANT CREATE SESSION TO dms_user;
GRANT SELECT ANY TABLE TO dms_user;
GRANT SELECT ON V_$DATABASE TO dms_user;
GRANT SELECT ON V_$LOG TO dms_user;
GRANT SELECT ON V_$LOGFILE TO dms_user;
GRANT EXECUTE ON DBMS_LOGMNR TO dms_user;
GRANT LOGMINING TO dms_user;  -- Oracle 12c+
```

#### PostgreSQL — Tạo Logical Replication Slot

```sql
-- postgresql.conf:
wal_level = logical
max_replication_slots = 5
max_wal_senders = 5

-- Tạo replication slot (DMS tự động tạo nếu có quyền):
SELECT pg_create_logical_replication_slot('dms_slot', 'test_decoding');

-- Tạo user DMS:
CREATE USER dms_user WITH REPLICATION PASSWORD 'password';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO dms_user;
ALTER USER dms_user WITH REPLICATION;
```

---

## 🔍 Giám Sát Và Xử Lý Lỗi

### CloudWatch Metrics Quan Trọng

```
DMS metrics cần theo dõi trong CloudWatch:

Full Load:
├── FullLoadThroughputRowsSource — số row/giây đọc từ source
├── FullLoadThroughputRowsTarget — số row/giây ghi vào target
└── FullLoadProgressPercent — % hoàn thành full load

CDC:
├── CDCIncomingChanges — số changes đang chờ xử lý
├── CDCLatencySource — độ trễ đọc từ source (giây)
├── CDCLatencyTarget — độ trễ ghi vào target (giây)
└── CDCChangesMemorySource — memory buffer đang dùng

Lỗi:
├── TableError — số table đang có lỗi
└── (Xem DMS Console → Task → Table Statistics để chi tiết hơn)
```

### Xử Lý Lỗi Thường Gặp

```
Lỗi 1: "Task failed due to insufficient memory"
├── Nguyên nhân: LOB data quá lớn vượt RAM
└── Giải pháp: Nâng cấp lên r5.xlarge, bật Full LOB mode

Lỗi 2: "CDC stopped — binlog position not found"
├── Nguyên nhân: Binary log trên MySQL đã bị rotate và xóa
└── Giải pháp: Tăng expire_logs_days trên MySQL; restart task từ đầu

Lỗi 3: "ORA-01291: missing logfile"
├── Nguyên nhân: Oracle redo log đã bị archive và không còn truy cập được
└── Giải pháp: Dùng Oracle LogMiner ở chế độ online; lưu archived logs đủ lâu

Lỗi 4: "Connection refused / timeout"
├── Nguyên nhân: Security group, network ACL, hoặc firewall chặn kết nối
└── Giải pháp: Kiểm tra security group của Replication Instance và DB

Lỗi 5: "Duplicate key violation on target"
├── Nguyên nhân: Task chạy lại không xóa dữ liệu cũ trên target
└── Giải pháp: Truncate target tables trước khi restart task (nếu Full Load)
```

### DMS Data Validation — Kiểm Tra Tính Toàn Vẹn Dữ Liệu

```
Bật Data Validation trong Task Settings:
├── DMS tự động so sánh row count và data checksum
├── Tạo báo cáo: matched / mismatched / unconfirmed rows
└── Xem trong Console: Task → Table Statistics → Validation State

Trạng thái validation:
├── Validated — Đã kiểm tra và khớp
├── Not validated — Chưa kiểm tra
├── Mismatched records — Có dữ liệu không khớp (cần điều tra)
└── Suspended records — DMS bỏ qua sau nhiều lần thử thất bại
```

---

## 💰 Phí Dịch Vụ

### Cơ Cấu Phí DMS

| Thành Phần | Phí | Ghi Chú |
| ---------- | --- | -------- |
| **Replication Instance** | Theo EC2 On-Demand pricing | dms.t3.medium ~$0.073/giờ |
| **Storage (log)** | $0.115/GB/tháng | Log lưu trên replication instance |
| **Data Transfer** | Miễn phí vào AWS | $0.09/GB ra internet |

### Ví Dụ Chi Phí Thực Tế

```
Kịch bản: Migrate DB 500 GB, chạy DMS trong 10 ngày
(dms.r5.large, Single-AZ)

Replication Instance:
└── $0.24/giờ × 240 giờ = $57.60

Storage (50 GB log):
└── $0.115 × 50 × (10/30 tháng) = $1.92

Data Transfer (500 GB from on-premises → AWS):
└── Miễn phí (inbound vào AWS)

Tổng ước tính: ~$59.52 cho 10 ngày
→ Rất rẻ so với chi phí downtime của production DB
```

> **Mẹo tiết kiệm:** Xóa Replication Instance ngay sau khi migration và validation hoàn thành. Không để instance chạy lãng phí.

---

## ⚠️ Hạn Chế Cần Biết

| Hạn Chế | Chi Tiết |
| ------- | -------- |
| **Không migrate stored procedures/triggers** | DMS chỉ migrate data — schema/code cần dùng SCT hoặc native tools |
| **Không hỗ trợ sequences (Oracle)** | Sequences phải được tạo thủ công hoặc dùng SCT |
| **LOB columns làm chậm migration** | BLOB/CLOB/TEXT lớn giảm tốc độ đáng kể |
| **CDC cần supplemental/binary logging** | Source DB phải được cấu hình trước — không thể bật sau khi đã migrate |
| **Giới hạn số task đồng thời** | Mỗi Replication Instance có giới hạn số task song song (phụ thuộc RAM) |
| **DMS không migrate indexes tự động** | Indexes trên target phải được tạo riêng (tạo sau full load để tăng tốc) |
| **Không hỗ trợ DDL changes trong CDC** | ALTER TABLE trên source trong quá trình CDC có thể gây lỗi |

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: DMS Replication Instance là gì? Nó làm gì?**

> Replication Instance là EC2 instance được AWS DMS quản lý, đóng vai trò trung gian giữa source và target database. Nó kết nối đến source DB để đọc dữ liệu (qua Source Endpoint), xử lý và áp dụng Table Mapping rules, rồi ghi dữ liệu vào target DB (qua Target Endpoint). Bạn chọn instance class dựa trên khối lượng dữ liệu và độ phức tạp của transformation.

**Q: Phân biệt ba loại DMS task: Full Load, Full Load + CDC, CDC Only?**

> Full Load tải toàn bộ dữ liệu một lần — đơn giản nhưng cần downtime để đảm bảo consistency. Full Load + CDC là lựa chọn phổ biến nhất cho zero-downtime: DMS tải toàn bộ dữ liệu trước, sau đó tự động chuyển sang CDC để bắt kịp các thay đổi xảy ra trong quá trình full load. CDC Only chỉ apply changes từ một điểm thời gian — dùng khi schema và dữ liệu lịch sử đã sẵn sàng trên target, chỉ cần đồng bộ delta.

**Q: Tại sao phải bật binary logging trên MySQL trước khi dùng DMS CDC?**

> DMS CDC cần đọc transaction log của database để biết những thay đổi nào đã xảy ra. Trên MySQL, transaction log được gọi là binary log (binlog). Để DMS đọc được đúng dữ liệu, binlog phải ở định dạng ROW (không phải STATEMENT) — định dạng ROW ghi lại giá trị thực của từng row trước và sau thay đổi, trong khi STATEMENT chỉ ghi SQL statement có thể không tái tạo được kết quả chính xác. Ngoài ra, `binlog_row_image=FULL` đảm bảo DMS có đủ thông tin để replicate UPDATE và DELETE.

**Q: Làm thế nào monitor DMS task đang chạy?**

> Theo dõi qua DMS Console (Task → Table Statistics), CloudWatch metrics (CDCLatencySource, CDCLatencyTarget, FullLoadProgressPercent), và CloudWatch Logs cho DMS task logs. Đặc biệt chú ý đến CDCLatencyTarget — nếu lag tăng liên tục là dấu hiệu target DB không theo kịp tốc độ write; giải pháp là nâng cấp target DB instance hoặc giảm concurrent writes. Bật Data Validation để DMS tự động kiểm tra tính toàn vẹn row count và checksum.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
