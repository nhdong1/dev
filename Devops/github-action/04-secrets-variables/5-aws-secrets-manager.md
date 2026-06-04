# ☁️ AWS Secrets Manager — Tích Hợp Quản Lý Bí Mật AWS

> AWS Secrets Manager là dịch vụ quản lý secrets của Amazon Web Services — hỗ trợ automatic rotation (tự động xoay vòng), tích hợp sẵn với RDS, Lambda và nhiều dịch vụ AWS khác. Kết hợp với GitHub Actions qua OIDC cho quy trình bảo mật toàn diện.

---

## 📚 Mục Lục

1. [AWS Secrets Manager Là Gì?](#aws-secrets-manager-là-gì)
2. [Kiến Trúc Tích Hợp](#kiến-trúc-tích-hợp)
3. [Setup OIDC Cho AWS Secrets Manager](#setup-oidc-cho-aws-secrets-manager)
4. [Đọc Secrets Trong Workflow](#đọc-secrets-trong-workflow)
5. [Automatic Rotation — Xoay Vòng Tự Động](#automatic-rotation--xoay-vòng-tự-động)
6. [So Sánh AWS SSM Parameter Store vs Secrets Manager](#so-sánh-aws-ssm-parameter-store-vs-secrets-manager)
7. [Patterns Thực Tế](#patterns-thực-tế)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## AWS Secrets Manager Là Gì?

AWS Secrets Manager (Quản Lý Bí Mật AWS) là managed service (dịch vụ được quản lý) cho phép:
- Lưu trữ và quản lý secrets an toàn (encrypted bằng KMS — Key Management Service)
- **Automatic rotation** (tự động xoay vòng) cho database passwords và API keys
- Tích hợp sẵn với RDS (Relational Database Service), Redshift, DocumentDB
- Fine-grained IAM permissions cho từng secret
- Versioning và staging labels (AWSCURRENT, AWSPREVIOUS, AWSPENDING)

### Khi Nào Dùng AWS Secrets Manager?

```
Dùng AWS Secrets Manager khi:
✅ Stack chủ yếu dùng AWS services
✅ Cần automatic rotation cho database credentials
✅ Cần tích hợp sẵn với RDS/Redshift
✅ Team đã quen với AWS IAM policies
✅ Cần versioning và rollback credentials

Dùng GitHub Secrets khi:
✅ Dự án nhỏ, chỉ cần secrets đơn giản
✅ Không có AWS infrastructure
✅ Muốn giải pháp đơn giản nhất
```

---

## Kiến Trúc Tích Hợp

```
┌──────────────────────────────────────────────────────────────────┐
│          GitHub Actions + AWS Secrets Manager                    │
│                                                                  │
│  GitHub Runner                                                   │
│  ┌──────────────────┐                                           │
│  │  Workflow Job    │                                           │
│  │                  │                                           │
│  │  1. Authenticate │──── OIDC JWT ───> AWS STS                │
│  │     via OIDC     │                       │                   │
│  │                  │<── Temp Creds (1h) ───┘                   │
│  │                  │                                           │
│  │  2. Read Secrets │──── GetSecretValue ─> AWS Secrets Manager │
│  │                  │<── Secret Value ──────────────────────────│
│  │                  │                                           │
│  │  3. Use Secrets  │   (Secrets masked in logs)               │
│  │     in Build/    │                                           │
│  │     Deploy Steps │                                           │
│  └──────────────────┘                                           │
│                                                                  │
│  AWS Secrets Manager                                             │
│  ┌──────────────────────────────────────────┐                   │
│  │  myapp/production/database               │                   │
│  │    host: db.prod.example.com             │                   │
│  │    port: 5432                            │                   │
│  │    username: app_user                    │                   │
│  │    password: *** (auto-rotated weekly)   │                   │
│  │                                          │                   │
│  │  myapp/production/api-keys               │                   │
│  │    stripe_key: sk_live_***               │                   │
│  │    sendgrid_key: SG.***                  │                   │
│  └──────────────────────────────────────────┘                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Setup OIDC Cho AWS Secrets Manager

### Bước 1: Tạo OIDC Provider

```bash
# Tạo OIDC Identity Provider (Nhà Cung Cấp Danh Tính OIDC)
aws iam create-open-id-connect-provider \
  --url "https://token.actions.githubusercontent.com" \
  --client-id-list "sts.amazonaws.com" \
  --thumbprint-list "6938fd4d98bab03faadb97b34396831e3780aea1"
```

### Bước 2: Tạo IAM Role Với Quyền Đọc Secrets

**Trust Policy (Chính Sách Tin Tưởng):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:environment:production"
        }
      }
    }
  ]
}
```

**Permission Policy (Chính Sách Quyền):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": [
        "arn:aws:secretsmanager:us-east-1:123456789012:secret:myapp/production/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "kms:Decrypt",
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/YOUR-KMS-KEY-ID",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "secretsmanager.us-east-1.amazonaws.com"
        }
      }
    }
  ]
}
```

```bash
# Tạo role
aws iam create-role \
  --role-name GitHubActionsSecretsReader \
  --assume-role-policy-document file://trust-policy.json

# Đính kèm permission policy
aws iam put-role-policy \
  --role-name GitHubActionsSecretsReader \
  --policy-name SecretsManagerReadAccess \
  --policy-document file://permission-policy.json
```

### Bước 3: Tạo Secrets Trong AWS Secrets Manager

```bash
# Tạo JSON secret (nhiều key-value trong một secret)
aws secretsmanager create-secret \
  --name "myapp/production/database" \
  --description "Production database credentials" \
  --secret-string '{
    "host": "db.prod.example.com",
    "port": "5432",
    "dbname": "myapp_prod",
    "username": "app_user",
    "password": "initial-password"
  }'

# Tạo string secret đơn
aws secretsmanager create-secret \
  --name "myapp/production/api-keys" \
  --secret-string '{
    "stripe_secret_key": "sk_live_...",
    "sendgrid_api_key": "SG...",
    "datadog_api_key": "abc123..."
  }'

# Update secret value
aws secretsmanager update-secret \
  --secret-id "myapp/production/database" \
  --secret-string '{"password": "new-password"}'
```

---

## Đọc Secrets Trong Workflow

### Cách 1: Dùng AWS CLI

```yaml
name: Deploy with AWS Secrets Manager

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsSecretsReader
          aws-region: us-east-1

      - name: Read secrets from AWS Secrets Manager
        run: |
          # Đọc JSON secret và extract từng field
          SECRET=$(aws secretsmanager get-secret-value \
            --secret-id "myapp/production/database" \
            --query SecretString \
            --output text)
          
          DB_HOST=$(echo $SECRET | jq -r '.host')
          DB_USER=$(echo $SECRET | jq -r '.username')
          DB_PASS=$(echo $SECRET | jq -r '.password')
          DB_NAME=$(echo $SECRET | jq -r '.dbname')
          
          # Mask values trong logs
          echo "::add-mask::$DB_PASS"
          
          # Set environment variables
          echo "DB_HOST=$DB_HOST" >> $GITHUB_ENV
          echo "DB_USER=$DB_USER" >> $GITHUB_ENV
          echo "DB_PASS=$DB_PASS" >> $GITHUB_ENV
          echo "DB_NAME=$DB_NAME" >> $GITHUB_ENV

      - name: Run database migration
        run: |
          DATABASE_URL="postgresql://${DB_USER}:${DB_PASS}@${DB_HOST}:5432/${DB_NAME}"
          npm run db:migrate
        env:
          DATABASE_URL: "postgresql://${{ env.DB_USER }}:${{ env.DB_PASS }}@${{ env.DB_HOST }}:5432/${{ env.DB_NAME }}"
```

### Cách 2: Dùng `aws-actions/aws-secretsmanager-get-secrets` Action

```yaml
name: Deploy using Secrets Manager Action

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
          aws-region: us-east-1

      - name: Get secrets from AWS Secrets Manager
        uses: aws-actions/aws-secretsmanager-get-secrets@v2
        with:
          secret-ids: |
            DB, myapp/production/database
            APIKEYS, myapp/production/api-keys
          # Tự động prefix các keys với tên secret
          # DB_HOST, DB_PASSWORD, DB_USERNAME...
          # APIKEYS_STRIPE_SECRET_KEY, APIKEYS_SENDGRID_API_KEY...
          parse-json-secrets: true

      - name: Deploy application
        run: |
          echo "Deploying with DB host: $DB_HOST"
          # DB_PASSWORD, DB_HOST, etc. đã được set
          ./deploy.sh
```

### Cách 3: Helper Script Linh Hoạt

```bash
#!/bin/bash
# scripts/load-aws-secrets.sh
# Load tất cả key-value pairs từ một AWS Secrets Manager secret

SECRET_NAME=$1
REGION=${2:-us-east-1}

echo "Loading secrets from: $SECRET_NAME"

# Đọc secret
SECRET_VALUE=$(aws secretsmanager get-secret-value \
  --secret-id "$SECRET_NAME" \
  --region "$REGION" \
  --query SecretString \
  --output text)

# Parse JSON và export từng key-value
echo "$SECRET_VALUE" | jq -r 'to_entries[] | "\(.key)=\(.value)"' | while IFS='=' read -r key value; do
  # Chuyển key về uppercase
  ENV_KEY=$(echo "$key" | tr '[:lower:]' '[:upper:]' | tr '-' '_')
  
  # Mask value trong GitHub Actions logs
  echo "::add-mask::$value"
  
  # Export vào GitHub ENV
  echo "${ENV_KEY}=${value}" >> $GITHUB_ENV
  echo "Loaded: $ENV_KEY"
done
```

```yaml
- name: Load production secrets
  run: |
    chmod +x ./scripts/load-aws-secrets.sh
    ./scripts/load-aws-secrets.sh "myapp/production/database"
    ./scripts/load-aws-secrets.sh "myapp/production/api-keys"
```

---

## Automatic Rotation — Xoay Vòng Tự Động

AWS Secrets Manager hỗ trợ automatic rotation cho nhiều loại secrets, đặc biệt là database credentials.

### Cấu Hình Rotation Cho RDS

```bash
# Bật automatic rotation cho RDS password
aws secretsmanager rotate-secret \
  --secret-id "myapp/production/database" \
  --rotation-lambda-arn "arn:aws:lambda:us-east-1:123456789012:function:SecretsManagerRotator" \
  --rotation-rules '{
    "AutomaticallyAfterDays": 30
  }'
```

### Rotation Với Lambda Function

```python
# Lambda function (tự động tạo bởi AWS khi chọn RDS rotation)
import boto3
import json

def lambda_handler(event, context):
    arn = event['SecretId']
    token = event['ClientRequestToken']
    step = event['Step']
    
    client = boto3.client('secretsmanager')
    
    if step == "createSecret":
        # Tạo password mới
        create_new_password(client, arn, token)
    elif step == "setSecret":
        # Cập nhật password trong RDS
        set_password_in_db(client, arn, token)
    elif step == "testSecret":
        # Kiểm tra kết nối với password mới
        test_connection(client, arn, token)
    elif step == "finishSecret":
        # Đánh dấu secret mới là AWSCURRENT
        client.update_secret_version_stage(
            SecretId=arn,
            VersionStage='AWSCURRENT',
            MoveToVersionId=token,
            RemoveFromVersionId=get_current_version(client, arn)
        )
```

### Staging Labels — Nhãn Phiên Bản

```bash
# AWS Secrets Manager dùng staging labels để quản lý versions
# AWSCURRENT  → Phiên bản hiện tại đang dùng
# AWSPREVIOUS → Phiên bản trước (dùng để rollback)
# AWSPENDING  → Phiên bản đang được rotation

# Đọc version cụ thể
aws secretsmanager get-secret-value \
  --secret-id "myapp/production/database" \
  --version-stage "AWSCURRENT"

# Rollback: set AWSPREVIOUS thành AWSCURRENT
aws secretsmanager update-secret-version-stage \
  --secret-id "myapp/production/database" \
  --version-stage "AWSCURRENT" \
  --move-to-version-id "PREVIOUS_VERSION_ID"
```

---

## So Sánh AWS SSM Parameter Store vs Secrets Manager

AWS có hai dịch vụ cho secrets/configs:

| Tiêu Chí | SSM Parameter Store | Secrets Manager |
|---|---|---|
| **Chi phí** | Free tier có (Standard params) | $0.40/secret/tháng |
| **Automatic rotation** | ❌ (cần tự làm) | ✅ Tích hợp sẵn |
| **Cross-account sharing** | Hạn chế | ✅ Dễ dàng |
| **Versioning** | Cơ bản | ✅ Đầy đủ với labels |
| **Max size** | 4KB (Standard), 8KB (Advanced) | 65KB |
| **RDS integration** | ❌ | ✅ Native |
| **Audit** | CloudTrail | CloudTrail + native audit |
| **Khi nào dùng** | Config + non-sensitive params | Credentials cần rotation |

### Dùng SSM Parameter Store Trong Workflow

```yaml
- name: Read from SSM Parameter Store
  run: |
    # SecureString parameters (mã hóa)
    DB_URL=$(aws ssm get-parameter \
      --name "/myapp/production/db-url" \
      --with-decryption \
      --query Parameter.Value \
      --output text)
    
    # Plain String parameters
    APP_VERSION=$(aws ssm get-parameter \
      --name "/myapp/version" \
      --query Parameter.Value \
      --output text)
    
    echo "::add-mask::$DB_URL"
    echo "DB_URL=$DB_URL" >> $GITHUB_ENV
    echo "APP_VERSION=$APP_VERSION" >> $GITHUB_ENV
```

---

## Patterns Thực Tế

### Pattern 1: Multi-Environment Deployment

```yaml
name: Multi-Environment Deploy

on:
  push:
    branches:
      - main       # → production
      - develop    # → staging

permissions:
  id-token: write
  contents: read

jobs:
  determine-env:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.env.outputs.environment }}
      role-arn: ${{ steps.env.outputs.role-arn }}
      secret-prefix: ${{ steps.env.outputs.secret-prefix }}
    steps:
      - id: env
        run: |
          if [ "$GITHUB_REF_NAME" == "main" ]; then
            echo "environment=production" >> $GITHUB_OUTPUT
            echo "role-arn=${{ vars.AWS_PROD_ROLE_ARN }}" >> $GITHUB_OUTPUT
            echo "secret-prefix=myapp/production" >> $GITHUB_OUTPUT
          else
            echo "environment=staging" >> $GITHUB_OUTPUT
            echo "role-arn=${{ vars.AWS_STAGING_ROLE_ARN }}" >> $GITHUB_OUTPUT
            echo "secret-prefix=myapp/staging" >> $GITHUB_OUTPUT
          fi

  deploy:
    needs: determine-env
    runs-on: ubuntu-latest
    environment: ${{ needs.determine-env.outputs.environment }}
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ needs.determine-env.outputs.role-arn }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Load environment secrets
        run: |
          SECRET_PREFIX="${{ needs.determine-env.outputs.secret-prefix }}"
          
          DB_SECRET=$(aws secretsmanager get-secret-value \
            --secret-id "${SECRET_PREFIX}/database" \
            --query SecretString --output text)
          
          echo "::add-mask::$(echo $DB_SECRET | jq -r '.password')"
          echo "DB_PASSWORD=$(echo $DB_SECRET | jq -r '.password')" >> $GITHUB_ENV
          echo "DB_HOST=$(echo $DB_SECRET | jq -r '.host')" >> $GITHUB_ENV

      - name: Deploy
        run: ./deploy.sh ${{ needs.determine-env.outputs.environment }}
```

### Pattern 2: Build và Push Docker Image Với ECR Credentials

```yaml
name: Build and Push to ECR

permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_BUILD_ROLE_ARN }}
          aws-region: us-east-1

      - name: Load build-time secrets
        uses: aws-actions/aws-secretsmanager-get-secrets@v2
        with:
          secret-ids: |
            NPM_TOKEN, myapp/build/npm-token
          parse-json-secrets: true

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push Docker image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build \
            --build-arg NPM_TOKEN_VALUE="${NPM_TOKEN_NPM_TOKEN}" \
            -t $ECR_REGISTRY/myapp:$IMAGE_TAG \
            .
          docker push $ECR_REGISTRY/myapp:$IMAGE_TAG
```

### Pattern 3: Secret Validation Trước Deploy

```yaml
- name: Validate required secrets exist
  run: |
    REQUIRED_SECRETS=(
      "myapp/production/database"
      "myapp/production/api-keys"
      "myapp/production/tls-cert"
    )
    
    MISSING=()
    for SECRET in "${REQUIRED_SECRETS[@]}"; do
      if ! aws secretsmanager describe-secret --secret-id "$SECRET" > /dev/null 2>&1; then
        MISSING+=("$SECRET")
      fi
    done
    
    if [ ${#MISSING[@]} -gt 0 ]; then
      echo "ERROR: Missing required secrets:"
      printf '  - %s\n' "${MISSING[@]}"
      exit 1
    fi
    
    echo "All required secrets are present ✓"
```

---

## Best Practices

### 1. Tổ Chức Secret Names Theo Hierarchy

```
/organization/application/environment/category
Ví dụ:
  myorg/myapp/production/database
  myorg/myapp/production/api-keys
  myorg/myapp/staging/database
  myorg/shared/datadog-api-key
```

### 2. KMS Customer Managed Key (CMK — Khóa Mã Hóa Tùy Chỉnh)

```bash
# Tạo CMK cho secrets
aws kms create-key \
  --description "GitHub Actions Secrets Encryption Key" \
  --key-usage ENCRYPT_DECRYPT

# Tạo secret với CMK
aws secretsmanager create-secret \
  --name "myapp/production/database" \
  --kms-key-id "arn:aws:kms:us-east-1:123456789:key/KEY-ID" \
  --secret-string '{"password": "my-password"}'
```

### 3. Resource-Based Policy Cho Cross-Account

```json
// Cho phép account khác đọc secret
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::OTHER-ACCOUNT-ID:root"
      },
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "*"
    }
  ]
}
```

### 4. CloudTrail Alerts Cho Unauthorized Access

```bash
# Tạo CloudWatch alarm cho unauthorized access
aws cloudwatch put-metric-alarm \
  --alarm-name "SecretsManagerUnauthorizedAccess" \
  --alarm-description "Alert on unauthorized secrets access" \
  --metric-name "ErrorCount" \
  --namespace "AWS/SecretsManager" \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789:security-alerts"
```

---

## Câu Hỏi Phỏng Vấn

### Q: Phân biệt AWS Secrets Manager và SSM Parameter Store?

**Trả lời:**

| | Secrets Manager | SSM Parameter Store |
|---|---|---|
| **Dùng khi** | Credentials cần rotation, database passwords | App configuration, non-sensitive configs |
| **Automatic rotation** | ✅ Native với RDS, Redshift | ❌ (cần tự implement) |
| **Chi phí** | $0.40/secret/tháng | Free (Standard tier) |
| **Best for** | Database creds, API keys có lifecycle | Configs, feature flags, URLs |

### Q: Tại sao dùng OIDC + Secrets Manager thay vì lưu AWS credentials trong GitHub Secrets?

**Trả lời:**
OIDC hoàn toàn loại bỏ long-lived credentials. Với OIDC:
- GitHub tạo JWT token mỗi job run → AWS cấp temp credentials (1h)
- Không có AWS key nào stored anywhere
- Nếu GitHub bị compromise → không có credentials nào bị lộ
- Kết hợp Secrets Manager: cả authentication AND secrets đều short-lived

Đây là "passwordless" — không password nào cần quản lý manually.

### Q: Cách tổ chức secrets trong AWS Secrets Manager cho multi-environment setup?

**Gợi ý tốt:**
```
Dùng path-based naming với IAM conditions:
  myapp/production/* → chỉ production role được đọc
  myapp/staging/*    → chỉ staging role được đọc

IAM Policy:
{
  "Action": "secretsmanager:GetSecretValue",
  "Resource": "arn:aws:secretsmanager:*:*:secret:myapp/production/*"
}
```

---

**Điều Hướng:**
- ← [4-vault-integration.md](4-vault-integration.md)
- → [05-reusable/README.md](../05-reusable/README.md)
- [Lên INDEX](../INDEX.md)

**Cập Nhật Lần Cuối:** 2026-05-11
