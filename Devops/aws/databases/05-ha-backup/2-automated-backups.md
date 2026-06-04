# Automated Backups — Sao Lưu Tự Động Trên RDS & Aurora

> Automated Backup (Sao Lưu Tự Động) là tính năng được bật mặc định trên Amazon RDS và Aurora, tự động tạo full backup hàng ngày kết hợp với transaction logs (nhật ký giao dịch) liên tục. Đây là nền tảng cho PITR (Point-in-Time Recovery — Khôi Phục Theo Thời Điểm) và là lớp bảo vệ dữ liệu đầu tiên trong mọi chiến lược DR (Disaster Recovery — Khôi Phục Thảm Họa).

---

## 🎯 Automated Backup Là Gì?

### Hai Thành Phần Cốt Lõi

```
┌─────────────────────────────────────────────────────────────────┐
│                    Automated Backup = A + B                     │
│                                                                  │
│  A. Daily Full Backup (Sao Lưu Đầy Đủ Hàng Ngày)              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ • Snapshot toàn bộ storage volume vào S3                 │  │
│  │ • Thực hiện trong Backup Window (Cửa Sổ Sao Lưu)        │  │
│  │ • Tạo nền tảng cho restore                               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  B. Transaction Logs (Nhật Ký Giao Dịch — Liên Tục)           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ • Upload lên S3 mỗi 5 phút                              │  │
│  │ • Ghi lại mọi thay đổi dữ liệu                          │  │
│  │ • Cho phép PITR về bất kỳ thời điểm nào trong window    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Kết hợp: Full Backup + Logs → Restore tại bất kỳ thời điểm   │
└─────────────────────────────────────────────────────────────────┘
```

### Retention Period (Thời Gian Lưu Giữ)

```
Retention Period (Thời Gian Lưu Giữ):
  - Tối thiểu: 0 ngày (tắt automated backup — KHÔNG khuyến nghị)
  - Mặc định:  7 ngày
  - Tối đa:    35 ngày
  - Sau khi hết retention: AWS tự động xóa backup cũ

Ví dụ với retention = 7 ngày:
  Hôm nay (15/5)   ←────── 7 ngày ──────→   (8/5)
       │                                        │
       ▼                                        ▼
  Backup mới nhất                     Backup cũ nhất còn giữ
  
  PITR có thể restore về bất kỳ thời điểm nào trong 7 ngày qua
```

---

## ⚙️ Cấu Hình Automated Backup

### Backup Window (Cửa Sổ Sao Lưu)

```
Backup Window là khoảng thời gian AWS thực hiện full snapshot hàng ngày.

Best Practices (Thực Hành Tốt Nhất):
  - Chọn thời điểm traffic thấp nhất (low-traffic window)
  - Không trùng với Maintenance Window (Cửa Sổ Bảo Trì)
  - Nên dùng: 02:00-03:00 AM theo giờ địa phương

Ảnh hưởng trong quá trình backup:
  - Multi-AZ: Backup thực hiện từ Standby → Không ảnh hưởng Primary
  - Single-AZ: Có thể tăng I/O latency (độ trễ vào/ra) nhẹ
```

### Cấu Hình Qua AWS Console

```
RDS Console → Instance → Modify → Backup:
  ✓ Enable automatic backups
  Backup retention period: [7] days
  Backup window: [02:00] UTC - [03:00] UTC
  
  → Apply immediately hoặc Apply during next maintenance window
```

### Cấu Hình Qua AWS CLI (Command Line Interface — Giao Diện Dòng Lệnh)

```bash
# Tạo RDS instance với automated backup
aws rds create-db-instance \
  --db-instance-identifier my-production-db \
  --db-instance-class db.r6g.large \
  --engine mysql \
  --engine-version 8.0 \
  --master-username admin \
  --master-user-password "SecurePassword123!" \
  --allocated-storage 100 \
  --backup-retention-period 14 \        # Giữ 14 ngày
  --preferred-backup-window "02:00-03:00" \  # 2-3 AM UTC
  --multi-az \                           # Bật Multi-AZ
  --storage-encrypted                    # Mã hóa storage

# Thay đổi retention period cho instance đã tồn tại
aws rds modify-db-instance \
  --db-instance-identifier my-production-db \
  --backup-retention-period 14 \
  --apply-immediately
```

### Cấu Hình Qua Terraform (Infrastructure as Code — Hạ Tầng Dưới Dạng Mã)

```hcl
resource "aws_db_instance" "production" {
  identifier        = "my-production-db"
  engine            = "mysql"
  engine_version    = "8.0"
  instance_class    = "db.r6g.large"
  
  # Backup Configuration (Cấu Hình Sao Lưu)
  backup_retention_period = 14          # Giữ 14 ngày
  backup_window           = "02:00-03:00"  # UTC
  
  # Encryption (Mã Hóa)
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn
  
  # Maintenance (Bảo Trì)
  maintenance_window = "sun:03:00-sun:04:00"  # Sau backup window
  
  # Không xóa backup khi xóa instance
  delete_automated_backups = false
  deletion_protection      = true
  
  tags = {
    Environment = "production"
    Backup      = "14-day-retention"
  }
}
```

---

## 📊 Automated Backup Cho Từng Engine

### RDS MySQL/PostgreSQL/MariaDB

```
Loại Backup: Full storage snapshot + binlogs (binary logs)
Tần Suất Full Backup: 1 lần/ngày trong backup window
Transaction Logs: Upload mỗi 5 phút
PITR Granularity: 5 phút (độ chính xác 5 phút)
Storage: S3 (managed by AWS, không hiển thị trong S3 console của bạn)
Chi Phí: Backup storage miễn phí đến kích thước bằng provisioned storage
```

### RDS Oracle/SQL Server

```
RDS Oracle:
  - Backup: RMAN (Recovery Manager) + redo logs
  - PITR: Supported (Được hỗ trợ)
  - Multi-AZ: Supported

RDS SQL Server:
  - Backup: Native SQL Server backup + transaction log shipping
  - PITR: Supported
  - Multi-AZ: Sử dụng SQL Server Mirroring hoặc Always On

Lưu ý: SQL Server backup window có thể dài hơn MySQL/PostgreSQL
```

### Aurora MySQL/PostgreSQL

```
Aurora Backup khác với RDS thông thường:
  - Continuous backup (Sao lưu liên tục) vào S3
  - Không cần backup window (không ảnh hưởng đến I/O)
  - Automatic, incremental backup
  - PITR granularity: 1 giây (không phải 5 phút như RDS)
  
Lý do: Aurora storage là distributed (phân tán), mỗi thay đổi
được ghi vào S3 liên tục — không cần full snapshot định kỳ.
```

---

## 🔍 Theo Dõi & Monitoring Automated Backup

### CloudWatch Metrics (Số Liệu CloudWatch) Quan Trọng

```
Metric (Số Liệu):      FreeStorageSpace
Unit (Đơn Vị):         Bytes
Ý Nghĩa:              Dung lượng storage còn trống
Cảnh Báo Nên Đặt:     Khi < 20% total storage
Tại Sao Quan Trọng:   Backup cần đủ space để chạy

Metric:      BackupRetentionPeriod
Ý Nghĩa:    Retention period hiện tại (ngày)
Dùng Để:    Audit compliance (Kiểm toán tuân thủ)

Metric:      LatestRestorableTime
Ý Nghĩa:    Thời điểm mới nhất có thể PITR về
Theo Dõi:   Nên cách hiện tại < 5 phút (nếu không → vấn đề logs)
```

### Kiểm Tra Backup Status Qua CLI

```bash
# Liệt kê tất cả automated backups của một instance
aws rds describe-db-instance-automated-backups \
  --db-instance-identifier my-production-db \
  --query 'DBInstanceAutomatedBackups[*].{
    DBInstanceId: DBInstanceIdentifier,
    Status: Status,
    From: RestoreWindow.EarliestTime,
    To: RestoreWindow.LatestTime,
    Region: DBInstanceArn
  }' \
  --output table

# Kiểm tra LatestRestorableTime (thời điểm gần nhất có thể restore về)
aws rds describe-db-instances \
  --db-instance-identifier my-production-db \
  --query 'DBInstances[0].LatestRestorableTime'
```

### Tạo CloudWatch Alarm (Cảnh Báo) Cho Backup

```bash
# Cảnh báo khi không có backup trong 24 giờ (backup failure)
aws cloudwatch put-metric-alarm \
  --alarm-name "RDS-Backup-Age-Alert" \
  --alarm-description "Alert when RDS backup is older than 24h" \
  --metric-name "TimeSinceLastBackup" \
  --namespace "AWS/RDS" \
  --dimensions Name=DBInstanceIdentifier,Value=my-production-db \
  --period 3600 \
  --evaluation-periods 1 \
  --threshold 86400 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789:ops-alerts
```

---

## 🔐 Backup Encryption (Mã Hóa Backup)

### Quy Tắc Mã Hóa

```
Quy Tắc 1: Nếu DB instance được mã hóa → Backup tự động được mã hóa
Quy Tắc 2: Nếu DB instance KHÔNG mã hóa → Backup KHÔNG được mã hóa
Quy Tắc 3: Không thể bật mã hóa trên instance đang chạy (unencrypted)

Cách bật mã hóa cho DB đang chạy (không có mã hóa):
  1. Tạo unencrypted snapshot
  2. Copy snapshot với encryption enabled (chỉ định KMS key)
  3. Restore từ encrypted snapshot → Tạo encrypted instance mới
  4. Update DNS/connection string để trỏ vào instance mới
  5. Xóa instance cũ sau khi verify

Thời gian dừng (downtime): Khoảng 15-60 phút tùy kích thước DB
```

### KMS Key Management (Quản Lý Khóa KMS)

```bash
# Tạo KMS key (khóa mã hóa) cho RDS backup
aws kms create-key \
  --description "KMS key for RDS backup encryption" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT

# Gán alias (tên dễ nhớ) cho key
aws kms create-alias \
  --alias-name alias/rds-backup-key \
  --target-key-id <key-id-from-above>

# Tạo DB instance với encryption sử dụng key vừa tạo
aws rds create-db-instance \
  --db-instance-identifier my-encrypted-db \
  --kms-key-id alias/rds-backup-key \
  --storage-encrypted \
  # ... other params
```

---

## 🌐 Cross-Region Backup Replication (Sao Chép Backup Xuyên Vùng)

### Tại Sao Cần Cross-Region Backup?

```
Scenario (Kịch Bản):
  Nếu toàn bộ AWS region us-east-1 bị sự cố (hiếm nhưng đã xảy ra):
  - Backup trong us-east-1 → Không truy cập được
  - Cần backup ở region khác (us-west-2, eu-west-1, ...)

Giải pháp: Enable cross-region automated backup replication
```

### Bật Cross-Region Backup Replication

```bash
# Bật cross-region automated backup replication
aws rds start-export-task \
  --export-task-identifier my-db-export \
  --source-arn arn:aws:rds:us-east-1:123456789:db:my-production-db \
  --s3-bucket-name my-backup-bucket-us-west-2 \
  --iam-role-arn arn:aws:iam::123456789:role/rds-export-role \
  --kms-key-id alias/rds-backup-key

# Hoặc bật automatic replication (AWS phiên bản mới hơn)
aws rds modify-db-instance \
  --db-instance-identifier my-production-db \
  --enable-automated-backups-replication \
  --backup-replication-region us-west-2
```

### Terraform Cross-Region Backup

```hcl
# Replication của automated backup sang region khác
resource "aws_db_instance_automated_backups_replication" "dr_backup" {
  source_db_instance_arn = aws_db_instance.production.arn
  retention_period       = 7
  
  # Backup này được tạo ở region khác
  provider = aws.us_west_2  # Destination region (Vùng Đích)
  
  kms_key_id = aws_kms_key.dr_region_key.arn
}
```

---

## 💰 Chi Phí Automated Backup

### Mô Hình Định Giá

```
Automated Backup Storage:
  - FREE (Miễn Phí): Đến kích thước bằng provisioned storage của DB
  - Ví dụ: DB có 100 GB storage → 100 GB backup storage miễn phí
  - Sau ngưỡng miễn phí: ~$0.095/GB/tháng (us-east-1)

Transaction Log Storage:
  - Được tính vào backup storage (không tách biệt)

Cross-Region Backup:
  - Thêm chi phí storage ở destination region
  - Thêm chi phí data transfer giữa regions (~$0.02/GB)

Công Thức Ước Tính:
  DB Size × (Retention Days / 30) × $0.095/GB/tháng
  
Ví Dụ:
  DB = 500 GB, Retention = 14 ngày
  Extra backup cost ≈ 500 × (14/30) × 0.095 ≈ $22/tháng
  (500 GB đầu miễn phí, chỉ tính phần storage dư ra)
```

### Tối Ưu Chi Phí Backup

```
1. Đặt retention period phù hợp với RPO — đừng giữ 35 ngày nếu RPO = 24h
2. Dùng Manual Snapshot cho milestones quan trọng thay vì tăng retention
3. Cross-region backup chỉ bật khi thực sự cần DR multi-region
4. Delete manual snapshots không cần thiết — automated backup tự dọn
```

---

## ⚠️ Những Điều Cần Lưu Ý

### Khi Xóa DB Instance

```
QUAN TRỌNG: Khi xóa DB instance:
  - Automated Backups: Bị xóa theo (theo mặc định)
  - Manual Snapshots: KHÔNG bị xóa — vẫn tồn tại

Để giữ lại automated backups khi xóa instance:
  AWS Console: Tick ✓ "Create final snapshot" + "Retain automated backups"
  
CLI:
  aws rds delete-db-instance \
    --db-instance-identifier my-production-db \
    --skip-final-snapshot false \          # Tạo final snapshot
    --final-db-snapshot-identifier my-final-snapshot
```

### Backup Window Conflict (Xung Đột Cửa Sổ)

```
TRÁNH: Đặt backup window trùng maintenance window

Backup Window:    02:00-03:00 UTC
Maintenance:      02:30-03:30 UTC  ← XẤU — Xung đột!

Nên làm:
Backup Window:    02:00-03:00 UTC
Maintenance:      03:30-04:30 UTC  ← TỐT — Không trùng
```

---

## 🎯 Câu Hỏi Phỏng Vấn Về Automated Backup

**Q: Automated Backup khác Manual Snapshot như thế nào?**
> Automated Backup được AWS tạo tự động hàng ngày trong backup window và bị xóa khi hết retention period hoặc khi xóa DB instance. Manual Snapshot do người dùng tạo thủ công, không bao giờ bị xóa tự động, và tồn tại vĩnh viễn đến khi bạn xóa. Cả hai đều cho phép restore, nhưng chỉ Automated Backup (kết hợp transaction logs) mới cho phép PITR (Point-in-Time Recovery — Khôi Phục Theo Thời Điểm).

**Q: Retention period tối đa của Automated Backup là bao nhiêu?**
> 35 ngày cho RDS. Nếu cần giữ lâu hơn, phải dùng Manual Snapshot (tồn tại vĩnh viễn) hoặc export sang S3.

**Q: Automated Backup ảnh hưởng đến performance của DB như thế nào?**
> Với Multi-AZ: Backup thực hiện từ standby replica, không ảnh hưởng primary. Với Single-AZ: Có thể tăng I/O latency nhẹ trong backup window. Với Aurora: Không ảnh hưởng gì vì backup là continuous (liên tục) và incremental vào S3.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
