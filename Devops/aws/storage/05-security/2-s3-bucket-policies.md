# S3 Bucket Policies — Chính Sách Bucket, Condition Keys, Cross-Account

> Bucket Policy là resource-based policy (chính sách dựa trên tài nguyên) gắn trực tiếp vào S3 bucket — công cụ mạnh nhất để kiểm soát ai được truy cập bucket và trong điều kiện nào.

## 📚 Mục Lục

1. [Bucket Policy vs IAM Policy](#1-bucket-policy-vs-iam-policy)
2. [Cấu Trúc Bucket Policy](#2-cấu-trúc-bucket-policy)
3. [Condition Keys Quan Trọng](#3-condition-keys-quan-trọng)
4. [Các Pattern Thực Tế](#4-các-pattern-thực-tế)
5. [Cross-Account Access](#5-cross-account-access)
6. [Policy Evaluation Logic](#6-policy-evaluation-logic)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Bucket Policy vs IAM Policy

### Bảng So Sánh

| Tiêu Chí | IAM Policy | Bucket Policy |
|----------|-----------|--------------|
| **Gắn vào** | User / Role / Group | S3 Bucket (resource) |
| **Principal** | Không cần khai báo | Phải khai báo rõ |
| **Cross-account** | Cần cả 2 bên | Bucket policy đủ cho read-only |
| **Anonymous access** | Không hỗ trợ | Hỗ trợ (Principal: "*") |
| **Dung lượng tối đa** | 6.144 bytes / entity | 20.480 bytes / bucket |
| **Quản lý tập trung** | Có (IAM) | Không — mỗi bucket riêng |

### Khi Nào Dùng Bucket Policy

```
Dùng Bucket Policy khi:
✅ Cần cấp quyền cross-account
✅ Cần cho phép anonymous (không xác thực) — ví dụ static website
✅ Cần điều kiện phức tạp dựa trên IP, VPC, MFA
✅ Cần deny toàn bộ trừ một số principal nhất định
✅ Cần enforce encryption hoặc TLS

Dùng IAM Policy khi:
✅ Quản lý quyền cho nhiều dịch vụ AWS từ một chỗ
✅ Giới hạn quyền của role/user theo nhiều dịch vụ
✅ Không cần cross-account
```

---

## 2. Cấu Trúc Bucket Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TênMôTả",
      "Effect": "Allow | Deny",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_ID:root"
      },
      "Action": ["s3:GetObject"],
      "Resource": [
        "arn:aws:s3:::bucket-name",
        "arn:aws:s3:::bucket-name/*"
      ],
      "Condition": {}
    }
  ]
}
```

### Principal — Chủ Thể

```json
// Toàn thế giới (anonymous + authenticated)
"Principal": "*"

// Tài khoản AWS cụ thể (toàn bộ account)
"Principal": {"AWS": "arn:aws:iam::123456789012:root"}

// IAM Role cụ thể
"Principal": {"AWS": "arn:aws:iam::123456789012:role/AppRole"}

// IAM User cụ thể
"Principal": {"AWS": "arn:aws:iam::123456789012:user/alice"}

// AWS Service (CloudFront, Config, v.v.)
"Principal": {"Service": "cloudfront.amazonaws.com"}

// Nhiều principal
"Principal": {
  "AWS": [
    "arn:aws:iam::111111111111:role/RoleA",
    "arn:aws:iam::222222222222:root"
  ]
}
```

---

## 3. Condition Keys Quan Trọng

### 3.1. Condition Keys cho S3

| Key | Loại | Dùng Để |
|-----|------|---------|
| `s3:prefix` | String | Giới hạn ListBucket theo prefix — tiền tố thư mục |
| `s3:delimiter` | String | Phân cách prefix trong ListBucket |
| `s3:max-keys` | Numeric | Giới hạn số object trả về |
| `s3:x-amz-acl` | String | Kiểm soát ACL khi upload |
| `s3:x-amz-server-side-encryption` | String | Enforce encryption algorithm |
| `s3:x-amz-server-side-encryption-aws-kms-key-id` | String | Enforce KMS key cụ thể |
| `s3:ExistingObjectTag/key` | String | Điều kiện dựa trên tag của object |
| `s3:RequestObjectTag/key` | String | Điều kiện dựa trên tag khi tạo object |
| `s3:DataAccessPointArn` | ARN | Giới hạn qua Access Point cụ thể |

### 3.2. Global Condition Keys

| Key | Loại | Dùng Để |
|-----|------|---------|
| `aws:SourceIp` | IP | Giới hạn theo IP nguồn |
| `aws:SourceVpc` | String | Giới hạn theo VPC ID |
| `aws:SourceVpce` | String | Giới hạn theo VPC Endpoint ID |
| `aws:SecureTransport` | Boolean | Bắt buộc HTTPS |
| `aws:MultiFactorAuthPresent` | Boolean | Bắt buộc MFA |
| `aws:MultiFactorAuthAge` | Numeric | Giới hạn thời gian MFA còn hiệu lực |
| `aws:RequestedRegion` | String | Giới hạn theo region |
| `aws:PrincipalTag/key` | String | Điều kiện dựa trên tag của principal |
| `aws:ResourceTag/key` | String | Điều kiện dựa trên tag của resource |
| `aws:CurrentTime` | Date | Giới hạn theo thời gian |

---

## 4. Các Pattern Thực Tế

### 4.1. Enforce HTTPS — Bắt Buộc Dùng TLS

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyHTTP",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": false
        }
      }
    }
  ]
}
```

### 4.2. Chỉ Cho Phép Truy Cập Từ VPC Cụ Thể

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowFromVPCOnly",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::internal-bucket",
        "arn:aws:s3:::internal-bucket/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpc": "vpc-xxxxxxxxx"
        }
      }
    }
  ]
}
```

### 4.3. Chỉ Cho Phép Qua VPC Endpoint Cụ Thể

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowFromVPCEndpointOnly",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::secure-bucket",
        "arn:aws:s3:::secure-bucket/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpce": "vpce-xxxxxxxxxxxxxxxxx"
        }
      }
    }
  ]
}
```

### 4.4. Enforce Mã Hóa KMS Khi Upload

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonKMSEncryptedUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::encrypted-bucket/*",
      "Condition": {
        "StringNotEqualsIfExists": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    },
    {
      "Sid": "DenyWrongKMSKey",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::encrypted-bucket/*",
      "Condition": {
        "StringNotEqualsIfExists": {
          "s3:x-amz-server-side-encryption-aws-kms-key-id": "arn:aws:kms:ap-southeast-1:123456789012:key/mrk-xxxxxxxx"
        }
      }
    }
  ]
}
```

### 4.5. Static Website — Cho Phép Đọc Public

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadForStaticWebsite",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-website-bucket/*"
    }
  ]
}
```

**Lưu ý:** Cần tắt Block Public Access trước khi policy này có hiệu lực.

### 4.6. Giới Hạn Upload Theo Object Tag

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RequireConfidentialTag",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::sensitive-bucket/*",
      "Condition": {
        "Null": {
          "s3:RequestObjectTag/DataClassification": true
        }
      }
    }
  ]
}
```

### 4.7. CloudFront OAC — Origin Access Control

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontOAC",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::cdn-bucket/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/XXXXXXXXXXXXXX"
        }
      }
    }
  ]
}
```

### 4.8. MFA Delete Protection — Bảo Vệ Xóa Bằng MFA

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDeleteWithoutMFA",
      "Effect": "Deny",
      "Principal": {"AWS": "*"},
      "Action": [
        "s3:DeleteObject",
        "s3:DeleteObjectVersion"
      ],
      "Resource": "arn:aws:s3:::critical-bucket/*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": false
        }
      }
    }
  ]
}
```

---

## 5. Cross-Account Access

### 5.1. Mô Hình Cross-Account

```
Account A (111111111111)          Account B (222222222222)
┌─────────────────────┐           ┌──────────────────────┐
│  S3 Bucket          │           │  IAM Role: DataReader │
│  "data-bucket"      │◄─────────▶│  → Assume role để    │
│                     │           │    đọc bucket của A  │
│  Bucket Policy:     │           │                      │
│  Cho phép Account B │           │  IAM Policy:         │
│  đọc dữ liệu        │           │  Cho phép s3:GetObject│
└─────────────────────┘           └──────────────────────┘
```

### 5.2. Bucket Policy Phía Account A (Owner)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCrossAccountRead",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::222222222222:role/DataReader"
      },
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::data-bucket",
        "arn:aws:s3:::data-bucket/*"
      ]
    }
  ]
}
```

### 5.3. IAM Policy Phía Account B (Requester)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AccessCrossAccountBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::data-bucket",
        "arn:aws:s3:::data-bucket/*"
      ]
    }
  ]
}
```

### 5.4. Cross-Account với Bucket Owner Condition

```json
{
  "Sid": "EnforceBucketOwnerFullControl",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::222222222222:root"
  },
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::data-bucket/*",
  "Condition": {
    "StringEquals": {
      "s3:x-amz-acl": "bucket-owner-full-control"
    }
  }
}
```

**Lý do:** Khi Account B upload vào bucket của Account A, mặc định Account B sở hữu object. Điều kiện `bucket-owner-full-control` buộc Account B phải nhường quyền sở hữu cho Account A.

---

## 6. Policy Evaluation Logic

### Thứ Tự Đánh Giá Đầy Đủ

```
1. DENY tường minh trong bất kỳ policy nào → TỪ CHỐI ngay
2. Block Public Access settings → nếu chặn → TỪ CHỐI
3. SCPs (Organization Service Control Policy) → nếu không có Allow → TỪ CHỐI
4. VPC Endpoint Policy → nếu không có Allow → TỪ CHỐI (chỉ khi qua VPC endpoint)
5. Resource-based policy (bucket policy) → nếu có Allow → cho phép
6. Identity-based policy (IAM) → nếu có Allow → cho phép
7. Không có Allow nào → TỪ CHỐI (implicit deny — từ chối ngầm định)
```

### Cross-Account Evaluation — Đánh Giá Liên Tài Khoản

```
Same-account:        Cross-account:
IAM Policy           Cần CẢ HAI:
OR                   → Bucket Policy ALLOW
Bucket Policy        → IAM Policy ALLOW
= ALLOW              (một bên không đủ)
```

---

## 7. Câu Hỏi Phỏng Vấn

**Q: Khác biệt giữa `aws:SourceIp` và `aws:VpcSourceIp`?**

A: `aws:SourceIp` là địa chỉ IP công cộng của request — nếu request đi qua NAT gateway thì là IP NAT. `aws:VpcSourceIp` là IP private của instance trong VPC, chỉ áp dụng khi request đến qua VPC Endpoint. Dùng `aws:SourceVpc` hoặc `aws:SourceVpce` để kiểm soát VPC thay vì dựa vào IP cụ thể.

---

**Q: Tại sao cần cả IAM Policy và Bucket Policy cho cross-account access?**

A: AWS đánh giá theo hai bước: trước hết bucket policy ở tài khoản owner phải Allow principal từ tài khoản khác; sau đó IAM policy ở tài khoản requester phải Allow action trên resource đó. Thiếu một trong hai thì bị Implicit Deny. Ngoại lệ: nếu bucket policy Allow cụ thể, IAM policy ở phía requester chỉ cần không có Deny.

---

**Q: `StringNotEqualsIfExists` khác `StringNotEquals` thế nào?**

A: `StringNotEquals` yêu cầu key phải tồn tại trong request. `StringNotEqualsIfExists` chỉ áp dụng điều kiện nếu key tồn tại — nếu key không có trong request thì bỏ qua điều kiện. Dùng `IfExists` khi key là optional để tránh block mọi request không có key đó.

---

**Q: OAC — Origin Access Control khác OAI — Origin Access Identity thế nào?**

A: OAI là cơ chế cũ dùng IAM identity ảo cho CloudFront. OAC là cơ chế mới (2022) dùng service principal `cloudfront.amazonaws.com` với điều kiện `AWS:SourceArn` chỉ định distribution cụ thể. OAC hỗ trợ SSE-KMS tốt hơn, không bị giới hạn 1.000 OAI, và an toàn hơn vì gắn với distribution ARN cụ thể.

---

**Cập Nhật:** 2026-05-16
