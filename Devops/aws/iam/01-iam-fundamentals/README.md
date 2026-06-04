# IAM Fundamentals — Nền Tảng Quản Lý Định Danh và Truy Cập AWS

> **IAM** (Identity and Access Management — Quản Lý Định Danh và Truy Cập) là dịch vụ trung tâm điều phối *ai* được làm *gì* trên AWS. Mọi lời gọi API đều đi qua IAM để kiểm tra quyền trước khi thực thi.

---

## 📚 Nội Dung Module

| File | Chủ Đề | Độ Ưu Tiên |
|---|---|---|
| [1-users-groups-roles.md](1-users-groups-roles.md) | Users, Groups, Roles — định danh trong IAM | ⭐⭐⭐ Bắt buộc |
| [2-policy-types.md](2-policy-types.md) | Các loại policy — Identity, Resource, SCP, Boundary | ⭐⭐⭐ Bắt buộc |
| [3-iam-conditions.md](3-iam-conditions.md) | Condition keys, operators, ví dụ thực tế | ⭐⭐⭐ Bắt buộc |
| [4-permission-boundaries.md](4-permission-boundaries.md) | Permission Boundaries — giới hạn quyền tối đa | ⭐⭐ Quan trọng |
| [5-iam-best-practices.md](5-iam-best-practices.md) | Checklist bảo mật IAM theo AWS Well-Architected | ⭐⭐⭐ Bắt buộc |

---

## 🧭 Tổng Quan Nhanh

### IAM Là Gì?

IAM là dịch vụ **toàn cầu** (global — không gắn với một Region cụ thể) cho phép bạn:

- Tạo và quản lý **định danh** (identities): Users, Groups, Roles
- Kiểm soát **quyền truy cập** (access) vào tài nguyên AWS
- Áp dụng nguyên tắc **Least Privilege** (Đặc Quyền Tối Thiểu): chỉ cấp đúng quyền cần thiết, không hơn

### Mô Hình Tư Duy Cốt Lõi

```
Câu hỏi IAM trả lời:
  "Principal X có được phép thực hiện Action Y
   trên Resource Z với Condition C không?"

Ví dụ:
  "User alice có được phép s3:GetObject
   trên arn:aws:s3:::my-bucket/*
   khi MFA đã được bật không?"
```

---

## 🏗️ Kiến Trúc IAM

### Các Thành Phần Chính

```
IAM
├── Identities (Định Danh)
│   ├── Users      — người dùng cụ thể, có credentials lâu dài
│   ├── Groups     — nhóm users, gán policy tập trung
│   └── Roles      — định danh tạm thời, không gắn credentials cố định
│
├── Policies (Chính Sách)
│   ├── Identity-based   — gắn với user/group/role
│   ├── Resource-based   — gắn với tài nguyên (S3, KMS...)
│   ├── SCPs             — giới hạn ở cấp Organization/OU/Account
│   ├── Boundaries       — giới hạn quyền tối đa của identity
│   ├── Session policies — giới hạn trong phiên AssumeRole
│   └── ACLs             — danh sách kiểm soát truy cập (cũ, S3/VPC)
│
└── Supporting Services (Dịch Vụ Hỗ Trợ)
    ├── STS    — Security Token Service, cấp temporary credentials
    ├── MFA    — Multi-Factor Authentication, xác thực đa yếu tố
    └── Access Analyzer — phát hiện quyền truy cập ngoài ý muốn
```

### Luồng Đánh Giá Quyền (Authorization Evaluation Flow)

```
Request → IAM đánh giá theo thứ tự:

1. Explicit Deny (Từ Chối Rõ Ràng)?
   → Nếu có: TỪ CHỐI NGAY (dừng lại)

2. SCPs cho phép không?
   → Nếu không: TỪ CHỐI

3. Resource-based policy cho phép không?
   → Nếu có: CHO PHÉP (trong cùng account)

4. Permission Boundary cho phép không?
   → Nếu không: TỪ CHỐI

5. Session Policy cho phép không?
   → Nếu không: TỪ CHỐI

6. Identity-based policy cho phép không?
   → Nếu có: CHO PHÉP
   → Nếu không: TỪ CHỐI (implicit deny — từ chối ngầm định)
```

> **Nguyên tắc quan trọng:** AWS mặc định **từ chối tất cả** (implicit deny). Quyền phải được cấp rõ ràng.

---

## 📖 Khái Niệm Cơ Bản

### Principal (Chủ Thể)

**Principal** là thực thể gửi request đến AWS:

| Loại | Ví Dụ | Dùng Khi |
|---|---|---|
| IAM User | `arn:aws:iam::123456789012:user/alice` | Người dùng có access key cố định |
| IAM Role | `arn:aws:iam::123456789012:role/ec2-role` | EC2, Lambda, cross-account |
| AWS Service | `lambda.amazonaws.com` | Dịch vụ AWS gọi dịch vụ khác |
| Federated User | Qua SAML/OIDC/Cognito | SSO từ IdP bên ngoài |
| Root Account | `arn:aws:iam::123456789012:root` | Chỉ cho tác vụ đặc biệt |

### Action (Hành Động)

Action là API call mà principal muốn thực hiện:

```json
"Action": [
  "s3:GetObject",
  "s3:PutObject",
  "ec2:DescribeInstances",
  "kms:Decrypt"
]
```

Định dạng: `service:APIMethodName` (ví dụ: `s3:PutObject`, `iam:CreateRole`)

### Resource (Tài Nguyên)

Resource là đối tượng mà action tác động lên, được xác định bằng **ARN** (Amazon Resource Name — Tên Tài Nguyên Amazon):

```
arn:aws:s3:::my-bucket/*
arn:aws:iam::123456789012:user/alice
arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0
```

Cú pháp ARN: `arn:partition:service:region:account-id:resource`

### Condition (Điều Kiện)

Condition cho phép kiểm soát chi tiết dựa trên context của request:

```json
"Condition": {
  "Bool": {
    "aws:MultiFactorAuthPresent": "true"
  },
  "StringEquals": {
    "aws:RequestedRegion": "us-east-1"
  }
}
```

---

## 🔑 Credentials (Thông Tin Xác Thực)

### IAM User Credentials

| Loại | Dùng Cho | Lưu Ý |
|---|---|---|
| Username + Password | AWS Console (Bảng Điều Khiển) | Bật MFA bắt buộc |
| Access Key + Secret Key | AWS CLI, SDK, API | Không nhúng vào code |
| MFA device | Tăng cường bảo mật | Bắt buộc cho user có quyền cao |

### Temporary Credentials (Thông Tin Xác Thực Tạm Thời)

**STS** (Security Token Service — Dịch Vụ Token Bảo Mật) cấp thông tin xác thực tạm thời gồm:

- `AccessKeyId` — key ID tạm thời
- `SecretAccessKey` — secret key tạm thời
- `SessionToken` — token phiên làm việc
- `Expiration` — thời điểm hết hạn (1 giờ đến 12 giờ)

```bash
# Assume một role và lấy temporary credentials
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/MyRole \
  --role-session-name my-session
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

### Mức Độ Cơ Bản

**Q: IAM User và IAM Role khác nhau như thế nào?**

> User có credentials lâu dài (access key, password) gắn với một người/ứng dụng cụ thể. Role là định danh tạm thời không có credentials cố định — được "assume" (đảm nhận) bởi services, users, hoặc accounts khác và nhận temporary credentials qua STS.

**Q: Group trong IAM có thể chứa Group khác không?**

> Không. IAM Groups không hỗ trợ lồng nhau (no nested groups). Một user có thể thuộc tối đa 10 groups.

**Q: Root account nên dùng cho việc gì?**

> Root account chỉ dùng cho các tác vụ không thể thực hiện bằng IAM user: thay đổi support plan, đóng account, bật MFA cho root, thay đổi payment method. Mọi thao tác thông thường nên dùng IAM user/role với least privilege.

### Mức Độ Trung Bình

**Q: Khi request bị Access Denied, bạn debug theo thứ tự nào?**

> 1. Kiểm tra có explicit Deny nào không (SCPs, identity policy, resource policy, boundary)
> 2. Kiểm tra identity-based policy có Allow đúng action/resource không
> 3. Nếu cross-account: kiểm tra cả resource policy lẫn identity policy
> 4. Kiểm tra Permission Boundary có chặn không
> 5. Dùng IAM Policy Simulator để test

**Q: Cách cross-account access hoạt động?**

> Account A tạo Role với Trust Policy cho phép Account B assume. Account B's principal gọi `sts:AssumeRole` để lấy temporary credentials của Role trong Account A, sau đó dùng credentials đó để truy cập tài nguyên Account A.

---

## 🗺️ Lộ Trình Học Module Này

```
Bước 1: [1-users-groups-roles.md]   — Hiểu 3 loại identity cơ bản (30 phút)
Bước 2: [2-policy-types.md]         — Nắm 6 loại policy và khi nào dùng (45 phút)
Bước 3: [3-iam-conditions.md]       — Thực hành viết Condition phức tạp (30 phút)
Bước 4: [4-permission-boundaries.md]— Hiểu cơ chế giới hạn quyền (30 phút)
Bước 5: [5-iam-best-practices.md]   — Áp dụng checklist vào dự án thực (20 phút)
```

---

## 🔗 Tiếp Theo

Sau khi hoàn thành module này, tiếp tục với:

- **[02-identity-federation/](../02-identity-federation/README.md)** — SSO, SAML, OIDC
- **[03-organizations/](../03-organizations/README.md)** — Multi-account với SCPs
- **[11-interview-prep/2-iam-troubleshooting.md](../11-interview-prep/2-iam-troubleshooting.md)** — Debug IAM thực tế

---

**Thời Gian Học Ước Tính:** 2–3 giờ lý thuyết + 3–5 giờ thực hành lab
