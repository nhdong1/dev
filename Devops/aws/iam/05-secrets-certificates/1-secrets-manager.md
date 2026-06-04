# AWS Secrets Manager — Quản Lý Bí Mật Tập Trung

> **Secrets Manager** là dịch vụ quản lý bí mật (credentials, API keys, passwords) với khả năng xoay vòng tự động, mã hóa tích hợp và kiểm soát truy cập chi tiết.

---

## 📚 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Kiến Trúc Và Cách Hoạt Động](#kiến-trúc-và-cách-hoạt-động)
3. [Tạo Và Lưu Secret](#tạo-và-lưu-secret)
4. [Auto-Rotation — Xoay Vòng Tự Động](#auto-rotation--xoay-vòng-tự-động)
5. [Truy Xuất Secret Từ Ứng Dụng](#truy-xuất-secret-từ-ứng-dụng)
6. [Resource Policy — Chính Sách Tài Nguyên](#resource-policy--chính-sách-tài-nguyên)
7. [Replication — Nhân Bản Đa Vùng](#replication--nhân-bản-đa-vùng)
8. [Giám Sát Và Kiểm Toán](#giám-sát-và-kiểm-toán)
9. [Tích Hợp Dịch Vụ Khác](#tích-hợp-dịch-vụ-khác)
10. [Bảo Mật Nâng Cao](#bảo-mật-nâng-cao)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan

### Secrets Manager Giải Quyết Vấn Đề Gì?

```
Vấn Đề Cũ (trước Secrets Manager):
├── Hardcode credentials trong source code → lộ qua git
├── Credentials trong environment variables → lộ qua process list
├── File config không mã hóa → lộ khi instance bị compromise
├── Rotation thủ công → quên rotation, secret cũ tồn tại mãi
└── Không audit ai đọc secret → không phát hiện được rò rỉ

Giải Pháp Của Secrets Manager:
├── Lưu trữ tập trung, mã hóa bằng KMS
├── Ứng dụng gọi API để lấy secret (không hardcode)
├── Auto-rotation qua Lambda (không gián đoạn)
├── CloudTrail audit mọi lần truy xuất
└── IAM + Resource Policy kiểm soát truy cập chi tiết
```

### Các Loại Secret Được Hỗ Trợ

| Loại Secret | Mô Tả | Rotation Native |
|---|---|---|
| **RDS database credentials** | Username/password cho MySQL, PostgreSQL, Oracle, SQL Server | ✅ |
| **Redshift credentials** | Username/password cho data warehouse | ✅ |
| **DocumentDB credentials** | Username/password cho MongoDB-compatible DB | ✅ |
| **Other database credentials** | Các DB khác (tự viết Lambda) | ✅ (custom) |
| **API keys** | Khóa API cho dịch vụ bên thứ ba | ✅ (custom) |
| **OAuth tokens** | Access token, refresh token | ✅ (custom) |
| **Arbitrary text/JSON** | Bất kỳ nội dung nào ≤ 65KB | ✅ (custom) |
| **SSH private keys** | Khóa SSH | ✅ (custom) |

---

## Kiến Trúc Và Cách Hoạt Động

### Luồng Dữ Liệu

```
Application (Ứng Dụng)
    │
    │  1. GetSecretValue API call (qua TLS 1.2+)
    ▼
Secrets Manager Service
    │
    │  2. Kiểm tra IAM permission
    │  3. Gọi KMS để decrypt DEK (Data Encryption Key)
    ▼
KMS CMK (Customer Master Key)
    │
    │  4. Trả DEK đã decrypt
    ▼
Secrets Manager (decrypt secret bằng DEK)
    │
    │  5. Trả secret plaintext về ứng dụng
    ▼
Application (nhận secret, dùng trong memory — không ghi vào disk)
```

### Cấu Trúc Secret

Mỗi secret bao gồm:

```
Secret Object:
├── Metadata (siêu dữ liệu):
│   ├── ARN (Amazon Resource Name): arn:aws:secretsmanager:region:account:secret:name-suffix
│   ├── Name (tên): prod/myapp/db-credentials
│   ├── Description (mô tả)
│   ├── Tags (nhãn)
│   └── KMS Key ID
│
├── Secret Value (giá trị):
│   ├── AWSCURRENT — phiên bản hiện tại, đang dùng
│   ├── AWSPENDING — phiên bản đang trong quá trình rotation
│   └── AWSPREVIOUS — phiên bản trước (giữ lại để rollback)
│
└── Rotation Configuration (cấu hình xoay vòng):
    ├── Lambda Function ARN
    ├── Rotation Schedule (lịch xoay vòng)
    └── Automatically After Days (số ngày)
```

### Version Staging Labels (Nhãn Phiên Bản)

```
Trước rotation:
  v1 [AWSCURRENT]

Trong quá trình rotation:
  v1 [AWSCURRENT, AWSPREVIOUS] ← nếu rollback cần
  v2 [AWSPENDING]              ← đang được tạo/test

Sau rotation thành công:
  v1 [AWSPREVIOUS]
  v2 [AWSCURRENT]
```

---

## Tạo Và Lưu Secret

### Tạo Secret Bằng AWS CLI

```bash
# Secret dạng key-value JSON
aws secretsmanager create-secret \
  --name "prod/myapp/db-credentials" \
  --description "PostgreSQL credentials cho myapp production" \
  --kms-key-id "arn:aws:kms:ap-southeast-1:123456789012:key/mrk-abc123" \
  --secret-string '{
    "username": "app_user",
    "password": "MyStr0ngP@ss!",
    "host": "mydb.cluster-xyz.ap-southeast-1.rds.amazonaws.com",
    "port": 5432,
    "dbname": "myappdb"
  }' \
  --tags '[
    {"Key": "Environment", "Value": "prod"},
    {"Key": "Application", "Value": "myapp"}
  ]'
```

```bash
# Secret dạng plaintext đơn giản
aws secretsmanager create-secret \
  --name "prod/myapp/stripe-api-key" \
  --secret-string "sk_live_xxxxxxxxxxxxxxxx"
```

### Tạo Secret Bằng Terraform (IaC — Infrastructure as Code)

```hcl
resource "aws_secretsmanager_secret" "db_credentials" {
  name        = "prod/myapp/db-credentials"
  description = "PostgreSQL credentials cho myapp production"
  kms_key_id  = aws_kms_key.secrets_key.arn

  # Đợi 7 ngày trước khi xóa thật (tránh xóa nhầm)
  recovery_window_in_days = 7

  tags = {
    Environment = "prod"
    Application = "myapp"
  }
}

resource "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db_credentials.id

  secret_string = jsonencode({
    username = "app_user"
    password = random_password.db_password.result
    host     = aws_db_instance.main.endpoint
    port     = 5432
    dbname   = "myappdb"
  })
}
```

---

## Auto-Rotation — Xoay Vòng Tự Động

### Tại Sao Rotation Cần Lambda?

Secrets Manager không tự biết cách đổi password trong database của bạn. Lambda function đóng vai trò **trung gian** thực hiện các bước:
1. Tạo secret mới
2. Cập nhật trong database
3. Test kết nối
4. Đặt secret mới là AWSCURRENT

### Rotation Strategies (Chiến Lược Xoay Vòng)

#### Chiến Lược 1: Single-User Rotation

```
Trạng Thái Ban Đầu:
  DB User: app_user / password_v1
  Secret: app_user / password_v1 [AWSCURRENT]

Trong Rotation:
  Step 1 — createSecret:   Tạo secret mới password_v2 [AWSPENDING]
  Step 2 — setSecret:      Đổi password app_user trong DB → password_v2
  Step 3 — testSecret:     Test kết nối DB với password_v2
  Step 4 — finishSecret:   Đặt password_v2 → AWSCURRENT, password_v1 → AWSPREVIOUS

Rủi Ro:
  └── Giữa step 2 và step 4: ứng dụng dùng password cũ → kết nối thất bại
  └── Thích hợp khi ứng dụng chấp nhận brief downtime (vài giây)
```

#### Chiến Lược 2: Multi-User Rotation (Khuyến Nghị Cho Production)

```
Trạng Thái Ban Đầu:
  DB User 1: app_user_a / password_a [ACTIVE]
  DB User 2: app_user_b / password_b [STANDBY]
  Secret AWSCURRENT: app_user_a / password_a

Rotation #1:
  Step 1: Tạo secret mới với app_user_b / new_password_b [AWSPENDING]
  Step 2: Cập nhật password của app_user_b trong DB
  Step 3: Test kết nối với app_user_b / new_password_b
  Step 4: app_user_b / new_password_b → AWSCURRENT

Kết Quả: Ứng dụng chuyển sang dùng app_user_b
Không có downtime vì app_user_a vẫn hoạt động trong quá trình rotation

Rotation #2 (lần sau):
  Lặp lại với app_user_a
```

### Bật Rotation Cho RDS (Native Integration)

```bash
aws secretsmanager rotate-secret \
  --secret-id "prod/myapp/db-credentials" \
  --rotation-lambda-arn "arn:aws:lambda:ap-southeast-1:123456789012:function:SecretsManagerRDSPostgreSQLRotation" \
  --rotation-rules '{
    "AutomaticallyAfterDays": 30
  }'
```

### Lambda Rotation Function — Cấu Trúc Cơ Bản

```python
import boto3
import json

def lambda_handler(event, context):
    """
    Lambda được Secrets Manager gọi với 4 steps.
    """
    arn = event['SecretId']
    token = event['ClientRequestToken']
    step = event['Step']

    service_client = boto3.client('secretsmanager')

    # Kiểm tra secret tồn tại và pending version hợp lệ
    metadata = service_client.describe_secret(SecretId=arn)
    if not metadata['RotationEnabled']:
        raise ValueError(f"Secret {arn} không bật rotation")

    versions = metadata['VersionIdsToStages']
    if token not in versions:
        raise ValueError(f"Token {token} không tồn tại trong secret {arn}")

    if step == "createSecret":
        create_secret(service_client, arn, token)
    elif step == "setSecret":
        set_secret(service_client, arn, token)
    elif step == "testSecret":
        test_secret(service_client, arn, token)
    elif step == "finishSecret":
        finish_secret(service_client, arn, token)
    else:
        raise ValueError(f"Step không hợp lệ: {step}")


def create_secret(client, arn, token):
    """Bước 1: Tạo secret mới với giá trị mới."""
    try:
        # Nếu AWSPENDING đã tồn tại thì bỏ qua (idempotent)
        client.get_secret_value(SecretId=arn, VersionStage="AWSPENDING",
                                VersionId=token)
        return
    except client.exceptions.ResourceNotFoundException:
        pass

    # Lấy secret hiện tại để giữ cấu trúc
    current = json.loads(
        client.get_secret_value(SecretId=arn, VersionStage="AWSCURRENT")['SecretString']
    )

    # Tạo password mới an toàn
    new_password = client.get_random_password(
        PasswordLength=32,
        ExcludeCharacters='/@"\''
    )['RandomPassword']

    current['password'] = new_password

    # Lưu version mới với label AWSPENDING
    client.put_secret_value(
        SecretId=arn,
        ClientRequestToken=token,
        SecretString=json.dumps(current),
        VersionStages=['AWSPENDING']
    )


def set_secret(client, arn, token):
    """Bước 2: Áp dụng secret mới vào database."""
    pending = json.loads(
        client.get_secret_value(SecretId=arn, VersionStage="AWSPENDING",
                                VersionId=token)['SecretString']
    )
    current = json.loads(
        client.get_secret_value(SecretId=arn, VersionStage="AWSCURRENT")['SecretString']
    )

    # Kết nối DB bằng credentials hiện tại và đổi password
    conn = get_db_connection(current)
    with conn.cursor() as cursor:
        cursor.execute(
            "ALTER USER %s WITH PASSWORD %s",
            (pending['username'], pending['password'])
        )
    conn.commit()
    conn.close()


def test_secret(client, arn, token):
    """Bước 3: Test kết nối với secret mới."""
    pending = json.loads(
        client.get_secret_value(SecretId=arn, VersionStage="AWSPENDING",
                                VersionId=token)['SecretString']
    )

    # Test kết nối DB với credentials mới
    conn = get_db_connection(pending)
    conn.close()


def finish_secret(client, arn, token):
    """Bước 4: Đặt AWSPENDING thành AWSCURRENT."""
    metadata = client.describe_secret(SecretId=arn)
    current_version = None
    for version, stages in metadata['VersionIdsToStages'].items():
        if 'AWSCURRENT' in stages:
            if version == token:
                return  # Đã là AWSCURRENT
            current_version = version
            break

    client.update_secret_version_stage(
        SecretId=arn,
        VersionStage='AWSCURRENT',
        MoveToVersionId=token,
        RemoveFromVersionId=current_version
    )


def get_db_connection(credentials):
    """Helper: tạo kết nối DB từ credentials dict."""
    import psycopg2
    return psycopg2.connect(
        host=credentials['host'],
        port=credentials['port'],
        database=credentials['dbname'],
        user=credentials['username'],
        password=credentials['password'],
        connect_timeout=5
    )
```

### Rotation Schedule Options (Tùy Chọn Lịch Xoay Vòng)

```bash
# Xoay vòng mỗi 30 ngày
--rotation-rules '{"AutomaticallyAfterDays": 30}'

# Xoay vòng theo lịch cron (ví dụ: 3 giờ sáng Chủ Nhật hàng tuần)
--rotation-rules '{
  "ScheduleExpression": "cron(0 3 ? * SUN *)",
  "Duration": "2h"
}'

# Duration: thời gian tối đa Lambda được phép chạy để hoàn thành rotation
```

---

## Truy Xuất Secret Từ Ứng Dụng

### Python — AWS SDK (Boto3)

```python
import boto3
import json
from botocore.exceptions import ClientError
from functools import lru_cache
import time

# Cache secret trong memory để tránh gọi API mỗi request
_secret_cache = {}
_cache_ttl = 300  # 5 phút

def get_secret(secret_name: str, region: str = "ap-southeast-1") -> dict:
    """Lấy secret và cache trong memory."""
    cache_key = f"{region}/{secret_name}"
    now = time.time()

    # Kiểm tra cache còn hạn
    if cache_key in _secret_cache:
        value, timestamp = _secret_cache[cache_key]
        if now - timestamp < _cache_ttl:
            return value

    # Gọi Secrets Manager API
    client = boto3.client('secretsmanager', region_name=region)

    try:
        response = client.get_secret_value(SecretId=secret_name)
    except ClientError as e:
        error_code = e.response['Error']['Code']
        if error_code == 'ResourceNotFoundException':
            raise Exception(f"Secret '{secret_name}' không tồn tại")
        elif error_code == 'AccessDeniedException':
            raise Exception(f"Không có quyền truy cập secret '{secret_name}'")
        raise

    # Parse JSON nếu cần
    secret_string = response.get('SecretString', '')
    try:
        value = json.loads(secret_string)
    except json.JSONDecodeError:
        value = secret_string  # Plaintext string

    # Lưu vào cache
    _secret_cache[cache_key] = (value, now)
    return value


# Sử dụng trong ứng dụng
def get_db_config() -> dict:
    secret = get_secret("prod/myapp/db-credentials")
    return {
        "host": secret["host"],
        "port": secret["port"],
        "database": secret["dbname"],
        "user": secret["username"],
        "password": secret["password"]
    }
```

### Java — AWS SDK v2

```java
import software.amazon.awssdk.services.secretsmanager.SecretsManagerClient;
import software.amazon.awssdk.services.secretsmanager.model.*;
import com.fasterxml.jackson.databind.ObjectMapper;

public class SecretsManagerHelper {

    private static final SecretsManagerClient client =
        SecretsManagerClient.builder()
            .region(Region.AP_SOUTHEAST_1)
            .build();

    public static String getSecretValue(String secretName) {
        GetSecretValueRequest request = GetSecretValueRequest.builder()
            .secretId(secretName)
            .build();

        try {
            GetSecretValueResponse response = client.getSecretValue(request);
            return response.secretString();
        } catch (ResourceNotFoundException e) {
            throw new RuntimeException("Secret không tồn tại: " + secretName, e);
        } catch (SecretsManagerException e) {
            throw new RuntimeException("Lỗi khi lấy secret: " + e.getMessage(), e);
        }
    }
}
```

### Node.js — AWS SDK v3

```javascript
import { SecretsManagerClient, GetSecretValueCommand } from "@aws-sdk/client-secrets-manager";

const client = new SecretsManagerClient({ region: "ap-southeast-1" });

// Cache đơn giản với TTL
const secretCache = new Map();
const CACHE_TTL_MS = 5 * 60 * 1000; // 5 phút

async function getSecret(secretName) {
  const now = Date.now();
  const cached = secretCache.get(secretName);

  if (cached && (now - cached.timestamp) < CACHE_TTL_MS) {
    return cached.value;
  }

  const command = new GetSecretValueCommand({ SecretId: secretName });
  const response = await client.send(command);

  let value;
  try {
    value = JSON.parse(response.SecretString);
  } catch {
    value = response.SecretString;
  }

  secretCache.set(secretName, { value, timestamp: now });
  return value;
}

// Sử dụng
const dbCreds = await getSecret("prod/myapp/db-credentials");
console.log(`Kết nối tới ${dbCreds.host}:${dbCreds.port}`);
```

### Tích Hợp Với ECS Task — Environment Variable Injection

Thay vì inject secret vào env var (vẫn có rủi ro lộ qua process list), dùng **ECS Secrets Integration**:

```json
{
  "containerDefinitions": [
    {
      "name": "myapp",
      "image": "myapp:latest",
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:prod/myapp/db-credentials:password::"
        }
      ],
      "environment": [
        {
          "name": "DB_HOST",
          "value": "mydb.cluster-xyz.rds.amazonaws.com"
        }
      ]
    }
  ]
}
```

---

## Resource Policy — Chính Sách Tài Nguyên

Resource policy cho phép kiểm soát truy cập trực tiếp trên secret, hữu ích cho cross-account access (truy cập liên tài khoản).

### Cho Phép Account Khác Đọc Secret

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCrossAccountRead",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::987654321098:role/AppRole"
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

### Chặn Mọi Truy Cập Ngoại Trừ VPC Endpoint

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonVPCAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpc": "vpc-0123456789abcdef0"
        }
      }
    }
  ]
}
```

---

## Replication — Nhân Bản Đa Vùng

Multi-region replication (nhân bản đa vùng) cho phép ứng dụng ở nhiều region đọc secret từ region gần nhất, giảm latency (độ trễ).

```bash
# Nhân bản secret sang region khác
aws secretsmanager replicate-secret-to-regions \
  --secret-id "prod/myapp/db-credentials" \
  --add-replica-regions '[
    {
      "Region": "ap-northeast-1",
      "KmsKeyId": "arn:aws:kms:ap-northeast-1:123456789012:key/mrk-def456"
    }
  ]'
```

```
Primary Secret (ap-southeast-1)
    │
    ├── Read/Write: rotation xảy ra ở đây
    │
    └── Replica (ap-northeast-1)
        └── Read-only: ứng dụng Tokyo đọc từ đây
        └── Sync tự động khi primary thay đổi
```

**Lưu ý:** Dùng **MRK (Multi-Region Key — Khóa Đa Vùng)** trong KMS để mã hóa replica — một CMK có thể dùng ở nhiều region.

---

## Giám Sát Và Kiểm Toán

### CloudTrail Events Quan Trọng

| Event | Ý Nghĩa | Cần Alert? |
|---|---|---|
| `GetSecretValue` | Ai đó đọc giá trị secret | ✅ Nếu từ IP lạ |
| `RotateSecret` | Bắt đầu xoay vòng | - |
| `RotationSucceeded` | Rotation thành công | - |
| `RotationFailed` | Rotation thất bại | ✅ Luôn alert |
| `DeleteSecret` | Xóa secret | ✅ Luôn alert |
| `PutSecretValue` | Ghi giá trị mới | ✅ Nếu ngoài giờ hành chính |

### CloudWatch Metrics Và Alerts

```bash
# Alert khi rotation thất bại
aws cloudwatch put-metric-alarm \
  --alarm-name "SecretsManager-RotationFailed" \
  --metric-name "ResourceCount" \
  --namespace "AWS/SecretsManager" \
  --dimensions Name=SecretId,Value="prod/myapp/db-credentials" \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions "arn:aws:sns:ap-southeast-1:123456789012:security-alerts"
```

### AWS Config Rule — Kiểm Tra Rotation Đang Bật

```
Managed Rule: secretsmanager-rotation-enabled-check
Mô tả: Kiểm tra mọi secret có bật auto-rotation
Trigger: Periodic (định kỳ)
Remediation: Bật rotation với default Lambda
```

---

## Tích Hợp Dịch Vụ Khác

### RDS Integration — Tích Hợp Với Database

```bash
# Khi tạo RDS instance, liên kết với Secrets Manager
aws rds create-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --master-username admin \
  --manage-master-user-password \
  --master-user-secret-kms-key-id "arn:aws:kms:...:key/mrk-abc"
```

RDS sẽ tự động:
- Tạo secret trong Secrets Manager
- Cấu hình rotation tự động
- Quản lý Lambda rotation function

### Lambda Integration (Tích Hợp Với Lambda Function)

```python
# Trong Lambda, lấy secret qua AWS Lambda Powertools
from aws_lambda_powertools.utilities import parameters

# Tự động cache 5 phút, tự động decrypt
db_secret = parameters.get_secret(
    "prod/myapp/db-credentials",
    transform="json"
)

# Hoặc với custom TTL (Time to Live — Thời Gian Sống)
db_secret = parameters.get_secret(
    "prod/myapp/db-credentials",
    transform="json",
    max_age=300  # giây
)
```

### Kubernetes Integration — External Secrets Operator

```yaml
# ExternalSecret resource trong K8s
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 5m
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: db-credentials  # Tên K8s Secret được tạo ra
  data:
    - secretKey: password
      remoteRef:
        key: prod/myapp/db-credentials
        property: password
    - secretKey: username
      remoteRef:
        key: prod/myapp/db-credentials
        property: username
```

---

## Bảo Mật Nâng Cao

### VPC Endpoint — Ngăn Traffic Qua Internet

```bash
# Tạo Interface VPC Endpoint cho Secrets Manager
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0123456789abcdef0 \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.ap-southeast-1.secretsmanager \
  --subnet-ids subnet-abc123 subnet-def456 \
  --security-group-ids sg-xyz789 \
  --private-dns-enabled
```

### Deletion Protection — Bảo Vệ Xóa Nhầm

```bash
# Tạo secret với deletion window 30 ngày (tối đa)
aws secretsmanager create-secret \
  --name "prod/critical/secret" \
  --recovery-window-in-days 30

# Xóa secret — chỉ đặt lịch xóa, chưa xóa ngay
aws secretsmanager delete-secret \
  --secret-id "prod/critical/secret" \
  --recovery-window-in-days 30

# Khôi phục nếu xóa nhầm (trong window)
aws secretsmanager restore-secret \
  --secret-id "prod/critical/secret"

# Xóa ngay lập tức (không khuyến nghị trong production)
aws secretsmanager delete-secret \
  --secret-id "prod/critical/secret" \
  --force-delete-without-recovery
```

### SCPs Để Ngăn Xóa Secret Quan Trọng

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDeleteCriticalSecrets",
      "Effect": "Deny",
      "Action": [
        "secretsmanager:DeleteSecret",
        "secretsmanager:RemoveRegionsFromReplication"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:prod/*",
      "Condition": {
        "StringNotLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:role/SecretAdminRole"
        }
      }
    }
  ]
}
```

---

## Chi Phí (Pricing)

| Thành Phần | Chi Phí |
|---|---|
| Mỗi secret lưu trữ | $0.40/secret/tháng |
| Mỗi 10,000 API calls | $0.05 |
| Replica secret | $0.40/replica/tháng |
| Rotation Lambda | Chi phí Lambda riêng (thường rất thấp) |

**Ví dụ:** 100 secrets, 1 triệu API calls/tháng = $40 + $5 = **$45/tháng**

---

## Câu Hỏi Phỏng Vấn

**Q1: Secrets Manager khác gì với hardcoding credentials trong environment variables?**

> Secrets Manager mã hóa secrets bằng KMS, cung cấp audit trail qua CloudTrail, hỗ trợ auto-rotation mà không cần redeploy ứng dụng, và kiểm soát truy cập qua IAM. Environment variables không có mã hóa at-rest, không có audit, và không thể rotate mà không restart container/instance.

**Q2: Giải thích 4 bước rotation và tại sao cần đến 4 bước?**

> 4 bước (createSecret → setSecret → testSecret → finishSecret) được thiết kế để **idempotent** (có thể chạy lại an toàn) và **rollback-safe** (an toàn khi rollback). Nếu Lambda crash ở bất kỳ bước nào, Secrets Manager sẽ retry từ bước đó. Test trước khi finalize đảm bảo secret mới hoạt động trước khi switch.

**Q3: Multi-user rotation khác single-user như thế nào? Khi nào dùng?**

> Single-user: đổi password user hiện tại → có brief window mà ứng dụng dùng password cũ không kết nối được. Multi-user: luân phiên hai user (A và B), một cái luôn hoạt động → zero downtime. Dùng multi-user cho production database khi không thể chấp nhận gián đoạn.

**Q4: Làm sao share secret giữa hai AWS account?**

> Thêm resource policy vào secret cho phép role từ account khác gọi `GetSecretValue`. Nếu secret mã hóa bằng CMK, cần thêm KMS key grant cho role đó. Không cần share qua VPC peering nếu dùng VPC Endpoint riêng ở mỗi account.

**Q5: Secrets Manager vs Parameter Store — chọn cái nào?**

> Xem chi tiết tại [3-secrets-vs-parameter.md](3-secrets-vs-parameter.md). Tóm tắt: Secrets Manager khi cần auto-rotation, cross-account, database native integration. Parameter Store khi cần hierarchy config, chi phí thấp, không cần rotation.

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
