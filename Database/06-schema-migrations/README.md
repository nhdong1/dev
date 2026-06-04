# Di Chuyển Schema & Quản Lý Thay Đổi

Thay đổi schema an toàn, không thời gian chết cho CSDL production.

## Các Tài Liệu

| Tài liệu | Mô tả |
|----------|-------|
| [Công Cụ Migration](cong-cu-migration.md) | Flyway, Liquibase, sqitch — so sánh và hướng dẫn sử dụng |
| [Mẫu Expand-Contract](expand-contract.md) | Zero-downtime schema changes — thêm cột, đổi tên, thay đổi type |
| [Triết Lý Chỉ Tiến](chi-tien-forward-only.md) | Forward-only philosophy — tại sao không rollback schema |
| [Xác Thực Dữ Liệu](xac-thuc-du-lieu.md) | Row count, backfill completeness, index validity, constraint check |
| [Triển Khai & Phối Hợp](trien-khai-phoi-hop.md) | Thứ tự deploy, CI/CD pipeline, feature flags, lock management |
| [Chiến Lược Rollback](chien-luoc-rollback.md) | Phòng ngừa, xử lý sự cố, khôi phục từ backup |
| [Testing Migration](testing-migration.md) | Staging data, automated tests, performance testing |
| [Giao Tiếp Nhóm](giao-tiep-nhom.md) | Thông báo, review, war room, post-mortem |

---

## Nguyên Tắc Cốt Lõi

```
1. Mọi migration phải backward compatible
   → Code cũ vẫn chạy sau khi schema mới được deploy
   → Cho phép rolling deployment và code rollback an toàn

2. Database migrate TRƯỚC khi deploy app
   → Schema sẵn sàng khi code mới cần dùng

3. Expand-Contract cho mọi thay đổi breaking
   → KHÔNG BAO GIỜ: DROP COLUMN / RENAME / CHANGE TYPE trong một bước

4. Forward-only — Không rollback schema
   → Sửa bằng migration mới, không quay ngược

5. Test trên staging với data production-like
   → Performance, edge cases, concurrency

6. Xác thực là bắt buộc
   → Migration chưa xong cho đến khi verification pass
```

---

## Phân Loại Rủi Ro Migration

```
RỦI RO THẤP (Làm bất kỳ lúc nào):
✓ ADD COLUMN nullable với DEFAULT
✓ CREATE INDEX CONCURRENTLY
✓ CREATE TABLE mới
✓ ADD stored procedure/function
✓ ADD CHECK CONSTRAINT với NOT VALID

RỦI RO TRUNG BÌNH (Cần phối hợp, test kỹ):
⚠ ADD COLUMN NOT NULL (phải backfill trước)
⚠ ADD FOREIGN KEY (dùng NOT VALID rồi VALIDATE)
⚠ RENAME COLUMN (dùng Expand-Contract)
⚠ RENAME TABLE (dùng trigger sync)
⚠ Backfill lớn (batch để tránh lock)

RỦI RO CAO (Cần maintenance window hoặc Expand-Contract):
❌ DROP COLUMN (breaking change)
❌ DROP TABLE (breaking change)
❌ ALTER COLUMN TYPE (có thể mất data)
❌ Xóa dữ liệu hàng loạt
❌ CREATE INDEX (không CONCURRENTLY → Khóa bảng)
```

---

## Luồng Migration Hoàn Chỉnh

```
Bước 1: THIẾT KẾ
  → Xác định loại thay đổi (Expand-Contract hay trực tiếp?)
  → Chọn pattern phù hợp
  → Viết migration SQL

Bước 2: REVIEW
  → Peer review (DBA + App Engineer)
  → Kiểm tra backward compatibility
  → Kiểm tra lock impact

Bước 3: TEST STAGING
  → Chạy trên staging với data production-like
  → Đo thời gian thực thi
  → Test concurrent access
  → Automated verification tests

Bước 4: CHUẨN BỊ
  → Tạo backup production
  → Thông báo team (48h trước)
  → Runbook sẵn sàng

Bước 5: DEPLOY
  → Chạy migration (TRƯỚC khi deploy app)
  → Monitor locks và errors
  → Verify ngay sau khi xong

Bước 6: VERIFY & THÔNG BÁO
  → Chạy verification queries
  → Thông báo completion
  → App team deploy feature

Bước 7: CONTRACT (Nếu dùng Expand-Contract)
  → Sprint sau: Xóa schema cũ
  → Verify không còn code dùng schema cũ
```

---

## Expand-Contract Nhanh

```sql
-- THÊM CỘT NOT NULL (Ví dụ kinh điển):

-- Giai đoạn 1: EXPAND (Migration V10)
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'active';
CREATE INDEX CONCURRENTLY idx_users_status ON users(status);

-- Giai đoạn 2: MIGRATE (Migration V11 hoặc script riêng)
UPDATE users SET status = 'active' WHERE status IS NULL;
-- Batch: WHERE id BETWEEN X AND Y để tránh lock

-- Giai đoạn 3: CONTRACT (Migration V12)
ALTER TABLE users ADD CONSTRAINT chk_status_not_null
    CHECK (status IS NOT NULL) NOT VALID;
ALTER TABLE users VALIDATE CONSTRAINT chk_status_not_null;
ALTER TABLE users ALTER COLUMN status SET NOT NULL;
ALTER TABLE users DROP CONSTRAINT chk_status_not_null;
```

---

## Checklist Tổng Hợp

**Thiết Kế:**
- [ ] Migration backward compatible (code cũ không bị break)
- [ ] Expand-Contract được áp dụng nếu là breaking change
- [ ] Index dùng CONCURRENTLY
- [ ] FK dùng NOT VALID rồi VALIDATE riêng
- [ ] Backfill theo batch (không lock toàn bảng)

**Review & Test:**
- [ ] 2+ approvals (DBA + App Engineer)
- [ ] Staging test với data production-like
- [ ] Thời gian thực thi acceptable
- [ ] Concurrent access test
- [ ] Automated verification tests pass

**Triển Khai:**
- [ ] Backup trước khi migrate
- [ ] Thông báo team (48h trước)
- [ ] Runbook sẵn sàng
- [ ] Migration TRƯỚC khi deploy app

**Xác Thực:**
- [ ] Row count không thay đổi
- [ ] NULL check (backfill hoàn chỉnh)
- [ ] Index valid (indisvalid = true)
- [ ] Smoke tests pass
- [ ] Thông báo completion

---

## Câu Hỏi Phỏng Vấn Thường Gặp

**1. Thêm cột NOT NULL vào bảng 100M hàng mà không có downtime?**
- Giai đoạn 1 (Expand): ADD COLUMN nullable với DEFAULT → Không lock
- Giai đoạn 2 (Migrate): Backfill theo batch trong business hours
- Giai đoạn 3 (Contract): ADD CONSTRAINT NOT VALID → VALIDATE → SET NOT NULL

**2. Migration thất bại giữa chừng — làm gì?**
- Đánh giá blast radius (bao nhiêu hàng bị ảnh hưởng)
- Nếu trong transaction: PostgreSQL tự rollback → OK
- Nếu không: Kiểm tra state, sửa thủ công, flyway repair, retry
- Forward fix với migration mới (không rollback schema)

**3. Thiết kế chiến lược rename table trong production?**
- Tạo bảng mới với tên mới (rỗng)
- Thêm trigger sync write từ bảng cũ sang mới
- Backfill dữ liệu hiện có
- Feature flag để chuyển đọc dần sang bảng mới
- Khi 100% traffic dùng mới: Xóa trigger và bảng cũ

---

> **Điểm Mấu Chốt:** Migration tốt nhất là migration bạn không bao giờ cần rollback. Đầu tư vào thiết kế tương thích ngược, test kỹ trên staging, và luôn có cách tiến về phía trước dù có vấn đề.
