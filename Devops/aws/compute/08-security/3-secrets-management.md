# Secrets Management — Quản Lý Bí Mật Trên AWS

> Secrets Management (Quản Lý Bí Mật) là quy trình lưu trữ, truy cập và xoay vòng thông tin nhạy cảm như database passwords (mật khẩu cơ sở dữ liệu), API keys (khóa API), và certificates (chứng chỉ) một cách an toàn. AWS cung cấp hai dịch vụ chính: Secrets Manager (Trình Quản Lý Bí Mật) và Parameter Store (Kho Tham Số).

---

## 📚 Mục Lục

1. [Vấn Đề Cần Giải Quyết](#vấn-đề-cần-giải-quyết)
2. [AWS Secrets Manager](#aws-secrets-manager)
3. [SSM Parameter Store](#ssm-parameter-store)
4. [Secrets Manager vs Parameter Store](#secrets-manager-vs-parameter-store)
5. [Automatic Rotation — Xoay Vòng Tự Động](#automatic-rotation--xoay-vòng-tự-động)
6. [Best Practices Thực Tế](#best-practices-thực-tế)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Vấn Đề Cần Giải Quyết

### Secrets Thường Bị Lộ Như Thế Nào

```
Cách Phổ Biến Nhất Để Lộ Secrets:

1. Hard-coded trong source code → commit lên Git (public repo!)
   DB_PASS = "SuperSecret123"  ← SAI

2. Lưu trong environment variables dạng plaintext
   → Log files, process listing đều có thể thấy

3. Chia sẻ qua Slack, email, chat
   → Không kiểm soát được ai có access

4. Không rotate (xoay vòng) → nếu bị lộ, attacker có access mãi mãi

5. Cùng secret dùng cho nhiều môi trường
   → Dev environment bị compromise → Production bị ảnh hưởng
```

### Yêu Cầu Của Một Secrets Management Tốt

```
✓ Centralized storage — lưu tập trung, không phân tán
✓ Access control — chỉ ai cần mới được đọc
✓ Audit trail — biết ai đã đọc secret, khi nào
✓ Automatic rotation — tự động đổi mật khẩu định kỳ
✓ Encryption at rest — mã hóa khi lưu trữ
✓ Encryption in transit — mã hóa khi truyền
✓ Version history — rollback khi có vấn đề
✓ Easy integration — dễ tích hợp vào applications
```

---

## AWS Secrets Manager

### Khái Niệm Cốt Lõi

```
Secrets Manager (Trình Quản Lý Bí Mật):
├── Secret (Bí Mật) — đơn vị lưu trữ, chứa JSON hoặc string
│   ├── Tên: /production/myapp/database
│   ├── Value: {"username": "admin", "password": "..."}
│   ├── KMS key để encrypt/decrypt
│   └── Rotation configuration (cấu hình xoay vòng)
│
├── Rotation (Xoay Vòng) — tự động thay đổi secret định kỳ
│   ├── Lambda function thực hiện rotation
│   ├── Built-in rotation cho RDS, Redshift, DocumentDB
│   └── Custom rotation cho other databases
│
└── Resource Policy — ai được phép access secret
```

### Tạo Và Quản Lý Secrets

```bash
# Tạo secret đơn giản
aws secretsmanager create-secret \
  --name "/production/myapp/api-key" \
  --description "Third-party API key for payment service" \
  --secret-string "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

# Tạo secret dạng JSON (phổ biến cho database credentials)
aws secretsmanager create-secret \
  --name "/production/myapp/database" \
  --description "RDS PostgreSQL credentials" \
  --secret-string '{
    "username": "appuser",
    "password": "InitialPassword123!",
    "host": "mydb.cluster.ap-southeast-1.rds.amazonaws.com",
    "port": 5432,
    "dbname": "production_db"
  }' \
  --kms-key-id "arn:aws:kms:ap-southeast-1:123456789012:key/..."

# Đọc secret
aws secretsmanager get-secret-value \
  --secret-id "/production/myapp/database"

# Cập nhật secret value
aws secretsmanager update-secret \
  --secret-id "/production/myapp/database" \
  --secret-string '{"username": "appuser", "password": "NewPassword456!"}'

# Xóa secret (có 30 ngày recovery window — cửa sổ khôi phục)
aws secretsmanager delete-secret \
  --secret-id "/production/myapp/api-key" \
  --recovery-window-in-days 30

# Xóa ngay lập tức (không thể khôi phục!)
aws secretsmanager delete-secret \
  --secret-id "/production/myapp/api-key" \
  --force-delete-without-recovery
```

### Tích Hợp Vào Application

```python
# Python — dùng boto3 SDK
import boto3
import json

def get_db_credentials():
    client = boto3.client('secretsmanager', region_name='ap-southeast-1')
    response = client.get_secret_value(SecretId='/production/myapp/database')
    secret = json.loads(response['SecretString'])
    return {
        'host': secret['host'],
        'user': secret['username'],
        'password': secret['password'],
        'database': secret['dbname']
    }

# Best practice: cache credentials để giảm API calls
import functools

@functools.lru_cache(maxsize=1)
def get_cached_credentials():
    return get_db_credentials()
    # Cache trong memory — tốt cho Lambda warm invocations
    # Nhưng nhớ invalidate khi rotation xảy ra!
```

```javascript
// Node.js — AWS SDK v3
import { SecretsManagerClient, GetSecretValueCommand } from "@aws-sdk/client-secrets-manager";

const client = new SecretsManagerClient({ region: "ap-southeast-1" });

async function getDbCredentials() {
  const command = new GetSecretValueCommand({
    SecretId: "/production/myapp/database",
  });
  const response = await client.send(command);
  return JSON.parse(response.SecretString);
}
```

```go
// Go — AWS SDK v2
import (
    "github.com/aws/aws-sdk-go-v2/service/secretsmanager"
)

func getSecret(ctx context.Context, secretName string) (string, error) {
    client := secretsmanager.NewFromConfig(cfg)
    result, err := client.GetSecretValue(ctx, &secretsmanager.GetSecretValueInput{
        SecretId: &secretName,
    })
    if err != nil {
        return "", err
    }
    return *result.SecretString, nil
}
```

### Tích Hợp Native Với AWS Services

```yaml
# ECS Task Definition — lấy secrets từ Secrets Manager
{
  "containerDefinitions": [{
    "name": "myapp",
    "image": "myapp:latest",
    "secrets": [
      {
        "name": "DB_PASSWORD",
        "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:/production/myapp/database:password::"
      },
      {
        "name": "API_KEY",
        "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:/production/myapp/api-key::"
      }
    ]
  }]
}
# Secret tự động được inject vào container environment variable
# ECS agent lấy secrets khi task start — không cần code thay đổi!
```

```yaml
# Lambda — lấy secrets từ Secrets Manager qua environment variable reference
# (Cần code application tự fetch — Lambda không inject tự động như ECS)

# Hoặc dùng Lambda Extension để cache secrets
# AWS Parameters and Secrets Lambda Extension
# → Giảm latency và số API calls
```

```yaml
# Kubernetes (EKS) — External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-db-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: myapp-db-secret  # Tên Kubernetes Secret được tạo ra
  data:
    - secretKey: password
      remoteRef:
        key: /production/myapp/database
        property: password
```

### Resource-Based Policy — Chính Sách Dựa Trên Tài Nguyên

```json
// Cho phép nhiều accounts truy cập cùng một secret (cross-account)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowProductionEC2",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/ProductionEC2Role"
      },
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "secretsmanager:VersionStage": "AWSCURRENT"
        }
      }
    }
  ]
}
```

---

## SSM Parameter Store

### Các Loại Parameter

```
Parameter Store (Kho Tham Số):

String (Chuỗi):
└── Lưu text bình thường, không encrypt
    Dùng cho: URLs, configuration, feature flags
    Ví dụ: DB_HOST, API_ENDPOINT, MAX_CONNECTIONS

StringList (Danh Sách Chuỗi):
└── Danh sách values phân cách bằng dấu phẩy
    Dùng cho: allowed IPs, feature list
    Ví dụ: "10.0.0.1,10.0.0.2,10.0.0.3"

SecureString (Chuỗi Bảo Mật):
└── Encrypted bằng KMS key
    Dùng cho: passwords, API keys, tokens
    Ví dụ: DB_PASSWORD, STRIPE_SECRET_KEY
```

### Phân Cấp Tham Số — Parameter Hierarchy

```
/company/
├── /production/
│   ├── /myapp/
│   │   ├── database/
│   │   │   ├── host       = "prod-db.cluster.rds.amazonaws.com"
│   │   │   ├── port       = "5432"
│   │   │   ├── name       = "production_db"
│   │   │   └── password   = "encrypted-value" (SecureString)
│   │   └── api/
│   │       ├── stripe-key = "sk_live_..." (SecureString)
│   │       └── endpoint   = "https://api.stripe.com"
│   └── /otherapp/
│       └── ...
└── /staging/
    └── /myapp/
        └── database/
            └── host       = "staging-db.cluster.rds.amazonaws.com"
```

```bash
# Thao tác với Parameter Store

# Tạo parameter thường
aws ssm put-parameter \
  --name "/production/myapp/database/host" \
  --value "prod-db.cluster.ap-southeast-1.rds.amazonaws.com" \
  --type "String" \
  --tier "Standard"

# Tạo secure parameter (encrypted)
aws ssm put-parameter \
  --name "/production/myapp/database/password" \
  --value "MySuperSecretPassword!" \
  --type "SecureString" \
  --key-id "arn:aws:kms:ap-southeast-1:123456789012:key/my-key-id"

# Cập nhật (tự động tăng version)
aws ssm put-parameter \
  --name "/production/myapp/database/password" \
  --value "NewPassword456!" \
  --type "SecureString" \
  --overwrite

# Đọc parameter
aws ssm get-parameter \
  --name "/production/myapp/database/host" \
  --query "Parameter.Value" \
  --output text

# Đọc SecureString (cần --with-decryption)
aws ssm get-parameter \
  --name "/production/myapp/database/password" \
  --with-decryption \
  --query "Parameter.Value" \
  --output text

# Đọc nhiều parameters theo path
aws ssm get-parameters-by-path \
  --path "/production/myapp/database/" \
  --recursive \
  --with-decryption

# Đọc version cụ thể
aws ssm get-parameter \
  --name "/production/myapp/database/password:3"  # version 3
```

### Tier Standard vs Advanced

```
Standard Tier (Bậc Chuẩn):
├── Free (miễn phí)
├── Max 10,000 parameters per account
├── Max 4KB per parameter
└── Không có parameter policies

Advanced Tier (Bậc Nâng Cao):
├── $0.05 per advanced parameter per month
├── Max 100,000 parameters per account
├── Max 8KB per parameter
└── Parameter Policies:
    ├── Expiration — tự động xóa parameter sau ngày hết hạn
    ├── ExpirationNotification — thông báo trước khi hết hạn
    └── NoChangeNotification — cảnh báo nếu không được cập nhật
```

---

## Secrets Manager vs Parameter Store

### So Sánh Chi Tiết

| Tiêu Chí | Secrets Manager | Parameter Store (SecureString) |
|----------|----------------|-------------------------------|
| **Chi Phí** | $0.40/secret/tháng + $0.05/10K API calls | $0 (Standard) / $0.05/param/tháng (Advanced) |
| **Automatic Rotation** | ✅ Built-in | ❌ Phải tự implement |
| **Max Size** | 65KB | 4KB (Standard) / 8KB (Advanced) |
| **Cross-account** | ✅ Resource policy | ❌ Phức tạp hơn |
| **RDS Integration** | ✅ Native rotation | ❌ Thủ công |
| **Versioning** | ✅ Staging labels | ✅ Version numbers |
| **Audit** | ✅ CloudTrail | ✅ CloudTrail |
| **KMS Encryption** | ✅ (bắt buộc) | ✅ (tùy chọn) |

### Khi Nào Dùng Cái Nào?

```
Dùng Secrets Manager khi:
✓ Cần automatic rotation (database passwords, API keys)
✓ Quản lý RDS, Redshift, DocumentDB credentials
✓ Cần cross-account secret sharing
✓ Compliance yêu cầu secrets phải được rotate
✓ Sẵn sàng trả thêm chi phí cho tính năng nâng cao

Dùng Parameter Store khi:
✓ Lưu configuration data (không nhạy cảm)
✓ Cần tiết kiệm chi phí (nhiều parameters)
✓ Tích hợp với SSM ecosystem
✓ Không cần automatic rotation
✓ Environment variables, feature flags, URLs
```

---

## Automatic Rotation — Xoay Vòng Tự Động

### Tại Sao Rotation Quan Trọng?

```
Nếu secret bị lộ:
   Không có rotation → attacker có access mãi mãi
   Có rotation → access chỉ trong khoảng thời gian ngắn

Compliance yêu cầu:
   PCI DSS — thẻ tín dụng: rotate mỗi 90 ngày
   SOC 2 — service organization: rotate hàng năm
   HIPAA — y tế: rotation policies bắt buộc
```

### Built-in Rotation Cho RDS

```bash
# Bật rotation tự động cho RDS secret
aws secretsmanager rotate-secret \
  --secret-id "/production/myapp/database" \
  --rotation-lambda-arn "arn:aws:lambda:ap-southeast-1:123456789012:function:SecretsManagerRotation" \
  --rotation-rules '{"AutomaticallyAfterDays": 30}'

# AWS cung cấp sẵn Lambda functions cho rotation:
# - SecretsManagerRDSMySQLRotationSingleUser
# - SecretsManagerRDSPostgreSQLRotationSingleUser
# - SecretsManagerRDSMariaDBRotationSingleUser
# → Deploy từ AWS Serverless Application Repository

# Rotation process (4 bước):
# 1. createSecret — tạo phiên bản mới với password mới
# 2. setSecret — cập nhật password trong database
# 3. testSecret — test kết nối với credentials mới
# 4. finishSecret — đánh dấu phiên bản mới là AWSCURRENT
```

### Versioning và Staging Labels

```
Secrets Manager Version Stages (Giai Đoạn Phiên Bản):

AWSCURRENT   → secret đang được dùng hiện tại
AWSPENDING   → secret mới đang được tạo (trong quá trình rotation)
AWSPREVIOUS  → secret vừa được thay thế (giữ 1 version cũ để rollback)

Timeline của rotation:
Before:  [v1: AWSCURRENT]
Step 1:  [v1: AWSCURRENT] [v2: AWSPENDING]
Step 4:  [v1: AWSPREVIOUS] [v2: AWSCURRENT]
Next:    [v2: AWSCURRENT] (v1 bị xóa sau 1 ngày)
```

```bash
# Đọc version hiện tại (mặc định)
aws secretsmanager get-secret-value \
  --secret-id "/production/myapp/database"

# Đọc version trước (để rollback)
aws secretsmanager get-secret-value \
  --secret-id "/production/myapp/database" \
  --version-stage "AWSPREVIOUS"

# Xem tất cả versions
aws secretsmanager list-secret-version-ids \
  --secret-id "/production/myapp/database"
```

### Custom Rotation Lambda

```python
# Lambda function thực hiện rotation cho custom database/service
import boto3
import json

def lambda_handler(event, context):
    arn = event['SecretId']
    token = event['ClientRequestToken']
    step = event['Step']

    client = boto3.client('secretsmanager')

    if step == "createSecret":
        create_secret(client, arn, token)
    elif step == "setSecret":
        set_secret(client, arn, token)
    elif step == "testSecret":
        test_secret(client, arn, token)
    elif step == "finishSecret":
        finish_secret(client, arn, token)

def create_secret(client, arn, token):
    # Tạo password mới ngẫu nhiên
    passwd = client.get_random_password(
        PasswordLength=32,
        ExcludeCharacters='"@/\\'
    )['RandomPassword']

    # Lấy current secret để biết username, host...
    current = json.loads(
        client.get_secret_value(SecretId=arn, VersionStage='AWSCURRENT')['SecretString']
    )

    # Lưu pending secret
    current['password'] = passwd
    client.put_secret_value(
        SecretId=arn,
        ClientRequestToken=token,
        SecretString=json.dumps(current),
        VersionStages=['AWSPENDING']
    )

def set_secret(client, arn, token):
    # Đọc pending secret
    pending = json.loads(
        client.get_secret_value(SecretId=arn, VersionId=token, VersionStage='AWSPENDING')['SecretString']
    )
    # Cập nhật password trong database thực tế
    # db.execute(f"ALTER USER '{pending['username']}' PASSWORD '{pending['password']}'")

def test_secret(client, arn, token):
    # Test kết nối với pending credentials
    pending = json.loads(
        client.get_secret_value(SecretId=arn, VersionId=token, VersionStage='AWSPENDING')['SecretString']
    )
    # Thử kết nối database với credentials mới
    # conn = connect(host=pending['host'], user=pending['username'], password=pending['password'])

def finish_secret(client, arn, token):
    # Đánh dấu version mới là AWSCURRENT
    current = client.describe_secret(SecretId=arn)
    for version, stages in current['VersionIdsToStages'].items():
        if 'AWSCURRENT' in stages and version != token:
            client.update_secret_version_stage(
                SecretId=arn,
                VersionStage='AWSCURRENT',
                MoveToVersionId=token,
                RemoveFromVersionId=version
            )
            break
```

---

## Best Practices Thực Tế

### 1. Cấu Trúc Naming Convention — Quy Ước Đặt Tên

```
/[environment]/[application]/[component]/[key]

Ví dụ tốt:
/production/payment-service/database/credentials
/staging/user-service/redis/auth-token
/shared/infrastructure/newrelic/api-key

Lợi ích:
→ IAM policy dễ viết theo path prefix
→ Dễ tìm kiếm và quản lý
→ Tách biệt rõ ràng giữa environments
```

### 2. IAM Policy Theo Path Prefix

```json
// EC2 chỉ được đọc secrets của application mình
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:/production/payment-service/*"
    },
    {
      "Effect": "Allow",
      "Action": "kms:Decrypt",
      "Resource": "arn:aws:kms:ap-southeast-1:123456789012:key/payment-service-key"
    }
  ]
}
```

### 3. Caching Secrets Trong Application

```python
# Tránh gọi Secrets Manager mỗi request — tốn phí và latency
import boto3
import json
import time

class SecretsCache:
    def __init__(self, ttl_seconds=300):  # Cache 5 phút
        self.cache = {}
        self.ttl = ttl_seconds
        self.client = boto3.client('secretsmanager')

    def get_secret(self, secret_id):
        now = time.time()
        if secret_id in self.cache:
            value, expires = self.cache[secret_id]
            if now < expires:
                return value  # Return from cache

        # Fetch from Secrets Manager
        response = self.client.get_secret_value(SecretId=secret_id)
        value = json.loads(response['SecretString'])
        self.cache[secret_id] = (value, now + self.ttl)
        return value

# Tốt hơn: dùng AWS Parameters and Secrets Lambda Extension
# (có sẵn, cache tự động, không cần code thêm)
```

### 4. Audit Secrets Access

```bash
# CloudTrail query để xem ai đã access secrets gần đây
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetSecretValue \
  --start-time $(date -d '-7 days' --iso-8601=seconds)

# CloudWatch Logs Insights query
fields @timestamp, userIdentity.arn, requestParameters.secretId
| filter eventSource = "secretsmanager.amazonaws.com"
| filter eventName = "GetSecretValue"
| sort @timestamp desc
| limit 100
```

### 5. Secret Replication — Nhân Bản Bí Mật

```bash
# Replicate secret sang region khác (cho multi-region deployment)
aws secretsmanager replicate-secret-to-regions \
  --secret-id "/production/myapp/database" \
  --add-replica-regions '[
    {"Region": "us-east-1"},
    {"Region": "eu-west-1"}
  ]'
```

### 6. Xử Lý Rotation Trong Application

```python
# Application phải handle rotation gracefully (xử lý xoay vòng khéo léo)
import boto3
import psycopg2
import json

def get_db_connection(secret_id):
    client = boto3.client('secretsmanager')
    secret = json.loads(
        client.get_secret_value(SecretId=secret_id)['SecretString']
    )

    try:
        conn = psycopg2.connect(
            host=secret['host'],
            user=secret['username'],
            password=secret['password'],
            database=secret['dbname']
        )
        return conn
    except psycopg2.OperationalError:
        # Auth failed → có thể rotation vừa xảy ra
        # Fetch lại secret (version mới nhất)
        secret = json.loads(
            client.get_secret_value(
                SecretId=secret_id,
                VersionStage='AWSCURRENT'  # Explicitly request current
            )['SecretString']
        )
        return psycopg2.connect(
            host=secret['host'],
            user=secret['username'],
            password=secret['password'],
            database=secret['dbname']
        )
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Secrets Manager và Parameter Store khác nhau như thế nào? Khi nào dùng cái nào?

**Trả lời:** Secrets Manager được tối ưu hóa cho secrets cần rotation tự động — nó có built-in integration với RDS, Redshift, DocumentDB để tự động rotate passwords theo lịch. Chi phí $0.40/secret/tháng nhưng đáng giá khi cần compliance hoặc rotation. Parameter Store miễn phí (Standard tier) và phù hợp hơn cho configuration data như database URLs, feature flags, API endpoints không nhạy cảm. SecureString trong Parameter Store encrypt bằng KMS nhưng không có built-in rotation. Tôi thường dùng Secrets Manager cho database passwords và API keys production, Parameter Store cho configuration data và environments khác.

### Q2: Giải thích rotation process trong Secrets Manager.

**Trả lời:** Rotation xảy ra qua Lambda function theo 4 bước: `createSecret` — tạo phiên bản mới với password ngẫu nhiên và gán staging label `AWSPENDING`; `setSecret` — cập nhật password mới vào database hoặc service thực tế; `testSecret` — test kết nối với credentials mới để xác nhận hoạt động; `finishSecret` — chuyển staging label `AWSCURRENT` sang phiên bản mới, version cũ thành `AWSPREVIOUS`. AWS cung cấp sẵn Lambda function cho RDS MySQL, PostgreSQL, MariaDB. Application phải handle rotation gracefully bằng cách retry khi authentication fail vì rotation có thể xảy ra bất cứ lúc nào.

### Q3: Làm thế nào để tránh hard-coding secrets trong container applications?

**Trả lời:** Có nhiều cách: Đối với ECS, dùng `secrets` field trong Task Definition — ECS agent tự lấy secret từ Secrets Manager hoặc Parameter Store và inject vào container environment variable khi task start. Đối với EKS, dùng External Secrets Operator hoặc AWS Secrets Store CSI Driver (Container Storage Interface Driver — Trình Điều Khiển Lưu Trữ Container) để mount secrets như Kubernetes Secrets, tự động sync từ Secrets Manager. Đối với Lambda, dùng AWS Parameters and Secrets Lambda Extension để cache và inject secrets — giảm latency và API calls. Trong tất cả trường hợp, ứng dụng không bao giờ cần biết rotation xảy ra vì extension/agent lo việc refresh.

---

## 🔗 Điều Hướng

| | |
|--|--|
| ← SSM Systems Manager | [2-systems-manager.md](./2-systems-manager.md) |
| → Encryption & KMS | [4-encryption.md](./4-encryption.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15
