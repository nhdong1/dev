# System Design — Thiết Kế Kiến Trúc Bảo Mật AWS

> Hướng dẫn giải quyết các bài toán system design bảo mật trong phỏng vấn — bao gồm phương pháp tiếp cận, mẫu kiến trúc thực tế, và cách trình bày ấn tượng.

---

## 📑 Mục Lục

1. [Framework Tiếp Cận System Design Security](#framework-tiếp-cận-system-design-security)
2. [Bài Toán 1: Kiến Trúc Multi-Account Doanh Nghiệp](#bài-toán-1-kiến-trúc-multi-account-doanh-nghiệp)
3. [Bài Toán 2: Bảo Mật Ứng Dụng Fintech PCI-DSS](#bài-toán-2-bảo-mật-ứng-dụng-fintech-pci-dss)
4. [Bài Toán 3: Security Monitoring Đa Tài Khoản](#bài-toán-3-security-monitoring-đa-tài-khoản)
5. [Bài Toán 4: Zero Trust cho Remote Workforce](#bài-toán-4-zero-trust-cho-remote-workforce)
6. [Bài Toán 5: Secure Data Lake trên S3](#bài-toán-5-secure-data-lake-trên-s3)
7. [Nguyên Tắc Trình Bày System Design](#nguyên-tắc-trình-bày-system-design)

---

## Framework Tiếp Cận System Design Security

Khi nhận bài toán system design trong phỏng vấn, luôn đi theo framework sau:

### Bước 1: Clarify Requirements (Làm Rõ Yêu Cầu) — 5 phút

```
Câu hỏi cần đặt:
1. Scale: Bao nhiêu users? Bao nhiêu team? Bao nhiêu accounts?
2. Compliance: Có regulatory requirement không? (PCI-DSS, HIPAA, SOC2?)
3. Threat model: Mối đe dọa chính là gì? External attack, insider threat, hay cả hai?
4. Budget: Cost constraint có không?
5. Existing infrastructure: Green field hay migration từ existing?
6. Team maturity: Dev team có kinh nghiệm với AWS security không?
```

### Bước 2: Identify Security Domains (Xác Định Các Lĩnh Vực Bảo Mật)

```
IAM & Identity    → Ai được làm gì?
Network Security  → Traffic flows như thế nào?
Data Protection   → Dữ liệu được bảo vệ ở đâu, lúc nào?
Monitoring        → Phát hiện và phản hồi sự cố thế nào?
Compliance        → Evidence collection và reporting?
```

### Bước 3: Design High-Level Architecture (Thiết Kế Kiến Trúc Tổng Quát)

```
Vẽ từ ngoài vào trong:
Internet → Perimeter → Network → Application → Data

Tại mỗi lớp: ai/gì có thể access? Điều kiện nào?
```

### Bước 4: Deep Dive Các Thành Phần Quan Trọng

```
Tập trung vào những gì phỏng vấn viên quan tâm nhất:
- IAM role design (luôn quan trọng)
- Encryption key management
- Incident detection và response
```

### Bước 5: Discuss Trade-offs (Thảo Luận Đánh Đổi)

```
Luôn đề cập:
- Security vs Performance
- Security vs Developer Productivity
- Cost implications
- Operational complexity
```

---

## Bài Toán 1: Kiến Trúc Multi-Account Doanh Nghiệp

### Đề Bài

*"Bạn được thuê vào một công ty fintech có 500 kỹ sư, 50 team, đang dùng 1 AWS account duy nhất cho tất cả mọi thứ. Thiết kế kiến trúc multi-account an toàn."*

### Clarifying Questions

- Hiện tại có bao nhiêu environments? (dev, staging, prod)
- Compliance requirements? (PCI-DSS, SOC2)
- Có SSO (Single Sign-On) hay chưa?
- Timeline migration là bao lâu?

### Kiến Trúc Đề Xuất

```
AWS Organizations
├── Management Account (Tài Khoản Quản Lý)
│   └── Chỉ dùng để quản lý organization + billing
│   └── SCP global: enforce MFA, disable unused regions, protect audit trail
│
├── Security OU (Đơn Vị Tổ Chức Bảo Mật)
│   ├── Log Archive Account
│   │   └── S3 bucket nhận CloudTrail + Config logs từ tất cả accounts
│   │   └── S3 Object Lock (WORM — Write Once Read Many) 7 năm
│   │   └── Glacier Intelligent Tiering sau 90 ngày
│   ├── Security Tooling Account
│   │   └── GuardDuty Administrator — nhận findings từ tất cả accounts
│   │   └── Security Hub Administrator — aggregate tất cả findings
│   │   └── Macie Administrator — scan S3 toàn org
│   │   └── Firewall Manager Admin
│   └── Audit Account
│       └── Audit Manager — evidence collection
│       └── AWS Config Aggregator
│       └── Access Analyzer — org-wide findings
│
├── Infrastructure OU
│   ├── Network Account (Tài Khoản Mạng)
│   │   └── Transit Gateway — hub cho tất cả VPC connectivity
│   │   └── Shared VPC subnets (qua RAM — Resource Access Manager)
│   │   └── Route53 Private Hosted Zones
│   │   └── Direct Connect / Site-to-Site VPN
│   └── Shared Services Account
│       └── IAM Identity Center (SSO)
│       └── AWS CodeArtifact (artifact registry)
│       └── Internal tooling (CI/CD, monitoring)
│
├── Workloads OU
│   ├── Production OU
│   │   ├── SCP: Deny delete CloudTrail, enforce encryption
│   │   ├── Prod-Payments-Account (PCI-DSS scope)
│   │   ├── Prod-Platform-Account
│   │   └── Prod-Data-Account
│   ├── Non-Production OU
│   │   ├── SCP: Restrict instance types, enforce tagging
│   │   └── Staging-Account, QA-Account
│   └── Development OU
│       ├── SCP: Budget cap, restrict expensive resources
│       └── Dev-TeamA-Account, Dev-TeamB-Account, ...
│
└── Sandbox OU (Isolated hoàn toàn, không connect về prod)
    └── Engineer-Personal-Accounts
```

### IAM Identity Center — Permission Sets

```
Permission Sets (Tập Quyền Hạn):
├── PlatformAdmin    → AdministratorAccess (chỉ infrastructure account)
├── SecurityAnalyst  → ReadOnlyAccess + GuardDuty + SecurityHub
├── DeveloperFull    → Full access trong dev account
├── DeveloperLimited → Restricted trong staging/prod (read-only + deploy)
├── DataEngineer     → S3 + Glue + Athena + EMR trong data accounts
└── FinancialAuditor → Billing + Cost Explorer + ReadOnly

Assignment:
├── Team-Platform → PlatformAdmin → Network-Account, Shared-Services-Account
├── Team-Security → SecurityAnalyst → ALL accounts (read)
├── Team-Alpha → DeveloperFull → Dev-TeamAlpha-Account
│              → DeveloperLimited → Staging-Account
└── Team-Alpha → ReadOnlyAccess → Production accounts (monitoring chỉ)
```

### SCPs Quan Trọng

```json
// SCP 1: Chỉ cho phép các region đã approved
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["us-east-1", "us-west-2", "ap-southeast-1"]
    },
    "StringNotEquals": {
      "aws:PrincipalArn": "arn:aws:iam::*:role/OrganizationAccountAccessRole"
    }
  }
}

// SCP 2: Bảo vệ audit infrastructure
{
  "Effect": "Deny",
  "Action": [
    "cloudtrail:DeleteTrail", "cloudtrail:StopLogging",
    "guardduty:DeleteDetector", "guardduty:DisassociateFromMasterAccount",
    "config:DeleteConfigRule", "config:StopConfigurationRecorder"
  ],
  "Resource": "*"
}

// SCP 3: Enforce encryption cho EBS và S3
{
  "Effect": "Deny",
  "Action": "ec2:RunInstances",
  "Resource": "arn:aws:ec2:*:*:volume/*",
  "Condition": {
    "Bool": {"ec2:Encrypted": "false"}
  }
}
```

### Trade-offs Cần Đề Cập

| Vấn Đề | Trade-off |
|---|---|
| Nhiều accounts hơn | Chi phí quản lý tăng, nhưng blast radius (phạm vi ảnh hưởng khi sự cố) nhỏ hơn |
| SCP nghiêm ngặt | Developer bị cản trở, nhưng security control mạnh hơn |
| IAM Identity Center | Dependency vào một điểm trung tâm — phải có disaster recovery plan |
| Transit Gateway | Chi phí thêm cho network, nhưng centralize security inspection |

---

## Bài Toán 2: Bảo Mật Ứng Dụng Fintech PCI-DSS

### Đề Bài

*"Thiết kế kiến trúc AWS cho ứng dụng payment xử lý thẻ tín dụng — phải tuân thủ PCI-DSS Level 1 (hơn 6 triệu transactions/năm)."*

### Phân Tích CDE (Cardholder Data Environment — Môi Trường Dữ Liệu Chủ Thẻ)

```
CHD (Cardholder Data — Dữ Liệu Chủ Thẻ) cần bảo vệ:
├── PAN (Primary Account Number — Số Tài Khoản Chính): 16 chữ số thẻ
├── Cardholder Name (Tên Chủ Thẻ)
├── Expiration Date (Ngày Hết Hạn)
└── Service Code (Mã Dịch Vụ)

SAD (Sensitive Authentication Data — Dữ Liệu Xác Thực Nhạy Cảm):
├── CVV/CVC — KHÔNG được lưu sau authorization
├── PIN — KHÔNG bao giờ được lưu
└── Full magnetic stripe data — KHÔNG được lưu
```

### Kiến Trúc PCI-DSS

```
INTERNET
    │
    ▼
[AWS Shield Advanced] ← PCI Req 6.6: DDoS protection
    │
[WAF] ← PCI Req 6.6: Block SQLi, XSS, OWASP Top 10
    │
[CloudFront] ← Edge layer, TLS 1.2+ enforcement
    │
[ALB — Application Load Balancer]  (Public Subnet)
│   └── Access logs → S3 (encrypted KMS)
    │
[ECS Fargate — Payment API]  (Private Subnet — CDE)
│   └── Task role: chỉ access payment-specific resources
│   └── Secrets Manager: API keys, DB credentials
│   └── IMDSv2 enforced
    │
[PrivateLink → Payment Network] ← PCI Req 4: Encrypted transmission
    │
[RDS PostgreSQL Multi-AZ]  (Isolated Subnet — CDE)
│   └── Encryption at rest: KMS CMK customer-managed
│   └── SSL/TLS cho connections
│   └── No public accessibility
│   └── Automated backup encrypted
    │
[DynamoDB — Session Store]
│   └── Encryption với KMS
│   └── VPC Endpoint (không qua internet)
```

### Security Controls Theo PCI-DSS Requirements

```
Req 1: Firewall
└── Security Groups: whitelist-only, no 0.0.0.0/0 inbound
└── NACLs: deny all trừ explicit allows
└── Network Firewall: deep packet inspection cho CDE traffic

Req 2: No vendor defaults
└── Systems Manager Patch Manager: automated patching
└── Secrets Manager: no default passwords
└── AWS Config Rule: restricted-ssh, no-unrestricted-admin-ports

Req 3: Protect stored CHD
└── RDS: SSE với KMS CMK
└── S3 (nếu có): SSE-KMS + S3 Object Lock
└── Macie: scan tìm PAN bị lưu sai chỗ
└── DLP (Data Loss Prevention — Ngăn Chặn Mất Dữ Liệu): qua Macie

Req 4: Encrypt in transit
└── ACM: TLS 1.2+ certificates
└── Network Firewall: block non-TLS traffic ra internet
└── VPC Endpoints: traffic không qua internet

Req 7: Least privilege
└── IAM roles per-service (không share)
└── Permission Boundaries cho developer-created roles
└── SCP: restrict actions trong payment account

Req 8: Authentication
└── IAM Identity Center + MFA bắt buộc
└── Privileged Access Workstations (PAW — Trạm Làm Việc Đặc Quyền) cho admin
└── Temporary credentials (AssumeRole), không long-lived keys

Req 10: Audit logging
└── CloudTrail: organization trail, S3 Object Lock
└── VPC Flow Logs: tất cả traffic trong CDE
└── RDS logs: audit log mọi query
└── ECS logs → CloudWatch Logs → Kinesis → S3

Req 11: Security testing
└── Inspector: automated vulnerability scanning
└── GuardDuty: realtime threat detection
└── Penetration testing: quarterly, theo PCI requirement
└── AWS Config Rules: continuous compliance checking
```

### Key Management cho PCI-DSS

```
KMS Key Hierarchy:
├── Root Key (CMK) — 1 key cho CDE
│   ├── RDS encryption key
│   ├── S3 encryption key (CHD backups)
│   └── EBS encryption key (Fargate volumes)
│
├── Key Policy: chỉ payment-service role + admin role được dùng
├── Key Rotation: auto-rotate annually
└── Key access log: CloudTrail ghi mọi kms:Decrypt call

CloudHSM (nếu cần FIPS 140-2 Level 3):
└── Thay KMS cho key storage
└── Dùng khi auditor yêu cầu dedicated HSM
```

---

## Bài Toán 3: Security Monitoring Đa Tài Khoản

### Đề Bài

*"Thiết kế hệ thống centralized security monitoring cho 100 AWS accounts, với khả năng phát hiện mối đe dọa realtime và alert trong vòng 5 phút."*

### Kiến Trúc Security Monitoring

```
100 MEMBER ACCOUNTS
├── GuardDuty (detector enabled)
├── Security Hub (enabled, auto-sending findings)
├── CloudTrail (organization trail → S3 centralized)
├── Config Recorder (configuration changes)
└── VPC Flow Logs → CloudWatch Logs

                    │ (findings via Organizations delegation)
                    ▼

SECURITY TOOLING ACCOUNT (Tài Khoản Công Cụ Bảo Mật)
├── GuardDuty Administrator Account
│   └── Nhận aggregated findings từ 100 accounts
│   └── Custom threat lists (IP blocklists)
│   └── Suppression rules (lọc false positives)
│
├── Security Hub Administrator Account
│   └── ASFF (Amazon Security Finding Format — Định Dạng Finding Bảo Mật)
│   └── Aggregation từ GuardDuty, Inspector, Macie, Config
│   └── AWS Foundational Security Best Practices standard
│   └── CIS AWS Foundations Benchmark standard
│   └── Custom insights cho executive reporting
│
├── EventBridge (event routing)
│   └── Rule 1: GuardDuty HIGH/CRITICAL → Lambda isolation
│   └── Rule 2: Security Hub CRITICAL → PagerDuty
│   └── Rule 3: Config non-compliant → SNS → Slack
│   └── Rule 4: Custom patterns → SOAR platform
│
├── Lambda Functions (auto-remediation)
│   └── isolate-ec2: quarantine compromised instance
│   └── revoke-credentials: invalidate leaked keys
│   └── block-ip: add to WAF IP set
│   └── notify-oncall: context-rich alert
│
└── OpenSearch Service (SIEM — tìm kiếm và phân tích)
    └── CloudTrail events → Kinesis Firehose → OpenSearch
    └── VPC Flow Logs → OpenSearch
    └── Dashboards: security overview, compliance, anomaly

LOG ARCHIVE ACCOUNT
└── S3 bucket (versioning + Object Lock)
    ├── cloudtrail/ — organization trail
    ├── config/ — configuration history
    ├── vpcflowlogs/ — network logs
    └── accesslogs/ — S3, ALB, CloudFront access logs
```

### Alert Routing (Phân Tầng Cảnh Báo)

```
Severity CRITICAL (9.0–10.0):
└── EventBridge → Lambda (immediate isolation) + PagerDuty (oncall wakeup)
└── SLA: 5 phút phản hồi

Severity HIGH (7.0–8.9):
└── EventBridge → SNS → Slack #security-alerts
└── Automatic triage ticket trong Jira
└── SLA: 30 phút phản hồi

Severity MEDIUM (4.0–6.9):
└── Security Hub insight aggregation
└── Daily digest report
└── SLA: next business day

Severity LOW (1.0–3.9):
└── Logged for metrics
└── Weekly trend report
└── No immediate action
```

### Dashboard Metrics Quan Trọng

```python
# CloudWatch Custom Metrics
metrics = {
    "UnresolvedHighFindings": guardduty.count_findings(severity="HIGH"),
    "ComplianceScore": security_hub.get_score(),
    "MeanTimeToDetect": avg_time_from_event_to_finding(),
    "MeanTimeToRemediate": avg_time_from_finding_to_resolution(),
    "RootAccountUsage": cloudtrail.count_root_events(),
    "AccessKeyAge": iam.count_keys_older_than(days=90),
    "UnencryptedResources": config.count_noncompliant("encrypted"),
}
```

---

## Bài Toán 4: Zero Trust cho Remote Workforce

### Đề Bài

*"Công ty có 200 kỹ sư làm remote, cần truy cập internal AWS resources. Hiện tại dùng VPN site-to-site. Thiết kế Zero Trust replacement."*

### Vấn Đề Với VPN Truyền Thống

```
VPN: "Trust the network" (Tin tưởng vào mạng)
├── Một khi vào VPN → truy cập được hầu hết internal resources
├── Lateral movement (di chuyển ngang) dễ dàng nếu compromised
└── Không có device health check, không có context-aware access
```

### Zero Trust Architecture

```
ENGINEER DEVICE (Thiết Bị Kỹ Sư)
└── Endpoint agent: check device compliance (OS patch, AV, disk encryption)
    │
    ▼ (HTTPS request)
AWS VERIFIED ACCESS (Truy Cập Đã Xác Minh)
├── Verify identity: IAM Identity Center / SAML với Okta
├── Verify device: integration với Jamf/Intune device compliance
├── Evaluate access policy per-application:
│   ├── "Allow nếu: authenticated + MFA + device compliant + working hours"
│   ├── "Deny nếu: device không patched, hoặc high-risk login"
│   └── "Require step-up MFA cho sensitive resources"
│
└── Trust Context (Ngữ Cảnh Tin Tưởng) được evaluate realtime
    │
    ▼ (chỉ cho phép nếu policy pass — không phải network-wide)
APPLICATION (chạy trong private VPC)
├── Internal tool A: accessible cho team Alpha, working hours, compliant device
├── Internal tool B: accessible cho all engineers, any time, any compliant device
└── Production database: accessible chỉ cho DBAs, MFA + compliant + IP condition
```

### Implementation

```hcl
# Terraform: AWS Verified Access Group
resource "aws_verifiedaccess_group" "engineering" {
  verifiedaccess_instance_id = aws_verifiedaccess_instance.main.id
  policy_document = <<EOF
    permit(principal, action, resource)
    when {
      principal has credentials.iam_identity_center.sub &&
      context.device.platform_type == "Windows" &&
      context.device.is_compliant == true &&
      context.network.trust_provider == "corporate-oidc"
    };
  EOF
}

# Verified Access Endpoint cho internal tool
resource "aws_verifiedaccess_endpoint" "internal_tool" {
  application_domain        = "tool.internal.company.com"
  attachment_type           = "vpc"
  domain_certificate_arn    = aws_acm_certificate.internal.arn
  endpoint_domain_prefix    = "internal-tool"
  security_group_ids        = [aws_security_group.va_endpoint.id]
  verifiedaccess_group_id   = aws_verifiedaccess_group.engineering.id
  
  network_interface_options {
    network_interface_id = aws_network_interface.tool.id
    port                 = 443
    protocol             = "https"
  }
}
```

---

## Bài Toán 5: Secure Data Lake trên S3

### Đề Bài

*"Xây dựng data lake an toàn trên S3 cho 50 data science teams — mỗi team chỉ được đọc data của project họ, data nhạy cảm phải được mask (che dấu) trước khi analytics."*

### Kiến Trúc Secure Data Lake

```
DATA INGESTION (Thu Thập Dữ Liệu)
├── Kinesis Firehose → S3 raw zone (encryption: SSE-KMS, CMK riêng)
├── DMS → S3 raw zone (database replication)
└── API Gateway → Lambda → S3 raw zone

S3 BUCKET STRUCTURE (Cấu Trúc Bucket)
├── s3://datalake-raw/
│   └── [Restricted — chỉ data engineers và ETL roles]
├── s3://datalake-curated/
│   ├── project-alpha/ [chỉ Team Alpha]
│   ├── project-beta/  [chỉ Team Beta]
│   └── shared/        [tất cả teams]
└── s3://datalake-analytics/
    └── [masked data — data scientists]

DATA CLASSIFICATION & MASKING (Phân Loại & Che Dấu)
├── Macie scan raw zone → tag PII objects
├── AWS Glue ETL:
│   └── Detect PII columns → mask/tokenize trước khi move sang curated
│   └── Tokenize: PAN → hash, SSN → pseudonym
│   └── Generalize: age → age range
└── Lake Formation — column-level security

ACCESS CONTROL (Kiểm Soát Truy Cập) — AWS Lake Formation
├── Database-level: Team Alpha → chỉ project_alpha database
├── Table-level: Data Scientists → aggregate tables only
├── Column-level: HR data → hide salary column cho non-HR
└── Row-level: Customer data → chỉ data của region mình
```

### ABAC Policy cho Data Lake

```json
// IAM Policy dùng ABAC — 1 policy cho tất cả teams
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::datalake-curated",
      "arn:aws:s3:::datalake-curated/*"
    ],
    "Condition": {
      "StringEquals": {
        // Tag của S3 object phải match tag của IAM role
        "s3:ExistingObjectTag/Project": "${aws:PrincipalTag/Project}"
      }
    }
  }]
}

// IAM Role của Team Alpha có tag: Project=alpha
// S3 objects của project Alpha có tag: Project=alpha
// → Team Alpha tự động có quyền đọc data của mình
// → Team Beta không đọc được (tag không match)
```

### Data Encryption Strategy

```
Raw Zone:   SSE-KMS với CMK riêng cho raw data
            └── Key policy: chỉ ETL role được dùng

Curated:    SSE-KMS với CMK per-project
            └── project-alpha-key: chỉ Team Alpha role được dùng
            └── project-beta-key: chỉ Team Beta role được dùng

Analytics:  SSE-KMS với shared analytics key
            └── Masked data — không cần per-team key
```

---

## Nguyên Tắc Trình Bày System Design

### Cấu Trúc Trả Lời (15–20 phút)

```
0–2 phút:   Clarify requirements, đặt câu hỏi
2–5 phút:   High-level architecture, vẽ sơ đồ tổng quan
5–12 phút:  Deep dive các thành phần quan trọng
12–15 phút: Discuss trade-offs và alternative approaches
15–20 phút: Operational considerations (monitoring, DR)
```

### Phrases Ấn Tượng Để Dùng

```
Thay vì:                     Dùng:
"Dùng IAM"              →    "Apply least privilege qua IAM roles per-service 
                              với permission boundaries để delegate admin safely"

"Encrypt data"          →    "Implement envelope encryption — S3 SSE-KMS với 
                              per-project CMKs để tenant isolation"

"Monitor everything"    →    "Defense-in-depth observability: GuardDuty cho 
                              realtime threat detection, Config cho compliance 
                              drift, CloudTrail cho forensics"

"Block bad traffic"     →    "Layered network defense: WAF cho L7 filtering, 
                              Shield Advanced cho DDoS, Network Firewall cho 
                              deep packet inspection"
```

### Trade-offs Phải Đề Cập

Phỏng vấn viên không mong đợi bạn có câu trả lời hoàn hảo — họ muốn thấy bạn hiểu trade-offs:

```
Security vs Cost:
"CloudHSM tốt hơn KMS về FIPS level, nhưng chi phí cao hơn 10x. 
Với workload này, KMS đủ dùng và để budget cho controls khác như GuardDuty."

Security vs Developer Experience:
"Strict permission boundaries làm developer chậm hơn lúc đầu, 
nhưng ngăn privilege escalation — trong production fintech, đây là trade-off chấp nhận được."

Centralized vs Decentralized:
"Centralized GuardDuty admin cho visibility toàn org, nhưng tạo dependency. 
Member accounts nên có local alerting backup."
```

---

**Cập Nhật Lần Cuối:** 2026-05-16
