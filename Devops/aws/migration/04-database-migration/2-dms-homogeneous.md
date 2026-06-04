# DMS Homogeneous Migration — Di Chuyển Cùng Loại Cơ Sở Dữ Liệu

> **Homogeneous Migration** — Di Chuyển Đồng Nhất — là kịch bản di chuyển database từ một engine sang chính engine đó trên AWS. Ví dụ: MySQL on-premises → Amazon RDS MySQL, hoặc PostgreSQL on-premises → Amazon Aurora PostgreSQL. Đây là loại migration đơn giản nhất vì schema tương thích hoàn toàn — không cần AWS SCT.

## 📚 Mục Lục

1. [Homogeneous Migration Là Gì?](#homogeneous-migration-là-gì)
2. [Use Cases Phổ Biến](#use-cases-phổ-biến)
3. [Quy Trình: MySQL → Amazon RDS MySQL](#quy-trình-mysql-rds)
4. [Quy Trình: PostgreSQL → Amazon Aurora PostgreSQL](#quy-trình-postgresql-aurora)
5. [Tối Ưu Tốc Độ Full Load](#tối-ưu-tốc-độ-full-load)
6. [Data Validation Sau Migration](#data-validation-sau-migration)
7. [Chiến Lược Cutover](#chiến-lược-cutover)
8. [Công Cụ Thay Thế DMS Cho Homogeneous](#công-cụ-thay-thế)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Homogeneous Migration Là Gì?

```
So sánh hai loại migration:

Homogeneous (cùng engine):
├── MySQL 5.7 on-premises → Amazon RDS MySQL 8.0
├── PostgreSQL 12 on-premises → Amazon Aurora PostgreSQL 14
├── SQL Server 2016 on-premises → Amazon RDS SQL Server 2019
└── Oracle 12c on-premises → Amazon RDS Oracle 19c
   Đặc điểm:
   ├── Schema tương thích (không cần chuyển đổi)
   ├── Data types giống nhau
   ├── DMS có thể tải trực tiếp không cần SCT
   └── Ít rủi ro hơn, nhanh hơn heterogeneous

Heterogeneous (khác engine) — xem file 3-dms-heterogeneous.md:
├── Oracle → Amazon Aurora PostgreSQL
└── SQL Server → Amazon Aurora MySQL
   Đặc điểm:
   ├── Schema KHÔNG tương thích
   ├── Phải dùng SCT để chuyển đổi schema trước
   └── Phức tạp hơn nhiều
```

### Tại Sao Vẫn Dùng DMS Cho Homogeneous?

Câu hỏi hợp lý: nếu cùng engine thì chỉ cần mysqldump/pg_dump rồi restore thôi, cần gì DMS?

```
Ưu điểm của DMS so với native dump/restore:

1. Zero-downtime (thời gian gián đoạn bằng không):
   ├── DMS có CDC — sync liên tục trong khi ứng dụng vẫn chạy
   └── Downtime chỉ vài phút khi cutover (drain remaining changes)

2. Lọc và transform dữ liệu:
   ├── Migrate chỉ một số table (không phải toàn bộ DB)
   ├── Đổi tên schema, table, column khi sang AWS
   └── Loại bỏ dữ liệu không cần thiết

3. Monitoring tập trung:
   └── CloudWatch metrics, task logs, validation report

4. Migrate data lớn qua mạng an toàn:
   ├── Có thể dùng qua Direct Connect/VPN với mã hóa TLS
   └── Không cần tạo file dump lớn và vận chuyển thủ công
```

---

## 📋 Use Cases Phổ Biến

### 1. Nâng Phiên Bản (Version Upgrade)

```
MySQL 5.7 on-premises → Amazon RDS MySQL 8.0
├── AWS RDS không hỗ trợ in-place upgrade từ 5.7 → 8.0 trong một bước
├── DMS: tải data từ 5.7 lên RDS 8.0 với CDC
└── Lợi ích: test ứng dụng với phiên bản mới trước khi cutover
```

### 2. Chuyển Sang Managed Service (RDS/Aurora)

```
Self-managed MySQL on EC2 → Amazon Aurora MySQL
├── Không còn phải quản lý OS, patches, backups
├── Aurora có khả năng mở rộng đọc tốt hơn (read replicas)
└── Multi-AZ tự động, failover < 30 giây
```

### 3. Migration Theo Vùng (Cross-Region)

```
RDS MySQL us-east-1 → RDS MySQL ap-southeast-1
├── DMS có thể migrate cross-region
└── Use case: mở rộng hoạt động sang khu vực mới
```

### 4. Chia Tách Database (Database Splitting)

```
Monolith DB (MySQL) → nhiều RDS instances nhỏ hơn
├── Table mapping rules: phân chia table theo domain
├── Orders tables → RDS MySQL #1
└── Products tables → RDS MySQL #2
```

---

## 🔄 Quy Trình: MySQL → Amazon RDS MySQL

### Bước 1: Chuẩn Bị Source MySQL

```sql
-- 1.1 Bật binary logging trong my.cnf:
-- [mysqld]
-- log-bin=mysql-bin
-- binlog-format=ROW
-- binlog-row-image=full
-- expire_logs_days=3

-- 1.2 Kiểm tra binary logging đã bật:
SHOW VARIABLES LIKE 'log_bin';        -- phải là ON
SHOW VARIABLES LIKE 'binlog_format';  -- phải là ROW

-- 1.3 Tạo user DMS với quyền cần thiết:
CREATE USER 'dms_user'@'%' IDENTIFIED BY 'StrongP@ssw0rd';
GRANT SELECT ON *.* TO 'dms_user'@'%';
GRANT REPLICATION SLAVE ON *.* TO 'dms_user'@'%';
GRANT REPLICATION CLIENT ON *.* TO 'dms_user'@'%';
FLUSH PRIVILEGES;

-- 1.4 Xác nhận user có đủ quyền:
SHOW GRANTS FOR 'dms_user'@'%';
```

### Bước 2: Tạo Target RDS MySQL

```
Tạo RDS MySQL instance trên AWS:
├── Engine: MySQL 8.0
├── Instance class: phù hợp với workload (ít nhất bằng source)
├── Storage: ít nhất bằng source DB (cộng thêm 20-30% buffer)
├── VPC: cùng VPC với Replication Instance (hoặc có route đến)
├── Security group: mở port 3306 từ Replication Instance SG
├── Tắt Multi-AZ trong quá trình full load (bật lại sau khi migrate xong)
│   └── Lý do: Multi-AZ ghi 2 bản đồng thời → tốc độ write chậm hơn
└── Parameter group:
    └── max_allowed_packet = 1073741824 (1 GB) — cho LOB data

Sau khi tạo RDS:
├── Chạy schema creation script (bằng mysqldump --no-data hoặc dùng SCT)
│   mysqldump -h source-host -u root -p --no-data retail > schema.sql
│   mysql -h rds-endpoint -u admin -p retail < schema.sql
└── Tắt foreign key checks và triggers trên target trong quá trình full load
    SET GLOBAL FOREIGN_KEY_CHECKS = 0;
```

### Bước 3: Tạo DMS Components

```
3.1 Tạo Replication Instance:
├── Instance class: dms.r5.large (cho DB > 100 GB)
├── Multi-AZ: Yes (production)
├── VPC: cùng VPC với target RDS
└── Subnet group: chọn subnet có route đến cả source và target

3.2 Tạo Source Endpoint:
├── Engine: MySQL
├── Server name: IP/hostname của MySQL on-premises
├── Port: 3306
├── Username: dms_user
├── Password: StrongP@ssw0rd
├── SSL: require (nếu có)
└── → Test Connection: phải Successful

3.3 Tạo Target Endpoint:
├── Engine: Amazon RDS MySQL
├── Instance: chọn RDS instance đã tạo
├── Username: admin (master user)
└── → Test Connection: phải Successful

3.4 Tạo Replication Task:
├── Task type: Full load and CDC
├── Replication Instance: instance vừa tạo
├── Source endpoint: endpoint source MySQL
├── Target endpoint: endpoint RDS MySQL
├── Migration type: Full load and CDC
└── Table mappings: chọn schema cần migrate
    {
      "rules": [{
        "rule-type": "selection",
        "rule-id": "1",
        "rule-name": "all-tables",
        "object-locator": {"schema-name": "retail", "table-name": "%"},
        "rule-action": "include"
      }]
    }
```

### Bước 4: Chạy Task Và Giám Sát

```
4.1 Start Task → DMS bắt đầu Full Load:
├── Theo dõi: DMS Console → Task → Table Statistics
├── Xem: rows loaded per table, throughput (rows/sec)
└── Thời gian ước tính: ~1 GB/phút qua đường 100 Mbps

4.2 Sau Full Load → DMS tự động chuyển sang CDC:
├── CDCLatencySource: độ trễ đọc từ source (< 5 giây là tốt)
├── CDCLatencyTarget: độ trễ ghi vào target (< 5 giây là tốt)
└── Nếu lag tăng liên tục → xem xét nâng cấp target DB

4.3 Theo dõi qua CloudWatch:
├── CDCIncomingChanges: số changes đang chờ xử lý
└── CDCChangesMemorySource: buffer hiện tại
```

### Bước 5: Validation Và Cutover

```
5.1 Bật Data Validation trong Task Settings
   → DMS tự so sánh row count và checksum

5.2 Chạy manual validation:
```

```sql
-- So sánh row count trên từng table:
-- Source (MySQL on-premises):
SELECT table_name, table_rows
FROM information_schema.tables
WHERE table_schema = 'retail'
ORDER BY table_name;

-- Target (RDS MySQL):
-- Chạy query tương tự trên RDS
-- row count phải khớp (±1% chấp nhận được do auto-increment)

-- Kiểm tra sample data:
SELECT COUNT(*) FROM retail.orders WHERE created_at > '2026-01-01';
-- So sánh kết quả giữa source và target
```

```
5.3 Cutover:
├── Chọn maintenance window (ít traffic nhất)
├── Tắt ứng dụng (application)
├── Kiểm tra CDCLatencyTarget = 0 (sync hoàn thành)
├── Chạy final validation query
├── Cập nhật connection string ứng dụng → RDS endpoint
├── Bật Multi-AZ trên RDS (nếu đã tắt)
├── Khởi động ứng dụng
└── Smoke test: chạy các endpoint quan trọng, kiểm tra logs
```

---

## ⚡ Tối Ưu Tốc Độ Full Load

### Cấu Hình Để Tăng Tốc

```
1. Parallel Load (Tải Song Song) — quan trọng nhất:
   ├── Trong Task Settings → Advanced settings:
   │   MaxFullLoadSubTasks: 8 (mặc định 8, tối đa 49)
   │   └── DMS tải tối đa 8 table đồng thời
   └── Với DB có 100 table, parallel load giảm thời gian từ 100x xuống ~13x

2. Disable indexes trên target:
   ├── Drop tất cả non-primary indexes trước khi full load
   └── Recreate indexes SAU KHI full load hoàn thành
   Lý do: Insert vào table không có index nhanh hơn 5-10x

3. Tắt foreign key constraints trên target:
   SET GLOBAL FOREIGN_KEY_CHECKS = 0;  -- MySQL
   -- Bật lại sau full load

4. Target RDS instance class lớn hơn tạm thời:
   ├── Scale up trong quá trình full load
   └── Scale down sau khi migration xong (tiết kiệm chi phí)

5. Dùng Direct Connect thay vì Internet:
   └── Băng thông ổn định, không bị throttle
```

### Ước Tính Thời Gian Full Load

```
Công thức ước tính:
Thời gian ≈ Data size / (Throughput × số parallel tables)

Ví dụ thực tế:
├── DB 500 GB, 50 tables, kết nối 1 Gbps, dms.r5.large
├── DMS throughput thực tế: ~100-200 MB/s (sau overhead)
├── Parallel load 8 tables
└── Ước tính: 500 GB / (150 MB/s × 8) ≈ ~7 phút per table batch
           Tổng: ~(50/8) × 7 phút ≈ 44 phút

Lưu ý: Con số thực tế phụ thuộc nhiều vào:
├── Băng thông thực tế giữa source và AWS
├── IOPs của target RDS
├── Kích thước và phân phối của rows
└── LOB columns làm chậm đáng kể
```

---

## ✅ Data Validation Sau Migration

### Kiểm Tra Tự Động Bằng DMS

```
Bật trong Task Settings:
├── Enable validation: Yes
├── Fail on data validation error: Yes (production)
└── Validation mode: Full

Xem kết quả:
├── DMS Console → Task → Table Statistics → Validation State
├── Trạng thái mong muốn: "Validated" cho mọi table
└── Nếu "Mismatched": xem chi tiết để tìm rows không khớp
```

### Kiểm Tra Thủ Công

```sql
-- Script kiểm tra nhanh (chạy trên cả source và target):
SELECT
    table_name,
    table_rows AS estimated_rows
FROM information_schema.tables
WHERE table_schema = 'retail'
ORDER BY table_name;

-- Kiểm tra chính xác hơn với COUNT(*):
SELECT 'orders' AS tbl, COUNT(*) AS cnt FROM retail.orders
UNION ALL SELECT 'products', COUNT(*) FROM retail.products
UNION ALL SELECT 'customers', COUNT(*) FROM retail.customers;

-- Kiểm tra dữ liệu mới nhất (sau CDC):
SELECT MAX(updated_at) AS latest_update FROM retail.orders;
-- Phải giống nhau trên source và target
```

---

## 🔀 Chiến Lược Cutover

### Blue-Green Cutover (Khuyến Nghị)

```
Luồng Blue-Green với DMS:

[Source MySQL on-premises] ←── đang chạy production (BLUE)
       │
       │ DMS CDC đang sync liên tục
       ▼
[Target RDS MySQL] ←── đang sync, sẵn sàng (GREEN)

Thời điểm cutover:
1. Kiểm tra CDCLatencyTarget ≈ 0 (lag gần bằng không)
2. Bật maintenance mode trên load balancer (503 cho users)
3. Đợi ứng dụng xử lý xong tất cả in-flight requests (~30 giây)
4. Kiểm tra lag = 0 lần nữa
5. Cập nhật connection string → RDS endpoint
6. Restart application servers
7. Smoke test (2-3 phút kiểm tra)
8. Tắt maintenance mode → traffic trở lại bình thường

Downtime thực tế: 2-5 phút

Rollback plan:
└── Cập nhật connection string lại → source MySQL
    (source vẫn chạy, không có gì bị xóa)
```

---

## 🔧 Công Cụ Thay Thế DMS Cho Homogeneous Migration

Trong một số trường hợp, công cụ native của DB engine phù hợp hơn DMS:

### mysqldump + binlog (MySQL)

```bash
# Phù hợp khi: DB nhỏ < 50 GB, có thể chấp nhận vài giờ downtime

# Bước 1: Dump schema và data:
mysqldump -h source-host -u root -p \
  --single-transaction \      # không lock table (InnoDB)
  --master-data=2 \           # ghi binlog position vào dump file
  --databases retail \
  > retail_full_backup.sql

# Bước 2: Import vào RDS:
mysql -h rds-endpoint -u admin -p < retail_full_backup.sql

# Bước 3: Tiếp tục CDC bằng binlog replication:
# Lấy binlog position từ dump file (comment -- CHANGE MASTER TO)
# Cấu hình RDS làm replica của source MySQL
# Sau khi sync xong → cutover
```

### pg_dump + pg_restore (PostgreSQL)

```bash
# Phù hợp khi: PostgreSQL → Aurora PostgreSQL, DB vừa

# Dump song song (-j: số core):
pg_dump -h source-host -U postgres -d retail \
  -Fd -j 8 \        # custom format, 8 parallel threads
  -f retail_dump/

# Restore vào Aurora:
pg_restore -h aurora-endpoint -U postgres -d retail \
  -Fd -j 8 \        # parallel restore
  retail_dump/

# CDC bằng logical replication:
# 1. Bật wal_level=logical trên source
# 2. Cấu hình pglogical hoặc AWS DMS CDC only
```

### Khi Nào Dùng DMS vs Native Tools?

| Tiêu Chí | Dùng DMS | Dùng Native Tools |
| -------- | -------- | ----------------- |
| **DB size** | > 100 GB hoặc bất kỳ | < 50 GB |
| **Downtime budget** | Zero-downtime (vài phút) | Vài giờ được chấp nhận |
| **Monitoring** | Cần dashboard tập trung | Tự monitor đơn giản |
| **Filtering** | Cần lọc table/row | Di chuyển toàn bộ |
| **Phức tạp** | Cần DMS CDC | mysqldump đủ dùng |

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: Homogeneous migration với DMS khác gì so với chỉ dùng mysqldump/pg_dump?**

> DMS cung cấp CDC — Change Data Capture (Bắt Thay Đổi Dữ Liệu) liên tục, cho phép zero-downtime migration: ứng dụng vẫn chạy bình thường trong suốt quá trình full load, và DMS tự động bắt kịp mọi thay đổi xảy ra sau đó. Cutover chỉ cần 2-5 phút. Ngược lại, mysqldump yêu cầu tắt ứng dụng hoặc chấp nhận data drift, và thời gian downtime tỷ lệ thuận với kích thước DB. DMS cũng có monitoring tích hợp (CloudWatch) và data validation tự động. Tuy nhiên, đối với DB nhỏ < 50 GB và có maintenance window, mysqldump đơn giản và nhanh hơn.

**Q: Tại sao nên tắt indexes trên target trước khi DMS full load?**

> Khi DMS insert hàng triệu row, mỗi row được insert database cũng phải cập nhật tất cả các B-tree index liên quan. Với table 10 triệu row và 5 index, thực tế có 50 triệu index operations trong quá trình load — làm chậm throughput 5-10 lần. Giải pháp tối ưu là drop tất cả non-primary index trước full load, sau đó recreate sau khi full load hoàn thành với `CREATE INDEX CONCURRENTLY` (PostgreSQL) hoặc `ALTER TABLE ... ADD INDEX` (MySQL). Điều này giảm thời gian full load đáng kể.

**Q: Blue-Green cutover với DMS là gì? Rollback như thế nào?**

> Blue-Green là chiến lược giữ source DB (Blue) vẫn hoạt động trong khi target DB (Green) được đồng bộ qua DMS. Khi CDCLatencyTarget tiến về 0, ta đặt ứng dụng vào maintenance mode, chờ drain in-flight requests, kiểm tra lag = 0, rồi cập nhật connection string sang Green (RDS) và restart ứng dụng. Toàn bộ downtime là 2-5 phút. Rollback rất đơn giản: chỉ cần cập nhật connection string lại về source MySQL — vì source vẫn đang chạy và không có gì bị xóa trên source.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
