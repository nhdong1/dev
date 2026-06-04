# 4 — Secrets Management — Quản Lý Thông Tin Bí Mật

> Secrets Management (Quản Lý Thông Tin Bí Mật) giải quyết câu hỏi: "Database password phải lưu ở đâu?" Câu trả lời: không phải trong code, không phải trong environment variables không bảo mật, không phải trong config files — mà trong **AWS Secrets Manager** hoặc **Parameter Store**, với automatic rotation và audit trail đầy đủ.

---

## 📚 Mục Lục

1. [Vấn Đề Secrets Management](#1-vấn-đề-secrets-management)
2. [AWS Secrets Manager](#2-aws-secrets-manager)
3. [Automatic Rotation — Tự Động Xoay Vòng Mật Khẩu](#3-automatic-rotation--tự-động-xoay-vòng-mật-khẩu)
4. [AWS Parameter Store](#4-aws-parameter-store)
5. [Secrets Manager vs Parameter Store](#5-secrets-manager-vs-parameter-store)
6. [Integration Patterns — Mẫu Tích Hợp](#6-integration-patterns--mẫu-tích-hợp)
7. [Multi-Account & Cross-Region Secrets](#7-multi-account--cross-region-secrets)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Vấn Đề Secrets Management

### Anti-Patterns — Mẫu Sai

```python
# ❌ KHÔNG BAO GIỜ làm điều này

# Hardcoded trong source code
DB_PASSWORD = "MySecretPassword123!"
conn = connect(host="db.example.com", password=DB_PASSWORD)
# → Bị commit lên Git, tồn tại mãi trong git history

# Environment variable không an toàn (Docker/Kubernetes)
# docker run -e DB_PASSWORD=MySecretPassword123! myapp
# → Visible trong docker inspect, process list, CI/CD logs

# .env file
# DB_PASSWORD=MySecretPassword123!  trong file .env
# → Dễ bị commit lên Git nếu không config .gitignore cẩn thận

# Config file trong source tree
# config/database.yml:
#   password: MySecretPassword123!
# → Ai clone repo là biết password
```

### Hậu Quả Khi Lộ Credentials

```
Incident timeline điển hình:
T+0:   Developer commit .env file lên GitHub public repo
T+15m: Bot tự động scan GitHub phát hiện credentials
T+16m: Attacker dùng credentials, connect vào database
T+1h:  Production data bị exfiltrate (sao chép ra ngoài)
T+4h:  Incident được phát hiện (nếu may mắn)
T+8h:  Database password được rotate
Damage: Customer PII exposed, regulatory fines, brand damage
```

### Giải Pháp — Secret Storage Hợp Lệ

```
Secrets Manager:
  Cho: Database credentials, API keys, OAuth tokens
  Lý do: Auto-rotation, native RDS integration, $0.40/secret

Parameter Store:
  Cho: App config, feature flags, non-sensitive strings
  Lý do: Rẻ hơn (có free tier), phù hợp cho config thông thường

HashiCorp Vault (third-party):
  Cho: Multi-cloud, on-prem hybrid
  Lý do: Cloud-agnostic, advanced features

→ Trong AWS-native stack: Secrets Manager + Parameter Store là đủ
```

---

## 2. AWS Secrets Manager

### Tạo Secret Cho Database

```bash
# Tạo secret cho RDS MySQL
aws secretsmanager create-secret \
  --name "prod/mysql/app-db" \
  --description "Production MySQL credentials for app-service" \
  --secret-string '{
    "username": "app_user",
    "password": "RandomStr0ng!Pass#2024",
    "host": "prod-db.cluster-xxx.ap-southeast-1.rds.amazonaws.com",
    "port": 3306,
    "dbname": "app_production",
    "engine": "mysql"
  }' \
  --kms-key-id arn:aws:kms:ap-southeast-1:123456789012:key/mrk-xxxx \
  --tags Key=Environment,Value=production Key=Service,Value=app-service

# Kết quả: ARN của secret
# arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:prod/mysql/app-db-AbCdEf
```

### Đọc Secret Trong Application

```python
import boto3
import json
from functools import lru_cache
from datetime import datetime, timedelta

class SecretsCache:
    """Cache secret để tránh gọi Secrets Manager quá nhiều lần"""

    def __init__(self, ttl_seconds=300):  # Cache 5 phút
        self._cache = {}
        self._ttl = ttl_seconds
        self._client = boto3.client('secretsmanager', region_name='ap-southeast-1')

    def get_secret(self, secret_name: str) -> dict:
        now = datetime.now()

        # Kiểm tra cache
        if secret_name in self._cache:
            value, expires_at = self._cache[secret_name]
            if now < expires_at:
                return value

        # Lấy từ Secrets Manager
        response = self._client.get_secret_value(SecretId=secret_name)
        secret = json.loads(response['SecretString'])

        # Lưu vào cache
        self._cache[secret_name] = (secret, now + timedelta(seconds=self._ttl))
        return secret

# Singleton instance
secrets_cache = SecretsCache(ttl_seconds=300)

def get_db_connection():
    secret = secrets_cache.get_secret('prod/mysql/app-db')

    import pymysql
    return pymysql.connect(
        host=secret['host'],
        port=secret['port'],
        user=secret['username'],
        password=secret['password'],
        database=secret['dbname'],
        ssl={'ca': '/etc/ssl/rds-ca/bundle.pem'},
        connect_timeout=5
    )
```

```javascript
// Node.js với AWS SDK v3
const { SecretsManagerClient, GetSecretValueCommand } = require("@aws-sdk/client-secrets-manager");

const client = new SecretsManagerClient({ region: "ap-southeast-1" });

// Cache secret với TTL
const secretCache = new Map();
const CACHE_TTL_MS = 5 * 60 * 1000; // 5 phút

async function getDbCredentials(secretName) {
  const cached = secretCache.get(secretName);
  if (cached && Date.now() < cached.expiresAt) {
    return cached.value;
  }

  const response = await client.send(new GetSecretValueCommand({ SecretId: secretName }));
  const secret = JSON.parse(response.SecretString);

  secretCache.set(secretName, {
    value: secret,
    expiresAt: Date.now() + CACHE_TTL_MS
  });

  return secret;
}
```

### Secret Versioning — Phiên Bản Secret

```
Secrets Manager duy trì nhiều versions của secret:

┌─────────────────────────────────────────────────────────┐
│ Secret: prod/mysql/app-db                               │
├─────────────────────────────────────────────────────────┤
│ Version 1 (cũ)    │ AWSPREVIOUS  │ password: "Old!Pass" │
│ Version 2 (hiện tại) │ AWSCURRENT  │ password: "New!Pass"│
│ Version 3 (đang tạo) │ AWSPENDING  │ password: "New2!Pass"│
└─────────────────────────────────────────────────────────┘

Stages (Giai Đoạn):
- AWSCURRENT:  Version đang active — dùng cho new connections
- AWSPREVIOUS: Version trước — dùng cho connections chưa refresh
- AWSPENDING:  Version đang trong quá trình rotation

→ Rotation không bao giờ break existing connections:
  1. Tạo AWSPENDING với password mới
  2. Test AWSPENDING có connect được không
  3. Promote AWSPENDING → AWSCURRENT
  4. Demote AWSCURRENT → AWSPREVIOUS
  5. Xóa AWSPREVIOUS sau vài phút (connections đã refresh)
```

---

## 3. Automatic Rotation — Tự Động Xoay Vòng Mật Khẩu

### Rotation Hoạt Động Thế Nào

```
Rotation Flow cho RDS:

1. CloudWatch Events kích hoạt Lambda Rotation Function
   (theo schedule: mỗi 30 ngày, hoặc on-demand)

2. Lambda thực hiện 4 bước (4-step rotation):

   Step 1: createSecret
   ─────────────────────
   • Tạo version mới với stage AWSPENDING
   • Generate random password mới (mạnh, đáp ứng policy)

   Step 2: setSecret
   ─────────────────
   • Kết nối vào RDS với AWSCURRENT credentials
   • Chạy: ALTER USER 'app_user'@'%' IDENTIFIED BY '<new_password>';
   • Password mới được set trong database

   Step 3: testSecret
   ──────────────────
   • Connect vào RDS bằng AWSPENDING credentials
   • Nếu thành công → tiếp tục
   • Nếu fail → rollback, alert, stop rotation

   Step 4: finishSecret
   ─────────────────────
   • Promote AWSPENDING → AWSCURRENT
   • Old AWSCURRENT → AWSPREVIOUS
   • Rotation hoàn thành

3. Existing connections: tự động refresh khi reconnect
   (connection pool detect disconnect → lấy AWSCURRENT → reconnect)
```

### Enable Rotation Cho RDS

```bash
# Enable automatic rotation với managed rotation (AWS Lambda có sẵn)
aws secretsmanager rotate-secret \
  --secret-id "prod/mysql/app-db" \
  --rotation-rules AutomaticallyAfterDays=30 \
  --rotate-immediately   # Rotate ngay lập tức (không chờ 30 ngày)

# AWS tự tạo Lambda function với managed rotation
# Supported engines: MySQL, PostgreSQL, Oracle, MSSQL, MariaDB, Aurora
```

### Custom Rotation Lambda

Cho credentials phức tạp hơn (multiple databases, custom logic):

```python
import boto3
import json
import string
import secrets

def lambda_handler(event, context):
    """Custom rotation Lambda cho database credentials"""
    secret_arn = event['SecretId']
    token = event['ClientRequestToken']
    step = event['Step']

    secrets_client = boto3.client('secretsmanager')

    if step == "createSecret":
        create_secret(secrets_client, secret_arn, token)
    elif step == "setSecret":
        set_secret(secrets_client, secret_arn, token)
    elif step == "testSecret":
        test_secret(secrets_client, secret_arn, token)
    elif step == "finishSecret":
        finish_secret(secrets_client, secret_arn, token)


def generate_strong_password(length=32) -> str:
    """Generate password đáp ứng MySQL complexity requirements"""
    alphabet = string.ascii_letters + string.digits + "!@#$%^&*()"
    while True:
        password = ''.join(secrets.choice(alphabet) for _ in range(length))
        # Kiểm tra có đủ các loại ký tự không
        if (any(c.islower() for c in password) and
            any(c.isupper() for c in password) and
            any(c.isdigit() for c in password) and
            any(c in "!@#$%^&*()" for c in password)):
            return password


def create_secret(client, arn, token):
    """Bước 1: Tạo version mới"""
    try:
        client.get_secret_value(SecretId=arn, VersionId=token, VersionStage='AWSPENDING')
        return  # Đã có AWSPENDING → bỏ qua (idempotent)
    except client.exceptions.ResourceNotFoundException:
        pass

    current = json.loads(client.get_secret_value(SecretId=arn, VersionStage='AWSCURRENT')['SecretString'])
    current['password'] = generate_strong_password()

    client.put_secret_value(
        SecretId=arn,
        ClientRequestToken=token,
        SecretString=json.dumps(current),
        VersionStages=['AWSPENDING']
    )
```

### Rotation Best Practices

```
✅ Rotate mỗi 30 ngày (balance giữa security và operational overhead)
✅ Test rotation trong staging trước khi enable ở production
✅ Monitor rotation failures trong CloudWatch (metric: RotationFailed)
✅ Set SNS alert khi rotation fail
✅ Connection pool phải có logic retry sau khi authentication fail:
   1. Nhận auth error từ DB
   2. Invalidate cached credentials
   3. Lấy AWSCURRENT secret từ Secrets Manager
   4. Retry connection
   5. Nếu thành công → continue; nếu fail lần 2 → raise exception

✅ Dual-stack support khi rotation đang chạy:
   App phải thử AWSCURRENT trước, nếu fail thử AWSPREVIOUS
   → Không có downtime trong rotation window
```

---

## 4. AWS Parameter Store

### Parameter Store Là Gì?

**Parameter Store** (Kho Tham Số) là dịch vụ lưu trữ configuration data và secrets của AWS Systems Manager. Đơn giản hơn, rẻ hơn Secrets Manager.

### Hai Tier — Hai Mức

| | Standard Tier | Advanced Tier |
|---|---|---|
| **Giá** | Miễn phí | $0.05/parameter/tháng |
| **Số lượng** | 10,000 parameters | 100,000 parameters |
| **Kích thước** | 4KB | 8KB |
| **TTL/Expiration** | ❌ | ✅ |
| **Policy** | ❌ | ✅ (thông báo trước expiry) |

### Các Loại Parameter

```bash
# String — chuỗi thường (không mã hóa)
aws ssm put-parameter \
  --name "/prod/app/db-host" \
  --value "prod-db.cluster-xxx.ap-southeast-1.rds.amazonaws.com" \
  --type String

# StringList — danh sách chuỗi
aws ssm put-parameter \
  --name "/prod/app/allowed-origins" \
  --value "https://app.example.com,https://admin.example.com" \
  --type StringList

# SecureString — mã hóa bằng KMS (dùng cho secrets)
aws ssm put-parameter \
  --name "/prod/app/db-password" \
  --value "MySecretP@ssword!" \
  --type SecureString \
  --key-id arn:aws:kms:...:key/mrk-xxxxx  # Nếu không chỉ định → dùng aws/ssm
```

### Parameter Hierarchy — Cấu Trúc Thư Mục Tham Số

```bash
# Naming convention theo hierarchy (phân cấp)
/environment/application/parameter-name

Ví dụ:
/prod/user-service/db-host
/prod/user-service/db-password
/prod/order-service/db-host
/staging/user-service/db-host

# Lấy tất cả parameters của một service
aws ssm get-parameters-by-path \
  --path "/prod/user-service/" \
  --with-decryption   # Decrypt SecureString

# Kết quả:
# [
#   {Name: "/prod/user-service/db-host", Value: "prod-db.xxx..."},
#   {Name: "/prod/user-service/db-password", Value: "MyPass!"}
# ]
```

### Python Integration

```python
import boto3

ssm = boto3.client('ssm', region_name='ap-southeast-1')

def get_parameter(name: str, decrypt: bool = True) -> str:
    response = ssm.get_parameter(Name=name, WithDecryption=decrypt)
    return response['Parameter']['Value']

def get_service_config(service: str, env: str = 'prod') -> dict:
    """Lấy tất cả config của một service"""
    path = f"/{env}/{service}/"
    params = {}

    paginator = ssm.get_paginator('get_parameters_by_path')
    for page in paginator.paginate(Path=path, WithDecryption=True):
        for param in page['Parameters']:
            key = param['Name'].replace(path, '')  # Bỏ prefix path
            params[key] = param['Value']

    return params

# Dùng:
config = get_service_config('user-service')
# config = {'db-host': 'prod-db.xxx...', 'db-password': 'MyPass!', ...}
```

---

## 5. Secrets Manager vs Parameter Store

### Bảng So Sánh Toàn Diện

| Tiêu Chí | Secrets Manager | Parameter Store |
|----------|----------------|-----------------|
| **Giá** | $0.40/secret/tháng + $0.05/10K API calls | Free (Standard); $0.05/Advanced param/tháng |
| **Automatic Rotation** | ✅ Native support (4-step Lambda) | ❌ Phải tự implement |
| **RDS Native Integration** | ✅ AWS quản lý rotation Lambda | ❌ |
| **Cross-Account Access** | ✅ Resource-based policy | ❌ (chỉ same account) |
| **Replication Multi-Region** | ✅ Automatic replica | ❌ |
| **Secret Versioning** | ✅ AWSCURRENT/AWSPREVIOUS/AWSPENDING | ✅ (không có stages) |
| **Max Size** | 65KB | 4KB (Standard) / 8KB (Advanced) |
| **Audit** | CloudTrail chi tiết | CloudTrail |
| **Expiration/TTL** | ✅ | Chỉ Advanced tier |
| **Tags** | ✅ | ✅ |

### Quyết Định Khi Nào Dùng Gì

```
Dùng Secrets Manager cho:
✅ Database credentials (password cần rotate)
✅ API keys cần rotation
✅ OAuth client secrets
✅ SSH private keys
✅ TLS private keys/certificates
✅ Bất kỳ credential nào cần auto-rotation
✅ Multi-account shared secrets

Dùng Parameter Store cho:
✅ Database host, port (không nhạy cảm)
✅ Feature flags, app configuration
✅ Environment-specific config (URLs, timeouts)
✅ License keys (không rotate)
✅ Non-sensitive API endpoints
✅ Khi budget là concern lớn (nhiều parameters, ít cần rotation)
```

### Hybrid Pattern — Mẫu Kết Hợp

```python
# Best practice: Dùng cả hai
# Secrets Manager: credentials (password, keys)
# Parameter Store: configuration (host, port, settings)

class DatabaseConfig:
    def __init__(self):
        self._ssm = boto3.client('ssm')
        self._sm = boto3.client('secretsmanager')
        self._cache = {}

    def get_connection_params(self, env='prod') -> dict:
        # Non-sensitive config từ Parameter Store (rẻ hơn)
        host = self._get_param(f'/{env}/db/host')
        port = int(self._get_param(f'/{env}/db/port'))
        dbname = self._get_param(f'/{env}/db/name')

        # Credentials từ Secrets Manager (có auto-rotation)
        secret = json.loads(
            self._sm.get_secret_value(SecretId=f'{env}/db/credentials')['SecretString']
        )

        return {
            'host': host,
            'port': port,
            'database': dbname,
            'user': secret['username'],
            'password': secret['password']
        }

    def _get_param(self, name: str) -> str:
        if name not in self._cache:
            self._cache[name] = self._ssm.get_parameter(
                Name=name, WithDecryption=True
            )['Parameter']['Value']
        return self._cache[name]
```

---

## 6. Integration Patterns — Mẫu Tích Hợp

### Kubernetes / ECS Secret Injection

```yaml
# Kubernetes: Dùng External Secrets Operator để sync từ Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
spec:
  refreshInterval: 5m        # Sync mỗi 5 phút
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-secret           # Tên Kubernetes Secret được tạo
    creationPolicy: Owner
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: prod/mysql/app-db
        property: password    # Extract field cụ thể từ JSON
    - secretKey: DB_USERNAME
      remoteRef:
        key: prod/mysql/app-db
        property: username
```

```json
// ECS Task Definition: Reference Secrets Manager secrets
{
  "family": "app-service",
  "containerDefinitions": [{
    "name": "app",
    "image": "123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/app:latest",
    "secrets": [
      {
        "name": "DB_PASSWORD",
        "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:prod/mysql/app-db:password::"
      },
      {
        "name": "DB_HOST",
        "valueFrom": "arn:aws:ssm:ap-southeast-1:123456789012:parameter/prod/db/host"
      }
    ]
  }]
}
```

### Lambda Environment Variables Với Secrets

```python
# Lambda: Không inject password vào env vars
# Thay vào đó, load từ Secrets Manager khi cold start
import boto3
import json
import os

_db_secret = None  # Module-level cache cho Lambda warm starts

def get_db_secret():
    global _db_secret
    if _db_secret is None:
        client = boto3.client('secretsmanager')
        response = client.get_secret_value(
            SecretId=os.environ['DB_SECRET_ARN']  # ARN trong env var (không phải password)
        )
        _db_secret = json.loads(response['SecretString'])
    return _db_secret

def lambda_handler(event, context):
    secret = get_db_secret()
    # Dùng secret['password'], secret['host'], etc.
```

### CDK / CloudFormation Secret Reference

```typescript
// AWS CDK (Cloud Development Kit): Auto-generate và reference secret
import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';
import * as rds from 'aws-cdk-lib/aws-rds';

// CDK tự generate random password
const dbSecret = new secretsmanager.Secret(this, 'DbSecret', {
  secretName: 'prod/mysql/app-db',
  generateSecretString: {
    secretStringTemplate: JSON.stringify({ username: 'app_user' }),
    generateStringKey: 'password',
    excludeCharacters: '/@"\' ',  // Loại bỏ ký tự đặc biệt gây vấn đề với JDBC
    passwordLength: 32
  },
  encryptionKey: kmsKey   // CMK tùy chọn
});

// RDS instance tự động dùng secret này
const db = new rds.DatabaseInstance(this, 'AppDb', {
  credentials: rds.Credentials.fromSecret(dbSecret),
  // ...
});
```

---

## 7. Multi-Account & Cross-Region Secrets

### Secrets Manager Multi-Region Replication

```bash
# Replicate secret sang region backup
aws secretsmanager replicate-secret-to-regions \
  --secret-id "prod/mysql/app-db" \
  --add-replica-regions '[
    {"Region": "us-east-1", "KmsKeyId": "arn:aws:kms:us-east-1:...key/mrk-yyy"},
    {"Region": "eu-west-1", "KmsKeyId": "arn:aws:kms:eu-west-1:...key/mrk-zzz"}
  ]'

# Replica secrets:
# - Read-only (không thể modify trực tiếp)
# - Sync tự động khi primary secret thay đổi
# - App ở us-east-1 dùng replica gần nhất → giảm latency
# - Nếu primary region down → failover app dùng replica

# Promote replica (khi primary region fail):
aws secretsmanager remove-regions-from-replication \
  --secret-id "arn:aws:secretsmanager:us-east-1:...:secret:prod/mysql/app-db-xxxxx"
# → Replica trở thành independent secret, có thể rotate độc lập
```

### Cross-Account Access

```bash
# Account A (production): Cho phép Account B (data pipeline) đọc secret
aws secretsmanager put-resource-policy \
  --secret-id "prod/mysql/app-db" \
  --resource-policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT-B:role/data-pipeline-role"
      },
      "Action": ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"],
      "Resource": "*"
    }]
  }'

# Account B cũng cần IAM permission để assume role có thể gọi cross-account
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: Khác biệt chính giữa Secrets Manager và Parameter Store? Khi nào dùng cái nào?**

> Secrets Manager: $0.40/secret/tháng, có native auto-rotation cho RDS, cross-account support, replication multi-region — dùng cho database credentials và bất kỳ secret cần rotate. Parameter Store: free tier (Standard), không có auto-rotation — dùng cho app configuration, host/port, feature flags. Pattern tốt nhất: dùng cả hai kết hợp, Secrets Manager chỉ cho credentials thực sự, Parameter Store cho config thông thường.

**Q: Giải thích 4-step rotation process của Secrets Manager cho RDS.**

> (1) createSecret: tạo version AWSPENDING với password mới random; (2) setSecret: connect DB bằng AWSCURRENT, chạy ALTER USER với password mới — database bây giờ accept cả hai password; (3) testSecret: test connect bằng AWSPENDING — nếu fail thì abort; (4) finishSecret: promote AWSPENDING → AWSCURRENT, old AWSCURRENT → AWSPREVIOUS. Quá trình này zero-downtime vì connections cũ vẫn dùng AWSPREVIOUS trong thời gian ngắn cho đến khi reconnect.

**Q: Application của bạn đang dùng connection pool. Khi Secrets Manager rotate password, làm thế nào để application không bị downtime?**

> Connection pool cần handle `AuthenticationError` (lỗi xác thực) như sau: (1) Khi nhận auth error, invalidate cached credential; (2) Lấy lại secret từ Secrets Manager (lúc này là AWSCURRENT mới); (3) Retry connection tối đa 3 lần với backoff; (4) Nếu thành công → return connection mới vào pool; (5) Connections cũ trong pool sẽ được health-check và replaced dần. Nếu dùng RDS Proxy, nó handle rotation tự động — application không cần biết gì.

**Q: Developer vô tình commit database password lên GitHub. Các bước cần làm ngay lập tức?**

> (1) Rotate password ngay lập tức qua Secrets Manager (không phải chỉnh sửa git); (2) Revoke mọi active database sessions; (3) Check CloudTrail xem có API calls nào với old credentials không; (4) Check Database Activity Streams (nếu đã bật) xem có truy vấn bất thường không; (5) Nếu có evidence truy cập trái phép → escalate incident, notify legal/security team; (6) Remove secret khỏi git history bằng `git filter-branch` hoặc BFG Repo Cleaner; (7) Force-push lên GitHub và notify all contributors để re-clone; (8) Post-mortem: implement git pre-commit hook để detect secrets (gitleaks, truffleHog).

---

## 📂 Điều Hướng

| ← Trước | Chủ Đề Hiện Tại | Tiếp → |
|---------|-----------------|--------|
| [3-encryption.md](./3-encryption.md) | **4-secrets-management.md** | [5-audit-compliance.md](./5-audit-compliance.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15 | **Trạng Thái:** ✅ Hoàn Thành
