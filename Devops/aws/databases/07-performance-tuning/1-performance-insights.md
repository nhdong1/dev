# RDS Performance Insights — Thông Tin Hiệu Năng RDS

> RDS Performance Insights (Thông Tin Hiệu Năng RDS) là công cụ giám sát hiệu năng cơ sở dữ liệu được tích hợp sẵn trong AWS, cho phép phân tích tải database theo thời gian thực và xác định các SQL statements gây tắc nghẽn.

## 📚 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Kiến Trúc & Cách Hoạt Động](#kiến-trúc--cách-hoạt-động)
3. [Dashboard — Bảng Điều Khiển](#dashboard--bảng-điều-khiển)
4. [DB Load — Tải Cơ Sở Dữ Liệu](#db-load--tải-cơ-sở-dữ-liệu)
5. [Wait Events — Sự Kiện Chờ](#wait-events--sự-kiện-chờ)
6. [Top SQL Analysis — Phân Tích SQL Hàng Đầu](#top-sql-analysis--phân-tích-sql-hàng-đầu)
7. [Cấu Hình & Thiết Lập](#cấu-hình--thiết-lập)
8. [Use Cases Thực Tế](#use-cases-thực-tế)
9. [Tích Hợp CloudWatch](#tích-hợp-cloudwatch)
10. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🎯 Tổng Quan

### Performance Insights Là Gì?

RDS Performance Insights là dịch vụ giám sát hiệu năng cấp database, cung cấp cái nhìn sâu vào:

- **DB Load** — tải tức thời của database theo đơn vị AAS (Average Active Sessions — Số Phiên Hoạt Động Trung Bình)
- **Wait Events** — loại tài nguyên mà sessions đang chờ
- **Top SQL** — những câu truy vấn đóng góp nhiều nhất vào tải

### Sự Khác Biệt Với CloudWatch

| Tiêu Chí | CloudWatch | Performance Insights |
|---------|-----------|---------------------|
| Góc nhìn | Hạ tầng (CPU, RAM, IOPS) | Database-level (SQL, waits) |
| Granularity — Độ Chi Tiết | 1 phút (mặc định) | 1 giây |
| Query analysis | Không | Có |
| Wait event analysis | Không | Có |
| Chi phí | Miễn phí (metrics cơ bản) | Miễn phí 7 ngày; tính phí retention dài hơn |

### Hỗ Trợ Engine

| Engine | Hỗ Trợ |
|--------|--------|
| MySQL 5.6+ | ✅ |
| PostgreSQL 9.6+ | ✅ |
| MariaDB 10.2+ | ✅ |
| Aurora MySQL | ✅ |
| Aurora PostgreSQL | ✅ |
| Oracle | ✅ |
| SQL Server | ✅ |

---

## 🏗️ Kiến Trúc & Cách Hoạt Động

### Cơ Chế Thu Thập Dữ Liệu

```
┌─────────────────────────────────────────────────────────────────┐
│  RDS/Aurora Instance                                             │
│                                                                   │
│  ┌─────────────────┐    ┌──────────────────────────────────┐   │
│  │  DB Engine      │    │  Performance Insights Agent      │   │
│  │  (MySQL/PG...)  │───→│                                  │   │
│  │                 │    │  - Sampling active sessions      │   │
│  │  pg_stat_activity│   │  - Capturing wait events         │   │
│  │  performance_   │   │  - Aggregating SQL digests        │   │
│  │  schema.events  │   │                                  │   │
│  └─────────────────┘    └──────────────────┬─────────────-┘   │
│                                             │                   │
└─────────────────────────────────────────────┼───────────────────┘
                                              │ HTTPS
                                              ▼
                              ┌───────────────────────────┐
                              │  Performance Insights API  │
                              │  (Lưu trữ time-series)    │
                              └───────────────┬───────────┘
                                              │
                                              ▼
                              ┌───────────────────────────┐
                              │  AWS Console Dashboard    │
                              │  / API / CloudWatch       │
                              └───────────────────────────┘
```

### Khái Niệm AAS (Average Active Sessions — Số Phiên Hoạt Động Trung Bình)

- Đơn vị đo lường chính của Performance Insights
- **AAS = 1** có nghĩa là trung bình có 1 session đang thực thi tại bất kỳ thời điểm nào
- **AAS > số vCPU** → database đang bị overloaded — quá tải

```
vCPU = 4
                              MAX CAPACITY LINE
                         ─────────────────────────── AAS = 4
                              ↑ Khi AAS > 4: DB bị quá tải
AAS |  ████
 3  |  ████ ████
 2  |  ████ ████ ████
 1  |  ████ ████ ████ ████ ████
    └──────────────────────────── Thời gian
```

---

## 📊 Dashboard — Bảng Điều Khiển

### Bố Cục Dashboard

```
┌────────────────────────────────────────────────────────────────────────┐
│  Performance Insights Dashboard                                         │
├─────────────────────────────────────────────────────────────────────── ┤
│                                                                          │
│  [Counter Metrics]  CPU | Connections | Commit rate | Deadlocks        │
│                                                                          │
│  ┌────────────────────── DB Load Chart ─────────────────────────────┐  │
│  │                                                                   │  │
│  │  AAS  █                                                           │  │
│  │   4   █  ███                                                      │  │
│  │   3   █  ███ ██                                                   │  │
│  │   2   █  ███ ███ █                                                │  │
│  │   1   █  ███ ███ █ ██                                             │  │
│  │   0   ─────────────────────────────────────────────              │  │
│  │        [Màu theo wait event hoặc SQL digest]                     │  │
│  └───────────────────────────────────────────────────────────────── ┘  │
│                                                                          │
│  [Top SQL]                         [Top Wait Events]                    │
│  1. SELECT * FROM orders...        1. io/file/sql (CPU)                │
│  2. UPDATE inventory...            2. lock/table/sql                   │
│  3. INSERT INTO logs...            3. wait/io/file                     │
│                                                                          │
└──────────────────────────────────────────────────────────────────────── ┘
```

### Các Tab Chính

| Tab | Nội Dung |
|-----|---------|
| **Counter Metrics** | CPU, Connections, Commit/Rollback rate, Deadlocks |
| **DB Load** | AAS chart phân tích theo wait event hoặc SQL |
| **Top SQL** | Danh sách SQL statements tiêu tốn tài nguyên nhiều nhất |
| **Top Hosts** | Client hosts đóng góp nhiều load nhất |
| **Top Users** | Database users đóng góp nhiều load nhất |

---

## ⏳ Wait Events — Sự Kiện Chờ

### Wait Events Là Gì?

Wait event mô tả lý do một session đang **chờ** thay vì **thực thi**. Phân tích wait events giúp xác định đúng nút thắt cổ chai.

### Wait Events Phổ Biến — MySQL / Aurora MySQL

| Wait Event | Ý Nghĩa | Nguyên Nhân Thường Gặp |
|-----------|--------|----------------------|
| `io/file/sql/FRM` | Đọc file schema | Table nhiều columns, cold cache |
| `io/file/innodb/innodb_data_file` | InnoDB I/O chờ disk | Thiếu buffer pool, IOPS thấp |
| `lock/table/sql_lock` | Table lock contention — tranh chấp khóa bảng | DDL đang chạy, lock không release |
| `synch/mutex/innodb/...` | InnoDB mutex contention | Concurrency quá cao |
| `synch/rwlock/innodb/dict_operation` | Dictionary lock | Nhiều DDL đồng thời |
| `io/socket/sql/client_connection` | Network wait | Slow clients, network latency |
| `CPU` | Đang dùng CPU (không chờ) | Heavy computation, full table scan |

### Wait Events Phổ Biến — PostgreSQL / Aurora PostgreSQL

| Wait Event | Ý Nghĩa | Nguyên Nhân Thường Gặp |
|-----------|--------|----------------------|
| `Lock:relation` | Table-level lock | DDL blocking DML |
| `Lock:tuple` | Row-level lock | Long-running transactions |
| `Lock:transactionid` | Waiting for transaction to commit | High concurrency INSERT/UPDATE |
| `IO:DataFileRead` | Reading data from disk | Buffer cache miss |
| `IO:WALWrite` | Writing WAL (Write-Ahead Log — Nhật Ký Ghi Trước) | High write workload |
| `IPC:MessageQueueSend` | Sending message to parallel worker | Parallel query overhead |
| `Client:ClientRead` | Waiting for client to send data | Slow application |
| `CPU` | On CPU | Computation, sort, hash |

### Cách Đọc Wait Events

```
DB Load Chart — Màu Sắc Theo Wait Event:

████ CPU          → Tốt (đang xử lý thực sự)
████ IO           → Disk I/O bottleneck → cần tăng IOPS hoặc tối ưu query
████ Lock         → Lock contention → xem xét transaction design
████ Network      → Slow network hoặc slow client
████ Concurrency  → Mutex/semaphore → giảm concurrency hoặc scale
```

---

## 🔍 Top SQL Analysis — Phân Tích SQL Hàng Đầu

### SQL Digest — Điểm Mấu Chốt

Performance Insights không lưu từng query riêng lẻ mà gom nhóm theo **SQL digest** (chuỗi đại diện của câu truy vấn sau khi thay thế literal values bằng `?`).

Ví dụ:
```sql
-- Hai câu truy vấn này được gom vào cùng digest:
SELECT * FROM users WHERE id = 42;
SELECT * FROM users WHERE id = 100;

-- Digest: SELECT * FROM users WHERE id = ?
```

### Metrics Cho Mỗi SQL Statement

| Metric | Ý Nghĩa |
|--------|--------|
| **% of Total Load** | Phần trăm tổng DB load mà SQL này đóng góp |
| **Rows Examined** | Số hàng được quét (cao → thiếu index) |
| **Rows Sent** | Số hàng trả về client |
| **Avg Latency** | Thời gian thực thi trung bình |
| **Calls** | Số lần gọi trong khoảng thời gian |
| **Total Load** | AAS tổng cộng = Latency × Calls / thời gian |

### Workflow Phân Tích Top SQL

```
1. Mở Performance Insights Dashboard
   → Chọn khoảng thời gian có vấn đề (slow period)

2. Nhìn vào DB Load Chart
   → Màu nào chiếm nhiều nhất? (IO? Lock? CPU?)

3. Chuyển sang tab Top SQL
   → Sort by "% of total" để tìm SQL đóng góp nhiều nhất

4. Click vào SQL statement đó
   → Xem execution history
   → Xem wait events breakdown

5. Click "Explain" (nếu được hỗ trợ)
   → Xem execution plan để tìm Full Table Scan

6. Đề xuất giải pháp:
   → Thiếu index?          → Thêm index
   → Rows examined >> sent? → Query không selective, cần index tốt hơn
   → Lock nhiều?           → Tối ưu transaction scope
   → IO nhiều?             → Index hoặc tăng buffer pool / IOPS
```

---

## ⚙️ Cấu Hình & Thiết Lập

### Kích Hoạt Performance Insights

**Qua AWS Console:**
```
RDS Console → Databases → Chọn DB → Modify
→ Monitoring → Performance Insights: Enable
→ Retention Period: 7 days (miễn phí) hoặc tối đa 24 tháng (có phí)
→ Encryption: Chọn KMS key (nếu cần)
→ Apply immediately hoặc trong maintenance window
```

**Qua AWS CLI:**
```bash
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --enable-performance-insights \
  --performance-insights-retention-period 7 \
  --no-reboot
```

**Qua CloudFormation / Terraform:**
```yaml
# CloudFormation
DBInstance:
  Type: AWS::RDS::DBInstance
  Properties:
    EnablePerformanceInsights: true
    PerformanceInsightsRetentionPeriod: 7
    PerformanceInsightsKMSKeyId: !Ref KMSKey
```

```hcl
# Terraform
resource "aws_db_instance" "main" {
  performance_insights_enabled          = true
  performance_insights_retention_period = 7
  performance_insights_kms_key_id       = aws_kms_key.rds.arn
}
```

### Performance Insights Retention — Thời Gian Lưu Trữ

| Retention | Chi Phí |
|----------|--------|
| 7 ngày | Miễn phí |
| 1 tháng | ~$0.02/vCPU/giờ |
| 2 năm | ~$0.02/vCPU/giờ |

### Performance Schema (MySQL) — Cần Kích Hoạt

Performance Insights dựa vào `performance_schema` của MySQL. Cần đảm bảo:

```sql
-- Kiểm tra performance_schema
SHOW VARIABLES LIKE 'performance_schema';
-- Kết quả mong muốn: ON

-- Xem các consumers đang bật
SELECT * FROM performance_schema.setup_consumers
WHERE enabled = 'YES';

-- Bật nếu cần (chỉ cho development — ảnh hưởng nhẹ đến hiệu năng)
UPDATE performance_schema.setup_consumers
SET enabled = 'YES'
WHERE name LIKE 'events_statements%';
```

---

## 💡 Use Cases Thực Tế

### Case 1 — Tìm Query Gây Cao CPU

**Triệu Chứng:**
- CPU Utilization 95%+ liên tục
- Người dùng báo chậm

**Chẩn Đoán với Performance Insights:**
```
1. Dashboard → DB Load → Chọn "Slice by: SQL"
2. Thấy 1 SQL màu đỏ chiếm 70% load
3. Click vào SQL đó:
   SELECT * FROM orders o
   JOIN order_items oi ON o.id = oi.order_id
   WHERE o.created_at > '2024-01-01'

4. Metrics:
   - Rows Examined: 5,000,000
   - Rows Sent: 1,200
   - Ratio: 4,167:1 → Rất không hiệu quả

5. Chạy EXPLAIN:
   → Seq Scan on order_items (cost=0.00..800000 rows=5M)
   → Không có index trên order_id
```

**Giải Pháp:**
```sql
CREATE INDEX idx_order_items_order_id ON order_items(order_id);

-- Sau khi tạo index:
-- Rows Examined: 1,200
-- Rows Sent: 1,200
-- Ratio: 1:1 → Hoàn hảo
```

### Case 2 — Chẩn Đoán Lock Contention (Tranh Chấp Khóa)

**Triệu Chứng:**
- DB Load cao nhưng CPU thấp
- Latency tăng đột biến trong giờ cao điểm

**Chẩn Đoán:**
```
1. Performance Insights → DB Load → Slice by Wait Event
2. Thấy màu "Lock:tuple" chiếm 60% load

3. Top SQL đang bị lock:
   UPDATE inventory SET quantity = quantity - 1
   WHERE product_id = 999

4. Xem "Blocking queries":
   → Transaction A giữ row lock trên product_id = 999 trong 30 giây
   → Transaction B đang chờ
```

**Giải Pháp:**
```sql
-- Phát hiện deadlocks (PostgreSQL)
SELECT * FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE NOT granted;

-- Giải pháp:
-- 1. Giảm transaction scope — không để transaction mở quá lâu
-- 2. Thêm timeout: SET lock_timeout = '5s';
-- 3. Cân nhắc SKIP LOCKED nếu business logic cho phép
SELECT * FROM tasks WHERE status = 'pending'
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

### Case 3 — Tìm N+1 Query Problem

**Triệu Chứng:**
- Số lượng query rất cao (hàng triệu calls/phút)
- Mỗi query nhỏ nhưng tổng tải lớn

**Phát Hiện:**
```
Top SQL:
1. SELECT * FROM users WHERE id = ?   → 500,000 calls/phút, 0.1ms each
   Tổng: 50 AAS

2. SELECT * FROM orders WHERE user_id = ?  → 500,000 calls/phút
   Tổng: 50 AAS

→ Đây là N+1 query: với mỗi user, gọi thêm 1 query lấy orders
```

**Giải Pháp:**
```sql
-- Thay N+1 queries bằng 1 JOIN query:
SELECT u.*, o.*
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.id IN (1, 2, 3, ...);

-- Hoặc dùng batch loading trong ORM
```

---

## 🔗 Tích Hợp CloudWatch

### Publish Performance Insights Metrics to CloudWatch

```bash
# Lấy metrics qua API
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier db:XXXXX \
  --metric-queries '[{
    "Metric": "db.load.avg",
    "GroupBy": {"Group": "db.sql", "Limit": 5}
  }]' \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T01:00:00Z \
  --period-in-seconds 60
```

### CloudWatch Alarm cho DB Load

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "RDS-High-DBLoad" \
  --metric-name DBLoad \
  --namespace AWS/RDS \
  --dimensions Name=DBInstanceIdentifier,Value=mydb \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 4 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:...
```

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Q: Performance Insights khác gì Enhanced Monitoring?

```
Performance Insights:
- Góc nhìn database-level: SQL statements, wait events
- Trả lời: "Câu SQL nào gây chậm?"
- Granularity: 1 giây
- Chỉ RDS/Aurora

Enhanced Monitoring:
- Góc nhìn OS-level: CPU per process, memory, disk I/O
- Trả lời: "Process nào dùng nhiều CPU?"
- Granularity: 1 giây đến 60 giây
- Hiển thị qua CloudWatch Logs

Cả hai bổ trợ nhau: dùng cả hai để có cái nhìn đầy đủ.
```

### Q: DB Load AAS là gì và ý nghĩa thực tế?

```
AAS (Average Active Sessions):
- = Số sessions đang active trung bình tại một thời điểm
- Đơn vị thể hiện "lượng công việc" database đang xử lý

Ý nghĩa:
- AAS < vCPU count: Database có headroom, không overloaded
- AAS = vCPU count: Database đang dùng hết capacity
- AAS > vCPU count: Database bị overloaded, sẽ có queue

Ví dụ thực tế:
- db.r5.xlarge = 4 vCPU
- AAS = 8 → Database đang xử lý gấp đôi capacity
- Triệu chứng: latency tăng, timeout, slow queries
```

### Q: Khi nào nên dùng Performance Insights thay vì slow query log?

```
Performance Insights:
+ Real-time, không cần config sẵn
+ Wait event analysis
+ Lịch sử theo thời gian (trend analysis)
+ Không ảnh hưởng đến performance
- Cần trả phí cho retention dài

Slow Query Log:
+ Lưu đầy đủ thông tin query (không phải digest)
+ Có thể filter theo query time, rows examined
+ Miễn phí, lưu trong CloudWatch Logs
- Cần bật và config trước
- Ảnh hưởng nhẹ đến performance khi ghi log nhiều

Thực tế: dùng cả hai — Performance Insights để triage nhanh,
slow query log để lấy full query text và phân tích offline.
```

---

**Tiếp Theo:** [2-cloudwatch-metrics.md](2-cloudwatch-metrics.md) — CloudWatch Metrics & Enhanced Monitoring

**Cập Nhật Lần Cuối:** 2026-05-15
