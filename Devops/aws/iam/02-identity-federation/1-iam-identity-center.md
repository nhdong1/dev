# 1 — AWS IAM Identity Center (Trung Tâm Định Danh IAM)

> AWS IAM Identity Center (tên cũ: AWS SSO — Single Sign-On — Đăng Nhập Một Lần) là dịch vụ quản lý truy cập tập trung cho phép người dùng đăng nhập một lần và truy cập nhiều AWS accounts cùng lúc.

---

## 📚 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Kiến Trúc](#kiến-trúc)
3. [Permission Sets — Bộ Quyền Hạn](#permission-sets)
4. [Tích Hợp IdP Bên Ngoài](#tích-hợp-idp-bên-ngoài)
5. [SCIM Provisioning — Đồng Bộ Người Dùng](#scim-provisioning)
6. [AWS Access Portal](#aws-access-portal)
7. [CLI/SDK Access](#clisdk-access)
8. [Thực Hành Tốt Nhất](#thực-hành-tốt-nhất)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan

### IAM Identity Center Là Gì?

IAM Identity Center giải quyết vấn đề quản lý truy cập khi doanh nghiệp có nhiều AWS accounts:

```
Vấn đề:
- 50 developer cần truy cập 10 AWS accounts (Dev, Staging, Prod, Security...)
- Nếu dùng IAM users: 50 × 10 = 500 IAM users, 500 password, 500 access key
- Quản lý nightmare: ai rời công ty phải xóa 10 user, forgot password × 10...

Giải pháp IAM Identity Center:
- Một điểm đăng nhập duy nhất (AWS Access Portal)
- User quản lý tập trung (hoặc sync từ Active Directory / Okta)
- Permission Sets định nghĩa một lần, gán cho nhiều accounts
- Temporary credentials tự động refresh
```

### Vị Trí Trong Hệ Sinh Thái AWS

```
IAM Identity Center
    ├── Yêu cầu: Phải bật AWS Organizations
    ├── Quản lý: Từ management account hoặc delegated admin account
    ├── Người dùng: Nhân viên nội bộ (không phải end-users của ứng dụng)
    └── Output: Temporary AWS credentials (via STS AssumeRole)
```

---

## Kiến Trúc

### Luồng Xác Thực Đầy Đủ

```
1. User truy cập https://<alias>.awsapps.com/start
   │
2. Identity Center kiểm tra Identity Source:
   ├── Identity Center directory (built-in)
   ├── Active Directory (via AWS Directory Service)
   └── External IdP (Okta, Azure AD, Google Workspace via SAML 2.0)
   │
3. User xác thực với IdP (username/password + MFA)
   │
4. IdP gửi SAML Assertion về Identity Center
   │
5. Identity Center tra cứu Permission Sets được gán cho user
   │
6. User chọn Account + Role từ Access Portal
   │
7. Identity Center gọi STS AssumeRole
   │
8. STS trả về Temporary Credentials:
   ├── AccessKeyId
   ├── SecretAccessKey
   └── SessionToken (hết hạn theo session duration)
   │
9. User dùng credentials truy cập AWS resources
```

### Các Thành Phần Chính

```
┌─────────────────────────────────────────────────────────┐
│              IAM Identity Center                         │
│                                                          │
│  ┌──────────────┐    ┌──────────────┐                   │
│  │  Identity    │    │  Permission  │                   │
│  │  Source      │    │  Sets        │                   │
│  │  (Users/     │    │  (Policy     │                   │
│  │  Groups)     │    │  Templates)  │                   │
│  └──────┬───────┘    └──────┬───────┘                   │
│         │                   │                            │
│         └─────────┬─────────┘                           │
│                   │                                      │
│  ┌────────────────▼──────────────────────────────────┐  │
│  │              Account Assignments                   │  │
│  │  User/Group + Permission Set + AWS Account         │  │
│  │  = "Alice có ViewOnly trong Account-Prod"          │  │
│  └────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## Permission Sets — Bộ Quyền Hạn

### Khái Niệm

Permission Set là template định nghĩa quyền hạn — tương đương với IAM Role nhưng được quản lý tập trung qua Identity Center.

Khi gán Permission Set cho user vào một account, Identity Center tự động tạo IAM Role trong account đó.

### Loại Permission Sets

```
1. AWS Managed Permission Sets (Bộ Quyền Quản Lý Bởi AWS):
   - AdministratorAccess — toàn quyền
   - PowerUserAccess — full access trừ IAM
   - ReadOnlyAccess — chỉ đọc
   - Billing — quản lý chi phí
   Ưu điểm: AWS cập nhật tự động khi có dịch vụ mới

2. Customer Managed Permission Sets (Bộ Quyền Tự Định Nghĩa):
   - Tự viết inline policy hoặc gắn managed policies
   - Kiểm soát chi tiết hơn
   - Phù hợp cho: DeveloperAccess, DeployAccess, AuditReadOnly...
```

### Ví Dụ Định Nghĩa Permission Set

```json
// Permission Set: "BackendDeveloperAccess"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "LambdaManage",
      "Effect": "Allow",
      "Action": [
        "lambda:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3ReadWrite",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-app-*",
        "arn:aws:s3:::my-app-*/*"
      ]
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyIAM",
      "Effect": "Deny",
      "Action": "iam:*",
      "Resource": "*"
    }
  ]
}
```

### Account Assignments — Gán Tài Khoản

```
Ví dụ thực tế cho doanh nghiệp:

Nhóm "backend-developers" (50 người):
  ├── Account: dev-account     → Permission Set: BackendDeveloperAccess
  ├── Account: staging-account → Permission Set: BackendDeveloperAccess
  └── Account: prod-account    → Permission Set: ReadOnlyAccess (không được sửa prod!)

Nhóm "devops-team" (5 người):
  ├── Account: dev-account     → Permission Set: AdministratorAccess
  ├── Account: staging-account → Permission Set: AdministratorAccess
  └── Account: prod-account    → Permission Set: AdministratorAccess

Nhóm "security-team" (3 người):
  ├── Account: * (tất cả)      → Permission Set: SecurityAuditAccess
  └── Account: security-account → Permission Set: AdministratorAccess
```

---

## Tích Hợp IdP Bên Ngoài

### Hỗ Trợ Các IdP Phổ Biến

```
Azure Active Directory (Azure AD / Entra ID):
  Protocol: SAML 2.0 + SCIM
  Steps:
  1. Tạo Enterprise Application trong Azure AD
  2. Cấu hình SAML SSO với metadata URL của Identity Center
  3. Bật SCIM provisioning để sync users/groups tự động
  4. Map Azure AD groups sang Identity Center groups

Okta:
  Protocol: SAML 2.0 + SCIM
  Steps:
  1. Thêm "AWS IAM Identity Center" app trong Okta catalog
  2. Cấu hình SAML settings (Issuer URL, ACS URL)
  3. Bật SCIM API để sync users/groups
  4. Assign Okta groups vào app

Google Workspace:
  Protocol: SAML 2.0 (SCIM không native — dùng third-party)
  Steps:
  1. Tạo SAML app trong Google Admin Console
  2. Download Google IdP metadata
  3. Upload metadata vào Identity Center
  4. Sync users thủ công hoặc qua tool như Google Cloud Directory Sync
```

### Cấu Hình SAML Metadata

```xml
<!-- AWS Identity Center cung cấp metadata endpoint như sau -->
<!-- User download về và upload lên IdP (Okta, Azure AD, etc.) -->

Metadata URL format:
https://region.signin.aws/platform/saml/metadata/<instance-id>

Hoặc download file XML và upload thủ công vào IdP.

Các trường quan trọng trong metadata:
- entityID: định danh của AWS SP (Service Provider — Nhà Cung Cấp Dịch Vụ)
- AssertionConsumerService URL: nơi IdP gửi SAML response
- SingleLogoutService URL: endpoint đăng xuất
- Certificate: để IdP verify response từ AWS
```

---

## SCIM Provisioning — Đồng Bộ Người Dùng

### SCIM Là Gì?

SCIM — System for Cross-domain Identity Management (Hệ Thống Quản Lý Định Danh Đa Miền) — là giao thức chuẩn để tự động đồng bộ users và groups từ IdP vào Identity Center.

```
Không có SCIM:
- Admin phải tạo user thủ công trong Identity Center
- Nhân viên mới: 2 bước (tạo trong IdP + tạo trong Identity Center)
- Nhân viên nghỉ: Phải nhớ xóa trong cả 2 hệ thống

Có SCIM:
- User tạo trong Okta/Azure AD → tự động sync sang Identity Center
- User bị vô hiệu hóa trong IdP → tự động revoke access trong Identity Center
- Group thay đổi → Permission Sets tự động cập nhật
```

### Cấu Hình SCIM Endpoint

```
Trong Identity Center Settings → Identity Source → External IdP:

SCIM endpoint: https://scim.us-east-1.amazonaws.com/.../<instance-id>/scim/v2
Access token: Generate từ Identity Center console (mang đến IdP)

IdP (Okta/Azure AD) dùng endpoint + token này để:
- POST /Users — tạo user mới
- PUT /Users/{id} — cập nhật user
- PATCH /Users/{id} — disable user
- POST /Groups — tạo group
- PATCH /Groups/{id} — thêm/xóa thành viên
```

---

## AWS Access Portal

### Giao Diện Người Dùng

```
URL: https://<alias>.awsapps.com/start
(Alias tùy chỉnh, ví dụ: https://mycompany.awsapps.com/start)

Sau khi đăng nhập, người dùng thấy:
┌─────────────────────────────────────────┐
│ AWS Access Portal — MyCompany           │
│                                         │
│ AWS Accounts (3)                        │
│ ┌─────────────────────────────────────┐ │
│ │ dev-account (123456789012)          │ │
│ │  → BackendDeveloperAccess  [Console]│ │
│ │  → BackendDeveloperAccess  [  CLI ] │ │
│ ├─────────────────────────────────────┤ │
│ │ staging-account (234567890123)      │ │
│ │  → BackendDeveloperAccess  [Console]│ │
│ ├─────────────────────────────────────┤ │
│ │ prod-account (345678901234)         │ │
│ │  → ReadOnlyAccess          [Console]│ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

---

## CLI/SDK Access

### Cấu Hình AWS CLI Với Identity Center

```bash
# Bước 1: Cấu hình SSO profile
aws configure sso

# Output interactive:
# SSO session name: mycompany-sso
# SSO start URL: https://mycompany.awsapps.com/start
# SSO region: us-east-1
# SSO registration scopes: sso:account:access

# Bước 2: Browser mở để xác thực
# Sau khi xác thực, chọn account và role

# Bước 3: Profile được lưu vào ~/.aws/config
# [profile mycompany-dev]
# sso_session = mycompany-sso
# sso_account_id = 123456789012
# sso_role_name = BackendDeveloperAccess
# region = us-east-1

# Sử dụng hàng ngày:
aws sso login --profile mycompany-dev
aws s3 ls --profile mycompany-dev

# Refresh token khi hết hạn:
aws sso login --profile mycompany-dev
```

### Tích Hợp Với Terraform

```hcl
# ~/.aws/config profile đã setup, Terraform dùng luôn:

provider "aws" {
  profile = "mycompany-dev"
  region  = "us-east-1"
}

# Hoặc dùng environment variable:
# export AWS_PROFILE=mycompany-dev
# terraform plan
```

### Tích Hợp Với SDK

```python
import boto3

# SDK tự động dùng profile từ ~/.aws/config
session = boto3.Session(profile_name='mycompany-dev')
s3 = session.client('s3')

# Hoặc set environment variable AWS_PROFILE=mycompany-dev
# rồi dùng boto3 bình thường
```

---

## Thực Hành Tốt Nhất

### 1. Cấu Trúc Permission Sets

```
Nguyên tắc đặt tên rõ ràng:
✅ BackendDeveloperAccess
✅ InfraReadOnly
✅ SecurityAuditFullAccess
✅ BillingViewOnly

❌ DevRole (không rõ môi trường)
❌ FullAccess (quá rộng, không cụ thể)
❌ TestPermission (không mô tả được dùng cho gì)

Phân tầng quyền:
ReadOnly < Developer < PowerUser < Administrator
```

### 2. Nguyên Tắc Least Privilege (Đặc Quyền Tối Thiểu)

```
- Developer KHÔNG có quyền iam:* (trừ khi cần thiết cho công việc)
- Developer KHÔNG có AdministratorAccess trên prod
- Tạo Permission Set riêng cho môi trường prod (restricted hơn)
- Dùng Permission Boundary trong Permission Set để giới hạn thêm
```

### 3. MFA (Multi-Factor Authentication — Xác Thực Đa Yếu Tố)

```
Cấu hình trong Identity Center Settings → Authentication:
- Bật MFA Required cho mọi user
- Cho phép: TOTP authenticator apps (Google Authenticator, Authy)
- Cho phép: FIDO2/WebAuthn (hardware security keys)
- Không cho phép: SMS (dễ bị SIM swap attack)

Context-aware MFA: Bật tính năng này để chỉ yêu cầu MFA
khi đăng nhập từ địa điểm/thiết bị lạ
```

### 4. Session Duration (Thời Hạn Phiên)

```
Permission Set settings:
- Developer role: 8 giờ (một ngày làm việc)
- ReadOnly role: 12 giờ
- Admin role: 1–4 giờ (sensitive, cần re-authenticate thường xuyên)
- Break-glass emergency role: 1 giờ (cho trường hợp khẩn cấp)

Lý do: Shorter session = smaller blast radius nếu credentials bị lộ
```

### 5. Audit Và Monitoring

```
CloudTrail events quan trọng cần monitor:
- sso:Authenticate — đăng nhập thành công/thất bại
- sso:ListAccountRoles — user xem danh sách roles
- sso:GetRoleCredentials — user lấy credentials

Tạo CloudWatch Alarm cho:
- Failed login attempts > 5 lần trong 10 phút
- Login từ IP bất thường (ngoài office range)
- Access to prod account ngoài giờ làm việc
```

---

## Câu Hỏi Phỏng Vấn

### Q1: IAM Identity Center khác gì với IAM user truyền thống?

```
IAM User:
- Permanent identity trong một account
- Long-term credentials (access key không hết hạn)
- Phải quản lý thủ công trong mỗi account

IAM Identity Center:
- Temporary credentials (hết hạn tự động)
- Quản lý tập trung, gán vào nhiều accounts
- Tích hợp với corporate IdP (AD, Okta, Azure AD)
- Audit tập trung tại một nơi

Kết luận: Identity Center là best practice cho doanh nghiệp
với nhiều accounts và nhiều nhân viên.
```

### Q2: Giải thích Permission Set và cách nó tạo ra IAM Role

```
Permission Set là template trong Identity Center.
Khi bạn gán Permission Set cho một user/group vào một account:
1. Identity Center tự động tạo IAM Role trong account đó
2. Role name format: AWSReservedSSO_<PermissionSetName>_<unique-id>
3. Role có Trust Policy cho phép Identity Center AssumeRole
4. Role có các policies từ Permission Set definition

Nếu bạn xóa assignment, Identity Center tự động xóa Role đó.
Nếu bạn update Permission Set, Identity Center update tất cả Roles liên quan.
```

### Q3: Nhân viên nghỉ việc — làm thế nào để revoke access ngay lập tức?

```
Với SCIM đã cấu hình:
1. Vô hiệu hóa user trong Okta/Azure AD
2. SCIM tự động sync sang Identity Center (trong vài phút)
3. User không thể đăng nhập Access Portal
4. Active sessions: Vẫn có thể dùng credentials còn hạn!

Để revoke ngay lập tức active sessions:
Identity Center Console → Users → chọn user → Delete active sessions

Lý do: STS temporary credentials không thể revoke trực tiếp,
nhưng Identity Center có thể invalidate session token
khiến user không lấy được credentials mới.
```

### Q4: Permission Set có thể dùng Permission Boundary không?

```
Có. Trong Permission Set definition:
- Gán Managed Policies (AWS hoặc Customer Managed)
- Gán Inline Policy
- Gán Permission Boundary (Managed Policy làm ranh giới)

Dùng Permission Boundary trong Permission Set khi:
- Developer được phép tạo IAM roles, nhưng chỉ trong giới hạn nhất định
- Ngăn privilege escalation (leo thang đặc quyền) kể cả khi có IAM access
```

---

## 🔗 Điều Hướng

- **Trước:** [README.md](README.md) — Tổng quan Identity Federation
- **Tiếp theo:** [2-saml-federation.md](2-saml-federation.md) — SAML 2.0 chi tiết
- **Liên quan:** [5-cross-account-roles.md](5-cross-account-roles.md) — Cross-account access patterns

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
