# Chiến Lược Caching

Caching giảm tải CSDL và tăng tốc độ phản hồi. Biết khi nào và cách nào để cache hiệu quả.

## Tại Sao Cần Caching?

```
Không có cache:
  Mỗi request → Hit database → Đọc disk/compute → Trả về
  Database: 1000 req/s → Chậm, tốn resource

Với cache:
  95% requests → Hit cache → Trả về ngay (< 1ms)
  5% requests → Database → Cache miss → Trả về (~10ms)
  Database: 50 req/s → Nhẹ nhàng hơn 20x
```

---

## Các Tầng Cache

```
Tầng 1: CPU Cache (L1/L2/L3)
  - Tự động, không control được
  - Microseconds

Tầng 2: OS Page Cache
  - OS cache disk reads trong RAM
  - Milliseconds

Tầng 3: Database Buffer Cache (shared_buffers)
  - PostgreSQL cache data pages
  - Microseconds khi hit, milliseconds khi miss

Tầng 4: Application Cache (Redis/Memcached)
  - Cache kết quả query hoặc objects
  - Microseconds đến milliseconds

Tầng 5: CDN Cache
  - Cache response HTTP (không liên quan DB)
  - Milliseconds từ edge location gần nhất
```

---

## PostgreSQL Buffer Cache

### shared_buffers: Cache Nội Tại

```sql
-- Xem cài đặt hiện tại:
SHOW shared_buffers;  -- Mặc định: 128MB (quá nhỏ!)

-- Khuyến nghị:
-- shared_buffers = 25% RAM
-- Ví dụ: Server 32GB RAM → shared_buffers = 8GB

-- postgresql.conf:
shared_buffers = 8GB

-- effective_cache_size: Gợi ý cho planner về total memory cache
-- Bao gồm cả OS page cache
effective_cache_size = 24GB  -- 75% RAM
```

### Kiểm Tra Cache Hit Rate

```sql
-- Cache hit rate của database:
SELECT
    SUM(blks_hit) AS cache_hits,
    SUM(blks_read) AS disk_reads,
    ROUND(100.0 * SUM(blks_hit) / NULLIF(SUM(blks_hit) + SUM(blks_read), 0), 2) AS hit_rate_pct
FROM pg_stat_database;
-- Mục tiêu: > 99% hit rate

-- Cache hit rate theo từng bảng:
SELECT
    relname AS table_name,
    heap_blks_hit,
    heap_blks_read,
    ROUND(100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0), 2) AS hit_rate_pct
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC
LIMIT 20;
```

### pg_prewarm: Preload Cache Sau Restart

```sql
-- Sau restart, shared_buffers trống → Cold start
-- pg_prewarm load dữ liệu vào buffer cache

CREATE EXTENSION pg_prewarm;

-- Preload bảng quan trọng:
SELECT pg_prewarm('orders');
SELECT pg_prewarm('users');
SELECT pg_prewarm('idx_orders_status');  -- Index cũng được

-- Lưu trạng thái cache để restore sau restart:
-- postgresql.conf:
shared_preload_libraries = 'pg_prewarm'
pg_prewarm.autoprewarm = on
```

---

## Application-Level Cache

### Pattern Cache-Aside (Lazy Loading)

```python
import redis
import json
import hashlib

redis_client = redis.Redis(host='localhost', port=6379)

def get_user(user_id: int):
    cache_key = f"user:{user_id}"

    # 1. Kiểm tra cache
    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)

    # 2. Cache miss: Truy vấn database
    user = db.query("SELECT * FROM users WHERE id = %s", [user_id])

    if user:
        # 3. Lưu vào cache với TTL
        redis_client.setex(
            cache_key,
            3600,  # TTL: 1 giờ
            json.dumps(user)
        )

    return user

def update_user(user_id: int, data: dict):
    db.execute("UPDATE users SET ... WHERE id = %s", [user_id, ...])

    # Invalidate cache
    redis_client.delete(f"user:{user_id}")
```

### Pattern Write-Through

```python
def update_user_writethrough(user_id: int, data: dict):
    # Viết vào cả database VÀ cache cùng lúc

    # 1. Update database
    db.execute("UPDATE users SET name = %s WHERE id = %s",
               [data['name'], user_id])

    # 2. Update cache ngay (không delete)
    user = db.query("SELECT * FROM users WHERE id = %s", [user_id])
    redis_client.setex(f"user:{user_id}", 3600, json.dumps(user))

# Ưu điểm: Không có cache miss sau update
# Nhược điểm: Mỗi write phải viết 2 nơi, chậm hơn một chút
```

### Pattern Write-Behind (Write-Back)

```python
# Viết vào cache trước, database sau (async)
# Dùng khi: Write throughput cao, có thể chịu mất dữ liệu nhỏ

def update_counter_writeback(key: str, increment: int):
    # 1. Cập nhật cache ngay
    redis_client.incrby(key, increment)

    # 2. Queue để flush vào database sau
    redis_client.rpush("db_write_queue",
                       json.dumps({"key": key, "increment": increment}))

# Background worker flush queue vào database mỗi 5 giây
# Rủi ro: Nếu cache chết trước khi flush → Mất dữ liệu
```

---

## Materialized Views: Cache Trong Database

### Tạo Materialized View

```sql
-- Tạo báo cáo tổng hợp được cache trong database:
CREATE MATERIALIZED VIEW mv_daily_revenue AS
SELECT
    DATE_TRUNC('day', created_at) AS day,
    COUNT(*) AS order_count,
    SUM(total) AS revenue,
    AVG(total) AS avg_order_value
FROM orders
WHERE status = 'completed'
GROUP BY DATE_TRUNC('day', created_at)
ORDER BY day DESC;

-- Tạo index trên materialized view:
CREATE INDEX idx_mv_revenue_day ON mv_daily_revenue(day);

-- Truy vấn nhanh:
SELECT * FROM mv_daily_revenue
WHERE day >= CURRENT_DATE - INTERVAL '30 days';
-- Không cần GROUP BY lại toàn bộ bảng orders!
```

### Refresh Materialized View

```sql
-- Refresh blocking (khóa đọc trong khi refresh):
REFRESH MATERIALIZED VIEW mv_daily_revenue;

-- Refresh không khóa (PostgreSQL 9.4+):
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_revenue;
-- Yêu cầu: Phải có UNIQUE index

-- Tạo unique index cho CONCURRENT refresh:
CREATE UNIQUE INDEX idx_mv_revenue_day_unique ON mv_daily_revenue(day);

-- Tự động refresh bằng cron:
-- Thêm vào crontab:
-- 0 * * * * psql -d mydb -c "REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_revenue;"
```

### Chiến Lược Refresh

```sql
-- Trigger-based refresh (phức tạp nhưng realtime):
CREATE OR REPLACE FUNCTION refresh_mv_revenue()
RETURNS trigger AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_revenue;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_refresh_mv_revenue
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH STATEMENT EXECUTE FUNCTION refresh_mv_revenue();

-- Lưu ý: Không dùng trigger-based nếu orders có nhiều writes!
-- → Refresh quá thường xuyên = Hiệu suất xấu

-- Scheduled refresh tốt hơn cho hầu hết trường hợp:
-- Mỗi giờ, mỗi 5 phút, v.v. tùy SLA
```

---

## Query Result Cache

### pg_query_cache (Ý Tưởng)

```sql
-- PostgreSQL không có query result cache tích hợp
-- Nhưng có thể dùng CTE với MATERIALIZED hint:

-- Cache kết quả subquery trong CTE:
WITH expensive_subquery AS MATERIALIZED (
    SELECT user_id, SUM(total) AS total_spent
    FROM orders
    WHERE created_at > NOW() - INTERVAL '30 days'
    GROUP BY user_id
)
SELECT u.*, es.total_spent
FROM users u
JOIN expensive_subquery es ON u.id = es.user_id
WHERE es.total_spent > 1000;

-- MATERIALIZED: Buộc PostgreSQL tính toán CTE một lần,
-- dùng lại kết quả (không merge vào query chính)
```

---

## Các Chiến Lược Cache Theo Use Case

### Cache Cho Read-Heavy Data (Ít Thay Đổi)

```
Ví dụ: Catalog sản phẩm, cài đặt hệ thống, danh sách quốc gia

Chiến lược:
- Cache dài hạn (24 giờ hoặc đến khi thay đổi)
- Preload khi khởi động ứng dụng
- Invalidate khi admin update

TTL: 1 giờ đến 24 giờ
Pattern: Cache-aside với event-based invalidation
```

### Cache Cho Session Data (Per-User)

```
Ví dụ: Giỏ hàng, preferences, quyền truy cập

Chiến lược:
- Cache theo user ID
- TTL ngắn hơn (30 phút - 2 giờ)
- Invalidate khi logout hoặc thay đổi quyền

TTL: 30 phút - 2 giờ
Pattern: Cache-aside
```

### Cache Cho Aggregate/Report Data

```
Ví dụ: Số liệu doanh thu, thống kê, ranking

Chiến lược:
- Materialized view + scheduled refresh
- Hoặc cache Redis với background refresh
- Chấp nhận stale data (dữ liệu cũ vài giây/phút)

TTL: 1 phút - 1 giờ
Pattern: Background refresh, stale-while-revalidate
```

---

## Vấn Đề Cache Phổ Biến

### Cache Stampede (Thundering Herd)

```python
# Vấn đề: TTL hết → Nhiều requests cùng lúc → Database bị overload

# Giải pháp 1: Probabilistic Early Expiration
import random
import math

def get_with_early_expiry(key, ttl, beta=1):
    cached = redis_client.get(key)
    if not cached:
        return None

    remaining_ttl = redis_client.ttl(key)
    # Xác suất refresh sớm tăng khi TTL gần hết
    if -math.log(random.random()) * beta > remaining_ttl:
        return None  # Giả vờ cache miss để refresh sớm

    return cached

# Giải pháp 2: Mutex/Lock
def get_with_mutex(key, fetcher_func, ttl):
    cached = redis_client.get(key)
    if cached:
        return cached

    lock_key = f"lock:{key}"
    if redis_client.set(lock_key, "1", ex=10, nx=True):  # Lấy lock
        try:
            value = fetcher_func()
            redis_client.setex(key, ttl, value)
            return value
        finally:
            redis_client.delete(lock_key)
    else:
        # Không lấy được lock → Chờ và retry
        import time
        time.sleep(0.1)
        return redis_client.get(key)  # Giờ này có thể đã được cache
```

### Cache Invalidation

```python
# Vấn đề: Dữ liệu cũ trong cache sau khi update

# Pattern 1: Tag-based invalidation
def update_product(product_id, category_id, data):
    db.execute("UPDATE products SET ... WHERE id = %s", [product_id])

    # Xóa cache liên quan:
    redis_client.delete(f"product:{product_id}")
    redis_client.delete(f"category:{category_id}:products")
    redis_client.delete(f"featured_products")

# Pattern 2: Versioned cache keys
def get_user(user_id):
    version = redis_client.get(f"user:{user_id}:version") or 1
    cache_key = f"user:{user_id}:v{version}"
    return redis_client.get(cache_key)

def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = %s", [user_id])
    # Tăng version → Cache cũ tự động bị bỏ qua
    redis_client.incr(f"user:{user_id}:version")
```

---

## Monitoring Cache

```sql
-- Cache hit rate PostgreSQL:
SELECT
    SUM(blks_hit) * 100.0 / NULLIF(SUM(blks_hit + blks_read), 0) AS buffer_cache_hit_pct
FROM pg_stat_database;

-- Redis: Dùng redis-cli
-- redis-cli INFO stats | grep hit
-- keyspace_hits: 1234567
-- keyspace_misses: 12345
-- Hit rate = hits / (hits + misses) * 100

-- Mục tiêu:
-- PostgreSQL buffer cache: > 99%
-- Redis: > 95% (tùy use case)
```

---

## Checklist Caching

- [ ] shared_buffers = 25% RAM được cài đặt
- [ ] effective_cache_size = 75% RAM được cài đặt
- [ ] Buffer cache hit rate > 99% (kiểm tra pg_stat_database)
- [ ] Redis/Memcached được cài đặt cho application cache
- [ ] Materialized views cho báo cáo nặng
- [ ] REFRESH MATERIALIZED VIEW CONCURRENTLY có unique index
- [ ] Cache invalidation strategy được định nghĩa rõ ràng
- [ ] TTL phù hợp với yêu cầu freshness của từng loại data
- [ ] Monitor cache hit rate và alert khi giảm
- [ ] Xử lý cache stampede cho data hot

---

> **Điểm Mấu Chốt:** Caching hiệu quả nhất ở đúng tầng. Database buffer cache cho hot data, Redis cho query results và session data, Materialized views cho complex aggregations. Đừng cache tất cả — chỉ cache những gì thực sự cần.
