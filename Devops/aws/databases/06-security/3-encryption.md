# 3 — Encryption — Mã Hóa Dữ Liệu Database

> Encryption (Mã Hóa) bảo vệ dữ liệu kể cả khi attacker có quyền truy cập vật lý vào storage hoặc intercept traffic. AWS cung cấp mã hóa toàn diện ở cả hai trạng thái: **at rest** (khi lưu trữ trên đĩa) và **in transit** (khi truyền qua mạng).

---

## 📚 Mục Lục

1. [Tổng Quan Encryption Cho Database](#1-tổng-quan-encryption-cho-database)
2. [KMS — Key Management Service](#2-kms--key-management-service)
3. [Encryption at Rest — Mã Hóa Khi Lưu Trữ](#3-encryption-at-rest--mã-hóa-khi-lưu-trữ)
4. [Encryption in Transit — Mã Hóa Khi Truyền Tải](#4-encryption-in-transit--mã-hóa-khi-truyền-tải)
5. [RDS & Aurora Encryption Chi Tiết](#5-rds--aurora-encryption-chi-tiết)
6. [DynamoDB Encryption](#6-dynamodb-encryption)
7. [ElastiCache Encryption](#7-elasticache-encryption)
8. [Key Rotation & Compliance](#8-key-rotation--compliance)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan Encryption Cho Database

### Hai Trạng Thái Cần Bảo Vệ

```
DATA AT REST (Dữ Liệu Khi Lưu Trữ)
───────────────────────────────────
• Dữ liệu trên EBS volumes của RDS
• Automated backups, snapshots
• DynamoDB table data trên SSDs
• ElastiCache data nếu có persistence (AOF/RDB)
• Transaction logs, redo logs, binlogs
• Read replica storage

→ Mã hóa bằng: AES-256 (Advanced Encryption Standard - 256-bit)
→ Quản lý khóa bằng: KMS (Key Management Service)

DATA IN TRANSIT (Dữ Liệu Khi Truyền Tải)
──────────────────────────────────────────
• Kết nối từ app đến database (JDBC, MySQL protocol)
• Replication giữa Primary và Read Replicas
• Replication Multi-AZ Primary → Standby
• Cross-region replication (Aurora Global)
• Kết nối từ admin tools (MySQL Workbench, DBeaver)

→ Mã hóa bằng: TLS 1.2 / TLS 1.3
→ Certificate: AWS-issued SSL certificates
```

### Encryption vs Security

```
Quan niệm sai: "Encrypt là bảo mật đủ rồi"

Thực tế:
Encryption at rest BẢO VỆ khỏi:
✅ Ai đó lấy cắp physical disk
✅ Backup file bị leaked
✅ Snapshot bị shared ra ngoài

Encryption at rest KHÔNG BẢO VỆ khỏi:
❌ SQL injection (dữ liệu được decrypt trước khi query)
❌ Compromised database credentials
❌ Overly-permissive IAM policies
❌ Malicious insider với database access

→ Encryption là 1 layer — cần kết hợp với VPC, IAM, audit
```

---

## 2. KMS — Key Management Service

### KMS Là Gì?

**KMS** (Key Management Service — Dịch Vụ Quản Lý Khóa) là dịch vụ AWS quản lý khóa mã hóa tập trung. KMS keys (CMK — Customer Master Key, hiện gọi là KMS Key) được lưu trong HSM (Hardware Security Module — Module Bảo Mật Phần Cứng) đạt chuẩn FIPS 140-2.

### Ba Loại KMS Keys

```
1. AWS Managed Keys (Khóa Được AWS Quản Lý)
   ──────────────────────────────────────────
   Ví dụ key alias: aws/rds, aws/dynamodb, aws/elasticache
   • AWS tự tạo, tự quản lý, tự rotate (mỗi 3 năm)
   • Bạn không thể control policy, disable, hay delete
   • Miễn phí
   • Phù hợp cho: non-sensitive workloads, start nhanh
   • Audit: biết service nào dùng, nhưng ít control

2. Customer Managed Keys — CMK (Khóa Do Khách Hàng Quản Lý)
   ─────────────────────────────────────────────────────────
   Ví dụ key alias: alias/prod-rds-key, alias/payments-db-key
   • Bạn tạo và quản lý
   • Control policy: ai được dùng key này
   • Có thể enable/disable, schedule deletion
   • Có thể rotate hàng năm (hoặc on-demand)
   • Chi phí: $1/key/tháng + $0.03/10K API calls
   • Phù hợp cho: sensitive data, compliance requirements
   • Audit: CloudTrail log mọi key usage

3. AWS Owned Keys (Khóa Thuộc Sở Hữu AWS)
   ─────────────────────────────────────────
   • Dùng nội bộ bởi AWS service cho multi-tenant operations
   • Không visible với customers, không control được
   • Miễn phí, không audit trail
   • Phù hợp cho: ephemeral, non-critical data
```

### KMS Key Policy — Chính Sách Khóa

```json
// Key Policy cho prod-rds-encryption-key
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "KeyAdministration",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/key-admin-role"
      },
      "Action": [
        "kms:Create*", "kms:Describe*", "kms:Enable*",
        "kms:List*", "kms:Put*", "kms:Update*",
        "kms:Revoke*", "kms:Disable*", "kms:Get*",
        "kms:Delete*", "kms:ScheduleKeyDeletion",
        "kms:CancelKeyDeletion"
      ],
      "Resource": "*"
    },
    {
      "Sid": "RDSServiceUsage",
      "Effect": "Allow",
      "Principal": {
        "Service": "rds.amazonaws.com"
      },
      "Action": [
        "kms:GenerateDataKey",
        "kms:Decrypt",
        "kms:DescribeKey",
        "kms:CreateGrant"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ApplicationDecrypt",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/app-server-role"
      },
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "rds.ap-southeast-1.amazonaws.com"
          // Chỉ được dùng key thông qua RDS service, không dùng trực tiếp
        }
      }
    }
  ]
}
```

### Envelope Encryption — Mã Hóa Phong Bì

KMS không mã hóa data trực tiếp bằng KMS Key — thay vào đó dùng kỹ thuật **Envelope Encryption** (Mã Hóa Phong Bì):

```
Vấn đề: KMS chỉ encrypt/decrypt tối đa 4KB dữ liệu trực tiếp
         Database data có thể hàng TB → không thể gửi cả lên KMS

Giải pháp — Envelope Encryption:
┌────────────────────────────────────────────────────────┐
│                                                          │
│  KMS Key (CMK) ─────────────┐                           │
│        ↓                    │                           │
│  Encrypt DEK (32 bytes)     │ KMS API call               │
│        ↓                    │ (chỉ DEK, không phải data) │
│  Encrypted DEK              │                           │
│        ↑                    │                           │
│  KMS Key (CMK) ─────────────┘                           │
│                                                          │
│  Data Encryption Key (DEK) — Khóa Mã Hóa Dữ Liệu       │
│        ↓                                                 │
│  AES-256 encrypt ──────────── Database Data (TB)        │
│        ↓                                                 │
│  Encrypted Data (lưu cùng Encrypted DEK trên disk)      │
│                                                          │
└────────────────────────────────────────────────────────┘

Khi cần đọc:
1. Lấy Encrypted DEK từ disk
2. Gọi KMS: Decrypt(Encrypted DEK) → Plain DEK
3. Dùng Plain DEK decrypt data
4. Xóa Plain DEK khỏi memory sau khi dùng
→ KMS Key không bao giờ rời HSM
```

---

## 3. Encryption at Rest — Mã Hóa Khi Lưu Trữ

### Mọi Thứ Được Mã Hóa Khi Enable

Khi enable encryption at rest cho RDS, **tất cả** đều được mã hóa tự động:

```
RDS Encryption at Rest bao gồm:
├── Database data files trên EBS
├── Automated backups
├── Manual snapshots
├── Read replicas
├── Transaction logs
├── Temporary files (sort, temp tables)
└── Audit logs

ElastiCache Encryption at Rest bao gồm:
├── Snapshot files (RDB)
├── Swap files
├── Synchronization và backup data
└── AOF files (nếu bật)
```

### Không Thể Encrypt Sau Khi Tạo

**Hạn chế quan trọng**: Không thể enable encryption cho RDS instance đang chạy (unencrypted):

```
Workaround — Cách encrypt RDS đã tồn tại:

Bước 1: Tạo snapshot của instance unencrypted
aws rds create-db-snapshot \
  --db-instance-identifier prod-db-unencrypted \
  --db-snapshot-identifier migration-snapshot

Bước 2: Copy snapshot với encryption enabled
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier migration-snapshot \
  --target-db-snapshot-identifier migration-snapshot-encrypted \
  --kms-key-id arn:aws:kms:ap-southeast-1:123456789012:key/mrk-xxxxx

Bước 3: Restore từ encrypted snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-db-new-encrypted \
  --db-snapshot-identifier migration-snapshot-encrypted

Bước 4: Update DNS/connection string, test, rồi cutover
Bước 5: Xóa instance cũ sau khi verified
```

---

## 4. Encryption in Transit — Mã Hóa Khi Truyền Tải

### TLS/SSL Cho RDS Connections

**TLS** (Transport Layer Security — Bảo Mật Lớp Truyền Tải) mã hóa data channel giữa client và database:

```
Application Server                    RDS/Aurora
        │                                 │
        │   TCP Handshake (port 3306)     │
        │ ────────────────────────────► │
        │   TLS Handshake                 │
        │ ◄──────────── Certificate ──── │
        │   Verify cert (check CA)        │
        │ ──── Encrypted Key Exchange ─► │
        │   Encrypted Channel Established │
        │ ◄══════ Encrypted Data ════════ │
        │ ═══════ Encrypted Data ════════► │
```

### RDS Certificate Authority (CA)

AWS RDS dùng CA certificate riêng. Cần download và trust CA cert:

```bash
# Download RDS CA bundle
wget https://truststore.pki.rds.amazonaws.com/ap-southeast-1/ap-southeast-1-bundle.pem

# Kiểm tra cert expiry
openssl x509 -in ap-southeast-1-bundle.pem -noout -dates

# Kết nối MySQL với SSL verification
mysql -h mydb.xxx.ap-southeast-1.rds.amazonaws.com \
  -u admin -p \
  --ssl-ca=ap-southeast-1-bundle.pem \
  --ssl-mode=VERIFY_CA   # Bắt buộc verify certificate
```

### Enforce SSL/TLS Trên RDS

```sql
-- MySQL/Aurora MySQL: Bật require SSL cho user
ALTER USER 'app_user'@'%' REQUIRE SSL;

-- Kiểm tra user có require SSL không
SELECT user, host, ssl_type FROM mysql.user;
```

```bash
# Hoặc qua Parameter Group — enforce cho toàn bộ instance
# MySQL: require_secure_transport = ON
aws rds modify-db-parameter-group \
  --db-parameter-group-name prod-mysql-params \
  --parameters "ParameterName=require_secure_transport,ParameterValue=1,ApplyMethod=immediate"

# PostgreSQL: ssl = 1 (không cho phép non-SSL connections)
aws rds modify-db-parameter-group \
  --db-parameter-group-name prod-pg-params \
  --parameters "ParameterName=rds.force_ssl,ParameterValue=1,ApplyMethod=immediate"
```

### Connection String Với SSL

```python
# Python + MySQL connector
import mysql.connector

conn = mysql.connector.connect(
    host='mydb.xxx.ap-southeast-1.rds.amazonaws.com',
    user='app_user',
    password=get_secret_from_secrets_manager(),
    database='myapp',
    ssl_ca='/etc/ssl/rds-ca/ap-southeast-1-bundle.pem',
    ssl_verify_cert=True,    # Bắt buộc verify CA
    ssl_verify_identity=True # Verify hostname khớp với cert
)
```

```java
// Java JDBC với SSL
String url = "jdbc:mysql://mydb.xxx.ap-southeast-1.rds.amazonaws.com:3306/myapp"
           + "?useSSL=true"
           + "&requireSSL=true"
           + "&trustCertificateKeyStoreUrl=file:rds-truststore.jks"
           + "&trustCertificateKeyStorePassword=changeit"
           + "&verifyServerCertificate=true";
```

### TLS Versions và Ciphers

AWS RDS hỗ trợ TLS 1.0, 1.1, 1.2, 1.3. Best practice:

```bash
# MySQL: Chỉ cho phép TLS 1.2 trở lên
aws rds modify-db-parameter-group \
  --parameters "ParameterName=tls_version,ParameterValue=TLSv1.2\,TLSv1.3,ApplyMethod=pending-reboot"

# Kiểm tra version TLS đang dùng (trong MySQL)
SHOW STATUS LIKE 'Ssl_version';
-- Kết quả: TLSv1.3
```

---

## 5. RDS & Aurora Encryption Chi Tiết

### Enable Encryption Khi Tạo RDS

```bash
aws rds create-db-instance \
  --db-instance-identifier prod-mysql \
  --db-instance-class db.r6g.2xlarge \
  --engine mysql \
  --master-username admin \
  --master-user-password $(aws secretsmanager get-secret-value ...) \
  --storage-encrypted \                    # ← Enable encryption at rest
  --kms-key-id arn:aws:kms:ap-southeast-1:123456789012:key/mrk-xxxxx \  # ← CMK
  --multi-az \
  --db-subnet-group-name prod-db-subnets \
  --vpc-security-group-ids sg-db-mysql
```

### Snapshot Encryption & Cross-Region Copy

```bash
# Snapshot tự động inherit encryption từ source instance
# Manual snapshot của encrypted instance → cũng encrypted tự động

# Cross-region copy phải chỉ định CMK của region đích
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:ap-southeast-1:123456789012:snapshot:prod-snapshot \
  --target-db-snapshot-identifier prod-snapshot-us-east \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/mrk-yyyyy \
  # ← KMS Multi-Region Key (mrk-) cho phép decrypt ở nhiều region
  --source-region ap-southeast-1
```

### KMS Multi-Region Keys — Khóa Đa Vùng

```
MRK (Multi-Region Key — Khóa Đa Vùng) là innovation của AWS:
Cùng key material, cùng key ID prefix (mrk-), có thể dùng ở nhiều regions

Lợi ích cho database:
├── Snapshot copy cross-region không cần re-encrypt (nhanh hơn)
├── Aurora Global Database dùng cùng key ở tất cả regions
└── DR setup đơn giản hơn — không cần manage nhiều keys riêng

aws kms replicate-key \
  --key-id mrk-xxxxx \
  --replica-region us-east-1
```

---

## 6. DynamoDB Encryption

### DynamoDB Luôn Encrypted

DynamoDB **luôn encrypt at rest** — không có option để disable. Chỉ có thể chọn loại key:

```bash
# Option 1: AWS Owned Key (mặc định, miễn phí)
aws dynamodb create-table \
  --table-name orders \
  --sse-specification Enabled=true,SSEType=AES256
  # AES256 = AWS Owned Key

# Option 2: AWS Managed Key (aws/dynamodb)
aws dynamodb create-table \
  --table-name orders \
  --sse-specification Enabled=true,SSEType=KMS
  # Không specify KMSMasterKeyId → dùng aws/dynamodb key

# Option 3: Customer Managed Key (CMK) — Recommended cho sensitive data
aws dynamodb create-table \
  --table-name orders \
  --sse-specification Enabled=true,SSEType=KMS,KMSMasterKeyId=arn:aws:kms:...
```

### DynamoDB In Transit

```python
# DynamoDB SDK tự động dùng HTTPS (TLS)
import boto3
client = boto3.client('dynamodb', region_name='ap-southeast-1')
# → Tự động connect qua https://dynamodb.ap-southeast-1.amazonaws.com
# → TLS 1.2+ built-in, không cần config thêm

# Verify bằng endpoint URL
client = boto3.client(
    'dynamodb',
    endpoint_url='https://dynamodb.ap-southeast-1.amazonaws.com'  # HTTPS
)
```

### DynamoDB Client-Side Encryption — Mã Hóa Phía Client

Cho data cực kỳ nhạy cảm, có thể encrypt trước khi gửi lên DynamoDB:

```python
# Dùng AWS DynamoDB Encryption Client (thư viện riêng)
from dynamodb_encryption_sdk import EncryptedClient
from dynamodb_encryption_sdk.identifiers import CryptoAction
from dynamodb_encryption_sdk.material_providers.aws_kms import AwsKmsCryptographicMaterialsProvider

kms_cmp = AwsKmsCryptographicMaterialsProvider(key_id='arn:aws:kms:...')

encrypted_client = EncryptedClient(
    client=boto3.client('dynamodb'),
    encryption_materials_provider=kms_cmp,
    attribute_actions={
        'credit_card': CryptoAction.ENCRYPT_AND_SIGN,   # Mã hóa field nhạy cảm
        'ssn':         CryptoAction.ENCRYPT_AND_SIGN,
        'user_id':     CryptoAction.SIGN_ONLY,           # Chỉ sign, không encrypt (cần query)
        'status':      CryptoAction.DO_NOTHING           # Không mã hóa
    }
)

# Data tự động encrypt trước khi lên DynamoDB
encrypted_client.put_item(TableName='users', Item={...})
```

---

## 7. ElastiCache Encryption

### Redis Encryption

```bash
# Tạo ElastiCache Redis với encryption đầy đủ
aws elasticache create-replication-group \
  --replication-group-id prod-redis \
  --replication-group-description "Production Redis" \
  --at-rest-encryption-enabled \           # ← Encryption at rest
  --transit-encryption-enabled \           # ← TLS in transit (AUTH cũng bắt buộc)
  --auth-token "$(generate-strong-password)" \  # ← Redis AUTH password
  --kms-key-id arn:aws:kms:...:key/xxx \   # ← CMK cho at-rest encryption
  --cache-node-type cache.r6g.large \
  --num-cache-clusters 3 \
  --cache-subnet-group-name prod-cache-subnets \
  --security-group-ids sg-redis
```

### Memcached Encryption

```bash
# Memcached chỉ hỗ trợ in-transit encryption (không có at-rest)
aws elasticache create-cache-cluster \
  --cache-cluster-id prod-memcached \
  --engine memcached \
  --transit-encryption-enabled \  # ← TLS only (không có at-rest)
  --cache-node-type cache.t3.medium
```

### Redis Connection Với TLS

```python
import redis

# Kết nối với TLS
r = redis.Redis(
    host='prod-redis.xxx.cache.amazonaws.com',
    port=6380,           # TLS port (6380 thay vì 6379)
    ssl=True,
    ssl_cert_reqs='required',
    ssl_ca_certs='/etc/ssl/certs/AmazonRootCA1.pem',
    password='your-auth-token',   # Redis AUTH
    decode_responses=True
)
```

---

## 8. Key Rotation & Compliance

### Automatic Key Rotation — Tự Động Xoay Vòng Khóa

```bash
# Enable annual rotation cho CMK
aws kms enable-key-rotation --key-id arn:aws:kms:...:key/mrk-xxxxx

# Kiểm tra rotation status
aws kms get-key-rotation-status --key-id mrk-xxxxx
# → {"KeyRotationEnabled": true}

# Rotation hoạt động thế nào:
# 1. AWS tạo new key material hàng năm
# 2. Key cũ giữ lại để decrypt data đã encrypt trước đó
# 3. Key mới tự động dùng cho encrypt mới
# 4. Không downtime, transparent với application
# 5. Key ID không thay đổi → không cần update code
```

### On-Demand Key Rotation (Available từ 2023)

```bash
# Rotate ngay lập tức (không cần đợi 1 năm)
# Dùng khi: key bị suspected compromise, compliance requirement
aws kms rotate-key-on-demand --key-id mrk-xxxxx

# Xem rotation history
aws kms list-key-rotations --key-id mrk-xxxxx
# Trả về danh sách mỗi lần rotation với timestamp
```

### Encryption Compliance Overview

| Framework | Yêu Cầu Encryption | AWS Service |
|-----------|-------------------|-------------|
| **PCI-DSS** (Payment Card Industry — Tiêu Chuẩn Bảo Mật Ngành Thẻ) | Encrypt cardholder data at rest & in transit | KMS CMK + TLS |
| **HIPAA** (Health Insurance Portability — Tính Di Động Bảo Hiểm Y Tế) | Encrypt PHI (Protected Health Info) | KMS CMK required |
| **GDPR** (General Data Protection Regulation — Quy Định Bảo Vệ Dữ Liệu Chung) | Pseudonymization & encryption recommended | KMS + Client-side encryption |
| **SOC 2 Type II** | Encryption at rest & in transit | KMS + TLS + documented key management |
| **FedRAMP** | FIPS 140-2 validated encryption | KMS (HSM backed) |

### KMS Key Deletion — Xóa Khóa

```bash
# NGUY HIỂM: Schedule deletion (7-30 ngày waiting period)
# Sau khi xóa → không thể decrypt data được encrypt bằng key này

# Schedule deletion
aws kms schedule-key-deletion \
  --key-id mrk-xxxxx \
  --pending-window-in-days 14   # 14 ngày để cancel nếu nhầm

# Cancel trước khi hết waiting period
aws kms cancel-key-deletion --key-id mrk-xxxxx

# Best practice: Disable key trước khi xóa
aws kms disable-key --key-id mrk-xxxxx
# → Test xem gì break → Đây là cách phát hiện dependencies
# Sau khi confident → mới schedule deletion
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Phân biệt AWS Managed Key và Customer Managed Key (CMK). Khi nào dùng CMK?**

> AWS Managed Key: AWS tự quản lý, tự rotate, miễn phí, ít control. CMK: bạn tự manage, control policy chi tiết, có thể disable/delete, $1/key/tháng. Dùng CMK khi: (1) compliance yêu cầu customer-controlled keys (HIPAA, PCI-DSS); (2) cần control ai được dùng key (cross-account, specific services); (3) cần audit chi tiết mọi key usage; (4) cần khả năng revoke access ngay lập tức bằng cách disable key.

**Q: Tại sao không thể enable encryption cho RDS instance đang chạy? Workaround là gì?**

> Encryption là property được thiết lập lúc khởi tạo EBS volume, không thể thay đổi sau. Workaround: (1) Tạo snapshot của instance unencrypted; (2) Copy snapshot với encryption enabled (chỉ định CMK); (3) Restore RDS từ encrypted snapshot; (4) Update DNS endpoint (CNAME), test thorough; (5) Cutover traffic sang instance mới; (6) Xóa instance cũ. Downtime có thể giảm thiểu bằng cách setup replication và sync trước khi cutover.

**Q: Giải thích Envelope Encryption trong KMS.**

> KMS không encrypt data trực tiếp (giới hạn 4KB). Thay vào đó: (1) KMS tạo Data Encryption Key (DEK); (2) DEK encrypt data thực bằng AES-256 (GB/TB data); (3) KMS encrypt DEK bằng CMK → Encrypted DEK; (4) Encrypted DEK lưu cùng data; (5) Khi cần đọc: gọi KMS decrypt Encrypted DEK → plain DEK → decrypt data. CMK không bao giờ rời khỏi HSM — chỉ DEK được gửi đi.

**Q: Làm thế nào enforce TLS cho RDS MySQL?**

> Hai cách: (1) Cấp per-user: `ALTER USER 'app_user'@'%' REQUIRE SSL;` — user đó không thể connect không có SSL; (2) Instance-wide qua Parameter Group: set `require_secure_transport=ON` — toàn bộ instance từ chối non-SSL connections. Cả hai cần kết hợp với CA certificate verification ở phía client (`ssl_verify_cert=True`) để tránh MITM (Man-in-the-Middle — Tấn Công Đứng Giữa) attack.

---

## 📂 Điều Hướng

| ← Trước | Chủ Đề Hiện Tại | Tiếp → |
|---------|-----------------|--------|
| [2-iam-authentication.md](./2-iam-authentication.md) | **3-encryption.md** | [4-secrets-management.md](./4-secrets-management.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15 | **Trạng Thái:** ✅ Hoàn Thành
