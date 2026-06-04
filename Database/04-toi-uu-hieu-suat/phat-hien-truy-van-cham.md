# Phát Hiện Truy Vấn Chậm

Xác định và theo dõi các truy vấn gây ra vấn đề hiệu suất trước khi user phàn nàn.

## Chiến Lược Phát Hiện

```
Hai cách tiếp cận:

1. Reactive (Phản ứng):
   User báo cáo chậm → Điều tra → Tìm nguyên nhân → Fix
   Vấn đề: User đã bị ảnh hưởng

2. Proactive (Chủ động):
   Monitor liên tục → Phát hiện sớm → Fix trước khi ảnh hưởng user
   Mục tiêu: Không bao giờ reactive
```

---

## Slow Query Log

### Cấu Hình PostgreSQL

```ini
# postgresql.conf

# Bật slow query log:
log_min_duration_statement = 1000   # Log queries chậm hơn 1000ms (1 giây)

# Điều chỉnh theo môi trường:
# Production: 500ms - 1000ms
# Staging: 200ms - 500ms
# Dev: 100ms

# Format log:
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_checkpoints = on
log_connections = off
log_disconnections = off
log_duration = off      # Đừng log tất cả (chỉ chậm)
log_statement = 'none'  # Đừng log tất cả statements

# Ví dụ log entry:
# 2026-04-26 10:15:30 [1234]: duration: 2500.345 ms  statement: SELECT ...
```

### Đọc Slow Query Log

```bash
# Xem log real-time:
tail -f /var/log/postgresql/postgresql.log | grep "duration:"

# Lọc queries chậm hơn 5 giây:
grep "duration:" /var/log/postgresql/postgresql.log | \
  awk '{if ($3 > 5000) print}' | \
  sort -k3 -rn | head -20

# Parse log để phân tích:
pgbadger /var/log/postgresql/postgresql.log -o report.html
# pgbadger tạo báo cáo HTML đẹp với top queries, statistics
```

---

## pg_stat_statements: Công Cụ Thiết Yếu

### Thiết Lập

```sql
-- postgresql.conf:
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 10000       -- Lưu max 10K queries khác nhau
pg_stat_statements.track = all       -- Track tất cả (all/top/none)
pg_stat_statements.track_io_timing = on  -- Track IO timing

-- Bật extension:
CREATE EXTENSION pg_stat_statements;

-- Kiểm tra:
SELECT * FROM pg_stat_statements LIMIT 1;
```

### Các Truy Vấn Phân Tích Quan Trọng

```sql
-- 1. TOP 10 QUERIES TỐN THỜI GIAN TỔNG CỘNG:
SELECT
    ROUND(total_exec_time::numeric / 1000, 2) AS total_seconds,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    calls,
    ROUND((100 * total_exec_time / SUM(total_exec_time) OVER())::numeric, 2) AS pct,
    LEFT(query, 100) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

```sql
-- 2. TOP 10 QUERIES CÓ THỜI GIAN TRUNG BÌNH CAO:
SELECT
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    ROUND(stddev_exec_time::numeric, 2) AS stddev_ms,
    calls,
    LEFT(query, 100) AS query
FROM pg_stat_statements
WHERE calls > 10  -- Phải được gọi ít nhất 10 lần
ORDER BY mean_exec_time DESC
LIMIT 10;
```

```sql
-- 3. QUERIES ĐỌC NHIỀU BUFFER NHẤT (I/O INTENSIVE):
SELECT
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    calls,
    shared_blks_hit + shared_blks_read AS total_blocks,
    ROUND(100.0 * shared_blks_hit /
        NULLIF(shared_blks_hit + shared_blks_read, 0), 2) AS cache_hit_pct,
    LEFT(query, 100) AS query
FROM pg_stat_statements
WHERE calls > 10
ORDER BY (shared_blks_hit + shared_blks_read) DESC
LIMIT 10;
```

```sql
-- 4. QUERIES CÓ CACHE HIT THẤP (DISK I/O NHIỀU):
SELECT
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    calls,
    shared_blks_hit,
    shared_blks_read,
    ROUND(100.0 * shared_blks_hit /
        NULLIF(shared_blks_hit + shared_blks_read, 0), 2) AS cache_hit_pct,
    LEFT(query, 100) AS query
FROM pg_stat_statements
WHERE calls > 100
  AND shared_blks_read > 1000  -- Đọc nhiều từ disk
ORDER BY cache_hit_pct ASC
LIMIT 10;
```

```sql
-- 5. QUERIES BẤT NHẤT QUÁN (stddev cao):
SELECT
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    ROUND(stddev_exec_time::numeric, 2) AS stddev_ms,
    ROUND(stddev_exec_time / NULLIF(mean_exec_time, 0), 2) AS cv,
    calls,
    LEFT(query, 100) AS query
FROM pg_stat_statements
WHERE calls > 50
  AND mean_exec_time > 100  -- Chỉ quan tâm query > 100ms
ORDER BY cv DESC  -- Coefficient of variation cao = bất nhất quán
LIMIT 10;
```

---

## Auto_explain: Log Execution Plan Tự Động

### Cấu Hình

```ini
# postgresql.conf:
shared_preload_libraries = 'pg_stat_statements,auto_explain'

# Ngưỡng log execution plan:
auto_explain.log_min_duration = 1000   # Plan của queries > 1 giây
auto_explain.log_analyze = on          # Bao gồm thời gian thực tế
auto_explain.log_buffers = on          # Bao gồm buffer info
auto_explain.log_format = text         # text hoặc json

# Ví dụ log:
# 2026-04-26 10:15:30 duration: 2500 ms plan:
# Query Text: SELECT * FROM orders WHERE status = 'pending'
# Seq Scan on orders  (cost=0.00..5000.00 rows=100 width=80)
#                     (actual time=0.02..2450.00 rows=100 loops=1)
#   Filter: (status = 'pending')
#   Rows Removed by Filter: 99900
```

### Bật Cho Session Cụ Thể

```sql
-- Bật auto_explain chỉ cho session hiện tại (không ảnh hưởng production):
LOAD 'auto_explain';
SET auto_explain.log_min_duration = 0;  -- Log tất cả
SET auto_explain.log_analyze = true;

-- Chạy query cần debug:
SELECT * FROM orders WHERE user_id = 1;

-- Kiểm tra log:
-- tail -f /var/log/postgresql/postgresql.log
```

---

## pg_stat_activity: Real-Time Monitoring

```sql
-- Xem tất cả queries đang chạy:
SELECT
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    query_start,
    NOW() - query_start AS duration,
    LEFT(query, 80) AS query_snippet
FROM pg_stat_activity
WHERE state != 'idle'
  AND pid != pg_backend_pid()  -- Trừ query này
ORDER BY query_start ASC;

-- Queries chạy > 30 giây:
SELECT pid, query_start, NOW() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
  AND query_start < NOW() - INTERVAL '30 seconds'
ORDER BY query_start;

-- Idle in transaction (transactions không làm gì nhưng giữ lock):
SELECT pid, xact_start, NOW() - xact_start AS idle_duration, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;
```

---

## Phân Tích Log Với pgbadger

### Cài Đặt

```bash
# Ubuntu/Debian:
apt-get install pgbadger

# Hoặc từ source:
git clone https://github.com/darold/pgbadger.git
cd pgbadger
perl Makefile.PL
make && make install
```

### Tạo Báo Cáo

```bash
# Phân tích log file:
pgbadger /var/log/postgresql/postgresql.log \
  -o report.html \
  --format html \
  --begin "2026-04-26 00:00:00" \
  --end "2026-04-26 23:59:59"

# Incremental report (chạy hàng giờ):
pgbadger /var/log/postgresql/postgresql.log \
  --incremental \
  --outdir /var/www/pgbadger/

# Kết quả:
# - Top slow queries
# - Query count over time
# - Lock waits
# - Connection statistics
# - Errors and warnings
```

---

## Dashboard Monitoring Tích Hợp

### Script Tổng Hợp Hàng Ngày

```sql
-- Chạy hàng ngày để track xu hướng:
SELECT
    DATE_TRUNC('hour', now()) AS hour,
    COUNT(*) AS total_queries,
    ROUND(AVG(mean_exec_time)::numeric, 2) AS avg_ms,
    SUM(calls) AS total_calls,
    ROUND(SUM(total_exec_time)::numeric / 1000, 2) AS total_seconds
FROM pg_stat_statements
GROUP BY 1
ORDER BY 1 DESC
LIMIT 24;
```

### Cài Đặt Alerts

```sql
-- Alert khi có query chậm đột xuất:
-- Chạy mỗi 5 phút (cron job hoặc monitoring system)

SELECT
    LEFT(query, 100) AS query,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms
FROM pg_stat_statements
WHERE mean_exec_time > 5000   -- Chậm hơn 5 giây
  AND calls > 5               -- Được gọi ít nhất 5 lần
ORDER BY mean_exec_time DESC;

-- Nếu có kết quả → Gửi alert (Slack, PagerDuty, email)
```

---

## Workflow Điều Tra Khi Nhận Alert

### Bước 1: Xác Định Query Vấn Đề

```sql
SELECT query, mean_exec_time, calls, total_exec_time
FROM pg_stat_statements
WHERE mean_exec_time > 1000
ORDER BY total_exec_time DESC
LIMIT 5;
```

### Bước 2: Lấy Execution Plan

```sql
-- Thay $1, $2 bằng giá trị thực tế:
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = $1 AND status = $2;
```

### Bước 3: Xác Định Bottleneck

```
Checklist nhanh:
□ Seq Scan trên bảng lớn? → Cần index
□ Ước tính hàng sai? → Cần ANALYZE
□ Sort spill to disk? → Tăng work_mem
□ Hash Batches > 1? → Tăng work_mem
□ Nhiều buffer reads? → Cache miss, cần shared_buffers lớn hơn
□ Lock waits? → Xem pg_locks
```

### Bước 4: Fix và Verify

```sql
-- Sau khi fix, reset stats và monitor:
SELECT pg_stat_statements_reset();

-- Chờ đủ traffic, kiểm tra lại:
SELECT mean_exec_time, calls FROM pg_stat_statements
WHERE query LIKE '%orders%user_id%';
-- mean_exec_time phải giảm đáng kể
```

---

## Checklist Phát Hiện Truy Vấn Chậm

- [ ] pg_stat_statements đã bật với track = all
- [ ] log_min_duration_statement = 1000ms (hoặc phù hợp)
- [ ] auto_explain cấu hình để log plan cho queries chậm
- [ ] Review pg_stat_statements hàng ngày
- [ ] Alert tự động khi mean_exec_time tăng đột biến
- [ ] pgbadger chạy hàng ngày để phân tích log
- [ ] Baseline được thiết lập (biết "bình thường" là gì)
- [ ] Quy trình điều tra được tài liệu hóa và thực hành

---

> **Điểm Mấu Chốt:** Không thể tối ưu cái bạn không đo. pg_stat_statements + slow query log là bộ đôi thiết yếu để biết hệ thống đang thực sự làm gì.
