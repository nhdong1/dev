# PCI-DSS trên AWS — Tuân Thủ Ngành Thẻ Thanh Toán

> PCI-DSS (Payment Card Industry Data Security Standard — Tiêu Chuẩn Bảo Mật Dữ Liệu Ngành Thẻ Thanh Toán) là tiêu chuẩn bảo mật bắt buộc cho mọi tổ chức xử lý, lưu trữ hoặc truyền dữ liệu thẻ tín dụng. Tài liệu này hướng dẫn kiến trúc và dịch vụ AWS để đáp ứng 12 yêu cầu PCI-DSS v4.0.

---

## 🎯 Hiểu PCI-DSS Trong Bối Cảnh AWS

### Shared Responsibility Model Cho PCI-DSS

```
AWS Chịu Trách Nhiệm (Compliance của hạ tầng):
├── Bảo mật vật lý data center (Req 9)
├── Hypervisor security
├── Network infrastructure cơ bản
└── Báo cáo: AWS PCI-DSS AOC (Attestation of Compliance) — tải từ AWS Artifact

Bạn Chịu Trách Nhiệm (Compliance của ứng dụng và cấu hình):
├── Cấu hình Security Groups, NACLs
├── Mã hóa dữ liệu thẻ
├── Quản lý access control và MFA
├── Giám sát, log, và cảnh báo
└── Quản lý lỗ hổng bảo mật (patching)
```

### Phạm Vi PCI-DSS (Scoping)

```
CDE (Cardholder Data Environment — Môi Trường Dữ Liệu Chủ Thẻ):
├── Systems lưu trữ PAN (Primary Account Number — Số Tài Khoản Chính)
├── Systems xử lý giao dịch thanh toán
├── Systems kết nối trực tiếp với payment processor
└── Systems quản trị CDE (jump boxes, monitoring)

Out-of-Scope Strategies (Chiến Lược Thu Hẹp Phạm Vi):
├── Tokenization (Mã Hóa Token): Thay thế PAN bằng token không có giá trị
│   └── AWS: Thanh toán qua Stripe/Braintree → chỉ lưu token
├── Network Segmentation: Cô lập CDE trong VPC riêng biệt
│   └── AWS: Dedicated VPC + Transit Gateway với strict routing
└── P2PE (Point-to-Point Encryption — Mã Hóa Đầu Đến Đầu):
    └── Mã hóa tại terminal → chỉ giải mã tại payment processor
```

---

## 🗺️ Kiến Trúc PCI-DSS Trên AWS

```
Internet
    │
    ▼
┌─────────────┐
│  CloudFront │ ← WAF (OWASP Top 10, Custom Rules)
│  (CDN)      │   Shield Advanced (DDoS protection)
└──────┬──────┘
       │ HTTPS only (TLS 1.2+)
       ▼
┌─────────────────────────────────────────────────────────────┐
│                    Public Subnet (DMZ)                       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  ALB (Application Load Balancer)                     │   │
│  │  Security Group: Port 443 from 0.0.0.0/0 only       │   │
│  └──────────────────────────┬───────────────────────────┘   │
└─────────────────────────────┼───────────────────────────────┘
                              │ Encrypted (TLS)
┌─────────────────────────────┼───────────────────────────────┐
│                    Private Subnet (Application Layer)        │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Payment Service (EC2 / ECS / Lambda)               │     │
│  │  ├── Không lưu PAN (dùng Stripe/Braintree token)   │     │
│  │  ├── Chỉ kết nối đến payment processor qua VPC EP  │     │
│  │  └── IMDSv2 only (Instance Metadata Service v2)     │     │
│  └────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────┼───────────────────────────────┐
│                    Private Subnet (Data Layer)                │
│  ┌─────────────────┐   ┌────────────────────────────────┐   │
│  │  RDS (Aurora)   │   │  Secrets Manager               │   │
│  │  Encrypted:KMS  │   │  (DB credentials, API keys)    │   │
│  │  No public IP   │   │  Auto-rotate: 90 days          │   │
│  │  SSL required   │   └────────────────────────────────┘   │
│  └─────────────────┘                                        │
└─────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────────────┐
                    │  S3 (Audit Logs)│
                    │  KMS encrypted  │
                    │  MFA delete ON  │
                    │  Versioning ON  │
                    └─────────────────┘
```

---

## 📋 12 Yêu Cầu PCI-DSS và Triển Khai AWS

### Requirement 1–2: Network Security Controls (Kiểm Soát Bảo Mật Mạng)

```
Yêu Cầu: Kiểm soát lưu lượng vào/ra CDE, cấu hình hệ thống an toàn

AWS Implementation:
```

```bash
# Security Group cho Payment Service — chỉ nhận traffic từ ALB
aws ec2 create-security-group \
  --group-name "payment-service-sg" \
  --description "Payment service — chỉ từ ALB" \
  --vpc-id "vpc-pci-xxx"

# Chỉ cho phép từ ALB security group, port 8443
aws ec2 authorize-security-group-ingress \
  --group-id "sg-payment-service" \
  --protocol tcp \
  --port 8443 \
  --source-group "sg-alb-id"

# NACL cho subnet payment — deny tất cả, cho phép explicit
aws ec2 create-network-acl-entry \
  --network-acl-id "acl-pci-xxx" \
  --rule-number 100 \
  --protocol tcp \
  --rule-action allow \
  --ingress \
  --cidr-block "10.0.1.0/24" \
  --port-range From=8443,To=8443

# Config rule kiểm tra: không có SG allow 0.0.0.0/0 inbound
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "pci-no-unrestricted-incoming-traffic",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "INCOMING_SSH_DISABLED"
    }
  }'
```

### Requirement 3–4: Protect Stored & Transmitted Data (Bảo Vệ Dữ Liệu)

```
Yêu Cầu 3: Không lưu SAD (Sensitive Authentication Data — Dữ Liệu Xác Thực Nhạy Cảm)
Yêu Cầu 4: Mã hóa PAN khi truyền qua mạng không tin cậy
```

```python
# Thiết kế đúng — KHÔNG lưu PAN vào database
class PaymentProcessor:
    def __init__(self):
        self.stripe = stripe.StripeClient(api_key=self._get_api_key())
    
    def _get_api_key(self) -> str:
        """Lấy Stripe API key từ Secrets Manager"""
        sm = boto3.client('secretsmanager')
        secret = sm.get_secret_value(SecretId='payment/stripe-api-key')
        return json.loads(secret['SecretString'])['api_key']
    
    def process_payment(self, amount: int, stripe_token: str) -> dict:
        """
        Chỉ nhận Stripe token — KHÔNG BAO GIỜ nhận PAN trực tiếp.
        Stripe xử lý mã hóa và tuân thủ PCI-DSS phía họ.
        """
        charge = self.stripe.charges.create(
            amount=amount,          # cents
            currency="usd",
            source=stripe_token,    # token từ Stripe.js, không phải PAN thật
            description="Order payment"
        )
        
        # Lưu vào DB: CHỈ lưu charge_id, không lưu PAN hay CVV
        return {
            "charge_id": charge.id,          # ch_1234abcd
            "last4": charge.payment_method_details.card.last4,  # "4242" (ok, bị mask)
            "brand": charge.payment_method_details.card.brand,  # "visa"
            "status": charge.status
        }

# Cấu hình RDS để yêu cầu SSL
# rds.force_ssl = 1 trong Parameter Group
# Kết nối phải có ssl_ca certificate
```

```bash
# S3 bucket cho audit logs — enforce HTTPS và mã hóa
aws s3api put-bucket-policy \
  --bucket pci-audit-logs \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "DenyHTTP",
        "Effect": "Deny",
        "Principal": "*",
        "Action": "s3:*",
        "Resource": [
          "arn:aws:s3:::pci-audit-logs",
          "arn:aws:s3:::pci-audit-logs/*"
        ],
        "Condition": {
          "Bool": {"aws:SecureTransport": "false"}
        }
      }
    ]
  }'

# Bật default encryption với KMS
aws s3api put-bucket-encryption \
  --bucket pci-audit-logs \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/pci-kms-key"
      },
      "BucketKeyEnabled": true
    }]
  }'
```

### Requirement 5–6: Vulnerability Management (Quản Lý Lỗ Hổng Bảo Mật)

```
Yêu Cầu 5: Bảo vệ khỏi malicious software
Yêu Cầu 6: Phát triển và bảo trì hệ thống an toàn (patching, secure coding)
```

```bash
# Inspector v2 — quét lỗ hổng tự động
aws inspector2 enable \
  --resource-types EC2 ECR LAMBDA

# Kiểm tra findings PCI-critical (CVSS >= 7.0)
aws inspector2 list-findings \
  --filter-criteria '{
    "severities": [{"comparison": "EQUALS", "value": "CRITICAL"},
                   {"comparison": "EQUALS", "value": "HIGH"}],
    "resourceTags": [{
      "comparison": "EQUALS",
      "key": "Environment",
      "value": "production-pci"
    }]
  }' \
  --query 'findings[*].{Title:title, Severity:severity, Resource:resources[0].id}'

# SSM Patch Manager — tự động patch
aws ssm create-patch-baseline \
  --name "PCI-Patch-Baseline-AmazonLinux2" \
  --operating-system AMAZON_LINUX_2 \
  --approval-rules '{
    "PatchRules": [{
      "PatchFilterGroup": {
        "PatchFilters": [
          {"Key": "CLASSIFICATION", "Values": ["Security"]},
          {"Key": "SEVERITY", "Values": ["Critical", "Important"]}
        ]
      },
      "ApproveAfterDays": 7,
      "ComplianceLevel": "CRITICAL"
    }]
  }'

# Maintenance window — patch hàng tuần
aws ssm create-maintenance-window \
  --name "PCI-Weekly-Patching" \
  --schedule "cron(0 2 ? * SUN *)" \
  --duration 4 \
  --cutoff 1 \
  --allow-unassociated-targets false
```

### Requirement 7–8: Access Control (Kiểm Soát Truy Cập)

```
Yêu Cầu 7: Restrict access to system components and cardholder data by business need to know
Yêu Cầu 8: Identify users and authenticate access
```

```json
// IAM Policy cho Payment Service — Least Privilege
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SecretsManagerReadOnly",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:s3:::payment/stripe-api-key",
        "arn:aws:s3:::payment/db-credentials"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    },
    {
      "Sid": "KMSDecryptOnly",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/payment-kms-key",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "secretsmanager.us-east-1.amazonaws.com"
        }
      }
    },
    {
      "Sid": "DenyNonProductionAccess",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-west-2"]
        }
      }
    }
  ]
}
```

```bash
# Bắt buộc MFA cho tất cả IAM users truy cập CDE
aws iam create-policy \
  --policy-name "DenyWithoutMFA" \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "DenyAllExceptMFASetup",
        "Effect": "Deny",
        "NotAction": [
          "iam:CreateVirtualMFADevice",
          "iam:EnableMFADevice",
          "iam:GetUser",
          "iam:ListMFADevices",
          "iam:ListVirtualMFADevices",
          "iam:ResyncMFADevice",
          "sts:GetSessionToken"
        ],
        "Resource": "*",
        "Condition": {
          "BoolIfExists": {
            "aws:MultiFactorAuthPresent": "false"
          }
        }
      }
    ]
  }'

# IAM Password Policy cho PCI-DSS
aws iam update-account-password-policy \
  --minimum-password-length 14 \
  --require-symbols \
  --require-numbers \
  --require-uppercase-characters \
  --require-lowercase-characters \
  --allow-users-to-change-password \
  --max-password-age 90 \
  --password-reuse-prevention 12 \
  --hard-expiry false
```

### Requirement 10: Log & Monitor All Access (Ghi Log và Giám Sát)

```
Yêu Cầu 10: Ghi log mọi truy cập vào hệ thống và dữ liệu chủ thẻ
```

```bash
# CloudTrail — ghi tất cả API calls
aws cloudtrail create-trail \
  --name "pci-audit-trail" \
  --s3-bucket-name "pci-cloudtrail-logs" \
  --is-multi-region-trail \
  --enable-log-file-validation \
  --kms-key-id "arn:aws:kms:us-east-1:123456789012:key/pci-kms-key" \
  --include-global-service-events \
  --is-organization-trail

# Bật data events — ghi access vào S3 và Lambda
aws cloudtrail put-event-selectors \
  --trail-name "pci-audit-trail" \
  --event-selectors '[
    {
      "ReadWriteType": "All",
      "IncludeManagementEvents": true,
      "DataResources": [
        {
          "Type": "AWS::S3::Object",
          "Values": ["arn:aws:s3:::pci-data-bucket/"]
        },
        {
          "Type": "AWS::Lambda::Function",
          "Values": ["arn:aws:lambda"]
        }
      ]
    }
  ]'

# Bật MFA Delete cho S3 audit log bucket — logs không thể xóa
aws s3api put-bucket-versioning \
  --bucket pci-audit-logs \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::123456789012:mfa/admin-mfa-device 123456"

# CloudWatch Alarm cho failed logins và privileged access
aws cloudwatch put-metric-alarm \
  --alarm-name "PCI-RootAccountLogin" \
  --alarm-description "Cảnh báo khi root account đăng nhập — vi phạm PCI-DSS" \
  --metric-name "RootAccountUsage" \
  --namespace "CloudTrailMetrics" \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:pci-security-alerts"
```

### Requirement 11: Security Testing (Kiểm Thử Bảo Mật)

```
Yêu Cầu 11: Kiểm thử bảo mật định kỳ — penetration test, vulnerability scan
```

```bash
# Inspector scan schedule tự động
aws inspector2 create-filter \
  --name "PCI-Critical-Findings" \
  --action SUPPRESS \
  --filter-criteria '{
    "resourceTags": [{
      "comparison": "NOT_EQUALS",
      "key": "PCIScope",
      "value": "in-scope"
    }]
  }'

# AWS Config rule kiểm tra network configuration
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "pci-vpc-flow-logs-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "VPC_FLOW_LOGS_ENABLED"
    }
  }'

# Penetration testing:
# AWS cho phép test các services sau mà không cần xin phép:
# EC2, RDS, CloudFront, API Gateway, Lambda, LightSail, EBS, Elastic Beanstalk
# Xem: https://aws.amazon.com/security/penetration-testing/
```

---

## 🔑 Key Controls Checklist

```
PCI-DSS AWS Architecture Checklist:

Network:
☐ CDE trong dedicated VPC, cô lập khỏi các VPC khác
☐ Không có Security Group allow 0.0.0.0/0 inbound (trừ 443 cho public-facing)
☐ WAF bắt buộc cho tất cả web applications
☐ VPC Flow Logs bật và lưu >= 12 tháng
☐ NACLs deny-all, cho phép explicit

Mã Hóa:
☐ Tất cả data at rest: mã hóa AES-256 với KMS
☐ Tất cả data in transit: TLS 1.2+ minimum
☐ S3 buckets: SSL-only bucket policy
☐ RDS: force_ssl = 1, mã hóa storage
☐ EBS volumes: mã hóa bằng KMS CMK

Truy Cập:
☐ MFA bắt buộc cho tất cả console access
☐ Không sử dụng root account hàng ngày
☐ Không có long-term access keys (dùng IAM roles)
☐ Password policy: 14+ ký tự, đổi 90 ngày
☐ Least privilege cho tất cả IAM roles

Giám Sát:
☐ CloudTrail bật, multi-region, toàn organization
☐ Log file validation bật
☐ CloudTrail logs mã hóa KMS
☐ MFA delete cho S3 audit log bucket
☐ CloudWatch alarms cho suspicious activity
☐ GuardDuty bật và findings được xử lý

Dữ Liệu:
☐ Không lưu PAN (dùng tokenization)
☐ Không lưu CVV/CVC (không được phép theo PCI-DSS)
☐ Nếu lưu PAN: mã hóa đúng format PCI-DSS
☐ Data retention policy: xóa dữ liệu thẻ sau khi không cần
```

---

## 📊 AWS Artifact — Báo Cáo Tuân Thủ PCI-DSS Của AWS

```bash
# Tải PCI-DSS AOC (Attestation of Compliance) của AWS
# 1. Vào AWS Console → AWS Artifact
# 2. Reports → PCI-DSS Attestation of Compliance
# 3. Download → Đây là bằng chứng AWS infrastructure đã tuân thủ

# Hoặc qua CLI (cần permissions)
aws artifact get-report \
  --report-id "report-pci-dss-aoc-2024" \
  --report-version 1 \
  --terminate-on-no-input
```

### Sử Dụng AWS Artifact Hiệu Quả Trong Audit

```
Khi Auditor Yêu Cầu Bằng Chứng Hạ Tầng:
1. Tải PCI-DSS AOC của AWS từ Artifact
2. Giải thích Shared Responsibility Model
3. Audit Manager → xuất evidence của bạn
4. Chỉ ra mapping: bạn responsible cho những gì, AWS cho phần còn lại

Documents Cần Chuẩn Bị:
├── AWS PCI-DSS AOC (tải từ Artifact)
├── System diagram showing CDE boundary
├── Data flow diagram (luồng dữ liệu thẻ)
├── Network segmentation documentation
├── Access control matrix (ma trận quyền truy cập)
└── Incident response plan cho payment systems
```

---

## 💡 Câu Hỏi Phỏng Vấn

**Q: Làm thế nào để giảm phạm vi PCI-DSS (reduce scope) khi dùng AWS?**
> Ba cách chính: (1) **Tokenization** — dùng Stripe/Braintree để xử lý thẻ, chỉ lưu token trong hệ thống của bạn, PAN không bao giờ chạm server bạn; (2) **Network segmentation** — đặt bất kỳ component nào xử lý thẻ trong CDE VPC riêng biệt, cô lập hoàn toàn; (3) **P2PE** — dùng thiết bị terminal P2PE certified, PAN được mã hóa tại điểm quẹt thẻ và chỉ giải mã tại payment processor.

**Q: CVV/CVC có thể lưu vào database không?**
> Tuyệt đối không. PCI-DSS Req 3.2.1 cấm lưu trữ SAD (Sensitive Authentication Data) sau khi authorization, bao gồm CVV/CVC/CAV2, full magnetic stripe data, và PIN. Vi phạm điều này là vi phạm nghiêm trọng nhất của PCI-DSS và là lý do của nhiều breach lớn. AWS không có dịch vụ nào hỗ trợ lưu CVV vì mục đích này — và không nên có.

**Q: QSA (Qualified Security Assessor) cần những gì để audit hệ thống trên AWS?**
> QSA cần: (1) AWS PCI-DSS AOC từ Artifact (bằng chứng hạ tầng AWS tuân thủ); (2) System diagram và data flow diagram của bạn; (3) Evidence từ Audit Manager (Config rules, CloudTrail logs, IAM policies); (4) Penetration test report (phải làm ít nhất mỗi năm 1 lần và sau major change); (5) Vulnerability scan reports từ Inspector; (6) Access control documentation và bằng chứng MFA; (7) Incident response plan.

---

## 🔗 Điều Hướng

| Trước | Hiện Tại | Sau |
|---|---|---|
| [3-firewall-manager.md](3-firewall-manager.md) | **4-pci-dss-aws.md** | [5-hipaa-aws.md](5-hipaa-aws.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
