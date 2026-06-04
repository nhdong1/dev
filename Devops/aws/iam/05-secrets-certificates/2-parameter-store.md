# SSM Parameter Store — Quản Lý Cấu Hình Và Bí Mật Theo Phân Cấp

> **AWS Systems Manager Parameter Store** là dịch vụ lưu trữ cấu hình và bí mật có phân cấp, mã hóa tùy chọn qua KMS, với khả năng versioning (quản lý phiên bản) và audit trail đầy đủ.

---

## 📚 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Các Loại Parameter](#các-loại-parameter)
3. [Standard vs Advanced Tier](#standard-vs-advanced-tier)
4. [Phân Cấp Tham Số — Hierarchy](#phân-cấp-tham-số--hierarchy)
5. [SecureString — Tham Số Mã Hóa](#securestring--tham-số-mã-hóa)
6. [Versioning — Quản Lý Phiên Bản](#versioning--quản-lý-phiên-bản)
7. [Parameter Policies — Chính Sách Tham Số](#parameter-policies--chính-sách-tham-số)
8. [Truy Xuất Từ Ứng Dụng](#truy-xuất-từ-ứng-dụng)
9. [IAM Access Control](#iam-access-control)
10. [Tích Hợp Với Dịch Vụ AWS](#tích-hợp-với-dịch-vụ-aws)
11. [Giám Sát Và Audit](#giám-sát-và-audit)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan

### Parameter Store Là Gì?

Parameter Store là một tính năng của **AWS Systems Manager (SSM)**, cho phép lưu trữ:
- **Configuration data** (dữ liệu cấu hình): database hostnames, feature flags, API endpoints
- **Secrets** (bí mật): passwords, API keys, tokens — dùng loại SecureString

### So Sánh Với Secrets Manager

| Khía Cạnh | Parameter Store | Secrets Manager |
|---|---|---|
| Chi phí cơ bản | Miễn phí (Standard) | $0.40/secret/tháng |
| Auto-rotation | ❌ Không có native | ✅ Có sẵn |
| Phân cấp (Hierarchy) | ✅ Native | ❌ Chỉ dùng tên |
| Kích thước | 4KB (Standard) / 8KB (Advanced) | 65KB |
| Use case chính | Config + simple secrets | Credentials cần rotation |

*Để so sánh đầy đủ, xem [3-secrets-vs-parameter.md](3-secrets-vs-parameter.md)*

---

## Các Loại Parameter

### 1. String

```bash
# Tham số văn bản thuần, không mã hóa
aws ssm put-parameter \
  --name "/myapp/prod/db/host" \
  --type "String" \
  --value "mydb.cluster-xyz.ap-southeast-1.rds.amazonaws.com"
```

**Dùng cho:** hostnames, URLs, feature flags, non-sensitive config

### 2. StringList

```bash
# Danh sách giá trị ngăn cách bằng dấu phẩy
aws ssm put-parameter \
  --name "/myapp/prod/allowed-ips" \
  --type "StringList" \
  --value "10.0.0.1,10.0.0.2,10.0.0.3"
```

**Dùng cho:** danh sách IP được phép, danh sách feature flags, whitelist (danh sách cho phép)

### 3. SecureString

```bash
# Tham số mã hóa bằng KMS
aws ssm put-parameter \
  --name "/myapp/prod/db/password" \
  --type "SecureString" \
  --key-id "arn:aws:kms:ap-southeast-1:123456789012:key/mrk-abc123" \
  --value "MyStr0ngP@ss!"
```

**Dùng cho:** passwords, API keys, tokens, SSH keys, certificates

---

## Standard vs Advanced Tier

### So Sánh Chi Tiết

| Tính Năng | Standard Tier | Advanced Tier |
|---|---|---|
| **Chi phí lưu trữ** | Miễn phí (đến 10,000 params) | $0.05/param/tháng |
| **Chi phí API** | Miễn phí (Higher throughput: có phí) | $0.05/10K API calls |
| **Kích thước tối đa** | 4KB | 8KB |
| **Số lượng tối đa** | 10,000 per account/region | 100,000 per account/region |
| **Parameter Policies** | ❌ | ✅ (TTL, notification, no-change-notification) |
| **Throughput (thông lượng)** | 40 TPS (Transaction Per Second) | 1,000 TPS |
| **Chuyển đổi** | Standard → Advanced: ✅ | Advanced → Standard: ❌ |

### Khi Nào Dùng Advanced?

- Cần lưu hơn 10,000 parameters
- Cần parameter có TTL (hết hạn tự động)
- Cần nhận notification khi parameter không thay đổi trong thời gian dài
- Cần throughput cao (>40 TPS)
- Lưu trữ dữ liệu lớn hơn 4KB (ví dụ: certificate content)

---

## Phân Cấp Tham Số — Hierarchy

### Cấu Trúc Phân Cấp

Parameter Store hỗ trợ đường dẫn phân cấp tối đa **15 cấp** sử dụng dấu `/`.

```
/                                          ← Root
├── myapp/                                 ← Ứng dụng
│   ├── prod/                              ← Môi trường
│   │   ├── db/
│   │   │   ├── host                       → "mydb.cluster.rds.amazonaws.com"
│   │   │   ├── port                       → "5432"
│   │   │   ├── name                       → "myappdb"
│   │   │   └── password (SecureString)    → "encrypted:..."
│   │   ├── redis/
│   │   │   ├── host                       → "redis.xyz.cache.amazonaws.com"
│   │   │   └── port                       → "6379"
│   │   └── features/
│   │       ├── new-checkout               → "true"
│   │       └── dark-mode                  → "false"
│   ├── staging/
│   │   ├── db/
│   │   └── redis/
│   └── dev/
│       ├── db/
│       └── redis/
│
└── shared/                                ← Config dùng chung
    ├── monitoring/
    │   ├── datadog-api-key (SecureString)
    │   └── pagerduty-key (SecureString)
    └── certificates/
        └── internal-ca-cert
```

### Lấy Cả Nhóm Parameter Theo Path

```bash
# Lấy tất cả parameters trong /myapp/prod/db/
aws ssm get-parameters-by-path \
  --path "/myapp/prod/db/" \
  --recursive \
  --with-decryption

# Output:
# {
#   "Parameters": [
#     {"Name": "/myapp/prod/db/host", "Value": "mydb.cluster...", "Type": "String"},
#     {"Name": "/myapp/prod/db/port", "Value": "5432", "Type": "String"},
#     {"Name": "/myapp/prod/db/password", "Value": "MyStr0ngP@ss!", "Type": "SecureString"}
#   ]
# }
```

```bash
# Lấy tất cả parameters của staging để so sánh với prod
aws ssm get-parameters-by-path \
  --path "/myapp/staging/" \
  --recursive \
  --with-decryption \
  --query "Parameters[*].{Name:Name,Value:Value}"
```

### Tạo Parameters Hàng Loạt Bằng Script

```bash
#!/bin/bash
# Script tạo parameters cho môi trường mới

ENVIRONMENT=$1  # prod, staging, dev
APP="myapp"

declare -A PARAMS=(
  ["db/host"]="mydb-${ENVIRONMENT}.cluster.rds.amazonaws.com"
  ["db/port"]="5432"
  ["db/name"]="${APP}db"
  ["redis/host"]="redis-${ENVIRONMENT}.cache.amazonaws.com"
  ["redis/port"]="6379"
)

for KEY in "${!PARAMS[@]}"; do
  aws ssm put-parameter \
    --name "/${APP}/${ENVIRONMENT}/${KEY}" \
    --value "${PARAMS[$KEY]}" \
    --type "String" \
    --overwrite
  echo "Đã tạo: /${APP}/${ENVIRONMENT}/${KEY}"
done
```

---

## SecureString — Tham Số Mã Hóa

### Cách Hoạt Động

```
put-parameter (SecureString):
  1. Bạn gửi plaintext value qua TLS
  2. Parameter Store gọi KMS Encrypt với CMK chỉ định
  3. KMS trả về ciphertext (văn bản mã hóa)
  4. Ciphertext được lưu vào Parameter Store

get-parameter (--with-decryption):
  1. IAM kiểm tra permission
  2. Parameter Store gọi KMS Decrypt
  3. KMS kiểm tra key policy + IAM grant
  4. KMS trả về plaintext DEK
  5. Parameter Store decrypt value và trả về bạn
```

### Tạo SecureString

```bash
# Dùng AWS managed key (aws/ssm) — mặc định, không tốn phí KMS riêng
aws ssm put-parameter \
  --name "/myapp/prod/db/password" \
  --type "SecureString" \
  --value "MyStr0ngP@ss!"

# Dùng CMK của bạn — kiểm soát tốt hơn
aws ssm put-parameter \
  --name "/myapp/prod/db/password" \
  --type "SecureString" \
  --key-id "alias/myapp-secrets-key" \
  --value "MyStr0ngP@ss!"
```

### Đọc SecureString

```bash
# Phải thêm --with-decryption để nhận giá trị thật
aws ssm get-parameter \
  --name "/myapp/prod/db/password" \
  --with-decryption \
  --query "Parameter.Value" \
  --output text

# Nếu không có --with-decryption → nhận về chuỗi mã hóa base64
```

### IAM Permission Cần Thiết Cho SecureString

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSSMRead",
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath"
      ],
      "Resource": "arn:aws:ssm:ap-southeast-1:123456789012:parameter/myapp/prod/*"
    },
    {
      "Sid": "AllowKMSDecrypt",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:ap-southeast-1:123456789012:key/mrk-abc123",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "ssm.ap-southeast-1.amazonaws.com"
        }
      }
    }
  ]
}
```

---

## Versioning — Quản Lý Phiên Bản

Parameter Store tự động tạo phiên bản mới mỗi khi update.

### Cách Hoạt Động

```
Lần 1: put-parameter "/myapp/prod/db/host" = "db1.example.com"
  └── Version 1: "db1.example.com"  ← CURRENT

Lần 2: put-parameter "/myapp/prod/db/host" = "db2.example.com"
  └── Version 1: "db1.example.com"  ← lịch sử
  └── Version 2: "db2.example.com"  ← CURRENT

Lần 3: put-parameter --overwrite
  └── Version 1: "db1.example.com"  ← lịch sử
  └── Version 2: "db2.example.com"  ← lịch sử
  └── Version 3: "db3.example.com"  ← CURRENT
```

**Giới hạn:** Standard giữ tối đa **100 phiên bản** per parameter. Phiên bản cũ nhất tự động bị xóa khi vượt giới hạn.

### Truy Xuất Phiên Bản Cụ Thể

```bash
# Lấy phiên bản hiện tại
aws ssm get-parameter --name "/myapp/prod/db/host"

# Lấy phiên bản cụ thể (rollback — khôi phục)
aws ssm get-parameter --name "/myapp/prod/db/host:2"

# Xem lịch sử tất cả phiên bản
aws ssm get-parameter-history \
  --name "/myapp/prod/db/host" \
  --query "Parameters[*].{Version:Version,Value:Value,LastModifiedDate:LastModifiedDate}"
```

### Labels — Nhãn Phiên Bản

Labels (nhãn) cho phép đặt tên có ý nghĩa cho từng phiên bản:

```bash
# Gắn label "stable" cho phiên bản 3
aws ssm label-parameter-version \
  --name "/myapp/prod/db/host" \
  --parameter-version 3 \
  --labels "stable"

# Gắn label "canary" cho phiên bản 4 (đang test)
aws ssm label-parameter-version \
  --name "/myapp/prod/db/host" \
  --parameter-version 4 \
  --labels "canary"

# Ứng dụng đọc theo label thay vì số phiên bản cứng
aws ssm get-parameter --name "/myapp/prod/db/host:stable"
```

**Pattern hay dùng:**
```
- stable:    phiên bản đã kiểm thử, dùng trong prod
- canary:    phiên bản đang test với % traffic nhỏ
- rollback:  phiên bản trước để phòng khi cần quay lại
```

---

## Parameter Policies — Chính Sách Tham Số

Chỉ có ở **Advanced Tier**. Cho phép đặt hành vi tự động trên parameter.

### 1. Expiration Policy (Chính Sách Hết Hạn)

```bash
aws ssm put-parameter \
  --name "/myapp/prod/temp-access-key" \
  --type "SecureString" \
  --value "AKIAIOSFODNN7EXAMPLE" \
  --tier "Advanced" \
  --policies '[
    {
      "Type": "Expiration",
      "Version": "1.0",
      "Attributes": {
        "Timestamp": "2026-12-31T00:00:00.000Z"
      }
    }
  ]'
# Parameter tự động bị xóa vào ngày 31/12/2026
```

### 2. ExpirationNotification Policy (Thông Báo Sắp Hết Hạn)

```bash
# Gửi thông báo qua EventBridge 7 ngày trước khi hết hạn
'[
  {
    "Type": "ExpirationNotification",
    "Version": "1.0",
    "Attributes": {
      "Before": "7",
      "Unit": "Days"
    }
  }
]'
```

### 3. NoChangeNotification Policy (Thông Báo Không Thay Đổi)

```bash
# Cảnh báo nếu secret không được rotate trong 90 ngày
'[
  {
    "Type": "NoChangeNotification",
    "Version": "1.0",
    "Attributes": {
      "After": "90",
      "Unit": "Days"
    }
  }
]'
```

### Kết Hợp Nhiều Policies

```bash
aws ssm put-parameter \
  --name "/myapp/prod/api-key" \
  --tier "Advanced" \
  --type "SecureString" \
  --value "sk_live_xxx" \
  --policies '[
    {
      "Type": "Expiration",
      "Version": "1.0",
      "Attributes": {"Timestamp": "2026-12-31T00:00:00.000Z"}
    },
    {
      "Type": "ExpirationNotification",
      "Version": "1.0",
      "Attributes": {"Before": "14", "Unit": "Days"}
    },
    {
      "Type": "NoChangeNotification",
      "Version": "1.0",
      "Attributes": {"After": "30", "Unit": "Days"}
    }
  ]'
```

---

## Truy Xuất Từ Ứng Dụng

### Python — Boto3

```python
import boto3
import json
from functools import lru_cache

ssm = boto3.client('ssm', region_name='ap-southeast-1')


def get_parameter(name: str, decrypt: bool = False) -> str:
    """Lấy một parameter theo tên."""
    response = ssm.get_parameter(
        Name=name,
        WithDecryption=decrypt
    )
    return response['Parameter']['Value']


def get_parameters_by_path(path: str, decrypt: bool = False) -> dict:
    """Lấy tất cả parameters dưới một path và trả về dict."""
    params = {}
    paginator = ssm.get_paginator('get_parameters_by_path')

    for page in paginator.paginate(
        Path=path,
        Recursive=True,
        WithDecryption=decrypt
    ):
        for param in page['Parameters']:
            # Lấy phần key sau path prefix
            key = param['Name'].replace(path, '').lstrip('/')
            # Parse JSON nếu có thể
            try:
                params[key] = json.loads(param['Value'])
            except (json.JSONDecodeError, TypeError):
                params[key] = param['Value']

    return params


# Sử dụng
db_config = get_parameters_by_path('/myapp/prod/db/', decrypt=True)
# Kết quả: {'host': 'mydb...', 'port': '5432', 'password': 'MyStr0ngP@ss!'}

# Kết nối database
import psycopg2
conn = psycopg2.connect(
    host=db_config['host'],
    port=int(db_config['port']),
    database=db_config['name'],
    user=db_config['user'],
    password=db_config['password']
)
```

### Python — AWS Lambda Powertools (Khuyến Nghị)

```python
from aws_lambda_powertools.utilities import parameters

# Lấy một parameter
db_host = parameters.get_parameter("/myapp/prod/db/host")

# Lấy SecureString (tự động decrypt)
db_password = parameters.get_parameter(
    "/myapp/prod/db/password",
    decrypt=True,
    max_age=300  # Cache 5 phút
)

# Lấy cả path và transform thành dict
db_config = parameters.get_parameters_by_name(
    parameters={
        "/myapp/prod/db/host": {},
        "/myapp/prod/db/port": {},
        "/myapp/prod/db/password": {"decrypt": True},
    },
    max_age=300
)
```

### Bash Shell Script

```bash
#!/bin/bash
# Script lấy config từ Parameter Store để dùng trong shell

get_param() {
  local name=$1
  local decrypt=${2:-false}

  if [ "$decrypt" = "true" ]; then
    aws ssm get-parameter \
      --name "$name" \
      --with-decryption \
      --query "Parameter.Value" \
      --output text
  else
    aws ssm get-parameter \
      --name "$name" \
      --query "Parameter.Value" \
      --output text
  fi
}

# Sử dụng trong deployment script
DB_HOST=$(get_param "/myapp/prod/db/host")
DB_PASSWORD=$(get_param "/myapp/prod/db/password" "true")
FEATURE_FLAG=$(get_param "/myapp/prod/features/new-checkout")

echo "Kết nối tới $DB_HOST"
echo "Feature flag new-checkout: $FEATURE_FLAG"

# Không in DB_PASSWORD ra log!
```

### CloudFormation Dynamic References

```yaml
# Tham chiếu Parameter Store trực tiếp trong CloudFormation
Resources:
  MyDBInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceIdentifier: myapp-prod
      MasterUsername: '{{resolve:ssm:/myapp/prod/db/username}}'
      # SecureString trong CloudFormation
      MasterUserPassword: '{{resolve:ssm-secure:/myapp/prod/db/password:3}}'
      #                                                              ^^^^
      #                                                              Version 3 cụ thể
```

---

## IAM Access Control

### Kiểm Soát Theo Path (Phân Cấp)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadProdConfig",
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath",
        "ssm:GetParameterHistory"
      ],
      "Resource": [
        "arn:aws:ssm:ap-southeast-1:123456789012:parameter/myapp/prod/*"
      ]
    },
    {
      "Sid": "DecryptSecureStrings",
      "Effect": "Allow",
      "Action": "kms:Decrypt",
      "Resource": "arn:aws:kms:ap-southeast-1:123456789012:key/mrk-abc123",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "ssm.ap-southeast-1.amazonaws.com"
        }
      }
    }
  ]
}
```

### Phân Tách Quyền Đọc/Ghi

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AppReadOnly",
      "Effect": "Allow",
      "Action": ["ssm:GetParameter", "ssm:GetParametersByPath"],
      "Resource": "arn:aws:ssm:*:*:parameter/myapp/prod/*"
    },
    {
      "Sid": "OpsWriteAccess",
      "Effect": "Allow",
      "Action": ["ssm:PutParameter", "ssm:DeleteParameter", "ssm:LabelParameterVersion"],
      "Resource": "arn:aws:ssm:*:*:parameter/myapp/prod/*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalTag/Role": "ops-engineer"
        }
      }
    }
  ]
}
```

### Ngăn Dev Đọc Production Secrets

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEnvAccess",
      "Effect": "Allow",
      "Action": "ssm:GetParametersByPath",
      "Resource": "arn:aws:ssm:*:*:parameter/myapp/${aws:PrincipalTag/Environment}/*"
    }
  ]
}
```

*Dev với tag `Environment=dev` chỉ đọc được `/myapp/dev/*`. Prod engineer với tag `Environment=prod` đọc được `/myapp/prod/*`.*

---

## Tích Hợp Với Dịch Vụ AWS

### ECS Task Definition

```json
{
  "containerDefinitions": [{
    "name": "myapp",
    "image": "myapp:latest",
    "secrets": [
      {
        "name": "DB_PASSWORD",
        "valueFrom": "arn:aws:ssm:ap-southeast-1:123456789012:parameter/myapp/prod/db/password"
      }
    ],
    "environment": [
      {
        "name": "DB_HOST",
        "value": "{{resolve:ssm:/myapp/prod/db/host}}"
      }
    ]
  }]
}
```

### EC2 Run Command & User Data

```bash
#!/bin/bash
# User data script — lấy config khi khởi động EC2

# Cài SSM Agent (thường đã có sẵn trên Amazon Linux)
yum install -y amazon-ssm-agent

# Lấy database config
export DB_HOST=$(aws ssm get-parameter \
  --name "/myapp/prod/db/host" \
  --query "Parameter.Value" --output text)

export DB_PASSWORD=$(aws ssm get-parameter \
  --name "/myapp/prod/db/password" \
  --with-decryption \
  --query "Parameter.Value" --output text)

# Khởi động ứng dụng với config từ Parameter Store
systemctl start myapp
```

### CodePipeline / CodeBuild

```yaml
# buildspec.yml — lấy secrets trong CI/CD pipeline
version: 0.2

env:
  parameter-store:
    DB_HOST: "/myapp/prod/db/host"
    DB_PASSWORD: "/myapp/prod/db/password"
    # CodeBuild tự động lấy và inject vào environment

phases:
  build:
    commands:
      - echo "Kết nối tới $DB_HOST"
      # DB_PASSWORD tự động được inject, không cần gọi API thủ công
      - ./run-integration-tests.sh
```

---

## Giám Sát Và Audit

### CloudTrail Events Quan Trọng

| Event | Ý Nghĩa |
|---|---|
| `ssm:GetParameter` | Ai đó đọc parameter |
| `ssm:PutParameter` | Tạo/cập nhật parameter |
| `ssm:DeleteParameter` | Xóa parameter |
| `ssm:GetParametersByPath` | Đọc cả path |
| `ssm:LabelParameterVersion` | Đặt/thay đổi label |

### EventBridge Rule Cho Parameter Policies

```json
{
  "source": ["aws.ssm"],
  "detail-type": ["Parameter Store Policy Action"],
  "detail": {
    "action-type": ["NOTIFY"],
    "parameter-name": [{"prefix": "/myapp/prod/"}]
  }
}
```

*EventBridge tự động nhận sự kiện từ Parameter Policies (Expiration, NoChange) và có thể forward đến SNS/Lambda.*

### CloudWatch Alarm — Parameter Không Tồn Tại

```bash
# Tạo alarm khi có lỗi ParameterNotFound từ ứng dụng
aws cloudwatch put-metric-alarm \
  --alarm-name "SSM-ParameterNotFound-Alarm" \
  --metric-name "4XXError" \
  --namespace "AWS/SSM" \
  --statistic Sum \
  --period 300 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions "arn:aws:sns:...:ops-alerts"
```

---

## Best Practices (Thực Hành Tốt Nhất)

### Đặt Tên Parameters

```
Quy Tắc Đặt Tên (Naming Convention):
/{app}/{environment}/{component}/{key}

Ví Dụ Tốt:
  /myapp/prod/database/host
  /myapp/prod/database/password
  /myapp/prod/redis/connection-string
  /shared/monitoring/datadog-api-key

Ví Dụ Xấu:
  myapp_prod_db_host         (không dùng dấu /)
  /myapp-prod-db-host        (không có phân cấp rõ ràng)
  /prod/myapp-db-host-v2     (không nhất quán)
```

### Phân Tách Theo Môi Trường

```
Dùng phân cấp để tách biệt hoàn toàn:
/myapp/prod/    ← chỉ prod role mới đọc được
/myapp/staging/ ← staging role + prod role
/myapp/dev/     ← mọi developer đọc được

Không dùng prefix environment trong tên:
❌ /prod-myapp-db-host  (khó kiểm soát IAM)
✅ /myapp/prod/db/host  (IAM dùng path prefix)
```

### Tránh Lưu Secret Nhạy Cảm Trong String (Không Mã Hóa)

```bash
# ❌ SAI — password không mã hóa
aws ssm put-parameter \
  --name "/myapp/prod/db/password" \
  --type "String" \     # ← Sai! Ai cũng đọc được
  --value "MyPassword"

# ✅ ĐÚNG
aws ssm put-parameter \
  --name "/myapp/prod/db/password" \
  --type "SecureString" \
  --key-id "alias/myapp-key" \
  --value "MyPassword"
```

---

## Câu Hỏi Phỏng Vấn

**Q1: Parameter Store khác Secrets Manager ở điểm nào quan trọng nhất?**

> Parameter Store hỗ trợ **phân cấp (hierarchy)** với path `/app/env/key` giúp quản lý config có tổ chức, miễn phí cho Standard tier. Secrets Manager hỗ trợ **auto-rotation native** và **cross-account sharing** tốt hơn. Chọn Parameter Store khi cần lưu config + secret đơn giản không cần rotation. Chọn Secrets Manager khi cần rotation tự động.

**Q2: SecureString dùng KMS nào mặc định?**

> Mặc định dùng **AWS managed key** có alias `aws/ssm` — miễn phí cho mã hóa SSM. Bạn có thể chỉ định CMK riêng để kiểm soát tốt hơn (audit key usage, key rotation, cross-account access). CMK riêng có chi phí $1/key/tháng + API calls.

**Q3: Giải thích cách IAM kiểm soát truy cập theo path?**

> ARN của parameter có dạng `arn:aws:ssm:region:account:parameter/path/to/key`. IAM policy dùng wildcard trên ARN: `parameter/myapp/prod/*` cho phép truy cập tất cả parameters dưới `/myapp/prod/`. Điều này cho phép tạo policy "dev role chỉ đọc `/myapp/dev/*`" mà không cần liệt kê từng parameter.

**Q4: Parameter Policies là gì? Khi nào cần?**

> Parameter Policies (chỉ Advanced tier) cho phép đặt TTL (tự động xóa sau ngày nhất định), gửi notification trước khi hết hạn, hoặc cảnh báo khi parameter không thay đổi lâu (dấu hiệu quên rotation). Dùng khi cần quản lý vòng đời secret không có Secrets Manager, hoặc khi cần temporary parameters cho automation scripts.

**Q5: Làm sao sử dụng Parameter Store trong CodeBuild pipeline mà không cần viết code?**

> Trong `buildspec.yml`, dùng section `env.parameter-store` — CodeBuild tự động lấy parameters và inject vào environment variables. Không cần code, không cần boto3. IAM role của CodeBuild cần quyền `ssm:GetParameter` và `kms:Decrypt` cho SecureString.

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
