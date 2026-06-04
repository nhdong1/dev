# IAM Conditions — Điều Kiện Kiểm Soát Truy Cập Chi Tiết

> **IAM Conditions** (Điều Kiện IAM) là cơ chế mở rộng mạnh mẽ cho phép bạn kiểm soát quyền truy cập dựa trên **context của request** — không chỉ là "ai" và "làm gì", mà còn "khi nào", "từ đâu", "bằng cách nào".

---

## Cấu Trúc Condition

```json
"Condition": {
  "<ConditionOperator>": {
    "<ConditionKey>": "<ConditionValue>"
  }
}
```

**Ví dụ đơn giản:**

```json
"Condition": {
  "StringEquals": {
    "aws:RequestedRegion": "ap-southeast-1"
  }
}
```

### Khi Nhiều Conditions (AND Logic)

```json
"Condition": {
  "Bool": {
    "aws:MultiFactorAuthPresent": "true"
  },
  "IpAddress": {
    "aws:SourceIp": "203.0.113.0/24"
  }
}
```

Tất cả conditions trong cùng block phải đúng (AND). Các operators khác nhau trong cùng statement cũng là AND.

### Khi Nhiều Values Trong Một Key (OR Logic)

```json
"Condition": {
  "StringEquals": {
    "aws:RequestedRegion": ["ap-southeast-1", "ap-northeast-1"]
  }
}
```

Các values trong mảng là OR — ít nhất một giá trị phải khớp.

---

## Condition Operators (Toán Tử Điều Kiện)

### String Operators (Toán Tử Chuỗi)

| Operator | Mô Tả | Case Sensitive? |
|---|---|---|
| `StringEquals` | Bằng chính xác | ✅ |
| `StringNotEquals` | Không bằng | ✅ |
| `StringEqualsIgnoreCase` | Bằng (không phân biệt hoa/thường) | ❌ |
| `StringLike` | Pattern matching với `*` và `?` | ✅ |
| `StringNotLike` | Không khớp pattern | ✅ |

```json
// Chỉ cho phép tag có prefix "env-"
"Condition": {
  "StringLike": {
    "aws:RequestTag/Environment": "env-*"
  }
}
```

### Numeric Operators (Toán Tử Số)

| Operator | Mô Tả |
|---|---|
| `NumericEquals` | Bằng |
| `NumericNotEquals` | Không bằng |
| `NumericLessThan` | Nhỏ hơn |
| `NumericLessThanEquals` | Nhỏ hơn hoặc bằng |
| `NumericGreaterThan` | Lớn hơn |
| `NumericGreaterThanEquals` | Lớn hơn hoặc bằng |

```json
// Chỉ cho phép download file nhỏ hơn 10MB qua S3
"Condition": {
  "NumericLessThanEquals": {
    "s3:max-keys": "1000"
  }
}
```

### Date Operators (Toán Tử Ngày Giờ)

| Operator | Mô Tả |
|---|---|
| `DateEquals` | Bằng |
| `DateLessThan` | Trước thời điểm |
| `DateGreaterThan` | Sau thời điểm |
| `DateLessThanEquals` | Trước hoặc đúng thời điểm |

```json
// Quyền tạm thời chỉ có hiệu lực đến hết năm 2026
"Condition": {
  "DateLessThan": {
    "aws:CurrentTime": "2026-12-31T23:59:59Z"
  }
}
```

### Boolean Operators (Toán Tử Boolean)

```json
"Condition": {
  "Bool": {
    "aws:MultiFactorAuthPresent": "true"
  }
}
```

### IP Address Operators (Toán Tử Địa Chỉ IP)

| Operator | Mô Tả |
|---|---|
| `IpAddress` | Địa chỉ IP nằm trong range |
| `NotIpAddress` | Địa chỉ IP nằm ngoài range |

```json
"Condition": {
  "IpAddress": {
    "aws:SourceIp": ["203.0.113.0/24", "198.51.100.0/24"]
  }
}
```

### ARN Operators (Toán Tử ARN)

| Operator | Mô Tả |
|---|---|
| `ArnEquals` | ARN bằng chính xác |
| `ArnLike` | ARN khớp pattern với `*` và `?` |
| `ArnNotEquals` | ARN không bằng |
| `ArnNotLike` | ARN không khớp pattern |

```json
// Chỉ cho phép từ VPC Endpoint cụ thể
"Condition": {
  "ArnEquals": {
    "aws:SourceVpce": "vpce-1234567890abcdef0"
  }
}
```

### Null Operator (Kiểm Tra Null)

Kiểm tra xem condition key có tồn tại trong request hay không:

```json
// Từ chối nếu KHÔNG có MFA (key không tồn tại trong request)
"Condition": {
  "Null": {
    "aws:MultiFactorAuthPresent": "true"
  }
}
```

```
Null: "true"  → key KHÔNG có trong request (giá trị là null)
Null: "false" → key CÓ trong request (giá trị khác null)
```

### IfExists Modifier (Chỉ Áp Dụng Khi Key Tồn Tại)

Thêm `IfExists` vào bất kỳ operator để chỉ kiểm tra khi key có mặt:

```json
// Nếu có tag Environment, nó phải là "production"
// Nếu không có tag, bỏ qua điều kiện này
"Condition": {
  "StringEqualsIfExists": {
    "ec2:ResourceTag/Environment": "production"
  }
}
```

---

## Condition Keys (Khóa Điều Kiện)

### Global Condition Keys (Khóa Toàn Cầu) — Tiền tố `aws:`

Áp dụng cho mọi dịch vụ AWS:

#### Kiểm Soát Định Danh & Xác Thực

| Key | Mô Tả | Ví Dụ Giá Trị |
|---|---|---|
| `aws:PrincipalArn` | ARN của principal | `arn:aws:iam::123:user/alice` |
| `aws:PrincipalType` | Loại principal | `User`, `AssumedRole`, `FederatedUser` |
| `aws:MultiFactorAuthPresent` | MFA đã được dùng | `true`, `false` |
| `aws:MultiFactorAuthAge` | Giây kể từ MFA auth | `3600` |
| `aws:PrincipalAccount` | Account ID của principal | `123456789012` |
| `aws:PrincipalOrgID` | Org ID của principal | `o-aa111bb222` |
| `aws:PrincipalOrgPaths` | Đường dẫn OU của principal | `o-aa111/r-aa11/ou-aa11-*/` |
| `aws:PrincipalTag/<key>` | Tag của IAM principal | Tùy chọn |

#### Kiểm Soát Nguồn Request

| Key | Mô Tả | Ví Dụ Giá Trị |
|---|---|---|
| `aws:SourceIp` | IP của người gọi (không qua VPC) | `203.0.113.0/24` |
| `aws:VpcSourceIp` | IP trong VPC | `10.0.0.0/8` |
| `aws:SourceVpc` | VPC ID nguồn | `vpc-12345678` |
| `aws:SourceVpce` | VPC Endpoint ID | `vpce-12345678` |
| `aws:RequestedRegion` | Region của request | `ap-southeast-1` |
| `aws:CurrentTime` | Thời điểm request | ISO 8601 |

#### Kiểm Soát Tài Nguyên

| Key | Mô Tả |
|---|---|
| `aws:ResourceTag/<key>` | Tag trên tài nguyên đích |
| `aws:RequestTag/<key>` | Tag được yêu cầu trong request tạo/sửa |
| `aws:TagKeys` | Tên các tag keys trong request |

#### Kiểm Soát Dịch Vụ

| Key | Mô Tả |
|---|---|
| `aws:CalledVia` | Dịch vụ AWS gọi thay mặt user |
| `aws:CalledViaFirst` | Dịch vụ đầu tiên trong chuỗi gọi |
| `aws:CalledViaLast` | Dịch vụ cuối trong chuỗi gọi |
| `aws:ViaAWSService` | Request đến qua dịch vụ AWS |

### Service-specific Condition Keys (Khóa Riêng Của Dịch Vụ)

Tiền tố bằng tên service: `s3:`, `ec2:`, `iam:`, `kms:`...

**S3 Keys:**

| Key | Mô Tả |
|---|---|
| `s3:prefix` | Prefix của object key |
| `s3:delimiter` | Delimiter trong ListObjects |
| `s3:max-keys` | Số object tối đa trả về |
| `s3:x-amz-acl` | ACL được yêu cầu |
| `s3:x-amz-server-side-encryption` | Loại mã hóa server-side |
| `s3:x-amz-storage-class` | Storage class |

**EC2 Keys:**

| Key | Mô Tả |
|---|---|
| `ec2:Region` | Region của resource |
| `ec2:ResourceTag/<key>` | Tag của EC2 resource |
| `ec2:InstanceType` | Loại instance |

**IAM Keys:**

| Key | Mô Tả |
|---|---|
| `iam:PassedToService` | Service nhận role được pass |
| `iam:PermissionsBoundary` | ARN của Permission Boundary |
| `iam:OrganizationsPolicyId` | Policy ID của SCP |

---

## Ví Dụ Thực Tế

### 1. Bắt Buộc MFA Cho Hành Động Nhạy Cảm

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAllWithMFA",
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    },
    {
      "Sid": "AllowReadOnlyWithoutMFA",
      "Effect": "Allow",
      "Action": [
        "iam:GetUser",
        "iam:ListUsers",
        "iam:GetMFADevice"
      ],
      "Resource": "*"
    }
  ]
}
```

### 2. Restrict Theo Region (Data Residency)

```json
{
  "Sid": "DenyOutsideApac",
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": [
        "ap-southeast-1",
        "ap-southeast-2",
        "ap-northeast-1"
      ]
    },
    "Bool": {
      "aws:ViaAWSService": "false"
    }
  }
}
```

> `aws:ViaAWSService: false` để không chặn dịch vụ global như IAM, CloudFront, Route53.

### 3. ABAC — Attribute-Based Access Control (Kiểm Soát Truy Cập Dựa Trên Thuộc Tính)

Cho phép truy cập tài nguyên dựa trên tag match giữa principal và resource:

```json
{
  "Sid": "AllowAccessByTeamTag",
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject"
  ],
  "Resource": "arn:aws:s3:::*/*",
  "Condition": {
    "StringEquals": {
      "s3:ExistingObjectTag/Team": "${aws:PrincipalTag/Team}"
    }
  }
}
```

Khi user có tag `Team=backend` → chỉ đọc/ghi object có tag `Team=backend`.

### 4. Protect Production Resources

```json
{
  "Sid": "DenyDeleteOnProduction",
  "Effect": "Deny",
  "Action": [
    "ec2:TerminateInstances",
    "rds:DeleteDBInstance",
    "s3:DeleteBucket"
  ],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "aws:ResourceTag/Environment": "production"
    }
  }
}
```

### 5. Giới Hạn EC2 Instance Type

```json
{
  "Sid": "AllowOnlyApprovedInstanceTypes",
  "Effect": "Deny",
  "Action": "ec2:RunInstances",
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": {
    "StringNotEquals": {
      "ec2:InstanceType": [
        "t3.micro",
        "t3.small",
        "t3.medium"
      ]
    }
  }
}
```

### 6. Chỉ Cho Phép Từ VPC Endpoint (Bảo Vệ S3 Bucket)

```json
{
  "Sid": "DenyAccessOutsideVpcEndpoint",
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": [
    "arn:aws:s3:::sensitive-data-bucket",
    "arn:aws:s3:::sensitive-data-bucket/*"
  ],
  "Condition": {
    "StringNotEquals": {
      "aws:SourceVpce": "vpce-0a1b2c3d4e5f67890"
    }
  }
}
```

### 7. Restrict Theo Organization (Toàn Bộ Org)

```json
{
  "Sid": "AllowOnlyWithinOrg",
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": [
    "arn:aws:s3:::org-internal-bucket",
    "arn:aws:s3:::org-internal-bucket/*"
  ],
  "Condition": {
    "StringNotEquals": {
      "aws:PrincipalOrgID": "o-aa111bb222cc"
    }
  }
}
```

### 8. Kiểm Soát IAM PassRole

```json
{
  "Sid": "AllowPassRoleOnlyToLambda",
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::*:role/lambda-*",
  "Condition": {
    "StringEquals": {
      "iam:PassedToService": "lambda.amazonaws.com"
    }
  }
}
```

---

## Lỗi Thường Gặp Với Conditions

### Lỗi 1: Nhầm aws:SourceIp và aws:VpcSourceIp

```
aws:SourceIp    → IP của request bên ngoài VPC (public IP)
aws:VpcSourceIp → IP của request trong VPC (private IP)

Nếu request đi qua VPC:
  aws:SourceIp sẽ là IP của NAT Gateway, không phải EC2
  → Dùng aws:VpcSourceIp hoặc aws:SourceVpc để chính xác hơn
```

### Lỗi 2: Quên aws:ViaAWSService Khi Deny Theo Region

```json
// Sai — sẽ chặn cả CloudFront, IAM global calls
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

// Đúng — loại trừ calls qua AWS services
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": "ap-southeast-1"
    },
    "Bool": {
      "aws:ViaAWSService": "false"
    }
  }
}
```

### Lỗi 3: Nhầm Null Operator

```json
// Ý định: Từ chối nếu KHÔNG có MFA
// Sai — "Null: true" nghĩa là key IS null (không tồn tại)
{
  "Effect": "Deny",
  "Condition": {
    "Bool": {
      "aws:MultiFactorAuthPresent": "false"  // ← Đúng
    }
  }
}

// Hoặc
{
  "Effect": "Allow",
  "Condition": {
    "Bool": {
      "aws:MultiFactorAuthPresent": "true"   // ← Đúng
    }
  }
}
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Tại sao cần thêm `aws:ViaAWSService: false` khi dùng Deny theo Region?**

> Nhiều dịch vụ AWS là **global** (IAM, CloudFront, Route53, STS) — request của chúng có `aws:RequestedRegion` là region khác hoặc không có region. Nếu không thêm điều kiện loại trừ, SCP/policy sẽ vô tình chặn cả dịch vụ global, gây lỗi khi deploy CloudFormation stacks và các tác vụ hạ tầng cơ bản.

**Q: Sự khác biệt giữa aws:RequestTag và aws:ResourceTag?**

> `aws:RequestTag` kiểm tra tag được TRUYỀN VÀO trong request tạo/sửa tài nguyên (ví dụ: khi tạo EC2, tag nào được thêm vào). `aws:ResourceTag` kiểm tra tag ĐÃ TỒN TẠI trên tài nguyên (khi đọc/xóa tài nguyên hiện có). Dùng `aws:RequestTag` để enforce tagging policy khi tạo resource, dùng `aws:ResourceTag` để kiểm soát access dựa trên tag hiện có.

**Q: Conditions trong cùng một Statement quan hệ AND hay OR?**

> AND. Mọi conditions phải đúng. Nhưng nhiều values trong một key là OR. Ví dụ: `StringEquals: {"aws:RequestedRegion": ["us-east-1", "ap-southeast-1"]}` nghĩa là region phải là us-east-1 HOẶC ap-southeast-1. Nếu cần OR giữa các conditions khác type, cần tách ra nhiều Statements hoặc dùng logic khác.
