# Phân Tích Truy Vấn với EXPLAIN ANALYZE

Hiểu cách PostgreSQL thực thi truy vấn để tìm và loại bỏ bottleneck hiệu suất.

## Tại Sao Phân Tích Truy Vấn Quan Trọng?

```
Vấn đề phổ biến:
- Truy vấn chạy nhanh trong dev nhưng chậm trong production
- Dữ liệu tăng → truy vấn tăng thời gian theo cấp số nhân
- Developer không biết tại sao query chậm

Giải pháp: EXPLAIN ANALYZE cho thấy CHÍNH XÁC:
- PostgreSQL đọc dữ liệu theo cách nào
- Mất bao lâu ở mỗi bước
- Ước tính hàng có chính xác không
```

---

## Công Cụ Chính: EXPLAIN ANALYZE

### Cú Pháp Cơ Bản

```sql
-- EXPLAIN: Chỉ hiển thị kế hoạch (không thực thi)
EXPLAIN SELECT * FROM orders WHERE user_id = 1;

-- EXPLAIN ANALYZE: Thực thi và hiển thị thời gian thực tế
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 1;

-- EXPLAIN ANALYZE BUFFERS: Thêm thông tin bộ nhớ đệm
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id = 1;

-- EXPLAIN ANALYZE FORMAT JSON: Dễ parse hơn
EXPLAIN (ANALYZE, FORMAT JSON) SELECT * FROM orders WHERE user_id = 1;
```

### Đọc Output EXPLAIN ANALYZE

```
QUERY PLAN
-----------
Nested Loop  (cost=0.43..12.50 rows=5 width=100)
             (actual time=0.050..0.075 rows=3 loops=1)
  ->  Index Scan using idx_orders_user on orders
        (cost=0.43..8.45 rows=5 width=80)
        (actual time=0.040..0.060 rows=3 loops=1)
        Index Cond: (user_id = 1)
  ->  Index Scan using users_pkey on users
        (cost=0.29..0.81 rows=1 width=20)
        (actual time=0.003..0.004 rows=1 loops=3)
        Index Cond: (id = orders.user_id)
Planning Time: 0.150 ms
Execution Time: 0.120 ms
```

**Giải thích từng phần:**

```
cost=0.43..12.50
  ↑           ↑
  Startup    Total
  cost       cost

rows=5        → Ước tính số hàng (từ planner)
width=100     → Bytes trung bình mỗi hàng

actual time=0.050..0.075
            ↑         ↑
         First row  Last row (ms)

rows=3        → Hàng THỰC TẾ trả về
loops=1       → Số lần node được thực thi
```

---

## Các Node Quan Trọng

### Seq Scan — NGUY HIỂM

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending';

-- Output xấu:
Seq Scan on orders  (cost=0.00..5420.00 rows=150 width=80)
                    (actual time=0.020..85.400 rows=150 loops=1)
  Filter: ((status)::text = 'pending'::text)
  Rows Removed by Filter: 99850

-- Đọc 100,000 hàng chỉ để lấy 150!
-- → Cần index trên cột status
```

### Index Scan — TỐT

```sql
-- Sau khi tạo index:
CREATE INDEX idx_orders_status ON orders(status);

EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending';

-- Output tốt:
Index Scan using idx_orders_status on orders
  (cost=0.42..185.00 rows=150 width=80)
  (actual time=0.050..1.200 rows=150 loops=1)
  Index Cond: ((status)::text = 'pending'::text)

-- Chỉ đọc 150 hàng!
```

### Index Only Scan — TỐT NHẤT

```sql
-- Covering index: query không cần truy cập bảng
CREATE INDEX idx_orders_covering ON orders(user_id) INCLUDE (status, total);

EXPLAIN ANALYZE
SELECT user_id, status, total FROM orders WHERE user_id = 1;

-- Output tốt nhất:
Index Only Scan using idx_orders_covering on orders
  (actual time=0.020..0.050 rows=5 loops=1)
  Index Cond: (user_id = 1)
  Heap Fetches: 0  ← Không truy cập bảng chính!
```

### Bitmap Scan — THƯỜNG TỐT

```sql
-- Cho kết quả trung bình (vài trăm đến vài nghìn hàng)
Bitmap Heap Scan on orders
  (actual time=1.200..5.400 rows=2500 loops=1)
  Recheck Cond: (status = 'active')
  ->  Bitmap Index Scan on idx_orders_status
        (actual time=0.900..0.900 rows=2500 loops=1)
        Index Cond: (status = 'active')

-- Tốt khi nhiều hàng cần đọc ngẫu nhiên
```

### Hash Join vs Nested Loop vs Merge Join

```sql
-- Hash Join: Tốt cho JOIN lớn, không có index
Hash Join  (cost=1500.00..3000.00 rows=50000)
  Hash Cond: (orders.user_id = users.id)
  ->  Seq Scan on orders
  ->  Hash
        ->  Seq Scan on users

-- Nested Loop: Tốt khi outer nhỏ, inner có index
Nested Loop  (cost=0.43..50.00 rows=10)
  ->  Index Scan on orders (user_id = 1)
  ->  Index Scan on users (id = orders.user_id)

-- Merge Join: Tốt khi cả hai đã được sắp xếp
Merge Join  (cost=500.00..1000.00)
  Merge Cond: (orders.user_id = users.id)
  ->  Index Scan on orders (user_id)
  ->  Index Scan on users (id)
```

---

## Dấu Hiệu Cảnh Báo Trong EXPLAIN ANALYZE

### 1. Ước Tính Hàng Sai Lệch Lớn

```sql
-- rows=5 (ước tính) vs rows=50000 (thực tế) = VẤN ĐỀ LỚN!
Seq Scan on orders
  (cost=0.00..5000.00 rows=5 width=80)
  (actual time=0.020..850.00 rows=50000 loops=1)

-- Nguyên nhân: Thống kê cũ
-- Giải pháp:
ANALYZE orders;

-- Hoặc tăng target thống kê:
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ANALYZE orders;
```

### 2. Filter Loại Bỏ Nhiều Hàng

```sql
-- Xấu: Đọc nhiều, lấy ít
Seq Scan on users
  Filter: (status = 'active')
  Rows Removed by Filter: 95000  ← Đọc 95000 hàng thừa!

-- Giải pháp: Partial index
CREATE INDEX idx_users_active ON users(id) WHERE status = 'active';
```

### 3. Sort Không Dùng Index

```sql
-- Xấu: Sort trong memory/disk
Sort  (actual time=150.00..180.00 rows=50000)
  Sort Key: created_at DESC
  Sort Method: external merge  Disk: 8192kB  ← Đang sort trên đĩa!

-- Giải pháp: Index hỗ trợ sort
CREATE INDEX idx_orders_created ON orders(created_at DESC);
```

### 4. Hash Batch > 1 (Spill to Disk)

```sql
Hash  (actual time=200.00..200.00 rows=100000)
  Buckets: 131072  Batches: 4  Memory Usage: 32768kB
  ↑                ↑
                   Batches > 1 = Đang dùng đĩa!

-- Giải pháp:
SET work_mem = '64MB';  -- Tăng memory cho sort/hash

-- Hoặc thêm vào postgresql.conf:
-- work_mem = 64MB
```

---

## pg_stat_statements: Tìm Truy Vấn Chậm

### Thiết Lập

```sql
-- Bật extension:
CREATE EXTENSION pg_stat_statements;

-- postgresql.conf:
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 10000
pg_stat_statements.track = all
```

### Truy Vấn Phân Tích

```sql
-- Top 10 truy vấn tốn thời gian nhất:
SELECT
    query,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS mean_ms,
    ROUND(total_exec_time::numeric, 2) AS total_ms,
    ROUND((100 * total_exec_time / SUM(total_exec_time) OVER())::numeric, 2) AS pct_total,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Truy vấn được gọi nhiều nhất:
SELECT query, calls, mean_exec_time
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- Truy vấn có độ lệch chuẩn cao (không nhất quán):
SELECT
    query,
    calls,
    mean_exec_time,
    stddev_exec_time,
    ROUND((stddev_exec_time / mean_exec_time * 100)::numeric, 2) AS cv_pct
FROM pg_stat_statements
WHERE calls > 100
ORDER BY stddev_exec_time DESC
LIMIT 10;
```

### Reset Thống Kê

```sql
-- Reset tất cả thống kê:
SELECT pg_stat_statements_reset();

-- Reset định kỳ (hàng ngày) để track xu hướng:
-- Thêm vào cron job
```

---

## Workflow Phân Tích Hoàn Chỉnh

### Bước 1: Xác Định Truy Vấn Vấn Đề

```sql
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
WHERE mean_exec_time > 1000  -- Chậm hơn 1 giây
ORDER BY mean_exec_time DESC;
```

### Bước 2: Lấy Kế Hoạch Chi Tiết

```sql
-- Luôn dùng BUFFERS để xem cache hit:
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, o.total, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending'
  AND o.created_at > NOW() - INTERVAL '7 days'
ORDER BY o.created_at DESC
LIMIT 100;
```

### Bước 3: Xác Định Bottleneck

```
Checklist đọc EXPLAIN ANALYZE:

□ Node nào tốn nhiều thời gian nhất?
□ Có Seq Scan trên bảng lớn không?
□ Ước tính hàng có chênh lệch lớn với thực tế không?
□ Có Sort/Hash spill to disk không?
□ Rows Removed by Filter có cao không?
□ Buffer hits vs reads (reads cao = cache miss)
□ Có Hash Batches > 1 không?
```

### Bước 4: Tối Ưu và Đo Lại

```sql
-- Trước tối ưu: ghi lại thời gian
-- Thêm index / viết lại truy vấn / tăng work_mem

-- Sau tối ưu: so sánh
-- Chênh lệch phải đáng kể (>20%) mới có giá trị
```

---

## Ví Dụ Thực Tế: Tối Ưu Truy Vấn Báo Cáo

### Trước Tối Ưu

```sql
-- Truy vấn báo cáo doanh thu theo tháng:
EXPLAIN ANALYZE
SELECT
    DATE_TRUNC('month', created_at) AS month,
    SUM(total) AS revenue,
    COUNT(*) AS order_count
FROM orders
WHERE status = 'completed'
  AND created_at >= '2025-01-01'
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY month;

-- Kết quả:
Sort (actual time=2500.00..2500.10 rows=12)
  ->  HashAggregate (actual time=2450.00..2450.50 rows=12)
        ->  Seq Scan on orders (actual time=0.050..1800.00 rows=500000)
              Filter: status = 'completed' AND created_at >= '2025-01-01'
              Rows Removed by Filter: 200000
-- Tổng: ~2.5 giây
```

### Sau Tối Ưu

```sql
-- Thêm composite index:
CREATE INDEX idx_orders_status_created
ON orders(status, created_at)
INCLUDE (total);

-- Chạy lại:
Sort (actual time=45.00..45.10 rows=12)
  ->  HashAggregate (actual time=40.00..40.50 rows=12)
        ->  Index Only Scan using idx_orders_status_created
              (actual time=0.050..25.00 rows=500000)
              Index Cond: (status = 'completed' AND created_at >= '2025-01-01')
              Heap Fetches: 0
-- Tổng: ~45ms (cải thiện 55x!)
```

---

## Checklist Phân Tích Truy Vấn

- [ ] Bật pg_stat_statements
- [ ] Xác định top truy vấn chậm hàng ngày
- [ ] Dùng EXPLAIN (ANALYZE, BUFFERS) để phân tích
- [ ] Kiểm tra Seq Scan trên bảng lớn
- [ ] Kiểm tra chênh lệch ước tính vs thực tế
- [ ] Kiểm tra Sort/Hash spill to disk
- [ ] Đo trước và sau khi tối ưu
- [ ] Tài liệu hóa cải thiện đạt được

---

> **Điểm Mấu Chốt:** EXPLAIN ANALYZE là công cụ thiết yếu nhất của DBA. Đọc kế hoạch truy vấn thường xuyên — đừng đợi có vấn đề mới nhìn vào.
