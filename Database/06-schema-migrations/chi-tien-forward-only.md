# Triết Lý Chỉ Tiến (Forward-Only Migrations)

Tại sao không rollback schema — và cách thay thế an toàn hơn.

## Hai Trường Phái

### Trường Phái Truyền Thống: Có Rollback

```
Mỗi migration có hai phiên bản:
- UP migration: Thay đổi schema lên
- DOWN migration: Hoàn tác thay đổi

Ví dụ Flyway:
V3__Add_status_column.sql   (UP)
U3__Undo_status_column.sql  (DOWN — chỉ Flyway Teams)

Liquibase:
<changeSet id="3">
    <addColumn>...</addColumn>
    <rollback>
        <dropColumn.../>
    </rollback>
</changeSet>

Mục tiêu: Có thể rollback bất kỳ lúc nào
```

### Trường Phái Chỉ Tiến: Không Rollback Schema

```
Chỉ có UP migration, không có DOWN migration.
Khi có vấn đề → Viết migration mới để sửa.

Ví dụ:
V3__Add_status_column.sql  ← Thêm cột (có bug)
V4__Fix_status_default.sql ← Migration mới để sửa (không rollback V3)

Mục tiêu: Mọi thay đổi đều có kiểm soát, có thể audit
```

---

## Tại Sao Rollback Schema Nguy Hiểm?

### Vấn Đề 1: Schema-Code Mismatch

```
Timeline:
T=0: Deploy V3 migration (thêm cột status)
T=1: Deploy app v2.0 (dùng cột status)
T=2: Phát hiện bug trong app v2.0
T=3: Rollback app về v1.0
T=4: Rollback schema về V2 (xóa cột status)

Vấn đề ở T=3:
- App v1.0 không biết về cột status → OK
- App v2.0 đang chạy trên một số server vẫn dùng status → LỖI!

Vấn đề ở T=4 (nếu rollback schema):
- App v2.0 đã lưu data vào cột status
- Rollback schema = Mất data!
```

### Vấn Đề 2: Data Loss

```sql
-- Migration V5: Xóa cột phone_number
ALTER TABLE users DROP COLUMN phone_number;

-- Sau vài giờ: Phát hiện cần phone_number!
-- DOWN migration:
ALTER TABLE users ADD COLUMN phone_number VARCHAR(20);
-- Dữ liệu phone_number đã MẤT VĨNH VIỄN!
```

### Vấn Đề 3: Rollback Down Migration Có Thể Có Bug

```sql
-- UP migration:
ALTER TABLE orders ADD COLUMN discount DECIMAL(5,2) DEFAULT 0.00;
UPDATE orders SET discount = 0.10 WHERE user_id IN (SELECT id FROM premium_users);

-- DOWN migration:
ALTER TABLE orders DROP COLUMN discount;
-- Bug: Mất track các orders đã có discount!
-- Khi UP lại: Discount về 0 cho tất cả — sai!
```

### Vấn Đề 4: Không Hoạt Động Với Production

```
Rollback schema trong production:
1. Backup trước khi rollback? Nếu có transaction mới trong lúc đó thì sao?
2. Replica lag? Replica vẫn đang sync schema mới trong khi primary đã rollback?
3. Connection pool đang giữ session với schema cũ?

= Quá nhiều edge cases, quá nhiều rủi ro
```

---

## Triết Lý Chỉ Tiến

```
Nguyên tắc cốt lõi:
"Migration là một chiều — như thời gian, không thể quay ngược."

Khi có vấn đề:
❌ KHÔNG rollback schema
✓ Viết migration mới để sửa
✓ Rollback code về version cũ nếu cần (schema compatible)
✓ Hotfix với migration mới
```

### Lợi Ích

```
1. Đơn giản hơn:
   - Chỉ cần UP migrations, không cần DOWN
   - Ít code hơn, ít test hơn
   - Flyway Core (miễn phí) hỗ trợ đầy đủ

2. An toàn hơn:
   - Không bao giờ mất data từ rollback
   - Không có schema-code mismatch
   - Luôn có audit trail đầy đủ

3. Tư duy đúng đắn:
   - Buộc thiết kế migration tương thích ngược
   - Dùng Expand-Contract pattern
   - Kiểm tra kỹ trên staging
```

---

## Cách Xử Lý Khi Migration Có Vấn Đề

### Kịch Bản 1: Migration Thêm Cột Sai

```sql
-- V10__Add_wrong_column.sql (đã deploy):
ALTER TABLE users ADD COLUMN age INTEGER;  -- Bug: Nên là birth_date

-- Giải pháp: Migration mới để sửa
-- V11__Replace_age_with_birth_date.sql:
ALTER TABLE users ADD COLUMN birth_date DATE;
-- Không xóa age ngay (backward compat)
UPDATE users SET birth_date = (NOW() - (age || ' years')::INTERVAL)::DATE
WHERE age IS NOT NULL;
-- Age sẽ được xóa trong V12 (Contract phase)
```

### Kịch Bản 2: Migration Thêm Index Sai

```sql
-- V10__Add_wrong_index.sql (đã deploy):
CREATE INDEX idx_orders_user ON orders(user_id, status, total);
-- Bug: Index quá lớn, không cần cột total

-- Giải pháp: Drop và tạo lại
-- V11__Fix_orders_index.sql:
DROP INDEX CONCURRENTLY idx_orders_user;
CREATE INDEX CONCURRENTLY idx_orders_user_status ON orders(user_id, status);
-- Đặt tên mới để rõ ràng hơn
```

### Kịch Bản 3: Migration Có Lỗi Dữ Liệu

```sql
-- V10__Update_user_status.sql (đã deploy):
UPDATE users SET status = 'inactive' WHERE last_login < '2025-01-01';
-- Bug: Cập nhật nhầm — nên là 'dormant' thay vì 'inactive'

-- Giải pháp: Migration mới để sửa
-- V11__Fix_dormant_user_status.sql:
UPDATE users
SET status = 'dormant'
WHERE status = 'inactive'
  AND last_login < '2025-01-01';
-- Rollback data theo logic, không rollback schema
```

### Kịch Bản 4: Migration Làm Chậm Production

```sql
-- V10__Add_heavy_index.sql (đang chạy, rất chậm):
CREATE INDEX idx_orders_all ON orders(user_id, status, created_at, total);
-- Khóa bảng quá lâu!

-- Nếu chưa xong: Kill process
SELECT pg_cancel_backend(pid) FROM pg_stat_activity
WHERE query LIKE '%CREATE INDEX%';

-- Migration mới với CONCURRENTLY:
-- V11__Add_heavy_index_concurrent.sql:
CREATE INDEX CONCURRENTLY idx_orders_all ON orders(user_id, status, created_at, total);
```

---

## Khi Nào NÊN Rollback Code (Không Phải Schema)?

```
Rollback code (app) là hoàn toàn OK khi:
✓ Schema thay đổi tương thích ngược (thêm cột nullable)
✓ Code cũ không biết về cột mới nhưng vẫn hoạt động
✓ Không có data dependency

KHÔNG rollback code khi:
✗ Schema thay đổi không tương thích ngược (xóa cột)
✗ Code cũ cần cột đã bị xóa
✗ Data format đã thay đổi (cần code mới để đọc)

→ Đây là lý do LUÔN dùng Expand-Contract:
  Sau Expand, code cũ VẪN HOẠT ĐỘNG
  → Rollback code về v_cũ an toàn hoàn toàn
```

---

## Quy Trình Forward-Only Trong Thực Tế

```
Phát hiện vấn đề sau khi migration đã chạy:

Bước 1: Đánh giá blast radius
  → Bao nhiêu hàng bị ảnh hưởng?
  → Hệ thống có đang chạy không?
  → User có bị ảnh hưởng không?

Bước 2: Quyết định chiến lược
  → Sửa với migration mới (forward fix)
  → Khôi phục từ backup (chỉ khi mất data nghiêm trọng)

Bước 3: Viết và test hotfix migration
  → Test trên staging với data production-like
  → Peer review

Bước 4: Deploy hotfix
  → Ưu tiên deploy nhanh nếu user bị ảnh hưởng

Bước 5: Post-mortem
  → Tại sao migration staging không phát hiện vấn đề?
  → Cải thiện quy trình test
```

---

## Cài Đặt Flyway Cho Forward-Only

```bash
# flyway.conf:
flyway.outOfOrder=false        # Không cho phép migration out of order
flyway.validateOnMigrate=true  # Validate checksum (phát hiện file bị sửa)
flyway.cleanDisabled=true      # NGUY HIỂM: Tắt lệnh clean (xóa toàn bộ schema)
flyway.baselineOnMigrate=false # Không tự động baseline

# CI/CD pipeline:
# - CHỈ chạy 'flyway migrate', không bao giờ 'flyway clean'
# - Thêm check: Không cho phép sửa migration files đã chạy
```

---

## Checklist Forward-Only

- [ ] Team đã đồng ý triết lý forward-only
- [ ] Không có DOWN migrations được viết
- [ ] Migration files không được sửa sau khi đã chạy
- [ ] flyway.validateOnMigrate=true (phát hiện sửa đổi)
- [ ] Quy trình hotfix migration được document
- [ ] Mọi migration được thiết kế tương thích ngược
- [ ] Expand-Contract được áp dụng cho thay đổi breaking

---

> **Điểm Mấu Chốt:** Forward-only không có nghĩa là không thể sửa sai — nó có nghĩa là bạn sửa sai bằng cách tiến về phía trước với migration mới, không phải quay ngược với rollback. Điều này an toàn hơn vì không bao giờ mất data và không có schema-code mismatch.
