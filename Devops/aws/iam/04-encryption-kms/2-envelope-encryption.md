# Envelope Encryption — Mã Hóa Phong Bì

> Envelope Encryption (Mã Hóa Phong Bì) là kỹ thuật mã hóa hai tầng: dùng một khóa dữ liệu (DEK — Data Encryption Key — Khóa Mã Hóa Dữ Liệu) để mã hóa dữ liệu thực tế, sau đó dùng CMK (Customer Master Key — Khóa Chính Do Khách Hàng Quản Lý) để mã hóa DEK đó. Đây là cơ chế cốt lõi của mọi dịch vụ mã hóa trên AWS.

---

## 📚 Mục Lục

1. [Tại Sao Cần Envelope Encryption?](#tại-sao-cần)
2. [Cơ Chế Hoạt Động](#cơ-chế-hoạt-động)
3. [Quy Trình Mã Hóa (Encrypt Flow)](#quy-trình-mã-hóa)
4. [Quy Trình Giải Mã (Decrypt Flow)](#quy-trình-giải-mã)
5. [DEK Lifecycle](#dek-lifecycle)
6. [Triển Khai Trong Dịch Vụ AWS](#trong-dịch-vụ-aws)
7. [Tự Triển Khai Client-Side](#tự-triển-khai)
8. [Caching DEK](#caching-dek)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần

### Vấn Đề Khi Mã Hóa Trực Tiếp Bằng CMK

```
Mã hóa trực tiếp với CMK:
  ├── KMS giới hạn 4,096 bytes (4KB) mỗi request encrypt/decrypt
  ├── Mỗi encrypt/decrypt phải gọi KMS API qua network
  ├── Latency: ~1-5ms mỗi call (cộng dồn khi xử lý nhiều records)
  ├── Chi phí: $0.03 / 10,000 API calls (tăng theo lượng dữ liệu)
  └── Bottleneck: KMS rate limit 50,000 TPS (requests per second) shared

File 1GB → 262,144 lần gọi KMS (4KB/call) → BẤT KHẢ THI
```

### Giải Pháp Envelope Encryption

```
Envelope Encryption:
  ├── Gọi KMS 1 lần để lấy DEK (Data Encryption Key)
  ├── DEK mã hóa dữ liệu locally (AES-256-GCM, không giới hạn kích thước)
  ├── Lưu Encrypted DEK cùng với dữ liệu đã mã hóa
  └── Giải mã: gọi KMS 1 lần để giải mã DEK → dùng DEK giải mã data

File 1GB → 1 lần gọi KMS → HIỆU QUẢ VÀ NHANH CHÓNG
```

---

## Cơ Chế Hoạt Động

### Sơ Đồ Tổng Quan

```
┌─────────────────────────────────────────────────────────────────┐
│                         MÃ HÓA                                  │
│                                                                 │
│  Data (dữ liệu gốc)                                            │
│       │                                                         │
│       │          ┌──────── KMS ────────┐                       │
│       │          │                     │                        │
│       │    CMK ──┤→ Generate DEK       │                       │
│       │          │   ├── Plaintext DEK  │── Encrypted DEK       │
│       │          │   └── Encrypted DEK │        │              │
│       │          └─────────────────────┘        │              │
│       │                 │                        │              │
│       ├──[AES-256]──────┘                        │              │
│       ↓                                          │              │
│  Encrypted Data ←──────────────────────────────┘              │
│  (lưu cùng Encrypted DEK)                                      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                         GIẢI MÃ                                 │
│                                                                 │
│  Encrypted Data + Encrypted DEK                                │
│                          │                                      │
│                    ┌─────┴──── KMS ────────┐                   │
│                    │                       │                    │
│              CMK ──┤→ Decrypt(Encrypted DEK)│                  │
│                    │       ↓               │                    │
│                    │  Plaintext DEK        │                    │
│                    └───────────────────────┘                   │
│                           │                                     │
│  Encrypted Data ──[AES-256]──→ Plaintext Data                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Quy Trình Mã Hóa

### Bước Chi Tiết

```
Bước 1: Ứng dụng yêu cầu KMS tạo Data Key (DEK)
        aws kms generate-data-key --key-id <CMK-ID> --key-spec AES_256

Bước 2: KMS trả về:
        ├── Plaintext DEK (32 bytes, AES-256)    ← dùng để mã hóa
        └── Encrypted DEK (ciphertext của DEK)   ← lưu trữ lâu dài

Bước 3: Ứng dụng dùng Plaintext DEK mã hóa dữ liệu (AES-256-GCM locally)

Bước 4: Xóa Plaintext DEK khỏi memory ngay lập tức (QUAN TRỌNG!)

Bước 5: Lưu trữ: Encrypted DEK + Encrypted Data
        (có thể lưu chung một file hoặc tách riêng)
```

### Ví Dụ Thực Tế

```python
import boto3
import os
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import base64

kms = boto3.client('kms', region_name='us-east-1')
CMK_KEY_ID = 'alias/prod-app-key'

def encrypt_data(plaintext: bytes) -> dict:
    """Mã hóa dữ liệu dùng Envelope Encryption."""

    # Bước 1: Lấy Data Key từ KMS
    response = kms.generate_data_key(
        KeyId=CMK_KEY_ID,
        KeySpec='AES_256'
    )

    plaintext_dek = response['Plaintext']      # 32 bytes, AES-256
    encrypted_dek = response['CiphertextBlob'] # dùng để lưu trữ

    try:
        # Bước 2: Mã hóa dữ liệu locally với AES-256-GCM
        nonce = os.urandom(12)  # 96-bit nonce cho GCM
        aesgcm = AESGCM(plaintext_dek)
        ciphertext = aesgcm.encrypt(nonce, plaintext, None)

        return {
            'encrypted_dek': base64.b64encode(encrypted_dek).decode(),
            'nonce': base64.b64encode(nonce).decode(),
            'ciphertext': base64.b64encode(ciphertext).decode()
        }
    finally:
        # Bước 3: Xóa plaintext DEK khỏi memory (quan trọng!)
        del plaintext_dek
```

---

## Quy Trình Giải Mã

### Bước Chi Tiết

```
Bước 1: Lấy Encrypted DEK và Encrypted Data từ storage

Bước 2: Gửi Encrypted DEK đến KMS để giải mã
        aws kms decrypt --ciphertext-blob <Encrypted DEK>
        (KMS tự biết CMK nào đã mã hóa DEK — thông tin được nhúng vào ciphertext)

Bước 3: KMS trả về Plaintext DEK (sau khi kiểm tra quyền)

Bước 4: Ứng dụng dùng Plaintext DEK giải mã Encrypted Data

Bước 5: Xóa Plaintext DEK khỏi memory
```

```python
def decrypt_data(encrypted_payload: dict) -> bytes:
    """Giải mã dữ liệu dùng Envelope Encryption."""

    encrypted_dek = base64.b64decode(encrypted_payload['encrypted_dek'])
    nonce = base64.b64decode(encrypted_payload['nonce'])
    ciphertext = base64.b64decode(encrypted_payload['ciphertext'])

    # Bước 1: KMS giải mã Encrypted DEK → Plaintext DEK
    response = kms.decrypt(
        CiphertextBlob=encrypted_dek
        # Không cần chỉ định KeyId — KMS tự biết từ ciphertext
    )
    plaintext_dek = response['Plaintext']

    try:
        # Bước 2: Giải mã data locally
        aesgcm = AESGCM(plaintext_dek)
        return aesgcm.decrypt(nonce, ciphertext, None)
    finally:
        del plaintext_dek
```

---

## DEK Lifecycle

### Vòng Đời Của Data Encryption Key

```
1. GENERATION (Tạo)
   KMS.GenerateDataKey() → Plaintext DEK + Encrypted DEK

2. USE (Sử Dụng)
   Plaintext DEK dùng để mã hóa/giải mã data trong memory

3. DELETION (Xóa)
   Plaintext DEK phải bị xóa khỏi memory sau khi dùng xong
   (Encrypted DEK vẫn được lưu trữ lâu dài)

4. STORAGE (Lưu Trữ)
   Chỉ Encrypted DEK được lưu trữ (thường cùng với encrypted data)

5. RECOVERY (Phục Hồi)
   Khi cần giải mã: KMS.Decrypt(Encrypted DEK) → Plaintext DEK
   (Cần quyền kms:Decrypt và CMK phải trong trạng thái Enabled)
```

### Nơi Lưu Encrypted DEK

| Pattern | Nơi Lưu Encrypted DEK | Khi Nào Dùng |
|---|---|---|
| **Inline** | Ghép vào đầu file cùng encrypted data | File storage, S3 objects |
| **Separate record** | Database column riêng | Database record encryption |
| **Sidecar file** | File `.key` cùng thư mục | Backup files |
| **Metadata store** | DynamoDB hoặc Secrets Manager | Microservices architecture |

### Định Dạng File Mã Hóa Chuẩn (Best Practice)

```
┌─────────────────────────────────────┐
│           Encrypted File             │
├─────────────────────────────────────┤
│  Header (4 bytes): magic bytes       │
│  Version (1 byte): format version    │
│  DEK Length (2 bytes)               │
│  Encrypted DEK (<DEK Length> bytes) │
│  IV/Nonce (12 bytes cho AES-GCM)    │
│  Encrypted Data (variable)          │
│  Auth Tag (16 bytes cho AES-GCM)    │
└─────────────────────────────────────┘
```

---

## Trong Dịch Vụ AWS

### AWS Dịch Vụ Dùng Envelope Encryption Tự Động

```
S3 (SSE-KMS):
  Upload → S3 tạo DEK từ KMS → mã hóa object → lưu Encrypted DEK trong metadata
  Download → S3 gửi Encrypted DEK đến KMS → giải mã DEK → giải mã object

RDS (Encryption at Rest):
  Mỗi DB instance có một DEK
  DEK được mã hóa bằng CMK và lưu trong RDS metadata

EBS (Encrypted Volume):
  Mỗi volume có một DEK
  Tất cả data I/O qua DEK; DEK được protect bởi CMK

DynamoDB:
  Mỗi table có DEK riêng
  DEK được rotate tự động

Secrets Manager:
  Mỗi secret version được mã hóa bằng DEK riêng
```

### Luồng Chi Tiết S3 SSE-KMS

```
PUT Object:
  1. App → S3: PUT /bucket/key (cùng dữ liệu)
  2. S3 → KMS: GenerateDataKey(KeyId=<CMK>)
  3. KMS → S3: {Plaintext DEK, Encrypted DEK}
  4. S3: Mã hóa object = AES-256(object, Plaintext DEK)
  5. S3: Xóa Plaintext DEK
  6. S3 lưu: Encrypted Object + Encrypted DEK (trong object metadata)

GET Object:
  1. App → S3: GET /bucket/key
  2. S3 đọc: Encrypted Object + Encrypted DEK
  3. S3 → KMS: Decrypt(Encrypted DEK) [yêu cầu kms:Decrypt permission]
  4. KMS → S3: Plaintext DEK
  5. S3: Giải mã object = AES-256-decrypt(Encrypted Object, Plaintext DEK)
  6. S3: Xóa Plaintext DEK
  7. S3 → App: Plaintext Object
```

---

## Tự Triển Khai Client-Side

### AWS Encryption SDK

AWS cung cấp Encryption SDK (Software Development Kit — Bộ Công Cụ Phát Triển Phần Mềm) để triển khai envelope encryption chuẩn hóa:

```python
import aws_encryption_sdk

# Khởi tạo client
client = aws_encryption_sdk.EncryptionSDKClient()

# Tạo KMS master key provider
kms_kwargs = {"key_ids": ["arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"]}
master_key_provider = aws_encryption_sdk.StrictAwsKmsMasterKeyProvider(**kms_kwargs)

# Mã hóa
plaintext = b"Sensitive data here"
ciphertext, encryptor_header = client.encrypt(
    source=plaintext,
    key_provider=master_key_provider,
    encryption_context={        # context bổ sung (không mã hóa nhưng được auth)
        "purpose": "user-pii",
        "environment": "prod"
    }
)

# Giải mã
decrypted_plaintext, decryptor_header = client.decrypt(
    source=ciphertext,
    key_provider=master_key_provider
)
```

### Encryption Context (Bối Cảnh Mã Hóa)

Encryption Context là tập hợp key-value pairs không bí mật, được **xác thực** (nhưng không mã hóa) cùng với dữ liệu:

```
Tác dụng:
  ├── Cung cấp thêm thông tin cho audit trail trong CloudTrail
  ├── Ngăn chặn tấn công ciphertext confusion (nhầm lẫn bản mã)
  ├── Điều kiện trong key policy (dùng kms:EncryptionContext)
  └── Nếu context khác nhau → decrypt sẽ thất bại

Ví dụ:
  {
    "table": "users",
    "column": "ssn",
    "environment": "production",
    "requestor": "billing-service"
  }
```

```json
// Trong CloudTrail log, bạn thấy encryption context:
{
  "eventName": "Decrypt",
  "requestParameters": {
    "encryptionContext": {
      "table": "users",
      "column": "ssn",
      "environment": "production"
    }
  }
}
```

---

## Caching DEK

### DEK Caching (Lưu Cache Khóa Dữ Liệu)

Để giảm số lượng KMS API calls, Encryption SDK hỗ trợ caching DEK trong memory:

```python
from aws_encryption_sdk.caches.local import LocalCryptoMaterialsCache
from aws_encryption_sdk.materials_managers.caching import CachingCryptographicMaterialsManager

# Tạo local cache (in-memory)
cache = LocalCryptoMaterialsCache(capacity=100)  # tối đa 100 DEK cached

# Caching Material Manager với giới hạn an toàn
caching_cmm = CachingCryptographicMaterialsManager(
    master_key_provider=master_key_provider,
    cache=cache,
    max_age=300.0,          # DEK hết hạn sau 300 giây (5 phút)
    max_messages_encrypted=100,  # Tối đa 100 message dùng 1 DEK
    max_bytes_encrypted=2**32    # Tối đa 4GB dùng 1 DEK
)
```

### Trade-offs Của Caching

| Lợi Ích | Rủi Ro |
|---|---|
| Giảm KMS API calls (tiết kiệm chi phí) | DEK tồn tại lâu trong memory hơn |
| Giảm latency (không chờ KMS) | Cửa sổ exposure rộng hơn nếu bị compromise |
| Giảm áp lực lên KMS rate limits | Cache có thể bị đọc nếu process bị tấn công |

**Quy tắc an toàn:**
- `max_age` không nên quá 5 phút cho dữ liệu nhạy cảm cao
- `max_messages_encrypted` nên ≤ 1,000 cho dữ liệu tài chính
- Không cache khi xử lý dữ liệu HIPAA/PCI-DSS nếu không có yêu cầu đặc biệt

---

## Câu Hỏi Phỏng Vấn

**Q: Giải thích Envelope Encryption từ đầu đến cuối.**

> Khi mã hóa dữ liệu, ứng dụng gọi KMS để tạo một Data Key (DEK). KMS trả về hai thứ: plaintext DEK và encrypted DEK. Ứng dụng dùng plaintext DEK để mã hóa dữ liệu thực tế bằng AES-256, sau đó **xóa ngay plaintext DEK khỏi memory**. Chỉ lưu encrypted DEK cùng encrypted data. Khi cần giải mã, gửi encrypted DEK đến KMS, KMS trả về plaintext DEK, dùng để giải mã data rồi xóa DEK khỏi memory. Tên "phong bì" (envelope) vì DEK bọc bên ngoài data như phong bì, CMK bọc bên ngoài DEK như phong bì ngoài cùng.

**Q: Tại sao phải xóa plaintext DEK khỏi memory ngay sau khi dùng?**

> Vì nếu attacker dump memory của process (memory scraping), họ có thể lấy plaintext DEK và giải mã tất cả dữ liệu mà không cần qua KMS. Encrypted DEK vô dụng nếu không có quyền gọi KMS. Đây là nguyên tắc "minimize exposure window" — thu hẹp cửa sổ thời gian khóa tồn tại ở dạng plaintext.

**Q: Encryption Context là gì, tại sao quan trọng?**

> Encryption Context là metadata key-value không mã hóa nhưng được xác thực (authenticated) cùng với ciphertext thông qua AEAD (Authenticated Encryption with Associated Data). Nếu giải mã với context khác → thất bại. Nó ngăn tấn công "ciphertext reuse" (dùng lại bản mã ở ngữ cảnh sai), xuất hiện trong CloudTrail giúp audit, và có thể dùng làm điều kiện trong key policy.

**Q: Khi CMK bị xóa (deleted), dữ liệu đã mã hóa có bị mất không?**

> Về mặt kỹ thuật, dữ liệu đã mã hóa vẫn còn trong storage nhưng **không thể giải mã được nữa** vì Encrypted DEK không thể giải mã mà không có CMK. Đây là lý do KMS có "pending deletion" period (7–30 ngày) để bạn có cơ hội hủy. Phải có chiến lược backup data hoặc backup key material trước khi xóa CMK.

---

## Tóm Tắt

```
Envelope Encryption — Điểm Chính:

Tại sao: KMS giới hạn 4KB; hiệu năng; chi phí
Cơ chế:  CMK → tạo DEK → DEK mã hóa data → Encrypted DEK lưu cùng data
Decrypt: Encrypted DEK → KMS giải mã → DEK → giải mã data
DEK lifecycle: Tạo → Dùng → Xóa khỏi memory → Encrypted DEK lưu lâu dài
AWS tự làm: S3 SSE-KMS, RDS, EBS, DynamoDB — bạn không phải viết code
Tự làm: AWS Encryption SDK + KMS; hỗ trợ caching DEK
```

---

**Trước Đó:** [1-kms-key-types.md](1-kms-key-types.md)
**Tiếp Theo:** [3-key-policies.md](3-key-policies.md) — Key Policy vs IAM Policy

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
