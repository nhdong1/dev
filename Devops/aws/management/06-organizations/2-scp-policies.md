# SCP — Service Control Policy (Chính Sách Kiểm Soát Dịch Vụ)

> **SCP — Service Control Policy** là loại policy trong AWS Organizations dùng để giới hạn **tối đa quyền** (maximum permissions) có thể được sử dụng trong một account, OU hoặc toàn bộ Organization. SCP không cấp quyền — nó chỉ thu hẹp quyền đã có từ IAM. Đây là công cụ bảo mật quan trọng nhất trong chiến lược đa tài khoản.

---

## 📚 Mục Lục

1. [Cơ Chế Hoạt Động](#cơ-chế-hoạt-động)
2. [Hai Chiến Lược SCP](#hai-chiến-lược-scp)
3. [Kế Thừa SCP Theo Cây Tổ Chức](#kế-thừa-scp)
4. [Cú Pháp SCP](#cú-pháp-scp)
5. [SCP Phổ Biến Trong Thực Tế](#scp-phổ-biến-trong-thực-tế)
6. [Thứ Tự Đánh Giá Policy](#thứ-tự-đánh-giá-policy)
7. [Quản Lý SCP Bằng CLI](#quản-lý-scp-bằng-cli)
8. [Kiểm Tra Hiệu Lực SCP](#kiểm-tra-hiệu-lực-scp)
9. [Giới Hạn SCP](#giới-hạn-scp)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Cơ Chế Hoạt Động

### SCP Là Guardrail, Không Phải Permission

```
Không có SCP:
┌─────────────────────────────────┐
│       Tất cả quyền AWS          │  ← Mọi action đều có thể được IAM cấp
└─────────────────────────────────┘

Có SCP:
┌─────────────────────────────────┐
│       Tất cả quyền AWS          │
│  ┌──────────────────────────┐   │
│  │  Quyền sau khi SCP lọc  │   │  ← Chỉ phần này IAM mới có thể cấp
│  └──────────────────────────┘   │
└─────────────────────────────────┘
```

### Quy Tắc Cơ Bản

```
1. SCP KHÔNG cấp quyền — chỉ giới hạn quyền
2. SCP áp dụng cho TẤT CẢ identities trong account:
   - IAM Users
   - IAM Roles
   - Root user của Member Account
   ❌ NGOẠI LỆ: Management Account không bị SCP ảnh hưởng

3. Để thực hiện một action:
   - SCP phải ALLOW (hoặc không có Deny)
   - IAM Policy phải ALLOW
   → Cả hai điều kiện phải đúng cùng lúc (AND logic)

4. Explicit Deny trong SCP > bất kỳ Allow nào trong IAM
```

### Biểu Đồ Đánh Giá Quyền

```
Request từ IAM Entity
         │
         ▼
   SCP Deny ─── Yes ──→ DENY (ngay lập tức)
   (bất kỳ cấp nào)?
         │ No
         ▼
   SCP Allow ─── No ───→ DENY (implicit deny)
   (tất cả cấp)?
         │ Yes
         ▼
   IAM Policy ─── No ──→ DENY
   Allow?
         │ Yes
         ▼
       ALLOW ✅
```

---

## Hai Chiến Lược SCP

### Chiến Lược 1: Deny List (Danh Sách Từ Chối)

**Nguyên tắc:** Mặc định cho phép mọi thứ, sau đó liệt kê những gì bị cấm.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAll",
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    },
    {
      "Sid": "DenySpecificActions",
      "Effect": "Deny",
      "Action": [
        "organizations:LeaveOrganization",
        "account:CloseAccount"
      ],
      "Resource": "*"
    }
  ]
}
```

**AWS FullAWSAccess SCP** là SCP mặc định do AWS tạo — chứa `Allow *` và được gắn tự động vào Root. Đây là basis của Deny List strategy.

**Ưu điểm:** Linh hoạt, ít cần cập nhật khi AWS ra service mới.
**Nhược điểm:** Rủi ro quên deny những gì cần chặn — không phù hợp môi trường high-security.

### Chiến Lược 2: Allow List (Danh Sách Cho Phép)

**Nguyên tắc:** Mặc định chặn mọi thứ, sau đó liệt kê chỉ những gì được phép.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOnlyApprovedServices",
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "s3:*",
        "rds:*",
        "lambda:*",
        "cloudwatch:*",
        "logs:*",
        "iam:*",
        "sts:AssumeRole"
      ],
      "Resource": "*"
    }
  ]
}
```

**Yêu cầu:** Phải xóa `FullAWSAccess` SCP mặc định trước khi áp dụng.

**Ưu điểm:** Tường minh hơn, an toàn hơn cho môi trường regulatory.
**Nhược điểm:** Phải cập nhật mỗi khi cần dùng AWS service mới — operational overhead cao.

### So Sánh Hai Chiến Lược

| Tiêu Chí | Deny List | Allow List |
|---------|-----------|------------|
| **Mặc định** | Cho phép mọi thứ | Chặn mọi thứ |
| **Cách làm** | Explicit Deny điều xấu | Explicit Allow điều tốt |
| **Operational overhead** | Thấp | Cao (update khi dùng service mới) |
| **Bảo mật** | Trung bình | Cao |
| **Phù hợp với** | Phần lớn account thông thường | PCI-DSS, HIPAA account |
| **AWS khuyến nghị** | ✅ Phổ biến hơn | Môi trường regulated |

---

## Kế Thừa SCP

### Cơ Chế Kế Thừa (Inheritance)

SCP áp dụng theo nguyên tắc **intersection** (giao) — một action chỉ được phép nếu được Allow ở **mọi cấp** trong cây tổ chức.

```
Root (FullAWSAccess: Allow *)
│
├── OU: Workloads (SCP: Deny eu-west-1 region)
│   │
│   └── OU: Production (SCP: Deny root user access)
│       │
│       └── Account: app-prod
│           IAM: Cho phép S3 full access
│
→ Effective permissions của IAM user trong app-prod:
  ✅ S3 access (allowed ở mọi cấp, IAM cũng allow)
  ❌ Tạo resource ở eu-west-1 (Deny ở OU Workloads)
  ❌ Root user bị chặn (Deny ở OU Production)
```

### Ví Dụ Kế Thừa Chi Tiết

```
Root SCP:   Allow *
            Deny: LeaveOrganization
                │
          ┌─────┴──────┐
          │            │
   OU: Security    OU: Workloads
   SCP: Deny *     SCP: DenyNonApprovedRegions
   Except Security │
   Services        │
                 ┌──┴───┐
                 │      │
              OU: Prod  OU: NonProd
              SCP: Deny SCP: Deny
              DeleteBucket  DeleteRole
              WithoutMFA    (không có)
                 │
              Account: app-prod
              IAM: Allow s3:DeleteBucket

→ s3:DeleteBucket trong app-prod:
  Root SCP: Allow (FullAWSAccess)
  OU Workloads SCP: Allow (không deny s3:DeleteBucket)
  OU Prod SCP: DENY DeleteBucket without MFA → DENIED ❌
```

---

## Cú Pháp SCP

### Cấu Trúc Cơ Bản

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "StatementId",           // Tùy chọn — tên định danh
      "Effect": "Allow" | "Deny",
      "Action": [                     // Hoặc "*" cho tất cả
        "service:action",
        "service:*"
      ],
      "Resource": "*",               // SCP chỉ hỗ trợ "*"
      "Condition": {                 // Tùy chọn
        "StringEquals": {
          "aws:RequestedRegion": "ap-southeast-1"
        }
      }
    }
  ]
}
```

**Giới hạn quan trọng:** SCP chỉ hỗ trợ `"Resource": "*"` — không thể giới hạn theo ARN cụ thể.

### Các Condition Keys Phổ Biến Trong SCP

```json
"Condition": {
  // Giới hạn region
  "StringNotEquals": {
    "aws:RequestedRegion": ["ap-southeast-1", "us-east-1"]
  },
  
  // Chỉ cho phép từ IP cụ thể
  "NotIpAddress": {
    "aws:SourceIp": ["203.0.113.0/24"]
  },
  
  // Yêu cầu MFA
  "BoolIfExists": {
    "aws:MultiFactorAuthPresent": "false"
  },
  
  // Chỉ cho phép qua VPC Endpoint
  "StringNotEquals": {
    "aws:sourceVpc": "vpc-xxxxxxxx"
  },
  
  // Yêu cầu tag khi tạo resource
  "Null": {
    "aws:RequestTag/Environment": "true"
  }
}
```

---

## SCP Phổ Biến Trong Thực Tế

### 1. Deny Leave Organization (Bắt Buộc Cho Mọi Org)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyLeaveOrganization",
    "Effect": "Deny",
    "Action": [
      "organizations:LeaveOrganization"
    ],
    "Resource": "*"
  }]
}
```

### 2. Restrict Regions (Giới Hạn Region)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyNonApprovedRegions",
    "Effect": "Deny",
    "NotAction": [
      "iam:*",
      "organizations:*",
      "support:*",
      "sts:*",
      "cloudfront:*",
      "route53:*",
      "waf:*",
      "budgets:*",
      "billing:*"
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
  }]
}
```

**Lưu ý:** Dùng `NotAction` để exclude global services (IAM, Route53, CloudFront...) vì chúng không có region concept.

### 3. Require MFA (Yêu Cầu Xác Thực Đa Yếu Tố)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyWithoutMFA",
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
  }]
}
```

### 4. Protect Security Resources (Bảo Vệ Tài Nguyên Bảo Mật)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDisableCloudTrail",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "cloudtrail:UpdateTrail"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyDisableGuardDuty",
      "Effect": "Deny",
      "Action": [
        "guardduty:DeleteDetector",
        "guardduty:DisassociateFromMasterAccount",
        "guardduty:UpdateDetector"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyModifySecurityAccount",
      "Effect": "Deny",
      "Action": [
        "config:StopConfigurationRecorder",
        "config:DeleteConfigurationRecorder",
        "config:DeleteDeliveryChannel"
      ],
      "Resource": "*"
    }
  ]
}
```

### 5. Deny Root User Actions (Chặn Root User)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyRootUser",
    "Effect": "Deny",
    "Action": "*",
    "Resource": "*",
    "Condition": {
      "StringLike": {
        "aws:PrincipalArn": "arn:aws:iam::*:root"
      }
    }
  }]
}
```

### 6. Require Tags Khi Tạo EC2 (Tag Enforcement)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyEC2WithoutRequiredTags",
    "Effect": "Deny",
    "Action": [
      "ec2:RunInstances"
    ],
    "Resource": "arn:aws:ec2:*:*:instance/*",
    "Condition": {
      "Null": {
        "aws:RequestTag/Environment": "true",
        "aws:RequestTag/CostCenter": "true"
      }
    }
  }]
}
```

### 7. Deny Purchase Reserved Instances (Chặn Mua RI Từ NonProd)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyPurchaseReservedInstances",
    "Effect": "Deny",
    "Action": [
      "ec2:PurchaseReservedInstancesOffering",
      "ec2:ModifyReservedInstances",
      "rds:PurchaseReservedDBInstancesOffering"
    ],
    "Resource": "*"
  }]
}
```

---

## Thứ Tự Đánh Giá Policy

### Evaluation Logic Đầy Đủ

```
1. Explicit DENY trong SCP? → DENY ❌
2. SCP ALLOW tất cả cấp? → Nếu không → DENY ❌
3. Explicit DENY trong IAM Permission Boundary? → DENY ❌
4. Permission Boundary ALLOW? → Nếu không → DENY ❌
5. Explicit DENY trong Session Policy (AssumeRole)? → DENY ❌
6. Session Policy ALLOW? → Nếu không → DENY ❌ (nếu có session policy)
7. Explicit DENY trong IAM Policy? → DENY ❌
8. IAM Policy ALLOW? → ALLOW ✅
   (Nếu không có explicit Allow → DENY ❌)
```

### Ví Dụ Thực Tế

```
Scenario: IAM User cố gắng tạo S3 bucket ở eu-west-2

SCP (gắn OU): DenyNonApprovedRegions (allow chỉ ap-southeast-1, us-east-1)
IAM Policy:   Allow s3:CreateBucket *

Kết quả: DENY ❌
→ SCP deny region eu-west-2 → bất kể IAM có Allow thì cũng bị chặn

Scenario: IAM User cố gắng tạo S3 bucket ở ap-southeast-1

SCP: DenyNonApprovedRegions (không deny ap-southeast-1)
IAM Policy: Allow s3:CreateBucket *

Kết quả: ALLOW ✅
→ SCP không deny, IAM allow → request thành công
```

---

## Quản Lý SCP Bằng CLI

```bash
# Tạo SCP mới
aws organizations create-policy \
  --name "DenyLeaveOrganization" \
  --description "Prevent member accounts from leaving the organization" \
  --type SERVICE_CONTROL_POLICY \
  --content file://deny-leave-org.json

# Liệt kê tất cả SCP
aws organizations list-policies \
  --filter SERVICE_CONTROL_POLICY

# Xem nội dung SCP
aws organizations describe-policy \
  --policy-id p-xxxxxxxxxxxx

# Gắn SCP vào OU
aws organizations attach-policy \
  --policy-id p-xxxxxxxxxxxx \
  --target-id ou-xxxx-aaaaaaaa

# Gắn SCP vào Account
aws organizations attach-policy \
  --policy-id p-xxxxxxxxxxxx \
  --target-id 123456789012

# Gắn SCP vào Root (áp dụng toàn Organization)
aws organizations attach-policy \
  --policy-id p-xxxxxxxxxxxx \
  --target-id r-xxxx

# Xem SCP đang gắn vào một target
aws organizations list-policies-for-target \
  --target-id ou-xxxx-aaaaaaaa \
  --filter SERVICE_CONTROL_POLICY

# Gỡ SCP
aws organizations detach-policy \
  --policy-id p-xxxxxxxxxxxx \
  --target-id ou-xxxx-aaaaaaaa

# Xóa SCP (phải gỡ khỏi tất cả targets trước)
aws organizations delete-policy \
  --policy-id p-xxxxxxxxxxxx

# Cập nhật nội dung SCP
aws organizations update-policy \
  --policy-id p-xxxxxxxxxxxx \
  --content file://updated-scp.json
```

---

## Kiểm Tra Hiệu Lực SCP

### Dùng IAM Policy Simulator

```bash
# Kiểm tra permission effective trong account
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/test-user \
  --action-names s3:CreateBucket \
  --resource-arns "*" \
  --context-entries '[
    {
      "ContextKeyName": "aws:RequestedRegion",
      "ContextKeyValues": ["eu-west-1"],
      "ContextKeyType": "string"
    }
  ]'
```

### Access Analyzer Để Validate SCP

```bash
# Tạo analyzer ở Organization level
aws accessanalyzer create-analyzer \
  --analyzer-name org-analyzer \
  --type ORGANIZATION

# Validate policy document
aws accessanalyzer validate-policy \
  --policy-document file://my-scp.json \
  --policy-type SERVICE_CONTROL_POLICY
```

### Effective Permissions Report

```bash
# Xem permissions hiệu lực của một entity
# (Bao gồm ảnh hưởng của SCP)
aws iam get-account-authorization-details \
  --filter User \
  --query 'UserDetailList[?UserName==`test-user`]'
```

---

## Giới Hạn SCP

| Giới Hạn | Giá Trị |
|---------|---------|
| SCP tối đa mỗi Organization | 300 |
| SCP tối đa gắn vào 1 target | 5 |
| Kích thước tối đa mỗi SCP | 5.120 ký tự |
| **Resource field** | Chỉ `"*"` (không hỗ trợ ARN cụ thể) |
| **Principal field** | Không hỗ trợ (SCP áp dụng cho mọi principal) |
| **NotPrincipal field** | Không hỗ trợ |

### Những Gì SCP Không Thể Làm

```
❌ SCP không thể:
   - Cấp quyền (chỉ giới hạn quyền)
   - Áp dụng cho Management Account
   - Chặn AWS service-linked roles
   - Giới hạn theo ARN cụ thể của resource
   - Áp dụng cho account ngoài Organization
   - Override explicit allow trong Resource Policy (S3 bucket policy...)
     → Khi resource policy grant cross-account access,
       SCP vẫn áp dụng cho principal trong Organization
```

---

## Câu Hỏi Phỏng Vấn

### Cơ Bản

**Q: SCP "Allow *" có nghĩa là tài khoản có thể làm mọi thứ không?**
> **Không.** SCP `Allow *` chỉ có nghĩa là SCP không giới hạn quyền — nhưng IAM Policy vẫn phải cấp quyền tường minh. SCP và IAM là **AND** — cả hai phải Allow thì mới thực hiện được. `FullAWSAccess` SCP (Allow *) chỉ là không có constraint từ phía SCP.

**Q: Nếu không gắn SCP nào vào OU, account trong OU đó có bị ảnh hưởng không?**
> Phụ thuộc vào chiến lược. Nếu dùng **Deny List**: Root gắn `FullAWSAccess` (Allow *), OU không có SCP → account không bị giới hạn thêm gì — hoạt động bình thường. Nếu Root không có Allow SCP → account bị từ chối mọi thứ (implicit deny).

**Q: Có thể dùng SCP để cấp quyền s3:GetObject cho một bucket cụ thể không?**
> **Không.** SCP không hỗ trợ ARN cụ thể trong `Resource` (chỉ `"*"`). SCP cũng không cấp quyền — chỉ giới hạn. Để cấp quyền truy cập bucket cụ thể, dùng IAM Policy hoặc S3 Bucket Policy.

### Nâng Cao

**Q: Tại sao dùng `NotAction` thay vì `Action` trong SCP giới hạn region?**
> Vì nhiều AWS services là **global** (IAM, CloudFront, Route53, STS...) — chúng không có khái niệm region. Nếu dùng `Action: "*"` với Condition region, các global service bị block luôn dù không liên quan đến region. `NotAction` cho phép exclude các global services khỏi region restriction, chỉ restrict regional services.

**Q: SCP có ảnh hưởng đến Service-Linked Roles không?**
> **Không.** AWS Service-Linked Roles (SLR) được AWS tạo và quản lý — SCP không áp dụng cho các actions được thực hiện bởi AWS services qua SLR khi chúng hoạt động thay mặt bạn. Tuy nhiên, SCP có thể chặn việc **tạo** SLR (iam:CreateServiceLinkedRole) nếu không cẩn thận.

**Q: Làm thế nào test SCP trước khi deploy vào production?**
> 1. Dùng **IAM Policy Simulator** với policy document SCP để test scenarios cụ thể
> 2. Dùng **Access Analyzer validate-policy** để kiểm tra cú pháp và logic
> 3. Deploy vào **Sandbox OU** trước — test với account sandbox thực tế
> 4. Dùng **AWS IAM Access Advisor** để thấy services nào đang được dùng trong account — tránh block accidental
> 5. Apply từng SCP một, không apply nhiều SCP cùng lúc

---

## 🔗 Điều Hướng

| Trước | File Này | Tiếp Theo |
|-------|---------|-----------|
| [1-account-structure.md](./1-account-structure.md) | **2-scp-policies.md** | [3-consolidated-billing.md](./3-consolidated-billing.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
