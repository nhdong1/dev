# KMS Key Types — Các Loại Khóa Trong AWS KMS

> AWS KMS (Key Management Service — Dịch Vụ Quản Lý Khóa) cung cấp ba loại khóa mã hóa với mức độ kiểm soát khác nhau: AWS Owned Keys, AWS Managed Keys, và Customer Managed Keys (CMKs — Khóa Do Khách Hàng Quản Lý). Ngoài ra còn có Data Keys (Khóa Dữ Liệu) dùng trực tiếp để mã hóa dữ liệu.

---

## 📚 Mục Lục

1. [Tổng Quan Ba Loại KMS Key](#tổng-quan)
2. [AWS Owned Keys](#aws-owned-keys)
3. [AWS Managed Keys](#aws-managed-keys)
4. [Customer Managed Keys (CMK)](#customer-managed-keys)
5. [Data Keys](#data-keys)
6. [Multi-Region Keys](#multi-region-keys)
7. [Asymmetric Keys](#asymmetric-keys)
8. [Thực Hành CLI](#thực-hành-cli)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan

### Phân Loại Khóa KMS

```
KMS Key Types
│
├── AWS Owned Keys (Khóa Sở Hữu Bởi AWS)
│   ├── AWS sở hữu, quản lý trong account của AWS (không phải của bạn)
│   ├── Ví dụ: S3 SSE-S3, DynamoDB default encryption
│   └── Không thể xem, quản lý, hay audit
│
├── AWS Managed Keys (Khóa Quản Lý Bởi AWS)
│   ├── AWS tạo và quản lý trong account của bạn
│   ├── Tên format: aws/<service> (ví dụ: aws/s3, aws/rds)
│   ├── Tự động rotate mỗi 1 năm
│   └── Có thể xem key metadata và CloudTrail logs
│
└── Customer Managed Keys / CMK (Khóa Do Khách Hàng Quản Lý)
    ├── Bạn tạo và kiểm soát hoàn toàn
    ├── Có thể tùy chỉnh key policy, rotation, deletion
    ├── $1/key/tháng + $0.03/10,000 API calls
    └── Hỗ trợ Multi-Region, Custom Key Store (CloudHSM)
```

### Bảng So Sánh Đầy Đủ

| Đặc Điểm | AWS Owned | AWS Managed | Customer Managed |
|---|---|---|---|
| **Nơi lưu trữ** | Account AWS | Account bạn | Account bạn |
| **Chi phí key** | Miễn phí | Miễn phí | $1/key/tháng |
| **Chi phí API** | Miễn phí | Miễn phí* | $0.03/10k calls |
| **Xem metadata** | Không | Có (read-only) | Có (full) |
| **Key policy** | Không | Không tùy chỉnh | Tùy chỉnh hoàn toàn |
| **Rotation** | AWS tự quản lý | Tự động 1 năm | Tự chọn (manual/auto) |
| **Xóa key** | Không thể | Không thể | Có (7–30 ngày delay) |
| **Disable key** | Không thể | Không thể | Có |
| **Import key material** | Không | Không | Có |
| **Multi-Region** | Không | Không | Có |
| **CloudTrail audit** | Không | Có | Có |
| **Cross-account** | Không | Không | Có |

*AWS Managed Keys tính phí API nếu vượt free tier

---

## AWS Owned Keys

### Đặc Điểm

AWS Owned Keys là khóa **AWS sở hữu và vận hành hoàn toàn** — không hiển thị trong AWS account của bạn. Chúng tồn tại trong hạ tầng nội bộ của AWS.

```
Dịch vụ AWS
  └── S3 SSE-S3 (mặc định)
  └── SQS SSE
  └── CloudWatch Logs (encrypted by default)
  └── AWS X-Ray
  └── ... nhiều dịch vụ khác
```

### Khi Nào Được Dùng

- Bạn không chỉ định key khi bật encryption → dịch vụ có thể dùng AWS Owned Key
- Đây là mức bảo vệ cơ bản nhất

### Hạn Chế

- Không audit được — không có CloudTrail log cho thao tác dùng AWS Owned Key
- Không thể revoke quyền truy cập tại tầng key
- Không đáp ứng yêu cầu compliance cần kiểm soát key

---

## AWS Managed Keys

### Đặc Điểm

AWS tạo khóa trong account của bạn khi bạn bật encryption cho dịch vụ lần đầu tiên. Key được tự động rotate mỗi **365 ngày**.

```bash
# Liệt kê AWS Managed Keys trong account
aws kms list-keys --query 'Keys[*].KeyId' --output text

# Xem chi tiết một AWS Managed Key
aws kms describe-key --key-id alias/aws/s3
```

Ví dụ output:

```json
{
  "KeyMetadata": {
    "AWSAccountId": "123456789012",
    "KeyId": "1234abcd-12ab-34cd-56ef-1234567890ab",
    "Arn": "arn:aws:kms:us-east-1:123456789012:key/1234abcd-...",
    "Description": "Default master key that protects my S3 objects",
    "KeyUsage": "ENCRYPT_DECRYPT",
    "KeyState": "Enabled",
    "Origin": "AWS_KMS",
    "KeyManager": "AWS",
    "CustomerMasterKeySpec": "SYMMETRIC_DEFAULT"
  }
}
```

### Tên Alias Chuẩn

| Dịch Vụ | AWS Managed Key Alias |
|---|---|
| S3 | `alias/aws/s3` |
| RDS | `alias/aws/rds` |
| EBS | `alias/aws/ebs` |
| DynamoDB | `alias/aws/dynamodb` |
| Secrets Manager | `alias/aws/secretsmanager` |
| CloudWatch Logs | `alias/aws/logs` |
| Systems Manager | `alias/aws/ssm` |
| ECR | `alias/aws/ecr` |

### Giới Hạn

- Không thể thay đổi key policy
- Không thể disable key
- Không thể dùng từ account khác (no cross-account)
- Không hỗ trợ import key material từ bên ngoài

---

## Customer Managed Keys (CMK)

### Tạo CMK

#### Qua Console

```
KMS → Customer managed keys → Create key
  → Key type: Symmetric / Asymmetric
  → Key usage: Encrypt and decrypt / Sign and verify
  → Key material origin: KMS / External / CloudHSM
  → Alias: my-app-encryption-key
  → Key administrator: <IAM roles/users>
  → Key usage permissions: <roles/users có thể dùng key>
```

#### Qua CLI

```bash
# Tạo symmetric CMK (AES-256-GCM)
aws kms create-key \
  --description "Production database encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --origin AWS_KMS \
  --tags TagKey=Environment,TagValue=prod TagKey=Team,TagValue=backend

# Tạo alias dễ nhớ
aws kms create-alias \
  --alias-name alias/prod-db-key \
  --target-key-id <key-id>
```

### Cấu Trúc Key ARN (Amazon Resource Name — Tên Tài Nguyên Amazon)

```
arn:aws:kms:<region>:<account-id>:key/<key-id>

Ví dụ:
arn:aws:kms:us-east-1:123456789012:key/mrk-1234abcd5678efgh
```

Với Multi-Region Key (MRK):
```
Primary key:  arn:aws:kms:us-east-1:123456789012:key/mrk-abc123
Replica key:  arn:aws:kms:eu-west-1:123456789012:key/mrk-abc123
                                                      ↑ cùng key ID
```

### Key State (Trạng Thái Khóa)

```
Enabled     → Có thể dùng để encrypt/decrypt
Disabled    → Không thể dùng; dữ liệu đã mã hóa vẫn có thể giải mã sau khi enable lại
Pending Deletion → Lên lịch xóa (7–30 ngày); không thể encrypt/decrypt
Unavailable → Key material không có (Custom Key Store bị disconnect)
```

```bash
# Disable key (giữ key, không dùng được)
aws kms disable-key --key-id alias/prod-db-key

# Enable lại
aws kms enable-key --key-id alias/prod-db-key

# Lên lịch xóa (tối thiểu 7 ngày, mặc định 30 ngày)
aws kms schedule-key-deletion \
  --key-id alias/prod-db-key \
  --pending-window-in-days 14

# Hủy lịch xóa
aws kms cancel-key-deletion --key-id alias/prod-db-key
```

### Key Rotation (Xoay Vòng Khóa)

```bash
# Bật automatic rotation (xoay mỗi 365 ngày)
aws kms enable-key-rotation --key-id alias/prod-db-key

# Kiểm tra trạng thái rotation
aws kms get-key-rotation-status --key-id alias/prod-db-key

# On-demand rotation (AWS hỗ trợ từ 2023)
aws kms rotate-key-on-demand --key-id alias/prod-db-key
```

**Quan trọng:** KMS giữ lại tất cả các version khóa cũ để giải mã dữ liệu đã mã hóa bằng version cũ. Rotation **không làm mất khả năng giải mã** dữ liệu cũ.

---

## Data Keys

### Khái Niệm

Data Keys (Khóa Dữ Liệu) là khóa **đối xứng AES-256** được tạo bởi KMS nhưng **trả về cho client** để mã hóa dữ liệu trực tiếp. Đây là nền tảng của Envelope Encryption.

```bash
# Tạo data key (KMS trả về cả plaintext và encrypted version)
aws kms generate-data-key \
  --key-id alias/prod-db-key \
  --key-spec AES_256

# Output:
# {
#   "CiphertextBlob": "<base64-encoded encrypted DEK>",
#   "Plaintext": "<base64-encoded plaintext DEK>",  ← dùng để mã hóa, xong thì xóa khỏi memory
#   "KeyId": "arn:aws:kms:..."
# }
```

### Tại Sao Không Mã Hóa Trực Tiếp Bằng CMK?

```
Vấn đề với mã hóa trực tiếp CMK:
  ├── KMS API có giới hạn: tối đa 4KB per request
  ├── Mọi encrypt/decrypt phải gọi KMS API → latency cao
  ├── Chi phí API tăng theo lượng dữ liệu
  └── Network round-trip cho mỗi thao tác

Giải pháp Envelope Encryption:
  ├── Chỉ gọi KMS 1 lần để lấy DEK
  ├── DEK mã hóa data locally (nhanh, không giới hạn kích thước)
  ├── Chỉ lưu Encrypted DEK (vài chục bytes)
  └── Giải mã: gọi KMS 1 lần để giải mã DEK → dùng DEK giải mã data
```

### Data Key Without Plaintext

```bash
# Tạo data key nhưng KHÔNG trả về plaintext
# Dùng khi chỉ cần lưu encrypted DEK (ví dụ: lưu vào database)
aws kms generate-data-key-without-plaintext \
  --key-id alias/prod-db-key \
  --key-spec AES_256
```

---

## Multi-Region Keys

### Khái Niệm

Multi-Region Keys (Khóa Đa Vùng) là CMK có cùng **key material** và **key ID** được replicate (sao chép) sang nhiều AWS regions.

```
Primary Region (us-east-1):
  mrk-1234abcd5678efgh

Replica Regions:
  eu-west-1:  mrk-1234abcd5678efgh  (cùng key ID)
  ap-southeast-1: mrk-1234abcd5678efgh
```

### Use Cases (Trường Hợp Sử Dụng)

```
1. Active-Active Multi-Region Architecture
   └── Encrypt ở us-east-1, Decrypt ở eu-west-1 (disaster recovery)

2. Global Client-Side Encryption
   └── Ứng dụng global mã hóa/giải mã không cần cross-region API call

3. DynamoDB Global Tables
   └── Dữ liệu mã hóa được replicate giữa regions
```

```bash
# Tạo primary key
aws kms create-key \
  --description "Multi-region primary key" \
  --multi-region true

# Replicate sang region khác
aws kms replicate-key \
  --key-id arn:aws:kms:us-east-1:123456789012:key/mrk-abc123 \
  --replica-region eu-west-1
```

---

## Asymmetric Keys

### Khi Nào Dùng

| Use Case | Key Type |
|---|---|
| Encrypt/Decrypt dữ liệu | Symmetric (RSA không khuyến nghị cho data encryption) |
| Digital signatures (Chữ Ký Số) | Asymmetric RSA hoặc ECC |
| Public key cryptography | Asymmetric RSA 2048/3072/4096 |
| TLS handshake verification | Asymmetric RSA hoặc ECC |

```bash
# Tạo asymmetric key cho signing
aws kms create-key \
  --key-usage SIGN_VERIFY \
  --key-spec RSA_2048

# Ký dữ liệu
aws kms sign \
  --key-id alias/signing-key \
  --message fileb://message.txt \
  --message-type RAW \
  --signing-algorithm RSASSA_PKCS1_V1_5_SHA_256

# Verify chữ ký
aws kms verify \
  --key-id alias/signing-key \
  --message fileb://message.txt \
  --message-type RAW \
  --signing-algorithm RSASSA_PKCS1_V1_5_SHA_256 \
  --signature fileb://signature.bin
```

---

## Thực Hành CLI

### Lab 1: Mã Hóa/Giải Mã Cơ Bản Với CMK

```bash
# 1. Tạo CMK
KEY_ID=$(aws kms create-key \
  --description "Lab encryption key" \
  --query 'KeyMetadata.KeyId' \
  --output text)

aws kms create-alias \
  --alias-name alias/lab-key \
  --target-key-id $KEY_ID

# 2. Mã hóa dữ liệu (tối đa 4KB plaintext)
ENCRYPTED=$(aws kms encrypt \
  --key-id alias/lab-key \
  --plaintext "Hello, KMS!" \
  --query CiphertextBlob \
  --output text)

echo "Encrypted: $ENCRYPTED"

# 3. Giải mã
DECRYPTED=$(aws kms decrypt \
  --ciphertext-blob fileb://<(echo "$ENCRYPTED" | base64 -d) \
  --query Plaintext \
  --output text | base64 -d)

echo "Decrypted: $DECRYPTED"
```

### Lab 2: Generate và Dùng Data Key

```bash
# 1. Tạo data key
DATA_KEY=$(aws kms generate-data-key \
  --key-id alias/lab-key \
  --key-spec AES_256)

PLAINTEXT_DEK=$(echo $DATA_KEY | jq -r '.Plaintext')
ENCRYPTED_DEK=$(echo $DATA_KEY | jq -r '.CiphertextBlob')

# 2. Dùng plaintext DEK để mã hóa với openssl
echo "Sensitive data" | openssl enc -aes-256-cbc \
  -K $(echo $PLAINTEXT_DEK | base64 -d | xxd -p -c 256) \
  -iv 00000000000000000000000000000000 \
  -out data.enc

# 3. Xóa plaintext DEK khỏi memory (quan trọng!)
unset PLAINTEXT_DEK

# 4. Lưu encrypted DEK cùng data.enc
echo $ENCRYPTED_DEK > data.enc.key

# --- Sau này để giải mã ---
# 5. Giải mã DEK bằng KMS
DECRYPTED_DEK=$(aws kms decrypt \
  --ciphertext-blob fileb://<(cat data.enc.key | base64 -d) \
  --query Plaintext --output text)

# 6. Dùng DEK giải mã data
openssl enc -aes-256-cbc -d \
  -K $(echo $DECRYPTED_DEK | base64 -d | xxd -p -c 256) \
  -iv 00000000000000000000000000000000 \
  -in data.enc
```

---

## Câu Hỏi Phỏng Vấn

**Q: Phân biệt AWS Owned, AWS Managed, và Customer Managed Keys?**

> AWS Owned Keys thuộc sở hữu của AWS, không hiển thị trong account của bạn, miễn phí nhưng không audit được. AWS Managed Keys AWS tạo trong account bạn, tự rotate hàng năm, có thể audit qua CloudTrail nhưng không tùy chỉnh được. Customer Managed Keys (CMK) bạn tạo và kiểm soát hoàn toàn: tùy chỉnh key policy, rotation, disable, xóa, dùng cross-account, import key material từ ngoài — tốn $1/key/tháng.

**Q: Tại sao KMS không cho mã hóa dữ liệu lớn trực tiếp?**

> KMS giới hạn 4KB per encrypt request vì mục đích của KMS là quản lý khóa (key management), không phải data encryption engine. Thay vào đó, dùng Envelope Encryption: KMS tạo Data Key (DEK), DEK mã hóa data locally, CMK mã hóa DEK. Chỉ DEK nhỏ (32 bytes) mới qua KMS.

**Q: Key Rotation trong KMS có làm mất khả năng giải mã dữ liệu cũ không?**

> Không. KMS giữ lại tất cả các version khóa cũ. Mỗi ciphertext (bản mã) chứa thông tin về key version đã dùng; khi giải mã, KMS tự động dùng đúng version. Rotation chỉ ảnh hưởng đến dữ liệu mã hóa mới.

**Q: Multi-Region Key khác CMK thường như thế nào?**

> Multi-Region Key (MRK) có cùng key material và key ID được replicate sang nhiều regions. Điều này cho phép encrypt ở một region và decrypt ở region khác mà không cần re-encrypt. Hữu ích cho Disaster Recovery (Khắc Phục Thảm Họa), global applications, và DynamoDB Global Tables. CMK thường chỉ tồn tại trong một region.

---

## Tóm Tắt

```
Chọn loại key:
  ├── Không cần audit, compliance tối thiểu → AWS Owned Keys (mặc định nhiều dịch vụ)
  ├── Cần audit, không cần tùy chỉnh        → AWS Managed Keys
  ├── Cần kiểm soát, audit, cross-account   → Customer Managed Keys (CMK)
  ├── Mã hóa dữ liệu lớn                   → CMK + Envelope Encryption với Data Keys
  ├── Cần encrypt/decrypt nhiều regions     → Multi-Region Keys
  ├── Digital signing                       → Asymmetric CMK (RSA/ECC)
  └── FIPS 140-2 Level 3 compliance         → CMK + Custom Key Store (CloudHSM)
```

---

**Tiếp Theo:** [2-envelope-encryption.md](2-envelope-encryption.md) — Cơ Chế Mã Hóa Phong Bì Chi Tiết

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
