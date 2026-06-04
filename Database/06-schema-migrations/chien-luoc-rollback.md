# Chiến Lược Rollback & Phục Hồi

Kế hoạch xử lý khi migration gặp sự cố — từ phòng ngừa đến khắc phục.

## Triết Lý Rollback

```
Thứ tự ưu tiên khi có sự cố:

1. FORWARD FIX (Ưu tiên nhất):
   Viết migration mới để sửa vấn đề
   → Không mất data, không risk thêm

2. CODE ROLLBACK:
   Rollback ứng dụng về version cũ
   Schema vẫn mới (nhưng backward compatible)
   → An toàn nếu dùng Expand-Contract

3. BACKUP RESTORE (Cuối cùng):
   Khôi phục database từ backup
   → Mất data sau thời điểm backup
   → Chỉ dùng khi data bị corrupt nghiêm trọng

Không có "4. Schema rollback" — vì quá nguy hiểm!
```

---

## Phòng Ngừa: Checklist Trước Migration

### 48 Giờ Trước

```sql
-- 1. Kiểm tra migration trên staging với data production-like:
flyway migrate  -- Trên staging

-- 2. Đo thời gian thực thi:
\timing
-- [Chạy migration SQL thủ công để đo]

-- 3. Kiểm tra lock impact:
EXPLAIN SELECT * FROM pg_locks;

-- 4. Verify data integrity sau staging migration:
SELECT COUNT(*) FROM users WHERE status IS NULL;  -- Phải là 0

-- 5. Test rollback plan trên staging:
-- (Không rollback schema, nhưng test code rollback)
```

### 4 Giờ Trước

```bash
# 1. Backup production:
pg_basebackup -h localhost -U backup_user -D /backup/$(date +%Y%m%d_%H%M%S) -Fp -Xs -P

# Hoặc cho AWS RDS:
aws rds create-db-snapshot \
    --db-instance-identifier mydb \
    --db-snapshot-identifier "pre-migration-$(date +%Y%m%d-%H%M%S)"

# 2. Verify backup thành công:
aws rds describe-db-snapshots \
    --db-snapshot-identifier "pre-migration-$(date +%Y%m%d-%H%M%S)" \
    --query 'DBSnapshots[0].Status'
# Phải là "available"

# 3. Đảm bảo team sẵn sàng:
# - App engineer
# - DBA
# - On-call engineer
# - Management (nếu maintenance window lớn)
```

---

## Kịch Bản Và Cách Xử Lý

### Kịch Bản 1: Migration Bị Kẹt (Lock Wait)

```sql
-- Triệu chứng: Migration chạy lâu bất thường
-- Nguyên nhân: Đang chờ lock từ transaction khác

-- Bước 1: Tìm blocking transaction:
SELECT
    blocked.pid AS blocked_pid,
    blocked_activity.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking_activity.query AS blocking_query,
    blocking_activity.usename AS blocking_user
FROM pg_locks blocked
JOIN pg_stat_activity blocked_activity ON blocked.pid = blocked_activity.pid
JOIN pg_locks blocking ON blocking.relation = blocked.relation
    AND blocking.locktype = blocked.locktype
    AND blocking.pid != blocked.pid
    AND blocking.granted = true
    AND blocked.granted = false
JOIN pg_stat_activity blocking_activity ON blocking.pid = blocking_activity.pid;

-- Bước 2: Quyết định có cancel blocking query không:
-- Chỉ cancel nếu đó là idle transaction hoặc query không quan trọng
SELECT pg_cancel_backend(blocking_pid);  -- Soft cancel
-- Hoặc:
SELECT pg_terminate_backend(blocking_pid);  -- Hard kill

-- Bước 3: Nếu migration timeout:
-- Migration thường tự rollback (PostgreSQL transaction)
-- Verify không có partial changes:
SELECT * FROM flyway_schema_history ORDER BY installed_rank DESC LIMIT 5;
-- State phải là 'Success' hoặc không có record cho migration đó
```

### Kịch Bản 2: Migration Thất Bại Giữa Chừng

```sql
-- Trường hợp migration là một transaction:
-- PostgreSQL tự động rollback toàn bộ → Schema sạch, OK!

-- Trường hợp migration có nhiều statements không trong transaction:
-- Phải check xem đã chạy đến đâu:

-- Kiểm tra cột đã tạo chưa:
SELECT column_name FROM information_schema.columns
WHERE table_name = 'users' AND column_name = 'status';

-- Kiểm tra index đã tạo chưa:
SELECT indexname FROM pg_indexes
WHERE tablename = 'users' AND indexname = 'idx_users_status';

-- Kiểm tra bao nhiêu hàng đã được update:
SELECT COUNT(*) FROM users WHERE status IS NOT NULL;
```

```bash
# Flyway: Migration thất bại → state = 'Failed' trong flyway_schema_history
flyway info
# Version 10: Failed

# Sửa vấn đề thủ công (nếu cần)
# Sau đó đánh dấu resolved:
flyway repair
# → Xóa 'Failed' record, cho phép retry

# Retry:
flyway migrate
```

### Kịch Bản 3: Migration Gây Ra Bug Trong Ứng Dụng

```
Timeline:
T=0: Migration V10 chạy thành công
T=5: Deploy App v2.0
T=20: Error rate tăng (bug phát hiện)

Đánh giá:
- Schema thay đổi gì? (Xem migration diff)
- Bug có liên quan đến schema không?
- Code cũ (v1.0) có hoạt động với schema mới không?
```

```bash
# Option 1: Rollback code (nếu schema backward compatible):
kubectl rollout undo deployment/myapp
# App v1.0 chạy lại với schema v10 (OK nếu dùng Expand-Contract)

# Option 2: Forward fix (deploy hotfix):
git checkout -b hotfix/migration-v10-fix
# Sửa bug
git push origin hotfix/migration-v10-fix
# Deploy app v2.0.1

# Monitor:
kubectl rollout status deployment/myapp
```

### Kịch Bản 4: Mất Dữ Liệu Nghiêm Trọng

```bash
# Migration xóa nhầm data quan trọng:
-- DELETE FROM orders WHERE status = 'completed' AND created_at < '2026-01-01';
-- Nhưng lẽ ra chỉ xóa test data, không phải production data!

# Đánh giá ngay:
SELECT COUNT(*) FROM orders;  -- Còn bao nhiêu?
SELECT MIN(created_at), MAX(created_at) FROM orders;  -- Phạm vi nào còn?

# Nếu data quan trọng bị mất → Restore từ backup!

# Bước 1: Stop writes nếu có thể (giảm window data loss):
# Đặt maintenance mode trong ứng dụng

# Bước 2: Restore từ backup gần nhất:
# AWS RDS:
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier mydb-restored \
    --db-snapshot-identifier pre-migration-20260426-090000

# Bước 3: PITR (Point-In-Time Recovery) nếu cần chính xác hơn:
aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier mydb \
    --target-db-instance-identifier mydb-restored-pitr \
    --restore-time 2026-04-26T08:59:59Z  # 1 phút trước khi migration

# Bước 4: Verify restored database:
psql -h mydb-restored.xxx.rds.amazonaws.com -U admin mydb
SELECT COUNT(*) FROM orders;  # Phải khớp với pre-migration count

# Bước 5: Switch traffic (sau khi verify):
# Cập nhật connection string trong ứng dụng
```

### Kịch Bản 5: CONCURRENTLY Index Bị Interrupt

```sql
-- CREATE INDEX CONCURRENTLY bị interrupt → Index invalid!
SELECT schemaname, tablename, indexname, indisvalid
FROM pg_indexes
JOIN pg_index ON indexrelid = (SELECT oid FROM pg_class WHERE relname = indexname)
WHERE indisvalid = false;

-- Output: idx_orders_status | valid: false

-- Xử lý: Drop invalid index và tạo lại
DROP INDEX CONCURRENTLY idx_orders_status;  -- Drop index invalid
CREATE INDEX CONCURRENTLY idx_orders_status ON orders(status);  -- Tạo lại
```

---

## Runbook Phục Hồi Chuẩn

```markdown
# Runbook: Database Migration Recovery

## Thông Tin
- Migration Version: V__
- Database: production/mydb
- Thời điểm sự cố: [TIMESTAMP]
- Người xử lý: [TÊN]

## Đánh Giá Nhanh (5 phút đầu)
- [ ] Blast radius: Bao nhiêu users bị ảnh hưởng?
- [ ] Loại sự cố: Schema error / Data error / Performance
- [ ] Có backup gần nhất không?
- [ ] Migration có trong transaction không?

## Quyết Định (phút 5-10)
- [ ] Forward fix (migration mới)
- [ ] Code rollback
- [ ] Backup restore

## Thực Hiện
### Nếu Code Rollback:
kubectl rollout undo deployment/myapp
kubectl rollout status deployment/myapp
./smoke-test.sh

### Nếu Forward Fix:
# Viết và test hotfix migration
# Deploy hotfix

### Nếu Backup Restore:
# Xem kịch bản 4 ở trên

## Verify Phục Hồi
- [ ] Error rate về bình thường
- [ ] Row count kiểm tra
- [ ] Smoke tests pass

## Thông Báo
- [ ] Team được thông báo
- [ ] Status page updated
- [ ] Post-mortem được lên lịch
```

---

## Checklist Chiến Lược Rollback

**Phòng Ngừa:**
- [ ] Backup được tạo trước mỗi migration quan trọng
- [ ] Migration test trên staging với data production-like
- [ ] Rollback plan được viết TRƯỚC khi migrate
- [ ] Mọi migration trong transaction (khi có thể)

**Khi Sự Cố:**
- [ ] Không panic — đánh giá blast radius trước
- [ ] Stop writes nếu mất data đang tiếp tục
- [ ] Chọn phương án ít rủi ro nhất
- [ ] Thông báo team và stakeholders

**Sau Phục Hồi:**
- [ ] Verify dữ liệu đầy đủ
- [ ] Smoke tests pass
- [ ] Post-mortem trong 48 giờ
- [ ] Cải thiện quy trình để không tái diễn

---

> **Điểm Mấu Chốt:** Rollback plan tốt nhất là không cần dùng đến. Đầu tư vào testing trên staging, backup đầy đủ, và Expand-Contract pattern để mọi migration đều backward compatible — đây là phòng thủ tốt hơn bất kỳ rollback plan nào.
