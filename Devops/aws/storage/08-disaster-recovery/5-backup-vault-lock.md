# Backup Vault Lock — WORM cho Backup và Compliance

> AWS Backup Vault Lock là tính năng áp dụng WORM — Write Once Read Many — Viết Một Lần Đọc Nhiều Lần cho recovery points trong backup vault. Khi bật, không ai — kể cả AWS root account và administrators — có thể xóa backup trước khi hết thời gian retention tối thiểu.

---

## 1. WORM là Gì và Tại Sao Cần

### WORM — Write Once Read Many

```
WORM Concept:
├── Dữ liệu có thể GHI MỘT LẦN
├── Sau đó chỉ có thể ĐỌC — không sửa, không xóa
└── Thậm chí người tạo ra cũng không xóa được (sau khi lock)

Ứng Dụng Trong DR:
├── Bảo vệ backup khỏi ransomware (không xóa được)
├── Bảo vệ khỏi insider threat (nhân viên nội bộ phá hoại)
├── Bảo vệ khỏi xóa nhầm (accident deletion)
└── Đáp ứng regulatory yêu cầu backup bất biến
```

### Regulatory Yêu Cầu WORM

```
Ngành Tài Chính:
├── SEC Rule 17a-4(f) — Yêu cầu financial records bất biến 3–6 năm
├── FINRA — Financial Industry Regulatory Authority
└── MiFID II — Markets in Financial Instruments Directive (EU)

Ngành Y Tế:
├── HIPAA — Health Insurance Portability and Accountability Act
│   → Backup y tế phải giữ 6 năm, không được xóa trước
└── FDA 21 CFR Part 11 — Clinical trial records

Ngành Chính Phủ:
├── FedRAMP — Federal Risk and Authorization Management Program
└── ITAR — International Traffic in Arms Regulations

Tiêu Chuẩn Chung:
├── ISO 27001 — Information Security Management
├── SOC 2 Type II — Service Organization Control
└── PCI-DSS — Payment Card Industry Data Security Standard
```

---

## 2. AWS Backup Vault Lock vs S3 Object Lock

### So Sánh Hai Cơ Chế WORM

| Tiêu Chí | Backup Vault Lock | S3 Object Lock |
|----------|------------------|----------------|
| Áp dụng cho | AWS Backup recovery points | S3 objects |
| Scope | Toàn bộ vault (tất cả recovery points) | Từng object hoặc bucket default |
| Chế độ | Governance hoặc Compliance | Governance hoặc Compliance |
| Min Retention | Bắt buộc khi bật | Tùy chọn |
| Max Retention | Tùy chỉnh | Tùy chỉnh |
| Ai có thể override Governance? | Root account + đặc quyền | S3:BypassGovernanceRetention permission |
| Compliance mode override | Không ai, kể cả AWS | Không ai, kể cả AWS |

### Khi Nào Dùng Gì

```
Dùng Backup Vault Lock khi:
├── Backup nhiều dịch vụ (EBS, RDS, EFS, DynamoDB...)
├── Cần bảo vệ toàn bộ backup vault
└── Muốn centralized WORM policy

Dùng S3 Object Lock khi:
├── Chỉ bảo vệ object trong S3 bucket
├── Cần WORM ở level object (từng file)
└── Dữ liệu application (log, report) lưu trực tiếp trên S3
```

---

## 3. Hai Chế Độ Vault Lock

### Governance Mode — Chế Độ Quản Trị

```
Governance Mode:
├── Ngăn chặn xóa recovery points trong retention period
├── NHƯNG: Admin với quyền đặc biệt (backup:DeleteBackupVault) CÓ THỂ:
│   ├── Xóa vault (khi vault rỗng hoặc với quyền đặc biệt)
│   ├── Thay đổi hoặc xóa lock settings
│   └── Xóa recovery points bằng quyền elevated privileges
│
└── Phù Hợp:
    ├── Bảo vệ khỏi lỗi vô tình (accidental deletion)
    ├── Môi trường cần flexibility (linh hoạt) thay đổi policy
    └── Test WORM trước khi chuyển Compliance mode

Cú Pháp:
"locked": true  (Governance)
```

### Compliance Mode — Chế Độ Tuân Thủ

```
Compliance Mode:
├── Ngăn chặn xóa recovery points trong retention period
├── KHÔNG AI có thể bypass — kể cả:
│   ├── AWS root account
│   ├── AWS support
│   └── Bản thân AWS (không thể xóa dù khách hàng yêu cầu)
│
├── Sau khi bật Compliance mode: KHÔNG THỂ TẮT hoặc thay đổi xuống
│   (chỉ có thể tăng min retention, không giảm được)
│
└── Phù Hợp:
    ├── SEC, FINRA, HIPAA compliance
    ├── Audit yêu cầu chứng minh backup bất biến
    └── Khi regulatory là yêu cầu bắt buộc

Cú Pháp:
"locked": true  (Compliance — không thể undo)
```

---

## 4. Cấu Hình Vault Lock

### Governance Mode

```bash
# Tạo vault trước
aws backup create-backup-vault \
    --backup-vault-name compliance-vault-governance \
    --encryption-key-arn arn:aws:kms:us-east-1:123456789012:key/backup-key

# Bật Vault Lock — Governance Mode
# min_retention_days: Tối thiểu phải giữ backup bao nhiêu ngày
# max_retention_days: Tối đa có thể giữ backup
# changeable_for_days: Cho phép thay đổi/tắt lock trong N ngày đầu
aws backup put-backup-vault-lock-configuration \
    --backup-vault-name compliance-vault-governance \
    --min-retention-days 30 \
    --max-retention-days 365 \
    --changeable-for-days 3
    # changeable-for-days: Trong 3 ngày có thể thay đổi/hủy lock
    # Sau 3 ngày → lock được "sealed" (đóng dấu)
    # Governance mode vẫn có thể tắt với quyền đặc biệt
```

### Compliance Mode

```bash
# Tạo vault mới (quan trọng: KHÔNG dùng vault đang có data quan trọng để test)
aws backup create-backup-vault \
    --backup-vault-name compliance-vault-locked \
    --encryption-key-arn arn:aws:kms:us-east-1:123456789012:key/compliance-key

# Bật Vault Lock — Compliance Mode
# CẢNH BÁO: Đây là hành động KHÔNG THỂ HOÀN TÁC
aws backup put-backup-vault-lock-configuration \
    --backup-vault-name compliance-vault-locked \
    --min-retention-days 90 \
    --max-retention-days 2555 \
    --changeable-for-days 3
    # Sau 3 ngày → Compliance mode được kích hoạt hoàn toàn
    # Từ đó: KHÔNG AI có thể xóa recovery points trước 90 ngày
    # KHÔNG AI có thể tắt lock
    # KHÔNG AI có thể giảm min_retention_days
```

### Xác Nhận Lock Configuration

```bash
# Kiểm tra cấu hình lock
aws backup describe-backup-vault \
    --backup-vault-name compliance-vault-locked

# Output quan trọng cần kiểm tra:
# {
#   "BackupVaultName": "compliance-vault-locked",
#   "Locked": true,
#   "MinRetentionDays": 90,
#   "MaxRetentionDays": 2555,
#   "LockDate": "2026-05-16T10:00:00Z"  ← lock đã active
# }
```

---

## 5. Changeable-for-Days — Thời Gian Thay Đổi

### Hiểu Tham Số changeable-for-days

```
changeable-for-days = "Cool-down period" (giai đoạn làm quen)

Ví Dụ với changeable-for-days = 3:

Ngày 0: Bật Vault Lock
   │
   ├── Ngày 1: Còn có thể thay đổi min/max retention
   ├── Ngày 2: Còn có thể tắt lock nếu phát hiện cấu hình sai
   ├── Ngày 3: Còn có thể điều chỉnh
   │
   └── Ngày 4: Lock SEALED (đóng hoàn toàn)
               → Compliance: KHÔNG AI có thể thay đổi
               → Governance: Chỉ admin đặc biệt mới bypass

Khuyến Nghị:
├── changeable-for-days = 3 (tối thiểu theo AWS best practices)
├── Dùng giai đoạn này để verify cấu hình đúng
└── Test restore trong giai đoạn này trước khi seal
```

---

## 6. Kết Hợp Vault Lock với Backup Plan

### Đảm Bảo Recovery Points Luôn Trong Vault Lock

```bash
# Backup Plan phải dùng vault đã được lock
aws backup create-backup-plan \
    --backup-plan '{
        "BackupPlanName": "Compliance-Backup-Plan",
        "Rules": [
            {
                "RuleName": "Daily-Compliance-Backup",
                "TargetBackupVaultName": "compliance-vault-locked",
                "ScheduleExpression": "cron(0 1 ? * * *)",
                "Lifecycle": {
                    "DeleteAfterDays": 90
                }
            }
        ]
    }'
```

### Lưu Ý: Retention phải trong range Min-Max của Vault Lock

```
Vault Lock cấu hình:
├── min_retention_days = 90
└── max_retention_days = 2555

Backup Plan lifecycle:
├── DeleteAfterDays = 30 → LỖI (dưới min 90 ngày)
├── DeleteAfterDays = 90 → OK (bằng min)
├── DeleteAfterDays = 365 → OK (trong range)
└── DeleteAfterDays = 3000 → LỖI (vượt max 2555 ngày)

Nếu backup plan tạo recovery point với retention ngoài range:
→ Recovery point vẫn được tạo nhưng không bị xóa trước min_retention
→ AWS sẽ không xóa recovery point cho đến khi đạt min_retention
```

---

## 7. Monitoring Vault Lock

```bash
# CloudWatch Metric cho locked vault
# Theo dõi số recovery points bị cố gắng xóa (sẽ bị từ chối)
aws cloudwatch put-metric-alarm \
    --alarm-name "Backup-Vault-Unauthorized-Delete-Attempt" \
    --metric-name "NumberOfBackupJobsFailed" \
    --namespace "AWS/Backup" \
    --statistic Sum \
    --period 300 \
    --threshold 1 \
    --comparison-operator GreaterThanOrEqualToThreshold \
    --alarm-actions arn:aws:sns:us-east-1:123456789:Security-Alert

# CloudTrail Event để audit ai cố gắng xóa
# Event: backup:DeleteRecoveryPoint → sẽ bị log + từ chối
aws cloudtrail create-trail \
    --name backup-audit-trail \
    --s3-bucket-name cloudtrail-backup-audit-logs \
    --include-global-service-events \
    --is-multi-region-trail \
    --enable-log-file-validation
```

### Lambda Kiểm Tra Vault Lock Status

```python
import boto3
import json

def check_vault_lock_compliance(event, context):
    """
    Kiểm tra hàng ngày tất cả backup vault có được lock không.
    Gửi cảnh báo nếu vault chưa lock hoặc lock hết hạn.
    """
    backup = boto3.client('backup')
    sns = boto3.client('sns')
    
    SNS_ARN = "arn:aws:sns:us-east-1:123456789:compliance-alerts"
    REQUIRED_VAULTS = ["compliance-vault-locked", "dr-vault-locked"]
    
    unlocked_vaults = []
    
    for vault_name in REQUIRED_VAULTS:
        try:
            vault = backup.describe_backup_vault(BackupVaultName=vault_name)
            
            if not vault.get('Locked', False):
                unlocked_vaults.append({
                    'vault': vault_name,
                    'issue': 'NOT LOCKED — compliance violation!'
                })
            else:
                min_days = vault.get('MinRetentionDays', 0)
                if min_days < 90:
                    unlocked_vaults.append({
                        'vault': vault_name,
                        'issue': f'Min retention {min_days} days < 90 days required'
                    })
        except Exception as e:
            unlocked_vaults.append({
                'vault': vault_name,
                'issue': f'Cannot access vault: {str(e)}'
            })
    
    if unlocked_vaults:
        sns.publish(
            TopicArn=SNS_ARN,
            Subject="COMPLIANCE ALERT: Backup Vault Lock Issue",
            Message=json.dumps(unlocked_vaults, indent=2)
        )
    
    return {
        'compliant_vaults': len(REQUIRED_VAULTS) - len(unlocked_vaults),
        'non_compliant': unlocked_vaults
    }
```

---

## 8. So Sánh Vault Lock và S3 Object Lock

### S3 Object Lock cho Backup Data

```bash
# Khi backup dữ liệu trực tiếp vào S3 (không qua AWS Backup service)
# Ví dụ: Database export, log archive

# Tạo bucket với Object Lock bật sẵn
aws s3api create-bucket \
    --bucket compliance-backup-bucket \
    --region us-east-1 \
    --object-lock-enabled-for-bucket

# Đặt default retention cho bucket
aws s3api put-object-lock-configuration \
    --bucket compliance-backup-bucket \
    --object-lock-configuration '{
        "ObjectLockEnabled": "Enabled",
        "Rule": {
            "DefaultRetention": {
                "Mode": "COMPLIANCE",
                "Days": 90
            }
        }
    }'
```

### Governance vs Compliance — Quyết Định Nhanh

```
Câu Hỏi Quyết Định:

"Regulatory có yêu cầu không ai có thể xóa backup không?"
├── Có → Compliance Mode
└── Không → Governance Mode

"Team có cần khả năng sửa/xóa backup trong trường hợp khẩn không?"
├── Có → Governance Mode
└── Không → Compliance Mode

"Đây là môi trường test hay production?"
├── Test → Governance Mode (linh hoạt)
└── Production với compliance → Compliance Mode

Lưu Ý Quan Trọng:
Không thể downgrade từ Compliance → Governance
Không thể tắt Compliance mode
Chỉ có thể TĂNG min_retention trong Compliance mode
```

---

## 9. Kết Hợp Vault Lock với Toàn Bộ DR Strategy

### Kiến Trúc DR Hoàn Chỉnh với WORM

```
Production Account (us-east-1)
│
├── Production Resources (EBS, RDS, EFS, DynamoDB)
│       │
│       │ backup every hour/day
│       ▼
├── Production Vault (Governance Lock)
│   ├── Min retention: 30 ngày
│   ├── Max retention: 365 ngày
│   └── Backup Plan: hourly/daily/monthly
│           │
│           │ copy to DR
│           ▼
└── DR Vault in eu-west-1 (Compliance Lock)
    ├── Min retention: 90 ngày ← không ai xóa được
    ├── Max retention: 2555 ngày (7 năm)
    └── Phục vụ: Regulatory compliance + ransomware protection

Backup Account (987654321098) — cross-account
        ▲
        │ cross-account copy
        │
Central Compliance Vault (Compliance Lock)
├── Min retention: 365 ngày
├── Max retention: 2555 ngày
└── Chỉ backup team trong account này được truy cập
```

---

## 10. Checklist Vault Lock Compliance

```
Chuẩn Bị:
□ Đã xác định regulatory requirements cụ thể (HIPAA/SEC/PCI-DSS)
□ Đã xác định min/max retention phù hợp với yêu cầu
□ Đã test cấu hình Governance mode trước khi dùng Compliance mode
□ Đã inform legal/compliance team về hành động không thể hoàn tác

Cấu Hình:
□ Vault được tạo với KMS key riêng (Customer Managed Key)
□ changeable-for-days đủ lớn để verify (tối thiểu 3 ngày)
□ Backup plan dùng đúng vault đã lock
□ Retention trong backup plan nằm trong [min, max] của vault lock

Kiểm Tra Sau Khi Bật:
□ Đã verify Locked=true trong describe-backup-vault
□ Đã thử tạo backup và kiểm tra recovery point
□ Đã thử restore từ locked vault (phải thành công)
□ Đã thử xóa recovery point (phải bị từ chối với Compliance mode)

Vận Hành:
□ CloudTrail bật để audit mọi truy cập vào vault
□ Lambda hoặc Config Rule kiểm tra vault lock status hàng ngày
□ SNS alert khi có attempt xóa bất hợp lệ
□ Quarterly audit report cho compliance team
□ Annual DR test bao gồm restore từ locked vault
```

---

## 11. Câu Hỏi Phỏng Vấn Về Vault Lock

**Q: "Sự khác biệt giữa Governance và Compliance mode?"**
> Governance mode cho phép admin có quyền đặc biệt xóa hoặc thay đổi lock. Compliance mode không cho phép ai xóa recovery points trước min retention — kể cả root account và AWS. Compliance mode không thể bị tắt sau khi đã seal.

**Q: "Tại sao cần Vault Lock khi đã có IAM policies?"**
> IAM policies có thể bị thay đổi bởi admin. Nếu admin bị compromise (bị xâm phạm) hoặc tự ý thay đổi policy, backup vẫn có thể bị xóa. Vault Lock là safeguard (bảo vệ) cấp độ service, hoạt động độc lập với IAM — không có IAM permission nào bypass được Compliance mode.

**Q: "Làm thế nào migrate sang Vault Lock khi đang có backup cũ?"**
> Tạo vault mới với lock cấu hình, sau đó copy recovery points từ vault cũ sang vault mới. Recovery points trong vault mới sẽ được bảo vệ bởi lock. Vault cũ vẫn giữ recovery points cũ (không lock) cho đến khi hết retention.

**Q: "Vault Lock có bảo vệ khỏi ransomware không?"**
> Có. Ngay cả khi ransomware xâm nhập vào AWS account và có quyền admin, với Compliance mode, nó không thể xóa recovery points trước thời hạn. Điều này giả định attacker chưa compromise tất cả mọi thứ trong 90 ngày (thời gian min retention).

---

**Kết Thúc Topic:** [README.md](README.md) — Quay lại tổng quan Disaster Recovery
**Topic Tiếp Theo:** [09-storage-gateway/](../09-storage-gateway/) — Storage Gateway & Hybrid Cloud
