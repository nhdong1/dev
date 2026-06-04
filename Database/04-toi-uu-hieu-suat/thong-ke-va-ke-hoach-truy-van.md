# Thống Kê & Kế Hoạch Truy Vấn

Query planner của PostgreSQL ra quyết định dựa trên thống kê. Thống kê lỗi thời = Kế hoạch tệ.

## Query Planner Hoạt Động Như Thế Nào?

```
Bạn viết SQL → Parser → Rewriter → Planner → Executor

                                      ↑
                             Planner dùng thống kê để:
                             1. Ước tính số hàng mỗi bước
                             2. Chọn Join method (hash/nested loop/merge)
                             3. Chọn Scan method (seq/index)
                             4. Quyết định thứ tự join
                             5. Chọn index nào dùng
```

---

## Thống Kê PostgreSQL

### Xem Thống Kê Bảng

```sql
-- Thông tin thống kê cơ bản:
SELECT
    tablename,
    n_live_tup AS live_rows,      -- Hàng sống
    n_dead_tup AS dead_rows,      -- Hàng chết (cần vacuum)
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;
```

### Xem Thống Kê Cột

```sql
-- Thống kê chi tiết từng cột (pg_stats):
SELECT
    attname AS column_name,
    n_distinct,           -- Số giá trị phân biệt (-1 = unique)
    null_frac,            -- Tỷ lệ NULL
    avg_width,            -- Bytes trung bình
    most_common_vals,     -- Giá trị phổ biến nhất
    most_common_freqs,    -- Tần suất của most_common_vals
    histogram_bounds      -- Phân bố dữ liệu
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';

-- Ví dụ output:
-- n_distinct: 5 (có 5 giá trị status khác nhau)
-- most_common_vals: {completed,active,pending,cancelled,refunded}
-- most_common_freqs: {0.65,0.20,0.08,0.05,0.02}
-- → 65% orders là 'completed', 20% là 'active'...
```

### Ý Nghĩa n_distinct

```
n_distinct > 0: Số giá trị phân biệt tuyệt đối
               Ví dụ: n_distinct = 5 → Có chính xác 5 giá trị

n_distinct < 0: Tỷ lệ giá trị phân biệt so với tổng hàng
               Ví dụ: n_distinct = -0.5 → 50% hàng có giá trị khác nhau
               n_distinct = -1 → Tất cả hàng có giá trị khác nhau (unique)
```

---

## ANALYZE: Cập Nhật Thống Kê

### Cập Nhật Thủ Công

```sql
-- Cập nhật thống kê tất cả bảng trong schema public:
ANALYZE;

-- Cập nhật một bảng cụ thể:
ANALYZE orders;

-- Cập nhật một cột cụ thể (nhanh hơn):
ANALYZE orders(status, created_at);

-- Xem ANALYZE đang làm gì:
ANALYZE VERBOSE orders;
-- Output: INFO: analyzing "public"."orders"
--         INFO: "orders": scanned 30000 of 100000 pages...
```

### Khi Nào Cần ANALYZE Thủ Công?

```sql
-- 1. Sau khi import dữ liệu lớn:
COPY orders FROM '/data/orders_import.csv' CSV HEADER;
ANALYZE orders;  -- Cập nhật ngay, không chờ autovacuum

-- 2. Sau khi DELETE/UPDATE lớn:
DELETE FROM orders WHERE created_at < '2024-01-01';
ANALYZE orders;

-- 3. Sau khi tạo index mới:
CREATE INDEX idx_orders_status ON orders(status);
ANALYZE orders;  -- Đảm bảo planner biết về index mới

-- 4. Khi query plan đột nhiên xấu đi:
-- → Kiểm tra last_analyze
-- → Nếu cũ → ANALYZE
```

---

## Statistics Target: Tăng Độ Chính Xác

### Vấn Đề Với Default Target

```sql
-- Mặc định: statistics target = 100
-- Planner sample khoảng 300 * 100 = 30,000 hàng
-- Cho bảng 100M hàng, sample 30K có thể không đủ

-- Triệu chứng:
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 12345;
-- rows=5000 (ước tính) vs rows=50 (thực tế) → Sai lệch 100x!
```

### Tăng Statistics Target

```sql
-- Tăng target cho cột cụ thể:
ALTER TABLE orders ALTER COLUMN user_id SET STATISTICS 500;
ANALYZE orders;

-- Tăng target toàn cục (postgresql.conf):
default_statistics_target = 200  -- Mặc định 100

-- Tăng mạnh cho cột có phân bố bất thường:
ALTER TABLE orders ALTER COLUMN customer_segment SET STATISTICS 1000;

-- Kiểm tra target hiện tại:
SELECT attname, attstattarget
FROM pg_attribute
WHERE attrelid = 'orders'::regclass
  AND attstattarget != -1;
-- attstattarget = -1 → Dùng default
```

### Đánh Đổi

```
Target cao hơn:
✓ Ước tính chính xác hơn
✓ Kế hoạch truy vấn tốt hơn
✗ ANALYZE chậm hơn
✗ Tốn thêm bộ nhớ cho pg_stats

Khuyến nghị:
- Default (100) cho hầu hết cột
- 200-500 cho cột JOIN thường xuyên
- 500-1000 cho cột có phân bố rất skewed
```

---

## Thống Kê Mở Rộng (PostgreSQL 10+)

### Vấn Đề: Tương Quan Giữa Các Cột

```sql
-- Bảng products: category VÀ subcategory có tương quan
-- category = 'Electronics' → subcategory chỉ có thể là
--   'Phones', 'Laptops', 'TVs' (không phải 'Shirts')

-- Planner không biết điều này → Ước tính sai:
EXPLAIN SELECT * FROM products
WHERE category = 'Electronics' AND subcategory = 'Phones';

-- Planner ước tính: P(Electronics) * P(Phones)
-- = 0.3 * 0.1 = 0.03 → 3% = 30,000 hàng
-- Thực tế: 5,000 hàng → Sai!
```

### Tạo Extended Statistics

```sql
-- Thống kê đa biến (biết về tương quan):
CREATE STATISTICS stat_products_cat_subcat
ON category, subcategory
FROM products;

ANALYZE products;

-- Kiểm tra extended stats:
SELECT stxname, stxkeys, stxdndistinct, stxddependencies
FROM pg_statistic_ext
JOIN pg_statistic_ext_data ON oid = stxoid
WHERE stxname = 'stat_products_cat_subcat';

-- Bây giờ planner ước tính chính xác hơn
```

### MCV Statistics (Most Common Values)

```sql
-- Thêm MCV cho multi-column:
CREATE STATISTICS stat_orders_status_user
(mcv) ON status, user_id
FROM orders;

ANALYZE orders;

-- Planner biết: user_id=1 luôn có status='active'
-- → Ước tính chính xác hơn khi kết hợp cả hai điều kiện
```

---

## Kế Hoạch Truy Vấn: Ảnh Hưởng Planner Settings

### Join Strategy Settings

```sql
-- Tắt/bật các phương thức join (để debug):
SET enable_hashjoin = off;
SET enable_mergejoin = off;
SET enable_nestloop = on;

-- Kiểm tra:
EXPLAIN SELECT * FROM orders o JOIN users u ON o.user_id = u.id;

-- Khôi phục:
RESET enable_hashjoin;
RESET enable_mergejoin;
RESET enable_nestloop;
```

### Scan Strategy Settings

```sql
-- Tắt seq scan để ép dùng index (chỉ để debug!):
SET enable_seqscan = off;
EXPLAIN SELECT * FROM orders WHERE status = 'active';
-- → Bây giờ dùng index dù không hiệu quả

-- Khôi phục:
RESET enable_seqscan;
```

### Cost Parameters

```sql
-- Các hệ số chi phí ảnh hưởng quyết định planner:
SHOW random_page_cost;    -- Mặc định: 4.0 (HDD)
SHOW seq_page_cost;       -- Mặc định: 1.0

-- Cho SSD: Giảm random_page_cost để planner ưa dùng index
ALTER SYSTEM SET random_page_cost = 1.1;
SELECT pg_reload_conf();

-- Cho SSD NVMe:
ALTER SYSTEM SET random_page_cost = 1.0;  -- Ngang với seq scan

-- Hiệu ứng: Planner ưu tiên index scan hơn seq scan
```

### JIT Compilation

```sql
-- JIT: Biên dịch truy vấn phức tạp → Tăng tốc OLAP
SHOW jit;  -- on/off

-- Ngưỡng bật JIT:
SHOW jit_above_cost;            -- Mặc định: 100000
SHOW jit_inline_above_cost;     -- Mặc định: 500000
SHOW jit_optimize_above_cost;   -- Mặc định: 500000

-- JIT tốt cho: Query phức tạp, nhiều tính toán
-- JIT xấu cho: OLTP - overhead khởi động, query ngắn
-- Tắt JIT cho OLTP:
SET jit = off;
```

---

## Parallel Query

```sql
-- PostgreSQL có thể chạy query song song:
SHOW max_parallel_workers_per_gather;  -- Mặc định: 2
SHOW max_parallel_workers;             -- Mặc định: 8

-- Kiểm tra query có chạy song song không:
EXPLAIN ANALYZE
SELECT COUNT(*), SUM(total)
FROM orders
WHERE created_at > '2025-01-01';

-- Tốt:
Finalize Aggregate (actual time=500.00..500.10)
  ->  Gather (actual time=499.00..499.50)
        Workers Planned: 2
        Workers Launched: 2
        ->  Partial Aggregate (actual time=495.00..495.10)
              ->  Parallel Seq Scan on orders

-- Tăng parallel workers cho query nặng:
SET max_parallel_workers_per_gather = 4;
```

---

## Kiểm Tra Sức Khỏe Thống Kê

### Dashboard Giám Sát

```sql
-- Bảng có thống kê cũ (cần ANALYZE):
SELECT
    schemaname,
    tablename,
    last_analyze,
    last_autoanalyze,
    n_live_tup,
    CASE
        WHEN last_analyze IS NULL THEN 'NEVER'
        WHEN last_analyze < NOW() - INTERVAL '3 days' THEN 'STALE'
        ELSE 'OK'
    END AS analyze_status
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;

-- Bảng có tỷ lệ dead tuples cao (cần VACUUM):
SELECT
    tablename,
    n_live_tup,
    n_dead_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY dead_pct DESC;
```

### Script Kiểm Tra Ước Tính Planner

```sql
-- So sánh ước tính planner vs thực tế:
-- (Chạy EXPLAIN ANALYZE và so sánh rows vs actual rows)

-- Ví dụ tự động hóa việc kiểm tra:
DO $$
DECLARE
    estimated FLOAT;
    actual_count BIGINT;
    ratio FLOAT;
BEGIN
    -- Chạy explain để lấy ước tính
    -- (Thực tế cần parse JSON output)
    SELECT COUNT(*) INTO actual_count FROM orders WHERE status = 'pending';
    RAISE NOTICE 'Actual rows: %', actual_count;
END $$;
```

---

## Checklist Thống Kê

- [ ] Autovacuum/autoanalyze đang chạy đúng
- [ ] last_analyze < 1 ngày cho các bảng hot
- [ ] n_dead_tup không quá 10% n_live_tup
- [ ] Statistics target tăng cho cột quan trọng (JOIN, WHERE)
- [ ] Extended statistics cho cột tương quan
- [ ] random_page_cost phù hợp với loại storage (SSD vs HDD)
- [ ] Parallel query được bật và cấu hình đúng
- [ ] ANALYZE sau import/delete lớn

---

> **Điểm Mấu Chốt:** Thống kê chính xác là nền tảng cho kế hoạch truy vấn tốt. Một ANALYZE kịp thời có thể biến một query 10 giây thành 100ms mà không cần thay đổi gì khác.
