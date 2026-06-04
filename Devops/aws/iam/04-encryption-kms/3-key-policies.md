# Key Policies — Chính Sách Khóa KMS

> Key Policy (Chính Sách Khóa) là tài liệu JSON **bắt buộc** gắn liền với mỗi CMK (Customer Managed Key — Khóa Do Khách Hàng Quản Lý) trong AWS KMS. Không giống như IAM resources khác, CMK **không thể truy cập** chỉ bằng IAM policy — phải có key policy cho phép trước. Đây là nguồn gốc của nhiều lỗi "Access Denied" bí ẩn khi làm việc với KMS.

---

## 📚 Mục Lục

1. [Nguyên Tắc Cốt Lõi](#nguyên-tắc-cốt-lõi)
2. [Cấu Trúc Key Policy](#cấu-trúc-key-policy)
3. [Ví Dụ Key Policy Hoàn Chỉnh](#ví-dụ-hoàn-chỉnh)
4. [IAM Policy vs Key Policy](#iam-vs-key-policy)
5. [Mô Hình Quyết Định Truy Cập](#mô-hình-quyết-định)
6. [Cross-Account Key Access](#cross-account)
7. [Condition Keys KMS](#condition-keys)
8. [Key Administrators vs Key Users](#admin-vs-users)
9. [Lỗi Phổ Biến](#lỗi-phổ-biến)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Nguyên Tắc Cốt Lõi

### Key Policy Là Bắt Buộc

```
CMK không có key policy → KHÔNG AI TRUY CẬP ĐƯỢC (kể cả root)

Khi tạo CMK, AWS tạo key policy mặc định:
{
  "Principal": {"AWS": "arn:aws:iam::<account-id>:root"},
  "Action": "kms:*",
  "Resource": "*"
}

Chú ý: "root" ở đây là account root, KHÔNG phải root user.
Ý nghĩa: IAM policies trong account có thể kiểm soát key này.
```

### Ba Cách Kiểm Soát Truy Cập CMK

```
1. Key Policy (chính sách trên key) — LUÔN CẦN THIẾT
   └── Xác định ai có thể làm gì với key này

2. IAM Policy (chính sách trên identity) — CẦN NẾU key policy ủy quyền cho IAM
   └── Grant/Deny permissions cho users, roles, groups

3. KMS Grants (cấp quyền tạm thời) — TÙY CHỌN
   └── Ủy quyền key usage tạm thời, programmatically
```

### Mô Hình "Default Deny" (Từ Chối Mặc Định)

```
Yêu cầu truy cập CMK → KMS kiểm tra:

1. Key policy có DENY không? → Từ chối
2. Key policy có ALLOW cho principal không? → Cho phép
3. Key policy có ủy quyền cho IAM không?
   ├── Có: Kiểm tra IAM policy
   │    └── IAM có ALLOW không? → Cho phép
   └── Không: Từ chối (kể cả IAM admin)

Kết quả: KHÔNG CÓ TRONG KEY POLICY = KHÔNG TRUY CẬP ĐƯỢC
```

---

## Cấu Trúc Key Policy

```json
{
  "Version": "2012-10-17",
  "Id": "key-policy-id",       // tên định danh, tùy chọn
  "Statement": [
    {
      "Sid": "statement-id",   // tên mô tả, tùy chọn
      "Effect": "Allow",       // Allow hoặc Deny
      "Principal": {           // AI được áp dụng rule này
        "AWS": [
          "arn:aws:iam::123456789012:root",
          "arn:aws:iam::123456789012:role/AdminRole"
        ]
      },
      "Action": [              // THAO TÁC nào được phép
        "kms:Encrypt",
        "kms:Decrypt"
      ],
      "Resource": "*",         // Luôn là "*" trong key policy (chính key này)
      "Condition": {           // Điều kiện bổ sung (tùy chọn)
        "StringEquals": {
          "kms:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}
```

### KMS Actions Quan Trọng

| Action | Mô Tả | Dành Cho |
|---|---|---|
| `kms:Encrypt` | Mã hóa dữ liệu hoặc DEK | App encrypt |
| `kms:Decrypt` | Giải mã dữ liệu hoặc DEK | App decrypt |
| `kms:GenerateDataKey` | Tạo DEK (plaintext + encrypted) | App envelope encryption |
| `kms:GenerateDataKeyWithoutPlaintext` | Tạo Encrypted DEK không có plaintext | Service encryption |
| `kms:ReEncrypt*` | Mã hóa lại với key khác mà không lộ plaintext | Key rotation |
| `kms:DescribeKey` | Xem metadata của key | Audit, app startup |
| `kms:ListKeys` | Liệt kê keys | Admin |
| `kms:CreateKey` | Tạo key mới | Admin |
| `kms:EnableKeyRotation` | Bật auto rotation | Admin |
| `kms:PutKeyPolicy` | Cập nhật key policy | Key administrator |
| `kms:CreateGrant` | Tạo grant ủy quyền | Dịch vụ AWS (EBS, EKS...) |
| `kms:ScheduleKeyDeletion` | Lên lịch xóa key | Admin |
| `kms:CancelKeyDeletion` | Hủy lịch xóa | Admin |

---

## Ví Dụ Hoàn Chỉnh

### Key Policy Production Chuẩn

```json
{
  "Version": "2012-10-17",
  "Id": "prod-app-key-policy",
  "Statement": [
    {
      "Sid": "EnableRootAccountFullControl",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowKeyAdministrators",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::123456789012:role/SecurityAdminRole",
          "arn:aws:iam::123456789012:role/BreakGlassRole"
        ]
      },
      "Action": [
        "kms:Create*",
        "kms:Describe*",
        "kms:Enable*",
        "kms:List*",
        "kms:Put*",
        "kms:Update*",
        "kms:Revoke*",
        "kms:Disable*",
        "kms:Get*",
        "kms:Delete*",
        "kms:TagResource",
        "kms:UntagResource",
        "kms:ScheduleKeyDeletion",
        "kms:CancelKeyDeletion"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowApplicationRoleToUseKey",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/ProdAppRole"
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowS3ServiceToUseKey",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": [
        "kms:GenerateDataKey",
        "kms:Decrypt"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:CallerAccount": "123456789012"
        },
        "StringLike": {
          "kms:ViaService": "s3.*.amazonaws.com"
        }
      }
    },
    {
      "Sid": "DenyExternalAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "kms:*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "kms:CallerAccount": "123456789012"
        },
        "Bool": {
          "aws:PrincipalIsAWSService": "false"
        }
      }
    }
  ]
}
```

---

## IAM vs Key Policy

### Sự Khác Biệt Căn Bản

```
IAM Policy (gắn với identity):
  ├── Ai: user, role, group
  ├── Làm gì: actions (kms:Encrypt, ...)
  ├── Trên gì: resource ARN
  └── Chỉ có tác dụng NẾU key policy cho phép account đó dùng IAM

Key Policy (gắn với key):
  ├── Ai: principal ARN (account, role, user, service)
  ├── Làm gì: actions
  ├── Luôn phải có — "source of truth" cho key access
  └── Có thể cho phép toàn tài khoản → IAM quyết định
```

### Kết Hợp Hai Cơ Chế

#### Mô Hình 1: Key Policy kiểm soát hoàn toàn

```json
// Key policy liệt kê từng principal cụ thể
{
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/SpecificRole"
  },
  "Action": ["kms:Decrypt"],
  ...
}

// IAM policy không cần thiết (key policy đủ)
```

#### Mô Hình 2: Ủy Quyền Cho IAM (Recommended)

```json
// Key policy ủy quyền cho toàn account
{
  "Sid": "Enable IAM User Permissions",
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:root"  // toàn account
  },
  "Action": "kms:*",
  "Resource": "*"
}

// IAM policy sau đó kiểm soát cụ thể từng role/user
{
  "Effect": "Allow",
  "Action": ["kms:Decrypt", "kms:DescribeKey"],
  "Resource": "arn:aws:kms:us-east-1:123456789012:key/*"
}
```

**Khuyến nghị:** Dùng Mô Hình 2 — dễ quản lý hơn khi nhiều identities cần truy cập.

---

## Mô Hình Quyết Định

### Logic Đánh Giá KMS Authorization

```
Request: role/AppRole muốn kms:Decrypt với CMK mrk-abc123

Bước 1: KMS kiểm tra key policy của mrk-abc123
        ├── Có EXPLICIT DENY cho AppRole không?
        │   └── CÓ → DENY (kết thúc, không kiểm tra tiếp)
        │
        ├── Có ALLOW cho AppRole trực tiếp không?
        │   └── CÓ → ALLOW
        │
        └── Key policy có Statement "Allow root" không?
            ├── KHÔNG → DENY (key policy không đề cập đến AppRole)
            └── CÓ → chuyển sang Bước 2

Bước 2: Kiểm tra IAM policy của AppRole
        ├── Có EXPLICIT DENY kms:Decrypt không?
        │   └── CÓ → DENY
        │
        └── Có ALLOW kms:Decrypt cho key ARN đó không?
            ├── CÓ → ALLOW
            └── KHÔNG → DENY

Kết quả: Cần BOTH key policy (allow account) AND IAM policy (allow action)
```

---

## Cross-Account Key Access

### Kịch Bản: Account B Muốn Dùng Key Của Account A

```
Account A (key owner): 123456789012
Account B (key user):  987654321098
Key: arn:aws:kms:us-east-1:123456789012:key/mrk-abc123
```

#### Bước 1: Key policy trong Account A cho phép Account B

```json
{
  "Sid": "AllowCrossAccountAccess",
  "Effect": "Allow",
  "Principal": {
    "AWS": [
      "arn:aws:iam::987654321098:root",
      "arn:aws:iam::987654321098:role/DataProcessingRole"
    ]
  },
  "Action": [
    "kms:Decrypt",
    "kms:GenerateDataKey",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

#### Bước 2: IAM policy trong Account B cho phép role dùng key

```json
{
  "Effect": "Allow",
  "Action": [
    "kms:Decrypt",
    "kms:GenerateDataKey",
    "kms:DescribeKey"
  ],
  "Resource": "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"
}
```

**Lưu ý quan trọng:**
- Phải có CẢ HAI: key policy (Account A) + IAM policy (Account B)
- Chỉ CMK mới hỗ trợ cross-account (AWS Managed Keys không hỗ trợ)
- CloudTrail logs sẽ ghi nhận ở cả hai account

---

## Condition Keys

### Các Condition Keys Quan Trọng Của KMS

#### `kms:CallerAccount` — Giới Hạn Theo Account

```json
{
  "Condition": {
    "StringEquals": {
      "kms:CallerAccount": "123456789012"
    }
  }
}
```

#### `kms:ViaService` — Chỉ Cho Phép Qua Dịch Vụ Cụ Thể

```json
// Chỉ cho phép khi request đến từ S3 service (không phải trực tiếp từ CLI/app)
{
  "Condition": {
    "StringEquals": {
      "kms:ViaService": [
        "s3.us-east-1.amazonaws.com",
        "rds.us-east-1.amazonaws.com"
      ]
    }
  }
}
```

#### `kms:EncryptionContext` — Yêu Cầu Đúng Context

```json
// Chỉ cho phép decrypt nếu encryption context đúng
{
  "Condition": {
    "StringEquals": {
      "kms:EncryptionContext:environment": "production",
      "kms:EncryptionContext:team": "backend"
    }
  }
}
```

#### `kms:RequestedRegion` — Giới Hạn Theo Region

```json
// Multi-Region Key: chỉ cho phép dùng ở specific regions
{
  "Condition": {
    "StringEquals": {
      "kms:RequestedRegion": ["us-east-1", "us-west-2"]
    }
  }
}
```

#### `kms:GrantConstraintType` — Kiểm Soát Grant Operations

```json
{
  "Condition": {
    "StringEquals": {
      "kms:GrantConstraintType": "EncryptionContextSubset"
    }
  }
}
```

### Ví Dụ Kết Hợp Condition Nâng Cao

```json
{
  "Sid": "AllowDecryptOnlyInProd",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/AppRole"
  },
  "Action": "kms:Decrypt",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "kms:EncryptionContext:environment": "production",
      "kms:ViaService": "s3.us-east-1.amazonaws.com"
    },
    "Bool": {
      "aws:MultiFactorAuthPresent": "true"
    },
    "IpAddress": {
      "aws:SourceIp": ["10.0.0.0/8", "172.16.0.0/12"]
    }
  }
}
```

---

## Admin vs Users

### Phân Tách Vai Trò (Separation of Duties)

```
Key Administrators (Quản Trị Viên Khóa):
  ├── Quản lý lifecycle của key (enable, disable, delete)
  ├── Cập nhật key policy
  ├── Quản lý grants
  ├── Không nhất thiết phải có quyền Encrypt/Decrypt
  └── Actions: Create*, Describe*, Enable*, Disable*, Delete*, Put*, Get*, List*

Key Users (Người Dùng Khóa):
  ├── Dùng key cho mục đích mã hóa/giải mã
  ├── Không thể thay đổi key policy
  └── Actions: Encrypt, Decrypt, GenerateDataKey*, ReEncrypt*, DescribeKey
```

### Tại Sao Phân Tách?

```
Nguyên tắc Separation of Duties (Phân Tách Nhiệm Vụ):
  ├── Admin xóa nhầm key → không ảnh hưởng data ngay (pending deletion)
  ├── App bị compromise → không thể xóa key hay thay policy
  ├── Admin không thể đọc dữ liệu mà không có app role
  └── Đáp ứng yêu cầu audit của PCI-DSS và SOC2
```

---

## Lỗi Phổ Biến

### 1. `AccessDeniedException` Khi Dùng AWS Managed Key

```bash
# Lỗi:
An error occurred (AccessDeniedException) when calling the Decrypt operation:
The ciphertext refers to a customer master key that does not exist,
does not exist in this region, or you are not allowed to access.

# Nguyên nhân thường gặp:
# - IAM role không có permission kms:Decrypt
# - Key policy không cho phép account/role này
# - Key đang ở trạng thái Disabled hoặc PendingDeletion
# - Đang dùng key ở sai region
```

### 2. Không Thể Thay Đổi Key Policy

```bash
# Lỗi khi PutKeyPolicy:
An error occurred (AccessDeniedException): You don't have permission
to change the key policy because you might lose access to the key.

# Nguyên nhân: Key policy hiện tại không có statement cho phép
# current principal thực hiện kms:PutKeyPolicy

# Giải pháp: Dùng root account để fix key policy
aws kms put-key-policy \
  --key-id alias/my-key \
  --policy-name default \
  --policy file://fixed-key-policy.json
```

### 3. "Locked Out" Khỏi Key

```
Tình huống: Xóa Statement "Enable IAM User Permissions" khỏi key policy
→ Không ai có thể truy cập key nữa (kể cả admin)

Giải pháp: Chỉ root user (arn:aws:iam::<account-id>:root) có thể
          gọi PutKeyPolicy khi không có ai khác có quyền

Phòng ngừa: Luôn giữ Statement với Principal root trong key policy
```

### 4. Cross-Account Không Hoạt Động

```
Tình huống: Account B có IAM policy cho phép kms:Decrypt
            nhưng vẫn bị từ chối

Nguyên nhân: Quên cập nhật key policy trong Account A
→ Phải có CÙNG LÚC: key policy (Account A) + IAM policy (Account B)

Debug:
aws kms get-key-policy \
  --key-id <key-id> \
  --policy-name default
```

---

## Câu Hỏi Phỏng Vấn

**Q: Key Policy và IAM Policy khác nhau như thế nào khi kiểm soát truy cập KMS?**

> Key policy là "resource policy" gắn với CMK — nó là nguồn xác thực chính (primary source of truth). Không có key policy → không ai truy cập được, kể cả IAM admin. IAM policy là "identity policy" gắn với user/role — chỉ có tác dụng khi key policy ủy quyền cho account thông qua statement `Allow root`. Điểm khác biệt lớn: với IAM resources khác (S3, EC2...), IAM policy đủ để truy cập; với CMK, cần CẢ HAI nếu key policy dùng mô hình "ủy quyền IAM."

**Q: Làm thế nào để cho phép một dịch vụ AWS (như S3) dùng CMK của bạn?**

> Thêm `"Service": "s3.amazonaws.com"` vào Principal trong key policy, kết hợp với Condition `kms:ViaService` để đảm bảo chỉ request thực sự từ S3 service mới được phép (không phải user giả vờ là S3). Ví dụ: `"kms:ViaService": "s3.us-east-1.amazonaws.com"` và `"kms:CallerAccount": "<account-id>"`.

**Q: Một developer vô tình xóa Statement "Enable IAM User Permissions" khỏi key policy và bây giờ không ai vào được key. Cách giải quyết?**

> Root user (account root IAM entity) có quyền đặc biệt: luôn có thể gọi `PutKeyPolicy` trên bất kỳ CMK nào trong account, ngay cả khi key policy không đề cập đến root. Đăng nhập bằng root account credentials và gọi `aws kms put-key-policy` với key policy mới bao gồm lại Statement ủy quyền. Đây là "emergency access" mechanism của KMS.

---

**Trước Đó:** [2-envelope-encryption.md](2-envelope-encryption.md)
**Tiếp Theo:** [4-kms-grants.md](4-kms-grants.md) — KMS Grants

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
