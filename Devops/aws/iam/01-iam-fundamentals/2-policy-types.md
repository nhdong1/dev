# Policy Types — Các Loại Chính Sách IAM

> AWS IAM có **6 loại policy** phối hợp với nhau để kiểm soát quyền truy cập. Hiểu rõ từng loại và thứ tự đánh giá của chúng là nền tảng để thiết kế kiến trúc bảo mật đúng đắn.

---

## Tổng Quan 6 Loại Policy

```
┌──────────────────────────────────────────────────────────┐
│  1. Identity-based Policies  — gắn với user/group/role   │
│  2. Resource-based Policies  — gắn với tài nguyên        │
│  3. SCPs (Service Control Policies) — giới hạn ở Org     │
│  4. Permission Boundaries    — giới hạn quyền tối đa     │
│  5. Session Policies         — giới hạn trong phiên      │
│  6. ACLs (Access Control Lists) — cũ, cho S3/VPC         │
└──────────────────────────────────────────────────────────┘
```

---

## 1. Identity-based Policies (Chính Sách Dựa Trên Định Danh)

### Định Nghĩa

Policy gắn trực tiếp với **IAM User, Group, hoặc Role** — xác định identity đó được phép làm gì.

### Hai Loại Con

#### Managed Policies (Chính Sách Được Quản Lý)

Tài liệu JSON độc lập, có thể gắn vào nhiều identities:

| Loại | Tạo Bởi | Ví Dụ | Khi Nào Dùng |
|---|---|---|---|
| **AWS Managed** | AWS | `ReadOnlyAccess`, `AdministratorAccess` | Quyền phổ biến, ít thay đổi |
| **Customer Managed** | Bạn | `MyApp-Lambda-Policy` | Quyền tùy chỉnh cho app |

```bash
# Tạo customer managed policy
aws iam create-policy \
  --policy-name S3-AppBucket-ReadWrite \
  --policy-document file://s3-policy.json

# Gắn vào role
aws iam attach-role-policy \
  --role-name MyAppRole \
  --policy-arn arn:aws:iam::123456789012:policy/S3-AppBucket-ReadWrite
```

#### Inline Policies (Chính Sách Nhúng)

Policy được nhúng trực tiếp vào một identity duy nhất — tồn tại cùng vòng đời với identity đó:

```bash
# Gắn inline policy trực tiếp vào role
aws iam put-role-policy \
  --role-name MyRole \
  --policy-name InlinePolicy \
  --policy-document file://inline-policy.json
```

**Khi nào dùng Inline Policy:**
- Quyền đặc biệt chỉ dành riêng cho một identity cụ thể
- Muốn đảm bảo policy bị xóa khi identity bị xóa
- Thường tránh dùng — khó quản lý ở quy mô lớn

### Cấu Trúc Policy JSON

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadOnBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-app-bucket",
        "arn:aws:s3:::my-app-bucket/*"
      ]
    },
    {
      "Sid": "DenyDeleteObjects",
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::my-app-bucket/*"
    }
  ]
}
```

| Trường | Bắt Buộc | Mô Tả |
|---|---|---|
| `Version` | Khuyến nghị | Luôn dùng `"2012-10-17"` |
| `Statement` | ✅ | Mảng các statements |
| `Sid` | Tùy chọn | Statement ID — identifier mô tả |
| `Effect` | ✅ | `"Allow"` hoặc `"Deny"` |
| `Action` | ✅ | API actions (hoặc `"*"` cho tất cả) |
| `Resource` | ✅ | ARN của tài nguyên |
| `Condition` | Tùy chọn | Điều kiện bổ sung |
| `Principal` | Resource policies | Ai được áp dụng (chỉ resource-based) |

---

## 2. Resource-based Policies (Chính Sách Dựa Trên Tài Nguyên)

### Định Nghĩa

Policy gắn trực tiếp vào **tài nguyên AWS** (S3 bucket, KMS key, SQS queue...) — xác định ai được phép truy cập tài nguyên đó và làm gì.

### Khác Biệt Với Identity-based Policy

```
Identity-based:  "User Alice được làm gì?"
Resource-based:  "Bucket này cho phép ai truy cập?"
```

### Các Tài Nguyên Hỗ Trợ Resource-based Policy

| Dịch Vụ | Loại Policy |
|---|---|
| **S3** | Bucket Policy |
| **KMS** | Key Policy |
| **SQS** | Queue Policy |
| **SNS** | Topic Policy |
| **Lambda** | Function Policy |
| **ECR** | Repository Policy |
| **Secrets Manager** | Resource Policy |
| **IAM Role** | Trust Policy |

### Ví Dụ: S3 Bucket Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCrossAccountRead",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::987654321098:role/DataAnalyticsRole"
      },
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::data-lake-bucket",
        "arn:aws:s3:::data-lake-bucket/*"
      ]
    },
    {
      "Sid": "DenyNonHTTPS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::data-lake-bucket",
        "arn:aws:s3:::data-lake-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

### Cross-Account Access Logic

```
Trong cùng account:
  Identity-based policy Allow → ĐƯỢC PHÉP
  (resource-based policy là bonus, không bắt buộc)

Cross-account:
  Identity-based policy Allow (Account B) 
  AND Resource-based policy Allow (Account A)
  → CẢ HAI ĐỀU CẦN để được phép
```

---

## 3. SCPs — Service Control Policies (Chính Sách Kiểm Soát Dịch Vụ)

### Định Nghĩa

SCPs là chính sách ở cấp **AWS Organizations** — thiết lập **trần quyền tối đa** (maximum permissions ceiling) cho tất cả identities trong Organization/OU/Account. SCPs không cấp quyền — chúng chỉ giới hạn quyền tối đa có thể có.

### Đặc Điểm Quan Trọng

```
SCP KHÔNG áp dụng cho:
✗ Root user của management account (payer account)
✗ AWS service-linked roles
✗ AWS Organizations service actions

SCP ÁP DỤNG cho:
✓ Tất cả IAM users trong member accounts
✓ Tất cả IAM roles trong member accounts
✓ Root user của MEMBER accounts (không phải management account)
```

### Cấu Trúc SCP

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRegionsOutsideApac",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "ap-southeast-1",
            "ap-northeast-1",
            "us-east-1"
          ]
        }
      }
    }
  ]
}
```

### Chiến Lược Deny List vs Allow List

#### Deny List (Danh Sách Từ Chối — Phổ Biến Hơn)

```
Mặc định: SCP "FullAWSAccess" cho phép tất cả
Thêm: Explicit Deny cho những gì bị cấm

Ưu điểm: Ít xáo trộn, mới thêm dịch vụ tự động được phép
Nhược điểm: Ít chặt chẽ hơn
```

#### Allow List (Danh Sách Cho Phép — Chặt Chẽ Hơn)

```
Xóa: SCP "FullAWSAccess"
Thêm: Explicit Allow chỉ cho dịch vụ được phê duyệt

Ưu điểm: Kiểm soát chặt, chỉ whitelist services
Nhược điểm: Cần cập nhật thường xuyên khi dùng dịch vụ mới
```

### Ví Dụ SCPs Thực Tế

```json
// Ngăn tắt CloudTrail
{
  "Sid": "DenyCloudTrailDisable",
  "Effect": "Deny",
  "Action": [
    "cloudtrail:StopLogging",
    "cloudtrail:DeleteTrail",
    "cloudtrail:UpdateTrail"
  ],
  "Resource": "*"
}
```

```json
// Bắt buộc mã hóa S3
{
  "Sid": "DenyUnencryptedS3",
  "Effect": "Deny",
  "Action": "s3:PutObject",
  "Resource": "*",
  "Condition": {
    "StringNotEqualsIfExists": {
      "s3:x-amz-server-side-encryption": ["aws:kms", "AES256"]
    },
    "Null": {
      "s3:x-amz-server-side-encryption": "true"
    }
  }
}
```

---

## 4. Permission Boundaries (Ranh Giới Quyền Hạn)

### Định Nghĩa

Permission Boundary là managed policy dùng để **giới hạn quyền tối đa** mà một IAM user hoặc role có thể có — ngay cả khi identity-based policy cấp quyền rộng hơn.

```
Quyền thực tế = Identity Policy ∩ Permission Boundary

Ví dụ:
  Identity Policy: Allow s3:*, ec2:*, iam:*
  Permission Boundary: Allow s3:*, ec2:*
  → Quyền thực tế: Allow s3:*, ec2:*
  → iam:* bị loại (nằm ngoài boundary)
```

### Use Case Chính: Delegate Role Creation

```
Vấn đề: Developer cần tạo IAM role cho Lambda
         nhưng không được phép tạo role với quyền quá lớn

Giải pháp:
1. Tạo Permission Boundary policy giới hạn quyền tối đa cho Lambda roles
2. Cấp developer quyền iam:CreateRole VỚI điều kiện phải attach boundary

Kết quả: Developer tạo được role nhưng role đó không bao giờ có
         quyền vượt quá Permission Boundary
```

Xem chi tiết: [4-permission-boundaries.md](4-permission-boundaries.md)

---

## 5. Session Policies (Chính Sách Phiên Làm Việc)

### Định Nghĩa

Policy tùy chọn được truyền vào khi gọi `sts:AssumeRole`, `sts:AssumeRoleWithWebIdentity`, hoặc `sts:GetFederationToken` — giới hạn quyền trong phiên làm việc đó mà không thay đổi role gốc.

```bash
# Assume role với session policy giới hạn chỉ một bucket
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/S3FullAccessRole \
  --role-session-name limited-session \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::specific-bucket/*"
    }]
  }'
```

### Logic Tính Quyền Với Session Policy

```
Quyền phiên = Role's identity policy ∩ Session policy

Role có: s3:*, ec2:*
Session policy: s3:GetObject only
→ Phiên này chỉ có: s3:GetObject
```

### Use Case

- **Temporary scoped access:** Cấp quyền hạn hẹp hơn cho tác vụ cụ thể
- **Just-in-time access:** Developer chỉ có quyền production trong 1 giờ làm việc cụ thể
- **Cognito Identity Pools:** Scope quyền AWS cho từng người dùng end-user

---

## 6. ACLs — Access Control Lists (Danh Sách Kiểm Soát Truy Cập)

### Định Nghĩa

Cơ chế kiểm soát truy cập cũ (legacy), chủ yếu còn tồn tại trong **Amazon S3** và **VPC**. Không dùng IAM JSON — dùng XML format và hỗ trợ ít granularity hơn.

### S3 ACL Canned Permissions

| ACL | Cấp Cho | Quyền |
|---|---|---|
| `private` | Owner | Full Control |
| `public-read` | All Users | READ |
| `public-read-write` | All Users | READ, WRITE |
| `authenticated-read` | Authenticated AWS users | READ |

### Khuyến Nghị

```
❌ Tránh dùng S3 ACLs — AWS khuyến nghị dùng Bucket Policy thay thế
✅ Bật "Block Public ACLs" cho tất cả S3 buckets
✅ Dùng Bucket Policy cho cross-account và public access
```

---

## Thứ Tự Đánh Giá Policy (Policy Evaluation Order)

```
Request đến AWS →

┌─ Bước 1: Explicit Deny? ─────────────────────────────────┐
│   Kiểm tra TẤT CẢ policies cho Deny statement            │
│   Nếu có ANY Deny → TỪ CHỐI (dừng lại)                  │
└───────────────────────────────────────────────────────────┘
          ↓ Không có Deny
┌─ Bước 2: SCPs? ──────────────────────────────────────────┐
│   Nếu dùng Organizations: SCP có Allow không?            │
│   Nếu không → TỪ CHỐI                                    │
└───────────────────────────────────────────────────────────┘
          ↓ SCP Allow
┌─ Bước 3: Resource-based Policy? ────────────────────────┐
│   Nếu resource có policy và có Allow → CHO PHÉP         │
│   (trong cùng account — không cần kiểm tra tiếp)        │
└───────────────────────────────────────────────────────────┘
          ↓ Không có resource policy hoặc cross-account
┌─ Bước 4: Permission Boundary? ──────────────────────────┐
│   Nếu có boundary: boundary có Allow không?             │
│   Nếu không → TỪ CHỐI                                   │
└───────────────────────────────────────────────────────────┘
          ↓ Boundary Allow
┌─ Bước 5: Session Policy? ───────────────────────────────┐
│   Nếu có session policy: có Allow không?                │
│   Nếu không → TỪ CHỐI                                   │
└───────────────────────────────────────────────────────────┘
          ↓ Session Policy Allow
┌─ Bước 6: Identity-based Policy? ───────────────────────┐
│   Có Allow cho action/resource này không?               │
│   Nếu có → CHO PHÉP                                     │
│   Nếu không → TỪ CHỐI (implicit deny)                  │
└───────────────────────────────────────────────────────────┘
```

---

## Bảng So Sánh Nhanh

| Loại | Gắn Vào | Cấp Quyền? | Giới Hạn Quyền? | Cross-account? |
|---|---|---|---|---|
| Identity-based | User/Group/Role | ✅ | ✅ (Deny) | Không trực tiếp |
| Resource-based | Tài nguyên | ✅ | ✅ (Deny) | ✅ Native |
| SCP | Org/OU/Account | ❌ | ✅ (trần quyền) | N/A |
| Permission Boundary | User/Role | ❌ | ✅ (trần quyền) | Không |
| Session Policy | STS session | ❌ | ✅ (trong phiên) | Không |
| ACL | S3/VPC | ✅ | ✅ | Hạn chế |

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: SCPs và Permission Boundaries khác nhau như thế nào?**

> SCP áp dụng ở cấp AWS Organizations — giới hạn quyền tối đa cho toàn bộ account/OU (kể cả root user của member accounts). Permission Boundary áp dụng ở cấp individual IAM user/role — giới hạn quyền tối đa cho một identity cụ thể. SCP là công cụ quản trị tập trung (centralized governance), Boundary là công cụ delegate an toàn (safe delegation).

**Q: Khi nào Identity-based Policy ĐỦ? Khi nào cần Resource-based Policy?**

> Identity-based policy đủ cho truy cập trong cùng account. Resource-based policy CẦN THIẾT khi: (1) cross-account access — cả hai phía đều phải Allow; (2) muốn cấp quyền cho AWS services không hỗ trợ role (ví dụ: S3 public read); (3) KMS key policy — mọi truy cập vào KMS key đều phải có trong key policy.

**Q: Explicit Deny hoạt động như thế nào? Có thể override được không?**

> Explicit Deny luôn thắng mọi Allow — không thể override. Ngay cả AdministratorAccess cũng bị Deny block nếu có explicit Deny statement. Ngoại lệ duy nhất: root user của management account không bị chặn bởi SCPs, nhưng vẫn bị chặn bởi explicit Deny trong identity/resource policy.
