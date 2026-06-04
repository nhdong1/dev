# Delegated Administrator & Trusted Access (Quản Trị Ủy Quyền & Truy Cập Tin Cậy)

> **Delegated Administrator** (Quản Trị Viên Ủy Quyền) cho phép chỉ định một member account làm quản trị viên cho một AWS service cụ thể trong toàn Organization — thay vì phải thực hiện mọi việc từ Management Account. **Trusted Access** (Truy Cập Tin Cậy) là tính năng nền tảng giúp AWS services tích hợp với Organizations để hoạt động cross-account. Kết hợp hai tính năng này giúp giảm quyền tập trung ở Management Account và phân tán trách nhiệm hợp lý.

---

## 📚 Mục Lục

1. [Vấn Đề Management Account Quá Quyền Lực](#vấn-đề-management-account-quá-quyền-lực)
2. [Trusted Access — Truy Cập Tin Cậy](#trusted-access)
3. [Delegated Administrator — Quản Trị Viên Ủy Quyền](#delegated-administrator)
4. [Các AWS Service Hỗ Trợ Delegated Admin](#aws-services-hỗ-trợ-delegated-admin)
5. [Thiết Lập Security Tooling Account](#thiết-lập-security-tooling-account)
6. [Quản Lý Delegated Admin Bằng CLI](#quản-lý-delegated-admin-bằng-cli)
7. [IAM Roles Cho Cross-Account Access](#iam-roles-cho-cross-account-access)
8. [Best Practice & Patterns](#best-practice--patterns)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Vấn Đề Management Account Quá Quyền Lực

### Anti-Pattern: Tất Cả Từ Management Account

```
❌ Thiết kế sai:
Management Account
├── Chạy Security Hub, GuardDuty, Config Aggregator
├── Là Delegated Admin cho tất cả security services
├── Nhiều team khác nhau đăng nhập vào đây
└── Có payment info + billing admin + security admin cùng lúc

Vấn đề:
- Compromise Management Account = compromise toàn Organization
- Tập trung quyền vi phạm least privilege principle
- Khó audit ai làm gì trong Management Account
- Một người không nên vừa quản billing vừa quản security
```

### Best Practice: Phân Tán Trách Nhiệm

```
✅ Thiết kế đúng:
Management Account
├── Chỉ: Organizations management, Billing, OU/SCP management
├── Tối thiểu IAM users — dùng IAM Identity Center
└── Không chạy workload

Security Tooling Account (Delegated Admin)
├── GuardDuty Administrator
├── Security Hub Administrator
├── AWS Config Aggregator
├── AWS Macie Administrator
└── AWS Inspector Administrator

Log Archive Account
├── CloudTrail Organization Trail logs (S3)
├── Config history (S3)
└── VPC Flow Logs (S3, read-only từ security team)
```

---

## Trusted Access

### Trusted Access Là Gì?

**Trusted Access** (Truy Cập Tin Cậy) cho phép một AWS service tích hợp với Organizations — service có thể thực hiện operations cross-account trong Organization thay mặt bạn.

```
Khi Trusted Access được enable cho một service:
├── Service tạo Service-Linked Role trong mỗi member account
├── Service có thể đọc Organizations metadata (account IDs, OUs...)
├── Service có thể aggregate data từ nhiều accounts
└── Service có thể deploy configuration xuống member accounts
```

### Bật/Tắt Trusted Access

```bash
# Liệt kê services đang có Trusted Access
aws organizations list-aws-services-access-for-organization

# Bật Trusted Access cho một service
aws organizations enable-aws-service-access \
  --service-principal config.amazonaws.com

# Tắt Trusted Access
aws organizations disable-aws-service-access \
  --service-principal config.amazonaws.com
```

### Service Principals Phổ Biến

```bash
# Security services
aws organizations enable-aws-service-access \
  --service-principal guardduty.amazonaws.com

aws organizations enable-aws-service-access \
  --service-principal securityhub.amazonaws.com

aws organizations enable-aws-service-access \
  --service-principal config.amazonaws.com

aws organizations enable-aws-service-access \
  --service-principal macie.amazonaws.com

aws organizations enable-aws-service-access \
  --service-principal inspector2.amazonaws.com

# Management services
aws organizations enable-aws-service-access \
  --service-principal cloudtrail.amazonaws.com

aws organizations enable-aws-service-access \
  --service-principal sso.amazonaws.com  # IAM Identity Center

aws organizations enable-aws-service-access \
  --service-principal ram.amazonaws.com  # Resource Access Manager

# Cost & Compliance
aws organizations enable-aws-service-access \
  --service-principal tagpolicies.tag.amazonaws.com

aws organizations enable-aws-service-access \
  --service-principal backup.amazonaws.com
```

---

## Delegated Administrator

### Delegated Administrator Là Gì?

Sau khi bật Trusted Access, bạn có thể **ủy quyền** quản lý service đó cho một member account cụ thể — account đó trở thành **Delegated Administrator** cho service đó.

```
Management Account:
  bật Trusted Access → enable-aws-service-access --service-principal guardduty.amazonaws.com
  ủy quyền → register-delegated-administrator --account-id SECURITY_ACCOUNT --service-principal guardduty.amazonaws.com

Security Tooling Account (Delegated Admin cho GuardDuty):
  ├── Có thể enable/disable GuardDuty trong member accounts
  ├── Có thể xem findings từ toàn Organization
  ├── Có thể tạo suppression rules áp dụng toàn Organization
  └── KHÔNG cần đăng nhập vào Management Account để làm những việc này
```

### Giới Hạn Delegated Administrator

```
Mỗi service có giới hạn số Delegated Admin:
├── Hầu hết services: 1–3 Delegated Admin
├── Không thể set Management Account là Delegated Admin
└── Một account có thể là Delegated Admin cho nhiều services

Quyền của Delegated Admin:
├── Đặc quyền riêng của từng service (không phải toàn bộ Organizations)
└── Không có quyền tạo/xóa accounts, tạo OU, sửa SCP
```

---

## AWS Services Hỗ Trợ Delegated Admin

### Security Services

| Service | Delegated Admin Có Thể Làm | Max Admin |
|---------|---------------------------|-----------|
| **GuardDuty** | Xem/quản lý findings toàn Org, tạo suppression rules | 1 |
| **Security Hub** | Aggregate findings, tạo insights, quản lý standards | 1 |
| **AWS Config** | Config Aggregator cho toàn Org, xem compliance | 1 |
| **Macie** | Quản lý sensitive data discovery toàn Org | 1 |
| **Inspector** | Quản lý vulnerability assessments | 1 |
| **Firewall Manager** | Quản lý WAF, Shield rules toàn Org | 1 |
| **Access Analyzer** | Aggregate access analysis toàn Org | 1 |

### Management Services

| Service | Delegated Admin Có Thể Làm | Max Admin |
|---------|---------------------------|-----------|
| **CloudFormation StackSets** | Deploy StackSets tới member accounts | 1 |
| **AWS Backup** | Quản lý backup policies, audit reports | 1 |
| **S3 Storage Lens** | Aggregate storage analytics toàn Org | 1 |
| **Resource Access Manager** | Chia sẻ resources cross-account | - |
| **Service Catalog** | Quản lý portfolios cho toàn Org | - |
| **AWS SSO/Identity Center** | Quản lý user access cho toàn Org | 1 |

---

## Thiết Lập Security Tooling Account

### Kiến Trúc Security Hub + GuardDuty Centralized

```
Step 1: Trong Management Account
  → Enable Trusted Access cho GuardDuty, Security Hub
  → Register Security Tooling Account as Delegated Admin

Step 2: Trong Security Tooling Account
  → Enable GuardDuty cho toàn Organization
  → Auto-enable GuardDuty trong accounts mới join
  → Enable Security Hub với standards (CIS, PCI-DSS, FSBP)
  → Configure findings aggregation

Step 3: Trong Member Accounts (tự động)
  → GuardDuty detector tự động được tạo
  → Security Hub tự động được enable
  → Findings tự động gửi về Security Tooling Account
```

### Thực Hành: Thiết Lập GuardDuty Organization

```bash
# === Trong Management Account ===

# 1. Enable Trusted Access cho GuardDuty
aws organizations enable-aws-service-access \
  --service-principal guardduty.amazonaws.com

# 2. Enable GuardDuty trong Management Account (bắt buộc trước)
aws guardduty create-detector \
  --enable \
  --finding-publishing-frequency SIX_HOURS

# 3. Register Security Tooling Account làm Delegated Admin
aws organizations register-delegated-administrator \
  --account-id 999888777666 \
  --service-principal guardduty.amazonaws.com


# === Trong Security Tooling Account ===

# 4. Lấy Detector ID của Security Tooling Account
DETECTOR_ID=$(aws guardduty list-detectors --query 'DetectorIds[0]' --output text)

# 5. Enable GuardDuty cho toàn Organization (auto-enable accounts)
aws guardduty create-members \
  --detector-id $DETECTOR_ID \
  --account-details '[
    {"AccountId": "111111111111", "Email": "account-a@company.com"},
    {"AccountId": "222222222222", "Email": "account-b@company.com"}
  ]'

# 6. Bật auto-enable cho accounts mới join Organization
aws guardduty update-organization-configuration \
  --detector-id $DETECTOR_ID \
  --auto-enable NEW

# 7. Xem findings từ toàn Organization
aws guardduty list-findings \
  --detector-id $DETECTOR_ID \
  --finding-criteria '{
    "Criterion": {
      "severity": {"Gte": 7}
    }
  }'
```

### Thực Hành: Thiết Lập Config Aggregator

```bash
# === Trong Management Account ===
aws organizations enable-aws-service-access \
  --service-principal config.amazonaws.com

aws organizations register-delegated-administrator \
  --account-id 999888777666 \
  --service-principal config.amazonaws.com


# === Trong Security Tooling Account ===

# Tạo Config Aggregator cho toàn Organization
aws configservice put-configuration-aggregator \
  --configuration-aggregator-name OrganizationAggregator \
  --organization-aggregation-source '{
    "RoleArn": "arn:aws:iam::999888777666:role/aws-service-role/config.amazonaws.com/AWSServiceRoleForConfig",
    "AllAwsRegions": true
  }'

# Xem compliance summary từ toàn Org
aws configservice get-aggregate-compliance-details-by-config-rule \
  --configuration-aggregator-name OrganizationAggregator \
  --config-rule-name required-tags
```

---

## Quản Lý Delegated Admin Bằng CLI

```bash
# Liệt kê tất cả Delegated Administrators
aws organizations list-delegated-administrators

# Liệt kê Delegated Admins cho service cụ thể
aws organizations list-delegated-administrators \
  --service-principal guardduty.amazonaws.com

# Liệt kê services mà một account đang là Delegated Admin
aws organizations list-delegated-services-for-account \
  --account-id 999888777666

# Hủy bỏ Delegated Admin (từ Management Account)
aws organizations deregister-delegated-administrator \
  --account-id 999888777666 \
  --service-principal guardduty.amazonaws.com
```

---

## IAM Roles Cho Cross-Account Access

### OrganizationAccountAccessRole

Khi tạo member account mới qua Organizations, AWS tự động tạo role này:

```json
{
  "RoleName": "OrganizationAccountAccessRole",
  "AssumeRolePolicyDocument": {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::MANAGEMENT_ACCOUNT_ID:root"
      },
      "Action": "sts:AssumeRole"
    }]
  },
  "Policies": ["AdministratorAccess"]
}
```

**Lưu ý:** Role này cực kỳ quyền lực (AdministratorAccess). Nên giới hạn ai được assume role này.

### Cross-Account Role Pattern

```
Security Tooling Account (999888777666) cần đọc EC2 inventory trong các accounts

1. Trong mỗi Member Account, tạo role SecurityReadRole:
   Trust: {"AWS": "arn:aws:iam::999888777666:role/SecurityScanner"}
   Permission: ec2:Describe*, s3:GetBucket*, rds:Describe*

2. Trong Security Tooling Account, Lambda/EC2 assume role:
   aws sts assume-role \
     --role-arn "arn:aws:iam::111111111111:role/SecurityReadRole" \
     --role-session-name "SecurityScan"

3. Dùng CloudFormation StackSets để deploy role vào tất cả accounts tự động:
   StackSet template → tạo SecurityReadRole → deploy tới tất cả Org accounts
```

---

## Best Practice & Patterns

### Pattern 1: Security Hub + GuardDuty Centralized

```
Organization
├── Management Account — Chỉ Organizations management
├── Security Tooling Account (Delegated Admin)
│   ├── GuardDuty Administrator → aggregate findings từ mọi account
│   ├── Security Hub Administrator → aggregate + prioritize
│   ├── Config Aggregator → compliance view toàn Org
│   └── SIEM integration (Splunk/QRadar/Elastic)
└── Log Archive Account
    ├── CloudTrail Organization Trail → S3
    ├── Config delivery channel → S3
    └── VPC Flow Logs → S3 (read-only)
```

### Pattern 2: IAM Identity Center Centralized

```
Management Account — Enable IAM Identity Center
    └── Sync với Active Directory hoặc Okta/Azure AD

IAM Identity Center Delegated Admin (IAM Admin Account):
    ├── Manage permission sets
    ├── Assign users/groups → accounts
    └── SSO access cho tất cả accounts trong Organization
    
→ Users đăng nhập 1 nơi, access tất cả accounts với quyền phù hợp
```

### Pattern 3: Backup Centralized

```
Management Account — Enable AWS Backup Trusted Access

Backup Admin Account (Delegated Admin):
├── Tạo backup policies cho toàn Organization
├── Xem backup compliance report
├── Cross-account restore testing
└── Centralized backup vault với vault lock (WORM)

SCP bảo vệ: DenyDeleteBackupVault, DenyModifyBackupPlan
```

### Checklist Thiết Lập Delegated Admin

```
✅ Enable Trusted Access cho service trước
✅ Enable service trong Management Account trước khi register Delegated Admin
✅ Chỉ định một account riêng biệt làm Delegated Admin (không dùng Management Account)
✅ Apply SCP chặn member accounts tắt security services
✅ Dùng StackSets để deploy IAM roles cần thiết vào tất cả accounts
✅ Test auto-enable khi account mới join Organization
✅ Document rõ ràng account nào là Delegated Admin cho service nào
```

---

## Câu Hỏi Phỏng Vấn

### Cơ Bản

**Q: Delegated Administrator khác gì so với chỉ cấp IAM permission cross-account?**
> **Delegated Administrator** là khái niệm của AWS Organizations — cho phép member account **hoạt động với quyền đặc biệt** của một AWS service ở cấp độ Organization (ví dụ: quản lý GuardDuty cho toàn Org). Còn IAM cross-account role chỉ là cơ chế assume role thông thường — account B assume role trong account A để thực hiện actions. Delegated Admin được AWS service recognize đặc biệt, không chỉ là IAM role thông thường.

**Q: Có thể đặt Management Account làm Delegated Administrator không?**
> **Không.** Management Account không thể là Delegated Administrator. Đây là thiết kế có chủ đích — buộc bạn phân tách việc quản lý Organizations (Management Account) và quản lý security services (Security Tooling Account). Nếu Delegated Admin có thể là Management Account, thì sẽ không có lợi ích gì về phân tách trách nhiệm.

**Q: Trusted Access và Delegated Administrator có phải bật cùng lúc không?**
> **Trusted Access phải bật trước** — không có Trusted Access thì service không thể tích hợp với Organizations, không thể có Delegated Admin. Sau khi bật Trusted Access, việc đặt Delegated Admin là tùy chọn (không bắt buộc). Nếu không đặt Delegated Admin, mọi việc vẫn phải thực hiện từ Management Account.

### Nâng Cao

**Q: Khi member account rời Organization, điều gì xảy ra với GuardDuty settings?**
> Account rời Organization sẽ bị **detach khỏi GuardDuty Organization settings** — GuardDuty detector trong account đó vẫn tồn tại nhưng không còn gửi findings về Security Tooling Account. Account trở thành standalone GuardDuty account. Các findings cũ trong Security Tooling Account vẫn giữ nguyên. SCP bảo vệ accounts khỏi rời Organization là best practice.

**Q: Tại sao cần Log Archive Account riêng ngoài Security Tooling Account?**
> **Tách biệt mục đích và quyền truy cập**: Log Archive chỉ cần quyền write logs và read audit — không cần quyền manage security tools. Security Tooling Account cần quyền rộng hơn để manage GuardDuty, Security Hub. Nếu Security Tooling Account bị compromise, attacker không có quyền xóa logs trong Log Archive (nếu được cấu hình S3 Object Lock). Ngoài ra, tách biệt giúp dễ kiểm soát chi phí và compliance — Log Archive account có policy tối giản.

---

## 🔗 Điều Hướng

| Trước | File Này | Tiếp Theo |
|-------|---------|-----------|
| [3-consolidated-billing.md](./3-consolidated-billing.md) | **4-delegated-admin.md** | [5-multi-account-patterns.md](./5-multi-account-patterns.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
