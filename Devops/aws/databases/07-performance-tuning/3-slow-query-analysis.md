# Slow Query Analysis — Phân Tích Truy Vấn Chậm

> Slow query log (nhật ký truy vấn chậm) và EXPLAIN plans (kế hoạch thực thi) là hai công cụ không thể thiếu để chẩn đoán và khắc phục các truy vấn SQL kém hiệu quả trong môi trường RDS và Aurora.

## 📚 Mục Lục

1. [Slow Query Log — Nhật Ký Truy Vấn Chậm](#slow-query-log--nhật-ký-truy-vấn-chậm)
2. [Cấu Hình Slow Query Log Trên RDS](#cấu-hình-slow-query-log-trên-rds)
3. [Phân Tích Slow Query Log](#phân-tích-slow-query-log)
4. [EXPLAIN — Kế Hoạch Thực Thi (MySQL)](#explain--kế-hoạch-thực-thi-mysql)
5. [EXPLAIN ANALYZE — PostgreSQL](#explain-analyze--postgresql)
6. [Các Pattern Query Chậm Phổ Biến](#các-pattern-query-chậm-phổ-biến)
7. [Quy Trình Tối Ưu Query](#quy-trình-tối-ưu-query)
8. [Công Cụ Phân Tích](#công-cụ-phân-tích)
9. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 📋 Slow Query Log — Nhật Ký Truy Vấn Chậm

### Slow Query Log Là Gì?

Slow query log ghi lại tất cả SQL statements có thời gian thực thi vượt quá ngưỡng cấu hình (`long_query_time`). Đây là nguồn dữ liệu quan trọng để:

- Xác định queries cần tối ưu
- Phân tích pattern truy cập
- Phát hiện N+1 query problems
- Audit performance regressions — hồi quy hiệu năng

### Thông Tin Trong Mỗi Entry

```
# Time: 2024-01-15T10:30:00.000000Z
# User@Host: appuser[appuser] @ 10.0.1.100 []  Id: 12345
# Query_time: 5.234567  Lock_time: 0.000123  Rows_sent: 1  Rows_examined: 5000000
# Bytes_sent: 256
use myapp_db;
SET timestamp=1705315800;
SELECT o.*, c.name, c.email
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'pending'
AND o.created_at > '2024-01-01 00:00:00'
ORDER BY o.created_at DESC
LIMIT 10;
```

Giải thích từng trường:
- `Query_time` — Tổng thời gian thực thi (giây)
- `Lock_time` — Thời gian chờ lock
- `Rows_sent` — Số hàng trả về client
- `Rows_examined` — Số hàng database phải đọc
- Tỷ lệ `Rows_examined / Rows_sent` cao → thiếu index hiệu quả

---

## ⚙️ Cấu Hình Slow Query Log Trên RDS

### MySQL / Aurora MySQL

**Qua Parameter Group:**
```bash
# Tạo custom parameter group
aws rds create-db-parameter-group \
  --db-parameter-group-name myapp-params \
  --db-parameter-group-family mysql8.0 \
  --description "Custom params for myapp"

# Cấu hình các tham số
aws rds modify-db-parameter-group \
  --db-parameter-group-name myapp-params \
  --parameters \
    "ParameterName=slow_query_log,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=long_query_time,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=log_queries_not_using_indexes,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=log_output,ParameterValue=FILE,ApplyMethod=immediate" \
    "ParameterName=general_log,ParameterValue=0,ApplyMethod=immediate"
```

**Tham Số Quan Trọng:**

| Tham Số | Giá Trị Khuyến Nghị | Ý Nghĩa |
|---------|--------------------|----|
| `slow_query_log` | 1 | Bật slow query log |
| `long_query_time` | 1 (giây) | Ngưỡng thời gian — query chậm hơn mức này bị log |
| `log_queries_not_using_indexes` | 1 | Log cả queries không dùng index (dù nhanh) |
| `log_output` | FILE | Ghi ra file (dùng CloudWatch Logs) |
| `log_slow_admin_statements` | 1 | Log cả ALTER TABLE, ANALYZE... |
| `min_examined_row_limit` | 1000 | Chỉ log query scan > 1000 rows |

### PostgreSQL / Aurora PostgreSQL

**Qua Parameter Group:**
```bash
aws rds modify-db-parameter-group \
  --db-parameter-group-name myapp-pg-params \
  --parameters \
    "ParameterName=log_min_duration_statement,ParameterValue=1000,ApplyMethod=immediate" \
    "ParameterName=log_statement,ParameterValue=none,ApplyMethod=immediate" \
    "ParameterName=log_duration,ParameterValue=0,ApplyMethod=immediate" \
    "ParameterName=log_lock_waits,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=log_temp_files,ParameterValue=0,ApplyMethod=immediate" \
    "ParameterName=auto_explain.log_min_duration,ParameterValue=1000,ApplyMethod=immediate" \
    "ParameterName=auto_explain.log_analyze,ParameterValue=1,ApplyMethod=immediate"
```

| Tham Số | Giá Trị | Ý Nghĩa |
|---------|---------|--------|
| `log_min_duration_statement` | 1000 (ms) | Log query chạy > 1 giây |
| `log_lock_waits` | 1 | Log khi chờ lock quá `deadlock_timeout` |
| `log_temp_files` | 0 | Log tất cả queries dùng temp files (sort, hash join) |
| `auto_explain.log_min_duration` | 1000 | Tự động log EXPLAIN plan cho queries chậm |
| `auto_explain.log_analyze` | 1 | Dùng EXPLAIN ANALYZE (có actual rows) |
| `auto_explain.log_buffers` | 1 | Log buffer usage |

### Đọc Slow Query Log Trên RDS

```bash
# Download slow query log file
aws rds download-db-log-file-portion \
  --db-instance-identifier mydb \
  --log-file-name slowquery/mysql-slowquery.log \
  --output text

# Xem danh sách log files
aws rds describe-db-log-files \
  --db-instance-identifier mydb \
  --filename-contains slowquery
```

### Gửi Slow Query Log vào CloudWatch Logs

```bash
# Cấu hình exports khi tạo/modify instance
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --cloudwatch-logs-export-configuration '{"EnableLogTypes":["slowquery","error","general"]}'
```

---

## 🔍 Phân Tích Slow Query Log

### Dùng mysqldumpslow (MySQL)

```bash
# Download log file về local (hoặc chạy trên bastion host)
# Tìm top 10 queries theo thời gian trung bình
mysqldumpslow -s at -t 10 /var/log/mysql/slow.log

# Tìm top 10 queries có nhiều rows examined nhất
mysqldumpslow -s r -t 10 /var/log/mysql/slow.log

# Tìm queries theo pattern
mysqldumpslow -s at -t 5 -a /var/log/mysql/slow.log

# Output mẫu:
# Count: 150  Time=5.23s (784s)  Lock=0.00s (0s)  Rows=1.0 (150), user[user]@host
# SELECT * FROM orders WHERE customer_id = N AND status = S
```

### Dùng pt-query-digest (Percona Toolkit)

```bash
# Cài đặt Percona Toolkit
yum install percona-toolkit

# Phân tích slow query log
pt-query-digest /var/log/mysql/slow.log

# Output:
# # Profile
# # Rank Query ID                            Response time  Calls R/Call  V/M   Item
# # ==== =================================== ============== ===== ======= ===== ====
# #    1 0x89B4BA93C0F7B7C8D0F98AE8F1B4F     5.2345 35.4%    150  0.0349  0.02 SELECT orders
# #    2 0x4F1A7B2E9C3D1234567890ABCDEF01     4.1234 27.8%     89  0.0463  0.01 UPDATE inventory
```

### Phân Tích Bằng CloudWatch Logs Insights

```
# CloudWatch Logs Insights query để tìm top slow queries
fields @timestamp, @message
| filter @message like /Query_time/
| parse @message "Query_time: * " as query_time
| stats avg(query_time) as avg_time,
        max(query_time) as max_time,
        count(*) as count
  by @message
| sort avg_time desc
| limit 20
```

---

## 📊 EXPLAIN — Kế Hoạch Thực Thi (MySQL)

### Syntax

```sql
-- MySQL — Xem execution plan
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- Với thêm thông tin về index
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE customer_id = 42;

-- EXPLAIN ANALYZE — chạy thật và hiển thị actual statistics
-- (MySQL 8.0+, Aurora MySQL 3+)
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

### Đọc Output EXPLAIN (MySQL)

```
+----+-------------+--------+------------+------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table  | partitions | type | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+--------+------------+------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | orders | NULL       | ref  | idx_customer  | idx_cust| 4       | const | 1250 |   100.00 | NULL  |
+----+-------------+--------+------------+------+---------------+---------+---------+-------+------+----------+-------+
```

### Giải Thích Từng Cột

**Cột `type` — Loại Join (quan trọng nhất)**

| Giá Trị | Ý Nghĩa | Hiệu Quả |
|---------|--------|---------|
| `system` | Bảng chỉ có 1 row | Tốt nhất |
| `const` | Dùng PRIMARY KEY hoặc UNIQUE với constant | Tốt nhất |
| `eq_ref` | Dùng PRIMARY KEY hoặc UNIQUE join | Rất tốt |
| `ref` | Dùng non-unique index | Tốt |
| `range` | Index range scan | Chấp nhận được |
| `index` | Full index scan | Chậm |
| `ALL` | **Full table scan — NGUY HIỂM** | Tệ nhất |

**Cột `Extra` — Thông Tin Bổ Sung**

| Giá Trị | Ý Nghĩa | Hành Động |
|---------|--------|----------|
| `Using index` | Covering index — không cần đọc table | Tốt |
| `Using where` | Filter được áp dụng sau index | Bình thường |
| `Using filesort` | Phải sort — không dùng index cho ORDER BY | Cần index |
| `Using temporary` | Dùng temp table — thường trong GROUP BY | Cần tối ưu |
| `Using join buffer` | Không có index cho join | Cần index |
| `Impossible WHERE` | WHERE condition không bao giờ true | Bug trong query |

**Cột `rows` — Ước Tính Số Hàng Quét**
- Đây là ước tính, không phải số thực
- Số nhỏ hơn = tốt hơn
- Tích `rows` của tất cả tables trong query ≈ tổng công việc

### Ví Dụ Thực Tế EXPLAIN

```sql
-- Query gốc — bị chậm
EXPLAIN SELECT o.id, o.total, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'pending'
AND o.created_at > '2024-01-01'
ORDER BY o.created_at DESC
LIMIT 10;

-- Output đáng lo ngại:
+----+-------------+-------+------+---------------+------+---------+------+----------+----------------------------------------------+
| id | select_type | table | type | possible_keys | key  | key_len | rows | filtered | Extra                                        |
+----+-------------+-------+------+---------------+------+---------+------+----------+----------------------------------------------+
|  1 | SIMPLE      | o     | ALL  | NULL          | NULL | NULL    | 5M   |    0.50  | Using where; Using filesort                 |
|  1 | SIMPLE      | c     | eq_ref| PRIMARY       | PRIMARY | 4  |    1 |  100.00  | NULL                                         |
+----+-------------+-------+------+---------------+------+---------+------+----------+----------------------------------------------+

Phát hiện vấn đề:
- orders: type=ALL → Full table scan 5 triệu rows
- Extra: Using filesort → ORDER BY không dùng index
- filtered: 0.50% → 5M rows quét nhưng chỉ 25,000 hàng relevant

Giải pháp:
CREATE INDEX idx_orders_status_created ON orders(status, created_at DESC);

-- Sau khi thêm index:
+----+-------------+-------+-------+---------------------+---------------------+---------+-------+------+----------+-------------+
| id | select_type | table | type  | possible_keys       | key                 | key_len | rows  | filtered | Extra   |
+----+-------------+-------+-------+---------------------+---------------------+---------+-------+----------+---------+
|  1 | SIMPLE      | o     | range | idx_orders_status_cr| idx_orders_status_cr| 1026    | 25000 |  100.00  | Using index condition |
|  1 | SIMPLE      | c     | eq_ref| PRIMARY             | PRIMARY             | 4       | 1     |  100.00  | NULL   |
+----+-------------+-------+-------+---------------------+---------------------+---------+-------+----------+---------+

Cải thiện: từ 5,000,000 rows quét xuống còn 25,000 rows
```

---

## 🐘 EXPLAIN ANALYZE — PostgreSQL

### Syntax

```sql
-- Chỉ xem plan (không chạy)
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- Chạy và hiển thị actual stats (an toàn với SELECT)
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;

-- Đầy đủ nhất — BUFFERS để xem cache hits
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE customer_id = 42;
```

### Đọc Output PostgreSQL

```
-- Ví dụ EXPLAIN ANALYZE output:
Nested Loop  (cost=1.00..450.23 rows=150 width=200) (actual time=0.035..12.450 rows=147 loops=1)
  ->  Index Scan using idx_orders_customer on orders  (cost=0.43..225.12 rows=150 width=200)
        (actual time=0.020..8.234 rows=147 loops=1)
        Index Cond: (customer_id = 42)
        Buffers: shared hit=12 read=85
  ->  Index Scan using customers_pkey on customers  (cost=0.43..1.50 rows=1 width=100)
        (actual time=0.028..0.028 rows=1 loops=147)
        Index Cond: (id = orders.customer_id)
        Buffers: shared hit=441
Planning Time: 0.456 ms
Execution Time: 12.892 ms
```

Giải thích:
- `cost=1.00..450.23` — ước tính cost (startup..total)
- `rows=150` — ước tính số rows
- `actual time=0.035..12.450` — thời gian thực (ms)
- `actual rows=147` — số rows thực tế (so với ước tính)
- `Buffers: shared hit=12 read=85` — 12 blocks từ cache, 85 từ disk

**Dấu Hiệu Cần Tối Ưu:**

| Dấu Hiệu | Vấn Đề | Hành Động |
|----------|--------|----------|
| `Seq Scan` trên bảng lớn | Full table scan | Thêm index |
| `rows=X` actual >> estimated | Statistics cũ | `ANALYZE tablename` |
| `Hash Join` với large hash | Hash table lớn, tốn memory | Tăng `work_mem`, thêm index join |
| `Sort` trên nhiều rows | Filesort | Thêm index cho ORDER BY |
| `Buffers: read=XXX` cao | Cache miss, đọc từ disk | Tăng `shared_buffers`, thêm index |
| `loops=N` với N lớn trong Nested Loop | Inefficient join | Tạo index hoặc dùng Hash Join |

### auto_explain — Tự Động Log Explain Plans

```sql
-- Bật auto_explain qua parameter group (Aurora PostgreSQL)
-- auto_explain.log_min_duration = 1000 (ms)
-- auto_explain.log_analyze = on
-- auto_explain.log_buffers = on
-- auto_explain.log_verbose = on

-- Hoặc bật cho session hiện tại:
LOAD 'auto_explain';
SET auto_explain.log_min_duration = 1000;
SET auto_explain.log_analyze = true;
```

---

## ⚠️ Các Pattern Query Chậm Phổ Biến

### Pattern 1 — Missing Index (Thiếu Chỉ Mục)

```sql
-- Trước: Full table scan
SELECT * FROM users WHERE email = 'user@example.com';
-- EXPLAIN: type=ALL, rows=1,000,000

-- Sau: Index lookup
CREATE INDEX idx_users_email ON users(email);
-- EXPLAIN: type=ref, rows=1
```

### Pattern 2 — Index Không Được Dùng Do Implicit Cast (Chuyển Kiểu Ngầm Định)

```sql
-- BAD: Cast ngầm vô hiệu hóa index trên user_id (VARCHAR)
SELECT * FROM users WHERE user_id = 12345;  -- 12345 là INT, user_id là VARCHAR
-- EXPLAIN: type=ALL — MySQL phải cast cả cột

-- GOOD: Đúng kiểu dữ liệu
SELECT * FROM users WHERE user_id = '12345';
-- EXPLAIN: type=ref
```

### Pattern 3 — Function Wrapping Column (Bọc Cột Trong Function)

```sql
-- BAD: Index trên created_at không được dùng
SELECT * FROM orders WHERE YEAR(created_at) = 2024;
-- EXPLAIN: type=ALL

-- GOOD: Range condition
SELECT * FROM orders
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
-- EXPLAIN: type=range (dùng index)
```

### Pattern 4 — LIKE Với Leading Wildcard (Ký Tự Đại Diện Đầu Chuỗi)

```sql
-- BAD: Leading wildcard vô hiệu hóa index
SELECT * FROM products WHERE name LIKE '%laptop%';
-- EXPLAIN: type=ALL

-- GOOD: Prefix search dùng được index
SELECT * FROM products WHERE name LIKE 'laptop%';
-- EXPLAIN: type=range

-- Giải pháp cho full-text search: dùng FULLTEXT index
ALTER TABLE products ADD FULLTEXT INDEX ft_name(name);
SELECT * FROM products WHERE MATCH(name) AGAINST('laptop');
```

### Pattern 5 — OR Condition Phá Vỡ Index

```sql
-- BAD: OR trên hai cột khác nhau
SELECT * FROM users WHERE email = 'a@a.com' OR phone = '0901234567';
-- EXPLAIN: type=ALL (trừ khi có index merge)

-- GOOD: Dùng UNION
SELECT * FROM users WHERE email = 'a@a.com'
UNION
SELECT * FROM users WHERE phone = '0901234567';
```

### Pattern 6 — N+1 Query

```sql
-- BAD: N+1 — gọi query trong vòng lặp
-- Application code:
for user in users:                            # 1 query
    orders = db.query("SELECT * FROM orders WHERE user_id = ?", user.id)  # N queries

-- GOOD: 1 query với JOIN hoặc IN clause
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.id IN (1, 2, 3, ..., N);
```

### Pattern 7 — SELECT * Thay Vì Cột Cụ Thể

```sql
-- BAD: Đọc toàn bộ row kể cả cột không cần
SELECT * FROM products;  -- Bao gồm description TEXT 10KB mỗi row

-- GOOD: Chỉ lấy cột cần thiết
SELECT id, name, price FROM products;

-- TUYỆT VỜI: Covering index
CREATE INDEX idx_products_name_price ON products(name, price);
SELECT name, price FROM products WHERE name LIKE 'laptop%';
-- EXPLAIN Extra: "Using index" — đọc chỉ từ index, không cần table access
```

### Pattern 8 — OFFSET Lớn Trong Pagination (Phân Trang)

```sql
-- BAD: OFFSET lớn phải đọc và bỏ qua rows
SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 100000;
-- Đọc 100,010 rows rồi bỏ 100,000

-- GOOD: Keyset pagination (Cursor-based pagination — Phân Trang Dựa Trên Con Trỏ)
SELECT * FROM orders WHERE id > :last_seen_id ORDER BY id LIMIT 10;
-- Đọc đúng 10 rows
```

---

## 🔄 Quy Trình Tối Ưu Query

```
1. IDENTIFY — Xác Định Query Chậm
   ├── Performance Insights → Top SQL by load
   ├── Slow query log → Long execution time
   └── Application APM → Slow endpoints

2. CAPTURE — Thu Thập Thông Tin
   ├── Ghi lại execution plan hiện tại (EXPLAIN)
   ├── Đo thời gian thực thi baseline
   └── Xác định bảng và columns liên quan

3. DIAGNOSE — Chẩn Đoán
   ├── type=ALL? → Thiếu index
   ├── rows_examined >> rows_sent? → Index kém selective
   ├── Using filesort? → ORDER BY không dùng index
   ├── Using temporary? → GROUP BY kém hiệu quả
   └── Lock_time cao? → Contention issue

4. OPTIMIZE — Tối Ưu
   ├── Thêm index phù hợp
   ├── Rewrite query
   ├── Denormalize nếu cần
   └── Tăng IOPS/memory nếu cần

5. VALIDATE — Xác Nhận
   ├── Chạy EXPLAIN sau khi thay đổi
   ├── Đo lại execution time
   ├── Kiểm tra production metrics (P95, P99)
   └── Monitor không có regression — hồi quy
```

---

## 🛠️ Công Cụ Phân Tích

### pt-query-digest (Percona Toolkit)

```bash
# Cài đặt
yum install percona-toolkit

# Phân tích và group by fingerprint
pt-query-digest slow.log > report.txt

# So sánh hai khoảng thời gian
pt-query-digest --since '2024-01-01 09:00:00' \
                --until '2024-01-01 10:00:00' \
                slow.log
```

### pgBadger (PostgreSQL)

```bash
# Cài đặt
pip install pgbadger

# Phân tích PostgreSQL log
pgbadger /var/log/postgresql/postgresql.log -o report.html

# Mở báo cáo: xem top slow queries, lock waits, error rates
```

### MySQL EXPLAIN Visualizer

```sql
-- Format JSON để dùng với visual tools
EXPLAIN FORMAT=JSON SELECT ...;

-- Tools online phân tích JSON output:
-- - explain.depesz.com (PostgreSQL)
-- - mysqlexplain.com (MySQL)
```

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Q: Quy trình của bạn khi phát hiện một slow query trong production?

```
Quy trình chẩn đoán từng bước:

Bước 1 — Xác nhận vấn đề:
- CloudWatch: CPU cao? Latency tăng? Từ khi nào?
- Performance Insights: Query nào đang consume nhiều load?

Bước 2 — Thu thập query text:
- Slow query log: lấy full query text (không phải digest)
- Ghi lại execution plan hiện tại: EXPLAIN ANALYZE

Bước 3 — Phân tích execution plan:
- type=ALL? → Missing index
- Using filesort? → Missing composite index
- Rows examined cao? → Index kém selective hoặc query kém

Bước 4 — Đề xuất fix:
- Thêm index trên columns trong WHERE/JOIN/ORDER BY
- Rewrite query để dùng index tốt hơn
- Tách query phức tạp thành nhiều query đơn giản

Bước 5 — Kiểm tra an toàn:
- Tạo index với CREATE INDEX CONCURRENTLY (PostgreSQL)
  hoặc Online DDL (MySQL InnoDB) để tránh lock
- Test trên read replica trước
- Deploy vào production trong giờ thấp điểm
- Verify với EXPLAIN sau khi thêm index
```

### Q: Làm thế nào để thêm index mà không lock bảng production?

```
MySQL (InnoDB):
- Từ MySQL 5.6+: Online DDL không lock bảng cho CREATE INDEX
- Dùng ALGORITHM=INPLACE, LOCK=NONE:
  ALTER TABLE orders ADD INDEX idx_status(status),
  ALGORITHM=INPLACE, LOCK=NONE;
- Lưu ý: vẫn có overhead trong quá trình rebuild

PostgreSQL:
- CREATE INDEX CONCURRENTLY idx_name ON table(column);
- Không lock bảng, nhưng:
  + Chậm hơn CREATE INDEX thông thường
  + Có thể fail nếu có conflict — cần DROP và tạo lại
  + Không thể chạy trong transaction block

Aurora:
- MySQL-compatible: dùng ALGORITHM=INPLACE
- Aurora Fast DDL: một số DDL operations nhanh hơn nhiều
- Clone DB → add index → test → swap endpoint
```

### Q: `rows_examined` cao gấp 1000 lần `rows_sent` có ý nghĩa gì?

```
Ý nghĩa:
- Database phải đọc 1000 rows để trả về 1 row
- Chỉ số efficiency rất thấp
- Nguyên nhân thường gặp:
  1. Thiếu index → full table scan
  2. Index kém selective → quét nhiều rows không liên quan
  3. Query không sử dụng composite index đúng thứ tự cột

Ví dụ:
- Bảng 1M rows, query không có index
- rows_examined = 1,000,000
- rows_sent = 10
- Ratio = 100,000:1 → Thêm index để giảm về 10:1

Hành động:
- Chạy EXPLAIN để xem access type
- Thêm index trên cột WHERE/JOIN
- Xem xét composite index nếu nhiều conditions
- Dùng covering index nếu chỉ cần một số cột nhất định
```

---

**Tiếp Theo:** [4-connection-pooling.md](4-connection-pooling.md) — RDS Proxy & Connection Pooling

**Cập Nhật Lần Cuối:** 2026-05-15
