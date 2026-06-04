# 5 — Parameter Groups & Option Groups — Nhóm Tham Số & Nhóm Tùy Chọn

> Parameter Groups (Nhóm Tham Số) và Option Groups (Nhóm Tùy Chọn) là cách RDS cho phép bạn tùy chỉnh cấu hình database engine mà không cần truy cập trực tiếp vào OS. Hiểu và dùng đúng đây là chìa khóa để tối ưu hiệu năng database.

## 📚 Mục Lục

1. [Parameter Groups — Nhóm Tham Số](#parameter-groups--nhóm-tham-số)
2. [Các Tham Số Quan Trọng Theo Engine](#các-tham-số-quan-trọng-theo-engine)
3. [Option Groups — Nhóm Tùy Chọn](#option-groups--nhóm-tùy-chọn)
4. [Quy Trình Thay Đổi Cấu Hình](#quy-trình-thay-đổi-cấu-hình)
5. [Best Practices — Thực Hành Tốt Nhất](#best-practices--thực-hành-tốt-nhất)
6. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Parameter Groups — Nhóm Tham Số

### Parameter Group Là Gì?

Parameter Group là một tập hợp các tham số cấu hình cho database engine. Thay vì edit file config trực tiếp (như `my.cnf` hay `postgresql.conf`), bạn quản lý qua AWS Console hoặc CLI.

### Default Parameter Group vs Custom Parameter Group

| Loại                                | Mô Tả                                       | Có Thể Thay Đổi? |
| ----------------------------------- | ------------------------------------------- | ----------------- |
| **Default Parameter Group**         | AWS tạo sẵn, giá trị mặc định của engine   | ❌ Không          |
| **Custom Parameter Group**          | Bạn tạo, kế thừa từ default và customize   | ✅ Có             |

> **Thực hành bắt buộc:** Luôn tạo custom parameter group cho production — không bao giờ modify default.

### Static vs Dynamic Parameters

| Loại Parameter                    | Áp Dụng Khi               | Ví Dụ                          |
| ---------------------------------- | -------------------------- | ------------------------------ |
| **Static** (Tham Số Tĩnh)         | Chỉ sau khi reboot instance | `max_connections` (MySQL)      |
| **Dynamic** (Tham Số Động)        | Ngay lập tức, không reboot  | `slow_query_log`, `long_query_time` |

### Tạo và Gắn Parameter Group

```bash
# Tạo custom parameter group cho MySQL 8.0
aws rds create-db-parameter-group \
    --db-parameter-group-name myapp-mysql80 \
    --db-parameter-group-family mysql8.0 \
    --description "Production MySQL 8.0 parameters for myapp"

# Thay đổi tham số
aws rds modify-db-parameter-group \
    --db-parameter-group-name myapp-mysql80 \
    --parameters \
        "ParameterName=innodb_buffer_pool_size,ParameterValue={DBInstanceClassMemory*3/4},ApplyMethod=pending-reboot" \
        "ParameterName=slow_query_log,ParameterValue=1,ApplyMethod=immediate" \
        "ParameterName=long_query_time,ParameterValue=1,ApplyMethod=immediate"

# Gắn vào DB instance
aws rds modify-db-instance \
    --db-instance-identifier mydb \
    --db-parameter-group-name myapp-mysql80 \
    --apply-immediately
```

---

## Các Tham Số Quan Trọng Theo Engine

### MySQL / MariaDB — Tham Số Quan Trọng

#### Bộ Nhớ (Memory Parameters)

| Tham Số                        | Giá Trị Khuyến Nghị      | Mô Tả                                                    |
| ------------------------------ | ------------------------- | --------------------------------------------------------- |
| `innodb_buffer_pool_size`      | `{DBInstanceClassMemory*3/4}` | Cache InnoDB — quan trọng nhất, 75% RAM               |
| `innodb_buffer_pool_instances` | CPU count (tối đa 64)    | Chia pool thành nhiều phần — giảm contention (Tranh Chấp) |
| `key_buffer_size`              | 8M (MyISAM legacy)        | Cache cho MyISAM indexes — ít dùng với InnoDB            |
| `sort_buffer_size`             | 256K – 1M                | RAM cho mỗi sort operation per connection                |
| `join_buffer_size`             | 256K – 1M                | RAM cho full scan joins                                   |
| `tmp_table_size`               | 64M                       | Kích thước tối đa cho in-memory temp tables              |
| `max_heap_table_size`          | 64M                       | Giới hạn MEMORY tables — phải bằng tmp_table_size        |

#### Connections (Kết Nối)

| Tham Số            | Giá Trị Khuyến Nghị | Mô Tả                                         |
| ------------------- | -------------------- | --------------------------------------------- |
| `max_connections`   | 200-500 (phụ thuộc RAM) | Số kết nối đồng thời tối đa                |
| `wait_timeout`      | 28800 (8h)          | Thời gian đóng kết nối idle (Nhàn Rỗi)      |
| `interactive_timeout` | 28800             | Tương tự wait_timeout cho interactive sessions |
| `connect_timeout`   | 10                   | Timeout khi thiết lập kết nối               |

> **Công thức max_connections an toàn:**
> `max_connections = (RAM_GB × 1024 - innodb_buffer_pool_MB) / 1.5`

#### I/O và Durability (Độ Bền)

| Tham Số                            | Giá Trị       | Mô Tả                                                     |
| ----------------------------------- | -------------- | --------------------------------------------------------- |
| `innodb_flush_log_at_trx_commit`   | 1 (Production) | 1 = ACID strict; 2 = flush mỗi giây (nhanh hơn, ít an toàn hơn) |
| `sync_binlog`                       | 1              | Đồng bộ binary log mỗi transaction — cần cho HA          |
| `innodb_flush_method`              | O_DIRECT       | Tránh double-buffering với OS cache                        |
| `innodb_log_file_size`             | 256M – 1G     | Redo log size — lớn hơn = hiệu năng write tốt hơn        |

#### Slow Query Log (Nhật Ký Truy Vấn Chậm)

| Tham Số                    | Giá Trị Khuyến Nghị | Mô Tả                                    |
| --------------------------- | -------------------- | ---------------------------------------- |
| `slow_query_log`            | 1 (ON)              | Bật slow query log — bắt buộc production |
| `long_query_time`           | 1 (giây)            | Log queries chậm hơn 1 giây             |
| `log_queries_not_using_indexes` | 0               | Log queries không dùng index (cẩn thận — có thể flood log) |
| `min_examined_row_limit`    | 0                   | Chỉ log nếu scan > N rows               |

### PostgreSQL — Tham Số Quan Trọng

#### Bộ Nhớ

| Tham Số                   | Giá Trị Khuyến Nghị  | Mô Tả                                              |
| -------------------------- | --------------------- | -------------------------------------------------- |
| `shared_buffers`           | 25% của RAM           | Buffer cache chính của PostgreSQL                  |
| `effective_cache_size`     | 75% của RAM           | Gợi ý cho query planner về OS cache               |
| `work_mem`                 | 64M – 256M            | RAM cho mỗi sort/hash — cẩn thận với nhiều connections |
| `maintenance_work_mem`     | 256M – 1G             | RAM cho VACUUM, CREATE INDEX, ALTER TABLE          |
| `wal_buffers`              | 64M                   | Buffer cho WAL (Write-Ahead Log — Nhật Ký Ghi Trước) |

> **Cảnh báo `work_mem`:** Mỗi query có thể dùng `work_mem` nhiều lần (mỗi sort/hash node). Với 100 connections × 10 operations = `100 × 10 × work_mem` RAM tổng cộng.

#### WAL và Checkpoints

| Tham Số                        | Giá Trị Khuyến Nghị | Mô Tả                                                 |
| ------------------------------- | -------------------- | ----------------------------------------------------- |
| `checkpoint_completion_target`  | 0.9                  | Trải đều I/O checkpoint trong 90% khoảng thời gian  |
| `wal_level`                     | replica              | Cần cho logical replication và read replicas          |
| `max_wal_size`                  | 1GB – 4GB           | Kích thước WAL tối đa trước khi force checkpoint     |
| `min_wal_size`                  | 80MB                 | Giữ WAL tối thiểu trong disk                          |

#### Query Planner (Bộ Lập Kế Hoạch Truy Vấn)

| Tham Số                | Giá Trị    | Mô Tả                                                |
| ----------------------- | ----------- | ---------------------------------------------------- |
| `random_page_cost`      | 1.1 (SSD)  | Chi phí ước tính random page read — thấp hơn cho SSD |
| `effective_io_concurrency` | 200 (SSD) | Số I/O requests đồng thời — cao hơn cho SSD         |
| `enable_seqscan`        | on          | Cho phép sequential scan (không tắt trừ debugging)  |

---

## Option Groups — Nhóm Tùy Chọn

### Option Group Là Gì?

Option Groups cung cấp các **tính năng bổ sung** cho database engine — chức năng không có trong cấu hình mặc định.

**Quan trọng:** Option Groups khác với Parameter Groups:
- **Parameter Groups**: thay đổi hành vi **engine** (bộ nhớ, kết nối, I/O)
- **Option Groups**: thêm **tính năng mới** vào engine

### Các Options Thường Dùng

#### MySQL Options

| Option                          | Mô Tả                                               |
| -------------------------------- | --------------------------------------------------- |
| `MARIADB_AUDIT_PLUGIN`          | Audit plugin — ghi log mọi SQL statements (Câu Lệnh SQL) |
| `MYSQL_AUDIT_PLUGIN`            | MySQL Enterprise Audit (Oracle support)              |

#### Oracle Options

| Option                    | Mô Tả                                                    |
| -------------------------- | -------------------------------------------------------- |
| `OEM`                      | Oracle Enterprise Manager (Quản Lý Doanh Nghiệp Oracle) |
| `OEM_AGENT`                | OEM agent deployment                                     |
| `APEX`                     | Oracle Application Express (Ứng Dụng Phát Triển Nhanh) |
| `STATSPACK`                | Performance statistics collection                        |
| `TDE`                      | Transparent Data Encryption (Mã Hóa Dữ Liệu Trong Suốt) |
| `Timezone`                 | Database timezone files update                           |

#### SQL Server Options

| Option                     | Mô Tả                                                                  |
| --------------------------- | ---------------------------------------------------------------------- |
| `TDE`                       | Transparent Data Encryption                                             |
| `SSRS`                      | SQL Server Reporting Services (Dịch Vụ Báo Cáo)                       |
| `SQLSERVER_BACKUP_RESTORE`  | Native backup/restore to S3                                             |

---

## Quy Trình Thay Đổi Cấu Hình

### An Toàn Khi Thay Đổi Tham Số

```
Quy trình chuẩn cho production:
┌─────────────────────────────────────────────────────┐
│ 1. Test trên môi trường dev/staging trước          │
│ 2. Backup instance trước khi thay đổi              │
│ 3. Đọc documentation của tham số — hiểu side effects│
│ 4. Chọn apply method phù hợp:                      │
│    - Dynamic: Apply immediately (an toàn)           │
│    - Static: Lên kế hoạch maintenance window       │
│ 5. Monitor metrics sau khi thay đổi                │
│    - CloudWatch: CPUUtilization, FreeableMemory     │
│    - Slow query log                                 │
│ 6. Có kế hoạch rollback (Quay Lại)                 │
└─────────────────────────────────────────────────────┘
```

### Pending Reboot Parameters

Khi thay đổi static parameter, instance cần reboot:

```
AWS Console sẽ hiển thị:
"Parameter group status: pending-reboot"

Cách reboot:
- Multi-AZ: Failover + reboot = 60-120 giây downtime
- Single-AZ: Reboot thẳng = 5-10 phút downtime
```

### Kiểm Tra Giá Trị Tham Số Hiện Tại

```sql
-- MySQL
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE '%timeout%';
SHOW GLOBAL STATUS LIKE 'Threads_connected';

-- PostgreSQL
SHOW shared_buffers;
SHOW work_mem;
SELECT name, setting, unit, source FROM pg_settings WHERE source != 'default';
```

---

## Best Practices — Thực Hành Tốt Nhất

### 1. Luôn Dùng Custom Parameter Group

```
❌ Sai: Dùng default parameter group
✅ Đúng: Tạo custom parameter group, kế thừa từ default
         Đặt tên có ý nghĩa: {app}-{engine}{version}-{env}
         Ví dụ: ecommerce-mysql80-prod
```

### 2. Dùng Formula Expressions Cho Memory

RDS hỗ trợ expressions tham chiếu đến instance memory:

```
innodb_buffer_pool_size = {DBInstanceClassMemory*3/4}

Điều này tự động tính toán dựa trên instance size:
- db.r7g.large (16 GB): buffer pool = 12 GB
- db.r7g.xlarge (32 GB): buffer pool = 24 GB
- db.r7g.2xlarge (64 GB): buffer pool = 48 GB

→ Không cần thay đổi parameter group khi scale instance!
```

### 3. Bật Slow Query Log Ngay Từ Đầu

```sql
-- Tham số cần thiết:
slow_query_log = 1
long_query_time = 1         -- Log queries > 1 giây
log_output = FILE           -- Ghi ra file (có thể query qua CloudWatch)
```

### 4. Tách Parameter Groups Theo Environment

```
ecommerce-mysql80-prod      ← Strict settings, performance-tuned
ecommerce-mysql80-staging   ← Tương tự prod nhưng có thể khác về logging
ecommerce-mysql80-dev       ← Nhiều logging hơn, stricter SQL mode cho catch bugs
```

### 5. Document Mọi Thay Đổi

```markdown
## Parameter Group Changes Log

| Ngày       | Tham Số                    | Giá Trị Cũ | Giá Trị Mới | Lý Do                    |
| ---------- | -------------------------- | ---------- | ------------ | ------------------------ |
| 2026-01-15 | innodb_buffer_pool_size    | default    | {Mem*3/4}   | Tăng cache hit rate      |
| 2026-02-01 | max_connections            | 151        | 500          | Scale application team   |
| 2026-03-10 | slow_query_log             | 0          | 1            | Enable monitoring        |
```

---

## Câu Hỏi Phỏng Vấn

**Q: Tại sao không thể chỉnh sửa Default Parameter Group?**

AWS giữ default parameter groups chỉ đọc (read-only) để:
- Đảm bảo consistency (Nhất Quán) — nhiều instances dùng default group, nếu thay đổi sẽ ảnh hưởng tất cả
- Dễ rollback (Quay Lại) về mặc định
- Audit trail (Vết Kiểm Toán) rõ ràng — mọi thay đổi đều nằm trong custom group

**Q: innodb_buffer_pool_size nên đặt bao nhiêu?**

- MySQL: **75% của RAM** là guideline phổ biến
- Ví dụ: instance 64 GB RAM → buffer pool = 48 GB
- Nhưng phải trừ đi: OS overhead (~1 GB), connection overhead (~2 MB/connection × max_connections), innodb_log_buffer, tmp_table_size
- Dùng formula expression `{DBInstanceClassMemory*3/4}` trong Parameter Group để tự động tính

**Q: Khác nhau giữa Parameter Groups và Option Groups?**

- **Parameter Groups**: Kiểm soát **hành vi** của engine — bộ nhớ, timeout, logging, I/O behavior
- **Option Groups**: Thêm **tính năng** vào engine — Oracle APEX, TDE, backup/restore tích hợp S3
- Parameter Groups áp dụng cho MySQL, PostgreSQL, MariaDB, Oracle, SQL Server
- Option Groups chủ yếu liên quan Oracle, SQL Server (MySQL và PostgreSQL ít dùng hơn)

**Q: Làm thế nào để thay đổi static parameter mà không có downtime?**

Với **Multi-AZ**:
1. Thay đổi parameter group → pending-reboot
2. Reboot với force-failover → failover sang standby (60-120s downtime)
3. Old primary reboot và trở thành new standby

Downtime 60-120 giây là không thể tránh với static parameters trên RDS thuần.
Với **Aurora**: có thể rolling restart không downtime.

---

## 🔗 Điều Hướng

- **Trước:** [4-read-replicas.md](4-read-replicas.md) — Read Replicas
- **Tiếp theo:** [6-rds-proxy.md](6-rds-proxy.md) — RDS Proxy
- **Liên quan:** [../07-performance-tuning/README.md](../07-performance-tuning/README.md) — Performance Tuning

---

**Cập Nhật Lần Cuối:** 2026-05-15
