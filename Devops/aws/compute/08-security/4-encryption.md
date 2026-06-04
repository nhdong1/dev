# Encryption — Mã Hóa Dữ Liệu Trên AWS Compute

> Encryption (Mã Hóa) là quá trình chuyển đổi dữ liệu thành dạng không đọc được trừ khi có encryption key (khóa mã hóa). AWS cung cấp mã hóa toàn diện cho dữ liệu at rest (khi lưu trữ) và in transit (khi truyền), với KMS (Key Management Service — Dịch Vụ Quản Lý Khóa) là trung tâm quản lý keys.

---

## 📚 Mục Lục

1. [Tổng Quan Encryption Trên AWS](#tổng-quan-encryption-trên-aws)
2. [AWS KMS — Key Management Service](#aws-kms--key-management-service)
3. [EBS Encryption — Mã Hóa Đĩa EC2](#ebs-encryption--mã-hóa-đĩa-ec2)
4. [Encryption In Transit — Mã Hóa Khi Truyền](#encryption-in-transit--mã-hóa-khi-truyền)
5. [Encryption Các Service Khác](#encryption-các-service-khác)
6. [Envelope Encryption — Mã Hóa Phong Bì](#envelope-encryption--mã-hóa-phong-bì)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Encryption Trên AWS

### Hai Loại Encryption Cần Biết

```
Encryption at Rest (Mã Hóa Khi Lưu Trữ):
└── Dữ liệu được mã hóa khi không được sử dụng
    → Bảo vệ khi: hard drive bị đánh cắp, backup bị lộ,
                  nhân viên AWS có access vật lý

    Áp dụng cho: EBS volumes, S3 buckets, RDS databases,
                 EFS file systems, DynamoDB tables

Encryption in Transit (Mã Hóa Khi Truyền):
└── Dữ liệu được mã hóa khi di chuyển qua mạng
    → Bảo vệ khi: network eavesdropping (nghe lén mạng),
                  man-in-the-middle attacks

    Áp dụng cho: HTTPS (HTTP Secure), TLS (Transport Layer Security
                 — Bảo Mật Tầng Truyền Tải), VPN, AWS API calls
```

### Mô Hình Trách Nhiệm

```
AWS Chịu Trách Nhiệm:
├── Physical security của data centers
├── Encryption của hypervisor layer
├── Encryption tự động của một số managed services
│   (ví dụ: SQS, CloudTrail mã hóa mặc định)
└── KMS infrastructure

Bạn Chịu Trách Nhiệm:
├── Bật encryption cho EBS, RDS, EFS (có thể off theo default)
├── Cấu hình TLS cho applications
├── Quản lý Customer Managed Keys trong KMS
├── S3 bucket encryption settings
└── Application-level encryption nếu cần
```

---

## AWS KMS — Key Management Service

### Khái Niệm Cốt Lõi

```
KMS (Key Management Service — Dịch Vụ Quản Lý Khóa):
└── Dịch vụ managed để tạo và quản lý cryptographic keys

Các Loại KMS Keys:
├── AWS Managed Keys (Khóa AWS Quản Lý):
│   ├── AWS tự tạo và rotate
│   ├── Miễn phí
│   ├── Tên: aws/s3, aws/ebs, aws/rds...
│   └── Không thể xóa, không tùy chỉnh policy
│
├── Customer Managed Keys — CMK (Khóa Khách Hàng Quản Lý):
│   ├── Bạn tạo, có toàn quyền control
│   ├── $1/key/tháng + $0.03/10K API calls
│   ├── Có thể set rotation policy (auto mỗi năm)
│   └── Có thể định nghĩa Key Policy chi tiết
│
└── Customer Provided Keys (Khóa Khách Hàng Tự Cung Cấp):
    ├── Bạn mang key của mình lên AWS (Bring Your Own Key — BYOK)
    ├── Dùng với S3 SSE-C
    └── AWS không lưu key, chỉ encrypt/decrypt khi bạn gửi key
```

### Tạo Và Quản Lý KMS Keys

```bash
# Tạo Customer Managed Key (CMK)
aws kms create-key \
  --description "EBS encryption key for production" \
  --key-usage ENCRYPT_DECRYPT \
  --origin AWS_KMS \
  --tags TagKey=Environment,TagValue=Production

# Tạo alias dễ nhớ
aws kms create-alias \
  --alias-name "alias/production-ebs-key" \
  --target-key-id "arn:aws:kms:ap-southeast-1:123456789012:key/key-id"

# Bật auto-rotation (rotate mỗi năm)
aws kms enable-key-rotation \
  --key-id "alias/production-ebs-key"

# Kiểm tra rotation status
aws kms get-key-rotation-status \
  --key-id "alias/production-ebs-key"

# Mã hóa data bằng KMS
aws kms encrypt \
  --key-id "alias/production-ebs-key" \
  --plaintext "my-secret-data" \
  --output text \
  --query CiphertextBlob | base64 -d > encrypted.bin

# Giải mã
aws kms decrypt \
  --ciphertext-blob fileb://encrypted.bin \
  --output text \
  --query Plaintext | base64 -d
```

### KMS Key Policy — Chính Sách Khóa

```json
// Key Policy quyết định ai có thể dùng key
{
  "Version": "2012-10-17",
  "Id": "key-policy-1",
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
      "Sid": "Allow key administrators",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/KeyAdminRole"
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
        "kms:ScheduleKeyDeletion",
        "kms:CancelKeyDeletion"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Allow use of the key by EC2",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/ProductionEC2Role"
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    }
  ]
}
```

### KMS Multi-Region Keys — Khóa Đa Vùng

```bash
# Tạo primary key trong ap-southeast-1
PRIMARY_KEY=$(aws kms create-key \
  --description "Multi-region primary key" \
  --multi-region \
  --region ap-southeast-1 \
  --query 'KeyMetadata.KeyId' --output text)

# Replicate sang us-east-1
aws kms replicate-key \
  --key-id $PRIMARY_KEY \
  --replica-region us-east-1 \
  --region ap-southeast-1

# Multi-region keys có cùng key material
# → Encrypt ở Singapore, Decrypt ở Virginia (cùng key ID prefix)
# → Hữu ích cho multi-region active-active architecture
```

---

## EBS Encryption — Mã Hóa Đĩa EC2

### EBS Encryption Hoạt Động Thế Nào

```
Khi EBS volume được encrypt:

EC2 Instance               EBS Service              KMS
──────────────             ───────────              ─────
Application
    │
    │ Write data
    ▼
OS Layer                  AWS Hypervisor
    │                          │
    │ Data (plaintext)         │ Encrypt with DEK
    └─────────────────────────►│◄─── Data Encryption Key (DEK)
                               │     (DEK được encrypt bởi KMS CMK)
                               │
                               │ Encrypted data → Stored on disk
                               │
                      DEK (encrypted) stored alongside data
                      KMS never sees plaintext data!

DEK = Data Encryption Key — Khóa Mã Hóa Dữ Liệu
CMK = Customer Master Key — Khóa Chính Của Khách Hàng
```

### Bật EBS Encryption

```bash
# Bật EBS encryption mặc định cho toàn bộ account/region
aws ec2 enable-ebs-encryption-by-default \
  --region ap-southeast-1

# Đặt KMS key mặc định cho EBS encryption
aws ec2 modify-ebs-default-kms-key-id \
  --kms-key-id "alias/production-ebs-key"

# Kiểm tra trạng thái
aws ec2 get-ebs-encryption-by-default
aws ec2 get-ebs-default-kms-key-id

# Tạo encrypted EBS volume
aws ec2 create-volume \
  --availability-zone ap-southeast-1a \
  --size 100 \
  --volume-type gp3 \
  --encrypted \
  --kms-key-id "alias/production-ebs-key"

# Encrypt volume chưa được encrypt (phải snapshot → copy → restore)
# 1. Tạo snapshot của volume unencrypted
SNAPSHOT_ID=$(aws ec2 create-snapshot \
  --volume-id vol-1234567890abcdef0 \
  --description "Pre-encryption snapshot" \
  --query 'SnapshotId' --output text)

# 2. Copy snapshot với encryption bật
ENCRYPTED_SNAPSHOT=$(aws ec2 copy-snapshot \
  --source-snapshot-id $SNAPSHOT_ID \
  --source-region ap-southeast-1 \
  --encrypted \
  --kms-key-id "alias/production-ebs-key" \
  --query 'SnapshotId' --output text)

# 3. Tạo volume mới từ encrypted snapshot
aws ec2 create-volume \
  --snapshot-id $ENCRYPTED_SNAPSHOT \
  --availability-zone ap-southeast-1a

# 4. Detach volume cũ, attach volume mới, test, xóa volume cũ
```

### Kiểm Tra Encryption Status

```bash
# Xem encryption status của tất cả volumes
aws ec2 describe-volumes \
  --query 'Volumes[*].[VolumeId,Encrypted,KmsKeyId,State]' \
  --output table

# Tìm volumes chưa được encrypt
aws ec2 describe-volumes \
  --filters Name=encrypted,Values=false \
  --query 'Volumes[*].VolumeId'

# Xem encryption status của EC2 root volume
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,RootDeviceEncrypted:BlockDeviceMappings[0].Ebs.Encrypted}'
```

### Launch Template Với Encrypted EBS

```json
// Launch Template đảm bảo mọi instance đều có encrypted volumes
{
  "BlockDeviceMappings": [
    {
      "DeviceName": "/dev/xvda",
      "Ebs": {
        "VolumeSize": 50,
        "VolumeType": "gp3",
        "Encrypted": true,
        "KmsKeyId": "arn:aws:kms:ap-southeast-1:123456789012:key/key-id",
        "DeleteOnTermination": true
      }
    },
    {
      "DeviceName": "/dev/xvdb",
      "Ebs": {
        "VolumeSize": 200,
        "VolumeType": "gp3",
        "Encrypted": true,
        "KmsKeyId": "arn:aws:kms:ap-southeast-1:123456789012:key/key-id",
        "DeleteOnTermination": false
      }
    }
  ]
}
```

---

## Encryption In Transit — Mã Hóa Khi Truyền

### TLS/HTTPS Cho Web Traffic

```
TLS (Transport Layer Security — Bảo Mật Tầng Truyền Tải):
└── Giao thức mã hóa toàn bộ data khi truyền qua mạng

AWS Certificate Manager — ACM:
├── Cung cấp SSL/TLS certificates miễn phí
├── Tự động renewal (gia hạn tự động)
├── Tích hợp với ALB, CloudFront, API Gateway
└── Không thể export certificate (chỉ dùng với AWS services)

Luồng HTTPS:
Client → [TLS Handshake] → ALB (TLS Termination) → EC2 (HTTP)
                                    hoặc
Client → [TLS Handshake] → ALB → [TLS Re-encryption] → EC2 (HTTPS)
```

```bash
# Yêu cầu certificate từ ACM
aws acm request-certificate \
  --domain-name "api.mycompany.com" \
  --validation-method DNS \
  --subject-alternative-names "*.mycompany.com"

# Xem danh sách certificates
aws acm list-certificates \
  --certificate-statuses ISSUED

# Import certificate của bên thứ ba
aws acm import-certificate \
  --certificate fileb://certificate.pem \
  --private-key fileb://private-key.pem \
  --certificate-chain fileb://chain.pem
```

### HTTPS Listener Trên ALB

```bash
# Tạo HTTPS listener trên ALB
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:...:certificate/cert-id \
  --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:...

# Redirect HTTP → HTTPS
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'
```

### VPC Endpoints — Traffic Không Ra Internet

```bash
# VPC Endpoint cho S3 (Gateway endpoint — miễn phí)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-1234567890abcdef0 \
  --service-name com.amazonaws.ap-southeast-1.s3 \
  --route-table-ids rtb-1234567890abcdef0

# VPC Interface Endpoint cho Secrets Manager (Interface endpoint — có phí)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-1234567890abcdef0 \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.ap-southeast-1.secretsmanager \
  --subnet-ids subnet-1234567890abcdef0 \
  --security-group-ids sg-1234567890abcdef0 \
  --private-dns-enabled

# Với VPC Endpoints:
# → Traffic từ EC2 → S3/Secrets Manager/SSM đi qua AWS backbone
# → Không ra internet công cộng
# → Tăng security và có thể giảm latency
```

### Encryption Cho Database Connections

```python
# Kết nối RDS với SSL/TLS
import psycopg2
import ssl

# Download RDS CA certificate
# wget https://truststore.pki.rds.amazonaws.com/ap-southeast-1/ap-southeast-1-bundle.pem

conn = psycopg2.connect(
    host="mydb.cluster.ap-southeast-1.rds.amazonaws.com",
    database="mydb",
    user="appuser",
    password="password",
    sslmode="verify-full",          # Verify CA certificate
    sslrootcert="/etc/ssl/rds-ca.pem"
)
```

```bash
# Enforce SSL cho RDS (parameter group)
aws rds modify-db-parameter-group \
  --db-parameter-group-name mydb-params \
  --parameters "ParameterName=rds.force_ssl,ParameterValue=1,ApplyMethod=immediate"
```

---

## Encryption Các Service Khác

### S3 Encryption

```bash
# S3 Server-Side Encryption (Mã Hóa Phía Máy Chủ) options:

# SSE-S3 (AES-256): AWS quản lý key, miễn phí
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      },
      "BucketKeyEnabled": true
    }]
  }'

# SSE-KMS: Dùng KMS key, audit được, có chi phí KMS API
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:...:key/..."
      },
      "BucketKeyEnabled": true   ← Giảm chi phí KMS API calls
    }]
  }'

# Enforce encryption policy (từ chối upload không encrypt)
aws s3api put-bucket-policy \
  --bucket my-bucket \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": ["aws:kms", "AES256"]
        }
      }
    }]
  }'
```

### EFS Encryption — Mã Hóa File System

```bash
# Tạo EFS với encryption at rest
aws efs create-file-system \
  --encrypted \
  --kms-key-id "arn:aws:kms:ap-southeast-1:123456789012:key/key-id" \
  --performance-mode generalPurpose

# Mount với TLS (encryption in transit)
sudo mount -t efs \
  -o tls \
  fs-1234567890abcdef0:/ /mnt/efs

# Hoặc dùng EFS mount helper trong fstab
# fs-1234567890abcdef0 /mnt/efs efs tls,_netdev 0 0
```

### RDS Encryption

```bash
# Bật encryption khi tạo RDS (không thể bật sau!)
aws rds create-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username admin \
  --master-user-password password \
  --storage-encrypted \
  --kms-key-id "arn:aws:kms:...:key/..."

# Encrypt RDS đang chạy (qua snapshot):
# 1. Tạo snapshot
SNAP=$(aws rds create-db-snapshot \
  --db-instance-identifier mydb \
  --db-snapshot-identifier mydb-snap \
  --query 'DBSnapshot.DBSnapshotIdentifier' --output text)

# 2. Copy snapshot với encryption
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier $SNAP \
  --target-db-snapshot-identifier mydb-snap-encrypted \
  --kms-key-id "alias/production-rds-key"

# 3. Restore từ encrypted snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier mydb-new \
  --db-snapshot-identifier mydb-snap-encrypted
```

---

## Envelope Encryption — Mã Hóa Phong Bì

### Khái Niệm Quan Trọng

```
Vấn Đề: Mã hóa 10GB dữ liệu bằng KMS → gửi 10GB lên KMS API?
→ Không thể! KMS chỉ mã hóa data tối đa 4KB

Giải Pháp: Envelope Encryption (Mã Hóa Phong Bì)

Bước 1: KMS tạo Data Encryption Key (DEK — Khóa Mã Hóa Dữ Liệu)
         → Trả về: plaintext DEK + encrypted DEK

Bước 2: Dùng plaintext DEK để mã hóa dữ liệu lớn (local, không gửi qua API)

Bước 3: Lưu encrypted DEK cùng với encrypted data
         (bỏ plaintext DEK khỏi memory!)

Bước 4: Để decrypt: gửi encrypted DEK lên KMS để lấy plaintext DEK
         → Dùng plaintext DEK để decrypt data

Phong Bì (Envelope):
┌─────────────────────────────────────┐
│  Encrypted Data (mã hóa bởi DEK)   │
│  ┌───────────────────────────────┐  │
│  │  Encrypted DEK (mã hóa bởi   │  │
│  │  KMS CMK)                     │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
→ Chỉ gửi encrypted DEK (nhỏ) lên KMS, không gửi toàn bộ data
```

```python
# Implement envelope encryption với Python + boto3
import boto3
import os
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

def encrypt_data(kms_key_id: str, plaintext_data: bytes) -> dict:
    kms = boto3.client('kms')

    # Bước 1: Tạo Data Encryption Key
    response = kms.generate_data_key(
        KeyId=kms_key_id,
        KeySpec='AES_256'
    )
    plaintext_dek = response['Plaintext']       # Dùng để encrypt, xóa sau
    encrypted_dek = response['CiphertextBlob']  # Lưu cùng data

    # Bước 2: Mã hóa data với DEK (local, không cần network)
    nonce = os.urandom(12)
    aesgcm = AESGCM(plaintext_dek)
    ciphertext = aesgcm.encrypt(nonce, plaintext_data, None)

    # Bước 3: Xóa plaintext DEK khỏi memory
    del plaintext_dek

    return {
        'encrypted_dek': encrypted_dek,  # Lưu cùng data
        'nonce': nonce,
        'ciphertext': ciphertext
    }

def decrypt_data(kms_key_id: str, encrypted_package: dict) -> bytes:
    kms = boto3.client('kms')

    # Bước 4: Gửi encrypted DEK lên KMS để lấy plaintext DEK
    response = kms.decrypt(
        KeyId=kms_key_id,
        CiphertextBlob=encrypted_package['encrypted_dek']
    )
    plaintext_dek = response['Plaintext']

    # Giải mã data với DEK
    aesgcm = AESGCM(plaintext_dek)
    plaintext = aesgcm.decrypt(
        encrypted_package['nonce'],
        encrypted_package['ciphertext'],
        None
    )

    del plaintext_dek
    return plaintext
```

---

## AWS Config Rules Cho Encryption Compliance

```bash
# Tạo Config Rule để phát hiện EBS volumes chưa encrypt
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "encrypted-volumes",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "ENCRYPTED_VOLUMES"
    },
    "Scope": {
      "ComplianceResourceTypes": ["AWS::EC2::Volume"]
    }
  }'

# Các AWS Managed Config Rules liên quan encryption:
# - encrypted-volumes: EBS volumes must be encrypted
# - rds-storage-encrypted: RDS instances must be encrypted
# - s3-bucket-server-side-encryption-enabled: S3 encryption bắt buộc
# - efs-encrypted-check: EFS file systems phải encrypted
# - dynamodb-table-encrypted-at-rest: DynamoDB encryption
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Giải thích Envelope Encryption và tại sao AWS dùng nó?

**Trả lời:** Envelope Encryption (Mã Hóa Phong Bì) giải quyết vấn đề KMS chỉ có thể xử lý data tối đa 4KB. Thay vì gửi toàn bộ data lên KMS, AWS tạo một Data Encryption Key (DEK — Khóa Mã Hóa Dữ Liệu) tạm thời: KMS trả về cả plaintext DEK lẫn encrypted DEK. Application dùng plaintext DEK để mã hóa data lớn locally (không qua network), sau đó xóa plaintext DEK. Lưu encrypted DEK cùng với encrypted data. Khi cần decrypt, gửi encrypted DEK lên KMS để lấy plaintext DEK, rồi dùng decrypt data. Lợi ích: chỉ gửi DEK nhỏ qua API (giảm latency, chi phí), data lớn không bao giờ rời khỏi application server.

### Q2: EBS Encryption bao gồm những gì và cách bật cho cả account?

**Trả lời:** EBS Encryption mã hóa toàn bộ: volume data, disk I/O (Input/Output — Nhập/Xuất), snapshots tạo từ volume, và volumes tạo từ snapshots đó. Encryption hoạt động ở hypervisor level — application không cần thay đổi gì, không có performance impact đáng kể (hardware AES acceleration). Để bật cho cả account và region, dùng `aws ec2 enable-ebs-encryption-by-default` — sau đó mọi EBS volume mới đều tự động được encrypt bằng default KMS key (hoặc key bạn chỉ định). Lưu ý quan trọng: encryption phải bật lúc tạo volume — không thể encrypt volume đang chạy trực tiếp. Phải làm qua: snapshot → copy với encryption → tạo volume mới.

### Q3: Sự khác biệt giữa SSE-S3, SSE-KMS và SSE-C trong S3?

**Trả lời:** SSE-S3 (Server-Side Encryption với S3 Managed Keys — Khóa S3 Quản Lý): AWS tự tạo và quản lý key, miễn phí, không có audit trail riêng cho key usage, dùng AES-256. SSE-KMS (Server-Side Encryption với KMS): dùng Customer Managed Key hoặc AWS Managed Key trong KMS, có chi phí KMS API calls nhưng có đầy đủ audit trail qua CloudTrail (biết ai đã encrypt/decrypt, lúc nào), có thể control access qua Key Policy. SSE-C (Server-Side Encryption với Customer-Provided Key — Khóa Khách Hàng Tự Cung Cấp): bạn gửi key trong mỗi request, AWS encrypt và ngay lập tức xóa key, bạn phải quản lý key hoàn toàn. Trong production, SSE-KMS là lựa chọn tốt nhất vì balance giữa security và manageability.

---

## 🔗 Điều Hướng

| | |
|--|--|
| ← Secrets Management | [3-secrets-management.md](./3-secrets-management.md) |
| → Compliance & Patching | [5-compliance-patching.md](./5-compliance-patching.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15
