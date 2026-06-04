# Users, Groups, Roles — Ba Loại Định Danh Trong IAM

> Ba loại định danh cốt lõi của IAM đáp ứng ba nhu cầu khác nhau: **User** cho con người/ứng dụng có thông tin xác thực lâu dài, **Group** để quản lý quyền tập trung, **Role** cho các thực thể cần quyền tạm thời.

---

## 1. IAM User (Người Dùng IAM)

### Định Nghĩa

**IAM User** là một định danh có thông tin xác thực (credentials) lâu dài đại diện cho một người hoặc một ứng dụng cụ thể trong tài khoản AWS.

### Đặc Điểm Kỹ Thuật

| Thuộc Tính | Chi Tiết |
|---|---|
| **ARN** | `arn:aws:iam::123456789012:user/alice` |
| **Credentials** | Password (Console), Access Key/Secret (API/CLI) |
| **Số lượng tối đa** | 5.000 users/account (soft limit) |
| **Groups tối đa** | Thuộc tối đa 10 groups |
| **Access keys tối đa** | 2 keys cùng lúc (để rotation không gián đoạn) |
| **Scope** | Trong một tài khoản AWS duy nhất |

### Loại Credentials

```
1. Console Password (Mật Khẩu Console):
   - Đăng nhập vào AWS Management Console
   - Nên bật MFA bắt buộc

2. Access Key ID + Secret Access Key (Khóa Truy Cập):
   - Dùng cho AWS CLI, SDK, trực tiếp gọi API
   - Chỉ hiển thị Secret Key MỘT LẦN khi tạo — lưu ngay
   - Mỗi user tối đa 2 key (để thực hiện rotation)

3. MFA Device (Thiết Bị Xác Thực Đa Yếu Tố):
   - Virtual MFA (Google Authenticator, Authy)
   - Hardware TOTP token
   - FIDO2 security key (YubiKey)
```

### Khi Nào Dùng IAM User

✅ **Nên dùng:**
- Kỹ sư cần đăng nhập vào AWS Console
- CI/CD pipeline cần credentials cố định (nhưng nên prefer OIDC nếu có thể)
- Legacy ứng dụng không hỗ trợ instance profile

❌ **Không nên dùng:**
- Ứng dụng chạy trên EC2, Lambda, ECS — dùng Role thay thế
- Truy cập từ nhiều người — tạo user riêng cho từng người
- Môi trường lớn — dùng IAM Identity Center (SSO) thay thế

### Ví Dụ: Tạo IAM User Với CLI

```bash
# Tạo user
aws iam create-user --user-name alice

# Tạo access key
aws iam create-access-key --user-name alice

# Tạo login profile (cho Console access)
aws iam create-login-profile \
  --user-name alice \
  --password "TempPass123!" \
  --password-reset-required

# Gắn policy
aws iam attach-user-policy \
  --user-name alice \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

---

## 2. IAM Group (Nhóm IAM)

### Định Nghĩa

**IAM Group** là tập hợp các IAM Users. Gắn policy vào group thay vì từng user riêng lẻ để quản lý quyền tập trung và nhất quán.

### Đặc Điểm Kỹ Thuật

| Thuộc Tính | Chi Tiết |
|---|---|
| **ARN** | `arn:aws:iam::123456789012:group/developers` |
| **Số lượng tối đa** | 300 groups/account |
| **User tối đa/group** | Không giới hạn cứng |
| **Groups lồng nhau** | ❌ Không hỗ trợ (no nested groups) |
| **Credentials** | ❌ Không có — groups không có credentials |
| **Assume Role** | ❌ Groups không thể assume role |

### Thiết Kế Cấu Trúc Group

```
Cấu trúc phổ biến theo chức năng:

Administrators
├── Quyền: AdministratorAccess (hoặc subset cụ thể)
└── Members: 2-3 kỹ sư hạ tầng cấp cao

Developers
├── Quyền: Developer-specific policy (EC2, S3, RDS read/write)
└── Members: Toàn bộ đội phát triển

ReadOnly
├── Quyền: ReadOnlyAccess
└── Members: Auditors, business stakeholders

SecurityAuditors
├── Quyền: SecurityAudit (AWS managed policy)
└── Members: Đội bảo mật
```

### Ví Dụ: Thiết Lập Group

```bash
# Tạo group
aws iam create-group --group-name Developers

# Gắn managed policy vào group
aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/PowerUserAccess

# Thêm user vào group
aws iam add-user-to-group \
  --group-name Developers \
  --user-name alice

# Kiểm tra members của group
aws iam get-group --group-name Developers
```

---

## 3. IAM Role (Vai Trò IAM)

### Định Nghĩa

**IAM Role** là một định danh IAM **không có credentials cố định**. Thay vào đó, role được "assume" (đảm nhận) bởi trusted entities và cấp **temporary credentials** (thông tin xác thực tạm thời) thông qua STS (Security Token Service).

### Tại Sao Role Quan Trọng Hơn User

```
Vấn đề với long-term credentials (credentials lâu dài):
✗ Dễ bị lộ qua code, logs, environment variables
✗ Khó rotate mà không gây gián đoạn dịch vụ
✗ Không tự hết hạn — nếu bị lộ, có thể bị dùng mãi

Ưu điểm của temporary credentials (credentials tạm thời):
✓ Tự động hết hạn (mặc định 1 giờ, tối đa 12 giờ)
✓ Không cần lưu trữ hoặc quản lý
✓ Được cấp tự động bởi AWS metadata service trên EC2/ECS/Lambda
✓ Hỗ trợ cross-account access an toàn
```

### Hai Loại Policy Của Role

#### Trust Policy (Chính Sách Tin Tưởng)

Trust Policy xác định **ai được phép assume role** này (the principal):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### Permission Policy (Chính Sách Quyền)

Permission Policy xác định **role này được làm gì** sau khi assumed:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-app-bucket/*"
    }
  ]
}
```

### Các Loại Role Phổ Biến

#### Service Role (Role Dịch Vụ)

Cho phép dịch vụ AWS thực hiện tác vụ thay mặt bạn:

```bash
# Tạo role cho EC2
aws iam create-role \
  --role-name EC2-S3-ReadRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Gắn policy vào role
aws iam attach-role-policy \
  --role-name EC2-S3-ReadRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Tạo instance profile (cần thiết để gắn role vào EC2)
aws iam create-instance-profile \
  --instance-profile-name EC2-S3-ReadRole

aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-S3-ReadRole \
  --role-name EC2-S3-ReadRole
```

**Các service role quan trọng:**

| Dịch Vụ | Trust Principal | Use Case |
|---|---|---|
| EC2 | `ec2.amazonaws.com` | Instance profile — EC2 gọi AWS services |
| Lambda | `lambda.amazonaws.com` | Execution role — Lambda đọc S3, ghi DynamoDB |
| ECS Task | `ecs-tasks.amazonaws.com` | Task role — container truy cập AWS services |
| CodeBuild | `codebuild.amazonaws.com` | Build role — CI/CD pipeline |
| CloudFormation | `cloudformation.amazonaws.com` | Deployment role |

#### Cross-Account Role (Role Liên Tài Khoản)

Cho phép principal từ tài khoản khác truy cập tài nguyên:

```
Kịch bản:
Account A (123456789012) — Production
Account B (987654321098) — DevOps/Tools

Mục tiêu: Engineers trong Account B assume role trong Account A
để quản lý tài nguyên production.
```

**Trong Account A — tạo role:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::987654321098:role/DevOpsEngineer"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

**Từ Account B — assume role:**

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/ProdReadOnly \
  --role-session-name devops-session \
  --duration-seconds 3600
```

#### Federated Role (Role Liên Kết Định Danh)

Cho phép người dùng từ IdP bên ngoài (Okta, Azure AD, Google) assume role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:saml-provider/OktaProvider"
      },
      "Action": "sts:AssumeRoleWithSAML",
      "Condition": {
        "StringEquals": {
          "SAML:aud": "https://signin.aws.amazon.com/saml"
        }
      }
    }
  ]
}
```

### Luồng AssumeRole Hoàn Chỉnh

```
EC2 Instance:
┌─────────────────────────────────────────────────────────┐
│ 1. Application gọi API AWS (ví dụ: s3:GetObject)        │
│                                                         │
│ 2. AWS SDK tự động gọi                                  │
│    http://169.254.169.254/latest/meta-data/             │
│    iam/security-credentials/MyRole                      │
│    → Nhận AccessKeyId, SecretAccessKey, SessionToken    │
│                                                         │
│ 3. SDK dùng temporary credentials để ký request        │
│                                                         │
│ 4. IAM kiểm tra role's permission policy               │
│    → Allow → Thực thi API call                         │
└─────────────────────────────────────────────────────────┘
```

---

## 4. So Sánh User vs Group vs Role

| Tiêu Chí | User | Group | Role |
|---|---|---|---|
| **Credentials** | Lâu dài (long-term) | Không có | Tạm thời (temporary) |
| **Có thể đăng nhập Console** | ✅ | ❌ | Với switch role |
| **Dùng cho EC2/Lambda** | ❌ Không nên | ❌ | ✅ Best practice |
| **Cross-account access** | Phức tạp | ❌ | ✅ Native support |
| **Xoay vòng credentials** | Thủ công | N/A | Tự động |
| **Số lượng giới hạn** | 5.000 | 300 | 1.000 |
| **Best for** | Con người đăng nhập | Nhóm quyền | Services, automation |

---

## 5. Patterns Thực Tế

### Pattern 1: Application On EC2

```
❌ Sai: Lưu access key trong environment variable
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...

✅ Đúng: Dùng instance profile
aws iam attach-role-to-instance-profile --instance-id i-xxx --role EC2AppRole
# AWS SDK tự động lấy credentials từ metadata service
```

### Pattern 2: Developer Access Quản Lý

```
✅ Cấu trúc đề xuất:

IAM Users (1 user / 1 người):
  alice@company.com → IAM User "alice"
  bob@company.com   → IAM User "bob"

IAM Groups (quản lý quyền):
  Group "BackendDevelopers" → Policy: s3:*, dynamodb:*, lambda:*
  Group "FrontendDevelopers" → Policy: s3:GetObject (chỉ static assets)
  Group "DevOpsEngineers" → Policy: ec2:*, ecs:*, cloudformation:*

Assign:
  alice → BackendDevelopers
  bob → FrontendDevelopers
```

### Pattern 3: Lambda Với Least Privilege

```python
# Lambda function chỉ cần đọc DynamoDB
# Không cần và không nên có quyền ghi/xóa

# Trust Policy: lambda.amazonaws.com
# Permission Policy:
{
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "dynamodb:GetItem",
      "dynamodb:Query",
      "dynamodb:Scan"
    ],
    "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/Orders"
  }]
}
# Không có: dynamodb:PutItem, dynamodb:DeleteItem, dynamodb:*
```

---

## 📋 Checklist Thực Hành

- [ ] Tạo IAM user riêng cho mỗi người (không share credentials)
- [ ] Bật MFA cho tất cả IAM users có quyền cao
- [ ] Dùng Groups để gán policy, không gán trực tiếp cho user
- [ ] Không dùng root account cho thao tác hàng ngày
- [ ] EC2/Lambda phải dùng Role, không nhúng access key
- [ ] Xoay vòng access key ít nhất 90 ngày một lần
- [ ] Xóa access key không dùng (inactive > 90 ngày)
- [ ] Dùng IAM Credentials Report để audit định kỳ

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa IAM User và IAM Role?**

> User có long-term credentials (password, access key) gắn với một người/ứng dụng cố định. Role không có credentials cố định — được assume bởi EC2, Lambda, users khác hoặc accounts khác và nhận temporary credentials (1–12 giờ) qua STS. Role là best practice cho mọi workload tự động.

**Q: Tại sao không dùng IAM User cho EC2 application?**

> Long-term access key trong environment variable hoặc code là rủi ro bảo mật nghiêm trọng: dễ bị lộ trong logs, git history, metadata. Instance profile với IAM Role cấp temporary credentials tự động luân chuyển qua metadata service (169.254.169.254), không cần quản lý, không có rủi ro lộ credentials cố định.

**Q: IAM Group có thể contain Group khác không? Tại sao?**

> Không. AWS không hỗ trợ nested groups. Lý do thiết kế: tránh complexity trong đánh giá quyền và vòng lặp phụ thuộc (circular dependencies). Nếu cần phân cấp quyền phức tạp, dùng nhiều groups gán cho cùng user, hoặc dùng Permission Boundaries và Attribute-Based Access Control (ABAC).
