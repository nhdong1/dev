# Mã Hóa At-Rest — SSE-S3, SSE-KMS, SSE-C, CSE

> Encryption at-rest — Mã hóa khi lưu trữ: bảo vệ dữ liệu khi được lưu trên đĩa, ngăn chặn truy cập trái phép vào storage media vật lý hoặc snapshot.

## 📚 Mục Lục

1. [Tổng Quan Các Phương Pháp](#1-tổng-quan-các-phương-pháp)
2. [SSE-S3 — Server-Side Encryption với S3 Key](#2-sse-s3)
3. [SSE-KMS — Server-Side Encryption với KMS](#3-sse-kms)
4. [SSE-C — Server-Side Encryption với Customer Key](#4-sse-c)
5. [CSE — Client-Side Encryption](#5-cse)
6. [Mã Hóa EBS](#6-mã-hóa-ebs)
7. [Mã Hóa EFS](#7-mã-hóa-efs)
8. [So Sánh Toàn Diện](#8-so-sánh-toàn-diện)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan Các Phương Pháp

### Bốn Loại Mã Hóa S3

```
                    Dữ Liệu Gốc
                         │
          ┌──────────────┼──────────────┐
          │              │              │
     Server-Side    Server-Side    Client-Side
     Encryption     Encryption     Encryption
     (SSE-S3)       (SSE-KMS)        (CSE)
          │              │              │
     AWS quản lý   Bạn kiểm soát  Bạn mã hóa
     hoàn toàn     audit, rotate  trước khi gửi
                         │
                    SSE-C — bạn
                    cung cấp key
                    từng request
```

### Ma Trận Lựa Chọn Nhanh

| Tiêu Chí | SSE-S3 | SSE-KMS | SSE-C | CSE |
|----------|--------|---------|-------|-----|
| Đơn giản | ✅ Nhất | ✅ | ❌ | ❌ Phức tạp nhất |
| Audit trail | ❌ | ✅ CloudTrail | ❌ | ✅ Bạn tự quản lý |
| Key rotation | Tự động | Tự động/Thủ công | Bạn tự làm | Bạn tự làm |
| Compliance | Cơ bản | HIPAA, PCI, FedRAMP | ✅ | Cao nhất |
| Chi phí thêm | Không | Chi phí KMS | Không | Không |
| Kiểm soát key | Không | Có | Có | Hoàn toàn |

---

## 2. SSE-S3

### SSE-S3 — Server-Side Encryption with S3-Managed Keys — Mã Hóa Phía Server Với Key S3 Quản Lý

### Cơ Chế Hoạt Động

```
Client                    S3                      AES-256 Key
  │                        │                          │
  │── PutObject ──────────▶│                          │
  │                        │── Yêu cầu key mã hóa ──▶│
  │                        │◀── Trả về data key ──────│
  │                        │                          │
  │                        │ Mã hóa object với data key
  │                        │ Lưu object đã mã hóa
  │                        │ Xóa plaintext data key
  │                        │ Lưu encrypted data key cùng object
```

### Đặc Điểm

- Thuật toán: **AES-256** — Advanced Encryption Standard 256-bit
- AWS quản lý hoàn toàn key — không có visibility về key
- **Không tốn thêm chi phí**
- Transparent với client — client không cần làm gì thêm
- **Không có audit trail** về ai dùng key

### Cấu Hình

```bash
# Bật SSE-S3 làm default encryption
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "AES256"
        },
        "BucketKeyEnabled": false
      }
    ]
  }'

# Upload với SSE-S3 tường minh
aws s3 cp file.txt s3://my-bucket/ \
  --server-side-encryption AES256
```

### Bucket Policy Enforce SSE-S3

```json
{
  "Statement": [
    {
      "Sid": "DenyNonSSES3",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "AES256"
        }
      }
    }
  ]
}
```

---

## 3. SSE-KMS

### SSE-KMS — Server-Side Encryption with AWS KMS Keys — Mã Hóa Phía Server Với Key AWS KMS

### KMS — AWS Key Management Service — Dịch Vụ Quản Lý Khóa AWS

```
Client                    S3                         KMS
  │                        │                           │
  │── PutObject ──────────▶│                           │
  │                        │── GenerateDataKey(CMK) ──▶│
  │                        │◀── Plaintext DEK + ───────│
  │                        │    Encrypted DEK          │
  │                        │                           │
  │                        │ Mã hóa object với DEK
  │                        │ Lưu Encrypted DEK cùng object
  │                        │ Xóa Plaintext DEK
  │                        │                           │
  │── GetObject ──────────▶│                           │
  │                        │── Decrypt(Encrypted DEK) ▶│
  │                        │◀── Plaintext DEK ─────────│
  │                        │ Giải mã object
  │◀── Object plaintext ───│
```

**DEK — Data Encryption Key — Khóa Mã Hóa Dữ Liệu**: key ngẫu nhiên dùng để mã hóa từng object.
**CMK — Customer Master Key — Khóa Master Khách Hàng** (còn gọi là KMS key): key trong KMS dùng để mã hóa DEK.

### Loại KMS Key

| Loại | Mô Tả | Chi Phí | Use Case |
|------|--------|---------|---------|
| **AWS managed key** (aws/s3) | AWS tạo và quản lý | $0 | Default SSE-KMS đơn giản |
| **Customer managed key (CMK)** | Bạn tạo, kiểm soát | $1/month/key + $0.03/10K API calls | Cần audit, rotation, cross-account |
| **AWS owned key** | AWS sở hữu, không thấy | $0 | SSE-S3 ẩn phía sau |

### Bucket Key — Tối Ưu Chi Phí KMS

```
Không có Bucket Key:              Có Bucket Key (S3 Bucket Key):
PutObject A → KMS API call       PutObject A → KMS API call
PutObject B → KMS API call       PutObject B → Không cần KMS call
PutObject C → KMS API call       PutObject C → Không cần KMS call
→ 3 KMS API calls               → 1 KMS call (giảm ~99% chi phí KMS)

Cơ chế: S3 tạo một data key cấp bucket ngắn hạn từ CMK,
dùng key đó để tạo DEK cho từng object mà không gọi KMS liên tục.
```

### Cấu Hình SSE-KMS

```bash
# Tạo CMK
aws kms create-key \
  --description "S3 encryption key for data-bucket" \
  --key-usage ENCRYPT_DECRYPT

# Bật SSE-KMS với CMK và Bucket Key
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "aws:kms",
          "KMSMasterKeyID": "arn:aws:kms:ap-southeast-1:123456789012:key/mrk-xxx"
        },
        "BucketKeyEnabled": true
      }
    ]
  }'

# Upload với SSE-KMS
aws s3 cp file.txt s3://my-bucket/ \
  --server-side-encryption aws:kms \
  --server-side-encryption-aws-kms-key-id alias/my-s3-key
```

### Cross-Account SSE-KMS

```json
// KMS Key Policy — cho phép Account B dùng key
{
  "Statement": [
    {
      "Sid": "AllowCrossAccountUse",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::222222222222:role/AppRole"
      },
      "Action": [
        "kms:GenerateDataKey",
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    }
  ]
}
```

### KMS Key Rotation — Luân Phiên Khóa Tự Động

```bash
# Bật automatic rotation (hàng năm)
aws kms enable-key-rotation \
  --key-id arn:aws:kms:ap-southeast-1:123456789012:key/mrk-xxx

# Kiểm tra rotation status
aws kms get-key-rotation-status \
  --key-id arn:aws:kms:ap-southeast-1:123456789012:key/mrk-xxx
```

---

## 4. SSE-C

### SSE-C — Server-Side Encryption with Customer-Provided Keys — Mã Hóa Phía Server Với Key Khách Hàng Cung Cấp

### Cơ Chế Hoạt Động

```
Client                              S3
  │                                  │
  │── PutObject + Key (Base64) ─────▶│
  │   + Key MD5 checksum             │
  │                                  │ S3 dùng key này mã hóa object
  │                                  │ S3 lưu HMAC của key (không lưu key)
  │                                  │ S3 trả về ETag
  │◀── Xác nhận lưu ─────────────────│
  │                                  │
  │                                  │ S3 XÓA key khỏi memory
  │                                  │
  │── GetObject + Key (Base64) ─────▶│
  │                                  │ S3 verify HMAC của key
  │                                  │ S3 giải mã rồi trả về
  │◀── Object plaintext ─────────────│
```

**Quan trọng:** AWS không lưu key — mất key là mất dữ liệu vĩnh viễn.

### Cấu Hình

```bash
# Tạo random 256-bit key
KEY=$(openssl rand -base64 32)
KEY_MD5=$(echo -n "$KEY" | md5sum | cut -d' ' -f1 | xxd -r -p | base64)

# Upload với SSE-C
aws s3api put-object \
  --bucket my-bucket \
  --key secure-file.txt \
  --body secure-file.txt \
  --sse-customer-algorithm AES256 \
  --sse-customer-key "$KEY" \
  --sse-customer-key-md5 "$KEY_MD5"

# Download phải cung cấp đúng key
aws s3api get-object \
  --bucket my-bucket \
  --key secure-file.txt \
  --sse-customer-algorithm AES256 \
  --sse-customer-key "$KEY" \
  --sse-customer-key-md5 "$KEY_MD5" \
  output-file.txt
```

### Hạn Chế SSE-C

- **Chỉ qua HTTPS** — AWS từ chối request HTTP với SSE-C
- Không thể dùng qua AWS Console — chỉ CLI/SDK
- Không có audit trail về việc dùng key
- Không hỗ trợ S3 Batch Operations
- Client phải tự quản lý key securely

---

## 5. CSE

### CSE — Client-Side Encryption — Mã Hóa Phía Client

### Cơ Chế Hoạt Động

```
Client (ứng dụng của bạn)
  │
  │ 1. Tạo data key (DEK)
  │ 2. Mã hóa object bằng DEK
  │ 3. Mã hóa DEK bằng master key (KMS hoặc tự quản lý)
  │ 4. Upload ciphertext + encrypted DEK lên S3
  │
  ▼
  S3 chỉ thấy ciphertext — không thể đọc nội dung
```

### CSE với AWS SDK và KMS

```python
import boto3
from cryptography.fernet import Fernet

s3_client = boto3.client('s3')
kms_client = boto3.client('kms')

def upload_encrypted(bucket, key, data: bytes, kms_key_id: str):
    # 1. Tạo data key từ KMS
    response = kms_client.generate_data_key(
        KeyId=kms_key_id,
        KeySpec='AES_256'
    )
    plaintext_key = response['Plaintext']
    encrypted_key = response['CiphertextBlob']
    
    # 2. Mã hóa dữ liệu với Fernet (AES-128-CBC + HMAC)
    fernet = Fernet(base64.urlsafe_b64encode(plaintext_key[:32]))
    ciphertext = fernet.encrypt(data)
    
    # 3. Upload ciphertext và metadata
    s3_client.put_object(
        Bucket=bucket,
        Key=key,
        Body=ciphertext,
        Metadata={
            'x-amz-cek-alg': 'AES/CBC/PKCS5Padding',
            'x-amz-key-v2': base64.b64encode(encrypted_key).decode(),
            'x-amz-matdesc': f'{{"kms_cmk_id":"{kms_key_id}"}}'
        }
    )
    
    # 4. Xóa plaintext key khỏi memory
    del plaintext_key
```

### Dùng Thư Viện Amazon S3 Encryption Client

```python
# Dùng amazon-s3-encryption-client-python
from s3transfer.encrypt import S3EncryptionClient

encryption_client = S3EncryptionClient(
    encryption_config=KMSEncryptionConfig(
        kms_key_id='arn:aws:kms:...',
    )
)

# Upload tự động mã hóa
encryption_client.put_object(
    Bucket='my-bucket',
    Key='data.bin',
    Body=b'sensitive data'
)
```

---

## 6. Mã Hóa EBS

### EBS Encryption — Mã Hóa EBS

```
┌──────────────────────────────────────────────┐
│            EC2 Instance                       │
│  ┌──────────────┐    ┌──────────────────────┐ │
│  │ Application  │───▶│ EBS Encryption Layer │ │
│  └──────────────┘    │ (transparent to app) │ │
│                      └──────────────────────┘ │
│                               │               │
└───────────────────────────────┼───────────────┘
                                │ AES-256 encrypted
                                ▼
                     ┌──────────────────┐
                     │   EBS Volume     │
                     │  (ciphertext)    │
                     └──────────────────┘
```

### Đặc Điểm EBS Encryption

- **Transparent** — ứng dụng không cần thay đổi gì
- Mã hóa **tất cả dữ liệu**: data at rest, data in transit giữa EBS và EC2, snapshots, volumes từ snapshot
- Dùng **KMS key** (AWS managed hoặc CMK)
- **Không ảnh hưởng hiệu suất đáng kể** — hardware-accelerated trên Nitro instances

### Bật Mã Hóa Mặc Định Cho EBS

```bash
# Bật encryption mặc định cho tất cả EBS mới trong region
aws ec2 enable-ebs-encryption-by-default \
  --region ap-southeast-1

# Đặt CMK mặc định (tùy chọn)
aws ec2 modify-ebs-default-kms-key-id \
  --kms-key-id arn:aws:kms:ap-southeast-1:123456789012:key/mrk-xxx

# Kiểm tra
aws ec2 get-ebs-encryption-by-default
```

### Mã Hóa Volume Đang Chạy

```bash
# EBS không thể mã hóa trực tiếp volume đang dùng
# Quy trình: Snapshot → Copy với mã hóa → Tạo volume mới → Replace

# 1. Tạo snapshot
SNAPSHOT_ID=$(aws ec2 create-snapshot \
  --volume-id vol-xxxxxxxxx \
  --description "Pre-encryption snapshot" \
  --query 'SnapshotId' --output text)

# 2. Copy snapshot với encryption
ENCRYPTED_SNAP=$(aws ec2 copy-snapshot \
  --source-region ap-southeast-1 \
  --source-snapshot-id $SNAPSHOT_ID \
  --encrypted \
  --kms-key-id alias/my-ebs-key \
  --query 'SnapshotId' --output text)

# 3. Tạo volume từ snapshot đã mã hóa
aws ec2 create-volume \
  --snapshot-id $ENCRYPTED_SNAP \
  --volume-type gp3 \
  --availability-zone ap-southeast-1a
```

---

## 7. Mã Hóa EFS

### EFS Encryption At-Rest

- Dùng **KMS key** để mã hóa dữ liệu trên các thiết bị lưu trữ EFS
- Phải bật khi **tạo file system** — không thể bật sau
- **Transparent** với ứng dụng

```bash
# Tạo EFS với encryption
aws efs create-file-system \
  --performance-mode generalPurpose \
  --encrypted \
  --kms-key-id arn:aws:kms:ap-southeast-1:123456789012:key/mrk-xxx \
  --tags Key=Name,Value=encrypted-efs
```

### EFS Encryption In-Transit

```bash
# Mount với TLS (cần amazon-efs-utils)
sudo mount -t efs -o tls,accesspoint=fsap-xxxxxxxx \
  fs-xxxxxxxx: /mnt/efs

# Mount helper tự động dùng stunnel để mã hóa TLS
```

---

## 8. So Sánh Toàn Diện

### Bảng So Sánh Mã Hóa S3

| Thuộc Tính | SSE-S3 | SSE-KMS | SSE-C | CSE |
|-----------|--------|---------|-------|-----|
| Mã hóa xảy ra ở | AWS S3 | AWS S3 | AWS S3 | Client |
| Key lưu ở | AWS | AWS KMS | Không lưu | Client |
| Key quản lý bởi | AWS | Bạn/AWS | Bạn | Bạn |
| Audit via CloudTrail | ❌ | ✅ | ❌ | Bạn tự làm |
| Chi phí thêm | $0 | $1/key/month + API calls | $0 | $0 |
| Rotate key | Tự động AWS | Tự động/Thủ công | Thủ công | Thủ công |
| Cross-account | ✅ | ✅ (với key policy) | ✅ (chia sẻ key ngoài AWS) | ✅ |
| S3 Console upload | ✅ | ✅ | ❌ | ❌ |
| Phù hợp compliance | Cơ bản | HIPAA, PCI, FIPS | Nghiêm ngặt | Cao nhất |

### Decision Tree — Cây Quyết Định

```
Cần mã hóa S3?
├── Cần audit trail ai dùng key? 
│   ├── Có → SSE-KMS (CMK)
│   └── Không → SSE-S3
│
├── Cần giữ key hoàn toàn trong tổ chức (không bao giờ lên cloud)?
│   └── Có → CSE
│
├── Cần AWS không biết gì về key nhưng vẫn server-side?
│   └── Có → SSE-C
│
└── Compliance đơn giản, không cần audit?
    └── SSE-S3 đủ dùng
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: SSE-KMS và SSE-S3 khác nhau thế nào về mặt security?**

A: SSE-S3 dùng key AWS tạo và quản lý hoàn toàn — không có visibility hay audit trail. SSE-KMS dùng KMS key cho phép kiểm soát ai có thể dùng key (qua key policy), theo dõi từng lần dùng key qua CloudTrail, rotate key thủ công hoặc tự động, và thu hồi quyền truy cập ngay lập tức bằng cách disable key. Đây là lý do SSE-KMS phù hợp cho compliance như HIPAA — Health Insurance Portability and Accountability Act, PCI-DSS — Payment Card Industry Data Security Standard.

---

**Q: S3 Bucket Key là gì và tại sao nên bật?**

A: Bucket Key là DEK — Data Encryption Key cấp bucket ngắn hạn do S3 tạo từ CMK, dùng để encrypt DEK cho từng object thay vì gọi KMS mỗi lần. Điều này giảm ~99% số KMS API calls, từ đó giảm chi phí KMS đáng kể khi bucket có nhiều object. Nên bật cho mọi bucket dùng SSE-KMS, trừ khi cần audit granular từng object-level access.

---

**Q: Nếu KMS key bị disable, điều gì xảy ra với dữ liệu đã mã hóa?**

A: Dữ liệu đã upload vẫn tồn tại nhưng không thể đọc được — mọi GetObject call sẽ thất bại vì S3 không thể gọi KMS để decrypt DEK. Sau khi re-enable key, dữ liệu có thể đọc lại. Nếu key bị xóa (sau 7-30 ngày grace period), dữ liệu bị mất vĩnh viễn. Đây là lý do cần dùng KMS multi-region keys và không xóa key khi còn dữ liệu được mã hóa.

---

**Q: Tại sao SSE-C yêu cầu HTTPS bắt buộc?**

A: SSE-C truyền raw encryption key trong HTTP header của mỗi request. Nếu gửi qua HTTP (không mã hóa), key có thể bị đánh cắp qua MITM — Man-in-the-Middle attack. AWS từ chối mọi SSE-C request qua HTTP để bảo vệ key trong transit. Sau khi nhận, AWS không lưu key — chỉ lưu HMAC để verify key đúng trong các request sau.

---

**Cập Nhật:** 2026-05-16
