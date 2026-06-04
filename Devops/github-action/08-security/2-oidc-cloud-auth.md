# 🔐 OIDC Cloud Authentication — Xác Thực Cloud Không Cần Long-lived Secrets

> OIDC — OpenID Connect — Giao Thức Xác Thực Mở cho phép GitHub Actions xác thực với cloud providers (AWS, GCP, Azure) mà không cần lưu long-lived credentials (thông tin xác thực tồn tại lâu dài) vào GitHub Secrets.

---

## 📚 Mục Lục

1. [Tại Sao OIDC Tốt Hơn Long-lived Credentials?](#tại-sao-oidc-tốt-hơn-long-lived-credentials)
2. [Cơ Chế Hoạt Động OIDC](#cơ-chế-hoạt-động-oidc)
3. [OIDC Với AWS](#oidc-với-aws)
4. [OIDC Với GCP](#oidc-với-gcp)
5. [OIDC Với Azure](#oidc-với-azure)
6. [OIDC Với HashiCorp Vault](#oidc-với-hashicorp-vault)
7. [Bảo Mật OIDC Claims](#bảo-mật-oidc-claims)
8. [Troubleshooting](#troubleshooting)

---

## Tại Sao OIDC Tốt Hơn Long-lived Credentials?

### Vấn Đề Với Credentials Truyền Thống

```yaml
# ❌ CÁCH CŨ — long-lived credentials lưu trong GitHub Secrets
- name: Deploy lên AWS
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}      # Tồn tại vĩnh viễn!
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }} # Rủi ro cao!
  run: aws s3 sync ./dist s3://my-bucket
```

**Rủi ro của long-lived credentials:**
- Nếu bị lộ → kẻ tấn công có quyền truy cập mãi mãi
- Phải luân phiên (rotate) định kỳ — tốn công sức
- Không biết credential được dùng ở đâu, khi nào
- Một người có quyền xem secrets có thể sao chép

### Giải Pháp OIDC

```yaml
# ✅ CÁCH MỚI — OIDC không cần lưu credentials
permissions:
  id-token: write
  contents: read

- name: Xác thực AWS qua OIDC
  uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
    aws-region: ap-southeast-1
# Token tự động hết hạn sau 15 phút — không lưu gì trong GitHub!
```

**Lợi ích OIDC:**
- Không cần lưu credentials trong GitHub Secrets
- Token tự động hết hạn (thường 15–60 phút)
- Có thể giới hạn theo repo, branch, environment
- Audit trail đầy đủ trong cloud provider logs

---

## Cơ Chế Hoạt Động OIDC

```
┌─────────────────────────────────────────────────────────────────┐
│                     OIDC Flow Chi Tiết                          │
│                                                                  │
│  1. Workflow trigger                                             │
│     ↓                                                            │
│  2. GitHub Actions phát JWT token (id-token)                    │
│     Claims trong token:                                         │
│     {                                                            │
│       "iss": "https://token.actions.githubusercontent.com",     │
│       "sub": "repo:owner/repo:ref:refs/heads/main",             │
│       "aud": "https://github.com/owner",                        │
│       "repository": "owner/repo",                               │
│       "environment": "production",                              │
│       "job_workflow_ref": "owner/repo/.github/workflows/...",   │
│       "exp": 1234567890  # hết hạn sau vài phút                │
│     }                                                            │
│     ↓                                                            │
│  3. Action (vd: aws-actions/configure-aws-credentials)          │
│     gửi JWT token lên Cloud Provider                            │
│     ↓                                                            │
│  4. Cloud Provider xác minh JWT với GitHub OIDC endpoint        │
│     https://token.actions.githubusercontent.com/.well-known/    │
│     openid-configuration                                        │
│     ↓                                                            │
│  5. Cloud Provider kiểm tra trust policy (chính sách tin tưởng) │
│     → Có khớp sub/aud/repository không?                         │
│     ↓                                                            │
│  6. Cloud Provider cấp short-lived credentials (15–60 phút)    │
│     ↓                                                            │
│  7. Workflow dùng credentials tạm này để truy cập cloud         │
└─────────────────────────────────────────────────────────────────┘
```

---

## OIDC Với AWS

### Bước 1: Tạo OIDC Identity Provider trong AWS IAM

```bash
# Dùng AWS CLI
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

Hoặc qua Terraform (Infrastructure as Code — Hạ Tầng Dưới Dạng Mã):

```hcl
resource "aws_iam_openid_connect_provider" "github_actions" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = [
    "sts.amazonaws.com",
  ]

  thumbprint_list = [
    "6938fd4d98bab03faadb97b34396831e3780aea1",
    "1c58a3a8518e8759bf075b76b750d4f2df264fcd",
  ]
}
```

### Bước 2: Tạo IAM Role Với Trust Policy

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
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": [
            "repo:my-org/my-repo:ref:refs/heads/main",
            "repo:my-org/my-repo:environment:production"
          ]
        }
      }
    }
  ]
}
```

**Giải thích Trust Policy:**
- `StringEquals aud` — chỉ chấp nhận token dành cho AWS STS
- `StringLike sub` — chỉ chấp nhận từ repo cụ thể, branch main hoặc environment production

### Bước 3: Thêm Permission Policy Cho Role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::my-production-bucket/*"
    }
  ]
}
```

### Bước 4: Workflow GitHub Actions

```yaml
name: Deploy lên AWS S3

on:
  push:
    branches: [main]

permissions:
  id-token: write   # Bắt buộc để request OIDC token
  contents: read    # Để checkout code

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production   # Giới hạn deploy chỉ từ environment này

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Configure AWS Credentials via OIDC
        uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsDeployRole
          role-session-name: GitHubActions-${{ github.run_id }}
          aws-region: ap-southeast-1

      - name: Kiểm tra identity (để debug)
        run: aws sts get-caller-identity

      - name: Deploy lên S3
        run: |
          aws s3 sync ./dist s3://my-production-bucket \
            --delete \
            --cache-control "public, max-age=31536000"

      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ABCDEFGHIJKLMNO \
            --paths "/*"
```

### OIDC Với ECS (Elastic Container Service)

```yaml
- name: Login to Amazon ECR
  uses: aws-actions/amazon-ecr-login@062b18b96a7aff071d4dc91bc00c4c1a7945b076 # v2.0.1

- name: Deploy to ECS
  uses: aws-actions/amazon-ecs-deploy-task-definition@69e0babb4c41c63f461ab5ae3f66fd7a23bbe87a # v2.3.0
  with:
    task-definition: task-definition.json
    service: my-service
    cluster: my-cluster
    wait-for-service-stability: true
```

---

## OIDC Với GCP

### Bước 1: Tạo Workload Identity Pool và Provider

```bash
# Tạo Workload Identity Pool (Bể Định Danh Khối Lượng Công Việc)
gcloud iam workload-identity-pools create "github-actions-pool" \
  --project="my-project-id" \
  --location="global" \
  --display-name="GitHub Actions Pool"

# Tạo OIDC Provider trong pool
gcloud iam workload-identity-pools providers create-oidc "github-provider" \
  --project="my-project-id" \
  --location="global" \
  --workload-identity-pool="github-actions-pool" \
  --display-name="GitHub Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.actor=assertion.actor" \
  --issuer-uri="https://token.actions.githubusercontent.com"
```

### Bước 2: Cấp Quyền Cho Service Account Từ Workload Identity

```bash
# Tạo Service Account
gcloud iam service-accounts create github-actions-sa \
  --project="my-project-id" \
  --display-name="GitHub Actions Service Account"

# Cấp quyền Workload Identity User
gcloud iam service-accounts add-iam-policy-binding \
  "github-actions-sa@my-project-id.iam.gserviceaccount.com" \
  --project="my-project-id" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-actions-pool/attribute.repository/my-org/my-repo"

# Cấp quyền truy cập GCS (Google Cloud Storage)
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --role=roles/storage.objectAdmin \
  --member="serviceAccount:github-actions-sa@my-project-id.iam.gserviceaccount.com"
```

### Bước 3: Workflow GitHub Actions

```yaml
name: Deploy lên GCP Cloud Run

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Authenticate to Google Cloud via OIDC
        uses: google-github-actions/auth@71fee32a0bb7e97b4d33d548e7d957010649d8fa # v2.1.3
        with:
          workload_identity_provider: "projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-actions-pool/providers/github-provider"
          service_account: "github-actions-sa@my-project-id.iam.gserviceaccount.com"

      - name: Set up Google Cloud SDK
        uses: google-github-actions/setup-gcloud@6189d56e4096ee891640bb02ac264be376592d6a # v2.1.2

      - name: Build và push Docker image
        run: |
          gcloud builds submit \
            --tag asia-southeast1-docker.pkg.dev/my-project-id/my-repo/app:${{ github.sha }}

      - name: Deploy lên Cloud Run
        run: |
          gcloud run deploy my-service \
            --image asia-southeast1-docker.pkg.dev/my-project-id/my-repo/app:${{ github.sha }} \
            --region asia-southeast1 \
            --platform managed
```

---

## OIDC Với Azure

### Bước 1: Tạo App Registration và Federated Credential

```bash
# Đăng ký ứng dụng (App Registration)
az ad app create --display-name "GitHub Actions"

# Lấy Application (client) ID
APP_ID=$(az ad app list --display-name "GitHub Actions" --query "[0].appId" -o tsv)

# Tạo Service Principal
az ad sp create --id $APP_ID

# Tạo Federated Identity Credential (Thông Tin Xác Thực Định Danh Liên Kết)
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "GitHubActionsBranch",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:my-org/my-repo:ref:refs/heads/main",
    "description": "GitHub Actions main branch",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

### Bước 2: Cấp Quyền Cho Service Principal

```bash
# Lấy Subscription ID
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
SP_OBJECT_ID=$(az ad sp list --display-name "GitHub Actions" --query "[0].id" -o tsv)

# Cấp quyền Contributor trên resource group
az role assignment create \
  --role "Contributor" \
  --assignee-object-id $SP_OBJECT_ID \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/my-resource-group"
```

### Bước 3: Thêm Secrets Vào GitHub Repository

```
GitHub Secrets cần thiết (không phải credentials, chỉ là IDs):
  AZURE_CLIENT_ID      = Application (client) ID
  AZURE_TENANT_ID      = Directory (tenant) ID
  AZURE_SUBSCRIPTION_ID = Subscription ID
```

### Bước 4: Workflow GitHub Actions

```yaml
name: Deploy lên Azure Container Apps

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Login to Azure via OIDC
        uses: azure/login@a457da9ea143d694b1b9c7c869ebb04ebe844ef5 # v2.3.0
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Build và push image lên ACR
        uses: azure/docker-login@15c4aadf093404726ab2ff205b2cdd33fa6d054c # v2
        with:
          login-server: myregistry.azurecr.io
          username: ${{ secrets.AZURE_CLIENT_ID }}
          password: ${{ secrets.AZURE_CLIENT_SECRET }}  # Hoặc dùng OIDC cho ACR

      - name: Deploy lên Azure Container Apps
        uses: azure/container-apps-deploy-action@5dfe7e41f5b85d8fe65e3aaafc1cf29440bf5ccd # v2
        with:
          resourceGroup: my-resource-group
          containerAppName: my-container-app
          imageToDeploy: myregistry.azurecr.io/app:${{ github.sha }}
```

---

## OIDC Với HashiCorp Vault

```yaml
name: Lấy Secrets Từ Vault

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Import Secrets từ HashiCorp Vault
        uses: hashicorp/vault-action@d1720f055e0635fd932a1d2a48f87a666a57906c # v3.0.0
        with:
          url: https://vault.my-company.com
          method: jwt
          role: github-actions-role
          secrets: |
            secret/data/production/database password | DB_PASSWORD ;
            secret/data/production/api key | API_KEY

      - name: Sử dụng secrets
        run: echo "Deploying with secrets..."
        env:
          DATABASE_PASSWORD: ${{ env.DB_PASSWORD }}
```

---

## Bảo Mật OIDC Claims

### Subject (sub) Claim — Định Danh Chủ Thể

GitHub tạo `sub` claim dựa trên ngữ cảnh workflow. Luôn giới hạn chặt trong trust policy:

```
# Patterns phổ biến cho sub claim

# Branch cụ thể
repo:org/repo:ref:refs/heads/main

# Pull Request
repo:org/repo:pull_request

# Environment cụ thể (khuyến nghị cho production)
repo:org/repo:environment:production

# Tag
repo:org/repo:ref:refs/tags/v1.0.0

# Tất cả (quá rộng — không nên dùng cho production)
repo:org/repo:*
```

### Giới Hạn Trust Policy Theo Environment

```json
{
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:sub":
        "repo:my-org/my-repo:environment:production"
    }
  }
}
```

**Lợi ích:** Chỉ workflows chạy trong GitHub Environment "production" (với required reviewers) mới có thể assume role production.

### Custom Claims Cho Kiểm Soát Chi Tiết Hơn

```yaml
# Workflow có thể request custom claims qua audience
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
    aws-region: ap-southeast-1
    role-duration-seconds: 900   # 15 phút — giảm cửa sổ tấn công
    role-session-name: deploy-${{ github.run_id }}-${{ github.run_attempt }}
```

---

## Troubleshooting

### Lỗi: "Unable to create credentials"

```yaml
# Kiểm tra: permissions block có id-token: write không?
permissions:
  id-token: write   # ← Phải có dòng này!
  contents: read
```

### Lỗi: "Not authorized to perform sts:AssumeRoleWithWebIdentity"

```bash
# Kiểm tra trust policy của IAM role
aws iam get-role --role-name GitHubActionsRole \
  --query 'Role.AssumeRolePolicyDocument'

# Kiểm tra sub claim trong OIDC token
# Thêm step debug vào workflow:
- name: Debug OIDC token
  run: |
    TOKEN=$(curl -s -H "Authorization: Bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
      "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=sts.amazonaws.com" \
      | jq -r '.value')
    echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | jq .
```

### Lỗi: "Token has expired"

- Role session duration quá ngắn so với thời gian deploy
- Tăng `role-duration-seconds` (tối đa 3600 = 1 giờ hoặc theo max session duration của role)

---

## 📊 So Sánh OIDC Với Các Cloud Providers

| Tiêu Chí | AWS | GCP | Azure |
|---|---|---|---|
| **Tên cơ chế** | IAM OIDC Identity Provider | Workload Identity Federation | Federated Identity Credential |
| **Thời gian setup** | ~10 phút | ~15 phút | ~15 phút |
| **Terraform support** | `aws_iam_openid_connect_provider` | `google_iam_workload_identity_pool_provider` | `azuread_application_federated_identity_credential` |
| **Token duration mặc định** | 1 giờ | 1 giờ | 1 giờ |

---

## 🔗 Xem Thêm

- [1-permissions.md](./1-permissions.md) — Permissions block và GITHUB_TOKEN
- [04-secrets-variables/3-oidc.md](../04-secrets-variables/3-oidc.md) — OIDC chi tiết hơn
- [6-security-hardening.md](./6-security-hardening.md) — Checklist bảo mật

---

**Cập Nhật Lần Cuối:** 2026-05-12
