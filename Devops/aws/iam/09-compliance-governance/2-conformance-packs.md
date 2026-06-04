# Config Conformance Packs — Gói Quy Tắc Tuân Thủ

> Config Conformance Packs — Gói Tuân Thủ Config — là tập hợp các AWS Config Rules (Quy Tắc Config) và remediation actions (hành động khắc phục) đóng gói thành một template YAML duy nhất, có thể deploy đồng loạt cho toàn bộ tổ chức AWS trong một lệnh.

---

## 🎯 Conformance Pack Là Gì?

### Vấn Đề Trước Khi Có Conformance Packs

```
Trước đây:
├── Tạo từng Config Rule một → hàng trăm rules để quản lý riêng lẻ
├── Deploy thủ công sang từng account → tốn hàng giờ
├── Không có mapping rõ ràng với PCI-DSS control nào
├── Báo cáo tuân thủ tổng hợp phải làm thủ công
└── Không nhất quán giữa Dev/Staging/Production environments
```

### Sau Khi Có Conformance Packs

```
Conformance Pack:
├── Đóng gói 50-100+ Config Rules vào một template
├── Deploy một lệnh → áp dụng cho toàn bộ Organizations
├── Mỗi rule ánh xạ rõ ràng với control ID của framework
├── Dashboard tuân thủ tổng hợp ngay lập tức
└── Nhất quán 100% giữa mọi environments và accounts
```

---

## 🏗️ Kiến Trúc Conformance Pack

```
┌─────────────────────────────────────────────────────────────┐
│               AWS Organizations Management Account           │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │         Conformance Pack Template (YAML)             │    │
│  │  ├── PCI-DSS Conformance Pack                       │    │
│  │  ├── HIPAA Conformance Pack                         │    │
│  │  └── Custom Conformance Pack                        │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│               deploy to Organizations                        │
└───────────────────────────┼──────────────────────────────────┘
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  Account A   │  │  Account B   │  │  Account C   │
  │  (Prod)      │  │  (Staging)   │  │  (Dev)       │
  │              │  │              │  │              │
  │ Config Rules │  │ Config Rules │  │ Config Rules │
  │ (Tự động     │  │ (Tự động     │  │ (Tự động     │
  │  được tạo)   │  │  được tạo)   │  │  được tạo)   │
  └──────────────┘  └──────────────┘  └──────────────┘
```

---

## 📦 Prebuilt Conformance Packs (Gói Có Sẵn)

### Danh Sách Gói Phổ Biến

```bash
# Xem tất cả sample conformance packs
aws configservice list-conformance-pack-compliance-scores \
  --query 'ConformancePackComplianceScores[*].ConformancePackName'

# Tải về sample templates từ AWS GitHub
# https://github.com/awslabs/aws-config-rules/tree/master/aws-config-conformance-packs
```

| Tên Gói | Tiêu Chuẩn | Số Rules |
|---|---|---|
| `operational-best-practices-for-pci-dss` | PCI-DSS v3.2.1 | 70+ |
| `operational-best-practices-for-hipaa-security` | HIPAA Security Rule | 45+ |
| `operational-best-practices-for-cis-level-1` | CIS AWS Benchmark v1.4 | 43 |
| `operational-best-practices-for-cis-level-2` | CIS AWS Benchmark v1.4 | 53 |
| `operational-best-practices-for-soc2` | SOC 2 | 60+ |
| `operational-best-practices-for-nist-csf` | NIST Cybersecurity Framework | 80+ |
| `operational-best-practices-for-iso-27001` | ISO 27001:2013 | 55+ |
| `operational-best-practices-for-gdpr` | GDPR | 25+ |

---

## 📝 Cấu Trúc Template YAML

### Template Đơn Giản

```yaml
# pci-dss-conformance-pack.yaml
Parameters:
  AccessKeysRotatedParamMaxAccessKeyAge:
    Default: "90"
    Type: String
    Description: "Số ngày tối đa trước khi access key phải được rotate"

Resources:
  # PCI-DSS Requirement 8: Identify and authenticate access
  IamRootAccessKeyCheck:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: iam-root-access-key-check
      Description: "Đảm bảo root account không có access keys"
      Source:
        Owner: AWS
        SourceIdentifier: IAM_ROOT_ACCESS_KEY_CHECK
      # Ánh xạ control: PCI-DSS Req 8.3.1

  AccessKeysRotated:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: access-keys-rotated
      Description: "Access keys phải được rotate trong vòng 90 ngày"
      InputParameters:
        maxAccessKeyAge: !Ref AccessKeysRotatedParamMaxAccessKeyAge
      Source:
        Owner: AWS
        SourceIdentifier: ACCESS_KEYS_ROTATED

  # PCI-DSS Requirement 3: Protect stored cardholder data
  S3BucketServerSideEncryptionEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-bucket-server-side-encryption-enabled
      Description: "Tất cả S3 buckets phải bật server-side encryption"
      Source:
        Owner: AWS
        SourceIdentifier: S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED

  S3BucketPublicReadProhibited:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-bucket-public-read-prohibited
      Description: "S3 buckets không được phép public read access"
      Source:
        Owner: AWS
        SourceIdentifier: S3_BUCKET_PUBLIC_READ_PROHIBITED

  # PCI-DSS Requirement 10: Log and monitor all access
  CloudTrailEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: cloud-trail-enabled
      Description: "CloudTrail phải được bật"
      Source:
        Owner: AWS
        SourceIdentifier: CLOUD_TRAIL_ENABLED

  CloudTrailEncryptionEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: cloud-trail-encryption-enabled
      Description: "CloudTrail logs phải được mã hóa bằng KMS"
      Source:
        Owner: AWS
        SourceIdentifier: CLOUD_TRAIL_ENCRYPTION_ENABLED

  # PCI-DSS Requirement 6: Protect against known vulnerabilities
  Ec2SecurityGroupAttachedToEni:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: ec2-security-group-attached-to-eni
      Description: "Security groups phải được gắn với ENI"
      Source:
        Owner: AWS
        SourceIdentifier: EC2_SECURITY_GROUP_ATTACHED_TO_ENI
```

### Template Với Remediation Action (Tự Động Khắc Phục)

```yaml
Resources:
  S3BucketPublicReadProhibited:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-bucket-public-read-prohibited
      Source:
        Owner: AWS
        SourceIdentifier: S3_BUCKET_PUBLIC_READ_PROHIBITED

  # Tự động khắc phục: Tắt public access khi phát hiện vi phạm
  RemediationForS3BucketPublicRead:
    DependsOn: S3BucketPublicReadProhibited
    Type: AWS::Config::RemediationConfiguration
    Properties:
      ConfigRuleName: s3-bucket-public-read-prohibited
      TargetType: SSM_DOCUMENT
      TargetId: AWS-DisableS3BucketPublicReadWrite
      Parameters:
        AutomationAssumeRole:
          StaticValue:
            Values:
              - arn:aws:iam::ACCOUNT_ID:role/RemediationRole
        S3BucketName:
          ResourceValue:
            Value: RESOURCE_ID
      Automatic: true
      MaximumAutomaticAttempts: 3
      RetryAttemptSeconds: 60
```

---

## 🚀 Deploy Conformance Pack

### Deploy Cho Một Account

```bash
# Deploy từ template local
aws configservice put-conformance-pack \
  --conformance-pack-name "PCI-DSS-Production" \
  --template-body file://pci-dss-conformance-pack.yaml \
  --delivery-s3-bucket "my-conformance-pack-bucket" \
  --delivery-s3-key-prefix "pci-dss/"

# Deploy từ S3 template
aws configservice put-conformance-pack \
  --conformance-pack-name "PCI-DSS-Production" \
  --template-s3-uri "s3://my-templates-bucket/pci-dss-template.yaml" \
  --delivery-s3-bucket "my-conformance-pack-bucket"

# Deploy từ AWS sample templates (SSM Document)
aws configservice put-conformance-pack \
  --conformance-pack-name "CIS-Level-1" \
  --template-ssm-document-details '{"documentName": "AWSConfigConformsPack-OB-CIS-Level1"}'
```

### Deploy Cho Toàn Bộ Organizations

```bash
# Deploy từ Organizations management account
aws configservice put-organization-conformance-pack \
  --organization-conformance-pack-name "PCI-DSS-All-Accounts" \
  --template-s3-uri "s3://my-templates-bucket/pci-dss-template.yaml" \
  --delivery-s3-bucket "org-conformance-results" \
  --excluded-accounts "sandbox-account-id-111" \
  --conformance-pack-input-parameters '[
    {
      "ParameterName": "AccessKeysRotatedParamMaxAccessKeyAge",
      "ParameterValue": "90"
    }
  ]'

# Kiểm tra trạng thái deploy
aws configservice get-organization-conformance-pack-detailed-status \
  --organization-conformance-pack-name "PCI-DSS-All-Accounts"
```

---

## 📊 Xem Kết Quả Tuân Thủ

### Compliance Score (Điểm Tuân Thủ)

```bash
# Xem compliance score tổng thể
aws configservice get-conformance-pack-compliance-summary \
  --conformance-pack-names "PCI-DSS-Production"

# Output mẫu:
# {
#   "ConformancePackComplianceSummaryList": [{
#     "ConformancePackName": "PCI-DSS-Production",
#     "ConformancePackComplianceStatus": {
#       "ConformancePackName": "PCI-DSS-Production",
#       "ConformancePackComplianceStatus": "NON_COMPLIANT",
#       "LastUpdateTime": "2024-10-15T08:30:00Z"
#     }
#   }]
# ]
```

### Chi Tiết Từng Rule

```bash
# Xem detail compliance per rule
aws configservice get-conformance-pack-compliance-details \
  --conformance-pack-name "PCI-DSS-Production" \
  --filters '{
    "ConfigRuleNames": ["s3-bucket-public-read-prohibited"],
    "ComplianceType": "NON_COMPLIANT"
  }'

# Xem compliance score với điểm số cụ thể
aws configservice get-conformance-pack-compliance-scores \
  --conformance-pack-names "PCI-DSS-Production" \
  --query 'ConformancePackComplianceScores[*].{Pack:ConformancePackName, Score:Score}'
```

### Athena Query Phân Tích Compliance Data

```sql
-- Truy vấn từ CloudTrail Lake hoặc Config Data Export
-- Tìm tất cả non-compliant resources
SELECT
  resourceType,
  resourceId,
  awsRegion,
  configRuleName,
  complianceType,
  lastResultRecordedTime
FROM
  aws_config_conformance_pack_compliance
WHERE
  conformancePackName = 'PCI-DSS-Production'
  AND complianceType = 'NON_COMPLIANT'
ORDER BY
  lastResultRecordedTime DESC;

-- Tổng hợp tỷ lệ tuân thủ theo rule
SELECT
  configRuleName,
  COUNT(CASE WHEN complianceType = 'COMPLIANT' THEN 1 END) as compliant_count,
  COUNT(CASE WHEN complianceType = 'NON_COMPLIANT' THEN 1 END) as non_compliant_count,
  ROUND(
    COUNT(CASE WHEN complianceType = 'COMPLIANT' THEN 1 END) * 100.0 /
    COUNT(*), 2
  ) as compliance_percentage
FROM aws_config_conformance_pack_compliance
WHERE conformancePackName = 'PCI-DSS-Production'
GROUP BY configRuleName
ORDER BY compliance_percentage ASC;
```

---

## 🔧 Custom Conformance Pack Thực Tế

### Ví Dụ: Gói Tuân Thủ Nội Bộ (Internal Security Baseline)

```yaml
# internal-security-baseline.yaml
# Kết hợp các rules bắt buộc theo chính sách bảo mật nội bộ

Parameters:
  MaxPasswordAge:
    Default: "90"
    Type: String
  MinPasswordLength:
    Default: "14"
    Type: String
  RootAccountMFAEnabled:
    Default: "true"
    Type: String

Resources:
  # === IAM Security ===
  RootAccountMFAEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: root-account-mfa-enabled
      Source:
        Owner: AWS
        SourceIdentifier: ROOT_ACCOUNT_MFA_ENABLED

  IamPasswordPolicyMinLength:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: iam-password-policy
      InputParameters:
        MinimumPasswordLength: !Ref MinPasswordLength
        MaxPasswordAge: !Ref MaxPasswordAge
        PasswordReusePrevention: "24"
        RequireUppercaseCharacters: "true"
        RequireLowercaseCharacters: "true"
        RequireNumbers: "true"
        RequireSymbols: "true"
      Source:
        Owner: AWS
        SourceIdentifier: IAM_PASSWORD_POLICY

  MfaEnabledForIamConsoleAccess:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: mfa-enabled-for-iam-console-access
      Source:
        Owner: AWS
        SourceIdentifier: MFA_ENABLED_FOR_IAM_CONSOLE_ACCESS

  # === S3 Security ===
  S3AccountLevelPublicAccessBlocksPeriodic:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-account-level-public-access-blocks-periodic
      InputParameters:
        BlockPublicAcls: "true"
        BlockPublicPolicy: "true"
        IgnorePublicAcls: "true"
        RestrictPublicBuckets: "true"
      Source:
        Owner: AWS
        SourceIdentifier: S3_ACCOUNT_LEVEL_PUBLIC_ACCESS_BLOCKS_PERIODIC

  S3BucketSslRequestsOnly:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-bucket-ssl-requests-only
      Description: "S3 bucket phải từ chối HTTP requests, chỉ chấp nhận HTTPS"
      Source:
        Owner: AWS
        SourceIdentifier: S3_BUCKET_SSL_REQUESTS_ONLY

  # === Encryption ===
  Ec2EbsEncryptionByDefault:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: ec2-ebs-encryption-by-default
      Source:
        Owner: AWS
        SourceIdentifier: EC2_EBS_ENCRYPTION_BY_DEFAULT

  RdsStorageEncrypted:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: rds-storage-encrypted
      Source:
        Owner: AWS
        SourceIdentifier: RDS_STORAGE_ENCRYPTED

  KmsCmkNotScheduledForDeletion:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: kms-cmk-not-scheduled-for-deletion
      Source:
        Owner: AWS
        SourceIdentifier: KMS_CMK_NOT_SCHEDULED_FOR_DELETION

  # === Network Security ===
  RestrictedSsh:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: restricted-ssh
      Description: "Security Groups không được allow SSH từ 0.0.0.0/0"
      Source:
        Owner: AWS
        SourceIdentifier: INCOMING_SSH_DISABLED

  RestrictedCommonPorts:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: restricted-common-ports
      InputParameters:
        blockedPort1: "20"
        blockedPort2: "21"
        blockedPort3: "3389"
        blockedPort4: "3306"
        blockedPort5: "4333"
      Source:
        Owner: AWS
        SourceIdentifier: RESTRICTED_INCOMING_TRAFFIC

  # === Monitoring ===
  CloudTrailEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: cloud-trail-enabled
      Source:
        Owner: AWS
        SourceIdentifier: CLOUD_TRAIL_ENABLED

  GuarddutyEnabledCentralized:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: guardduty-enabled-centralized
      Source:
        Owner: AWS
        SourceIdentifier: GUARDDUTY_ENABLED_CENTRALIZED

  SecurityhubEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: securityhub-enabled
      Source:
        Owner: AWS
        SourceIdentifier: SECURITYHUB_ENABLED
```

---

## 🔄 Vòng Đời Conformance Pack

```
Deploy → Evaluate → Alert → Remediate → Report

1. Deploy:
   Conformance Pack template → CloudFormation stack → Config Rules được tạo

2. Evaluate:
   Config Rules chạy theo trigger:
   ├── Periodic (Định Kỳ): Mỗi 1/3/6/12/24 giờ
   └── Change-triggered (Theo Thay Đổi): Khi tài nguyên thay đổi

3. Alert:
   NON_COMPLIANT → EventBridge event → SNS → Email/Slack notification

4. Remediate:
   Auto: RemediationConfiguration chạy SSM Automation document
   Manual: DevOps team nhận notification và fix thủ công

5. Report:
   Security Hub tổng hợp → Audit Manager thu thập → Báo cáo tuân thủ
```

### Tự Động Alert Khi Vi Phạm

```bash
# EventBridge rule để bắt Config compliance events
aws events put-rule \
  --name "ConformancePackViolation" \
  --event-pattern '{
    "source": ["aws.config"],
    "detail-type": ["Config Rules Compliance Change"],
    "detail": {
      "messageType": ["ComplianceChangeNotification"],
      "newEvaluationResult": {
        "complianceType": ["NON_COMPLIANT"]
      },
      "configRuleName": [{
        "prefix": "pci-dss-"
      }]
    }
  }' \
  --state ENABLED

# Gắn SNS target
aws events put-targets \
  --rule "ConformancePackViolation" \
  --targets '[{
    "Id": "SecurityTeamNotification",
    "Arn": "arn:aws:sns:us-east-1:123456789012:security-alerts"
  }]'
```

---

## 💰 Chi Phí

```
Config Rule Evaluations (Đánh Giá Quy Tắc):
├── First 100,000 rule evaluations/month: $0.001/evaluation
├── Next 400,000: $0.0008/evaluation  
├── Over 500,000: $0.0005/evaluation
└── Conformance Packs: Không phí thêm ngoài Config Rules

Ước Tính Chi Phí:
├── 50 Config Rules × 1,000 resources × 30 ngày = 1,500,000 evaluations
├── Chi phí: $100 × $0.001 + $400 × $0.0008 + $1000 × $0.0005 = ~$0.92/ngày
└── ~$28/tháng cho 50 rules, 1,000 resources, đánh giá hàng ngày

Tối Ưu Chi Phí:
├── Dùng change-triggered rules thay vì periodic khi có thể
├── Giới hạn resource types trong scope của rule
└── Tắt rules không cần thiết trong Dev environment
```

---

## 📌 Best Practices

### 1. Phân Tầng Conformance Packs Theo Môi Trường

```yaml
# Chiến Lược Phân Tầng:
Production:
  - PCI-DSS Conformance Pack (đầy đủ, 70+ rules)
  - Internal Security Baseline (bắt buộc)
  - Auto-remediation: BẬT cho critical controls

Staging:
  - PCI-DSS Conformance Pack (đầy đủ)
  - Internal Security Baseline
  - Auto-remediation: TẮT (để developer test)

Development:
  - Internal Security Baseline chỉ (basic controls)
  - Auto-remediation: TẮT
  - Thêm exception cho một số rules (VD: public IP allowed)
```

### 2. Versioning Cho Templates

```bash
# Lưu template vào S3 với versioning
aws s3api put-bucket-versioning \
  --bucket my-conformance-templates \
  --versioning-configuration Status=Enabled

# Upload version mới với ngày tháng
aws s3 cp pci-dss-template.yaml \
  s3://my-conformance-templates/pci-dss/v2.1/pci-dss-template.yaml

# Khi update, đổi version trong conformance pack name
# "PCI-DSS-v2-Production" → "PCI-DSS-v21-Production"
```

### 3. Integration Với Security Hub

```
Config Conformance Pack → Security Hub Integration:

Config Rules trong Conformance Pack tự động:
├── Gửi findings vào Security Hub (nếu Security Hub enabled)
├── Findings được phân loại theo Security Standard
├── Compliance score Security Hub tự động cập nhật
└── ASFF (Amazon Security Finding Format) được dùng nhất quán
```

---

## 💡 Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa Conformance Pack và Security Hub Standards?**
> Conformance Pack là tập hợp Config Rules được đóng gói để deploy và quản lý tập trung — bạn kiểm soát rules nào trong pack. Security Hub Standards (CIS, FSBP, PCI-DSS) là bộ kiểm tra được AWS quản lý hoàn toàn, tích hợp sẵn vào Security Hub với scoring. Hai thứ bổ sung cho nhau: Conformance Pack cho enforcement và audit evidence, Security Hub Standards cho visibility và scoring.

**Q: Làm thế nào để update Conformance Pack mà không gây downtime?**
> Update `put-conformance-pack` với template mới — Config xử lý việc thêm/xóa/sửa rules mà không ảnh hưởng đến evaluations đang chạy. Rules mới sẽ chạy evaluation đầu tiên sau khi deploy. Rules bị xóa sẽ ngừng đánh giá nhưng history vẫn giữ nguyên. Không có downtime với dịch vụ khác.

**Q: Conformance Pack có thể deploy custom Config Rules không?**
> Có. Bạn có thể include custom Lambda-backed Config Rules trong Conformance Pack template. Lambda function cần deploy trước, và rule trong pack tham chiếu ARN của Lambda. Đây là cách tạo gói tuân thủ tùy chỉnh kết hợp managed rules của AWS và rules kiểm tra logic nghiệp vụ riêng.

---

## 🔗 Điều Hướng

| Trước | Hiện Tại | Sau |
|---|---|---|
| [1-audit-manager.md](1-audit-manager.md) | **2-conformance-packs.md** | [3-firewall-manager.md](3-firewall-manager.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
