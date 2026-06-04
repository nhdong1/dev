# Block Public Access — Chặn Truy Cập Công Khai S3

> S3 Block Public Access — Chặn Truy Cập Công Khai: tập hợp 4 cấu hình bảo vệ bucket S3 khỏi bị vô tình mở ra công khai, ngay cả khi có bucket policy hoặc ACL cho phép.

## 📚 Mục Lục

1. [Tổng Quan Block Public Access](#1-tổng-quan-block-public-access)
2. [Bốn Cấu Hình Chi Tiết](#2-bốn-cấu-hình-chi-tiết)
3. [Cấp Độ Áp Dụng](#3-cấp-độ-áp-dụng)
4. [Cách Cấu Hình](#4-cách-cấu-hình)
5. [Khi Nào Cần Tắt](#5-khi-nào-cần-tắt)
6. [Best Practices](#6-best-practices)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan Block Public Access

### Vấn Đề Block Public Access Giải Quyết

```
Tình huống nguy hiểm:
Developer A tạo bucket với Block Public Access BẬT ✅
Developer B viết bucket policy cho phép s3:GetObject với Principal: "*"

Kết quả:
→ Bucket vẫn private vì Block Public Access chặn policy đó
→ Tránh được data breach do misconfiguration

Không có Block Public Access:
→ Bucket bị public ngay lập tức
→ Dữ liệu nhạy cảm bị lộ
```

### Block Public Access vs Bucket Policy

```
Block Public Access hoạt động TRƯỚC bucket policy:
┌──────────────────────────────────────┐
│  Block Public Access = ON            │
│  ↓                                   │
│  Bucket Policy: Allow Principal: "*" │
│  ↓                                   │
│  Kết quả: BỊ CHẶN — không public     │
└──────────────────────────────────────┘

Block Public Access = OFF:
┌──────────────────────────────────────┐
│  Block Public Access = OFF           │
│  ↓                                   │
│  Bucket Policy: Allow Principal: "*" │
│  ↓                                   │
│  Kết quả: PUBLIC — ai cũng đọc được  │
└──────────────────────────────────────┘
```

---

## 2. Bốn Cấu Hình Chi Tiết

### Tổng Quan Bốn Cấu Hình

| Cấu Hình | Tên Ngắn | Chức Năng |
|----------|---------|-----------|
| `BlockPublicAcls` | Block ACL mới | Chặn tạo ACL public mới |
| `IgnorePublicAcls` | Bỏ qua ACL cũ | Bỏ qua mọi ACL public hiện tại |
| `BlockPublicPolicy` | Block policy public | Chặn bucket policy cho phép public |
| `RestrictPublicBuckets` | Hạn chế bucket public | Ngăn anonymous và cross-account |

### 2.1. BlockPublicAcls — Chặn ACL Public Mới

```
Khi BẬT:
→ Chặn mọi PUT Bucket ACL và PUT Object ACL nếu ACL đó cấp public access
→ Chặn tạo object mới với public ACL (public-read, public-read-write)
→ ACL public HIỆN TẠI vẫn tồn tại (không xóa cái cũ)

Khi TẮT:
→ Cho phép tạo ACL public mới
→ Rủi ro: developer vô tình upload với acl=public-read
```

**Ví dụ bị chặn khi BlockPublicAcls = ON:**
```bash
# Lệnh này sẽ BỊ TỪ CHỐI
aws s3api put-object-acl \
  --bucket my-bucket \
  --key sensitive-file.txt \
  --acl public-read

# Lỗi trả về:
# An error occurred (AccessDenied): Access Denied
```

### 2.2. IgnorePublicAcls — Bỏ Qua ACL Public Hiện Tại

```
Khi BẬT:
→ S3 bỏ qua tất cả ACL public khi đánh giá quyền truy cập
→ Object có ACL public-read vẫn private
→ Không xóa ACL, chỉ bỏ qua khi đánh giá
→ Bucket có ACL public-read-write vẫn private

Khi TẮT:
→ ACL public được áp dụng như bình thường
→ Object với acl=public-read có thể đọc bởi bất kỳ ai
```

### 2.3. BlockPublicPolicy — Chặn Bucket Policy Public

```
Khi BẬT:
→ Chặn PUT Bucket Policy nếu policy đó cấp public access
→ Chặn policy cho phép Principal: "*" (anonymous)
→ Policy PUBLIC HIỆN TẠI vẫn có hiệu lực
→ Chỉ chặn TẠO MỚI hoặc CẬP NHẬT policy public

Khi TẮT:
→ Cho phép tạo bucket policy với Principal: "*"
→ Rủi ro: static website vô tình trở thành open bucket
```

**Ví dụ bị chặn khi BlockPublicPolicy = ON:**
```bash
# aws s3api put-bucket-policy sẽ THẤT BẠI nếu policy có Principal: "*"
# Lỗi: InvalidBucketAclWithBlockPublicAccessError
```

### 2.4. RestrictPublicBuckets — Hạn Chế Bucket Public

```
Khi BẬT:
→ Nếu bucket có bucket policy cấp public access:
  - Chặn anonymous access (Principal: "*" cho object)
  - Chặn cross-account access qua policy (trừ AWS services)
→ Chỉ owner account và AWS services được truy cập
→ Hoạt động kể cả khi bucket đã có policy public

Khi TẮT:
→ Public bucket policy có hiệu lực đầy đủ
→ Anonymous user có thể truy cập nếu policy cho phép
```

### Bảng Tóm Tắt Toàn Bộ

```
┌─────────────────────┬──────────────────────────────────────────┐
│ Cấu hình            │ Tác Động                                  │
├─────────────────────┼──────────────────────────────────────────┤
│ BlockPublicAcls     │ Chặn TẠO MỚI ACL public                  │
│ IgnorePublicAcls    │ Bỏ qua ACL public ĐÃ CÓ và mới            │
│ BlockPublicPolicy   │ Chặn TẠO MỚI policy public               │
│ RestrictPublicBuckets│ Hạn chế TRUY CẬP dù policy đã public    │
└─────────────────────┴──────────────────────────────────────────┘

Để bảo vệ hoàn toàn: BẬT cả 4 cấu hình
```

---

## 3. Cấp Độ Áp Dụng

### 3.1. Account Level — Cấp Tài Khoản

```
Cài đặt ở account level ảnh hưởng đến TẤT CẢ bucket trong account.
Được ghi đè bởi cài đặt bucket-level nếu bucket-level nghiêm ngặt hơn.

Kiểm tra:
aws s3control get-public-access-block \
  --account-id 123456789012
```

### 3.2. Bucket Level — Cấp Bucket

```
Cài đặt riêng cho từng bucket.
Nếu account-level BẬT, bucket-level không thể TẮT.
Bucket-level có thể nghiêm ngặt HƠN account-level.

Kiểm tra:
aws s3api get-public-access-block \
  --bucket my-bucket
```

### 3.3. Organization Level (qua AWS Organizations + SCPs)

```json
// SCP — Service Control Policy ngăn tắt Block Public Access
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RequireS3BlockPublicAccess",
      "Effect": "Deny",
      "Action": [
        "s3:PutBucketPublicAccessBlock",
        "s3:DeletePublicAccessBlock"
      ],
      "Resource": "*",
      "Condition": {
        "ArnNotLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:role/InfraAdmin"
        }
      }
    }
  ]
}
```

---

## 4. Cách Cấu Hình

### 4.1. AWS Console

```
1. Mở S3 Console → chọn bucket
2. Tab "Permissions" (Quyền)
3. Phần "Block public access (bucket settings)"
4. Nhấn "Edit"
5. Bật tất cả 4 checkboxes
6. Nhấn "Save changes"
7. Xác nhận bằng cách gõ "confirm"
```

### 4.2. AWS CLI — Cho Một Bucket

```bash
# Bật tất cả 4 cấu hình
aws s3api put-public-access-block \
  --bucket my-bucket \
  --public-access-block-configuration \
    BlockPublicAcls=true,\
    IgnorePublicAcls=true,\
    BlockPublicPolicy=true,\
    RestrictPublicBuckets=true

# Kiểm tra cấu hình hiện tại
aws s3api get-public-access-block --bucket my-bucket

# Kiểm tra nhiều bucket (script đơn giản)
for bucket in $(aws s3 ls | awk '{print $3}'); do
  echo "=== $bucket ==="
  aws s3api get-public-access-block --bucket $bucket 2>/dev/null || echo "No config set"
done
```

### 4.3. AWS CLI — Cấp Account

```bash
# Bật cho toàn bộ account
aws s3control put-public-access-block \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --public-access-block-configuration \
    BlockPublicAcls=true,\
    IgnorePublicAcls=true,\
    BlockPublicPolicy=true,\
    RestrictPublicBuckets=true

# Kiểm tra
aws s3control get-public-access-block \
  --account-id $(aws sts get-caller-identity --query Account --output text)
```

### 4.4. CloudFormation — Cơ Sở Hạ Tầng Dạng Code

```yaml
Resources:
  SecureBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-secure-bucket
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        IgnorePublicAcls: true
        BlockPublicPolicy: true
        RestrictPublicBuckets: true
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: aws:kms
              KMSMasterKeyID: !Ref MyKMSKey
```

### 4.5. Terraform

```hcl
resource "aws_s3_bucket_public_access_block" "example" {
  bucket = aws_s3_bucket.example.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

---

## 5. Khi Nào Cần Tắt

### Trường Hợp Hợp Lệ Cần Tắt

| Use Case | Cấu Hình Cần Tắt | Giải Thích |
|----------|-----------------|-----------|
| S3 static website hosting | BlockPublicPolicy + RestrictPublicBuckets | Cần bucket policy public cho website |
| Public software downloads | BlockPublicPolicy + RestrictPublicBuckets | Binary/package cần anonymous download |
| CloudFront với OAI cũ | Không cần tắt | OAI không phải anonymous |
| CloudFront với OAC | Không cần tắt | OAC dùng service principal |

**Lưu ý:** CloudFront OAC — Origin Access Control không cần tắt Block Public Access vì dùng signed requests từ CloudFront, không phải anonymous access.

### Quy Trình An Toàn Khi Tắt

```
1. Ghi lại lý do business cần tắt
2. Chỉ tắt đúng cấu hình cần thiết (không tắt hết)
3. Chỉ áp dụng cho bucket cụ thể (không tắt account level)
4. Bật S3 Access Analyzer để theo dõi
5. Bật CloudTrail data events để audit
6. Review định kỳ bằng AWS Config rule
```

---

## 6. Best Practices

### Bật Theo Mặc Định Cho Tất Cả Bucket

```bash
# Script kiểm tra bucket nào chưa bật Block Public Access
aws s3 ls | awk '{print $3}' | while read bucket; do
  result=$(aws s3api get-public-access-block --bucket "$bucket" 2>/dev/null)
  if [ $? -ne 0 ] || echo "$result" | grep -q '"false"'; then
    echo "WARNING: $bucket cần kiểm tra Block Public Access"
  fi
done
```

### AWS Config Rule Tự Động Kiểm Tra

```bash
# Bật AWS Config rule kiểm tra Block Public Access
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-public-access-prohibited",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_PUBLIC_ACCESS_PROHIBITED"
    }
  }'
```

### Dùng AWS Security Hub

Security Hub — Trung Tâm Bảo Mật tự động kiểm tra theo chuẩn CIS — Center for Internet Security với rule `[S3.1] S3 Block Public Access setting should be enabled`.

---

## 7. Câu Hỏi Phỏng Vấn

**Q: Block Public Access có thể bị vượt qua không?**

A: Có thể nếu ai đó có quyền `s3:PutBucketPublicAccessBlock` tắt đi. Để ngăn chặn, dùng SCP ở Organization level để Deny action này với tất cả trừ InfraAdmin role. Ngoài ra, S3 Access Analyzer và AWS Config rule `S3_BUCKET_PUBLIC_ACCESS_PROHIBITED` giám sát liên tục.

---

**Q: Sự khác biệt giữa BlockPublicAcls và IgnorePublicAcls?**

A: `BlockPublicAcls` ngăn tạo mới ACL public nhưng không ảnh hưởng ACL public đã tồn tại. `IgnorePublicAcls` bỏ qua tất cả ACL public khi đánh giá quyền — kể cả ACL cũ và mới. Để bảo vệ hoàn toàn nên bật cả hai: một cái ngăn tạo mới, cái kia vô hiệu hóa cái đã có.

---

**Q: Tại sao nên bật Block Public Access ở account level thay vì bucket level?**

A: Account level cung cấp guardrail — rào chắn bảo vệ mặc định cho mọi bucket mới tạo. Bucket level chỉ áp dụng cho bucket cụ thể — bucket mới tạo không được bảo vệ nếu quên cấu hình. Kết hợp cả hai: bật account level làm mặc định, và dùng SCP để ngăn tắt account-level setting.

---

**Cập Nhật:** 2026-05-16
