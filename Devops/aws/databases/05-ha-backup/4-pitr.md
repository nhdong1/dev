# PITR — Point-in-Time Recovery (Khôi Phục Theo Thời Điểm)

> PITR (Point-in-Time Recovery — Khôi Phục Theo Thời Điểm) cho phép restore database về bất kỳ thời điểm nào trong khoảng retention period, với độ chính xác đến 5 phút (RDS) hoặc 1 giây (Aurora). Đây là công cụ cứu dữ liệu quan trọng nhất khi xảy ra human error (lỗi do con người) như xóa nhầm bảng, chạy nhầm UPDATE/DELETE.

---

## 🎯 PITR Là Gì và Hoạt Động Như Thế Nào?

### Cơ Chế Kỹ Thuật

```
PITR = Full Backup + Transaction Logs Replay

Timeline phục hồi:
                    PITR Target
                    (Thời Điểm Cần Phục Hồi)
                          │
                          ▼
──────●────────────────────●────────────────────────► Thời gian
      │                    │
  Daily Full            Transaction
  Backup                Logs Apply
  (Backup Đầy Đủ)      (Áp Dụng Nhật Ký)
  
Quy Trình:
  1. AWS restore full backup gần nhất TRƯỚC thời điểm target
  2. Replay transaction logs từ full backup đến target time
  3. Tạo DB instance mới với dữ liệu tại target time
```

### Độ Chính Xác PITR

```
RDS MySQL/PostgreSQL/MariaDB:
  Backup type: Daily full + binlogs upload mỗi 5 phút
  PITR granularity: 5 phút
  Nghĩa là: Restore về 14:30:00 → Thực tế có thể là 14:25:xx hoặc 14:30:xx

Aurora MySQL/PostgreSQL:
  Backup type: Continuous (liên tục) + redo logs real-time
  PITR granularity: 1 giây
  Nghĩa là: Restore về 14:30:22 → Chính xác đến giây

Khuyến nghị: Dùng Aurora nếu PITR granularity quan trọng
```

---

## 🔧 Thực Hiện PITR

### Qua AWS Console

```
RDS Console → Databases → Chọn instance
→ Actions → Restore to point in time
→ Restore time: Chọn "Custom date and time"
  Date: 2026-05-15
  Time: 14:30 (UTC)
→ DB instance identifier: "my-db-pitr-20260515-1430"
→ DB instance class: db.r6g.large (chọn class phù hợp)
→ Restore DB instance
```

### Qua AWS CLI

```bash
# PITR về một thời điểm cụ thể
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier my-production-db \
  --target-db-instance-identifier my-db-pitr-20260515-1430 \
  --restore-time "2026-05-15T14:30:00Z" \    # UTC timezone
  --db-instance-class db.r6g.large \
  --multi-az \
  --publicly-accessible false \
  --vpc-security-group-ids sg-xxxxxxxx \
  --db-subnet-group-name my-db-subnet-group

# Đợi instance available
aws rds wait db-instance-available \
  --db-instance-identifier my-db-pitr-20260515-1430

# Lấy endpoint để kết nối
ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier my-db-pitr-20260515-1430 \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text)
echo "PITR instance endpoint: $ENDPOINT"
```

### PITR Về Thời Điểm Gần Nhất Có Thể

```bash
# Tìm LatestRestorableTime (thời điểm mới nhất có thể restore về)
LATEST_TIME=$(aws rds describe-db-instances \
  --db-instance-identifier my-production-db \
  --query 'DBInstances[0].LatestRestorableTime' \
  --output text)
echo "Thời điểm mới nhất có thể restore: $LATEST_TIME"

# Restore về thời điểm mới nhất
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier my-production-db \
  --target-db-instance-identifier my-db-latest-restore \
  --use-latest-restorable-time    # Không cần chỉ định --restore-time
```

---

## 🚨 Kịch Bản Sử Dụng PITR Thực Tế

### Kịch Bản 1 — Xóa Nhầm Bảng (DROP TABLE)

```
Timeline Sự Cố:
  14:00 — Developer A chạy nhầm: DROP TABLE orders;
  14:05 — Hệ thống bắt đầu báo lỗi "Table not found"
  14:10 — Alert (cảnh báo) được gửi đến on-call engineer
  14:15 — Xác nhận dữ liệu bị mất, bắt đầu quy trình phục hồi

Quy Trình PITR:

BƯỚC 1: Xác định thời điểm cần restore (trước 14:00)
  Target time: 2026-05-15T13:55:00Z  (5 phút trước sự cố)

BƯỚC 2: Tạo instance PITR để kiểm tra (không thay thế production ngay)
  aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier production-db \
    --target-db-instance-identifier incident-restore-1355 \
    --restore-time "2026-05-15T13:55:00Z"

BƯỚC 3: Verify dữ liệu trong instance PITR
  mysql -h <pitr-endpoint> -e "SELECT COUNT(*) FROM orders;"
  → Kết quả: 1,245,678 rows ✅ (dữ liệu orders còn nguyên)

BƯỚC 4: Options để phục hồi production:
  Option A (Nhanh, ít an toàn): Đổi tên instance PITR thành production
  Option B (An toàn hơn): Export bảng orders từ PITR → Import vào production
  
  Chọn Option B nếu cần minimize downtime:
  
  mysqldump -h <pitr-endpoint> myapp orders > orders_backup.sql
  mysql -h <production-endpoint> myapp < orders_backup.sql
  
  Thời gian downtime: 0 (chỉ missing orders từ 13:55-14:00)

BƯỚC 5: Verify production ổn định, cleanup PITR instance
  aws rds delete-db-instance \
    --db-instance-identifier incident-restore-1355 \
    --skip-final-snapshot
```

### Kịch Bản 2 — UPDATE Nhầm Hàng Loạt (Mass Update Without WHERE)

```sql
-- Developer đã chạy câu lệnh tai hại này:
UPDATE users SET subscription_tier = 'free';
-- Mất WHERE clause → Tất cả 500,000 users bị downgrade về free!
```

```bash
# Thời điểm sự cố: 10:30 UTC
# Phát hiện lúc: 10:45 UTC
# Target restore: 10:29 UTC (1 phút trước sự cố)

# Bước 1: Tạo PITR instance
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-userdb \
  --target-db-instance-identifier pitr-userdb-pre-incident \
  --restore-time "2026-05-15T10:29:00Z"

# Bước 2: Export chỉ cột subscription_tier từ PITR (không cần toàn bộ table)
mysql -h <pitr-endpoint> -u admin -p myapp -e \
  "SELECT user_id, subscription_tier FROM users" > user_tiers_backup.csv

# Bước 3: Update production từ backup data
mysql -h <prod-endpoint> -u admin -p myapp << 'EOF'
CREATE TEMPORARY TABLE user_tiers_restore (
  user_id INT,
  subscription_tier VARCHAR(50)
);

LOAD DATA INFILE '/tmp/user_tiers_backup.csv' 
INTO TABLE user_tiers_restore
FIELDS TERMINATED BY ',' LINES TERMINATED BY '\n';

UPDATE users u 
JOIN user_tiers_restore r ON u.user_id = r.user_id
SET u.subscription_tier = r.subscription_tier;

SELECT COUNT(*) as updated_rows FROM users WHERE subscription_tier != 'free';
EOF
```

### Kịch Bản 3 — Data Corruption (Dữ Liệu Bị Hỏng) Từ Bug

```
Kịch Bản:
  - Bug trong code payment service đã chạy 2 giờ
  - Đã corrupt (làm hỏng) ~5,000 order records
  - Bug được fix và deploy lúc 16:00
  - Cần restore dữ liệu 5,000 orders về trạng thái trước khi bug

Approach (Cách Tiếp Cận):
  1. PITR về 14:00 (trước khi bug bắt đầu gây hại)
  2. Export 5,000 affected orders từ PITR instance
  3. Reconcile (đối chiếu) với transactions xảy ra sau 14:00
  4. Selectively restore (khôi phục có chọn lọc) từng order

Lưu Ý: Không phải lúc nào cũng có thể "simply" PITR về thời điểm cũ
khi hệ thống đã tiếp tục nhận giao dịch mới sau sự cố.
Cần kết hợp PITR với manual reconciliation (đối chiếu thủ công).
```

---

## ⏱️ Thời Gian Thực Hiện PITR

### Ước Tính Thời Gian

```
Thời gian PITR phụ thuộc vào:
  1. Kích thước database (Database Size)
  2. Khoảng cách thời gian từ full backup đến target time
     (Nhiều logs cần replay → Lâu hơn)
  3. DB instance class (Loại instance) — Mạnh hơn → Nhanh hơn

Ước Tính Thực Tế (Rough Estimates):
  DB 100 GB, restore từ 4 giờ trước: ~15-30 phút
  DB 500 GB, restore từ 12 giờ trước: ~60-90 phút
  DB 1 TB, restore từ 24 giờ trước: ~2-4 giờ

Aurora PITR:
  Nhanh hơn đáng kể vì backup là incremental, liên tục
  DB 1 TB: ~30-60 phút bất kể target time
```

### Tính RTO Thực Tế Khi Dùng PITR

```
Tổng RTO = PITR time + Verify time + Cutover time

PITR time:    30-90 phút (tùy DB size)
Verify time:  15-30 phút (kiểm tra dữ liệu)
Cutover time: 5-15 phút (update DNS/connection)

Tổng:         50-135 phút

→ Nếu RTO < 1 giờ, PITR KHÔNG phù hợp — cần Multi-AZ hoặc Aurora
→ PITR phù hợp với RTO > 2 giờ
```

---

## 🔍 Monitoring PITR Capability (Khả Năng PITR)

### Kiểm Tra PITR Window Hiện Tại

```bash
# Kiểm tra PITR window của DB instance
aws rds describe-db-instances \
  --db-instance-identifier my-production-db \
  --query 'DBInstances[0].{
    EarliestRestorableTime: RestoreWindow.EarliestTime,
    LatestRestorableTime: LatestRestorableTime,
    BackupRetentionDays: BackupRetentionPeriod
  }'

# Output mong đợi:
# {
#   "EarliestRestorableTime": "2026-05-08T02:00:00.000Z",
#   "LatestRestorableTime": "2026-05-15T14:25:00.000Z",
#   "BackupRetentionDays": 7
# }
```

### CloudWatch Alarm Cho PITR Health

```bash
# Alert khi LatestRestorableTime cách thời điểm hiện tại > 30 phút
# (Có thể transaction logs không được upload đúng cách)
aws cloudwatch put-metric-alarm \
  --alarm-name "RDS-PITR-Staleness-Alert" \
  --alarm-description "PITR may be compromised — latest restorable time is stale" \
  --metric-name "OldestRestorableTime" \
  --namespace "Custom/RDS" \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 1800 \    # 30 phút = 1800 giây
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789:critical-alerts
```

---

## 📋 PITR Runbook (Sổ Tay Vận Hành PITR)

### Runbook Khi Cần PITR Khẩn Cấp

```markdown
## PITR Emergency Runbook (Sổ Tay PITR Khẩn Cấp)
**Thời Gian Ước Tính: 90-120 phút**

### P0 — Ngay Lập Tức (0-5 phút)
- [ ] Thông báo Incident Commander (Người Chỉ Huy Sự Cố)
- [ ] Xác định thời điểm sự cố xảy ra (hỏi developer, check logs)
- [ ] Set target restore time = 5-10 phút TRƯỚC thời điểm sự cố
- [ ] KHÔNG xóa hay modify gì trên production instance

### P1 — Bắt Đầu PITR (5-10 phút)
- [ ] Verify backup window và LatestRestorableTime còn valid
- [ ] Trigger PITR restore vào instance mới (KHÔNG replace production)
- [ ] Document restore time và instance name

Command:
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier PROD_INSTANCE_NAME \
  --target-db-instance-identifier pitr-YYYYMMDD-HHMM \
  --restore-time YYYY-MM-DDTHH:MM:SSZ \
  --db-instance-class db.r6g.large \
  --no-multi-az \                          # Tiết kiệm thời gian, test trước
  --db-subnet-group-name SUBNET_GROUP_NAME \
  --vpc-security-group-ids SECURITY_GROUP_ID

### P2 — Verify Dữ Liệu (30-60 phút sau khi restore xong)
- [ ] Connect vào PITR instance
- [ ] Verify các bảng bị ảnh hưởng có đúng data không
- [ ] Check row counts so với production trước sự cố
- [ ] Confirm với business stakeholders rằng dữ liệu đúng

### P3 — Phục Hồi Production
Option A (Thay thế hoàn toàn):
- [ ] Rename production instance cũ → "old-prod-incident"
- [ ] Rename PITR instance → production name
- [ ] Update DNS CNAME nếu dùng custom endpoint
- [ ] Test connectivity từ application
- [ ] Thông báo all-clear

Option B (Surgical restore — Khôi phục có chọn lọc):
- [ ] Export bảng bị ảnh hưởng từ PITR instance
- [ ] Import vào production (trong maintenance window nếu có thể)
- [ ] Verify transaction count sau import

### P4 — Cleanup & Post-Mortem
- [ ] Xóa PITR instance sau 48 giờ (khi chắc chắn ổn)
- [ ] Viết Post-Mortem report
- [ ] Cập nhật runbook nếu có bài học
- [ ] Review backup retention policy
```

---

## ⚠️ Giới Hạn và Lưu Ý Của PITR

### Những Gì PITR KHÔNG Thể Làm

```
❌ PITR KHÔNG thể:
  - Restore về thời điểm trước khi backup retention window
    (Ví dụ: Retention = 7 ngày → Không thể restore về 10 ngày trước)
  
  - Restore ghi đè lên DB instance đang chạy
    (Luôn tạo instance mới)
  
  - Restore chỉ một số bảng cụ thể (phải restore toàn bộ DB)
  
  - Restore nếu automated backup bị tắt (retention = 0)
  
  - Restore về đúng giây nếu dùng RDS (chỉ Aurora mới có 1 giây granularity)

✅ Nếu cần restore một bảng:
  1. PITR ra instance mới
  2. mysqldump / pg_dump bảng cần thiết
  3. Import vào production
```

### PITR Cho Aurora

```
Aurora PITR có ưu điểm:
  - Granularity 1 giây (thay vì 5 phút của RDS)
  - Backup continuous (liên tục) — không cần backup window
  - Restore nhanh hơn RDS (do incremental backup)
  - Retain period lên đến 35 ngày (giống RDS)

Điểm khác của Aurora PITR:
  aws rds restore-db-cluster-to-point-in-time \   # Restore cluster, không phải instance
    --db-cluster-identifier my-aurora-cluster-pitr \
    --restore-type full-copy \
    --source-db-cluster-identifier my-production-aurora \
    --restore-to-time "2026-05-15T14:30:00Z"
    
  Sau đó tạo instance trong cluster mới:
  aws rds create-db-instance \
    --db-cluster-identifier my-aurora-cluster-pitr \
    --db-instance-identifier my-aurora-pitr-instance \
    --db-instance-class db.r6g.large \
    --engine aurora-mysql
```

---

## 🎯 Câu Hỏi Phỏng Vấn Về PITR

**Q: PITR hoạt động như thế nào ở cấp độ kỹ thuật?**
> PITR kết hợp hai thứ: Full backup hàng ngày (lưu trên S3) và transaction logs (binlogs cho MySQL, WAL cho PostgreSQL) được upload lên S3 liên tục (mỗi 5 phút với RDS, real-time với Aurora). Khi restore, AWS lấy full backup gần nhất trước target time, rồi replay transaction logs từ backup đó đến đúng target time. Kết quả là một DB instance mới với trạng thái dữ liệu tại target time.

**Q: Developer vô tình DROP TABLE một bảng quan trọng. Quy trình PITR như thế nào?**
> Đầu tiên xác định chính xác thời điểm xảy ra DROP TABLE từ logs. Sau đó trigger PITR về 5-10 phút trước thời điểm đó vào một instance mới (không thay thế production). Verify dữ liệu trong PITR instance để xác nhận bảng còn đó. Sau đó chọn cách phục hồi: hoặc thay thế hoàn toàn production bằng PITR instance, hoặc export bảng từ PITR và import vào production (để không mất các thay đổi khác sau thời điểm DROP).

**Q: PITR vs Manual Snapshot — khi nào dùng cái nào?**
> PITR dùng khi cần khôi phục về một thời điểm chính xác trong quá khứ, thường khi có human error như DELETE/UPDATE/DROP nhầm. Manual Snapshot dùng khi muốn lưu trạng thái DB tại một milestone cụ thể (trước migration, trước deployment lớn) và có thể cần restore về đúng thời điểm đó. PITR linh hoạt hơn (bất kỳ thời điểm nào trong window) nhưng không thể restore trước retention window.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
