# Mẫu Expand-Contract (Zero-Downtime Schema Changes)

Thực hiện thay đổi schema phức tạp mà không gây downtime cho hệ thống production.

## Vấn Đề Với Thay Đổi Schema Trực Tiếp

```
Tình huống: Cần đổi tên cột "email" thành "email_address" trong bảng users

Cách ngây thơ:
ALTER TABLE users RENAME COLUMN email TO email_address;
→ Deploy ứng dụng mới dùng "email_address"

Vấn đề:
- Trong quá trình deploy: Code cũ dùng "email" → LỖI!
- Nếu rollback ứng dụng: Code cũ dùng "email" nhưng cột là "email_address" → LỖI!
- Không có zero-downtime!
```

## Mẫu Expand-Contract

```
EXPAND (Mở Rộng):
  Thêm cấu trúc mới mà không xóa cũ
  → Code cũ và mới đều hoạt động

CONTRACT (Thu Hẹp):
  Xóa cấu trúc cũ sau khi code cũ đã không còn
  → Chỉ code mới còn lại

Ba giai đoạn:
1. Expand: Thêm schema mới, giữ cũ
2. Migrate: Chuyển data, dual-write
3. Contract: Xóa schema cũ
```

---

## Ví Dụ 1: Thêm Cột NOT NULL

### Bối Cảnh

```
Cần: Thêm cột "status" với ràng buộc NOT NULL vào bảng users (10M hàng)
Không thể: ALTER TABLE users ADD COLUMN status VARCHAR(50) NOT NULL;
→ Khóa bảng + PostgreSQL phải rewrite toàn bộ bảng!
```

### Giai Đoạn 1: EXPAND

```sql
-- Migration V10__Add_status_column.sql
-- Thêm cột NULLABLE với default (không khóa bảng):
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'active';

-- Thêm index ngay (CONCURRENTLY = không khóa):
CREATE INDEX CONCURRENTLY idx_users_status ON users(status);

-- Kết quả:
-- - Migration xong trong < 1 giây (chỉ thêm cột metadata)
-- - Code cũ: Bỏ qua cột status (tương thích ngược)
-- - Code mới: Có thể đọc/ghi status
-- - Hàng cũ: status = 'active' (từ DEFAULT)
-- - Hàng mới: status được set

-- Deploy ứng dụng v2 (đọc status nhưng không require NOT NULL)
```

### Giai Đoạn 2: MIGRATE

```sql
-- Migration V11__Backfill_user_status.sql
-- Chạy backfill theo batch để không lock bảng:

DO $$
DECLARE
    batch_size INT := 10000;
    last_id BIGINT := 0;
    max_id BIGINT;
    updated INT;
BEGIN
    SELECT MAX(id) INTO max_id FROM users;

    WHILE last_id <= max_id LOOP
        UPDATE users
        SET status = 'active'
        WHERE id BETWEEN last_id + 1 AND last_id + batch_size
          AND status IS NULL;

        GET DIAGNOSTICS updated = ROW_COUNT;
        RAISE NOTICE 'Updated % rows, last_id: %', updated, last_id + batch_size;

        last_id := last_id + batch_size;

        -- Nghỉ ngắn giữa các batch (tránh overload I/O):
        PERFORM pg_sleep(0.1);
    END LOOP;
END;
$$;

-- Kiểm tra không còn NULL:
SELECT COUNT(*) FROM users WHERE status IS NULL;
-- Phải là 0!
```

### Giai Đoạn 3: CONTRACT

```sql
-- Migration V12__Add_status_not_null_constraint.sql
-- Sau khi backfill xong và code cũ không còn:

-- Thêm constraint NOT VALID (nhanh, không check hàng hiện có):
ALTER TABLE users
    ADD CONSTRAINT users_status_not_null
    CHECK (status IS NOT NULL) NOT VALID;

-- Validate riêng (chỉ lấy SHARE UPDATE EXCLUSIVE lock, không block đọc/ghi):
ALTER TABLE users VALIDATE CONSTRAINT users_status_not_null;

-- Cuối cùng: Đặt NOT NULL (giờ rất nhanh vì constraint đã validated):
ALTER TABLE users ALTER COLUMN status SET NOT NULL;

-- Xóa constraint dư:
ALTER TABLE users DROP CONSTRAINT users_status_not_null;
```

---

## Ví Dụ 2: Đổi Tên Cột

### Giai Đoạn 1: EXPAND

```sql
-- Thêm cột mới với tên mới:
ALTER TABLE users ADD COLUMN email_address VARCHAR(255);

-- Copy data ngay (nếu bảng nhỏ):
UPDATE users SET email_address = email;

-- Hoặc copy theo batch (nếu bảng lớn):
-- Dùng script batch tương tự Ví dụ 1
```

### Giai Đoạn 2: DUAL-WRITE

```python
# Code ứng dụng trong giai đoạn chuyển đổi:
# Ghi vào CẢ HAI cột:

def update_user_email(user_id: int, new_email: str):
    db.execute("""
        UPDATE users
        SET email = %s,           -- Cột cũ (cho code cũ)
            email_address = %s    -- Cột mới (cho code mới)
        WHERE id = %s
    """, [new_email, new_email, user_id])

# Đọc từ cột mới, fallback sang cũ:
def get_user_email(user):
    return user.email_address or user.email
```

### Giai Đoạn 3: CONTRACT

```sql
-- Sau khi tất cả code đã dùng email_address:

-- Tạo NOT NULL constraint trên cột mới:
ALTER TABLE users ALTER COLUMN email_address SET NOT NULL;

-- Thêm UNIQUE constraint:
ALTER TABLE users ADD CONSTRAINT users_email_address_unique UNIQUE (email_address);

-- Xóa cột cũ (chỉ khi chắc chắn không còn code dùng):
ALTER TABLE users DROP COLUMN email;

-- Đổi tên index nếu cần:
ALTER INDEX idx_users_email RENAME TO idx_users_email_address;
```

---

## Ví Dụ 3: Đổi Tên Bảng

### Giai Đoạn 1: EXPAND — Tạo Bảng Mới

```sql
-- Tạo bảng mới với tên mới (rỗng):
CREATE TABLE user_accounts (LIKE users INCLUDING ALL);

-- Thêm trigger sync từ bảng cũ sang mới:
CREATE OR REPLACE FUNCTION sync_users_to_user_accounts()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO user_accounts VALUES (NEW.*);
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        UPDATE user_accounts SET ROW = NEW WHERE id = NEW.id;
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        DELETE FROM user_accounts WHERE id = OLD.id;
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_users
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION sync_users_to_user_accounts();
```

### Giai Đoạn 2: BACKFILL + SWITCH READ

```sql
-- Copy dữ liệu hiện có:
INSERT INTO user_accounts
SELECT * FROM users
ON CONFLICT (id) DO NOTHING;

-- Verify số hàng:
SELECT
    (SELECT COUNT(*) FROM users) AS users_count,
    (SELECT COUNT(*) FROM user_accounts) AS user_accounts_count;
-- Phải bằng nhau!
```

```python
# Ứng dụng bắt đầu đọc từ bảng mới:
# Feature flag để chuyển đổi dần:
def get_users():
    if feature_flag.is_enabled('use_user_accounts_table'):
        return db.query("SELECT * FROM user_accounts")
    return db.query("SELECT * FROM users")
```

### Giai Đoạn 3: CONTRACT

```sql
-- Sau khi 100% traffic dùng bảng mới:
-- Xóa trigger sync:
DROP TRIGGER trg_sync_users ON users;
DROP FUNCTION sync_users_to_user_accounts;

-- Xóa bảng cũ (cẩn thận! Kiểm tra không còn references):
DROP TABLE users;

-- Tạo view để backward compatibility nếu cần:
CREATE VIEW users AS SELECT * FROM user_accounts;
```

---

## Ví Dụ 4: Thêm Foreign Key Vào Bảng Lớn

```sql
-- SAI: Khóa bảng khi validate toàn bộ:
-- ALTER TABLE orders ADD CONSTRAINT fk_user
--     FOREIGN KEY (user_id) REFERENCES users(id);

-- ĐÚNG: NOT VALID trước, validate sau:
-- Bước 1: Thêm FK constraint không validate (nhanh):
ALTER TABLE orders
    ADD CONSTRAINT fk_orders_user_id
    FOREIGN KEY (user_id) REFERENCES users(id)
    NOT VALID;

-- Bước 2: Validate riêng (SHARE UPDATE EXCLUSIVE — không block DML):
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_user_id;

-- Lưu ý: Giữa bước 1 và 2, FK constraint tồn tại nhưng chưa validate
-- New inserts/updates vẫn được kiểm tra
-- Chỉ existing data chưa được validate
```

---

## Ví Dụ 5: Thay Đổi Kiểu Cột

```sql
-- Mục tiêu: status VARCHAR(50) → status_id INTEGER (thêm bảng statuses)

-- Giai đoạn 1: EXPAND
CREATE TABLE statuses (
    id SERIAL PRIMARY KEY,
    code VARCHAR(50) NOT NULL UNIQUE,
    label VARCHAR(255)
);
INSERT INTO statuses (code, label) VALUES
    ('active', 'Active'),
    ('inactive', 'Inactive'),
    ('deleted', 'Deleted');

ALTER TABLE users ADD COLUMN status_id INTEGER REFERENCES statuses(id);

-- Giai đoạn 2: MIGRATE (backfill status_id từ status):
UPDATE users u
SET status_id = s.id
FROM statuses s
WHERE u.status = s.code;

-- Giai đoạn 3: Ứng dụng chuyển sang dùng status_id
-- Giai đoạn 4: CONTRACT
ALTER TABLE users ALTER COLUMN status_id SET NOT NULL;
ALTER TABLE users DROP COLUMN status;
```

---

## Template Runbook Expand-Contract

```sql
-- ============================================================
-- Migration: [Tên thay đổi]
-- Ngày: YYYY-MM-DD
-- Tác giả: [Tên]
-- Pattern: Expand-Contract
-- Rủi ro: THẤP / TRUNG BÌNH / CAO
-- Thời gian ước tính: [X phút]
-- ============================================================
--
-- Giai đoạn: 1 của 3 (EXPAND)
-- Tác động: Không có downtime
-- Rollback: [Lệnh rollback]
-- Đã test trên staging: [Ngày]
-- ============================================================

BEGIN;

-- [Câu lệnh SQL của Giai đoạn 1]

-- Xác minh:
-- SELECT [kiểm tra kết quả mong đợi];

COMMIT;

-- Giai đoạn 2 (MIGRATE) — Chạy riêng biệt sau deploy:
-- [Script backfill]

-- Giai đoạn 3 (CONTRACT) — Migration tiếp theo:
-- [Sẽ thực hiện trong sprint sau]
```

---

## Checklist Expand-Contract

- [ ] Xác định phase hiện tại (Expand / Migrate / Contract)
- [ ] Code cũ vẫn hoạt động sau Expand
- [ ] Backfill chạy theo batch (không full table lock)
- [ ] Verify số hàng trước và sau migration
- [ ] Constraint thêm với NOT VALID, validate riêng
- [ ] Index tạo với CONCURRENTLY
- [ ] Trigger sync được test (nếu dùng)
- [ ] Contract chỉ sau khi chắc chắn không còn code cũ

---

> **Điểm Mấu Chốt:** Expand-Contract là pattern cốt lõi cho zero-downtime deployments. Mỗi thay đổi schema phải được suy nghĩ như: "Làm thế nào để làm điều này mà code cũ vẫn chạy được?"
