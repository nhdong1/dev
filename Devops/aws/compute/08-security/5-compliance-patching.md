# Compliance & Patching — Tuân Thủ và Vá Lỗi AWS Compute

> Compliance (Tuân Thủ) là khả năng chứng minh rằng hệ thống đáp ứng các yêu cầu bảo mật của regulatory frameworks (khung pháp lý) như PCI DSS, HIPAA, SOC 2. AWS cung cấp Amazon Inspector (Thanh Tra Tự Động), AWS Security Hub (Trung Tâm Bảo Mật), và AWS Config để tự động hóa compliance monitoring (giám sát tuân thủ).

---

## 📚 Mục Lục

1. [Compliance Frameworks Trên AWS](#compliance-frameworks-trên-aws)
2. [Amazon Inspector — Quét Lỗ Hổng Tự Động](#amazon-inspector--quét-lỗ-hổng-tự-động)
3. [AWS Security Hub — Trung Tâm Bảo Mật](#aws-security-hub--trung-tâm-bảo-mật)
4. [AWS Config — Theo Dõi Cấu Hình](#aws-config--theo-dõi-cấu-hình)
5. [Patch Management Toàn Diện](#patch-management-toàn-diện)
6. [GuardDuty — Phát Hiện Mối Đe Dọa](#guardduty--phát-hiện-mối-đe-dọa)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Compliance Frameworks Trên AWS

### Các Standards Phổ Biến

```
PCI DSS (Payment Card Industry Data Security Standard
         — Tiêu Chuẩn Bảo Mật Dữ Liệu Ngành Thẻ Thanh Toán):
├── Yêu cầu: mã hóa dữ liệu thẻ, network segmentation, access control
├── Patch trong 30 ngày (critical) / 3 tháng (non-critical)
└── AWS Services: WAF, GuardDuty, Inspector, CloudTrail, KMS

HIPAA (Health Insurance Portability and Accountability Act
        — Luật Về Tính Di Động và Trách Nhiệm Giải Trình Bảo Hiểm Y Tế):
├── Bảo vệ PHI (Protected Health Information — Thông Tin Sức Khỏe Được Bảo Vệ)
├── Encryption at rest and in transit bắt buộc
└── BAA (Business Associate Agreement) với AWS cần ký

SOC 2 (System and Organization Controls 2 — Kiểm Soát Tổ Chức và Hệ Thống):
├── Dựa trên 5 Trust Service Criteria
├── Audit annually (kiểm toán hàng năm)
└── CloudTrail, Config, Security Hub giúp evidence collection

ISO 27001:
├── Information Security Management System (Hệ Thống Quản Lý Bảo Mật Thông Tin)
└── AWS nhiều regions đã có ISO 27001 certification
```

### AWS Compliance Programs

```bash
# Xem AWS compliance reports (FedRAMP, HIPAA, PCI, SOC)
# Qua AWS Artifact — miễn phí
aws artifact list-reports-by-prefix \
  --report-name-prefix "SOC"

# Download báo cáo compliance
aws artifact get-report \
  --report-name "AWS-SOC3-Report-2024"
```

### Shared Responsibility Matrix (Ma Trận Trách Nhiệm)

```
Compliance Area          AWS Responsibility    Customer Responsibility
──────────────────────   ─────────────────    ──────────────────────
Physical security        ✅ AWS               ❌
Hypervisor patching      ✅ AWS               ❌
Managed service patches  ✅ AWS (RDS, Lambda) ❌
OS patching (EC2)        ❌                   ✅ Customer
Application patching     ❌                   ✅ Customer
IAM configuration        ❌                   ✅ Customer
Data encryption          ❌ (optional feature) ✅ Customer (must enable)
Network firewall rules   ❌                   ✅ Customer
Audit logging enable     ❌                   ✅ Customer
Compliance reporting     ❌ (tools provided)  ✅ Customer
```

---

## Amazon Inspector — Quét Lỗ Hổng Tự Động

### Tổng Quan Inspector v2

```
Amazon Inspector (Thanh Tra Tự Động):
└── Tự động quét lỗ hổng bảo mật (vulnerabilities) trong:
    ├── EC2 instances — OS vulnerabilities, network reachability
    ├── Lambda functions — code vulnerabilities, package CVEs
    └── Container images trong ECR — layer-by-layer scanning

CVE = Common Vulnerabilities and Exposures — Lỗ Hổng và Rủi Ro Phổ Biến
    → Database lỗ hổng được biết đến công khai
    → Inspector so sánh software trong môi trường với CVE database
```

### Inspector Hoạt Động Thế Nào

```
Luồng Hoạt Động (với EC2):

1. Inspector agent hoặc SSM Agent thu thập:
   - Danh sách packages đã cài (rpm, dpkg, npm...)
   - Network configuration (ports mở, security groups)
   - OS version và kernel version

2. Inspector so sánh với CVE database (cập nhật liên tục)

3. Tính CVSS score (0-10):
   - CRITICAL (9.0-10.0): Phải vá ngay, thường remote exploitable
   - HIGH (7.0-8.9): Vá trong 30 ngày
   - MEDIUM (4.0-6.9): Vá trong 90 ngày
   - LOW (0.1-3.9): Theo dõi, vá theo schedule

4. Gửi findings đến:
   - Inspector console
   - Security Hub (tổng hợp)
   - EventBridge (tự động hóa response)
```

### Bật Và Cấu Hình Inspector

```bash
# Bật Inspector cho account (bắt buộc bật trong từng region)
aws inspector2 enable \
  --resource-types EC2 LAMBDA ECR

# Xem findings
aws inspector2 list-findings \
  --filter-criteria '{
    "severity": [{"comparison": "EQUALS", "value": "CRITICAL"}]
  }'

# Tìm findings cho specific instance
aws inspector2 list-findings \
  --filter-criteria '{
    "resourceId": [{
      "comparison": "EQUALS",
      "value": "i-1234567890abcdef0"
    }]
  }'

# Xem tổng hợp findings theo account
aws inspector2 list-account-permissions

# Suppress (tạm ẩn) false positive
aws inspector2 create-filter \
  --name "SuppressDevInstances" \
  --action SUPPRESS \
  --filter-criteria '{
    "ec2InstanceTags": [{
      "comparison": "EQUALS",
      "key": "Environment",
      "value": "development"
    }]
  }'
```

### Inspector Cho Lambda

```bash
# Inspector quét Lambda function packages
# Phát hiện: outdated dependencies, known CVEs trong npm/pip packages

# Ví dụ finding:
# {
#   "findingArn": "arn:aws:inspector2:...",
#   "type": "PACKAGE_VULNERABILITY",
#   "severity": "CRITICAL",
#   "title": "CVE-2021-44228 - Log4Shell in log4j-core",
#   "description": "Remote code execution vulnerability",
#   "remediation": {
#     "recommendation": {
#       "text": "Update log4j-core to version 2.17.1 or later"
#     }
#   }
# }
```

### Tự Động Hóa Response Với EventBridge

```json
// EventBridge rule để auto-create ticket khi có CRITICAL finding
{
  "source": ["aws.inspector2"],
  "detail-type": ["Inspector2 Finding"],
  "detail": {
    "severity": ["CRITICAL"],
    "status": ["ACTIVE"]
  }
}

// Target: Lambda function gửi notification và tạo Jira ticket
// Hoặc SNS topic để notify team
```

---

## AWS Security Hub — Trung Tâm Bảo Mật

### Security Hub Là Gì?

```
AWS Security Hub (Trung Tâm Bảo Mật):
└── Tổng hợp, ưu tiên, và hiển thị security findings từ nhiều nguồn:
    ├── Amazon Inspector (vulnerability findings)
    ├── Amazon GuardDuty (threat detection findings)
    ├── Amazon Macie (sensitive data findings)
    ├── AWS Config (compliance findings)
    ├── IAM Access Analyzer (access findings)
    └── Third-party integrations (Splunk, CrowdStrike...)

→ Một nơi duy nhất để xem toàn bộ security posture
```

### Security Standards Tích Hợp

```
Security Hub tự động check theo các standards:

1. AWS Foundational Security Best Practices (FSBP):
   → 200+ automated checks
   → ví dụ: "EC2 instances should not have a public IP"

2. CIS AWS Foundations Benchmark:
   → Center for Internet Security recommendations
   → ví dụ: "Ensure MFA is enabled for root account"

3. PCI DSS v3.2.1:
   → Payment card industry requirements
   → ví dụ: "Restrict inbound traffic to Cardholder Data Environment"

4. NIST SP 800-53 Rev 5:
   → National Institute of Standards and Technology
```

### Bật Và Dùng Security Hub

```bash
# Bật Security Hub
aws securityhub enable-security-hub \
  --enable-default-standards

# Xem compliance status cho AWS Foundational Security Best Practices
aws securityhub describe-standards-controls \
  --standards-subscription-arn "arn:aws:securityhub:ap-southeast-1:123456789012:subscription/aws-foundational-security-best-practices/v/1.0.0"

# Xem findings
aws securityhub get-findings \
  --filters '{
    "SeverityLabel": [{"Value": "CRITICAL", "Comparison": "EQUALS"}],
    "RecordState": [{"Value": "ACTIVE", "Comparison": "EQUALS"}],
    "WorkflowStatus": [{"Value": "NEW", "Comparison": "EQUALS"}]
  }'

# Update finding status (sau khi remediate)
aws securityhub batch-update-findings \
  --finding-identifiers '[{
    "Id": "arn:aws:securityhub:...",
    "ProductArn": "arn:aws:securityhub:..."
  }]' \
  --workflow '{"Status": "RESOLVED"}' \
  --note '{"Text": "Fixed by enabling EBS encryption", "UpdatedBy": "ops-team"}'
```

### Security Hub Trong Multi-Account Environment

```bash
# Security Hub hỗ trợ aggregation từ nhiều accounts
# (Administrator account tổng hợp findings từ member accounts)

# Trong administrator account: invite member accounts
aws securityhub create-members \
  --account-details '[
    {"AccountId": "111111111111", "Email": "team-a@company.com"},
    {"AccountId": "222222222222", "Email": "team-b@company.com"}
  ]'

# Trong member account: accept invitation
aws securityhub accept-administrator-invitation \
  --administrator-id 000000000000 \
  --invitation-id invite-id-here
```

---

## AWS Config — Theo Dõi Cấu Hình

### Config Là Gì?

```
AWS Config (Cấu Hình AWS):
└── Ghi lại và theo dõi các thay đổi trong AWS resource configurations

Hai tính năng chính:
1. Configuration History (Lịch Sử Cấu Hình):
   → "EC2 instance này đã thay đổi Security Group lúc nào và thành gì?"
   → Giúp debug, audit, và compliance

2. Config Rules (Quy Tắc Cấu Hình):
   → Kiểm tra liên tục xem resources có comply với rules không
   → ví dụ: "Mọi EC2 instance phải có tags Environment"
   → ví dụ: "Mọi S3 bucket phải có logging enabled"
```

### Config Rules Quan Trọng Cho Security

```bash
# Bật AWS Config
aws configservice start-configuration-recorder \
  --configuration-recorder-name default

# Tạo delivery channel (để lưu config history vào S3)
aws configservice put-delivery-channel \
  --delivery-channel '{
    "name": "default",
    "s3BucketName": "my-config-bucket",
    "configSnapshotDeliveryProperties": {
      "deliveryFrequency": "TwentyFour_Hours"
    }
  }'

# Rules bảo mật quan trọng:
SECURITY_RULES=(
  "encrypted-volumes"                           # EBS phải encrypt
  "rds-storage-encrypted"                       # RDS phải encrypt
  "s3-bucket-server-side-encryption-enabled"   # S3 phải encrypt
  "iam-root-access-key-check"                  # Root không được có access key
  "mfa-enabled-for-iam-console-access"         # IAM users phải có MFA
  "restricted-ssh"                              # Port 22 không mở 0.0.0.0/0
  "restricted-common-ports"                     # Ports nguy hiểm phải restricted
  "vpc-flow-logs-enabled"                       # VPC Flow Logs phải bật
  "cloudtrail-enabled"                          # CloudTrail phải bật
  "cloud-trail-encryption-enabled"              # CloudTrail phải encrypt
  "guardduty-enabled-centralized"               # GuardDuty phải bật
  "securityhub-enabled"                         # Security Hub phải bật
)

for RULE in "${SECURITY_RULES[@]}"; do
  aws configservice put-config-rule \
    --config-rule "{
      \"ConfigRuleName\": \"$RULE\",
      \"Source\": {
        \"Owner\": \"AWS\",
        \"SourceIdentifier\": \"$(echo $RULE | tr '[:lower:]' '[:upper:]' | tr '-' '_')\"
      }
    }"
done
```

### Config Conformance Packs — Gói Tuân Thủ

```bash
# Conformance Pack = tập hợp Config Rules theo một standard

# Deploy CIS Benchmark pack
aws configservice put-conformance-pack \
  --conformance-pack-name "CIS-AWS-Foundations-Benchmark" \
  --template-s3-uri "s3://aws-configservice-us-east-1/conformance-packs/Security/CIS-AWS-Foundations-Benchmark.yaml" \
  --delivery-s3-bucket "my-conformance-pack-bucket"

# Xem compliance status
aws configservice describe-conformance-pack-compliance \
  --conformance-pack-name "CIS-AWS-Foundations-Benchmark"

# Xem compliance overview
aws configservice get-conformance-pack-compliance-summary \
  --conformance-pack-names "CIS-AWS-Foundations-Benchmark"
```

### Config Aggregator — Tổng Hợp Đa Account

```bash
# Tổng hợp compliance data từ nhiều accounts/regions
aws configservice put-configuration-aggregator \
  --configuration-aggregator-name "OrgAggregator" \
  --organization-aggregation-source '{
    "RoleArn": "arn:aws:iam::123456789012:role/ConfigAggregatorRole",
    "AllAwsRegions": true
  }'
```

---

## Patch Management Toàn Diện

### Patch Strategy Theo Môi Trường

```
Development:
├── Patch ngay khi có update
├── Auto-approve tất cả patches
└── Restart services ngay, không cần maintenance window

Staging:
├── Patch sau 7 ngày (để verify ở dev trước)
├── Auto-approve Critical và Important security patches
└── Test application sau khi patch

Production:
├── Patch sau 14 ngày (ổn định từ dev và staging)
├── Chỉ auto-approve Critical security patches
├── Maintenance window: 2:00 AM - 6:00 AM Chủ Nhật
└── Rolling update: patch từng AZ (Availability Zone) một
    → Không patch tất cả instances cùng lúc
```

### Patch Testing Pipeline — Quy Trình Kiểm Tra Vá Lỗi

```
┌────────────────────────────────────────────────────────────┐
│                   Patch Pipeline                            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Tuesday:  AWS releases patches                             │
│      ↓                                                     │
│  Wednesday: Auto-patch DEV instances                        │
│      ↓                                                     │
│  Thursday: Run integration tests → verify DEV working       │
│      ↓                                                     │
│  Friday:   Auto-patch STAGING instances                     │
│      ↓                                                     │
│  Weekend:  Monitor STAGING, load test                       │
│      ↓                                                     │
│  Sunday 2AM: Auto-patch PROD instances (rolling)            │
│      ↓                                                     │
│  Monday:   Verify PROD health metrics                       │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### AWS-RunPatchBaseline Document

```bash
# Patch instance ngay lập tức (scan only — không install)
aws ssm send-command \
  --document-name "AWS-RunPatchBaseline" \
  --targets "Key=instanceIds,Values=i-1234567890abcdef0" \
  --parameters '{"Operation": ["Scan"]}'

# Install patches
aws ssm send-command \
  --document-name "AWS-RunPatchBaseline" \
  --targets "Key=tag:PatchGroup,Values=production" \
  --parameters '{
    "Operation": ["Install"],
    "RebootOption": ["RebootIfNeeded"]
  }' \
  --max-concurrency "1"   # Patch từng instance một trong production!
  --max-errors "0"        # Dừng ngay nếu có 1 failure

# Xem patch compliance sau khi chạy
aws ssm list-compliance-items \
  --resource-types "ManagedInstance" \
  --filters "Key=ComplianceType,Type=EQUAL,Values=Patch" \
            "Key=PatchGroup,Type=EQUAL,Values=production"
```

### Patch Compliance Dashboard

```bash
# Tổng hợp patch compliance status
aws ssm list-compliance-summaries \
  --filters "Key=ComplianceType,Type=EQUAL,Values=Patch"

# Output mẫu:
# {
#   "ComplianceType": "Patch",
#   "CompliantSummary": {"CompliantCount": 45},
#   "NonCompliantSummary": {
#     "NonCompliantCount": 3,
#     "SeveritySummary": {
#       "CriticalCount": 1,
#       "HighCount": 2
#     }
#   }
# }
```

---

## GuardDuty — Phát Hiện Mối Đe Dọa

### GuardDuty Là Gì?

```
Amazon GuardDuty (Vệ Sĩ Amazon):
└── Intelligent threat detection (phát hiện mối đe dọa thông minh)
    sử dụng Machine Learning và threat intelligence feeds

Phân Tích:
├── VPC Flow Logs — traffic patterns bất thường
├── CloudTrail Events — API calls đáng ngờ
├── DNS Logs — domain names ác ý
├── EKS Audit Logs — Kubernetes API anomalies
└── S3 Data Events — data access patterns

Không cần agent, không ảnh hưởng performance
```

### Các Loại Findings Quan Trọng

```
Threat Categories (Danh Mục Mối Đe Dọa):

Backdoor:
└── EC2 instance giao tiếp với C&C (Command and Control) server

CryptoCurrency:
└── EC2 instance mining Bitcoin/Ethereum (cryptojacking)

Impact:
├── S3 data được exfiltrate (rút dữ liệu ra ngoài)
└── EC2 resource được dùng cho DDoS

InitialAccess:
├── Root user đăng nhập (bất thường)
└── Brute force SSH thành công

PrivilegeEscalation:
└── IAM policy bị thay đổi để tăng quyền

Reconnaissance:
└── Port scanning từ EC2 instance (bị compromised)

UnauthorizedAccess:
├── API calls từ Tor exit node
└── Credentials được dùng từ IP lạ
```

```bash
# Bật GuardDuty
aws guardduty create-detector \
  --enable \
  --features '[
    {"Name": "S3_DATA_EVENTS", "Status": "ENABLED"},
    {"Name": "EKS_AUDIT_LOGS", "Status": "ENABLED"},
    {"Name": "MALWARE_PROTECTION", "Status": "ENABLED"}
  ]'

# Xem findings
aws guardduty list-findings \
  --detector-id detector-id-here \
  --finding-criteria '{
    "Criterion": {
      "severity": {
        "Gte": 7   ← High và Critical findings
      }
    }
  }'

aws guardduty get-findings \
  --detector-id detector-id-here \
  --finding-ids finding-id-here

# Archive finding sau khi investigate
aws guardduty archive-findings \
  --detector-id detector-id-here \
  --finding-ids finding-id-here
```

### Tự Động Hóa Response Với GuardDuty + EventBridge

```python
# Lambda function xử lý GuardDuty finding tự động
import boto3
import json

def lambda_handler(event, context):
    finding = event['detail']
    finding_type = finding['type']
    severity = finding['severity']
    instance_id = finding.get('resource', {}).get('instanceDetails', {}).get('instanceId')

    if severity >= 7.0 and instance_id:  # High severity
        if 'CryptoCurrency' in finding_type or 'Backdoor' in finding_type:
            # Isolate compromised instance ngay lập tức
            isolate_instance(instance_id)
            notify_security_team(finding)

def isolate_instance(instance_id):
    ec2 = boto3.client('ec2')

    # Tạo "quarantine" security group không có inbound/outbound
    quarantine_sg = ec2.create_security_group(
        GroupName=f'QUARANTINE-{instance_id}',
        Description=f'Quarantine SG for compromised instance {instance_id}',
        VpcId=get_instance_vpc(instance_id)
    )

    # Gán quarantine SG (remove all other SGs)
    ec2.modify_instance_attribute(
        InstanceId=instance_id,
        Groups=[quarantine_sg['GroupId']]
    )

    # Tạo snapshot trước khi forensic analysis
    ec2.create_snapshot(
        VolumeId=get_root_volume(instance_id),
        Description=f'Forensic snapshot - {instance_id}'
    )

def notify_security_team(finding):
    sns = boto3.client('sns')
    sns.publish(
        TopicArn='arn:aws:sns:...:SecurityAlerts',
        Subject=f"SECURITY ALERT: {finding['type']}",
        Message=json.dumps(finding, indent=2)
    )
```

---

## Security Score và Continuous Improvement

### Đo Lường Security Posture — Trạng Thái Bảo Mật

```bash
# Security Hub Security Score (0-100)
aws securityhub get-findings \
  --filters '{"RecordState": [{"Value": "ACTIVE", "Comparison": "EQUALS"}]}' \
  --query 'length(Findings)'

# Config Compliance percentage
aws configservice get-compliance-summary-by-config-rule \
  --query 'ComplianceSummary'

# Inspector Critical findings count
aws inspector2 list-findings \
  --filter-criteria '{"severity": [{"comparison": "EQUALS", "value": "CRITICAL"}]}' \
  --query 'length(findings)'
```

### Security Metrics Dashboard

```
Metrics Quan Trọng Cần Theo Dõi Hàng Tuần:

1. Patch Compliance Rate (Tỷ Lệ Tuân Thủ Vá Lỗi):
   → Mục tiêu: 100% instances patched trong SLA
   → Alert nếu < 95%

2. Inspector Critical Findings Count:
   → Mục tiêu: 0 findings unresolved > 24 giờ
   → Alert nếu tăng so với tuần trước

3. Security Hub Score:
   → Mục tiêu: > 85%
   → Review weekly, fix top failed controls

4. GuardDuty High/Critical Findings:
   → Mục tiêu: 0 unresolved > 4 giờ
   → Incident response plan cho từng finding type

5. IAM Unused Credentials:
   → Mục tiêu: 0 users với credentials không dùng > 90 ngày
   → Review monthly
```

---

## Incident Response — Phản Ứng Sự Cố

### Runbook Cơ Bản Khi Phát Hiện Compromise

```
Bước 1: Contain (Ngăn Chặn) — 0-15 phút
├── Isolate EC2: remove từ Load Balancer target group
├── Revoke IAM credentials của instance
├── Apply restrictive Security Group (quarantine SG)
└── Block outbound traffic để prevent data exfiltration

Bước 2: Evidence Collection (Thu Thập Bằng Chứng) — 15-60 phút
├── Create EBS snapshot ngay (forensic evidence)
├── Export CloudTrail logs cho 24-48 giờ trước khi incident
├── Export VPC Flow Logs
├── Download memory dump nếu có thể
└── Document timeline từ GuardDuty findings

Bước 3: Analysis (Phân Tích) — 1-24 giờ
├── Phân tích CloudTrail: API calls bất thường
├── Phân tích VPC Flow Logs: network connections bất thường
├── Xem user-data và running processes (qua snapshot)
└── Xác định attack vector (IAM leak, CVE exploitation...)

Bước 4: Remediation (Khắc Phục) — 24-72 giờ
├── Patch lỗ hổng đã bị khai thác
├── Rotate tất cả credentials có thể bị lộ
├── Launch replacement instance từ known-good AMI
└── Review và tăng cường security controls

Bước 5: Lessons Learned (Bài Học Kinh Nghiệm) — 1 tuần sau
├── Post-mortem report
├── Update runbooks và playbooks
└── Implement preventive controls
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Inspector và GuardDuty khác nhau như thế nào?

**Trả lời:** Amazon Inspector (Thanh Tra Tự Động) tập trung vào vulnerability assessment (đánh giá lỗ hổng) — nó scan software trên EC2, Lambda, và container images để tìm known CVEs (lỗ hổng đã biết). Inspector trả lời câu hỏi: "Phần mềm nào trong môi trường của tôi có lỗ hổng bảo mật đã biết?" GuardDuty tập trung vào threat detection (phát hiện mối đe dọa) bằng Machine Learning — nó phân tích VPC Flow Logs, CloudTrail, và DNS logs để phát hiện anomalies (bất thường) như cryptomining, data exfiltration, compromised credentials. GuardDuty trả lời câu hỏi: "Có ai đang tấn công hoặc đã compromise môi trường của tôi không?" Nên dùng cả hai: Inspector để proactive patching, GuardDuty để detect active attacks.

### Q2: AWS Config Rules và Inspector khác nhau như thế nào?

**Trả lời:** AWS Config Rules theo dõi configuration compliance (tuân thủ cấu hình) của AWS resources — ví dụ: "EBS volumes phải được encrypt", "Security Groups không được mở port 22 ra internet", "S3 bucket logging phải bật". Nó trả lời câu hỏi: "Resources của tôi có được cấu hình đúng theo best practices không?" Inspector tập trung vào software vulnerability scanning (quét lỗ hổng phần mềm) bên trong instances — tìm outdated packages, known CVEs. Hai dịch vụ bổ trợ cho nhau và cả hai đều gửi findings đến Security Hub để có central view.

### Q3: Khi GuardDuty phát hiện EC2 instance bị compromise, bạn làm gì?

**Trả lời:** Quy trình incident response theo 4 bước: Đầu tiên, Contain (Ngăn Chặn) trong vòng 15 phút — isolate instance bằng cách gán quarantine Security Group (không cho phép traffic), remove khỏi Load Balancer, revoke IAM role tạm thời. Thứ hai, Evidence Collection (Thu Thập Bằng Chứng) — tạo EBS snapshot ngay để forensic analysis, export CloudTrail và VPC Flow Logs. Thứ ba, Analysis (Phân Tích) — phân tích logs để xác định attack vector, tìm hiểu data gì đã bị access. Cuối cùng, Remediation (Khắc Phục) — patch lỗ hổng, rotate tất cả credentials có thể bị lộ, launch instance mới từ clean AMI, và viết post-mortem report. Quan trọng: KHÔNG tắt instance ngay vì sẽ mất memory evidence.

---

## 🔗 Điều Hướng

| | |
|--|--|
| ← Encryption & KMS | [4-encryption.md](./4-encryption.md) |
| → Monitoring | [../09-monitoring/README.md](../09-monitoring/README.md) |
| ↑ Security Overview | [README.md](./README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15
