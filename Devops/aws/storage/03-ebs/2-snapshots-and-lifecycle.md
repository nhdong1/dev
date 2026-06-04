# EBS Snapshots & DLM — Ảnh Chụp Nhanh & Quản Lý Vòng Đời

> EBS Snapshot (Ảnh Chụp Nhanh) là bản sao lưu của EBS volume được lưu trên S3, bảo vệ dữ liệu khỏi mất mát và cho phép khôi phục linh hoạt. DLM (Data Lifecycle Manager — Trình Quản Lý Vòng Đời Dữ Liệu) tự động hóa quá trình tạo, lưu giữ và xóa snapshot theo chính sách định sẵn.

---

## 🧠 Snapshot Hoạt Động Như Thế Nào?

### Cơ Chế Incremental (Tăng Dần)

```
Snapshot 1 (Full — Đầy Đủ):
  Thời gian: T=0
  Dữ liệu: Block A, B, C, D, E (100GB)
  Lưu lên S3: 100GB
  
Snapshot 2 (Incremental — Tăng Dần):
  Thời gian: T=1 ngày
  Thay đổi: Block C' (sửa), Block F (thêm mới)
  Lưu lên S3: chỉ 2 block thay đổi (~5GB)
  
Snapshot 3:
  Thời gian: T=2 ngày
  Thay đổi: Block A' (sửa)
  Lưu lên S3: chỉ 1 block thay đổi (~2GB)

Tổng dung lượng S3 sau 3 snapshots: 107GB (không phải 300GB)
```

### Tính Độc Lập Của Snapshot

Mặc dù snapshot là incremental khi **tạo**, mỗi snapshot khi **xem** đều là đầy đủ và độc lập:

```
Muốn restore Snapshot 2?
→ AWS tự động tổng hợp: S1(A,B,D,E) + S2(C',F) = volume hoàn chỉnh
→ Không cần giữ S1 để xóa S2 — mỗi snapshot tự quản lý dữ liệu của mình
→ Có thể xóa bất kỳ snapshot nào mà không ảnh hưởng các snapshot khác
```

---

## 📸 Tạo Snapshot

### Từ AWS Console / CLI

```bash
# Tạo snapshot thủ công
aws ec2 create-snapshot \
  --volume-id vol-1234567890abcdef0 \
  --description "Database backup trước khi deploy v2.0" \
  --tag-specifications 'ResourceType=snapshot,Tags=[
    {Key=Name,Value=db-snapshot-20260515},
    {Key=Environment,Value=production},
    {Key=Application,Value=payment-service}
  ]'

# Kết quả trả về SnapshotId
# "SnapshotId": "snap-0123456789abcdef0"
# "State": "pending" → "completed"
```

### Trạng Thái Snapshot

```
pending    → Đang tạo (volume vẫn dùng được trong lúc này)
completed  → Hoàn thành, sẵn sàng restore
error      → Lỗi khi tạo
recovering → Đang phục hồi từ lỗi
```

### Best Practice: Flush Trước Khi Snapshot

```bash
# Đối với database (Linux):
# 1. Flush và lock table
mysql -e "FLUSH TABLES WITH READ LOCK;"

# 2. Sync filesystem để đảm bảo dữ liệu ghi hết vào disk
sync

# 3. Tạo snapshot
aws ec2 create-snapshot --volume-id vol-xxx ...

# 4. Unlock database
mysql -e "UNLOCK TABLES;"
```

> **Lưu ý:** EBS snapshot là **crash-consistent** (nhất quán khi crash) — an toàn để khôi phục, nhưng **application-consistent** (nhất quán ứng dụng) cần flush thủ công hoặc dùng AWS VSS (Volume Shadow Copy Service — Dịch Vụ Sao Chép Bóng).

---

## 🔄 Restore Từ Snapshot

### Tạo Volume Mới Từ Snapshot

```bash
# Restore trong cùng AZ (Availability Zone — Vùng Khả Dụng)
aws ec2 create-volume \
  --snapshot-id snap-0123456789abcdef0 \
  --availability-zone us-east-1a \
  --volume-type gp3 \
  --iops 3000

# Restore sang AZ khác (di chuyển dữ liệu)
aws ec2 create-volume \
  --snapshot-id snap-0123456789abcdef0 \
  --availability-zone us-east-1b \  # AZ khác
  --volume-type gp3

# Copy snapshot sang Region khác trước
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-0123456789abcdef0 \
  --destination-region us-west-2 \
  --description "DR copy to us-west-2"
```

### Fast Snapshot Restore (FSR) — Khôi Phục Nhanh

Khi volume mới được tạo từ snapshot, dữ liệu được **lazy-load** (tải lười biếng) từ S3:
- Volume sẵn sàng ngay lập tức
- Nhưng lần đọc đầu tiên cho mỗi block sẽ chậm hơn (I/O latency cao hơn)

FSR (Fast Snapshot Restore — Khôi Phục Nhanh Từ Snapshot) giải quyết vấn đề này:

```bash
# Bật FSR cho snapshot (tính phí theo giờ)
aws ec2 enable-fast-snapshot-restores \
  --availability-zones us-east-1a us-east-1b \
  --source-snapshot-ids snap-0123456789abcdef0

# Giá FSR: $0.75 per snapshot per AZ per hour
# Khi nào cần: DR với RTO (Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục) yêu cầu volume ready ngay
```

---

## 📋 DLM — Data Lifecycle Manager — Trình Quản Lý Vòng Đời Dữ Liệu

DLM là dịch vụ tự động hóa việc tạo, lưu giữ và xóa EBS snapshots theo policy (chính sách).

### Tại Sao Cần DLM?

```
Vấn đề không có DLM:
  - Phải nhớ tạo snapshot thủ công
  - Snapshot cũ tích lũy → tốn chi phí S3
  - Không nhất quán giữa các môi trường
  - Không đáp ứng compliance (tuân thủ) tự động

Với DLM:
  - Tự động tạo theo lịch (cron-like schedule)
  - Tự động xóa snapshot hết hạn
  - Hỗ trợ retention (lưu giữ) linh hoạt
  - Tag-based targeting — nhắm đến volume theo tag
  - Cross-region copy tự động
  - Cross-account sharing tự động
```

### Cấu Trúc DLM Policy

```
DLM Lifecycle Policy
├── Target Resources  — Volume nào cần quản lý (dựa trên tags)
├── Schedule          — Khi nào tạo snapshot (giờ, ngày, tuần)
├── Retention         — Giữ bao nhiêu snapshot (số lượng hoặc ngày)
├── Copy Actions      — Copy sang region khác không?
├── Cross-Account     — Chia sẻ sang account khác không?
└── Fast Restore      — Bật FSR tự động không?
```

### Tạo DLM Policy Cơ Bản

```bash
# Tạo IAM role cho DLM
aws iam create-role \
  --role-name AWSDataLifecycleManagerDefaultRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "dlm.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Policy: backup hàng ngày, giữ 30 ngày, cho volume có tag Backup=daily
aws dlm create-lifecycle-policy \
  --description "Daily backup - retain 30 days" \
  --state ENABLED \
  --execution-role-arn arn:aws:iam::123456789012:role/AWSDataLifecycleManagerDefaultRole \
  --policy-details '{
    "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
    "ResourceTypes": ["VOLUME"],
    "TargetTags": [{"Key": "Backup", "Value": "daily"}],
    "Schedules": [{
      "Name": "Daily backup",
      "CreateRule": {
        "Interval": 24,
        "IntervalUnit": "HOURS",
        "Times": ["03:00"]
      },
      "RetainRule": {
        "Count": 30
      },
      "TagsToAdd": [
        {"Key": "SnapshotType", "Value": "automated"},
        {"Key": "ManagedBy", "Value": "DLM"}
      ],
      "CopyTags": true
    }]
  }'
```

### DLM Policy Nâng Cao: Cross-Region DR

```json
{
  "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
  "ResourceTypes": ["VOLUME"],
  "TargetTags": [{"Key": "Environment", "Value": "production"}],
  "Schedules": [
    {
      "Name": "Hourly snapshot",
      "CreateRule": {
        "Interval": 1,
        "IntervalUnit": "HOURS"
      },
      "RetainRule": {"Count": 24},
      "Name": "Keep 24 hourlies"
    },
    {
      "Name": "Daily snapshot with cross-region copy",
      "CreateRule": {
        "CronExpression": "cron(0 2 * * ? *)"
      },
      "RetainRule": {"Count": 30},
      "CrossRegionCopyRules": [
        {
          "TargetRegion": "us-west-2",
          "Encrypted": true,
          "CmkArn": "arn:aws:kms:us-west-2:123456789012:key/xxx",
          "RetainRule": {"Interval": 7, "IntervalUnit": "DAYS"}
        }
      ]
    }
  ]
}
```

---

## 🏷️ Snapshot Tagging Strategy (Chiến Lược Gán Nhãn)

Tagging là nền tảng để quản lý snapshot hiệu quả:

```bash
# Tag chuẩn cho snapshot production
aws ec2 create-tags \
  --resources snap-0123456789abcdef0 \
  --tags \
    Key=Name,Value="prod-db-daily-20260515" \
    Key=Environment,Value=production \
    Key=Application,Value=payment-service \
    Key=VolumeId,Value=vol-1234567890abcdef0 \
    Key=RetentionDays,Value=30 \
    Key=CreatedBy,Value=DLM-policy-prod

# Tìm snapshot theo tag
aws ec2 describe-snapshots \
  --filters \
    Name=tag:Environment,Values=production \
    Name=tag:Application,Values=payment-service \
  --query "Snapshots[*].[SnapshotId,StartTime,Description]" \
  --output table
```

---

## 🔐 Snapshot Encryption (Mã Hóa Snapshot)

### Quy Tắc Mã Hóa

```
Snapshot từ volume KHÔNG mã hóa:
  → Snapshot mặc định KHÔNG mã hóa
  → Có thể copy snapshot và bật mã hóa khi copy

Snapshot từ volume MÃ HÓA:
  → Snapshot LUÔN mã hóa (không thể tắt)
  → Dùng cùng KMS key với volume gốc

Volume tạo từ snapshot mã hóa:
  → Volume tạo ra LUÔN mã hóa
```

### Encrypt Snapshot Khi Copy

```bash
# Copy snapshot và bật mã hóa
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-unencrypted-xxx \
  --destination-region us-east-1 \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/your-key-id \
  --description "Encrypted copy of unencrypted snapshot"
```

### Bật Mã Hóa Mặc Định Cho Toàn Account

```bash
# Bật default encryption cho toàn account trong region
aws ec2 enable-ebs-encryption-by-default

# Tất cả snapshot và volume mới sẽ tự động mã hóa
# Dùng AWS managed key hoặc custom CMK
```

---

## 💰 Quản Lý Chi Phí Snapshot

### Pricing (Giá)

```
EBS Snapshot: $0.05/GB-month (cho dữ liệu thực sự lưu trên S3)
  → Incremental: chỉ tính phần thay đổi, không phải toàn bộ volume
  → Ví dụ: 1TB volume, 10GB thay đổi mỗi ngày
    - Snapshot 1: $0.05 × 1.000 = $50 (full)
    - Snapshot 2: $0.05 × 10 = $0.50 (incremental)
    - Snapshot 30: $0.50/ngày = ~$15/tháng incremental

EBS Snapshot Archive: $0.0125/GB-month (rẻ hơn 75%)
  → Dành cho snapshot cũ, truy cập hiếm (Glacier-like)
  → Restore mất 24-72 giờ
```

### Tiết Kiệm Với Snapshot Archive

```bash
# Chuyển snapshot sang Archive tier (lưu trữ lạnh)
aws ec2 modify-snapshot-tier \
  --snapshot-id snap-0123456789abcdef0 \
  --storage-tier archive

# Kiểm tra tier
aws ec2 describe-snapshot-tier-status \
  --snapshot-ids snap-0123456789abcdef0

# Restore từ Archive (mất 24-72 giờ)
aws ec2 restore-snapshot-tier \
  --snapshot-id snap-0123456789abcdef0 \
  --temporary-restore-days 7  # Giữ ở Standard tier trong 7 ngày
```

---

## 📊 AMI — Amazon Machine Image — Ảnh Máy Ảo Amazon

AMI (Amazon Machine Image — Ảnh Máy Ảo Amazon) là tập hợp snapshot + metadata để tạo EC2 instance. AMI được xây dựng trên EBS snapshot.

```
AMI = EBS Snapshot(s) + Launch Configuration
    = OS + Application + Config

Luồng tạo AMI:
EC2 Instance → Create Image → AMI → Launch new EC2

Ứng dụng:
  - Golden AMI pattern: tạo AMI chuẩn hóa với OS, patches, agent
  - Auto Scaling: dùng AMI để launch instance mới
  - DR: copy AMI sang region khác để restore nhanh
```

```bash
# Tạo AMI từ EC2 instance
aws ec2 create-image \
  --instance-id i-1234567890abcdef0 \
  --name "payment-service-v2.0-$(date +%Y%m%d)" \
  --description "Golden AMI for payment service" \
  --no-reboot  # Cẩn thận: có thể không nhất quán nếu có write đang chạy
```

---

## 🎯 Chiến Lược Backup Thực Tế

### Retention Policy Theo Môi Trường

```
Production Database:
  - Hourly snapshots: giữ 24 snapshots (24 giờ)
  - Daily snapshots:  giữ 30 snapshots (30 ngày)
  - Weekly snapshots: giữ 12 snapshots (3 tháng)
  - Monthly snapshots: giữ 12 snapshots (1 năm)
  - Annual snapshots:  giữ 7 snapshots (7 năm — compliance)

Development:
  - Daily snapshots: giữ 7 snapshots

Test/Staging:
  - Weekly snapshots: giữ 4 snapshots
```

### 3-2-1 Backup Rule (Quy Tắc 3-2-1)

```
3 bản sao dữ liệu:
  - Bản 1: EBS volume gốc (production)
  - Bản 2: EBS Snapshot trong cùng region
  - Bản 3: EBS Snapshot copy sang region khác

2 loại media/location khác nhau:
  - EBS Volume (block storage)
  - S3 (object storage — nơi snapshot được lưu)

1 bản offsite (ngoài site):
  - Snapshot copy sang region/account khác
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Snapshot incremental hoạt động như thế nào?**
> Snapshot đầu tiên copy toàn bộ dữ liệu. Các snapshot sau chỉ lưu các block có thay đổi kể từ snapshot trước. Tuy nhiên, mỗi snapshot vẫn là một điểm khôi phục đầy đủ độc lập — AWS tự tổng hợp khi restore. Điều này tiết kiệm chi phí S3 đáng kể.

**Q: Nếu xóa snapshot S2 (giữa S1 và S3), điều gì xảy ra?**
> S3 vẫn hoạt động bình thường. Khi S2 bị xóa, AWS tự động di chuyển bất kỳ block nào S3 đang tham chiếu từ S2 vào S3. Không mất dữ liệu, nhưng S3 có thể tăng dung lượng.

**Q: DLM khác với AWS Backup như thế nào?**
> DLM chuyên cho EBS snapshot, đơn giản hơn và rẻ hơn. AWS Backup là dịch vụ backup tập trung đa dịch vụ (EBS, RDS, DynamoDB, EFS, FSx) với một policy thống nhất, hỗ trợ Vault Lock (WORM), phù hợp compliance phức tạp.

**Q: Fast Snapshot Restore là gì và khi nào dùng?**
> Thường thì khi tạo volume từ snapshot, dữ liệu được lazy-load từ S3, gây I/O latency cao cho lần đọc đầu tiên. FSR pre-warms dữ liệu để volume có hiệu suất đầy đủ ngay lập tức. Nên dùng khi có yêu cầu RTO thấp trong DR scenario.

---

## 🔗 Điều Hướng

- **Trước:** [1-volume-types.md](./1-volume-types.md) — Volume Types
- **Tiếp theo:** [3-ebs-multi-attach.md](./3-ebs-multi-attach.md) — Multi-Attach
- **Liên quan:** [../08-disaster-recovery/3-ebs-snapshot-strategy.md](../08-disaster-recovery/3-ebs-snapshot-strategy.md) — DR Strategy

---

**Cập Nhật Lần Cuối:** 2026-05-15
