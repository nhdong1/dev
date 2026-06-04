# Conformance Packs — Gói Tuân Thủ Framework

> **Conformance Pack** (Gói Tuân Thủ) là bộ sưu tập các Config Rules và Remediation Actions được đóng gói lại theo một compliance framework tiêu chuẩn (CIS, PCI-DSS, NIST, HIPAA...). Thay vì bật từng rule riêng lẻ, bạn triển khai toàn bộ framework chỉ với một lần deploy.

---

## 📚 Mục Lục

1. [Conformance Pack Là Gì?](#conformance-pack-là-gì)
2. [Lợi Ích So Với Config Rules Đơn Lẻ](#lợi-ích-so-với-config-rules-đơn-lẻ)
3. [AWS Sample Conformance Packs](#aws-sample-conformance-packs)
4. [CIS AWS Foundations Benchmark](#cis-aws-foundations-benchmark)
5. [PCI-DSS Conformance Pack](#pci-dss-conformance-pack)
6. [NIST 800-53 Conformance Pack](#nist-800-53-conformance-pack)
7. [HIPAA Conformance Pack](#hipaa-conformance-pack)
8. [Tạo Custom Conformance Pack](#tạo-custom-conformance-pack)
9. [Triển Khai Qua Organizations](#triển-khai-qua-organizations)
10. [Scoring & Reporting — Báo Cáo Tuân Thủ](#scoring--reporting--báo-cáo-tuân-thủ)
11. [Thực Hành Tốt Nhất](#thực-hành-tốt-nhất)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Conformance Pack Là Gì?

### Định Nghĩa

**Conformance Pack** là một template YAML định nghĩa:
- Tập hợp **Config Rules** (Managed hoặc Custom)
- **Remediation Configurations** (tùy chọn)
- **Input parameters** cho từng rule

```yaml
# Ví dụ cấu trúc conformance pack đơn giản
Parameters:
  AccessKeysRotatedParamMaxAccessKeyAge:
    Default: '90'
    Type: String

Resources:
  # Rule 1: Access keys phải rotate trong 90 ngày
  AccessKeysRotated:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: access-keys-rotated
      Source:
        Owner: AWS
        SourceIdentifier: ACCESS_KEYS_ROTATED
      InputParameters:
        maxAccessKeyAge: !Ref AccessKeysRotatedParamMaxAccessKeyAge

  # Rule 2: CloudTrail phải bật
  CloudTrailEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: cloudtrail-enabled
      Source:
        Owner: AWS
        SourceIdentifier: CLOUD_TRAIL_ENABLED

  # Rule 3: MFA bắt buộc cho root account
  RootAccountMFAEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: root-account-mfa-enabled
      Source:
        Owner: AWS
        SourceIdentifier: ROOT_ACCOUNT_MFA_ENABLED
```

### Vòng Đời Conformance Pack

```
1. Tạo template YAML (tự viết hoặc dùng AWS sample)
       ↓
2. Deploy lên S3 hoặc trực tiếp qua console/CLI
       ↓
3. AWS Config triển khai tất cả rules trong pack
       ↓
4. Config đánh giá tài nguyên theo từng rule
       ↓
5. Xem compliance score và chi tiết trên console
       ↓
6. Xuất báo cáo (export) cho auditor
```

---

## Lợi Ích So Với Config Rules Đơn Lẻ

| Tiêu Chí | Config Rules Đơn Lẻ | Conformance Pack |
|---------|--------------------|--------------------|
| **Triển khai** | Từng rule một | Toàn bộ framework một lần |
| **Xóa** | Từng rule một | Xóa pack → tất cả rules bị xóa |
| **Versioning** | Khó quản lý | Template YAML có thể version trên Git |
| **Báo cáo** | Phải tổng hợp thủ công | Compliance score tự động |
| **Organization-wide** | Phức tạp | StackSets tự động |
| **Audit mapping** | Tự map sang control | Đã map sẵn (CIS control 1.x → rule) |

---

## AWS Sample Conformance Packs

AWS cung cấp sẵn hơn 30 sample conformance packs, lưu tại:

```
https://github.com/awslabs/aws-config-rules/tree/master/aws-config-conformance-packs
```

Hoặc trên console: **AWS Config → Conformance Packs → Add conformance pack → Use sample template**

### Danh Sách Sample Packs

| Pack | Framework | Số Rules |
|------|-----------|---------|
| `operational-best-practices-for-cis-aws-v1.4-level1` | CIS v1.4 Level 1 | 20+ |
| `operational-best-practices-for-cis-aws-v1.4-level2` | CIS v1.4 Level 2 | 35+ |
| `operational-best-practices-for-pci-dss` | PCI-DSS v3.2.1 | 50+ |
| `operational-best-practices-for-nist-800-53` | NIST SP 800-53 | 70+ |
| `operational-best-practices-for-hipaa-security` | HIPAA Security | 40+ |
| `operational-best-practices-for-soc2` | SOC 2 | 30+ |
| `operational-best-practices-for-fsbp` | AWS Foundational Security Best Practices | 100+ |
| `operational-best-practices-for-gdpr` | GDPR | 20+ |

---

## CIS AWS Foundations Benchmark

**CIS — Center for Internet Security AWS Foundations Benchmark** (Tiêu Chuẩn Bảo Mật Nền Tảng AWS của Trung Tâm An Ninh Internet) là bộ tiêu chuẩn bảo mật phổ biến nhất cho AWS, dùng rộng rãi trong ngành.

### CIS Benchmark Phiên Bản Hiện Hành

- **CIS AWS Foundations Benchmark v1.4** — Phiên bản phổ biến nhất hiện tại
- **CIS AWS Foundations Benchmark v2.0** — Phiên bản mới nhất (2023)

### CIS Level 1 vs Level 2

| Level | Mô Tả | Ứng Dụng |
|-------|-------|---------|
| **Level 1** | Controls cơ bản, ít ảnh hưởng đến tính năng | Mọi môi trường |
| **Level 2** | Controls chuyên sâu, có thể ảnh hưởng chức năng | Môi trường yêu cầu bảo mật cao |

### CIS Controls Quan Trọng và Rules Tương Ứng

```
CIS Control 1 — Identity and Access Management
├── 1.1  root-account-mfa-enabled          (Bật MFA cho root)
├── 1.4  iam-root-access-key-check         (Root không có access key)
├── 1.5  iam-password-policy               (Password policy đủ mạnh)
├── 1.9  access-keys-rotated               (Rotate access key <90 ngày)
└── 1.16 iam-no-inline-policy-check        (Không dùng inline policies)

CIS Control 2 — Storage
├── 2.1.1 s3-bucket-public-read-prohibited  (S3 không public read)
├── 2.1.2 s3-bucket-public-write-prohibited (S3 không public write)
└── 2.2.1 s3-bucket-server-side-encryption-enabled (S3 SSE bật)

CIS Control 3 — Logging
├── 3.1  cloudtrail-enabled                (CloudTrail bật)
├── 3.2  cloud-trail-log-file-validation-enabled (Log validation bật)
├── 3.4  cloudtrail-encryption-enabled     (CloudTrail mã hóa KMS)
└── 3.9  vpc-flow-logs-enabled             (VPC Flow Logs bật)

CIS Control 4 — Monitoring (CloudWatch Alarms)
├── 4.1  Alarm cho unauthorized API calls
├── 4.2  Alarm cho console signin without MFA
└── 4.x  Alarm cho mọi thay đổi nhạy cảm
```

### Deploy CIS Level 1 Pack

```bash
# Deploy từ AWS sample
aws configservice put-conformance-pack \
  --conformance-pack-name "cis-aws-foundations-level1" \
  --template-s3-uri "s3://my-config-templates/cis-level1.yaml" \
  --delivery-s3-bucket "my-conformance-results-bucket"
```

---

## PCI-DSS Conformance Pack

**PCI-DSS — Payment Card Industry Data Security Standard** (Tiêu Chuẩn Bảo Mật Dữ Liệu Ngành Thẻ Thanh Toán) bắt buộc cho mọi hệ thống xử lý thẻ tín dụng.

### 12 PCI-DSS Requirements và Controls AWS Tương Ứng

| PCI Requirement | Mô Tả | Config Rules |
|----------------|-------|-------------|
| **Req 1** | Bảo vệ network (firewall) | `vpc-sg-open-only-to-authorized-ports` |
| **Req 2** | Không dùng default passwords | `ec2-managedinstance-no-applications-required` |
| **Req 3** | Bảo vệ stored cardholder data | `rds-storage-encrypted`, `encrypted-volumes` |
| **Req 4** | Mã hóa data in transit | `acm-certificate-expiration-check` |
| **Req 6** | Bảo vệ và cập nhật hệ thống | `ec2-managedinstance-patch-compliance-status-check` |
| **Req 7** | Hạn chế access theo "need to know" | `iam-policy-no-statements-with-admin-access` |
| **Req 8** | Unique IDs, MFA | `mfa-enabled-for-iam-console-access` |
| **Req 10** | Logging & monitoring | `cloudtrail-enabled`, `vpc-flow-logs-enabled` |
| **Req 11** | Test bảo mật thường xuyên | `guardduty-enabled-centralized` |

### Ví Dụ PCI-DSS Rules Trong Conformance Pack

```yaml
# Một phần của PCI-DSS conformance pack
Resources:
  # PCI Req 3: Mã hóa dữ liệu thẻ
  RDSEncrypted:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: pci-rds-storage-encrypted
      Source:
        Owner: AWS
        SourceIdentifier: RDS_STORAGE_ENCRYPTED

  EBSEncrypted:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: pci-encrypted-volumes
      Source:
        Owner: AWS
        SourceIdentifier: ENCRYPTED_VOLUMES

  # PCI Req 8: MFA bắt buộc
  MFAEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: pci-mfa-enabled-for-iam-console-access
      Source:
        Owner: AWS
        SourceIdentifier: MFA_ENABLED_FOR_IAM_CONSOLE_ACCESS

  # PCI Req 10: Logging bắt buộc
  CloudTrailEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: pci-cloudtrail-enabled
      Source:
        Owner: AWS
        SourceIdentifier: CLOUD_TRAIL_ENABLED

  VPCFlowLogsEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: pci-vpc-flow-logs-enabled
      Source:
        Owner: AWS
        SourceIdentifier: VPC_FLOW_LOGS_ENABLED
```

---

## NIST 800-53 Conformance Pack

**NIST SP 800-53 — National Institute of Standards and Technology Special Publication 800-53** (Hướng Dẫn Kiểm Soát Bảo Mật và Quyền Riêng Tư của Viện Tiêu Chuẩn và Công Nghệ Quốc Gia Hoa Kỳ) được dùng trong các hệ thống chính phủ và cơ sở hạ tầng quan trọng.

### NIST 800-53 Control Families

| Family | Mã | Ví Dụ Controls |
|--------|---|---------------|
| **Access Control** | AC | Least privilege, MFA, session management |
| **Audit and Accountability** | AU | Audit logging, log protection, review |
| **Configuration Management** | CM | Baseline configuration, change control |
| **Identification and Authentication** | IA | User identification, authenticator management |
| **Incident Response** | IR | Incident handling, testing |
| **System Protection** | SC | Boundary protection, encryption |
| **System and Information Integrity** | SI | Malicious code protection, monitoring |

### NIST Controls và Config Rules

```
NIST AC-2  (Account Management)
  → iam-user-no-policies-check
  → iam-no-inline-policy-check

NIST AC-3  (Access Enforcement)
  → iam-policy-no-statements-with-admin-access
  → s3-bucket-policy-not-more-permissive

NIST AU-2  (Event Logging)
  → cloudtrail-enabled
  → multi-region-cloudtrail-enabled
  → vpc-flow-logs-enabled

NIST CM-2  (Baseline Configuration)
  → ec2-managedinstance-association-compliance-status-check
  → ec2-managedinstance-patch-compliance-status-check

NIST SC-7  (Boundary Protection)
  → restricted-ssh
  → restricted-common-ports
  → vpc-default-security-group-closed

NIST SC-28 (Protection of Information at Rest)
  → encrypted-volumes
  → rds-storage-encrypted
  → s3-bucket-server-side-encryption-enabled
```

---

## HIPAA Conformance Pack

**HIPAA — Health Insurance Portability and Accountability Act** (Đạo Luật Về Khả Năng Chuyển Đổi và Trách Nhiệm Giải Trình Bảo Hiểm Y Tế) yêu cầu bảo vệ thông tin y tế cá nhân (PHI — Protected Health Information).

### HIPAA Security Rule Controls Quan Trọng

| HIPAA Control | Mô Tả | Config Rules |
|--------------|-------|-------------|
| **164.312(a)(1)** | Access control (kiểm soát truy cập) | `iam-root-access-key-check`, `mfa-enabled-for-iam-console-access` |
| **164.312(b)** | Audit controls (kiểm soát kiểm toán) | `cloudtrail-enabled`, `cloud-trail-log-file-validation-enabled` |
| **164.312(c)(1)** | Integrity (toàn vẹn dữ liệu) | `s3-bucket-ssl-requests-only` |
| **164.312(e)(2)(ii)** | Encryption in transit | `acm-certificate-expiration-check`, `elb-tls-https-listeners-only` |
| **164.312(a)(2)(iv)** | Encryption at rest | `rds-storage-encrypted`, `dynamodb-table-encrypted-at-rest` |

---

## Tạo Custom Conformance Pack

### Khi Nào Cần Custom Pack?

- AWS sample pack không đủ rules cho internal policy
- Cần kết hợp rules từ nhiều framework khác nhau
- Cần rules riêng cho business requirements (VD: tất cả EC2 phải có cost-center tag)
- Cần điều chỉnh parameters của managed rules (VD: password length = 14 thay vì 12)

### Cấu Trúc Template Custom Conformance Pack

```yaml
# custom-security-baseline.yaml
Parameters:
  # Tham số có thể override khi deploy
  MaxAccessKeyAge:
    Default: '90'
    Type: String
  RequiredTag1Key:
    Default: 'Environment'
    Type: String
  RequiredTag2Key:
    Default: 'CostCenter'
    Type: String
  ApprovedAMIs:
    Default: 'ami-0abc123,ami-0def456'
    Type: String

Resources:
  # ===== SECTION 1: Identity & Access Management =====

  RootMFAEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: root-account-mfa-enabled
      Description: Root account phải bật MFA
      Source:
        Owner: AWS
        SourceIdentifier: ROOT_ACCOUNT_MFA_ENABLED

  IAMPasswordPolicy:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: iam-password-policy
      Description: Password policy phải đủ mạnh
      Source:
        Owner: AWS
        SourceIdentifier: IAM_PASSWORD_POLICY
      InputParameters:
        RequireUppercaseCharacters: 'true'
        RequireLowercaseCharacters: 'true'
        RequireSymbols: 'true'
        RequireNumbers: 'true'
        MinimumPasswordLength: '14'
        PasswordReusePrevention: '24'
        MaxPasswordAge: '90'

  AccessKeysRotated:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: access-keys-rotated
      Source:
        Owner: AWS
        SourceIdentifier: ACCESS_KEYS_ROTATED
      InputParameters:
        maxAccessKeyAge: !Ref MaxAccessKeyAge

  # ===== SECTION 2: Storage Security =====

  S3PublicReadProhibited:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-bucket-public-read-prohibited
      Source:
        Owner: AWS
        SourceIdentifier: S3_BUCKET_PUBLIC_READ_PROHIBITED

  S3ServerSideEncryption:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-bucket-server-side-encryption-enabled
      Source:
        Owner: AWS
        SourceIdentifier: S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED

  EBSEncrypted:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: encrypted-volumes
      Source:
        Owner: AWS
        SourceIdentifier: ENCRYPTED_VOLUMES
      Scope:
        ComplianceResourceTypes:
          - AWS::EC2::Volume

  # ===== SECTION 3: Logging & Monitoring =====

  CloudTrailEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: cloudtrail-enabled
      Source:
        Owner: AWS
        SourceIdentifier: CLOUD_TRAIL_ENABLED

  VPCFlowLogs:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: vpc-flow-logs-enabled
      Source:
        Owner: AWS
        SourceIdentifier: VPC_FLOW_LOGS_ENABLED

  # ===== SECTION 4: Tagging =====

  RequiredTags:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: required-tags
      Source:
        Owner: AWS
        SourceIdentifier: REQUIRED_TAGS
      Scope:
        ComplianceResourceTypes:
          - AWS::EC2::Instance
          - AWS::RDS::DBInstance
          - AWS::S3::Bucket
          - AWS::Lambda::Function
      InputParameters:
        tag1Key: !Ref RequiredTag1Key
        tag2Key: !Ref RequiredTag2Key
```

### Deploy Custom Conformance Pack

```bash
# Upload template lên S3
aws s3 cp custom-security-baseline.yaml \
  s3://my-config-templates/custom-security-baseline.yaml

# Deploy conformance pack với parameter override
aws configservice put-conformance-pack \
  --conformance-pack-name "company-security-baseline" \
  --template-s3-uri "s3://my-config-templates/custom-security-baseline.yaml" \
  --delivery-s3-bucket "my-conformance-results-bucket" \
  --conformance-pack-input-parameters '[
    {"ParameterName": "MaxAccessKeyAge", "ParameterValue": "60"},
    {"ParameterName": "RequiredTag1Key", "ParameterValue": "Environment"},
    {"ParameterName": "RequiredTag2Key", "ParameterValue": "CostCenter"}
  ]'

# Kiểm tra trạng thái
aws configservice describe-conformance-packs \
  --conformance-pack-names "company-security-baseline"
```

---

## Triển Khai Qua Organizations

**Organization Conformance Packs** cho phép deploy pack cho toàn bộ organization với một lần thao tác.

### Điều Kiện Tiên Quyết

1. AWS Config phải được bật trong **tất cả member accounts**
2. Account quản lý Organization phải là **delegated administrator** cho Config
3. **Trusted access** phải được bật cho Config trong Organizations

### Deploy Organization Conformance Pack

```bash
# Bật trusted access cho Config trong Organizations
aws organizations enable-aws-service-access \
  --service-principal config-multiaccountsetup.amazonaws.com

# Deploy conformance pack cho toàn bộ organization
aws configservice put-organization-conformance-pack \
  --organization-conformance-pack-name "org-security-baseline" \
  --template-s3-uri "s3://my-config-templates/security-baseline.yaml" \
  --delivery-s3-bucket "central-config-results" \
  --excluded-accounts "111122223333"  # Loại trừ account cụ thể nếu cần

# Kiểm tra trạng thái trên tất cả accounts
aws configservice describe-organization-conformance-pack-statuses \
  --organization-conformance-pack-names "org-security-baseline"
```

### Ví Dụ Kết Quả Organization-wide Deploy

```json
{
  "OrganizationConformancePackStatuses": [
    {
      "OrganizationConformancePackName": "org-security-baseline",
      "Status": "UPDATE_SUCCESSFUL",
      "OrganizationConformancePackArn": "arn:aws:config:...",
      "LastUpdateTime": "2026-05-17T10:00:00Z"
    }
  ]
}
```

---

## Scoring & Reporting — Báo Cáo Tuân Thủ

### Compliance Score

AWS Config tự động tính **Compliance Score** (Điểm Tuân Thủ) cho mỗi conformance pack:

```
Compliance Score = (Số Rules COMPLIANT) / (Tổng số Rules có đủ dữ liệu) × 100%

Ví dụ:
  Tổng: 30 rules
  COMPLIANT: 24 rules
  NON_COMPLIANT: 6 rules
  Score = 24/30 × 100% = 80%
```

### Xem Compliance Score Qua CLI

```bash
# Xem tổng quan compliance của pack
aws configservice get-conformance-pack-compliance-summary \
  --conformance-pack-names "company-security-baseline"

# Xem chi tiết từng rule trong pack
aws configservice get-conformance-pack-compliance-details \
  --conformance-pack-name "company-security-baseline" \
  --filters '{
    "ConfigRuleNames": ["s3-bucket-public-read-prohibited"],
    "ComplianceType": "NON_COMPLIANT"
  }'
```

### Xuất Báo Cáo Cho Auditor

```bash
# Yêu cầu tạo báo cáo
aws configservice start-config-rules-evaluation \
  --config-rule-names s3-bucket-public-read-prohibited encrypted-volumes

# Export kết quả ra S3 để chia sẻ với auditor
aws configservice deliver-config-snapshot \
  --delivery-channel-name default
```

### Tích Hợp Với AWS Audit Manager

AWS Audit Manager tự động thu thập bằng chứng (evidence) từ Config conformance packs để phục vụ kiểm toán:

```
Config Conformance Pack
       ↓ (findings)
AWS Audit Manager
       ↓ (evidence collection)
Assessment Report
       ↓
Auditor nhận báo cáo PDF đầy đủ
```

---

## Thực Hành Tốt Nhất

### 1. Chọn Đúng Framework Ngay Từ Đầu

```
Câu hỏi để xác định framework cần dùng:
  • Có xử lý thẻ tín dụng? → PCI-DSS
  • Có dữ liệu y tế? → HIPAA
  • Hệ thống chính phủ Mỹ? → NIST 800-53 / FedRAMP
  • Cần baseline bảo mật chung? → CIS Benchmark Level 1
  • Muốn tổng hợp? → AWS Foundational Security Best Practices
```

### 2. Bắt Đầu Với Sample Pack, Sau Đó Tùy Chỉnh

```
Bước 1: Deploy AWS sample pack → Xem compliance score
Bước 2: Phân tích NON_COMPLIANT resources → Hiểu gaps
Bước 3: Tùy chỉnh parameters (VD: password length, key rotation days)
Bước 4: Thêm/bớt rules phù hợp với môi trường
Bước 5: Document lý do exclude rules nào (cho auditor)
```

### 3. Lưu Template Trên Git

```
configs/
  conformance-packs/
    company-baseline-v1.2.yaml       ← Template hiện tại
    CHANGELOG.md                     ← Lịch sử thay đổi
    parameters/
      production.json                ← Tham số cho production
      staging.json                   ← Tham số cho staging
```

### 4. Monitor Compliance Score Theo Thời Gian

Tạo CloudWatch Dashboard theo dõi compliance score:

```bash
# Tạo CloudWatch metric từ Config conformance pack score
# (Dùng EventBridge + Lambda để push custom metric)
aws cloudwatch put-metric-data \
  --namespace "ComplianceScore" \
  --metric-name "ConformancePackScore" \
  --dimensions Name=PackName,Value=company-security-baseline \
  --value 80 \
  --unit Percent
```

### 5. Exclude Rules Có Lý Do Rõ Ràng

Không phải rule nào cũng phù hợp với mọi môi trường. Document rõ:

```markdown
# Excluded Rules từ CIS Level 2 Pack

| Rule | Lý Do Exclude | Approved By | Date |
|------|--------------|-------------|------|
| `s3-bucket-versioning-enabled` | Chỉ áp dụng cho critical buckets, không phải tất cả | CTO | 2026-01-15 |
| `guardduty-enabled-centralized` | Đang dùng third-party SIEM thay thế | CISO | 2026-02-01 |
```

---

## Câu Hỏi Phỏng Vấn

**Q: Conformance Pack khác Config Rule như thế nào?**

A: Config Rule là đơn vị đánh giá đơn lẻ cho một điều kiện cụ thể. Conformance Pack là *bộ sưu tập* nhiều Config Rules được đóng gói theo một compliance framework, triển khai và quản lý cùng nhau, với compliance score tổng hợp. Conformance Pack giúp map kết quả đánh giá sang các controls của framework (CIS 1.1, PCI Req 3...) dễ dàng hơn.

**Q: Tại sao nên dùng Conformance Pack thay vì tự tạo từng Config Rule?**

A: Ba lý do chính: (1) **Quản lý vòng đời dễ hơn** — xóa pack là xóa toàn bộ rules, không cần xóa từng cái; (2) **Audit-ready** — có sẵn mapping giữa rules và framework controls, auditor nhận được báo cáo rõ ràng; (3) **Compliance score tự động** — AWS tính % tuân thủ tổng thể của cả framework thay vì phải tính thủ công từ kết quả từng rule.

**Q: Có thể deploy Conformance Pack mà không ảnh hưởng đến Config Rules hiện có không?**

A: Có. Nếu một rule trong Conformance Pack có cùng tên với rule đang tồn tại, Config sẽ **tạo mới rule đó dưới tên khác** (thêm prefix của pack). Rules hiện có không bị ảnh hưởng. Tuy nhiên, để tránh lộn xộn, nên lên kế hoạch naming scheme rõ ràng trước khi deploy.

---

**Tiếp Theo:** [4-remediation.md](4-remediation.md) — Remediation Actions: Tự Động Khắc Phục Với SSM Automation
