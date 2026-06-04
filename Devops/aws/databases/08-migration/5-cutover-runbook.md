# Cutover Runbook — Kế Hoạch Cắt Chuyển & Rollback

> Cutover (Cắt Chuyển) là thời điểm quan trọng nhất trong toàn bộ quá trình migration — khi traffic thực từ source database được chuyển sang target. Runbook (Sách Hướng Dẫn Vận Hành) chi tiết, được luyện tập trước, và có plan rollback (kế hoạch quay lại) rõ ràng là sự khác biệt giữa một migration thành công và một incident nghiêm trọng.

## 📚 Mục Lục

1. [Điều Kiện Cutover](#điều-kiện-cutover)
2. [Pre-Cutover Checklist](#pre-cutover-checklist)
3. [Cutover Runbook Từng Bước](#cutover-runbook-từng-bước)
4. [Rollback Plan (Kế Hoạch Quay Lại)](#rollback-plan)
5. [Post-Cutover Validation](#post-cutover-validation)
6. [Lessons Learned từ Thực Tế](#lessons-learned-từ-thực-tế)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## ✅ Điều Kiện Cutover

### Go/No-Go Criteria (Tiêu Chí Tiếp Tục/Dừng Lại)

Trước khi xác nhận ngày cutover, tất cả tiêu chí sau phải được đáp ứng:

```
Data:
  □ DMS full load hoàn thành, row counts khớp ±0.1%
  □ DMS CDC lag ổn định < 30 giây trong 24 giờ liên tiếp
  □ Checksum validation (xác minh tổng kiểm tra) PASSED trên 5 bảng lớn nhất
  □ DMS validation task PASSED (0 errors)

Performance:
  □ Query benchmark (điểm chuẩn truy vấn) trên target: p95 < 2x so với source
  □ Target database đủ connection pool cho production traffic
  □ Target storage có dự phòng ≥ 30%

Application:
  □ Integration tests PASSED 100% trên target DB
  □ Staging environment đã chạy trên target DB ≥ 1 tuần không có incident
  □ All application teams đã sign-off (xác nhận)

Operations:
  □ Rollback plan được viết và review bởi ≥ 2 người
  □ Rollback tested (kiểm thử rollback) trong staging
  □ On-call rotation đã được thông báo
  □ Communication plan gửi đến stakeholders
  □ Maintenance window đã được announce (thông báo) trước ≥ 3 ngày
```

### Chọn Maintenance Window (Cửa Sổ Bảo Trì)

```
Tiêu chí chọn:
  - Traffic thấp nhất trong tuần (thường Chủ Nhật 2-6 AM theo múi giờ users)
  - Tránh ngay trước/sau holidays lớn
  - Tránh cuối tháng/quý (batch jobs quan trọng)
  - Dự phòng thêm 50% thời gian ước tính
    Ví dụ: Ước tính 2 giờ → Book 3 giờ window

Thông báo:
  - Internal teams: 1 tuần trước
  - Customers (nếu có downtime): 3 ngày trước qua email/status page
  - Format: "Maintenance window: Chủ Nhật 15/06 02:00-05:00 ICT"
```

---

## 📋 Pre-Cutover Checklist

### T-72 giờ (3 Ngày Trước)

```
□ Confirm Go/No-Go với tất cả stakeholders
□ Finalize runbook (hoàn thiện sách hướng dẫn) và distribute (phân phát) cho team
□ Test rollback procedure trong staging environment
□ Xác nhận on-call engineers (kỹ sư trực) sẽ có mặt
□ Prepare communication templates (mẫu thông báo) cho users
□ Verify target DB security groups cho phép traffic từ ứng dụng production
□ Confirm DMS task vẫn healthy, CDC lag < 30s
```

### T-24 giờ (1 Ngày Trước)

```
□ Freeze schema changes — NO ALTER TABLE trên source
□ Thông báo team: không deploy code changes trong window
□ Backup thủ công source database (ngoài automated backup)
□ Export current source DB snapshot → store riêng
□ Verify target DB has latest snapshot
□ Double-check connection strings mới trong config repo
□ Pre-pull Docker images / pre-build application (nếu cần deploy mới)
□ Review DMS metrics lần cuối
```

### T-2 giờ (2 Giờ Trước)

```
□ Check DMS lag hiện tại
□ Confirm all application servers accessible
□ Verify monitoring dashboards (bảng điều khiển giám sát) đang chạy
□ War room setup: video call, Slack channel #migration-cutover
□ Contact list sẵn sàng: DBA, App Lead, Infra Lead, Manager
□ Rollback scripts được copy vào terminal, sẵn sàng chạy
□ Smoke test scripts sẵn sàng
```

---

## 🚀 Cutover Runbook Từng Bước

### Tổng Quan Timeline

```
T+00:00  Bắt đầu maintenance window
T+00:05  Drain connections (xả kết nối) và dừng writes
T+00:10  Chờ CDC lag = 0
T+00:20  Final validation
T+00:30  Switch connection strings
T+00:40  Start application servers với config mới
T+00:50  Smoke tests
T+01:00  Go/No-Go quyết định mở traffic
T+01:05  Mở traffic cho users
T+01:30  Monitor 30 phút, confirm stable
T+01:30  Declare success (tuyên bố thành công)
```

### Bước 1: Bắt Đầu Maintenance Window

```bash
# 1. Post announcement trên status page
curl -X POST https://api.statuspage.io/v1/pages/PAGE_ID/incidents \
  -H "Authorization: OAuth TOKEN" \
  -d '{
    "incident": {
      "name": "Scheduled Database Maintenance",
      "status": "investigating",
      "body": "Database maintenance in progress. Expected duration: 2 hours."
    }
  }'

# 2. Thông báo trong Slack
echo "🔧 DATABASE MIGRATION CUTOVER STARTED at $(date)"
echo "Expected completion: $(date -d '+2 hours')"
echo "War room: #migration-cutover"
echo "Rollback trigger: Post 🔴 RED in war room channel"
```

### Bước 2: Drain Connections & Stop Writes

```
Approach 1 — Application Maintenance Mode:
  # Bật maintenance mode trên load balancer
  # Hoặc đổi application config
  APP_MODE=maintenance
  systemctl reload nginx   # Return 503 cho new requests
  
  # Chờ existing requests hoàn thành (graceful drain — xả dần)
  sleep 30
  
  # Verify: không còn active transactions trên source
  # MySQL:
  SHOW PROCESSLIST;
  # Chờ không còn queries chạy > 1 giây

Approach 2 — Read-Only Mode:
  # MySQL:
  SET GLOBAL read_only = ON;
  SET GLOBAL super_read_only = ON;
  
  # PostgreSQL:
  ALTER DATABASE myapp SET default_transaction_read_only = on;
```

### Bước 3: Chờ DMS CDC Lag Về 0

```bash
# Monitor DMS lag (theo dõi độ trễ DMS)
while true; do
  LAG=$(aws cloudwatch get-metric-statistics \
    --namespace AWS/DMS \
    --metric-name CDCLatencySource \
    --dimensions Name=ReplicationInstanceIdentifier,Value=my-dms-instance \
    --start-time $(date -u -d '-5 minutes' +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 60 \
    --statistics Average \
    --query 'Datapoints[0].Average' \
    --output text)
  
  echo "$(date): CDCLatencySource = ${LAG} seconds"
  
  if (( $(echo "$LAG < 5" | bc -l) )); then
    echo "✅ CDC lag is < 5 seconds. Proceeding..."
    break
  fi
  
  sleep 10
done
```

**Nếu lag không giảm sau 20 phút → Kích hoạt ROLLBACK.**

### Bước 4: Final Validation (Xác Minh Cuối Cùng)

```sql
-- Chạy trên SOURCE và TARGET, so sánh kết quả
-- Kiểm tra 5 bảng quan trọng nhất

-- Source (MySQL):
SELECT 
  'orders' AS tbl, COUNT(*) AS cnt FROM orders
UNION ALL SELECT 'users', COUNT(*) FROM users
UNION ALL SELECT 'products', COUNT(*) FROM products
UNION ALL SELECT 'payments', COUNT(*) FROM payments
UNION ALL SELECT 'inventory', COUNT(*) FROM inventory;

-- Target (Aurora MySQL / PostgreSQL):
-- Chạy query tương tự, so sánh số rows

-- ✅ PASS: Sai số ≤ 0.01% (vài rows cuối cùng đang được process)
-- ❌ FAIL: Sai số > 0.1% → Điều tra → Xem xét Rollback
```

### Bước 5: Update Connection Strings

```bash
# Approach 1: Environment Variables
export DB_HOST="aurora-cluster.cluster-abc123.us-east-1.rds.amazonaws.com"
export DB_PORT="5432"
export DB_NAME="myapp"
export DB_USER="app_user"
export DB_PASS="$(aws secretsmanager get-secret-value --secret-id prod/myapp/db --query SecretString --output text | jq -r .password)"

# Approach 2: Update AWS Systems Manager Parameter Store
aws ssm put-parameter \
  --name "/prod/myapp/db_host" \
  --value "aurora-cluster.cluster-abc123.us-east-1.rds.amazonaws.com" \
  --overwrite

# Approach 3: DNS (nếu dùng internal DNS alias)
# Update Route53 record:
aws route53 change-resource-record-sets \
  --hosted-zone-id ZONEID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "db.production.internal.",
        "Type": "CNAME",
        "TTL": 30,
        "ResourceRecords": [{"Value": "aurora-cluster.cluster-abc123.us-east-1.rds.amazonaws.com"}]
      }
    }]
  }'
```

### Bước 6: Restart Application Servers

```bash
# Graceful restart để pick up new DB connection strings
# Kubernetes:
kubectl rollout restart deployment/myapp -n production
kubectl rollout status deployment/myapp -n production --timeout=300s

# Systemd:
systemctl restart myapp

# Docker Compose:
docker-compose up -d --force-recreate app

# Sau restart, verify connections đến target:
# Check application logs
kubectl logs -f deployment/myapp -n production | grep "Database connected"
```

### Bước 7: Smoke Tests (Kiểm Thử Khói)

```bash
# Chạy smoke test script (kịch bản kiểm thử nhanh)
# Smoke test = kiểm tra các chức năng QUAN TRỌNG NHẤT chạy được

# Test 1: Health check endpoint
curl -f https://api.myapp.com/health || { echo "❌ Health check FAILED"; exit 1; }

# Test 2: Database read
curl -f "https://api.myapp.com/api/users/1" || { echo "❌ User read FAILED"; exit 1; }

# Test 3: Database write
curl -f -X POST "https://api.myapp.com/api/orders" \
  -H "Content-Type: application/json" \
  -d '{"user_id": 1, "product_id": 999, "quantity": 1}' \
  || { echo "❌ Order create FAILED"; exit 1; }

# Test 4: Critical business flow
python3 scripts/smoke_test_checkout.py || { echo "❌ Checkout flow FAILED"; exit 1; }

echo "✅ All smoke tests PASSED"
```

### Bước 8: Go/No-Go Decision & Open Traffic

```
War room vote:
  DBA Lead:       GO / NO-GO
  App Lead:       GO / NO-GO  
  Infra Lead:     GO / NO-GO
  QA Lead:        GO / NO-GO

Nếu tất cả GO:
  → Tắt maintenance mode
  → Mở traffic cho users
  → Post "Maintenance complete" trên status page

Nếu bất kỳ NO-GO:
  → Kích hoạt ROLLBACK ngay lập tức
  → Không tranh luận — safety first
```

---

## 🔴 Rollback Plan (Kế Hoạch Quay Lại)

### Khi Nào Rollback?

```
Rollback NGAY LẬP TỨC khi:
  □ Smoke tests FAIL sau khi switch
  □ Error rate (tỷ lệ lỗi) tăng > 5% so với baseline
  □ p99 latency tăng > 3x so với baseline
  □ Dữ liệu bị mất hoặc corrupt
  □ Application không kết nối được đến target DB

Rollback CÓ THỂ sau khi điều tra:
  □ Một số features không hoạt động nhưng core còn ok
  □ Performance degraded nhưng vẫn acceptable
```

### RTO cho Rollback (Recovery Time Objective — Mục Tiêu Thời Gian Phục Hồi)

**Target rollback time: < 5 phút**

### Rollback Steps

```bash
# ============================================================
# ROLLBACK SCRIPT — Chạy ngay khi quyết định rollback
# ============================================================

echo "🔴 INITIATING ROLLBACK at $(date)"

# Bước R1: Bật maintenance mode (tránh additional writes vào target)
# [Tùy thuộc cách bạn bật ở Bước 2]

# Bước R2: Revert connection strings về source DB
export DB_HOST="source-mysql.on-premises.internal"  # Hoặc original value
# Hoặc revert SSM Parameter:
aws ssm put-parameter \
  --name "/prod/myapp/db_host" \
  --value "source-mysql.on-premises.internal" \
  --overwrite

# Bước R3: Restart application về source DB
kubectl rollout restart deployment/myapp -n production
kubectl rollout status deployment/myapp -n production --timeout=120s

# Bước R4: Disable read_only trên source (nếu đã set)
mysql -h source-db -u admin -p -e "SET GLOBAL read_only = OFF; SET GLOBAL super_read_only = OFF;"

# Bước R5: Run smoke tests trên source
curl -f https://api.myapp.com/health && echo "✅ Back on source DB"

# Bước R6: Thông báo
echo "🟡 ROLLBACK COMPLETE at $(date)"
echo "Application is back on SOURCE database"
echo "Post-mortem (rà soát sau sự cố) scheduled for tomorrow 10 AM"
```

### Xử Lý Data Divergence Sau Rollback

```
Vấn đề:
  Nếu application đã ghi vào target DB (vài phút trước rollback)
  rồi rollback về source → target có data mới hơn source
  
  Ví dụ:
    User tạo order #5000 trên target DB (T+01:05)
    Rollback về source DB (T+01:10)
    Source không có order #5000

Giải pháp:
  1. Extract (trích xuất) các writes xảy ra sau cutover từ target:
     SELECT * FROM orders WHERE created_at > '[cutover_time]';
  
  2. Manually apply vào source (nếu quan trọng — orders, payments)
  
  3. Notify affected users (thông báo người dùng bị ảnh hưởng)
  
  4. Compensation logic (logic bù đắp):
     - Cancel duplicate orders
     - Refund if payment processed on target
     
  5. Document everything cho post-mortem
```

---

## 📊 Post-Cutover Validation (Xác Minh Sau Cắt Chuyển)

### T+1 giờ: Immediate Monitoring (Theo Dõi Ngay)

```
CloudWatch Dashboard cần check mỗi 5 phút:
  □ DatabaseConnections — không quá 80% max
  □ CPUUtilization — < 80% sustained
  □ ReadLatency / WriteLatency — so với baseline
  □ FreeStorageSpace — > 20%
  
Application Metrics:
  □ Error rate (tỷ lệ lỗi) — < baseline + 1%
  □ p95, p99 latency — < 2x baseline
  □ Active user count — bình thường
  □ Business metrics (orders/min, signups/min)
```

### T+24 giờ: Extended Monitoring

```
□ Review slow query log trên target (queries chưa được optimize?)
□ Check cho missing indexes (chỉ mục bị thiếu) bằng cách nhìn vào slow queries
□ Review CloudWatch RDS Performance Insights — top wait events
□ Verify automated backup đã chạy thành công
□ Check CDC task trên DMS (nếu vẫn để chạy thêm an toàn)
```

### T+72 giờ: Stability Confirmation (Xác Nhận Ổn Định)

```
□ 3 ngày ổn định → Confirm migration SUCCESS
□ Announce internally (thông báo nội bộ)
□ Schedule DMS task stop (không cần CDC nữa)
□ Schedule source DB decommission (ngừng hoạt động) sau 2 tuần thêm
□ Document lessons learned (bài học rút ra)
□ Update runbook cho lần migration tiếp theo
```

---

## 💡 Lessons Learned từ Thực Tế

### Thường Gặp Nhất Trong Migration Projects

**1. Index thiếu trên target làm query chậm**
```
Vấn đề: DMS copy data nhưng không tạo secondary indexes đúng lúc
        → Sau cutover, queries bị table scan thay vì index scan
        
Giải pháp: Tạo indexes TRƯỚC khi cutover
           Verify indexes với SHOW INDEX / \d+ table_name
           Chạy EXPLAIN ANALYZE trên top 10 queries quan trọng nhất
```

**2. Connection pool exhaustion sau cutover**
```
Vấn đề: Target DB có max_connections khác source
        Application tạo quá nhiều connections → "Too many connections"
        
Giải pháp: Kiểm tra max_connections trên target TRƯỚC cutover
           Dùng RDS Proxy (Proxy RDS) để pool connections
           Cấu hình connection pool size trong application
```

**3. Character encoding mismatch**
```
Vấn đề: Source MySQL dùng utf8 (3-byte), target Aurora dùng utf8mb4 (4-byte)
        Emoji và một số Unicode characters bị lỗi hoặc mất
        
Giải pháp: Đảm bảo DMS endpoint character set setting đúng
           Test với emoji và special characters trong smoke tests
```

**4. Timezone differences**
```
Vấn đề: Source ở UTC+7, target ở UTC
        DATETIME columns bị lệch 7 giờ
        
Giải pháp: Chuẩn hóa tất cả timestamps về UTC trước migration
           Verify timezone config trong target DB: SET time_zone = 'UTC'
```

**5. Auto-increment gaps sau full load**
```
Vấn đề: Source có rows bị delete, auto_increment có gaps
        Target full load copy gaps này
        New inserts sau cutover có thể conflict
        
Giải pháp: Verify AUTO_INCREMENT giá trị trên target = source
           MySQL: SELECT AUTO_INCREMENT FROM information_schema.tables WHERE table_name = 'orders';
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Mô tả các bước trong một database cutover?**

> Cutover bao gồm: (1) Bật maintenance mode trên ứng dụng để dừng writes vào source; (2) Chờ DMS CDC lag về 0 để đảm bảo target đồng bộ hoàn toàn; (3) Final row count validation để confirm data integrity; (4) Update connection strings của ứng dụng sang target; (5) Restart application servers; (6) Chạy smoke tests để verify các chức năng quan trọng; (7) Go/No-Go decision với tất cả leads; (8) Mở traffic. Toàn bộ quá trình thường mất 30-60 phút, downtime thực tế là từ bước 1 đến bước 7.

**Q: Khi nào bạn nên rollback và quy trình rollback như thế nào?**

> Rollback ngay lập tức khi: smoke tests fail sau switch, error rate tăng > 5%, latency tăng > 3x, hoặc data corruption phát hiện. Quy trình rollback: (1) bật maintenance mode ngay, (2) revert connection strings về source DB, (3) restart applications, (4) disable read_only mode trên source nếu đã bật, (5) verify smoke tests trên source pass, (6) thông báo stakeholders. RTO cho rollback cần < 5 phút. Sau rollback, cần extract và manually apply các writes đã xảy ra trên target trước khi rollback để tránh mất data.

**Q: Làm thế nào để validate dữ liệu sau migration?**

> Có 3 lớp validation: (1) Row count validation — SELECT COUNT(*) cho các bảng lớn nhất, so sánh source vs target với sai số < 0.1%; (2) Data sampling — lấy mẫu 1% dữ liệu ngẫu nhiên và so sánh chi tiết; (3) Business logic validation — chạy application integration tests và smoke tests kiểm tra các business flows quan trọng như checkout, payment. Ngoài ra, DMS có tích hợp validation task tự động so sánh row hashes. Quan trọng nhất là validation các bảng business-critical như orders, payments, users trước bảng phụ.

---

**Liên Kết:**
- [README.md](./README.md) — Tổng quan migration
- [1-dms-overview.md](./1-dms-overview.md) — AWS DMS chi tiết
- [3-cdc-online-migration.md](./3-cdc-online-migration.md) — CDC và zero-downtime
- [4-migration-strategies.md](./4-migration-strategies.md) — Chiến lược migration

**Cập Nhật Lần Cuối:** 2026-05-15
