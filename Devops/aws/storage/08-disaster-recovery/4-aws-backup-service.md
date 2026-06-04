# AWS Backup — Dịch Vụ Backup Tập Trung

> AWS Backup là dịch vụ managed (được quản lý hoàn toàn) cho phép tập trung hóa và tự động hóa backup cho nhiều dịch vụ AWS từ một nơi duy nhất — thay thế việc quản lý snapshot/backup rải rác ở từng dịch vụ.

---

## 1. Tại Sao Cần AWS Backup

### Vấn Đề Khi Backup Phân Tán

```
Trước AWS Backup:
├── S3 → Lifecycle policies + versioning (cấu hình riêng)
├── EBS → DLM policies (cấu hình riêng)
├── RDS → Automated Backups + Manual Snapshots (cấu hình riêng)
├── DynamoDB → On-demand backups (cấu hình riêng)
├── EFS → AWS Backup (bắt buộc, không có native backup)
└── FSx → Backup natively (cấu hình riêng)

Hậu Quả:
├── Không có view tập trung "hôm nay đã backup đủ chưa?"
├── Mỗi dịch vụ có retention policy khác nhau → dễ sót
├── Compliance audit (kiểm toán tuân thủ) rất khó
├── Cross-account backup không có chuẩn hóa
└── Không enforce minimum backup frequency (tần suất tối thiểu)
```

### Sau AWS Backup

```
Một Policy → Quản Lý Tất Cả:
├── Backup Plan (kế hoạch backup) áp dụng qua nhiều dịch vụ
├── Single dashboard (bảng điều khiển duy nhất) cho toàn tổ chức
├── Cross-account và cross-region backup tự động
├── Audit trail đầy đủ qua AWS Backup Audit Manager
└── Vault Lock cho WORM (Write Once Read Many) compliance
```

---

## 2. Kiến Trúc AWS Backup

### Các Thành Phần Chính

```
AWS Backup Architecture:

Backup Plan (Kế Hoạch Backup)
├── Backup Rule 1: Daily — 30 ngày retention
├── Backup Rule 2: Weekly — 90 ngày retention
└── Backup Rule 3: Monthly — 1 năm retention
        │
        │ áp dụng cho
        ▼
Resource Assignment (Phân Công Tài Nguyên)
├── By Tag: Backup=daily → tất cả tài nguyên có tag này
├── By ARN: arn:aws:rds:... → specific database
└── By Resource Type: Tất cả EFS trong account
        │
        │ tạo ra
        ▼
Recovery Points (Điểm Khôi Phục)
├── Stored trong Backup Vault
├── Immutable nếu Vault Lock bật (bất biến)
└── Có thể copy sang vault khác (account hoặc region)
        │
        │ lưu vào
        ▼
Backup Vault (Kho Backup)
├── Logical container (container logic) cho recovery points
├── Mã hóa với KMS key riêng
└── Access policy kiểm soát ai có thể xóa
```

### Dịch Vụ Được Hỗ Trợ

| Dịch Vụ | Loại Backup | Ghi Chú |
|---------|------------|---------|
| Amazon EBS | Snapshots | Full + incremental |
| Amazon RDS | Automated backups | Mọi engine |
| Amazon Aurora | Cluster snapshots | Backup đặc biệt hiệu quả |
| Amazon DynamoDB | On-demand backup | Point-in-time restore |
| Amazon EFS | Backup to vault | Không có native backup |
| Amazon FSx | Backup | Windows, Lustre, ONTAP, OpenZFS |
| Amazon S3 | Backup | Cần bật versioning trước |
| AWS Storage Gateway | Backup | Volume gateway |
| Amazon DocumentDB | Cluster snapshots | MongoDB-compatible |
| Amazon Neptune | Cluster snapshots | Graph database |
| VMware Cloud on AWS | VM snapshots | Hybrid workloads |

---

## 3. Tạo Backup Plan — Hướng Dẫn Từng Bước

### Tạo qua AWS CLI

```bash
# Bước 1: Tạo Backup Plan với nhiều quy tắc
aws backup create-backup-plan \
    --backup-plan '{
        "BackupPlanName": "Production-Comprehensive-Backup",
        "Rules": [
            {
                "RuleName": "Daily-Backups",
                "TargetBackupVaultName": "production-vault",
                "ScheduleExpression": "cron(0 2 ? * * *)",
                "StartWindowMinutes": 60,
                "CompletionWindowMinutes": 180,
                "Lifecycle": {
                    "MoveToColdStorageAfterDays": 30,
                    "DeleteAfterDays": 90
                },
                "CopyActions": [
                    {
                        "DestinationBackupVaultArn": "arn:aws:backup:eu-west-1:123456789012:backup-vault:dr-vault",
                        "Lifecycle": {
                            "DeleteAfterDays": 90
                        }
                    }
                ],
                "EnableContinuousBackup": false
            },
            {
                "RuleName": "Weekly-Backups",
                "TargetBackupVaultName": "production-vault",
                "ScheduleExpression": "cron(0 3 ? * 1 *)",
                "StartWindowMinutes": 60,
                "CompletionWindowMinutes": 480,
                "Lifecycle": {
                    "MoveToColdStorageAfterDays": 7,
                    "DeleteAfterDays": 365
                }
            },
            {
                "RuleName": "Monthly-Backups",
                "TargetBackupVaultName": "production-vault",
                "ScheduleExpression": "cron(0 4 1 * ? *)",
                "StartWindowMinutes": 120,
                "CompletionWindowMinutes": 720,
                "Lifecycle": {
                    "MoveToColdStorageAfterDays": 30,
                    "DeleteAfterDays": 2555
                }
            }
        ]
    }'
```

```bash
# Bước 2: Gán tài nguyên vào backup plan qua Tags
aws backup create-backup-selection \
    --backup-plan-id $PLAN_ID \
    --backup-selection '{
        "SelectionName": "All-Production-Resources",
        "IamRoleArn": "arn:aws:iam::123456789012:role/AWSBackupDefaultServiceRole",
        "ListOfTags": [
            {
                "ConditionType": "STRINGEQUALS",
                "ConditionKey": "Environment",
                "ConditionValue": "production"
            },
            {
                "ConditionType": "STRINGEQUALS",
                "ConditionKey": "Backup",
                "ConditionValue": "required"
            }
        ]
    }'
```

---

## 4. Backup Vault — Kho Backup

### Tạo Backup Vault với Mã Hóa

```bash
# Tạo vault với KMS key riêng (không dùng default key)
aws backup create-backup-vault \
    --backup-vault-name production-vault \
    --encryption-key-arn arn:aws:kms:us-east-1:123456789012:key/backup-key-id \
    --backup-vault-tags '{
        "Environment": "production",
        "Purpose": "DR-backup",
        "CostCenter": "platform-team"
    }'

# Tạo DR vault ở region khác
aws backup create-backup-vault \
    --region eu-west-1 \
    --backup-vault-name dr-vault \
    --encryption-key-arn arn:aws:kms:eu-west-1:123456789012:key/dr-backup-key
```

### Vault Access Policy — Kiểm Soát Truy Cập

```bash
# Policy ngăn chặn xóa vault và recovery points
aws backup put-backup-vault-access-policy \
    --backup-vault-name production-vault \
    --policy '{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "DenyDeleteVault",
                "Effect": "Deny",
                "Principal": "*",
                "Action": [
                    "backup:DeleteBackupVault",
                    "backup:DeleteRecoveryPoint",
                    "backup:UpdateRecoveryPointLifecycle"
                ],
                "Resource": "*",
                "Condition": {
                    "StringNotEquals": {
                        "aws:PrincipalArn": [
                            "arn:aws:iam::123456789012:role/BackupAdminRole"
                        ]
                    }
                }
            }
        ]
    }'
```

---

## 5. Cross-Account Backup — Backup Liên Tài Khoản

### Tại Sao Cần Cross-Account?

```
Vấn Đề Khi Backup Trong Cùng Account:
├── Ransomware xâm nhập account → xóa cả production và backup
├── Admin xóa nhầm → xóa cả backup
├── Insider threat → truy cập và xóa backup
└── Regulatory: Nhiều tổ chức yêu cầu backup account riêng

Giải Pháp Cross-Account:
Production Account (123456789012)
        │
        │ backup copies
        ▼
Backup Account (987654321098)
├── Vault Lock bảo vệ (không ai xóa được)
├── IAM policy hạn chế chặt chẽ
└── Tách biệt hoàn toàn với production
```

### Cấu Hình Cross-Account Backup

```bash
# Trong Backup Account: Enable cross-account backup
aws backup put-backup-vault-access-policy \
    --backup-vault-name central-backup-vault \
    --policy '{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowCrossAccountCopy",
                "Effect": "Allow",
                "Principal": {
                    "AWS": "arn:aws:iam::123456789012:role/AWSBackupDefaultServiceRole"
                },
                "Action": [
                    "backup:CopyIntoBackupVault"
                ],
                "Resource": "*"
            }
        ]
    }'

# Trong Production Account: Thêm CopyActions vào backup plan
# (Đã thấy trong ví dụ trên — CopyActions → cross-account vault ARN)
```

---

## 6. Point-in-Time Recovery — PITR — Khôi Phục Tại Thời Điểm Cụ Thể

### PITR cho DynamoDB

```bash
# DynamoDB hỗ trợ PITR — restore đến bất kỳ giây nào trong 35 ngày
aws dynamodb restore-table-to-point-in-time \
    --source-table-name production-orders \
    --target-table-name production-orders-restored \
    --restore-date-time 2026-05-15T10:30:00Z \
    --use-latest-restorable-time  # hoặc specify exact time

# PITR cho RDS (database instance)
aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier prod-mysql \
    --target-db-instance-identifier prod-mysql-restored \
    --restore-time 2026-05-15T10:30:00Z \
    --db-instance-class db.r5.xlarge \
    --availability-zone us-east-1a
```

### Continuous Backup cho S3

```bash
# Enable continuous backup cho S3 (PITR cho S3)
aws backup create-backup-plan \
    --backup-plan '{
        "BackupPlanName": "S3-Continuous-Backup",
        "Rules": [
            {
                "RuleName": "S3-Continuous",
                "TargetBackupVaultName": "production-vault",
                "ScheduleExpression": "cron(0 5 ? * * *)",
                "EnableContinuousBackup": true,
                "Lifecycle": {
                    "DeleteAfterDays": 35
                }
            }
        ]
    }'
```

---

## 7. AWS Backup Audit Manager — Quản Lý Kiểm Toán Backup

### Framework và Control

```bash
# Tạo Audit Framework với các control tự động
aws backup create-framework \
    --framework-name "Production-Backup-Compliance" \
    --framework-description "Ensure all production resources are backed up" \
    --framework-controls '[
        {
            "ControlName": "BACKUP_PLAN_MIN_FREQUENCY_AND_MIN_RETENTION_CHECK",
            "ControlInputParameters": [
                {"ParameterName": "requiredFrequencyUnit", "ParameterValue": "hours"},
                {"ParameterName": "requiredFrequencyValue", "ParameterValue": "24"},
                {"ParameterName": "requiredRetentionDays", "ParameterValue": "30"}
            ]
        },
        {
            "ControlName": "BACKUP_RECOVERY_POINT_ENCRYPTED",
            "ControlInputParameters": []
        },
        {
            "ControlName": "BACKUP_RECOVERY_POINT_MINIMUM_RETENTION_CHECK",
            "ControlInputParameters": [
                {"ParameterName": "requiredRetentionDays", "ParameterValue": "30"}
            ]
        },
        {
            "ControlName": "BACKUP_LAST_RECOVERY_POINT_CREATED",
            "ControlInputParameters": [
                {"ParameterName": "recoveryPointAgeUnit", "ParameterValue": "hours"},
                {"ParameterName": "recoveryPointAgeValue", "ParameterValue": "24"}
            ]
        }
    ]'
```

### Tạo Báo Cáo Compliance Tự Động

```bash
# Tạo report plan — báo cáo tuân thủ hàng ngày
aws backup create-report-plan \
    --report-plan-name "Daily-Backup-Compliance-Report" \
    --report-delivery-channel '{
        "S3BucketName": "backup-compliance-reports",
        "S3KeyPrefix": "aws-backup/reports"
    }' \
    --report-setting '{
        "ReportTemplate": "CONTROL_COMPLIANCE_REPORT",
        "FrameworkArns": [
            "arn:aws:backup:us-east-1:123456789012:framework:Production-Backup-Compliance"
        ]
    }' \
    --report-plan-tags '{"Purpose": "compliance-audit"}'
```

---

## 8. AWS Backup với Organizations — Quản Lý Đa Tài Khoản

```
AWS Organizations + AWS Backup:
├── Backup Policy được định nghĩa ở Management Account
├── Policy tự động áp dụng cho tất cả member accounts
└── Không ai trong member account có thể bypass policy

Lợi Ích:
├── 1 policy cho 100+ accounts
├── Enforce minimum backup frequency toàn tổ chức
├── Prevent accounts từ tắt backup
└── Centralized compliance reporting
```

```bash
# Trong Management Account: Enable Backup Policy cho Organization
aws organizations enable-policy-type \
    --root-id r-xxxx \
    --policy-type BACKUP_POLICY

# Tạo Organization Backup Policy
aws organizations create-policy \
    --name "Org-Production-Backup-Policy" \
    --type BACKUP_POLICY \
    --content '{
        "plans": {
            "OrgDailyBackupPlan": {
                "regions": {"@@assign": ["us-east-1", "eu-west-1"]},
                "rules": {
                    "DailyBackups": {
                        "schedule_expression": {"@@assign": "cron(0 2 ? * * *)"},
                        "start_backup_window_minutes": {"@@assign": "60"},
                        "target_backup_vault_name": {"@@assign": "aws-backup-default"},
                        "lifecycle": {
                            "delete_after_days": {"@@assign": "30"}
                        }
                    }
                },
                "selections": {
                    "tags": {
                        "OrgBackupRequired": {
                            "iam_role_arn": {"@@assign": "arn:aws:iam::$account:role/AWSBackupDefaultServiceRole"},
                            "tag_key": {"@@assign": "Backup"},
                            "tag_value": {"@@assign": ["required"]}
                        }
                    }
                }
            }
        }
    }'
```

---

## 9. Restore từ AWS Backup

### Restore EBS Volume

```bash
# Tìm recovery point cần restore
aws backup list-recovery-points-by-backup-vault \
    --backup-vault-name production-vault \
    --by-resource-type EBS \
    --by-created-before 2026-05-16T00:00:00Z

# Restore
aws backup start-restore-job \
    --recovery-point-arn arn:aws:backup:us-east-1:123456789012:recovery-point:abc123 \
    --iam-role-arn arn:aws:iam::123456789012:role/AWSBackupDefaultServiceRole \
    --resource-type EBS \
    --metadata '{
        "availabilityZone": "us-east-1a",
        "volumeType": "gp3",
        "encrypted": "true",
        "kmsKeyId": "arn:aws:kms:us-east-1:123456789012:key/key-id"
    }'
```

### Restore RDS Instance

```bash
aws backup start-restore-job \
    --recovery-point-arn arn:aws:backup:us-east-1:123456789012:recovery-point:rds:abc123 \
    --iam-role-arn arn:aws:iam::123456789012:role/AWSBackupDefaultServiceRole \
    --resource-type RDS \
    --metadata '{
        "DBInstanceIdentifier": "prod-mysql-restored",
        "DBInstanceClass": "db.r5.xlarge",
        "AvailabilityZone": "us-east-1a",
        "MultiAZ": "false",
        "Engine": "mysql"
    }'
```

---

## 10. Chi Phí AWS Backup

```
Cấu Trúc Giá:

Backup Storage (lưu trữ backup):
├── Warm storage: $0.05/GB-month
├── Cold storage (sau 30 ngày): $0.01/GB-month
└── Không tính phí thêm cho AWS Backup service (chỉ trả storage)

Cross-Account/Cross-Region Copy:
├── Data transfer: Theo rate inter-region thông thường ($0.02/GB)
└── Storage tại destination: $0.05/GB-month

Continuous Backup (PITR cho S3):
├── $0.05/GB-month cho continuous backup storage
└── Restore request: $0.03/GB data restored

Ví Dụ (100 resources, 1TB tổng):
├── Daily backup (warm 30 ngày): 1TB × $0.05 = $50/tháng
├── Monthly backup (cold): ~500GB × $0.01 = $5/tháng
└── Cross-region copy: 100GB/ngày × $0.02 = $60/tháng data transfer
```

---

## 11. Checklist AWS Backup

```
Thiết Lập Ban Đầu:
□ Đã tạo KMS key riêng cho backup vault (không dùng AWS managed key)
□ Đã tạo vault với access policy ngăn chặn xóa trái phép
□ Đã tạo Backup Plan với đầy đủ daily/weekly/monthly rules
□ Đã gán resources bằng tag (không hardcode ARN — quản lý linh hoạt hơn)

Cross-Account/Cross-Region:
□ Đã tạo DR vault ở region khác
□ Đã cấu hình cross-account copy nếu cần
□ Đã test restore từ DR vault

Compliance:
□ Đã tạo Audit Framework với controls phù hợp regulatory
□ Đã bật Report Plan cho báo cáo hàng ngày
□ Đã xem xét Vault Lock cho WORM requirement

Vận Hành:
□ CloudWatch alarm khi backup job fails
□ SNS notification cho team
□ Monthly restore test (kiểm tra restore hàng tháng)
□ Review compliance report hàng tuần
```

---

**File Tiếp Theo:** [5-backup-vault-lock.md](5-backup-vault-lock.md) — Backup Vault Lock — WORM cho Compliance
