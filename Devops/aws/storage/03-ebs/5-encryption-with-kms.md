# EBS Encryption With KMS — Mã Hóa EBS Với KMS

> EBS Encryption (Mã Hóa EBS) sử dụng AES-256 (Advanced Encryption Standard 256-bit — Tiêu Chuẩn Mã Hóa Nâng Cao 256-bit) kết hợp với AWS KMS (Key Management Service — Dịch Vụ Quản Lý Khóa) để bảo vệ dữ liệu at-rest (khi lưu trữ) và in-transit (khi truyền) giữa EC2 và EBS. Quá trình mã hóa hoàn toàn transparent (trong suốt) với ứng dụng và không ảnh hưởng đáng kể đến hiệu suất.

---

## 🔐 Cơ Chế Mã Hóa EBS

### Kiến Trúc Mã Hóa

```
┌─────────────────────────────────────────────────────────┐
│                    KMS                                  │
│  ┌─────────────────────────────────────────────────┐   │
│  │  CMK (Customer Master Key — Khóa Chính KH)      │   │
│  │  arn:aws:kms:us-east-1:123456789:key/abc-xxx    │   │
│  └───────────────────────┬─────────────────────────┘   │
│                          │ GenerateDataKey API           │
│                          ▼                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │  DEK (Data Encryption Key — Khóa Mã Hóa Dữ Liệu│   │
│  │  Plaintext DEK: dùng để encrypt data            │   │
│  │  Encrypted DEK: lưu trong EBS metadata          │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
              ┌────────────────────────┐
              │       EC2 Instance     │
              │  AES-256 encryption    │
              │  (in EC2 hypervisor)   │
              └────────────┬───────────┘
                           │
                           ▼ Encrypted data
              ┌────────────────────────┐
              │      EBS Volume        │
              │  Dữ liệu đã mã hóa    │
              │  DEK encrypted lưu kèm │
              └────────────────────────┘
```

### Quy Trình Chi Tiết

```
1. Tạo EBS volume mã hóa:
   EBS → yêu cầu KMS GenerateDataKey
   KMS → trả về: plaintext DEK + encrypted DEK
   EBS → lưu encrypted DEK trong volume metadata

2. Mount volume vào EC2:
   EC2 Hypervisor → yêu cầu KMS Decrypt(encrypted DEK)
   KMS → xác thực IAM permissions → trả về plaintext DEK
   EC2 Hypervisor → giữ plaintext DEK trong memory để encrypt/decrypt I/O

3. Read/Write:
   App → write plaintext data
   EC2 Hypervisor → encrypt với DEK → gửi encrypted data → EBS
   
   App → read request
   EBS → trả encrypted data → EC2 Hypervisor → decrypt với DEK → App nhận plaintext

4. Unmount / Stop EC2:
   Plaintext DEK bị xóa khỏi memory
   Lần mount tiếp theo phải call KMS lại
```

---

## 🔑 KMS Key Types (Các Loại Khóa KMS)

### AWS Managed Key (Khóa Được Quản Lý Bởi AWS)

```
Tên: aws/ebs
ARN: arn:aws:kms:us-east-1:123456789012:alias/aws/ebs
Rotation: Tự động mỗi năm (không tắt được)
Giá: Miễn phí (không tính phí key storage)
Control: Không tùy chỉnh key policy được

Khi dùng:
  → Bắt đầu nhanh, không cần quản lý key
  → Không có yêu cầu compliance đặc biệt
  → Cross-account sharing không cần (không chia sẻ AWS managed key được)
```

### Customer Managed Key — CMK (Khóa Do Khách Hàng Quản Lý)

```
Tên: Tự đặt (ví dụ: prod-ebs-encryption-key)
ARN: arn:aws:kms:us-east-1:123456789012:key/your-key-id
Rotation: Tùy chọn — hàng năm hoặc thủ công
Giá: $1/key/tháng + $0.03/10.000 API calls
Control: Toàn quyền kiểm soát key policy

Khi dùng:
  → Cần chia sẻ encrypted snapshot sang account khác
  → Cần kiểm soát rõ ràng ai có thể dùng key (audit log)
  → Compliance yêu cầu customer-managed keys (HIPAA, PCI-DSS)
  → Cần import own key material (Bring Your Own Key — BYOK)
```

### Customer Managed Key — External Key Store

```
KMS External Key Store (XKS) — Kho Khóa Bên Ngoài:
  → Khóa lưu trong HSM (Hardware Security Module — Module Bảo Mật Phần Cứng) của khách hàng
  → AWS không bao giờ có plaintext key
  → Dùng khi: sovereignty requirements, financial regulatory compliance
  → Phức tạp hơn nhiều, chỉ dùng khi thực sự cần
```

---

## ⚙️ Cấu Hình Encryption

### Tạo Volume Đã Mã Hóa

```bash
# Dùng AWS managed key (aws/ebs)
aws ec2 create-volume \
  --volume-type gp3 \
  --size 100 \
  --availability-zone us-east-1a \
  --encrypted  # Dùng aws/ebs key mặc định

# Dùng Customer Managed Key cụ thể
aws ec2 create-volume \
  --volume-type io2 \
  --size 500 \
  --iops 10000 \
  --availability-zone us-east-1a \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/your-cmk-id

# Dùng alias (dễ nhớ hơn ARN)
aws ec2 create-volume \
  --volume-type gp3 \
  --size 100 \
  --availability-zone us-east-1a \
  --encrypted \
  --kms-key-id alias/prod-ebs-key
```

### Bật Default Encryption Cho Toàn Account

```bash
# Bật mã hóa mặc định (best practice cho production account)
aws ec2 enable-ebs-encryption-by-default \
  --region us-east-1

# Kiểm tra status
aws ec2 get-ebs-encryption-by-default \
  --region us-east-1
# {"EbsEncryptionByDefault": true}

# Set default CMK cho region
aws ec2 modify-ebs-default-kms-key-id \
  --kms-key-id alias/prod-ebs-key

# Từ giờ, tất cả volume mới (kể cả trong Auto Scaling Group)
# sẽ tự động được mã hóa với prod-ebs-key
```

---

## 🔄 Encrypt Volume Đang Chạy

EBS **không thể** trực tiếp encrypt volume đang có dữ liệu. Phải dùng quy trình gián tiếp:

### Quy Trình Encrypt Volume Không Có Mã Hóa

```
Bước 1: Tạo Snapshot từ volume gốc (unencrypted)
          ┌──────────────┐
          │  EBS Vol     │ → Create Snapshot → snap-unencrypted
          │  (No encrypt)│
          └──────────────┘

Bước 2: Copy Snapshot và bật mã hóa
          snap-unencrypted → Copy (with encryption) → snap-encrypted

Bước 3: Tạo Volume mới từ encrypted snapshot
          snap-encrypted → Create Volume → vol-encrypted (gp3, 100GB)

Bước 4: Detach volume cũ, attach volume mới
          EC2 → Detach old → Attach new
          (Cần stop instance nếu là root volume)
```

```bash
# Bước 1: Snapshot
SNAP_ID=$(aws ec2 create-snapshot \
  --volume-id vol-unencrypted \
  --description "Pre-encrypt snapshot" \
  --query SnapshotId --output text)

echo "Snapshot: $SNAP_ID"

# Chờ snapshot completed
aws ec2 wait snapshot-completed --snapshot-ids $SNAP_ID

# Bước 2: Copy với mã hóa
ENCRYPTED_SNAP=$(aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id $SNAP_ID \
  --destination-region us-east-1 \
  --encrypted \
  --kms-key-id alias/prod-ebs-key \
  --description "Encrypted copy" \
  --query SnapshotId --output text)

aws ec2 wait snapshot-completed --snapshot-ids $ENCRYPTED_SNAP

# Bước 3: Tạo volume mới
NEW_VOL=$(aws ec2 create-volume \
  --snapshot-id $ENCRYPTED_SNAP \
  --availability-zone us-east-1a \
  --volume-type gp3 \
  --query VolumeId --output text)

echo "New encrypted volume: $NEW_VOL"
```

---

## 📋 KMS Key Policy — Chính Sách Khóa KMS

Key policy kiểm soát **ai được dùng CMK để mã hóa/giải mã EBS volumes**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow EC2 to use key for EBS encryption",
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey*",
        "kms:CreateGrant",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Allow developers to use key",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/developers"
      },
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Allow key administrators",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/key-admins"
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
        "kms:Delete*",
        "kms:ScheduleKeyDeletion",
        "kms:CancelKeyDeletion"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 🔄 Key Rotation — Xoay Vòng Khóa

### Automatic Key Rotation (Xoay Vòng Tự Động)

```bash
# Bật automatic rotation cho CMK (mỗi năm 1 lần)
aws kms enable-key-rotation \
  --key-id alias/prod-ebs-key

# Kiểm tra rotation status
aws kms get-key-rotation-status \
  --key-id alias/prod-ebs-key

# Key rotation KHÔNG yêu cầu re-encrypt dữ liệu:
#   - KMS giữ tất cả phiên bản key cũ
#   - DEK được re-encrypt với key mới theo thời gian
#   - Volume vẫn hoạt động bình thường trong suốt quá trình
```

### Manual Key Rotation (Xoay Vòng Thủ Công)

Khi cần rotate sang CMK hoàn toàn mới (BYOK — Bring Your Own Key):

```bash
# 1. Tạo CMK mới
NEW_KEY_ID=$(aws kms create-key \
  --description "EBS encryption key v2" \
  --key-usage ENCRYPT_DECRYPT \
  --query KeyMetadata.KeyId --output text)

aws kms create-alias \
  --alias-name alias/prod-ebs-key-v2 \
  --target-key-id $NEW_KEY_ID

# 2. Re-encrypt volumes bằng cách: snapshot → copy với key mới → volume mới
# (Giống quy trình encrypt volume ở phần trên)
```

---

## 🔗 Cross-Account Snapshot Sharing (Chia Sẻ Snapshot Giữa Accounts)

Để chia sẻ encrypted snapshot sang account khác, cần chia sẻ cả CMK:

```bash
# Account A: Chia sẻ snapshot
aws ec2 modify-snapshot-attribute \
  --snapshot-id snap-encrypted-xxx \
  --attribute createVolumePermission \
  --operation-type add \
  --user-ids 111122223333  # Account B ID

# Account A: Cho phép Account B dùng CMK
# Thêm vào key policy:
{
  "Sid": "Allow Account B",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::111122223333:root"
  },
  "Action": [
    "kms:Decrypt",
    "kms:GenerateDataKey*",
    "kms:DescribeKey",
    "kms:CreateGrant"
  ],
  "Resource": "*"
}

# Account B: Tạo volume từ shared snapshot (re-encrypt với own key)
aws ec2 create-volume \
  --snapshot-id snap-encrypted-xxx \
  --availability-zone us-east-1a \
  --encrypted \
  --kms-key-id alias/account-b-key  # Dùng key của account B
```

---

## 📊 Ảnh Hưởng Đến Hiệu Suất

### Overhead Của Mã Hóa

```
AES-256 mã hóa trong EC2 Hypervisor:
  - Các EC2 instance dựa trên AWS Nitro System: overhead gần 0%
  - Instance cũ (non-Nitro): overhead < 1-2%

Overhead KMS API calls:
  - Chỉ gọi KMS khi attach volume (lần đầu mount)
  - Không gọi KMS cho mỗi read/write operation
  - DEK được cache trong EC2 hypervisor memory

Kết luận: Mã hóa EBS hầu như không ảnh hưởng performance
→ Không có lý do kỹ thuật nào để không bật mã hóa
```

---

## ✅ Best Practices — Thực Hành Tốt Nhất

### 1. Bật Default Encryption

```bash
# Làm ngay cho tất cả production regions
aws ec2 enable-ebs-encryption-by-default
aws ec2 modify-ebs-default-kms-key-id --kms-key-id alias/prod-ebs-key
```

### 2. Dùng CMK Thay Vì AWS Managed Key

- Kiểm soát key policy — audit rõ ai dùng key
- Có thể revoke access khi cần (ví dụ: nhân viên nghỉ việc)
- Hỗ trợ cross-account sharing
- Tuân thủ compliance HIPAA, PCI-DSS, SOC2

### 3. Enable CloudTrail Cho KMS API Calls

```bash
# CloudTrail tự động log tất cả KMS API calls
# Kiểm tra log: ai gọi Decrypt, GenerateDataKey, khi nào, từ IP nào
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=kms.amazonaws.com \
  --max-results 10
```

### 4. Tách Key Theo Môi Trường

```
Production:    alias/prod-ebs-key
Staging:       alias/staging-ebs-key
Development:   alias/dev-ebs-key (có thể dùng AWS managed)

→ Nhân viên dev không có quyền dùng prod key
→ Breach ở dev không ảnh hưởng prod data
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: EBS encryption hoạt động như thế nào?**
> EBS dùng envelope encryption (mã hóa phong bì): KMS tạo DEK (Data Encryption Key — Khóa Mã Hóa Dữ Liệu), plaintext DEK dùng để encrypt data với AES-256 trong EC2 hypervisor, encrypted DEK lưu trong EBS metadata. Khi mount volume, EC2 gọi KMS Decrypt để lấy plaintext DEK. Ứng dụng không thấy quá trình này.

**Q: Sự khác biệt giữa AWS Managed Key và Customer Managed Key?**
> AWS Managed Key: tự động rotate, miễn phí, không tùy chỉnh được, không chia sẻ cross-account. CMK: $1/tháng, toàn quyền kiểm soát policy, hỗ trợ cross-account sharing, BYOK, audit log rõ ràng — phù hợp production và compliance.

**Q: Làm sao encrypt EBS volume đang có dữ liệu mà không có downtime?**
> Không thể encrypt trực tiếp. Quy trình: (1) Tạo snapshot từ volume gốc; (2) Copy snapshot với encryption bật; (3) Tạo volume mới từ encrypted snapshot; (4) Migrate ứng dụng sang volume mới. Nếu cần zero-downtime, có thể dùng replication ở tầng ứng dụng (database replication).

**Q: Key rotation có yêu cầu re-encrypt dữ liệu không?**
> Không. AWS KMS dùng envelope encryption. Khi rotate, KMS tạo key material mới nhưng vẫn giữ key cũ. DEK được re-encrypt với key mới theo thời gian (lazy re-wrapping). Volume tiếp tục hoạt động không gián đoạn. Key cũ chỉ bị xóa sau khi tất cả DEK đã được re-wrapped.

---

## 🔗 Điều Hướng

- **Trước:** [4-performance-tuning.md](./4-performance-tuning.md) — Performance Tuning
- **Tiếp theo:** [6-ebs-vs-instance-store.md](./6-ebs-vs-instance-store.md) — EBS vs Instance Store
- **Liên quan:** [../05-security/4-encryption-at-rest.md](../05-security/4-encryption-at-rest.md) — Encryption At Rest tổng quan

---

**Cập Nhật Lần Cuối:** 2026-05-15
