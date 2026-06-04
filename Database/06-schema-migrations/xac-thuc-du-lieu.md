# Xác Thực Dữ Liệu Sau Migration

Đảm bảo migration hoàn thành đúng và dữ liệu toàn vẹn sau mỗi thay đổi schema.

## Tại Sao Xác Thực Quan Trọng?

```
Migration thành công ≠ Dữ liệu đúng

Ví dụ:
- Migration chạy xong nhưng backfill chỉ update 99.9% hàng
- Constraint thêm vào nhưng 0.01% hàng vi phạm (NOT VALID)
- Index tạo xong nhưng partial, không dùng được
- Số hàng không khớp (mất data trong quá trình migrate)

Xác thực = Bằng chứng rằng migration đúng
```

---

## Các Loại Kiểm Tra Xác Thực

### 1. Row Count Verification

```sql
-- Ghi lại số hàng TRƯỚC migration:
CREATE TABLE migration_checkpoints (
    id SERIAL PRIMARY KEY,
    migration_version TEXT NOT NULL,
    table_name TEXT NOT NULL,
    row_count BIGINT NOT NULL,
    checked_at TIMESTAMPTZ DEFAULT NOW(),
    phase TEXT NOT NULL  -- 'before', 'after'
);

-- Trước migration:
INSERT INTO migration_checkpoints (migration_version, table_name, row_count, phase)
SELECT 'V10', relname, n_live_tup, 'before'
FROM pg_stat_user_tables
WHERE relname IN ('users', 'orders', 'payments');

-- Sau migration:
INSERT INTO migration_checkpoints (migration_version, table_name, row_count, phase)
SELECT 'V10', relname, n_live_tup, 'after'
FROM pg_stat_user_tables
WHERE relname IN ('users', 'orders', 'payments');

-- So sánh:
SELECT
    before.table_name,
    before.row_count AS before_count,
    after.row_count AS after_count,
    after.row_count - before.row_count AS diff
FROM migration_checkpoints before
JOIN migration_checkpoints after
    ON before.table_name = after.table_name
    AND before.migration_version = after.migration_version
WHERE before.phase = 'before' AND after.phase = 'after'
  AND before.migration_version = 'V10';

-- Kỳ vọng: diff = 0 (không mất hàng)
-- Trừ khi migration có DELETE (diff phải bằng số hàng đã xóa)
```

### 2. Backfill Completeness

```sql
-- Sau khi backfill, kiểm tra không còn NULL:
SELECT
    COUNT(*) AS total_rows,
    COUNT(status) AS rows_with_status,
    COUNT(*) - COUNT(status) AS null_count
FROM users;
-- null_count phải = 0

-- Kiểm tra giá trị hợp lệ:
SELECT status, COUNT(*) AS count
FROM users
GROUP BY status
ORDER BY count DESC;
-- Phải có đúng các giá trị kỳ vọng

-- Không có giá trị ngoài danh sách cho phép:
SELECT *
FROM users
WHERE status NOT IN ('active', 'inactive', 'deleted', 'dormant')
  AND status IS NOT NULL;
-- Phải trả về 0 hàng
```

### 3. Referential Integrity

```sql
-- Kiểm tra FK sau khi thêm FK constraint:
-- Tìm orphan records:
SELECT o.id, o.user_id
FROM orders o
LEFT JOIN users u ON o.user_id = u.id
WHERE u.id IS NULL;
-- Phải trả về 0 hàng

-- Nếu có orphans trước khi thêm FK:
SELECT COUNT(*)
FROM orders o
WHERE NOT EXISTS (SELECT 1 FROM users u WHERE u.id = o.user_id);

-- Xử lý orphans trước khi thêm FK:
-- Option 1: Xóa orphans
DELETE FROM orders WHERE user_id NOT IN (SELECT id FROM users);

-- Option 2: Tạo placeholder user
INSERT INTO users (id, name, email)
SELECT DISTINCT o.user_id, 'Deleted User', 'deleted_' || o.user_id || '@invalid'
FROM orders o
WHERE o.user_id NOT IN (SELECT id FROM users);
```

### 4. Index Validity

```sql
-- Kiểm tra index đã tạo đúng:
SELECT
    indexname,
    tablename,
    indexdef,
    -- Kiểm tra index không bị invalid (xảy ra khi CONCURRENTLY bị interrupt)
    indisvalid
FROM pg_indexes
JOIN pg_index ON indexrelid = (
    SELECT oid FROM pg_class WHERE relname = indexname
)
WHERE tablename = 'orders';
-- indisvalid phải là TRUE

-- Kiểm tra index thực sự được dùng:
EXPLAIN SELECT * FROM orders WHERE status = 'pending';
-- Phải thấy: Index Scan using idx_orders_status

-- Test performance:
EXPLAIN ANALYZE SELECT * FROM orders
WHERE user_id = 1 AND status = 'active';
-- So sánh với baseline trước migration
```

### 5. Constraint Validation

```sql
-- Kiểm tra constraint đã được validate:
SELECT
    conname AS constraint_name,
    contype AS constraint_type,
    convalidated AS is_validated
FROM pg_constraint
WHERE conrelid = 'users'::regclass;
-- is_validated phải là TRUE (nếu false → chưa validate)

-- Kiểm tra CHECK constraint:
SELECT COUNT(*)
FROM users
WHERE NOT (status IN ('active', 'inactive', 'deleted'));
-- Phải là 0

-- Kiểm tra UNIQUE constraint:
SELECT email, COUNT(*) AS count
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
-- Phải trả về 0 hàng
```

---

## Script Xác Thực Tổng Hợp

```sql
-- Chạy sau mỗi migration để verify:
CREATE OR REPLACE PROCEDURE verify_migration_v10()
LANGUAGE plpgsql AS $$
DECLARE
    check_result BOOLEAN;
    error_messages TEXT[] := '{}';
BEGIN
    -- Check 1: Cột status tồn tại
    check_result := EXISTS (
        SELECT 1 FROM information_schema.columns
        WHERE table_name = 'users' AND column_name = 'status'
    );
    IF NOT check_result THEN
        error_messages := error_messages || 'ERROR: Column status not found on users';
    END IF;

    -- Check 2: Không còn NULL trong status
    SELECT COUNT(*) = 0 INTO check_result
    FROM users WHERE status IS NULL;
    IF NOT check_result THEN
        error_messages := error_messages ||
            ('ERROR: Found ' || (SELECT COUNT(*) FROM users WHERE status IS NULL) || ' NULL status values');
    END IF;

    -- Check 3: Index tồn tại và valid
    check_result := EXISTS (
        SELECT 1 FROM pg_indexes
        WHERE tablename = 'users' AND indexname = 'idx_users_status'
    );
    IF NOT check_result THEN
        error_messages := error_messages || 'ERROR: Index idx_users_status not found';
    END IF;

    -- Check 4: Row count không thay đổi
    check_result := (
        SELECT row_count FROM migration_checkpoints
        WHERE table_name = 'users' AND phase = 'before' AND migration_version = 'V10'
    ) = (SELECT COUNT(*) FROM users);
    IF NOT check_result THEN
        error_messages := error_messages || 'ERROR: Row count mismatch for users table';
    END IF;

    -- Báo cáo kết quả:
    IF array_length(error_messages, 1) > 0 THEN
        RAISE EXCEPTION 'Migration V10 verification FAILED: %', array_to_string(error_messages, '; ');
    ELSE
        RAISE NOTICE 'Migration V10 verification PASSED';
    END IF;
END;
$$;

CALL verify_migration_v10();
```

---

## Checksum Verification

```sql
-- Tạo checksum của dữ liệu để verify không bị corrupt:
-- Trước migration:
SELECT
    MD5(string_agg(
        id::text || email || COALESCE(name, ''),
        ',' ORDER BY id
    )) AS users_checksum
FROM users;
-- Lưu giá trị này

-- Sau migration:
SELECT
    MD5(string_agg(
        id::text || email || COALESCE(name, ''),
        ',' ORDER BY id
    )) AS users_checksum
FROM users;
-- So sánh với giá trị trước — phải khớp nếu migration chỉ thêm cột
```

---

## sqitch Verify Step

```sql
-- sqitch tích hợp verify step natively:
-- verify/add_status_to_users.sql:

-- Verify column exists:
SELECT status FROM users WHERE FALSE;

-- Verify no NULLs:
DO $$
BEGIN
    IF EXISTS (SELECT 1 FROM users WHERE status IS NULL) THEN
        RAISE EXCEPTION 'Verification failed: NULL status values found';
    END IF;
END;
$$;

-- Verify index exists:
DO $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM pg_indexes
        WHERE tablename = 'users' AND indexname = 'idx_users_status'
    ) THEN
        RAISE EXCEPTION 'Verification failed: idx_users_status not found';
    END IF;
END;
$$;

-- sqitch chạy verify sau mỗi deploy:
-- sqitch deploy --verify db:pg://localhost/mydb
```

---

## Post-Migration Smoke Tests

```python
# Python smoke tests sau migration:
import psycopg2
import pytest

@pytest.fixture
def db_conn():
    conn = psycopg2.connect("postgresql://localhost/mydb")
    yield conn
    conn.close()

def test_users_status_column_exists(db_conn):
    with db_conn.cursor() as cur:
        cur.execute("""
            SELECT column_name FROM information_schema.columns
            WHERE table_name = 'users' AND column_name = 'status'
        """)
        assert cur.fetchone() is not None, "Column 'status' not found"

def test_no_null_status(db_conn):
    with db_conn.cursor() as cur:
        cur.execute("SELECT COUNT(*) FROM users WHERE status IS NULL")
        count = cur.fetchone()[0]
        assert count == 0, f"Found {count} NULL status values"

def test_status_values_valid(db_conn):
    with db_conn.cursor() as cur:
        cur.execute("""
            SELECT COUNT(*) FROM users
            WHERE status NOT IN ('active', 'inactive', 'deleted')
        """)
        count = cur.fetchone()[0]
        assert count == 0, f"Found {count} invalid status values"

def test_index_exists(db_conn):
    with db_conn.cursor() as cur:
        cur.execute("""
            SELECT 1 FROM pg_indexes
            WHERE tablename = 'users' AND indexname = 'idx_users_status'
        """)
        assert cur.fetchone() is not None, "Index idx_users_status not found"

def test_query_uses_index(db_conn):
    with db_conn.cursor() as cur:
        cur.execute("EXPLAIN SELECT * FROM users WHERE status = 'active'")
        plan = cur.fetchall()
        plan_text = str(plan)
        assert 'Index' in plan_text, "Query not using index for status filter"
```

---

## Báo Cáo Xác Thực

```sql
-- Tạo báo cáo migration health:
SELECT
    migration_version,
    table_name,
    SUM(CASE WHEN phase = 'before' THEN row_count END) AS rows_before,
    SUM(CASE WHEN phase = 'after' THEN row_count END) AS rows_after,
    SUM(CASE WHEN phase = 'after' THEN row_count END) -
    SUM(CASE WHEN phase = 'before' THEN row_count END) AS row_diff,
    CASE
        WHEN SUM(CASE WHEN phase = 'after' THEN row_count END) =
             SUM(CASE WHEN phase = 'before' THEN row_count END)
        THEN 'OK'
        ELSE 'WARNING: Row count changed'
    END AS status
FROM migration_checkpoints
GROUP BY migration_version, table_name
ORDER BY migration_version, table_name;
```

---

## Checklist Xác Thực

- [ ] Row count ghi lại trước migration
- [ ] Row count so sánh sau migration (không mất hàng)
- [ ] NULL check cho cột mới (backfill hoàn chỉnh)
- [ ] Giá trị hợp lệ (không có giá trị lạ)
- [ ] Referential integrity (không có orphan records)
- [ ] Index valid (indisvalid = true)
- [ ] Index được dùng (EXPLAIN ANALYZE)
- [ ] Constraint validated (convalidated = true)
- [ ] Smoke tests tự động chạy sau migration
- [ ] sqitch verify hoặc custom verification script

---

> **Điểm Mấu Chốt:** Migration chưa xong cho đến khi xác thực thành công. Tự động hóa các bước xác thực trong CI/CD pipeline để không bao giờ bỏ sót.
