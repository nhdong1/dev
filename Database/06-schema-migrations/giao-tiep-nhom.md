# Giao Tiếp & Phối Hợp Nhóm

Quy trình thông báo, review và phối hợp để migration diễn ra suôn sẻ.

## Tại Sao Giao Tiếp Quan Trọng?

```
Migration không chỉ là thay đổi kỹ thuật — nó ảnh hưởng đến:

App Engineers: Code phải tương thích với schema mới
QA Engineers: Test cases cần cập nhật
DevOps: Deployment pipeline cần phối hợp
Product/Business: Có thể có downtime (dù ngắn)
Customer Support: Alert khi có vấn đề
On-call: Sẵn sàng xử lý sự cố

Thiếu giao tiếp → Ai đó bị bất ngờ → Sự cố không được xử lý đúng cách
```

---

## Quy Trình Trước Migration

### Thông Báo Kế Hoạch (48h Trước)

```markdown
# Template Thông Báo Migration

**Kênh:** #engineering-all, #database-changes
**Mức độ:** Thông tin / Cần hành động

---

## 📋 Database Migration: [Mô tả ngắn]

**Migration:** V10 — Thêm cột `status` vào bảng `users`
**Thời gian dự kiến:** Thứ Năm, 27/04/2026, 14:00-14:15 ICT
**Người thực hiện:** [Tên DBA]
**Estimated downtime:** Không có (zero-downtime migration)

### Thay Đổi Schema
```sql
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'active';
CREATE INDEX CONCURRENTLY idx_users_status ON users(status);
```

### Tác Động
✅ Backward compatible — code hiện tại không bị ảnh hưởng
✅ Không downtime — migration không khóa bảng
⚠️ App team cần deploy feature trong tuần tới

### Hành Động Cần Thiết
- **App Team:** Review PR #456 (feature sử dụng cột status mới)
- **QA Team:** Cập nhật test cases cho user status feature
- **DevOps:** Không cần hành động

### Rollback Plan
- Code rollback về v1.0 an toàn (backward compatible)
- Backup sẽ được tạo trước khi migrate

### Contact
- Questions: DBA channel hoặc ping @[tên DBA]
- Incidents: @oncall-dba
```

### Peer Review Migration

```
Quy trình review migration:

1. DBA tạo PR với migration file
2. Reviewer checklist:
   □ Migration có trong transaction không?
   □ Backward compatible không?
   □ CONCURRENTLY cho index lớn không?
   □ Batch size cho backfill có hợp lý không?
   □ Verification query đã viết chưa?
   □ Rollback plan đã viết chưa?
   □ Đã test trên staging chưa?
3. Yêu cầu minimum 2 approvals (1 DBA, 1 App Engineer)
4. App Engineer kiểm tra code tương thích
```

```markdown
# PR Description Template

## Migration: V10__Add_status_to_users

### Mục Đích
Thêm cột `status` để track trạng thái user account.

### Thay Đổi Schema
```sql
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'active';
CREATE INDEX CONCURRENTLY idx_users_status ON users(status);
```

### Checklist
- [x] Migration trong transaction
- [x] Backward compatible (code cũ không dùng cột mới)
- [x] Index dùng CONCURRENTLY
- [x] Không có downtime dự kiến
- [x] Test trên staging thành công (xem #staging-test-results)
- [x] Backfill plan: Script V10b để update existing rows
- [x] Rollback plan: Code rollback an toàn, không cần schema rollback

### Staging Test Results
- Duration: 2.3 giây (trên 10M rows)
- Lock impact: Không (verified với concurrent SELECT)
- Row count: 10,000,000 → 10,000,000 ✅
- NULL check: 0 rows ✅

### Related PRs
- App feature: PR #456

### Reviewer Notes
Cần App Engineer verify backward compatibility.
```

---

## Quy Trình Trong Migration

### Real-Time Status Update

```bash
# Khi bắt đầu migration:
# Post vào Slack:
curl -X POST $SLACK_WEBHOOK -d '{
    "text": "🚀 Starting database migration V10 (Add status to users). ETA: 5 minutes. Contact @dba-oncall if issues."
}'

# Script tự động post updates:
#!/bin/bash

post_slack() {
    curl -X POST $SLACK_WEBHOOK -d "{\"text\": \"$1\"}"
}

post_slack "🚀 Migration V10 starting..."
START_TIME=$(date +%s)

flyway migrate
FLYWAY_EXIT=$?

END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))

if [ $FLYWAY_EXIT -eq 0 ]; then
    post_slack "✅ Migration V10 completed in ${DURATION}s. Running verification..."

    # Verification
    NULL_COUNT=$(psql $DB_URL -t -c "SELECT COUNT(*) FROM users WHERE status IS NULL;")
    if [ "$NULL_COUNT" -eq 0 ]; then
        post_slack "✅ Verification passed. Migration V10 successful!"
    else
        post_slack "⚠️ Verification WARNING: Found ${NULL_COUNT} NULL status values. Investigating..."
    fi
else
    post_slack "❌ Migration V10 FAILED! @oncall-dba please investigate immediately."
    exit 1
fi
```

### War Room Protocol (Cho Migration Lớn)

```
Khi migration có rủi ro trung bình trở lên:

Participants:
  - DBA Lead (người thực hiện migration)
  - Senior App Engineer (verify ứng dụng)
  - DevOps Engineer (deploy pipeline)
  - Incident Commander (theo dõi tổng thể)
  - (Optional) Engineering Manager

Setup:
  - Video call mở suốt quá trình
  - Shared terminal/screen để tất cả xem
  - Slack channel riêng cho migration (#migration-v10-warroom)
  - Monitoring dashboard mở (Grafana/Datadog)

Script war room:
  T-15m: Kiểm tra backup, staging test results
  T-5m:  Verify team có mặt, monitoring sẵn sàng
  T=0:   DBA bắt đầu migration, announce "Migration started"
  T+Xm:  DBA announce "Migration complete"
  T+Xm:  Run verification queries, share results
  T+Xm:  App engineer verify ứng dụng hoạt động
  T+Xm:  Declare success/failure
```

---

## Quy Trình Sau Migration

### Thông Báo Hoàn Thành

```markdown
# Template Thông Báo Hoàn Thành

## ✅ Migration V10 Completed Successfully

**Thời gian:** 14:03 - 14:07 ICT (4 phút)
**Thực hiện:** [Tên DBA]

### Kết Quả
- Duration: 4 phút 12 giây
- Downtime: Không có
- Row count: Không thay đổi (10,000,000 users)
- Verification: ✅ Passed (0 NULL status values)

### Actions Required
- **App Team:** Có thể deploy PR #456 ngay bây giờ
- **QA Team:** Môi trường staging đã sẵn sàng để test

### Next Steps
- Migration V11 (Backfill script): Thứ Hai, 01/05/2026
- Migration V12 (Add NOT NULL constraint): Sau khi V11 hoàn thành
```

---

## Post-Mortem Khi Có Sự Cố

### Template Post-Mortem

```markdown
# Post-Mortem: Migration V10 Incident

**Ngày:** 26/04/2026
**Severity:** SEV-2 (Service degraded, not down)
**Duration:** 14:05 - 14:45 ICT (40 phút degraded)
**Affected:** 15% users gặp lỗi khi update profile

## Timeline
14:03 - Migration V10 bắt đầu
14:07 - Migration hoàn thành
14:10 - Deploy App v2.0
14:15 - Error rate tăng từ 0.1% lên 5%
14:20 - Incident declared, on-call engaged
14:30 - Root cause identified: NULL status không được handle trong code
14:35 - Hotfix deployed (App v2.0.1)
14:45 - Error rate về bình thường

## Root Cause
Migration thêm cột status với DEFAULT 'active', nhưng code mới
giả định status LUÔN có giá trị ('active', 'inactive', etc.)
mà không handle trường hợp NULL (dù DEFAULT đã set,
các rows được INSERT trước migration bằng code cũ có thể có NULL
nếu INSERT chỉ định explicit column list không bao gồm status).

## Contributing Factors
1. Test staging không cover edge case: INSERT không chỉ định status column
2. Code review không phát hiện missing NULL check
3. Không có smoke test cho case INSERT sau migration

## Action Items
- [ ] [@dev] Thêm test case: INSERT không chỉ định status column
- [ ] [@dba] Thêm CHECK constraint: status IS NOT NULL (sau backfill)
- [ ] [@qa] Thêm regression test cho edge cases NULL values
- [ ] [@eng-process] Cập nhật migration review checklist: Verify NULL handling
- [ ] [@all] Cập nhật staging test suite với production-like edge cases

## Lessons Learned
1. DEFAULT không đảm bảo không có NULL (explicit INSERT bypass DEFAULT)
2. Migration review cần App Engineer kiểm tra edge cases cụ thể
3. Smoke test cần cover INSERT patterns phổ biến
```

---

## Communication Templates

### Kênh Thông Báo

```
#engineering-all:
  - Migration plans (trước 48h)
  - Completion notifications
  - Incidents

#database-changes:
  - Technical details
  - Review requests
  - Staging test results

#incidents (nếu có sự cố):
  - Real-time updates
  - Resolution timeline

#oncall:
  - Page khi cần immediate action
```

### RACI Matrix Cho Migration

| Hoạt Động | DBA | App Eng | DevOps | QA | Manager |
|-----------|-----|---------|--------|----|---------|
| Viết migration | **R** | C | - | - | - |
| Review migration | A | **R** | C | - | - |
| Test staging | **R** | C | - | C | - |
| Announce plan | **R** | I | I | I | I |
| Chạy migration | **R** | C | C | - | I |
| Verify post | **R** | **R** | - | C | - |
| Incident response | **R** | C | C | - | A |
| Post-mortem | **R** | C | C | C | A |

*R=Responsible, A=Accountable, C=Consulted, I=Informed*

---

## Runbook Template Chuẩn

```markdown
# Runbook: Migration [Version] — [Tên]

## Thông Tin Cơ Bản
| | |
|--|--|
| Migration | V___ |
| Mô tả | |
| Ngày thực hiện | |
| Người thực hiện | |
| Reviewer | |
| Risk level | LOW / MEDIUM / HIGH |
| Estimated duration | |
| Downtime | Không có / [X phút] |

## Pre-requisites
- [ ] PR approved (minimum 2 reviewers)
- [ ] Staging test: PASSED (link: ___)
- [ ] Backup created: [timestamp]
- [ ] Team notified (#engineering-all)
- [ ] War room setup (nếu HIGH risk)

## Steps

### 1. Pre-migration checks
```sql
-- Verify current state:
SELECT COUNT(*) FROM users;  -- Expected: ___
flyway info  -- Expected: V___ = Success (previous)
```

### 2. Run migration
```bash
flyway migrate
```

### 3. Verify
```sql
SELECT COUNT(*) FROM users WHERE status IS NULL;  -- Expected: 0
SELECT * FROM flyway_schema_history ORDER BY installed_rank DESC LIMIT 1;  -- State: Success
```

### 4. Notify completion
- Post kết quả vào #database-changes
- Tag App team để deploy feature

## Rollback Procedure
Nếu migration thất bại:
1. Kiểm tra flyway_schema_history — State = Failed?
2. Chạy flyway repair
3. Sửa vấn đề
4. Chạy lại flyway migrate

Nếu ứng dụng lỗi sau migration:
1. kubectl rollout undo deployment/myapp
2. Verify error rate giảm
3. Investigate root cause
4. Deploy hotfix

## Contacts
- DBA on-call: @[name]
- App on-call: @[name]
- Escalation: @[manager]
```

---

## Checklist Giao Tiếp Nhóm

**48h Trước:**
- [ ] Thông báo kế hoạch trên #engineering-all
- [ ] PR tạo và assign reviewers
- [ ] Staging test results shared
- [ ] War room setup (nếu cần)

**4h Trước:**
- [ ] Reminder announcement
- [ ] Backup confirmed
- [ ] All participants confirmed available

**Trong Migration:**
- [ ] "Migration started" announcement
- [ ] Real-time updates nếu có delay
- [ ] "Migration complete" announcement với results

**Sau Migration:**
- [ ] Completion notification với metrics
- [ ] Action items cho các team
- [ ] Post-mortem lên lịch (nếu có incident)

---

> **Điểm Mấu Chốt:** Migration thành công là kết quả của cả team, không phải chỉ DBA. Giao tiếp rõ ràng trước, trong và sau migration giúp mọi người chuẩn bị đúng và phản ứng nhanh khi cần.
