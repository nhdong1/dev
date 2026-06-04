# Chiến Lược Index

## Tổng Quan

Index là công cụ chính để tối ưu hóa truy vấn. Chúng đánh đổi hiệu suất ghi để tăng tốc đọc bằng cách duy trì các cấu trúc dữ liệu cho phép tra cứu nhanh.

---

## Cơ Bản Về Index

### Tại Sao Index Quan Trọng?

**Không có Index (Quét toàn bộ bảng):**

```
1 triệu hàng → Kiểm tra mỗi hàng → O(n) = 1,000,000 lần kiểm tra
```

**Có Index (Tìm kiếm B-Tree):**

```
1 triệu hàng → Tìm kiếm nhị phân → O(log n) = 20 lần kiểm tra
```

**Chi Phí Thực Tế:**

```
Bảng 10GB, quét toàn bộ: ~1 giây mỗi truy vấn
Cùng bảng, có index: ~10 milliseconds
```

### Đánh Đổi Index

| Khía cạnh              | Chi phí                           |
| ---------------------- | --------------------------------- |
| **Lưu trữ index**      | 10-20% kích thước bảng            |
| **Chi phí ghi**        | Chậm hơn 5-10% (duy trì index)   |
| **Cải thiện đọc**      | Nhanh hơn 100-1000 lần (điển hình)|
| **Áp lực bộ nhớ**      | Index được cache trong RAM (tốt)  |

---

## Các Loại Index

### 1. B-Tree (Đa Năng) ⭐

**Cấu trúc:** Cây cân bằng, tra cứu log(n)

**Khi nào dùng:**
- ✓ Loại index mặc định
- ✓ Tìm kiếm bằng (id = 5)
- ✓ Tìm kiếm phạm vi (salary BETWEEN 50k AND 100k)
- ✓ Sắp xếp (ORDER BY)
- ✓ Khớp tiền tố (name LIKE 'Nguyen%')

**Ví dụ:**

```sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_created ON orders(created_at);
CREATE INDEX idx_salary_range ON employees(salary);

-- Các truy vấn sử dụng những index này:
SELECT * FROM users WHERE email = 'john@example.com';            -- ✓
SELECT * FROM orders WHERE created_at > '2026-01-01';           -- ✓
SELECT * FROM employees WHERE salary BETWEEN 50000 AND 100000;  -- ✓
SELECT * FROM employees ORDER BY salary;                        -- ✓
```

**Quy Tắc Cột Đầu (Composite Index):**

```sql
CREATE INDEX idx_users_name_email ON users(first_name, last_name, email);

-- ✓ Dùng index
SELECT * FROM users WHERE first_name = 'Nguyen' AND last_name = 'Van A';

-- ✓ Dùng index
SELECT * FROM users WHERE first_name = 'Nguyen';

-- Không dùng index (bỏ qua cột đầu tiên)
SELECT * FROM users WHERE last_name = 'Van A';

-- ✓ Dùng index (có thể bỏ qua cột giữa)
SELECT * FROM users WHERE first_name = 'Nguyen' AND email = 'nguyen@example.com';
```

**Hiệu Suất B-Tree:**

```
Tra cứu:      O(log n)
Chèn:         O(log n)
Xóa:          O(log n)
Quét phạm vi: O(log n + số kết quả)
```

---

### 2. Hash Index

**Cấu trúc:** Bảng hash (rất nhanh cho khớp chính xác)

**Khi nào dùng:**
- ✓ Chỉ khớp bằng (id = 5)
- ✓ Khi cần tra cứu O(1)
- ❌ Không dùng cho phạm vi (KHÔNG dùng cho BETWEEN)
- ❌ Không dùng cho sắp xếp

**Ví dụ:**

```sql
-- PostgreSQL hỗ trợ hash index
CREATE INDEX idx_user_id_hash ON users USING HASH (id);

-- ✓ Rất nhanh
SELECT * FROM users WHERE id = 12345;

-- Không thể dùng hash index (không hỗ trợ phạm vi)
SELECT * FROM users WHERE id BETWEEN 100 AND 200;
```

**Hash vs B-Tree:**

```
Hash:   O(1) tra cứu, nhưng không dùng cho phạm vi
B-Tree: O(log n) tra cứu, hoạt động cho mọi loại truy vấn
```

---

### 3. Bitmap Index

**Cấu trúc:** Lưu trữ sự có mặt/vắng mặt của giá trị (bitmap)

**Khi nào dùng:**
- ✓ Cột có cardinality thấp (ít giá trị phân biệt)
- ✓ Giới tính (Nam/Nữ), Trạng thái (Active/Inactive)
- ✓ Truy vấn phân tích (không phải OLTP)

**Ví dụ:**

```sql
CREATE BITMAP INDEX idx_status ON users(status);

-- ✓ Tốt (cardinality thấp)
SELECT * FROM users WHERE status = 'active' AND region = 'vn';

-- Không tốt (cardinality cao, quá nhiều bitmap)
SELECT * FROM users WHERE email = 'john@example.com';
```

---

### 4. Full-Text Index (Tìm Kiếm Toàn Văn)

**Cấu trúc:** Inverted index (từ → tài liệu)

**Khi nào dùng:**
- ✓ Tìm kiếm văn bản (LIKE '%từ khóa%')
- ✓ Truy vấn ngôn ngữ tự nhiên
- ✓ Search engine

**Ví dụ (PostgreSQL):**

```sql
CREATE INDEX idx_posts_content ON posts USING GIN (
  to_tsvector('english', title || ' ' || content)
);

-- Tìm kiếm văn bản nhanh
SELECT * FROM posts
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'database & performance');
```

---

### 5. Composite Index (Index Nhiều Cột)

**Cấu trúc:** B-Tree trên nhiều cột

**Khi nào dùng:**
- ✓ Truy vấn lọc trên nhiều cột
- ✓ Tuân theo quy tắc cột đầu

**Ví dụ:**

```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- ✓ Dùng index (lọc user_id trước)
SELECT * FROM orders WHERE user_id = 5 AND status = 'pending';

-- ✓ Dùng index (chỉ phần đầu)
SELECT * FROM orders WHERE user_id = 5;

-- Không dùng index (sai thứ tự cột)
SELECT * FROM orders WHERE status = 'pending';
```

---

### 6. Partial Index (Index Có Điều Kiện)

**Cấu trúc:** Chỉ index các hàng khớp với điều kiện

**Khi nào dùng:**
- ✓ Index chỉ hàng đang active
- ✓ Giảm kích thước index
- ✓ WHERE clause phổ biến

**Ví dụ:**

```sql
-- Chỉ index user đang active (tiết kiệm không gian)
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;

-- ✓ Dùng index
SELECT * FROM users WHERE email = 'john@example.com' AND is_active = true;

-- Không dùng index (điều kiện không khớp)
SELECT * FROM users WHERE email = 'john@example.com' AND is_active = false;
```

**Ứng dụng thực tế: Bản ghi bị xóa mềm (soft-deleted)**

```sql
CREATE INDEX idx_orders_active ON orders(user_id, created_at)
  WHERE deleted_at IS NULL;

-- ✓ Nhanh (dùng partial index)
SELECT * FROM orders WHERE user_id = 5 AND deleted_at IS NULL;

-- Chậm (quét toàn bảng cho đơn đã xóa)
SELECT * FROM orders WHERE user_id = 5 AND deleted_at IS NOT NULL;
```

---

### 7. Covering Index (Index-Only Scan)

**Cấu trúc:** Tất cả cột cần trong truy vấn được bao gồm trong index

**Khi nào dùng:**
- ✓ Truy vấn cần: WHERE + ORDER BY + cột SELECT đều trong index
- ✓ Loại bỏ việc tra cứu bảng

**Ví dụ:**

```sql
-- Bao gồm user email trong index trên user_id
CREATE INDEX idx_orders_covering ON orders(user_id, created_at)
INCLUDE (user_email, status);

-- ✓ Index-only scan (không cần tra cứu bảng)
SELECT user_email, status FROM orders
WHERE user_id = 5
ORDER BY created_at DESC;

-- Phải tra cứu bảng (phone_number không trong index)
SELECT user_email, phone_number FROM orders WHERE user_id = 5;
```

---

## Chiến Lược Index

### Chiến Lược 1: Index Cho Mệnh Đề WHERE

```sql
-- Phân tích các truy vấn phổ biến
SELECT * FROM users WHERE email = ? AND status = ?;
SELECT * FROM users WHERE created_at > ?;

-- Tạo index composite
CREATE INDEX idx_users_email_status ON users(email, status);
CREATE INDEX idx_users_created ON users(created_at);
```

### Chiến Lược 2: Index Cho Cột JOIN

```sql
-- Chậm: Quét toàn bảng bên phải
SELECT o.* FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.status = 'active';

-- Giải pháp: Index cả hai phía của join
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_users_id_status ON users(id, status);
```

### Chiến Lược 3: Index Cho ORDER BY / GROUP BY

```sql
-- Chậm: Sắp xếp 1 triệu hàng trong bộ nhớ
SELECT * FROM orders WHERE user_id = 5 ORDER BY created_at DESC;

-- Giải pháp: Index có thể cung cấp dữ liệu đã sắp xếp
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);
```

### Chiến Lược 4: Tránh Over-Indexing

```sql
-- QUÁ NHIỀU INDEX
CREATE INDEX idx1 ON users(email);
CREATE INDEX idx2 ON users(phone);
CREATE INDEX idx3 ON users(username);
CREATE INDEX idx4 ON users(email, phone);
-- Ghi chậm hơn, tốn bộ nhớ, khó bảo trì

-- CÂN BẰNG
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);
-- Thêm chỉ khi cần thiết
```

### Chiến Lược 5: Selectivity Quan Trọng

**Selectivity cao (index tốt):**

```sql
-- Email: hàng triệu giá trị, mỗi giá trị xuất hiện một lần
CREATE INDEX idx_users_email ON users(email);

-- Giới tính: 2-3 giá trị, mỗi giá trị xuất hiện nhiều lần
-- Không đáng index (hoặc dùng bitmap)
CREATE INDEX idx_users_gender ON users(gender);  -- Không tốt!

-- Đơn hàng mỗi user: 0-1000 mỗi user
-- ✓ Tốt để index
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

---

## Tìm & Phân Tích Index

### PostgreSQL

```sql
-- Tất cả index trên một bảng
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'users';

-- Kích thước index
SELECT indexname, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_indexes
JOIN pg_class ON indexname = relname
WHERE tablename = 'orders';

-- Index không được dùng (ứng viên để xóa)
SELECT schemaname, tablename, indexname
FROM pg_indexes
WHERE indexname NOT IN (
  SELECT indexrelname FROM pg_stat_user_indexes WHERE idx_scan > 0
);

-- Thống kê quét index
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

### MySQL

```sql
-- Tất cả index
SELECT TABLE_NAME, INDEX_NAME, COLUMN_NAME, SEQ_IN_INDEX
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'users';

-- Index không được dùng
SELECT * FROM sys.schema_unused_indexes;
```

---

## Các Lỗi Thường Gặp

### 1. Index Không Khớp Truy Vấn

```sql
-- Sai
CREATE INDEX idx_users_name ON users(last_name, first_name);
SELECT * FROM users WHERE first_name = 'Nguyen';  -- Không dùng index

-- Đúng (quy tắc cột đầu)
CREATE INDEX idx_users_name ON users(first_name, last_name);
```

### 2. LIKE Với Wildcard Đầu

```sql
-- Dùng index
SELECT * FROM users WHERE name LIKE 'Nguyen%';

-- Không dùng index (bắt đầu bằng wildcard)
SELECT * FROM users WHERE name LIKE '%Nguyen%';  -- Quét toàn bảng

-- Giải pháp: Dùng full-text search cho wildcard giữa
```

### 3. Hàm Trên Cột Có Index

```sql
-- Không dùng index (có hàm)
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';

-- ✓ Dùng functional index
CREATE INDEX idx_email_lower ON users(LOWER(email));

-- Hoặc tốt hơn: Chuẩn hóa dữ liệu lúc chèn
SELECT * FROM users WHERE email = 'test@example.com';
```

---

## Câu Hỏi Phỏng Vấn

1. **Sự khác biệt giữa B-Tree và Hash index?**
   - Gợi ý: B-Tree cho phạm vi, Hash chỉ cho khớp chính xác

2. **Giải thích quy tắc cột đầu cho composite index**
   - Gợi ý: Thứ tự quan trọng, cột đầu phải có trong WHERE

3. **Khi nào dùng partial index?**
   - Gợi ý: Bản ghi bị xóa mềm, dữ liệu chỉ active

4. **Làm thế nào tìm index không được dùng?**
   - Gợi ý: Truy vấn system tables (pg_stat_user_indexes)

5. **Thiết kế index cho pattern truy vấn này**
   - Gợi ý: Phân tích WHERE, JOIN, ORDER BY, SELECT

---

## Bảng Tham Chiếu Nhanh

| Loại      | Tốc độ   | Phạm vi | Join | Chi phí  |
| --------- | -------- | ------- | ---- | -------- |
| B-Tree    | O(log n) | ✓       | ✓    | Trung bình |
| Hash      | O(1)     | ❌      | ✓    | Thấp     |
| Bitmap    | O(1)     | ✓       | ✓    | Thấp     |
| Full-Text | O(?)     | ✓       | ❌   | Trung bình |
| Partial   | O(log n) | ✓       | ✓    | Thấp     |
| Covering  | O(log n) | ✓       | ✓    | Trung bình+ |
