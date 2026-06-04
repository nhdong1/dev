# Autovacuum & Quản Lý Bloat

PostgreSQL dùng MVCC để xử lý concurrency. Hiểu MVCC và autovacuum là chìa khóa tránh bloat.

## MVCC: Tại Sao Cần Vacuum?

```
MVCC (Multi-Version Concurrency Control):

UPDATE users SET name = 'Bob' WHERE id = 1;
→ Không xóa hàng cũ ngay!
→ Tạo phiên bản mới của hàng
→ Hàng cũ được đánh dấu "dead" (chết)

Kết quả: Các transaction khác đang đọc vẫn thấy dữ liệu cũ
         Không cần khóa đọc/ghi → Tốt cho concurrency

Vấn đề: Dead tuples tích lũy → Bloat → Hiệu suất giảm
Giải pháp: VACUUM dọn dẹp dead tuples
```

### Dead Tuples Ảnh Hưởng Thế Nào?

```
Bảng orders: 1,000,000 hàng
Sau 3 tháng UPDATE nhiều: 3,000,000 "hàng" trong file
→ 2,000,000 là dead tuples (bloat)

Hệ quả:
- Index lớn hơn cần thiết (cũng có dead entries)
- Seq Scan chậm (phải đọc qua dead tuples)
- Cache không hiệu quả (dead tuples chiếm space)
- IO tăng
```

---

## VACUUM: Dọn Dẹp Dead Tuples

### Các Lệnh VACUUM

```sql
-- VACUUM thông thường: Dọn dead tuples, không trả space về OS
VACUUM orders;

-- VACUUM ANALYZE: Dọn + Cập nhật thống kê
VACUUM ANALYZE orders;

-- VACUUM FULL: Rewrite bảng hoàn toàn, trả space về OS
-- CẢNH BÁO: Khóa bảng hoàn toàn trong thời gian dài!
VACUUM FULL orders;

-- VACUUM VERBOSE: Hiển thị thông tin chi tiết
VACUUM VERBOSE orders;

-- Output VERBOSE:
-- INFO: vacuuming "public"."orders"
-- INFO: scanned index "idx_orders_user_id" to remove 50000 row versions
-- INFO: removed 50000 row versions in 2500 pages
-- INFO: found 50000 removable, 500000 nonremovable row versions in 25000 out of 50000 pages
```

### Khi Nào Dùng Lệnh Nào?

```
VACUUM orders:
→ Dùng hàng ngày/tự động
→ Không khóa bảng
→ Không trả space về OS (dùng lại cho bảng)

VACUUM ANALYZE orders:
→ Sau import/delete lớn
→ Cần cập nhật stats ngay

VACUUM FULL orders:
→ Khi bloat > 50% VÀ có maintenance window
→ Khóa bảng hoàn toàn!
→ Thay thế: pg_repack (online, không khóa)
```

---

## Autovacuum: Cấu Hình Tự Động

### Cài Đặt Mặc Định

```sql
-- Xem cài đặt hiện tại:
SHOW autovacuum;                        -- on
SHOW autovacuum_vacuum_scale_factor;    -- 0.2 (20%)
SHOW autovacuum_analyze_scale_factor;   -- 0.1 (10%)
SHOW autovacuum_vacuum_threshold;       -- 50 hàng
SHOW autovacuum_analyze_threshold;      -- 50 hàng
SHOW autovacuum_vacuum_cost_limit;      -- 200
SHOW autovacuum_vacuum_cost_delay;      -- 2ms

-- Autovacuum trigger:
-- VACUUM khi: dead_tuples > (n_live_tup * scale_factor + threshold)
-- = dead_tuples > (1,000,000 * 0.2 + 50) = 200,050 dead tuples

-- Với bảng 1M hàng, phải có 200K dead tuples mới trigger!
-- → Mặc định quá conservative cho bảng lớn
```

### Điều Chỉnh Autovacuum Cho Bảng Cụ Thể

```sql
-- Bảng nhỏ, hot (ví dụ: sessions, cache):
ALTER TABLE sessions SET (
    autovacuum_vacuum_scale_factor = 0.01,   -- Trigger ở 1%
    autovacuum_analyze_scale_factor = 0.005, -- Analyze ở 0.5%
    autovacuum_vacuum_threshold = 10         -- Hoặc 10 hàng
);

-- Bảng lớn, ít update (ví dụ: historical data):
ALTER TABLE order_history SET (
    autovacuum_vacuum_scale_factor = 0.05,   -- Trigger ở 5%
    autovacuum_analyze_scale_factor = 0.02
);

-- Bảng lớn, cập nhật nhiều (ví dụ: orders, users):
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_vacuum_cost_limit = 1000,     -- Tích cực hơn
    autovacuum_vacuum_cost_delay = 2         -- ms delay giữa pages
);

-- Xem cài đặt per-table:
SELECT relname, reloptions
FROM pg_class
WHERE reloptions IS NOT NULL;
```

### Cài Đặt Toàn Cục (postgresql.conf)

```ini
# postgresql.conf

# Bật autovacuum
autovacuum = on
autovacuum_max_workers = 3          # Tăng nếu nhiều bảng hot

# Ngưỡng trigger (giảm để vacuum sớm hơn)
autovacuum_vacuum_scale_factor = 0.1    # 10% thay vì 20%
autovacuum_analyze_scale_factor = 0.05  # 5% thay vì 10%

# Chi phí I/O (tăng để vacuum tích cực hơn, nhưng tốn IO hơn)
autovacuum_vacuum_cost_limit = 400      # Tăng từ 200
autovacuum_vacuum_cost_delay = 2ms      # Giảm delay

# Log autovacuum chậm (để monitor):
log_autovacuum_min_duration = 250ms     # Log khi >250ms
```

---

## Phát Hiện Bloat

### Kiểm Tra Bloat Bảng

```sql
-- Cách 1: Dùng pgstattuple extension
CREATE EXTENSION pgstattuple;

SELECT * FROM pgstattuple('orders');
-- tuple_count:      1000000
-- tuple_len:        80000000
-- dead_tuple_count: 200000
-- dead_tuple_len:   16000000
-- free_space:       5000000
-- table_len:        101000000  ← Size thực của file

-- Bloat: (dead_tuple_len + free_space) / table_len = 21%

-- Cách 2: Ước tính từ pg_stat_user_tables
SELECT
    tablename,
    n_live_tup,
    n_dead_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 2) AS dead_pct,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY dead_pct DESC;
```

### Kiểm Tra Index Bloat

```sql
-- Index cũng bị bloat!
SELECT
    indexrelname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;

-- REINDEX để xây dựng lại index:
REINDEX INDEX CONCURRENTLY idx_orders_status;  -- Không khóa (PG 12+)
-- Hoặc:
REINDEX TABLE CONCURRENTLY orders;  -- Rebuild tất cả index
```

### Script Bloat Chi Tiết

```sql
-- Ước tính bloat chi tiết (không cần extension):
WITH constants AS (
    SELECT current_setting('block_size')::numeric AS bs
),
relation_size AS (
    SELECT
        c.relname AS tablename,
        c.oid,
        c.relpages,
        c.reltuples,
        c.relam
    FROM pg_class c
    JOIN pg_namespace n ON n.oid = c.relnamespace
    WHERE n.nspname = 'public'
      AND c.relkind = 'r'
)
SELECT
    tablename,
    pg_size_pretty(pg_relation_size(oid)) AS actual_size,
    ROUND(relpages::numeric, 0) AS pages,
    ROUND(reltuples::numeric, 0) AS est_rows
FROM relation_size
ORDER BY pg_relation_size(oid) DESC
LIMIT 20;
```

---

## Xử Lý Bloat Nặng

### Phương Án 1: VACUUM FULL (Có Downtime)

```sql
-- Chọn thời điểm traffic thấp
-- Kiểm tra kích thước trước:
SELECT pg_size_pretty(pg_relation_size('orders'));  -- 10 GB

-- Chạy VACUUM FULL (KHÓA BẢNG):
VACUUM FULL VERBOSE orders;

-- Kiểm tra sau:
SELECT pg_size_pretty(pg_relation_size('orders'));  -- 6 GB (giảm 40%)
```

### Phương Án 2: pg_repack (Không Downtime)

```bash
# Cài pg_repack:
apt-get install postgresql-15-repack

# Chạy không lock bảng:
pg_repack -d mydb -t orders

# Chạy cho tất cả bảng bloat > 10%:
pg_repack -d mydb --no-order
```

### Phương Án 3: Online Rewrite (Chậm Nhưng An Toàn)

```sql
-- Tạo bảng mới (tránh lock):
CREATE TABLE orders_new (LIKE orders INCLUDING ALL);

-- Copy dữ liệu theo batch:
INSERT INTO orders_new
SELECT * FROM orders WHERE id BETWEEN 1 AND 100000;

INSERT INTO orders_new
SELECT * FROM orders WHERE id BETWEEN 100001 AND 200000;
-- ... tiếp tục

-- Swap (cần maintenance window ngắn):
BEGIN;
ALTER TABLE orders RENAME TO orders_old;
ALTER TABLE orders_new RENAME TO orders;
COMMIT;

-- Xóa bảng cũ sau khi xác nhận:
DROP TABLE orders_old;
```

---

## Transaction ID Wraparound: Mối Nguy Hiểm Lớn

### Giải Thích

```
PostgreSQL dùng 32-bit transaction ID (XID)
Tối đa: 2^31 ≈ 2.1 tỷ transactions
Khi hết: PHẢI vacuum để "đặt lại"

Nếu không vacuum đủ:
→ XID gần mức max
→ PostgreSQL vào "emergency mode"
→ NGỪNG nhận write transactions!
→ Đây là sự cố nghiêm trọng nhất của PostgreSQL
```

### Monitor Nguy Cơ Wraparound

```sql
-- Kiểm tra khoảng cách đến wraparound:
SELECT
    datname,
    age(datfrozenxid) AS xid_age,
    2147483647 - age(datfrozenxid) AS xid_remaining,
    ROUND(100.0 * age(datfrozenxid) / 2147483647, 2) AS pct_used
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- Cảnh báo: xid_age > 1.5 tỷ → Cần vacuum gấp!
-- Nguy hiểm: xid_age > 1.9 tỷ → PostgreSQL sẽ tự ngừng writes

-- Bảng có xid_age cao nhất:
SELECT
    relname,
    age(relfrozenxid) AS xid_age,
    pg_size_pretty(pg_relation_size(oid)) AS size
FROM pg_class
WHERE relkind = 'r'
ORDER BY age(relfrozenxid) DESC
LIMIT 10;
```

### Xử Lý Wraparound Emergency

```sql
-- Nếu xid_age nguy hiểm:
-- 1. Vacuum aggressive (đóng băng XID cũ):
VACUUM FREEZE orders;

-- 2. Tất cả bảng:
VACUUM FREEZE;

-- 3. Cấu hình autovacuum tích cực hơn:
ALTER TABLE orders SET (
    autovacuum_freeze_max_age = 150000000,  -- Giảm từ 200M
    autovacuum_freeze_table_age = 120000000
);
```

---

## Giám Sát Autovacuum

### Xem Autovacuum Đang Chạy

```sql
-- Autovacuum workers đang hoạt động:
SELECT pid, query, state, wait_event_type, wait_event
FROM pg_stat_activity
WHERE query LIKE 'autovacuum:%';

-- Log autovacuum (postgresql.conf):
-- log_autovacuum_min_duration = 1000  # Log khi >1 giây
```

### Bảng Dashboard

```sql
SELECT
    schemaname,
    tablename,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    vacuum_count,
    autovacuum_count,
    n_live_tup,
    n_dead_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

---

## Checklist Autovacuum & Bloat

- [ ] Autovacuum đang bật và chạy (pg_stat_activity)
- [ ] Cài đặt autovacuum phù hợp với workload (không dùng mặc định)
- [ ] log_autovacuum_min_duration được bật để monitor
- [ ] Bloat < 20% cho các bảng chính
- [ ] XID age < 1 tỷ (xa giới hạn wraparound)
- [ ] REINDEX định kỳ cho index bloat cao
- [ ] pg_repack được chuẩn bị cho xử lý bloat không downtime
- [ ] Alert khi xid_age > 1.5 tỷ

---

> **Điểm Mấu Chốt:** Transaction ID wraparound là sự cố catastrophic có thể ngừng toàn bộ database. Monitor XID age hàng ngày và đảm bảo autovacuum đủ tích cực để ngăn chặn.
