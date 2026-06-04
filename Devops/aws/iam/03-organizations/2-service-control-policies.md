# Service Control Policies (SCPs) — Chính Sách Kiểm Soát Dịch Vụ

> **SCPs** (Service Control Policies — Chính Sách Kiểm Soát Dịch Vụ) là loại policy trong AWS Organizations dùng để giới hạn **quyền tối đa** (maximum permissions) có thể cấp trong một account hoặc OU. SCPs không cấp quyền — chúng chỉ thiết lập *rào chắn* (guardrails).

---

## 📚 Mục Lục

1. [Cơ Chế Hoạt Động](#1-cơ-chế-hoạt-động)
2. [Cú Pháp SCP](#2-cú-pháp-scp)
3. [Chiến Lược Allow-list vs Deny-list](#3-chiến-lược-allow-list-vs-deny-list)
4. [SCPs Thực Tế Phổ Biến](#4-scps-thực-tế-phổ-biến)
5. [Kế Thừa và Đánh Giá SCP](#5-kế-thừa-và-đánh-giá-scp)
6. [Quản Lý SCPs với AWS CLI](#6-quản-lý-scps-với-aws-cli)
7. [Lỗi Phổ Biến và Cách Tránh](#7-lỗi-phổ-biến-và-cách-tránh)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Cơ Chế Hoạt Động

### SCPs vs IAM Policies — Bảng So Sánh

| Đặc Điểm | SCP | IAM Policy |
|---|---|---|
| **Mục đích** | Guardrail — giới hạn tối đa | Authorization — cấp quyền thực tế |
| **Cấp áp dụng** | Organization / OU / Account | User / Group / Role / Resource |
| **Cấp quyền?** | ❌ Không | ✅ Có |
| **Áp dụng cho root user?** | ✅ Có (member accounts) | ❌ Không |
| **Áp dụng cho Management Account?** | ❌ Không bao giờ | ✅ Có |
| **Tích lũy (inheritance)?** | ✅ Kế thừa từ cha | ✅ Explicit per identity |

### Quyền Thực Tế = Giao Của SCP và IAM

```
Công thức:
  Effective Permissions = IAM Policy Allows ∩ SCPs Allow

Ví dụ minh họa:
  IAM Policy: Allow s3:*, ec2:*, rds:*
  SCP:        Allow s3:*, ec2:*  (không cho phép rds:*)
  ──────────────────────────────────────
  Kết quả:    Allow s3:*, ec2:*  ← rds:* bị chặn dù IAM cho phép
```

### Luồng Đánh Giá Khi Có SCPs

```
Request đến AWS API
        │
        ▼
1. SCP Explicit Deny? ──→ CÓ → TỪ CHỐI (dừng)
        │ KHÔNG
        ▼
2. SCP Allow? ──────────→ KHÔNG → TỪ CHỐI (implicit deny)
        │ CÓ
        ▼
3. IAM Explicit Deny? ──→ CÓ → TỪ CHỐI
        │ KHÔNG
        ▼
4. IAM Allow? ──────────→ KHÔNG → TỪ CHỐI
        │ CÓ
        ▼
     CHO PHÉP
```

> **Nguyên tắc quan trọng:** SCPs áp dụng cho **tất cả principals** trong account, kể cả root user của member account (nhưng không áp dụng cho Management Account).

---

## 2. Cú Pháp SCP

### Cấu Trúc JSON Cơ Bản

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TênGợiNhớTùyChọn",
      "Effect": "Allow" | "Deny",
      "Action": [...],
      "Resource": "*",
      "Condition": { ... }  // Tùy chọn
    }
  ]
}
```

**Khác biệt so với IAM Policy:**
- SCP không có trường `Principal` — áp dụng cho mọi principal trong account
- `Resource` chỉ được dùng `*` hoặc ARN (không có `NotResource` trong practice)
- Không hỗ trợ `NotAction`, `NotPrincipal` như IAM

### FullAWSAccess — SCP Mặc Định

AWS tự động đính kèm SCP này vào Root và mọi OU/Account mới tạo:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```

> **Quan trọng:** `FullAWSAccess` không cấp quyền thực sự — nó chỉ nói "SCP không chặn gì cả". Quyền thực tế vẫn phải đến từ IAM policies.

---

## 3. Chiến Lược Allow-list vs Deny-list

### Deny-list Strategy (Chiến Lược Danh Sách Từ Chối) — Phổ Biến Hơn

**Cách hoạt động:** Giữ `FullAWSAccess`, thêm các SCP Deny cụ thể.

```
Mặc định: Allow tất cả (FullAWSAccess)
+ Thêm: Deny các hành động nguy hiểm cụ thể
```

**Ưu điểm:**
- Dễ triển khai, ít ảnh hưởng đến team đang làm việc
- Chỉ cần định nghĩa những gì *không được phép*
- Phù hợp khi bắt đầu áp dụng guardrails

**Nhược điểm:**
- Theo mặc định cho phép mọi dịch vụ mới của AWS
- Cần cập nhật Deny list khi có dịch vụ nguy hiểm mới

```json
// Deny-list example: Ngăn tắt GuardDuty
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyGuardDutyDisable",
      "Effect": "Deny",
      "Action": [
        "guardduty:DeleteDetector",
        "guardduty:DisassociateFromMasterAccount",
        "guardduty:StopMonitoringMembers",
        "guardduty:UpdateDetector"
      ],
      "Resource": "*"
    }
  ]
}
```

### Allow-list Strategy (Chiến Lược Danh Sách Cho Phép) — Nghiêm Ngặt Hơn

**Cách hoạt động:** Xóa `FullAWSAccess`, chỉ Allow các dịch vụ được phê duyệt.

```
Mặc định: Deny tất cả (implicit)
+ Thêm: Allow chỉ những dịch vụ được dùng
```

**Ưu điểm:**
- Kiểm soát chặt chẽ nhất — zero-trust approach
- Dịch vụ AWS mới không tự động được phép

**Nhược điểm:**
- Cần liệt kê mọi dịch vụ cần thiết (bảo trì tốn kém)
- Dễ gây access denied cho team nếu quên thêm dịch vụ
- Phức tạp khi có nhiều team với nhu cầu khác nhau

```json
// Allow-list example: Chỉ cho phép EC2, S3, RDS, CloudWatch
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCoreServices",
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "s3:*",
        "rds:*",
        "cloudwatch:*",
        "logs:*",
        "iam:*",
        "sts:AssumeRole",
        "support:*"
      ],
      "Resource": "*"
    }
  ]
}
```

> **Khuyến nghị thực tế:** Hầu hết tổ chức dùng **Deny-list** vì dễ triển khai hơn. Allow-list phù hợp cho Sandbox OU hoặc tài khoản có yêu cầu tuân thủ cực cao.

---

## 4. SCPs Thực Tế Phổ Biến

### SCP 1: Deny Tắt Security Services (Guardrail Bắt Buộc)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDisableSecurityServices",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:DeleteTrail",
        "cloudtrail:StopLogging",
        "cloudtrail:UpdateTrail",
        "config:DeleteConfigRule",
        "config:DeleteConfigurationRecorder",
        "config:DeleteDeliveryChannel",
        "config:StopConfigurationRecorder",
        "guardduty:DeleteDetector",
        "guardduty:DisassociateFromAdministratorAccount",
        "securityhub:DisableSecurityHub",
        "securityhub:DeleteMembers",
        "access-analyzer:DeleteAnalyzer"
      ],
      "Resource": "*"
    }
  ]
}
```

### SCP 2: Deny Root User Actions (Chặn Dùng Root User)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootUserActions",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    }
  ]
}
```

> **Lưu ý:** SCP này chặn cả hành động chỉ root mới làm được. Cần cẩn thận — có một số tác vụ khẩn cấp cần root. Thường chỉ áp dụng cho Workloads OU, không áp cho Security OU.

### SCP 3: Deny Tạo IAM User (Buộc Dùng Roles/SSO)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyIAMUserCreation",
      "Effect": "Deny",
      "Action": [
        "iam:CreateUser",
        "iam:CreateAccessKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalArn": [
            "arn:aws:iam::*:role/BreakGlassRole",
            "arn:aws:iam::*:role/OrganizationAccountAccessRole"
          ]
        }
      }
    }
  ]
}
```

### SCP 4: Giới Hạn Region (Region Lockdown)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyOutsideApprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "a4b:*",
        "acm:*",
        "aws-marketplace-management:*",
        "aws-marketplace:*",
        "budgets:*",
        "ce:*",
        "chime:*",
        "cloudfront:*",
        "config:*",
        "cur:*",
        "directconnect:*",
        "ec2:DescribeRegions",
        "ecr-public:*",
        "globalaccelerator:*",
        "health:*",
        "iam:*",
        "importexport:*",
        "organizations:*",
        "pricing:*",
        "route53:*",
        "route53domains:*",
        "s3:GetAccountPublic*",
        "s3:ListAllMyBuckets",
        "s3:PutAccountPublic*",
        "shield:*",
        "sts:*",
        "support:*",
        "trustedadvisor:*",
        "waf-regional:*",
        "waf:*",
        "wafv2:*",
        "wellarchitected:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "ap-southeast-1",
            "us-east-1"
          ]
        }
      }
    }
  ]
}
```

> **Lưu ý:** Dùng `NotAction` thay `Action` để loại trừ các global services như IAM, Route53, CloudFront — những dịch vụ này không gắn với region cụ thể nhưng vẫn cần dùng được.

### SCP 5: Protect Log Archive Account (Bảo Vệ S3 Audit Logs)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyLogBucketModification",
      "Effect": "Deny",
      "Action": [
        "s3:DeleteBucket",
        "s3:DeleteBucketPolicy",
        "s3:DeleteObject",
        "s3:DeleteObjectVersion",
        "s3:PutBucketPolicy",
        "s3:PutBucketLogging",
        "s3:PutEncryptionConfiguration",
        "s3:PutLifecycleConfiguration",
        "s3:PutReplicationConfiguration",
        "s3:PutBucketVersioning",
        "s3:PutObjectAcl",
        "s3:PutObjectVersionAcl"
      ],
      "Resource": [
        "arn:aws:s3:::org-cloudtrail-logs-*",
        "arn:aws:s3:::org-cloudtrail-logs-*/*"
      ]
    }
  ]
}
```

### SCP 6: Deny Tắt VPC Flow Logs

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyVPCFlowLogDisable",
      "Effect": "Deny",
      "Action": [
        "ec2:DeleteFlowLogs",
        "logs:DeleteLogGroup",
        "logs:DeleteLogStream"
      ],
      "Resource": "*"
    }
  ]
}
```

### SCP 7: Require MFA cho Hành Động Nhạy Cảm

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyHighRiskActionsWithoutMFA",
      "Effect": "Deny",
      "Action": [
        "iam:DeletePolicy",
        "iam:DeletePolicyVersion",
        "iam:SetDefaultPolicyVersion",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:PutRolePolicy",
        "iam:DeleteRolePolicy",
        "organizations:LeaveOrganization"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

### SCP 8: Suspended Account — Chặn Hoàn Toàn

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllExceptBilling",
      "Effect": "Deny",
      "NotAction": [
        "budgets:*",
        "ce:*",
        "cur:*",
        "health:*",
        "support:*",
        "organizations:DescribeOrganization"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 5. Kế Thừa và Đánh Giá SCP

### Cơ Chế Kế Thừa (Inheritance)

```
Root ← SCP-A (Deny region ngoài us-east-1)
│
├── OU: Production ← SCP-B (Deny tắt GuardDuty) + kế thừa SCP-A
│   │
│   └── Account: Prod-App ← kế thừa SCP-A + SCP-B
│
└── OU: Security ← Không thêm SCP gì + kế thừa SCP-A
    │
    └── Account: Security-Tooling ← kế thừa SCP-A
```

**Quyền thực tế của Prod-App:**
- Bị chặn bởi SCP-A (region)
- Bị chặn bởi SCP-B (tắt GuardDuty)
- Phần còn lại phụ thuộc vào IAM

### Effective SCP = Giao Của Tất Cả SCPs Trên Chuỗi Cha

```
Effective Allow = FullAWSAccess
               ∩ (tất cả Allow SCPs từ Root đến Account)
               - (tất cả Deny SCPs từ Root đến Account)
```

### Kiểm Tra SCPs Đang Áp Dụng

```bash
# Xem SCPs được gắn vào một OU
aws organizations list-policies-for-target \
  --target-id ou-exampleid \
  --filter SERVICE_CONTROL_POLICY

# Xem SCPs được gắn vào một account
aws organizations list-policies-for-target \
  --target-id 123456789012 \
  --filter SERVICE_CONTROL_POLICY

# Xem chi tiết một SCP
aws organizations describe-policy \
  --policy-id p-examplepolicyid
```

---

## 6. Quản Lý SCPs với AWS CLI

### Tạo và Áp Dụng SCP

```bash
# Tạo SCP từ file
aws organizations create-policy \
  --content file://deny-disable-security-services.json \
  --name "DenyDisableSecurityServices" \
  --description "Ngăn tắt các dịch vụ bảo mật cốt lõi" \
  --type SERVICE_CONTROL_POLICY

# Gán SCP vào OU
aws organizations attach-policy \
  --policy-id "p-examplepolicyid" \
  --target-id "ou-exampleid"

# Gán SCP vào Account cụ thể
aws organizations attach-policy \
  --policy-id "p-examplepolicyid" \
  --target-id "123456789012"

# Gỡ SCP khỏi OU
aws organizations detach-policy \
  --policy-id "p-examplepolicyid" \
  --target-id "ou-exampleid"
```

### Cập Nhật SCP

```bash
# Cập nhật nội dung SCP
aws organizations update-policy \
  --policy-id "p-examplepolicyid" \
  --content file://updated-scp.json \
  --description "Cập nhật: thêm s3:DeleteBucket vào danh sách Deny"
```

### Kiểm Tra SCP Trước Khi Áp Dụng

```bash
# Simulate — Dùng IAM Policy Simulator (không hỗ trợ SCP trực tiếp)
# Thay vào đó: dùng IAM Access Analyzer
aws accessanalyzer validate-policy \
  --policy-document file://scp.json \
  --policy-type RESOURCE_POLICY  # Gần nhất với SCP syntax

# Kiểm tra list targets của một policy
aws organizations list-targets-for-policy \
  --policy-id "p-examplepolicyid"
```

---

## 7. Lỗi Phổ Biến và Cách Tránh

### Lỗi 1: Tắt Dịch Vụ AWS Bản Thân (Breaking Self)

```json
// ❌ NGUY HIỂM: SCP này tắt luôn cả Organizations API
{
  "Effect": "Deny",
  "Action": "organizations:*",
  "Resource": "*"
}

// ✅ AN TOÀN HƠN: Chỉ chặn các hành động nguy hiểm
{
  "Effect": "Deny",
  "Action": "organizations:LeaveOrganization",
  "Resource": "*"
}
```

### Lỗi 2: Quên Loại Trừ Global Services Khi Deny Region

```json
// ❌ SAI: Sẽ chặn cả IAM, Route53, CloudFront (global services không có region)
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": "ap-southeast-1"
    }
  }
}

// ✅ ĐÚNG: Dùng NotAction để loại trừ global services
{
  "Effect": "Deny",
  "NotAction": ["iam:*", "sts:*", "route53:*", "cloudfront:*", ...],
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": "ap-southeast-1"
    }
  }
}
```

### Lỗi 3: Lock Out Management Account

```
Management Account không bị ảnh hưởng bởi SCPs →
Nhưng nếu bạn muốn test SCP ở account chính (trước khi tổ chức) →
SCP sẽ không có tác dụng → tưởng SCP hoạt động sai
```

### Lỗi 4: Gán SCP Deny Quá Rộng Vào Root

```
SCP Deny áp dụng lên Root → ảnh hưởng TẤT CẢ accounts (trừ Management)
→ Bao gồm cả Security OU, Log Archive
→ Có thể tự lock out khả năng truy cập/audit

✅ Giải pháp: Dùng Condition để loại trừ emergency roles:
"Condition": {
  "StringNotLike": {
    "aws:PrincipalArn": "arn:aws:iam::*:role/BreakGlassEmergencyRole"
  }
}
```

### Lỗi 5: Vượt Giới Hạn 5,120 Ký Tự

```bash
# Kiểm tra kích thước SCP trước khi tạo
wc -c scp.json

# Nếu > 5,120 bytes: tách thành nhiều SCP nhỏ hơn
# Mỗi OU/Account có thể gắn tối đa 5 SCPs
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: SCP có áp dụng cho Service-Linked Roles (Role Liên Kết Dịch Vụ) không?**

> Có. SCP áp dụng cho mọi principal trong account, kể cả service-linked roles. Nếu SCP Deny một action mà AWS service cần dùng qua service-linked role, action đó sẽ bị chặn. Đây là lý do cần cẩn thận khi viết Deny quá rộng.

**Q: Một account có thể gắn tối đa bao nhiêu SCP?**

> Tổng số SCPs được gắn vào một account (từ account itself, OU cha, và Root) bị giới hạn bởi kích thước tổng cộng và số lượng policies. Mỗi target (OU/Account) có thể đính kèm tối đa 5 SCPs trực tiếp. Kế thừa từ cha không tính vào giới hạn này.

**Q: Có thể dùng SCP để cấp quyền không?**

> Không. SCP không bao giờ cấp quyền — chúng chỉ giới hạn quyền tối đa. Kể cả khi SCP Allow `s3:*`, nếu IAM policy không cho phép `s3:GetObject`, action đó vẫn bị từ chối.

**Q: SCP Allow-list hay Deny-list nên dùng khi nào?**

> **Deny-list** phù hợp khi: tổ chức đang dùng nhiều dịch vụ AWS, muốn bắt đầu áp guardrails mà không gián đoạn. **Allow-list** phù hợp khi: Sandbox OU muốn giới hạn chặt chi phí, hoặc tài khoản PCI-DSS cần chỉ dùng dịch vụ đã kiểm chứng.

**Q: Làm thế nào debug khi SCP gây ra Access Denied ngoài ý muốn?**

> 1. Xem CloudTrail event — trường `serviceControlPolicies` trong authorization detail. 2. Dùng `aws organizations list-policies-for-target` để xem SCPs đang áp dụng. 3. Kiểm tra từng SCP để tìm Deny matching action. 4. Dùng IAM Policy Simulator (hỗ trợ một phần SCP simulation).

---

## 🔗 Tiếp Theo

- **[3-control-tower.md](3-control-tower.md)** — Tự động hóa việc tạo OU, gán SCPs, thiết lập baseline
- **[../01-iam-fundamentals/2-policy-types.md](../01-iam-fundamentals/2-policy-types.md)** — So sánh SCP với các loại policy khác
- **[../09-compliance-governance/3-firewall-manager.md](../09-compliance-governance/3-firewall-manager.md)** — Quản lý WAF/SG tập trung theo Organizations
