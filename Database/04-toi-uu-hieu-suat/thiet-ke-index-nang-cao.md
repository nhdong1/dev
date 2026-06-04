# Thiết Kế Index Nâng Cao

Chọn và tạo index đúng cách để tối đa hóa hiệu suất truy vấn.

## Tại Sao Index Quan Trọng?

```
Không có index:
  Tìm user theo email → Quét toàn bộ 10 triệu hàng
  Thời gian: O(n) → Hàng giây

Với B-Tree index:
  Tìm user theo email → Đi theo cây nhị phân cân bằng
  Thời gian: O(log n) → Vài millisecond
```

---

## Các Loại Index PostgreSQL

### B-Tree (Mặc Định)

```sql
-- Dùng cho: =, <, >, <=, >=, BETWEEN, LIKE 'abc%'
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_created ON orders(created_at);

-- Hỗ trợ sort (tránh Sort node trong query plan)
CREATE INDEX idx_orders_created_desc ON orders(created_at DESC);

-- Kiểm tra dùng index cho sort:
EXPLAIN SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;
-- Index Scan Backward → Tốt, không cần Sort node
```

### Hash

```sql
-- Dùng cho: Chỉ = (không hỗ trợ range, sort)
CREATE INDEX idx_users_email_hash ON users USING hash(email);

-- Hash nhanh hơn B-Tree cho exact match nhưng:
-- - Không hỗ trợ range queries
-- - Không hỗ trợ LIKE
-- - Hiếm khi dùng trong thực tế
```

### GIN (Generalized Inverted Index)

```sql
-- Dùng cho: Tìm kiếm full-text, array, JSONB
CREATE INDEX idx_posts_content ON posts USING gin(to_tsvector('english', content));

-- Tìm kiếm full-text:
SELECT * FROM posts
WHERE to_tsvector('english', content) @@ to_tsquery('postgresql & index');

-- Array containment:
CREATE INDEX idx_products_tags ON products USING gin(tags);
SELECT * FROM products WHERE tags @> ARRAY['electronics', 'sale'];

-- JSONB:
CREATE INDEX idx_users_data ON users USING gin(data);
SELECT * FROM users WHERE data @> '{"role": "admin"}';
```

### GiST (Generalized Search Tree)

```sql
-- Dùng cho: Dữ liệu hình học, phạm vi (range types)
CREATE INDEX idx_locations ON places USING gist(location);

-- Tìm kiếm địa lý (với PostGIS):
SELECT * FROM places
WHERE ST_DWithin(location, ST_MakePoint(-73.97, 40.77), 1000);

-- Range types:
CREATE INDEX idx_reservations_period ON reservations USING gist(during);
SELECT * FROM reservations WHERE during && '[2026-01-01, 2026-01-07)';
```

### BRIN (Block Range Index)

```sql
-- Dùng cho: Cột có tương quan vật lý với thứ tự lưu trữ
-- Kích thước rất nhỏ, phù hợp bảng rất lớn
CREATE INDEX idx_logs_created ON logs USING brin(created_at);

-- Tốt nhất khi: Dữ liệu được INSERT theo thứ tự thời gian
-- Không tốt: Dữ liệu phân tán ngẫu nhiên
```

---

## Nguyên Tắc Composite Index

### Thứ Tự Cột Quan Trọng

```sql
-- Quy tắc: Cột equality trước, range sau
-- Truy vấn: WHERE status = 'active' AND created_at > '2026-01-01'

-- Sai: Range trước, equality sau
CREATE INDEX idx_bad ON orders(created_at, status);
-- Không tận dụng index cho status = 'active'

-- Đúng: Equality trước, range sau
CREATE INDEX idx_good ON orders(status, created_at);
-- Bước 1: Tìm status = 'active' (equality = nhanh)
-- Bước 2: Trong kết quả đó, lọc created_at (range)

-- Kiểm tra:
EXPLAIN SELECT * FROM orders
WHERE status = 'active' AND created_at > '2026-01-01';
-- Phải thấy: Index Scan using idx_good
```

### Quy Tắc Leading Column

```sql
-- Composite index (a, b, c) hỗ trợ:
-- WHERE a = ?              ✓
-- WHERE a = ? AND b = ?    ✓
-- WHERE a = ? AND b = ? AND c = ?  ✓
-- WHERE a = ? AND c = ?    ✓ (bỏ qua b)
-- WHERE b = ?              ✗ (không dùng được)
-- WHERE b = ? AND c = ?    ✗ (không dùng được)

-- Ví dụ:
CREATE INDEX idx_orders ON orders(user_id, status, created_at);

-- Dùng được:
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 1 AND status = 'active';
SELECT * FROM orders WHERE user_id = 1 AND status = 'active' AND created_at > '2026-01-01';

-- Không dùng được (index):
SELECT * FROM orders WHERE status = 'active';  -- Thiếu user_id
SELECT * FROM orders WHERE created_at > '2026-01-01';  -- Thiếu user_id
```

### Nhiều Điều Kiện Equality

```sql
-- Thứ tự cột có selectivity cao lên trước
-- Selectivity = % hàng khác nhau (cao = tốt hơn)

-- users: 1M hàng, status có 3 giá trị, country có 50 giá trị
-- country có selectivity cao hơn → Đặt trước

CREATE INDEX idx_users_country_status ON users(country, status);
-- Khi query: WHERE country = 'VN' AND status = 'active'
-- Lọc bằng country trước (từ 1M → 20K), rồi status (20K → 5K)
```

---

## Partial Index

```sql
-- Index chỉ một phần dữ liệu → Nhỏ hơn, nhanh hơn

-- Ví dụ: 95% users là 'active', chỉ cần tìm kiếm active users
CREATE INDEX idx_users_active_email ON users(email)
WHERE status = 'active';

-- Kết quả: Index nhỏ hơn 20x (chỉ chứa 5% active users)

-- Truy vấn dùng được:
SELECT * FROM users WHERE email = 'test@example.com' AND status = 'active';

-- Truy vấn KHÔNG dùng được:
SELECT * FROM users WHERE email = 'test@example.com';  -- Thiếu predicate

-- Ứng dụng thực tế:
-- Orders chưa xử lý (pending/processing là thiểu số):
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status IN ('pending', 'processing');

-- Null values (ít khi NULL):
CREATE INDEX idx_users_deleted ON users(deleted_at)
WHERE deleted_at IS NOT NULL;
```

---

## Covering Index (Index-Only Scan)

```sql
-- Mục tiêu: Query không cần truy cập bảng chính (heap)

-- Truy vấn: Chỉ cần user_id, status, total
SELECT user_id, status, total FROM orders WHERE user_id = 1;

-- Index thông thường:
CREATE INDEX idx_orders_user ON orders(user_id);
-- Phải đọc index, rồi đọc bảng cho status và total

-- Covering index (PostgreSQL 11+):
CREATE INDEX idx_orders_covering ON orders(user_id)
INCLUDE (status, total);
-- Chỉ đọc index, không cần đọc bảng!

-- Kiểm tra:
EXPLAIN SELECT user_id, status, total FROM orders WHERE user_id = 1;
-- Phải thấy: Index Only Scan
-- Heap Fetches: 0  ← Không đọc heap!

-- Lưu ý: INCLUDE columns không được dùng trong WHERE,
-- chỉ thêm để tránh heap lookup
```

---

## Functional Index

```sql
-- Index trên expression, không phải raw column

-- Vấn đề: Case-insensitive search
-- Xấu: Không dùng index
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';

-- Tạo functional index:
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Bây giờ dùng được:
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';
-- → Index Scan using idx_users_email_lower

-- Các ví dụ khác:
-- Index trên year:
CREATE INDEX idx_orders_year ON orders(EXTRACT(YEAR FROM created_at));

-- Index trên JSONB field:
CREATE INDEX idx_users_city ON users((data->>'city'));

-- Index trên hàm tùy chỉnh:
CREATE INDEX idx_products_slug ON products(generate_slug(name));
```

---

## Quản Lý Index

### Tìm Index Không Dùng

```sql
-- Index tốn bộ nhớ và làm chậm INSERT/UPDATE/DELETE
-- Index không dùng = lãng phí

SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- Chưa bao giờ được dùng
  AND indexrelname NOT LIKE '%pkey'  -- Bỏ qua primary key
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Tìm Index Bị Trùng

```sql
-- Index trùng lặp: Lãng phí storage và performance
SELECT
    t.tablename,
    i1.indexname AS index1,
    i2.indexname AS index2,
    i1.indexdef
FROM pg_indexes i1
JOIN pg_indexes i2 ON i1.tablename = i2.tablename
    AND i1.indexname < i2.indexname
    AND i1.indexdef = i2.indexdef
JOIN pg_tables t ON i1.tablename = t.tablename
WHERE t.schemaname = 'public';
```

### Tìm Bảng Thiếu Index

```sql
-- Bảng bị seq scan nhiều = Có thể thiếu index
SELECT
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_stat_user_tables
WHERE seq_scan > 1000  -- Nhiều seq scan
ORDER BY seq_tup_read DESC
LIMIT 10;
```

### Tạo Index Không Khóa Bảng

```sql
-- Thông thường CREATE INDEX khóa bảng
-- Dùng CONCURRENTLY để tạo không khóa:

CREATE INDEX CONCURRENTLY idx_orders_status ON orders(status);

-- Lưu ý: CONCURRENTLY chậm hơn, không thể dùng trong transaction
-- Nên dùng trong production để tránh downtime
```

---

## Chiến Lược Index Cho Từng Loại Query

### Pagination

```sql
-- Phân trang thông thường:
CREATE INDEX idx_orders_id ON orders(id);
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 0;  -- Trang 1

-- Keyset pagination (tốt hơn với offset lớn):
CREATE INDEX idx_orders_id ON orders(id);
SELECT * FROM orders WHERE id > :last_id ORDER BY id LIMIT 20;
```

### Filter + Sort + Limit

```sql
-- Pattern phổ biến: Lọc → Sort → Limit
-- Truy vấn: Pending orders, mới nhất trước, 20 cái

-- Index tối ưu: (filter_col, sort_col DESC)
CREATE INDEX idx_orders_status_created
ON orders(status, created_at DESC);

-- Query:
SELECT * FROM orders
WHERE status = 'pending'
ORDER BY created_at DESC
LIMIT 20;

-- Kết quả: PostgreSQL đọc đúng 20 hàng từ index, không cần sort!
```

### Search + Filter

```sql
-- Full-text search kết hợp filter:
CREATE INDEX idx_posts_search ON posts
USING gin(to_tsvector('english', title || ' ' || content));

CREATE INDEX idx_posts_published ON posts(published_at)
WHERE is_published = true;

-- Query:
SELECT * FROM posts
WHERE to_tsvector('english', title || ' ' || content)
      @@ to_tsquery('postgresql')
  AND is_published = true
ORDER BY published_at DESC
LIMIT 10;
```

---

## Checklist Thiết Kế Index

- [ ] Xác định truy vấn quan trọng nhất (từ pg_stat_statements)
- [ ] Phân tích EXPLAIN ANALYZE trước khi tạo index
- [ ] Equality columns trước, range sau trong composite index
- [ ] Cân nhắc covering index cho query chỉ cần ít columns
- [ ] Dùng partial index cho dữ liệu có skew cao
- [ ] Dùng CONCURRENTLY khi tạo index trong production
- [ ] Xóa index không dùng định kỳ
- [ ] Đo lại hiệu suất sau khi tạo index
- [ ] Monitor index bloat (REINDEX khi cần)

---

> **Điểm Mấu Chốt:** Index không miễn phí — mỗi index làm chậm INSERT/UPDATE/DELETE. Chỉ tạo index khi có bằng chứng từ EXPLAIN ANALYZE rằng nó cần thiết.
