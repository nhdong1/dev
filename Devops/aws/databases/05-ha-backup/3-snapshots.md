# Manual Snapshots — Ảnh Chụp Thủ Công & Cross-Region Copy

> Manual Snapshot (Ảnh Chụp Thủ Công) là bản sao đầy đủ của database storage volume tại một thời điểm cụ thể, được tạo bởi người dùng. Khác với Automated Backup, Manual Snapshot không bao giờ bị xóa tự động và tồn tại vĩnh viễn cho đến khi bạn chủ động xóa — kể cả sau khi xóa DB instance gốc.

---

## 🎯 Manual Snapshot Là Gì?

### Đặc Điểm Cốt Lõi

```
┌─────────────────────────────────────────────────────────────────┐
│                    Manual Snapshot                              │
│                                                                 │
│  ✅ Tạo bởi: Người dùng (thủ công hoặc tự động qua script)    │
│  ✅ Xóa bởi: Người dùng (không bao giờ tự động xóa)           │
│  ✅ Tồn tại: Vĩnh viễn sau khi xóa DB instance gốc            │
│  ✅ Chia sẻ: Có thể share với AWS account khác                 │
│  ✅ Copy:    Có thể copy sang region khác                       │
│  ✅ Restore: Tạo DB instance mới từ snapshot                   │
│                                                                 │
│  ❌ PITR:   Không hỗ trợ (chỉ restore về đúng thời điểm chụp) │
│  ❌ Auto-delete: Không có tính năng này                         │
└─────────────────────────────────────────────────────────────────┘
```

### So Sánh Automated Backup và Manual Snapshot

| Tiêu Chí | Automated Backup | Manual Snapshot |
|---------|-----------------|----------------|
| **Tạo bởi** | AWS tự động | Người dùng |
| **Thời điểm** | Daily trong backup window | Bất kỳ lúc nào |
| **PITR** | ✅ Hỗ trợ | ❌ Không |
| **Retention** | 1-35 ngày, tự động xóa | Vĩnh viễn, không tự xóa |
| **Sau khi xóa instance** | Bị xóa theo | Tồn tại |
| **Chi phí** | Miễn phí đến kích thước DB | $0.095/GB/tháng |
| **Chia sẻ** | Không thể | Có thể share |
| **Cross-region** | Cần enable riêng | Copy được dễ dàng |

---

## 🔧 Tạo Manual Snapshot

### Qua AWS Console

```
RDS Console → Databases → Chọn instance
→ Actions → Take snapshot
→ Snapshot name: "pre-migration-2026-05-15" (đặt tên có ý nghĩa)
→ Take snapshot
```

### Qua AWS CLI

```bash
# Tạo snapshot cho RDS instance
aws rds create-db-snapshot \
  --db-instance-identifier my-production-db \
  --db-snapshot-identifier "pre-migration-20260515-1430" \
  --tags Key=Purpose,Value=PreMigration \
         Key=Environment,Value=Production \
         Key=CreatedBy,Value=ops-team

# Kiểm tra trạng thái snapshot
aws rds describe-db-snapshots \
  --db-snapshot-identifier "pre-migration-20260515-1430" \
  --query 'DBSnapshots[0].{Status:Status, PercentProgress:PercentProgress}'

# Đợi snapshot hoàn thành (available)
aws rds wait db-snapshot-available \
  --db-snapshot-identifier "pre-migration-20260515-1430"
echo "Snapshot sẵn sàng!"
```

### Qua Terraform

```hcl
# Tạo snapshot thủ công cho instance hiện tại
resource "aws_db_snapshot" "pre_migration" {
  db_instance_identifier = aws_db_instance.production.identifier
  db_snapshot_identifier = "pre-migration-${formatdate("YYYYMMDD", timestamp())}"
  
  tags = {
    Purpose     = "PreMigration"
    Environment = "Production"
    AutoDelete  = "false"  # Nhắc nhở: cần xóa thủ công
  }
}
```

---

## 📋 Naming Convention (Quy Ước Đặt Tên) Tốt

### Template Đặt Tên

```
Format: {env}-{db-name}-{purpose}-{YYYYMMDD}-{HHMM}

Ví dụ:
  prod-userdb-pre-migration-20260515-1430
  prod-orderdb-pre-upgrade-20260520-0900
  staging-paydb-weekly-20260515-0200
  prod-catalogdb-incident-investigation-20260512-1645

Tại Sao Quan Trọng:
  - Dễ tìm kiếm khi cần restore khẩn cấp
  - Biết ngay mục đích snapshot
  - Tránh nhầm lẫn giữa các môi trường
```

---

## 🌐 Cross-Region Snapshot Copy (Sao Chép Snapshot Xuyên Vùng)

### Khi Nào Cần Cross-Region Copy?

```
Lý Do Cần Cross-Region:
  1. DR (Disaster Recovery): Nếu region chính sự cố → Dùng backup ở region khác
  2. Compliance: Một số quy định yêu cầu backup ở địa lý khác
  3. Migration: Chuẩn bị dữ liệu cho region mới
  4. Testing: Tạo môi trường test ở region khác với production data
```

### Copy Snapshot Sang Region Khác

```bash
# Copy snapshot từ us-east-1 sang us-west-2
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier \
    arn:aws:rds:us-east-1:123456789012:snapshot:prod-userdb-20260515 \
  --target-db-snapshot-identifier prod-userdb-20260515-dr-copy \
  --kms-key-id alias/rds-dr-key \    # Key ở destination region
  --copy-tags \                       # Copy tags từ source
  --region us-west-2                  # Destination region (Vùng Đích)

# Verify copy ở region đích
aws rds describe-db-snapshots \
  --region us-west-2 \
  --db-snapshot-identifier prod-userdb-20260515-dr-copy \
  --query 'DBSnapshots[0].{Status:Status, Engine:Engine, Size:AllocatedStorage}'
```

### Tự Động Hóa Cross-Region Copy Với Lambda

```python
import boto3
import os
from datetime import datetime

def lambda_handler(event, context):
    """
    Lambda function để tự động copy snapshot sang DR region
    Trigger: CloudWatch Events mỗi ngày lúc 04:00 AM
    """
    source_region = os.environ['SOURCE_REGION']    # us-east-1
    dr_region     = os.environ['DR_REGION']        # us-west-2
    db_instance   = os.environ['DB_INSTANCE_ID']   # my-production-db
    kms_key_id    = os.environ['DR_KMS_KEY_ID']    # KMS key ở DR region
    
    # Kết nối với source region
    source_rds = boto3.client('rds', region_name=source_region)
    
    # Tìm automated backup mới nhất
    response = source_rds.describe_db_instance_automated_backups(
        DBInstanceIdentifier=db_instance
    )
    
    if not response['DBInstanceAutomatedBackups']:
        print("Không tìm thấy automated backup")
        return
    
    latest_backup = response['DBInstanceAutomatedBackups'][0]
    source_arn = latest_backup['DBInstanceArn']
    
    # Tạo tên snapshot với timestamp (dấu thời gian)
    timestamp = datetime.now().strftime('%Y%m%d-%H%M')
    snapshot_id = f"{db_instance}-dr-copy-{timestamp}"
    
    # Copy snapshot sang DR region
    dr_rds = boto3.client('rds', region_name=dr_region)
    dr_rds.copy_db_snapshot(
        SourceDBSnapshotIdentifier=source_arn,
        TargetDBSnapshotIdentifier=snapshot_id,
        KmsKeyId=kms_key_id,
        CopyTags=True,
        Tags=[
            {'Key': 'Purpose', 'Value': 'DRBackup'},
            {'Key': 'SourceRegion', 'Value': source_region},
            {'Key': 'CopiedAt', 'Value': timestamp}
        ]
    )
    
    print(f"Đã bắt đầu copy snapshot {snapshot_id} sang {dr_region}")
    
    # Dọn dẹp snapshots cũ hơn 30 ngày ở DR region
    cleanup_old_dr_snapshots(dr_rds, db_instance, max_age_days=30)

def cleanup_old_dr_snapshots(rds_client, db_instance, max_age_days):
    """Xóa DR snapshots cũ hơn max_age_days ngày"""
    from datetime import timezone, timedelta
    
    cutoff = datetime.now(timezone.utc) - timedelta(days=max_age_days)
    
    snapshots = rds_client.describe_db_snapshots(
        DBInstanceIdentifier=db_instance,
        SnapshotType='manual',
        Filters=[{'Name': 'tag:Purpose', 'Values': ['DRBackup']}]
    )['DBSnapshots']
    
    for snap in snapshots:
        if snap['SnapshotCreateTime'] < cutoff:
            rds_client.delete_db_snapshot(
                DBSnapshotIdentifier=snap['DBSnapshotIdentifier']
            )
            print(f"Đã xóa snapshot cũ: {snap['DBSnapshotIdentifier']}")
```

---

## 🔄 Restore Từ Manual Snapshot

### Restore Tạo Instance Mới

```
QUAN TRỌNG: Restore KHÔNG ghi đè lên instance hiện tại.
Thay vào đó, nó tạo một DB instance mới với dữ liệu từ snapshot.

Quy Trình Restore:
  Snapshot → Restore → Instance mới (với tên khác)
                          ↓
                    Update connection string
                          ↓
                    Test & Verify
                          ↓
                    Promote (nếu muốn dùng làm production)
```

### Restore Qua CLI

```bash
# Restore từ snapshot tạo instance mới
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier my-db-restored-20260515 \
  --db-snapshot-identifier pre-migration-20260515-1430 \
  --db-instance-class db.r6g.large \
  --multi-az \
  --publicly-accessible false \
  --vpc-security-group-ids sg-xxxxxxxx \
  --db-subnet-group-name my-db-subnet-group

# Đợi instance available
aws rds wait db-instance-available \
  --db-instance-identifier my-db-restored-20260515

# Lấy endpoint mới
aws rds describe-db-instances \
  --db-instance-identifier my-db-restored-20260515 \
  --query 'DBInstances[0].Endpoint.{Host:Address,Port:Port}'
```

### Verify Sau Restore (Xác Minh Sau Khôi Phục)

```bash
# Connect và kiểm tra dữ liệu
mysql -h <new-endpoint> -u admin -p -e "
  SELECT 
    table_name,
    table_rows,
    ROUND(data_length/1024/1024, 2) AS 'Data Size (MB)'
  FROM information_schema.tables 
  WHERE table_schema = 'myapp'
  ORDER BY data_length DESC;
"

# So sánh row counts với production
mysql -h <production-endpoint> -u admin -p -e \
  "SELECT COUNT(*) FROM myapp.orders;" > prod_count.txt

mysql -h <new-endpoint> -u admin -p -e \
  "SELECT COUNT(*) FROM myapp.orders;" > restored_count.txt

diff prod_count.txt restored_count.txt
```

---

## 🔗 Chia Sẻ Snapshot (Snapshot Sharing)

### Share Với Account Khác

```bash
# Chia sẻ snapshot với AWS account khác (Cross-Account Sharing)
aws rds modify-db-snapshot-attribute \
  --db-snapshot-identifier prod-userdb-20260515 \
  --attribute-name restore \
  --values-to-add 987654321098  # Target account ID

# Chia sẻ với mọi người (public) — CỰC KỲ CẨN THẬN!
# Chỉ dùng cho demo data (dữ liệu demo), không bao giờ dùng cho production data
aws rds modify-db-snapshot-attribute \
  --db-snapshot-identifier demo-snapshot \
  --attribute-name restore \
  --values-to-add all

# Kiểm tra ai có quyền restore snapshot
aws rds describe-db-snapshot-attributes \
  --db-snapshot-identifier prod-userdb-20260515
```

### Encrypted Snapshot Sharing

```
Lưu Ý Quan Trọng Khi Share Encrypted Snapshots:
  1. Phải share KMS key với target account trước
  2. Target account cần có quyền sử dụng key để decrypt (giải mã)
  3. Không thể share KMS key mặc định của AWS (aws/rds) — phải dùng CMK
     (Customer Managed Key — Khóa Do Khách Hàng Quản Lý)

Quy Trình:
  Source Account              Target Account
       │                           │
       │ 1. Create CMK key         │
       │ 2. Create encrypted snap  │
       │ 3. Share key với target ──────►  Nhận quyền dùng key
       │ 4. Share snapshot ─────────────► Nhận snapshot
       │                           │ 5. Restore từ snapshot
```

---

## 🏗️ Snapshot Lifecycle Policy (Chính Sách Vòng Đời Snapshot)

### AWS Backup Service (Dịch Vụ Sao Lưu AWS)

```hcl
# Dùng AWS Backup để quản lý lifecycle snapshot tự động
resource "aws_backup_plan" "rds_snapshots" {
  name = "rds-snapshot-lifecycle-plan"

  rule {
    rule_name         = "weekly-snapshots"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 2 ? * SUN *)"  # Mỗi Chủ Nhật 2 AM

    lifecycle {
      cold_storage_after = 30   # Chuyển sang cold storage sau 30 ngày
      delete_after       = 365  # Xóa sau 1 năm
    }

    copy_action {
      destination_vault_arn = aws_backup_vault.dr_region.arn  # DR region

      lifecycle {
        delete_after = 365
      }
    }
  }

  rule {
    rule_name         = "monthly-long-term"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 2 1 * ? *)"  # Ngày 1 hàng tháng

    lifecycle {
      cold_storage_after = 90    # 90 ngày → cold storage
      delete_after       = 2555  # 7 năm (cho compliance)
    }
  }
}

resource "aws_backup_selection" "rds_databases" {
  name         = "rds-databases"
  iam_role_arn = aws_iam_role.backup_role.arn
  plan_id      = aws_backup_plan.rds_snapshots.id

  selection_tag {
    type  = "STRINGEQUALS"
    key   = "Backup"
    value = "true"
  }
}
```

---

## 📊 Snapshot Best Practices (Thực Hành Tốt Nhất)

### Khi Nào Nên Tạo Manual Snapshot

```
✅ Trước khi thực hiện:
  - Database migration (di chuyển database)
  - Schema change lớn (ALTER TABLE, thêm/xóa cột)
  - Major application deployment (triển khai ứng dụng lớn)
  - OS patching / DB engine upgrade

✅ Định kỳ:
  - Weekly snapshot (tuần) → Giữ 30 ngày
  - Monthly snapshot (tháng) → Giữ 1 năm
  - Yearly snapshot (năm) → Giữ 7 năm (compliance)

✅ Theo sự kiện:
  - Trước khi xóa DB instance (để không mất dữ liệu)
  - Sau khi load testing hoàn thành (capture clean state)
  - Sau khi data cleanup quan trọng
```

### Checklist Trước Khi Restore

```markdown
## Pre-Restore Checklist (Danh Sách Kiểm Tra Trước Khôi Phục)

### Xác Minh Snapshot
- [ ] Snapshot đúng thời điểm cần (kiểm tra SnapshotCreateTime)
- [ ] Snapshot ở trạng thái "available"
- [ ] Snapshot đúng DB engine và version
- [ ] Snapshot trong đúng region cần restore

### Chuẩn Bị Môi Trường
- [ ] Đã xác định DB instance class phù hợp
- [ ] Subnet group và security groups đã sẵn sàng
- [ ] KMS key available (nếu snapshot encrypted)
- [ ] IAM permissions đủ để restore

### Thông Báo
- [ ] Đã thông báo team về kế hoạch restore
- [ ] Đã đặt maintenance window nếu restore vào production
- [ ] Rollback plan đã chuẩn bị
```

---

## 🎯 Câu Hỏi Phỏng Vấn Về Manual Snapshots

**Q: Nếu tôi xóa một RDS instance, manual snapshot có bị xóa theo không?**
> Không. Manual Snapshot tồn tại độc lập với DB instance và không bị xóa tự động khi instance bị xóa. Đây là điểm khác biệt quan trọng so với Automated Backup (bị xóa khi instance bị xóa theo mặc định). Bạn phải chủ động xóa manual snapshot khi không cần nữa để tránh tốn chi phí storage.

**Q: Làm thế nào để copy snapshot sang region khác?**
> Dùng lệnh `aws rds copy-db-snapshot` với `--region` chỉ định destination region. Nếu snapshot được mã hóa, cần thêm `--kms-key-id` với KMS key ở destination region. Sau khi copy xong, snapshot mới sẽ xuất hiện ở destination region và có thể dùng để restore instance ở đó.

**Q: Bạn sẽ share snapshot với AWS account khác như thế nào để họ restore DB?**
> Có hai cách: (1) Với unencrypted snapshot: Dùng `modify-db-snapshot-attribute` thêm target account ID vào attribute `restore`. (2) Với encrypted snapshot: Phải share KMS CMK (Customer Managed Key) với target account trước, rồi mới share snapshot. Không thể dùng AWS managed key (aws/rds) cho cross-account sharing.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
