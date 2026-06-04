# SSM Parameter Store — Quản Lý Cấu Hình & Secret Phân Cấp

> **Parameter Store** (Kho Tham Số) là dịch vụ lưu trữ cấu hình (configuration) và secret (bí mật) có phân cấp (hierarchical), phiên bản (versioning), kiểm soát truy cập (access control) và mã hóa (encryption) — tích hợp sâu với hầu hết dịch vụ AWS.

---

## 📚 Mục Lục

1. [Khái Niệm Cốt Lõi](#khái-niệm-cốt-lõi)
2. [Loại Parameter](#loại-parameter)
3. [Standard vs Advanced Tier](#standard-vs-advanced-tier)
4. [SecureString & Mã Hóa KMS](#securestring--mã-hóa-kms)
5. [Phân Cấp & Đặt Tên](#phân-cấp--đặt-tên)
6. [Versioning & Labels](#versioning--labels)
7. [Tích Hợp Với AWS Services](#tích-hợp-với-aws-services)
8. [Parameter Store vs Secrets Manager](#parameter-store-vs-secrets-manager)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khái Niệm Cốt Lõi

### Vì Sao Cần Parameter Store?

```
Vấn đề với hardcode config:
┌──────────────────────────────────────────────────────────┐
│  Code:  DB_HOST = "prod-db.cluster.ap-southeast-1.rds"  │
│         DB_PASS = "SuperSecretP@ssw0rd123"              │
│                                                          │
│  Vấn đề:                                                 │
│  ✗ Secret lộ trong source code / Git history             │
│  ✗ Thay đổi config → phải redeploy code                  │
│  ✗ Không có audit trail (ai đổi gì, khi nào)             │
│  ✗ Mỗi môi trường (dev/staging/prod) phải dùng file .env │
└──────────────────────────────────────────────────────────┘

Giải pháp với Parameter Store:
┌──────────────────────────────────────────────────────────┐
│  Code:  DB_HOST = ssm.get("/prod/myapp/db/host")         │
│         DB_PASS = ssm.get("/prod/myapp/db/password")     │
│                                                          │
│  Lợi ích:                                                │
│  ✅ Secret không bao giờ xuất hiện trong code            │
│  ✅ Thay đổi config không cần redeploy                   │
│  ✅ Audit trail đầy đủ qua CloudTrail                    │
│  ✅ IAM policy kiểm soát ai đọc/ghi được gì              │
└──────────────────────────────────────────────────────────┘
```

---

## Loại Parameter

### 3 Kiểu Dữ Liệu Parameter

| Kiểu | Mô Tả | Khi Nào Dùng |
|------|--------|--------------|
| **String** | Chuỗi văn bản thông thường | Config values, URLs, non-sensitive data |
| **StringList** | Danh sách chuỗi, phân cách bằng dấu phẩy | Danh sách IP, AMI IDs, subnet IDs |
| **SecureString** | Chuỗi được mã hóa KMS | Passwords, API keys, connection strings |

### Ví Dụ Tạo Từng Loại

```bash
# String — Lưu database host
aws ssm put-parameter \
  --name "/prod/myapp/db/host" \
  --value "prod-cluster.abc123.ap-southeast-1.rds.amazonaws.com" \
  --type String \
  --description "Production RDS cluster endpoint"

# StringList — Lưu danh sách allowed IPs
aws ssm put-parameter \
  --name "/prod/myapp/allowed-ips" \
  --value "10.0.1.0/24,10.0.2.0/24,192.168.1.100/32" \
  --type StringList

# SecureString — Lưu database password (mã hóa bằng KMS)
aws ssm put-parameter \
  --name "/prod/myapp/db/password" \
  --value "MySecretP@ssw0rd" \
  --type SecureString \
  --key-id "arn:aws:kms:ap-southeast-1:123456789:key/abcd-1234" \
  --description "Production RDS master password"
```

---

## Standard vs Advanced Tier

### So Sánh Chi Tiết

| Thuộc Tính | Standard Tier | Advanced Tier |
|-----------|---------------|---------------|
| **Chi phí** | Miễn phí | $0.05/parameter/tháng |
| **Số lượng parameters** | 10,000/account/region | 100,000/account/region |
| **Kích thước giá trị** | Tối đa 4KB | Tối đa 8KB |
| **Parameter policies** | ❌ Không | ✅ Có (TTL, notification) |
| **Thông lượng (Throughput)** | 40 requests/giây | 1,000 requests/giây |
| **Tags** | ✅ Có | ✅ Có |
| **SecureString** | ✅ Có | ✅ Có |

### Nâng Cấp Lên Advanced

```bash
# Nâng cấp parameter hiện tại lên Advanced
aws ssm put-parameter \
  --name "/prod/myapp/db/password" \
  --value "MySecretP@ssw0rd" \
  --type SecureString \
  --tier Advanced \
  --overwrite
```

> **Lưu ý:** Không thể hạ cấp từ Advanced về Standard nếu đã có parameter policies hoặc kích thước > 4KB.

### Parameter Policies (Chỉ Advanced)

```bash
# Tự động xóa parameter sau 30 ngày (TTL — Time To Live)
aws ssm put-parameter \
  --name "/temp/deployment/feature-flag" \
  --value "true" \
  --type String \
  --tier Advanced \
  --policies '[
    {
      "Type": "Expiration",
      "Version": "1.0",
      "Attributes": {
        "Timestamp": "2026-06-17T00:00:00.000Z"
      }
    },
    {
      "Type": "ExpirationNotification",
      "Version": "1.0",
      "Attributes": {
        "Before": "5",
        "Unit": "Days"
      }
    }
  ]'
```

---

## SecureString & Mã Hóa KMS

### Cách SecureString Hoạt Động

```
Luồng mã hóa khi lưu SecureString:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  1. Bạn gửi: PUT /prod/db/password = "MySecret"        │
│                                                         │
│  2. SSM gọi KMS để mã hóa:                             │
│     KMS.Encrypt(plaintext="MySecret", KeyId=...)       │
│     → Trả về ciphertext (bản mã)                        │
│                                                         │
│  3. SSM lưu ciphertext vào Parameter Store             │
│     (plaintext KHÔNG BAO GIỜ được lưu)                 │
│                                                         │
│  4. Khi đọc:                                            │
│     SSM gọi KMS.Decrypt(ciphertext)                    │
│     → Trả về plaintext chỉ khi IAM cho phép           │
└─────────────────────────────────────────────────────────┘
```

### Hai Loại KMS Key

| Loại | Mô Tả | Dùng Khi |
|------|--------|---------|
| **AWS Managed Key** (`aws/ssm`) | AWS tự quản lý, không tính phí KMS | Cùng account, không cần cross-account |
| **Customer Managed Key (CMK)** | Bạn tự tạo và quản lý | Cross-account access, rotate key thủ công, fine-grained control |

```bash
# Dùng AWS Managed Key (mặc định)
aws ssm put-parameter \
  --name "/prod/secret" \
  --value "value" \
  --type SecureString
  # Không cần --key-id → dùng aws/ssm key

# Dùng Customer Managed Key
aws ssm put-parameter \
  --name "/prod/secret" \
  --value "value" \
  --type SecureString \
  --key-id "arn:aws:kms:ap-southeast-1:123456789:key/abcd-1234"
```

### Đọc SecureString

```bash
# Đọc với giải mã (decryption)
aws ssm get-parameter \
  --name "/prod/myapp/db/password" \
  --with-decryption \
  --query 'Parameter.Value' \
  --output text

# Đọc KHÔNG giải mã (chỉ thấy ciphertext)
aws ssm get-parameter \
  --name "/prod/myapp/db/password" \
  --query 'Parameter.Value' \
  --output text
# Output: AQICAHh+Aq... (ciphertext, vô nghĩa)
```

---

## Phân Cấp & Đặt Tên

### Cấu Trúc Phân Cấp (Hierarchy)

```
Cấu trúc khuyến nghị:
/[environment]/[application]/[component]/[parameter-name]

Ví dụ thực tế:
/prod/
  ├── myapp/
  │   ├── db/
  │   │   ├── host           → "prod-db.abc.rds.amazonaws.com"
  │   │   ├── port           → "5432"
  │   │   ├── name           → "myapp_prod"
  │   │   ├── username       → "app_user"
  │   │   └── password       → [SecureString] "***"
  │   ├── redis/
  │   │   ├── host           → "prod-redis.abc.cache.amazonaws.com"
  │   │   └── port           → "6379"
  │   ├── api/
  │   │   ├── stripe-key     → [SecureString] "sk_live_***"
  │   │   └── sendgrid-key   → [SecureString] "SG.***"
  │   └── feature-flags/
  │       ├── dark-mode      → "true"
  │       └── new-checkout   → "false"
  └── shared/
      ├── datadog-api-key    → [SecureString] "***"
      └── slack-webhook      → [SecureString] "https://hooks.slack.com/..."
```

### Lấy Toàn Bộ Parameters Theo Path

```bash
# Lấy tất cả parameters của /prod/myapp/ cùng lúc
aws ssm get-parameters-by-path \
  --path "/prod/myapp/" \
  --recursive \
  --with-decryption \
  --query 'Parameters[*].[Name,Value]' \
  --output table

# Chỉ lấy config DB (không recursive)
aws ssm get-parameters-by-path \
  --path "/prod/myapp/db/" \
  --with-decryption
```

---

## Versioning & Labels

### Versioning Tự Động

Mỗi khi update parameter, SSM tự động tăng version number. Mặc định giữ 100 versions.

```bash
# Cập nhật parameter (tạo version mới)
aws ssm put-parameter \
  --name "/prod/myapp/db/host" \
  --value "prod-db-new.abc.rds.amazonaws.com" \
  --type String \
  --overwrite
# Version 1 → Version 2

# Xem lịch sử version
aws ssm get-parameter-history \
  --name "/prod/myapp/db/host" \
  --query 'Parameters[*].[Version,Value,LastModifiedDate,LastModifiedUser]' \
  --output table

# Đọc version cụ thể
aws ssm get-parameter \
  --name "/prod/myapp/db/host:1" \
  --query 'Parameter.Value'
```

### Labels — Nhãn Cho Version

**Labels** (Nhãn) cho phép đặt tên có nghĩa cho version cụ thể, ví dụ: `stable`, `candidate`, `rollback`.

```bash
# Gán label "stable" cho version 2
aws ssm label-parameter-version \
  --name "/prod/myapp/db/host" \
  --parameter-version 2 \
  --labels "stable"

# Gán label "candidate" cho version 3 (đang test)
aws ssm label-parameter-version \
  --name "/prod/myapp/db/host" \
  --parameter-version 3 \
  --labels "candidate"

# Đọc theo label thay vì version number
aws ssm get-parameter \
  --name "/prod/myapp/db/host:stable" \
  --query 'Parameter.Value'
```

### Chiến Lược Rollback Với Labels

```bash
# Deployment workflow:
# 1. Deploy candidate → test
aws ssm get-parameter --name "/prod/myapp/db/host:candidate"

# 2. Nếu OK: promote candidate → stable
aws ssm label-parameter-version \
  --name "/prod/myapp/db/host" \
  --parameter-version $(aws ssm get-parameter --name "/prod/myapp/db/host:candidate" --query 'Parameter.Version' --output text) \
  --labels "stable"

# 3. Nếu lỗi: đọc stable để rollback
aws ssm get-parameter --name "/prod/myapp/db/host:stable"
```

---

## Tích Hợp Với AWS Services

### Lambda

```python
import boto3
import os

ssm = boto3.client('ssm', region_name='ap-southeast-1')

def get_db_config():
    params = ssm.get_parameters_by_path(
        Path='/prod/myapp/db/',
        WithDecryption=True
    )
    config = {}
    for param in params['Parameters']:
        key = param['Name'].split('/')[-1]
        config[key] = param['Value']
    return config

def lambda_handler(event, context):
    db = get_db_config()
    # db['host'], db['port'], db['password']...
```

### ECS Task Definition

```json
{
  "family": "myapp",
  "containerDefinitions": [
    {
      "name": "myapp",
      "image": "myapp:latest",
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:123456789:parameter/prod/myapp/db/password"
        },
        {
          "name": "STRIPE_KEY",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:123456789:parameter/prod/myapp/api/stripe-key"
        }
      ],
      "environment": [
        {
          "name": "ENVIRONMENT",
          "value": "production"
        }
      ]
    }
  ]
}
```

### CloudFormation Dynamic References

```yaml
# Dùng trực tiếp trong CloudFormation template
# Không cần custom resource hay Lambda
Resources:
  MyRDSInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      MasterUsername: "{{resolve:ssm:/prod/myapp/db/username}}"
      MasterUserPassword: "{{resolve:ssm-secure:/prod/myapp/db/password}}"
      DBInstanceClass: db.t3.medium

  MyLambda:
    Type: AWS::Lambda::Function
    Properties:
      Environment:
        Variables:
          DB_HOST: "{{resolve:ssm:/prod/myapp/db/host}}"
          API_VERSION: "{{resolve:ssm:/prod/myapp/api/version:3}}"
```

### AWS CodeDeploy / CodePipeline

```yaml
# buildspec.yml — CodeBuild lấy secrets từ SSM
version: 0.2
env:
  parameter-store:
    DB_PASSWORD: /prod/myapp/db/password
    STRIPE_KEY: /prod/myapp/api/stripe-key
    DB_HOST: /prod/myapp/db/host

phases:
  build:
    commands:
      - echo "DB_HOST=$DB_HOST" >> .env
      - docker build -t myapp .
```

---

## Parameter Store vs Secrets Manager

### Bảng So Sánh Chi Tiết

| Tiêu Chí | SSM Parameter Store | AWS Secrets Manager |
|----------|---------------------|---------------------|
| **Chi phí** | Standard: Miễn phí; Advanced: $0.05/param/tháng | $0.40/secret/tháng + $0.05/10K API calls |
| **Auto rotation** | ❌ Không tích hợp sẵn | ✅ Tích hợp RDS, Redshift, DocumentDB |
| **Cross-account** | Khó — cần share CMK | ✅ Native resource-based policy |
| **Versioning** | ✅ Tự động, giữ 100 versions | ✅ Tự động, không giới hạn |
| **Labels/Stages** | ✅ Labels | ✅ Staging labels (AWSCURRENT, AWSPENDING, AWSPREVIOUS) |
| **Kích thước** | Standard: 4KB; Advanced: 8KB | 64KB |
| **Phân cấp** | ✅ Path-based | ❌ Flat namespace |
| **Tích hợp CloudFormation** | `{{resolve:ssm:...}}` và `{{resolve:ssm-secure:...}}` | `{{resolve:secretsmanager:...}}` |
| **Config values** | ✅ Phù hợp | ⚠️ Overkill và tốn tiền |

### Khi Nào Chọn Cái Nào

```
Chọn SSM Parameter Store khi:
✅ Config values không nhạy cảm (URLs, feature flags, version numbers)
✅ Secrets đơn giản không cần auto-rotation (API keys tĩnh)
✅ Muốn tổ chức theo phân cấp /environment/app/component/
✅ Tích hợp với ECS, Lambda, CloudFormation tiện lợi
✅ Cần tiết kiệm chi phí

Chọn AWS Secrets Manager khi:
✅ Database passwords cần rotate tự động
✅ Cần cross-account secret sharing
✅ Tích hợp sẵn với RDS, Redshift, Elasticache
✅ Cần secret lớn hơn 8KB
✅ Compliance yêu cầu automatic rotation
```

---

## Câu Hỏi Phỏng Vấn

### Cơ Bản

**Q: Parameter Store Standard và Advanced khác nhau như thế nào?**

> Standard: Miễn phí, tối đa 4KB, 10,000 parameters, không có parameter policies. Advanced: Có phí ($0.05/parameter/tháng), tối đa 8KB, 100,000 parameters, hỗ trợ parameter policies (TTL, expiration notifications). Hầu hết trường hợp Standard đủ dùng; chuyển Advanced khi cần số lượng lớn hoặc parameter policies.

**Q: SecureString hoạt động thế nào? Ai có thể đọc giá trị?**

> SecureString dùng AWS KMS để mã hóa giá trị trước khi lưu. Giá trị plaintext không bao giờ được lưu. Khi đọc với `--with-decryption`, SSM gọi KMS để giải mã — chỉ thành công nếu IAM principal có quyền `kms:Decrypt` trên KMS key đó. Nếu không có quyền KMS, sẽ nhận được ciphertext vô nghĩa.

**Q: Tại sao nên dùng phân cấp `/env/app/component/name` thay vì tên phẳng?**

> Phân cấp cho phép: (1) Lấy toàn bộ config một ứng dụng bằng một API call `get-parameters-by-path`; (2) IAM policy dựa trên path — ví dụ cho phép Lambda chỉ đọc `/prod/myapp/*`; (3) Quản lý nhiều môi trường dễ dàng — chỉ thay `/dev/` thành `/prod/`; (4) CloudFormation dynamic references gọn hơn.

### Nâng Cao

**Q: Thiết kế giải pháp quản lý config/secret cho microservices trên ECS — 20 services, 3 môi trường?**

> **(1) Cấu trúc phân cấp:** `/[env]/[service]/[category]/[name]` — ví dụ `/prod/order-service/db/password`.
>
> **(2) IAM Policy per service:** Mỗi ECS Task Role chỉ được đọc path của service đó: `Resource: "arn:aws:ssm:*:*:parameter/prod/order-service/*"`. Không service nào đọc được secret của service khác.
>
> **(3) Phân biệt loại:** String cho config thông thường, SecureString (CMK riêng) cho credentials. Dùng CMK per-environment để secrets của prod/staging/dev được mã hóa bằng key khác nhau.
>
> **(4) Config injection:** Khai báo secrets trong ECS Task Definition `secrets` section — ECS tự inject vào environment variables khi container start. Không cần code xử lý SSM SDK.
>
> **(5) Audit:** CloudTrail ghi mọi `GetParameter` call — biết service nào đọc secret nào, lúc nào. CloudWatch alarm khi có GetParameter từ IP hoặc role lạ.
>
> **(6) Rotation:** Secrets cần rotate dùng Secrets Manager thay Parameter Store. Hoặc dùng Lambda scheduled function để rotate và update SSM parameter.

---

**Liên Quan:** [Session Manager](./1-session-manager.md) | [Run Command & Automation](./4-run-command-automation.md) | [Inventory & Compliance](./5-inventory-compliance.md)
