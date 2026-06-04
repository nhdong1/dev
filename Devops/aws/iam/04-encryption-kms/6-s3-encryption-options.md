# S3 Encryption Options — Các Tùy Chọn Mã Hóa S3

> Amazon S3 (Simple Storage Service — Dịch Vụ Lưu Trữ Đơn Giản) hỗ trợ bốn phương thức mã hóa khác nhau, mỗi phương thức phù hợp với nhu cầu kiểm soát, chi phí và compliance khác nhau: SSE-S3 (Server-Side Encryption with S3 — Mã Hóa Phía Máy Chủ Bằng Khóa S3), SSE-KMS (với AWS KMS), SSE-C (Customer-Provided Keys — Khóa Do Khách Hàng Cung Cấp), và CSE (Client-Side Encryption — Mã Hóa Phía Client).

---

## 📚 Mục Lục

1. [Tổng Quan Bốn Phương Thức](#tổng-quan)
2. [SSE-S3 — Server-Side Encryption với S3](#sse-s3)
3. [SSE-KMS — Server-Side Encryption với KMS](#sse-kms)
4. [SSE-C — Customer-Provided Keys](#sse-c)
5. [CSE — Client-Side Encryption](#cse)
6. [Bảng So Sánh Đầy Đủ](#so-sánh)
7. [Mã Hóa In-Transit](#in-transit)
8. [Chính Sách Bắt Buộc Mã Hóa](#enforce-encryption)
9. [Bucket-Level Default Encryption](#default-encryption)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan

### Vị Trí Mã Hóa

```
Client ──────── Internet ──────── S3 API ──────── S3 Storage
  │                                   │               │
  │  CSE: mã hóa tại đây             │               │
  │  (trước khi gửi đi)              │               │
  │                                  │               │
  │                       SSE-C: key gửi kèm,        │
  │                       S3 mã hóa tại đây          │
  │                                                   │
  │                       SSE-S3, SSE-KMS:            │
  │                       S3 mã hóa, lưu tại đây ────┘
```

### Luồng Quyết Định Chọn Phương Thức

```
Cần mã hóa S3 objects?
│
├── Kiểm soát key (key management)?
│   ├── AWS quản lý hoàn toàn, miễn phí → SSE-S3
│   ├── KMS CMK, audit trail, IAM control → SSE-KMS
│   ├── Tự mang key, không lưu key ở AWS → SSE-C
│   └── Mã hóa trước khi upload, AWS không thấy → CSE
│
├── Compliance yêu cầu gì?
│   ├── FIPS 140-2 Level 3 → CSE với CloudHSM key
│   ├── Audit mọi decrypt operation → SSE-KMS
│   └── Đơn giản, không yêu cầu đặc biệt → SSE-S3
│
└── Ai kiểm soát key lifecycle?
    ├── Chỉ AWS → SSE-S3 hoặc SSE-KMS với AWS Managed
    ├── Bạn và AWS → SSE-KMS với CMK
    └── Chỉ bạn → SSE-C hoặc CSE
```

---

## SSE-S3

### Cơ Chế

S3 dùng **AES-256** để mã hóa mỗi object với một unique key. Key đó được mã hóa bằng master key do S3 quản lý và rotate định kỳ. Bạn không cần quản lý gì cả.

```
PUT Object:
  S3 nhận object → tạo unique object key (AES-256)
  → mã hóa object → xóa object key plaintext
  → lưu Encrypted Object + Encrypted Object Key
  (master key do S3 lưu và rotate tự động)

GET Object:
  S3 đọc Encrypted Object + Encrypted Object Key
  → dùng master key giải mã object key
  → giải mã object → trả về plaintext object
```

### Bật SSE-S3

```bash
# Upload với SSE-S3
aws s3api put-object \
  --bucket my-bucket \
  --key data/file.txt \
  --body file.txt \
  --server-side-encryption AES256

# Hoặc dùng s3 cp
aws s3 cp file.txt s3://my-bucket/data/file.txt \
  --sse AES256
```

```python
import boto3

s3 = boto3.client('s3')

# Upload với SSE-S3
s3.put_object(
    Bucket='my-bucket',
    Key='data/file.txt',
    Body=b'Hello World',
    ServerSideEncryption='AES256'
)
```

### Header HTTP

```
Request header: x-amz-server-side-encryption: AES256
Response header: x-amz-server-side-encryption: AES256
```

### Khi Nên Dùng SSE-S3?

```
✅ Mã hóa cơ bản không cần kiểm soát key
✅ Không có yêu cầu compliance đặc biệt
✅ Tối ưu chi phí (miễn phí, không API call KMS)
✅ Workload có throughput cao (không bị giới hạn KMS API)
✅ Static assets, media files, log archives

❌ Không có audit trail cho từng decrypt operation
❌ Không kiểm soát được ai decrypt
❌ AWS owned key — không đáp ứng yêu cầu "customer controls keys"
```

---

## SSE-KMS

### Cơ Chế

S3 dùng **KMS** để tạo Data Key (DEK) cho mỗi object (hoặc theo cấu hình). CMK bảo vệ DEK qua Envelope Encryption.

```
PUT Object (SSE-KMS):
  1. S3 → KMS: GenerateDataKey(KeyId=<CMK>)
  2. KMS → S3: {Plaintext DEK, Encrypted DEK}
  3. S3: AES-256 encrypt object với Plaintext DEK
  4. S3: Lưu Encrypted Object + Encrypted DEK (trong metadata)
  5. S3: Xóa Plaintext DEK

GET Object (SSE-KMS):
  1. S3 lấy Encrypted DEK từ metadata
  2. S3 → KMS: Decrypt(Encrypted DEK) [phải có kms:Decrypt permission]
  3. KMS → S3: Plaintext DEK
  4. S3: Giải mã object → trả về
  5. S3: Xóa Plaintext DEK
```

### Bật SSE-KMS

```bash
# Upload với AWS Managed Key (aws/s3)
aws s3api put-object \
  --bucket my-bucket \
  --key data/file.txt \
  --body file.txt \
  --server-side-encryption aws:kms

# Upload với CMK cụ thể
aws s3api put-object \
  --bucket my-bucket \
  --key data/file.txt \
  --body file.txt \
  --server-side-encryption aws:kms \
  --ssekms-key-id alias/prod-s3-key
```

```python
# SSE-KMS với CMK
s3.put_object(
    Bucket='my-bucket',
    Key='sensitive/ssn.json',
    Body=json_data.encode(),
    ServerSideEncryption='aws:kms',
    SSEKMSKeyId='arn:aws:kms:us-east-1:123456789012:key/mrk-abc123'
)
```

### SSE-KMS với Bucket Key

**S3 Bucket Key** là tính năng giảm chi phí KMS API:

```
Không có Bucket Key:
  Mỗi object upload/download = 1 KMS API call
  10,000 objects GET → 10,000 KMS calls

Với Bucket Key (S3 tạo key riêng cho bucket, bảo vệ bởi CMK):
  1 KMS call → Bucket Key → mã hóa nhiều DEKs locally
  Giảm 99% KMS API calls
  Tiết kiệm đáng kể chi phí cho bucket nhiều objects nhỏ
```

```bash
# Bật Bucket Key khi set default encryption
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "alias/prod-s3-key"
      },
      "BucketKeyEnabled": true
    }]
  }'
```

### Quyền IAM Cần Thiết Cho SSE-KMS

```json
// Role/User cần có để PUT objects
{
  "Effect": "Allow",
  "Action": [
    "kms:GenerateDataKey",
    "kms:DescribeKey"
  ],
  "Resource": "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"
}

// Role/User cần có để GET objects
{
  "Effect": "Allow",
  "Action": [
    "kms:Decrypt"
  ],
  "Resource": "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"
}
```

### Khi Nên Dùng SSE-KMS?

```
✅ Cần audit trail: ai decrypt object nào, khi nào (CloudTrail)
✅ Cần kiểm soát truy cập qua IAM + Key Policy
✅ Dữ liệu nhạy cảm: PII, PHI, tài chính
✅ Compliance: PCI-DSS, HIPAA
✅ Cần revoke quyền decrypt ngay lập tức (disable CMK)
✅ Cross-account bucket access với kiểm soát key

❌ Chi phí cao hơn (KMS API calls + optional key cost)
❌ Bị giới hạn KMS API rate (50,000 TPS mặc định)
❌ Thêm latency (~1-5ms mỗi request)
```

---

## SSE-C

### Cơ Chế

Bạn tự quản lý key và **gửi key kèm theo mỗi request**. S3 dùng key đó để mã hóa/giải mã rồi **không lưu key** — chỉ lưu HMAC (Hash-based Message Authentication Code — Mã Xác Thực Thông Điệp Dựa Trên Hàm Băm) của key để verify về sau.

```
PUT Object (SSE-C):
  Client gửi: Object + Customer Key (trong request header)
  S3: mã hóa object với customer key
  S3: tính HMAC(customer key) → lưu vào metadata
  S3: XÓA customer key — không lưu lại
  S3 lưu: Encrypted Object + HMAC

GET Object (SSE-C):
  Client gửi: GET request + Customer Key
  S3: verify HMAC(customer key) khớp với stored HMAC
  S3: giải mã object
  S3: trả về object, XÓA customer key
```

### Yêu Cầu Bắt Buộc: HTTPS

```
SSE-C PHẢI dùng HTTPS — không cho phép HTTP
Lý do: customer key truyền trong header request
       HTTP → key lộ trên network → vi phạm mục đích mã hóa
```

### Bật SSE-C

```python
import boto3
import os
import base64
import hashlib

s3 = boto3.client('s3')

# Tạo key 32 bytes (AES-256)
customer_key = os.urandom(32)  # Bạn phải lưu key này ở chỗ an toàn!
customer_key_md5 = base64.b64encode(
    hashlib.md5(customer_key).digest()
).decode()

# Upload với SSE-C
s3.put_object(
    Bucket='my-bucket',
    Key='data/file.txt',
    Body=b'Sensitive content',
    SSECustomerAlgorithm='AES256',
    SSECustomerKey=base64.b64encode(customer_key).decode(),
    SSECustomerKeyMD5=customer_key_md5
)

# Download (phải dùng ĐÚNG key)
response = s3.get_object(
    Bucket='my-bucket',
    Key='data/file.txt',
    SSECustomerAlgorithm='AES256',
    SSECustomerKey=base64.b64encode(customer_key).decode(),
    SSECustomerKeyMD5=customer_key_md5
)
```

### Lưu Ý Quan Trọng

```
⚠️ Nếu mất customer key → KHÔNG THỂ GIẢI MÃ object (vĩnh viễn)
⚠️ S3 không lưu key, không thể recover
⚠️ Không hỗ trợ từ S3 Console (phải dùng CLI/SDK)
⚠️ Không hỗ trợ qua Presigned URLs với SSE-C
⚠️ Không hỗ trợ S3 replication cross-region với SSE-C
```

### Khi Nên Dùng SSE-C?

```
✅ Yêu cầu nghiêm ngặt: AWS không được phép lưu key material
✅ Đã có HSM on-premises lưu key, muốn extend sang S3
✅ Key rotation tự quản lý hoàn toàn (không muốn KMS)
✅ Compliance: key phải tồn tại ngoài AWS

❌ Phức tạp trong quản lý key (phải tự backup, rotate)
❌ Không có audit trail từ AWS về key usage
❌ Rủi ro mất data nếu mất key
❌ Không tích hợp với AWS Console
```

---

## CSE

### Cơ Chế

Ứng dụng mã hóa dữ liệu **trước khi upload** lên S3. S3 chỉ lưu ciphertext — không bao giờ thấy plaintext.

```
CSE Upload:
  Application: Plaintext → Encrypt locally → Ciphertext
  Application → S3: PUT Ciphertext
  S3 lưu: chỉ Ciphertext (không biết nội dung)

CSE Download:
  Application → S3: GET Ciphertext
  S3 → Application: Ciphertext
  Application: Ciphertext → Decrypt locally → Plaintext
```

### AWS Encryption SDK Cho CSE

```python
import aws_encryption_sdk
import boto3

# Dùng KMS CMK làm key provider (nhưng mã hóa xảy ra client-side)
client = aws_encryption_sdk.EncryptionSDKClient()
master_key_provider = aws_encryption_sdk.StrictAwsKmsMasterKeyProvider(
    key_ids=["arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"]
)

s3 = boto3.client('s3')

# Mã hóa trước khi upload
plaintext_data = b"Top secret financial data"
ciphertext, header = client.encrypt(
    source=plaintext_data,
    key_provider=master_key_provider,
    encryption_context={
        "purpose": "financial-report",
        "classification": "confidential"
    }
)

# Upload ciphertext lên S3
s3.put_object(
    Bucket='my-bucket',
    Key='reports/q4-2026.enc',
    Body=ciphertext
)

# Download và giải mã
response = s3.get_object(Bucket='my-bucket', Key='reports/q4-2026.enc')
downloaded_ciphertext = response['Body'].read()

decrypted, decryptor_header = client.decrypt(
    source=downloaded_ciphertext,
    key_provider=master_key_provider
)
```

### CSE Với Non-AWS Key

```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

# Key từ on-premises HSM hoặc external KMS
encryption_key = fetch_key_from_hsm()  # 32 bytes

def encrypt_and_upload(plaintext: bytes, bucket: str, key: str):
    aesgcm = AESGCM(encryption_key)
    nonce = os.urandom(12)
    ciphertext = aesgcm.encrypt(nonce, plaintext, None)

    # Lưu: nonce + ciphertext
    payload = nonce + ciphertext

    s3.put_object(Bucket=bucket, Key=key, Body=payload)

def download_and_decrypt(bucket: str, key: str) -> bytes:
    response = s3.get_object(Bucket=bucket, Key=key)
    payload = response['Body'].read()

    nonce = payload[:12]
    ciphertext = payload[12:]

    aesgcm = AESGCM(encryption_key)
    return aesgcm.decrypt(nonce, ciphertext, None)
```

### Khi Nên Dùng CSE?

```
✅ AWS không được biết nội dung dữ liệu (highest security)
✅ Dùng key từ on-premises/external HSM
✅ Kiểm soát hoàn toàn encryption algorithm
✅ Regulatory: data must be encrypted BEFORE leaving organization's control
✅ Multi-cloud: cùng ciphertext có thể lưu ở nhiều cloud providers

❌ Phức tạp nhất trong triển khai và vận hành
❌ Không tích hợp với AWS features (S3 Select, Glacier, ...)
❌ Không thể tìm kiếm trong encrypted data
❌ Key management hoàn toàn thuộc trách nhiệm của bạn
```

---

## So Sánh

### Bảng So Sánh Đầy Đủ

| Tiêu Chí | SSE-S3 | SSE-KMS | SSE-C | CSE |
|---|---|---|---|---|
| **Nơi mã hóa** | S3 (server) | S3 (server) | S3 (server) | Client |
| **Ai giữ key** | AWS | AWS/Bạn | Bạn | Bạn |
| **AWS thấy plaintext** | Trong transit đến S3 | Trong transit | Trong transit | Không bao giờ |
| **Quản lý key** | AWS toàn bộ | KMS (shared/CMK) | Bạn toàn bộ | Bạn toàn bộ |
| **Chi phí** | Miễn phí | KMS API calls | Miễn phí | Miễn phí* |
| **Audit trail** | Không | CloudTrail ✅ | Không | Không (hoặc tự làm) |
| **Revoke quyền** | Không thể | Disable CMK ✅ | N/A | Tự quản lý |
| **Rotation** | AWS tự động | KMS (auto/manual) | Bạn tự làm | Bạn tự làm |
| **Key backup** | AWS | AWS/Bạn | Bạn phải tự backup | Bạn phải tự backup |
| **Cross-region replication** | ✅ | ✅ (cần key ở cả hai region) | ❌ | ✅ |
| **S3 Select** | ✅ | ✅ | ❌ | ❌ |
| **Presigned URL** | ✅ | ✅ | ❌ | ✅ |
| **Complexity** | Thấp | Trung bình | Cao | Rất cao |
| **AWS Console** | ✅ | ✅ | ❌ | ❌ |

*CSE có thể tốn nếu dùng external KMS

---

## In-Transit

### Mã Hóa Trong Quá Trình Truyền (Data in Transit)

Bất kể SSE method nào, dữ liệu **cũng phải được bảo vệ trong quá trình truyền**:

```bash
# Bắt buộc HTTPS cho S3 bucket (Bucket Policy)
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
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

### TLS Versions

```
S3 hỗ trợ TLS 1.0, 1.1, 1.2, 1.3
Best practice: Enforce TLS 1.2+ với Condition:

"Condition": {
  "NumericLessThan": {
    "s3:TlsVersion": "1.2"
  }
}
→ Deny requests dùng TLS version thấp hơn 1.2
```

---

## Enforce Encryption

### Bucket Policy Bắt Buộc SSE-KMS Với Specific Key

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedObjectUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    },
    {
      "Sid": "DenyWrongKMSKey",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption-aws-kms-key-id":
            "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"
        }
      }
    }
  ]
}
```

### Bắt Buộc SSE-S3 (AES256)

```json
{
  "Condition": {
    "StringNotEquals": {
      "s3:x-amz-server-side-encryption": "AES256"
    }
  }
}
```

---

## Default Encryption

### Cấu Hình Default Encryption Cho Bucket

```bash
# Set default encryption: SSE-S3
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

# Set default encryption: SSE-KMS với CMK + Bucket Key
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"
      },
      "BucketKeyEnabled": true
    }]
  }'

# Kiểm tra cấu hình hiện tại
aws s3api get-bucket-encryption --bucket my-bucket
```

### Terraform: Default Encryption

```hcl
resource "aws_s3_bucket" "sensitive_data" {
  bucket = "my-sensitive-bucket"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "example" {
  bucket = aws_s3_bucket.sensitive_data.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3_key.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_policy" "enforce_https" {
  bucket = aws_s3_bucket.sensitive_data.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "DenyHTTP"
      Effect    = "Deny"
      Principal = "*"
      Action    = "s3:*"
      Resource  = [
        aws_s3_bucket.sensitive_data.arn,
        "${aws_s3_bucket.sensitive_data.arn}/*"
      ]
      Condition = {
        Bool = {
          "aws:SecureTransport" = "false"
        }
      }
    }]
  })
}
```

---

## Câu Hỏi Phỏng Vấn

**Q: Phân biệt SSE-S3, SSE-KMS, SSE-C và CSE?**

> **SSE-S3**: AWS quản lý toàn bộ key, tự động rotate, miễn phí, không audit trail — dùng cho dữ liệu không nhạy cảm. **SSE-KMS**: dùng KMS CMK, có CloudTrail audit, kiểm soát IAM, có thể revoke — dùng cho PII/PHI/tài chính. **SSE-C**: bạn gửi key theo mỗi request, S3 không lưu key — dùng khi AWS không được lưu key, nhưng phức tạp và không có Console support. **CSE**: mã hóa trước khi upload, AWS không bao giờ thấy plaintext — highest security, phức tạp nhất.

**Q: S3 Bucket Key là gì? Nó giải quyết vấn đề gì?**

> Bucket Key là intermediate key do S3 tạo, được bảo vệ bởi CMK. Thay vì mỗi object phải gọi KMS (1 API call/object), S3 gọi KMS 1 lần để lấy Bucket Key, sau đó dùng Bucket Key locally để tạo DEK cho từng object. Điều này giảm số lượng KMS API calls lên đến 99%, giảm chi phí và latency đáng kể cho bucket có nhiều objects nhỏ. Nhược điểm nhỏ: granularity của audit giảm (không biết chính xác object nào được decrypt).

**Q: Tại sao SSE-C yêu cầu HTTPS bắt buộc?**

> Với SSE-C, customer key được gửi trong HTTP request header (`x-amz-server-side-encryption-customer-key`). Nếu dùng HTTP (không mã hóa), bất kỳ ai có thể nghe lén (man-in-the-middle — kẻ tấn công ở giữa) trên network sẽ thấy customer key trong plaintext, hoàn toàn phá vỡ mục đích mã hóa. HTTPS/TLS bảo vệ key trong quá trình truyền đến S3.

**Q: Bucket Policy bắt buộc mã hóa SSE-KMS hoạt động như thế nào?**

> Thêm `Deny` statement với điều kiện `"s3:x-amz-server-side-encryption" != "aws:kms"` — bất kỳ PUT request nào không kèm header SSE-KMS đều bị từ chối. Có thể thêm điều kiện thứ hai kiểm tra `s3:x-amz-server-side-encryption-aws-kms-key-id` để bắt buộc phải dùng đúng CMK (không cho phép dùng AWS Managed Key hoặc CMK khác). Kết hợp với Default Encryption để đảm bảo objects luôn được mã hóa đúng cách.

---

## Tóm Tắt

```
S3 Encryption — Chọn Nhanh:

SSE-S3   → Đơn giản, miễn phí, AWS quản lý key — dữ liệu không nhạy cảm cao
SSE-KMS  → Audit, IAM control, revoke — PII/PHI/Tài chính (khuyến nghị production)
SSE-C    → Bạn giữ key, phức tạp — khi AWS không được phép lưu key material
CSE      → Mã hóa trước upload, tối đa bảo mật — khi AWS không được thấy data

Luôn nhớ:
  ├── Bật HTTPS (aws:SecureTransport) cho tất cả buckets nhạy cảm
  ├── Dùng Bucket Key cho SSE-KMS để giảm chi phí KMS
  ├── Set Default Encryption để không bỏ sót object nào
  └── SSE-C không hỗ trợ cross-region replication
```

---

**Trước Đó:** [5-cloudhsm.md](5-cloudhsm.md)
**Module Tiếp Theo:** [../05-secrets-certificates/README.md](../05-secrets-certificates/README.md) — Secrets & Certificates

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
