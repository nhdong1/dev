# Khóa & Deadlock

Quản lý tranh chấp khóa và ngăn deadlock trong môi trường concurrent.

## Tại Sao Khóa Quan Trọng?

```
Concurrent transactions cần khóa để:
- Đảm bảo tính nhất quán dữ liệu
- Ngăn các thao tác xung đột

Vấn đề:
- Khóa quá nhiều → Hệ thống chờ nhau → Chậm
- Khóa vòng tròn → Deadlock → Transaction thất bại
```

---

## Các Loại Khóa PostgreSQL

### Lock Modes (Từ Nhẹ Đến Nặng)

```sql
-- Thứ tự từ ít conflict đến nhiều conflict:

-- 1. ACCESS SHARE (SELECT):
-- → Chỉ conflict với ACCESS EXCLUSIVE
-- → Nhiều transaction có thể giữ cùng lúc
SELECT * FROM orders;  -- Lấy ACCESS SHARE lock

-- 2. ROW SHARE (SELECT FOR UPDATE/SHARE):
SELECT * FROM orders WHERE id = 1 FOR UPDATE;  -- ROW SHARE

-- 3. ROW EXCLUSIVE (INSERT/UPDATE/DELETE):
UPDATE orders SET status = 'completed' WHERE id = 1;  -- ROW EXCLUSIVE

-- 4. SHARE UPDATE EXCLUSIVE:
-- VACUUM, CREATE INDEX CONCURRENTLY
VACUUM orders;

-- 5. SHARE:
-- CREATE INDEX (không concurrent)
CREATE INDEX idx_orders_status ON orders(status);

-- 6. SHARE ROW EXCLUSIVE:
-- Hiếm dùng trực tiếp

-- 7. EXCLUSIVE:
-- Rất hiếm

-- 8. ACCESS EXCLUSIVE (DDL):
-- → Conflict với TẤT CẢ locks khác!
-- → Không có SELECT nào có thể chạy cùng lúc!
ALTER TABLE orders ADD COLUMN note TEXT;  -- ACCESS EXCLUSIVE!
```

### Row-Level Locks

```sql
-- SELECT FOR UPDATE: Khóa hàng để update
BEGIN;
SELECT * FROM orders WHERE id = 1 FOR UPDATE;
-- Hàng id=1 bị khóa, transaction khác phải chờ
UPDATE orders SET status = 'processing' WHERE id = 1;
COMMIT;

-- SELECT FOR SHARE: Khóa chia sẻ (cho phép read, ngăn write)
SELECT * FROM orders WHERE id = 1 FOR SHARE;

-- SELECT FOR NO KEY UPDATE: Nhẹ hơn FOR UPDATE
-- Cho phép SELECT FOR KEY SHARE chạy song song
SELECT * FROM orders WHERE id = 1 FOR NO KEY UPDATE;

-- SELECT FOR UPDATE SKIP LOCKED: Bỏ qua hàng đang bị khóa
-- → Dùng cho queue processing
SELECT * FROM tasks
WHERE status = 'pending'
ORDER BY created_at
LIMIT 10
FOR UPDATE SKIP LOCKED;
-- Nhiều workers có thể chạy song song mà không xung đột!
```

---

## Xác Định Tranh Chấp Khóa

### Xem Locks Hiện Tại

```sql
-- Tất cả locks đang được giữ:
SELECT
    pg_locks.pid,
    pg_locks.mode,
    pg_locks.granted,
    pg_class.relname AS table_name,
    pg_stat_activity.query,
    pg_stat_activity.state,
    pg_stat_activity.query_start,
    NOW() - pg_stat_activity.query_start AS duration
FROM pg_locks
LEFT JOIN pg_class ON pg_locks.relation = pg_class.oid
LEFT JOIN pg_stat_activity ON pg_locks.pid = pg_stat_activity.pid
WHERE pg_class.relname IS NOT NULL
ORDER BY duration DESC NULLS LAST;
```

### Tìm Lock Waits

```sql
-- Transactions đang chờ lock:
SELECT
    blocked.pid AS blocked_pid,
    blocked_activity.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking_activity.query AS blocking_query,
    NOW() - blocked_activity.query_start AS wait_duration
FROM pg_locks blocked
JOIN pg_stat_activity blocked_activity ON blocked.pid = blocked_activity.pid
JOIN pg_locks blocking ON blocking.relation = blocked.relation
    AND blocking.locktype = blocked.locktype
    AND blocking.pid != blocked.pid
    AND blocking.granted = true
    AND blocked.granted = false
JOIN pg_stat_activity blocking_activity ON blocking.pid = blocking_activity.pid
ORDER BY wait_duration DESC;
```

### Query Đang Chạy Lâu

```sql
-- Tìm queries chạy lâu (có thể gây bloat cho lock waits):
SELECT
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    query_start,
    NOW() - query_start AS duration,
    LEFT(query, 100) AS query_snippet
FROM pg_stat_activity
WHERE state != 'idle'
  AND query_start < NOW() - INTERVAL '1 minute'
ORDER BY query_start ASC;

-- Giết query cụ thể (soft kill):
SELECT pg_cancel_backend(12345);  -- Gửi SIGINT, query tự kết thúc

-- Giết connection (hard kill):
SELECT pg_terminate_backend(12345);  -- Gửi SIGTERM
```

---

## Deadlock

### Deadlock Là Gì?

```
Transaction A giữ Lock trên bảng orders
Transaction B giữ Lock trên bảng users

Transaction A cần Lock trên bảng users → Chờ B
Transaction B cần Lock trên bảng orders → Chờ A

= Deadlock! Vòng chờ không bao giờ kết thúc

PostgreSQL phát hiện deadlock, chọn một transaction làm victim:
→ Victim bị rollback với lỗi: "ERROR: deadlock detected"
→ Transaction còn lại tiếp tục
```

### Ví Dụ Deadlock

```sql
-- Session 1:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Lock id=1

-- Session 2:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 2;  -- Lock id=2

-- Session 1: Tiếp tục...
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Chờ Session 2!

-- Session 2: Tiếp tục...
UPDATE accounts SET balance = balance + 100 WHERE id = 1;  -- Chờ Session 1!

-- DEADLOCK! PostgreSQL phát hiện và rollback một bên:
-- ERROR: deadlock detected
-- DETAIL: Process 1234 waits for ShareLock on transaction 5678;
--         blocked by process 5678.
--         Process 5678 waits for ShareLock on transaction 1234;
--         blocked by process 1234.
-- HINT: See server log for query details.
```

### Phát Hiện Deadlock

```sql
-- Bật log deadlock (postgresql.conf):
log_min_messages = warning
log_lock_waits = on
deadlock_timeout = 1s  -- Log khi chờ > 1 giây

-- Tìm deadlock trong log:
-- grep "deadlock" /var/log/postgresql/postgresql.log

-- Hoặc dùng pg_locks để phát hiện trước:
-- Chạy định kỳ để tìm vòng chờ
WITH RECURSIVE lock_graph AS (
    SELECT
        blocked.pid AS blocked_pid,
        blocking.pid AS blocking_pid
    FROM pg_locks blocked
    JOIN pg_locks blocking ON blocking.relation = blocked.relation
        AND blocking.locktype = blocked.locktype
        AND blocking.pid != blocked.pid
        AND blocking.granted = true
        AND blocked.granted = false
)
SELECT * FROM lock_graph;
```

---

## Chiến Lược Ngăn Deadlock

### 1. Khóa Theo Cùng Thứ Tự

```sql
-- Quy tắc: Luôn truy cập bảng/hàng theo cùng thứ tự

-- Xấu: Thứ tự ngẫu nhiên
-- Transaction A: Lock orders → Lock users
-- Transaction B: Lock users → Lock orders → DEADLOCK!

-- Tốt: Thứ tự cố định (theo alphabetical hoặc dependency)
-- Transaction A: Lock users → Lock orders
-- Transaction B: Lock users → Lock orders → OK (chờ A xong)

-- Trong code, sort IDs trước khi lock:
-- Xấu:
UPDATE accounts SET balance = balance - 100 WHERE id = from_id;
UPDATE accounts SET balance = balance + 100 WHERE id = to_id;

-- Tốt: Lock hàng có ID nhỏ hơn trước
-- Đảm bảo mọi transaction đều lock theo thứ tự ID tăng dần
WITH sorted AS (
    SELECT LEAST(1, 2) AS first_id, GREATEST(1, 2) AS second_id
)
UPDATE accounts SET balance =
    CASE id
        WHEN 1 THEN balance - 100
        WHEN 2 THEN balance + 100
    END
WHERE id IN (1, 2);
-- Single UPDATE = Single lock acquisition!
```

### 2. Giữ Transaction Ngắn

```sql
-- Xấu: Transaction dài
BEGIN;
SELECT * FROM orders WHERE status = 'pending';  -- Lock
-- ... Xử lý phức tạp trong ứng dụng (5 giây) ...
UPDATE orders SET status = 'processing' WHERE id = :id;  -- Vẫn giữ lock!
COMMIT;

-- Tốt: Tách business logic ra ngoài transaction
-- 1. Đọc dữ liệu (không transaction hoặc transaction ngắn)
SELECT * FROM orders WHERE status = 'pending';

-- 2. Xử lý trong ứng dụng

-- 3. Transaction ngắn chỉ để update
BEGIN;
UPDATE orders SET status = 'processing' WHERE id = :id AND status = 'pending';
COMMIT;
-- Transaction chỉ tồn tại vài millisecond!
```

### 3. Dùng NOWAIT Để Fail Fast

```sql
-- Thay vì chờ mãi, fail ngay nếu không lấy được lock:
BEGIN;
SELECT * FROM orders WHERE id = 1 FOR UPDATE NOWAIT;
-- Nếu bị lock → Lỗi ngay: "ERROR: could not obtain lock on row in relation orders"
-- Ứng dụng có thể retry sau

-- Hoặc timeout ngắn:
SET lock_timeout = '100ms';
SELECT * FROM orders WHERE id = 1 FOR UPDATE;
-- Lỗi sau 100ms: "ERROR: canceling statement due to lock timeout"
```

### 4. SKIP LOCKED Cho Queue Processing

```sql
-- Pattern để xử lý queue không bị deadlock:
-- Nhiều workers lấy task khác nhau mà không xung đột

-- Worker 1:
BEGIN;
SELECT id, payload
FROM tasks
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;  -- Bỏ qua task đang bị worker khác xử lý

UPDATE tasks SET status = 'processing' WHERE id = :taken_id;
COMMIT;

-- Worker 2 chạy đồng thời:
BEGIN;
SELECT id, payload
FROM tasks
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;  -- Lấy task khác (không phải task Worker 1 đang giữ)
```

### 5. Giảm Lock Footprint với Index

```sql
-- Index phù hợp → PostgreSQL khóa ít hàng hơn

-- Xấu: Không có index → Table lock hoặc nhiều row lock
UPDATE orders SET status = 'archived'
WHERE created_at < '2024-01-01';  -- Không có index → Scan nhiều hàng

-- Tốt: Index → Chỉ lock hàng cần thiết
CREATE INDEX idx_orders_created ON orders(created_at);
UPDATE orders SET status = 'archived'
WHERE created_at < '2024-01-01';  -- Dùng index, lock ít hàng hơn
```

---

## Lock Monitoring Dashboard

```sql
-- View tổng hợp về lock health:
CREATE OR REPLACE VIEW lock_health AS
SELECT
    -- Số transactions đang chờ lock
    COUNT(*) FILTER (WHERE NOT granted) AS waiting_locks,
    -- Thời gian chờ lâu nhất
    MAX(EXTRACT(EPOCH FROM (NOW() - query_start))) FILTER (WHERE NOT granted) AS max_wait_secs,
    -- Số blocked transactions > 5 giây
    COUNT(*) FILTER (WHERE NOT granted AND query_start < NOW() - INTERVAL '5 seconds') AS long_waits
FROM pg_locks
JOIN pg_stat_activity USING (pid);

SELECT * FROM lock_health;
```

---

## Cài Đặt Lock PostgreSQL

```sql
-- postgresql.conf:

-- Thời gian chờ deadlock detection (thấp hơn = phát hiện nhanh hơn)
deadlock_timeout = 1s

-- Log khi chờ lock > threshold
log_lock_waits = on

-- Timeout cho statement (tránh lock lâu):
statement_timeout = '30s'  -- Giết query sau 30 giây

-- Timeout cho lock wait:
lock_timeout = '10s'  -- Fail nếu chờ lock > 10 giây

-- Idle transaction timeout:
idle_in_transaction_session_timeout = '60s'  -- Đóng idle transactions
```

---

## Checklist Khóa & Deadlock

- [ ] log_lock_waits = on để monitor
- [ ] deadlock_timeout cài đặt phù hợp (1s)
- [ ] statement_timeout ngăn queries chạy quá lâu
- [ ] idle_in_transaction_session_timeout ngăn transactions treo
- [ ] Application code truy cập bảng theo thứ tự nhất quán
- [ ] Transactions được giữ ngắn nhất có thể
- [ ] SKIP LOCKED cho queue processing
- [ ] Monitor lock waits hàng ngày
- [ ] Alert khi wait_duration > 5 giây

---

> **Điểm Mấu Chốt:** Deadlock không thể tránh hoàn toàn, nhưng có thể giảm thiểu bằng cách: (1) Truy cập resource theo thứ tự nhất quán, (2) Giữ transaction ngắn, (3) Dùng index phù hợp để giảm lock footprint.
