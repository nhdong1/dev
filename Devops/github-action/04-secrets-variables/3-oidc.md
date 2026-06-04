# 🔓 OIDC — OpenID Connect — Xác Thực Không Cần Long-lived Credentials

> OIDC (OpenID Connect — Giao Thức Kết Nối Danh Tính Mở) là phương pháp xác thực hiện đại nhất cho GitHub Actions với cloud providers. Thay vì lưu AWS/GCP/Azure credentials trong GitHub Secrets, workflows nhận token ngắn hạn tự động hết hạn sau khi job kết thúc.

---

## 📚 Mục Lục

1. [Tại Sao Cần OIDC?](#tại-sao-cần-oidc)
2. [Cơ Chế Hoạt Động](#cơ-chế-hoạt-động)
3. [OIDC với AWS](#oidc-với-aws)
4. [OIDC với GCP](#oidc-với-gcp)
5. [OIDC với Azure](#oidc-với-azure)
6. [Kiểm Soát Truy Cập Với Conditions](#kiểm-soát-truy-cập-với-conditions)
7. [Security Best Practices](#security-best-practices)
8. [Troubleshooting](#troubleshooting)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần OIDC?

### Vấn Đề Với Long-lived Credentials

```
Cách cũ (KHÔNG nên dùng):
┌─────────────────────────────────────────────────────────┐
│  1. Tạo AWS Access Key (không hết hạn)                  │
│  2. Lưu vào GitHub Secrets                              │
│  3. Workflow dùng key này mãi mãi                       │
│                                                         │
│  Rủi ro:                                                │
│  ❌ Nếu GitHub bị breach → key bị lộ mãi mãi           │
│  ❌ Quên rotate → key cũ vẫn hoạt động vô thời hạn     │
│  ❌ Khó audit: ai, khi nào dùng key này?                │
│  ❌ Key bị lộ trong PR logs → tồn tại mãi trong history │
└─────────────────────────────────────────────────────────┘

Cách mới với OIDC:
┌─────────────────────────────────────────────────────────┐
│  1. GitHub tạo short-lived JWT token cho mỗi job        │
│  2. Cloud provider exchange token → temporary creds     │
│  3. Credentials tự hết hạn sau job kết thúc             │
│                                                         │
│  Lợi ích:                                               │
│  ✅ Không có long-lived credentials nào bị lộ           │
│  ✅ Không cần rotation thủ công                         │
│  ✅ Audit trail chi tiết (ai, từ workflow nào, khi nào) │
│  ✅ Giới hạn được chính xác repo/branch có thể access   │
└─────────────────────────────────────────────────────────┘
```

---

## Cơ Chế Hoạt Động

### Luồng OIDC Đầy Đủ

```
┌─────────────────────────────────────────────────────────────────┐
│                     OIDC Auth Flow                              │
│                                                                 │
│  GitHub Runner                  GitHub OIDC Provider            │
│       │                               │                        │
│       │── 1. Request OIDC token ─────>│                        │
│       │                               │                        │
│       │<── 2. Return JWT token ───────│                        │
│       │      (có claims: repo, branch,│                        │
│       │       workflow, actor, etc.)  │                        │
│       │                                                        │
│       │              AWS STS (Security Token Service)          │
│       │                               │                        │
│       │── 3. AssumeRoleWithWebIdentity ─────────────>│         │
│       │      (gửi JWT token)          │                        │
│       │                               │                        │
│       │<── 4. Temporary Credentials ──│                        │
│       │      (AccessKeyId,            │                        │
│       │       SecretAccessKey,        │                        │
│       │       SessionToken — ~1h TTL) │                        │
│       │                                                        │
│       │── 5. Dùng temporary creds để deploy/access AWS ──────> │
└─────────────────────────────────────────────────────────────────┘
```

### JWT Claims (Các Trường Trong Token)

Khi GitHub tạo OIDC token, token chứa các claims (thông tin xác nhận):

```json
{
  "jti": "unique-token-id",
  "sub": "repo:owner/repo:ref:refs/heads/main",
  "aud": "https://github.com/owner",
  "ref": "refs/heads/main",
  "sha": "abc123def456",
  "repository": "owner/repo",
  "repository_owner": "owner",
  "repository_owner_id": "12345",
  "run_id": "1234567890",
  "run_number": "42",
  "run_attempt": "1",
  "workflow": "Deploy to Production",
  "workflow_ref": "owner/repo/.github/workflows/deploy.yml@refs/heads/main",
  "head_ref": "",
  "base_ref": "",
  "event_name": "push",
  "actor": "username",
  "actor_id": "67890",
  "environment": "production",
  "environment_node_id": "EN_xxx",
  "job_workflow_ref": "owner/repo/.github/workflows/deploy.yml@refs/heads/main",
  "iss": "https://token.actions.githubusercontent.com",
  "nbf": 1704067200,
  "exp": 1704070800,
  "iat": 1704067200
}
```

**Claims quan trọng nhất để restrict access:**
- `sub` — Subject: định danh đầy đủ (repo + branch)
- `repository` — Tên repository
- `ref` — Git ref (branch/tag)
- `environment` — GitHub Environment name
- `actor` — Người trigger workflow

---

## OIDC với AWS

### Bước 1: Tạo Identity Provider Trong AWS

```bash
# Dùng AWS CLI
aws iam create-open-id-connect-provider \
  --url "https://token.actions.githubusercontent.com" \
  --client-id-list "sts.amazonaws.com" \
  --thumbprint-list "6938fd4d98bab03faadb97b34396831e3780aea1"
```

Hoặc qua AWS Console:
```
IAM → Identity providers → Add provider
Provider type: OpenID Connect
Provider URL: https://token.actions.githubusercontent.com
Audience: sts.amazonaws.com
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
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:owner/repo:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

**Trust Policy nâng cao — cho phép nhiều repos và branches:**

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
          "token.actions.githubusercontent.com:sub": "repo:my-org/*:*"
        }
      }
    }
  ]
}
```

### Bước 3: Workflow Dùng OIDC

```yaml
name: Deploy to AWS

on:
  push:
    branches: [main]

permissions:
  id-token: write   # BẮT BUỘC — cho phép tạo OIDC token
  contents: read    # Cần để checkout code

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          role-session-name: GitHubActions-${{ github.run_id }}
          aws-region: us-east-1
          # Không cần aws-access-key-id hay aws-secret-access-key!

      - name: Verify AWS identity
        run: aws sts get-caller-identity

      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster production \
            --service my-service \
            --force-new-deployment
```

### Ví Dụ: Deploy Multi-Stage

```yaml
name: Multi-Stage AWS Deploy

on:
  push:
    branches: [main, develop]

permissions:
  id-token: write
  contents: read

jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/StagingDeployRole
          aws-region: ap-southeast-1

      - name: Deploy to staging EKS
        run: |
          aws eks update-kubeconfig --name staging-cluster
          kubectl apply -f k8s/staging/

  deploy-production:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production  # Có required reviewers
    steps:
      - uses: actions/checkout@v4
      
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/ProductionDeployRole
          aws-region: us-east-1

      - name: Deploy to production EKS
        run: |
          aws eks update-kubeconfig --name production-cluster
          kubectl apply -f k8s/production/
```

---

## OIDC với GCP

### Bước 1: Tạo Workload Identity Pool

```bash
# Tạo Workload Identity Pool (Nhóm Danh Tính Khối Lượng Công Việc)
gcloud iam workload-identity-pools create "github-actions-pool" \
  --project="my-project" \
  --location="global" \
  --display-name="GitHub Actions Pool"

# Tạo OIDC Provider trong Pool
gcloud iam workload-identity-pools providers create-oidc "github-actions-provider" \
  --project="my-project" \
  --location="global" \
  --workload-identity-pool="github-actions-pool" \
  --display-name="GitHub Actions Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.actor=assertion.actor,attribute.aud=assertion.aud" \
  --issuer-uri="https://token.actions.githubusercontent.com"
```

### Bước 2: Bind Service Account

```bash
# Cho phép GitHub repo cụ thể impersonate (mạo danh) service account
gcloud iam service-accounts add-iam-policy-binding \
  "my-service-account@my-project.iam.gserviceaccount.com" \
  --project="my-project" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-actions-pool/attribute.repository/owner/repo"
```

### Bước 3: Workflow Dùng OIDC với GCP

```yaml
name: Deploy to GCP

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
      - uses: actions/checkout@v4

      - name: Authenticate to GCP via OIDC
        id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: "projects/123456789/locations/global/workloadIdentityPools/github-actions-pool/providers/github-actions-provider"
          service_account: "my-service-account@my-project.iam.gserviceaccount.com"

      - name: Setup gcloud CLI
        uses: google-github-actions/setup-gcloud@v2

      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy my-service \
            --image gcr.io/my-project/my-app:${{ github.sha }} \
            --region us-central1 \
            --platform managed
```

---

## OIDC với Azure

### Bước 1: Đăng Ký App và Tạo Federated Credential

```bash
# Tạo App Registration (Đăng Ký Ứng Dụng)
APP_ID=$(az ad app create --display-name "GitHub Actions" --query appId -o tsv)

# Tạo Service Principal
az ad sp create --id $APP_ID

# Gán Role
az role assignment create \
  --assignee $APP_ID \
  --role Contributor \
  --scope /subscriptions/SUBSCRIPTION_ID/resourceGroups/my-resource-group
```

```json
// Federated Identity Credential (Xác Thực Danh Tính Liên Kết)
// az ad app federated-credential create --id $APP_ID --parameters credential.json
{
  "name": "github-actions-main",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:owner/repo:ref:refs/heads/main",
  "description": "GitHub Actions main branch",
  "audiences": ["api://AzureADTokenExchange"]
}
```

### Bước 2: Workflow Dùng OIDC với Azure

```yaml
name: Deploy to Azure

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
      - uses: actions/checkout@v4

      - name: Azure Login via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          # Không cần client secret hay certificate!

      - name: Deploy to Azure Container Apps
        uses: azure/container-apps-deploy-action@v1
        with:
          containerAppName: my-app
          resourceGroup: my-resource-group
          imageToDeploy: myregistry.azurecr.io/my-app:${{ github.sha }}
```

---

## Kiểm Soát Truy Cập Với Conditions

OIDC cho phép kiểm soát chi tiết ai có thể access gì dựa trên JWT claims.

### Restrict Theo Branch

```json
// AWS Trust Policy — chỉ main branch được deploy production
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:sub": 
      "repo:owner/repo:ref:refs/heads/main"
  }
}
```

### Restrict Theo Environment

```json
// AWS Trust Policy — chỉ environment "production" được dùng role này
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:sub": 
      "repo:owner/repo:environment:production"
  }
}
```

### Restrict Theo Tag

```json
// Chỉ tag release mới có thể deploy
"Condition": {
  "StringLike": {
    "token.actions.githubusercontent.com:sub": 
      "repo:owner/repo:ref:refs/tags/v*"
  }
}
```

### Restrict Theo Organization

```json
// Mọi repo trong organization đều có thể dùng
"Condition": {
  "StringLike": {
    "token.actions.githubusercontent.com:sub": 
      "repo:my-org/*"
  }
}
```

### Ví Dụ Phức Tạp: Multi-Condition

```json
{
  "Effect": "Allow",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
      "token.actions.githubusercontent.com:sub": 
        "repo:my-org/my-repo:environment:production"
    },
    "StringLike": {
      "token.actions.githubusercontent.com:ref": 
        "refs/tags/v*"
    }
  }
}
```

---

## Security Best Practices

### 1. Luôn Đặt `permissions: id-token: write` Tối Thiểu

```yaml
# ✅ Chỉ mở quyền cần thiết
permissions:
  id-token: write
  contents: read

# ❌ Tránh mở rộng permissions không cần thiết
permissions: write-all  # Quá rộng
```

### 2. Dùng Subject Conditions Chặt Chẽ

```json
// ✅ TỐT — Chỉ production environment trên main branch
"sub": "repo:owner/repo:environment:production"

// ⚠️ CẦN THẬN — Cho phép tất cả branches
"sub": "repo:owner/repo:*"

// ❌ TỆ — Wildcard quá rộng
"sub": "*"
```

### 3. Kết Hợp Với GitHub Environments

```yaml
jobs:
  deploy:
    environment: production  # Kích hoạt protection rules + environment secrets
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          # Role này chỉ tin tưởng environment "production"
          role-to-assume: ${{ secrets.AWS_PROD_ROLE_ARN }}
          aws-region: us-east-1
```

### 4. Principle of Least Privilege (Đặc Quyền Tối Thiểu) Cho IAM Roles

```json
// ❌ Quá rộng
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}

// ✅ Đúng — Chỉ quyền cần thiết cho deployment
{
  "Effect": "Allow",
  "Action": [
    "ecs:UpdateService",
    "ecs:DescribeServices",
    "ecr:GetAuthorizationToken",
    "ecr:BatchCheckLayerAvailability",
    "ecr:PutImage"
  ],
  "Resource": [
    "arn:aws:ecs:us-east-1:123456789012:service/production/my-service",
    "arn:aws:ecr:us-east-1:123456789012:repository/my-app"
  ]
}
```

---

## Troubleshooting

### Lỗi: "Not authorized to perform sts:AssumeRoleWithWebIdentity"

```
Nguyên nhân phổ biến:
1. Trust Policy conditions không khớp với JWT claims
2. Subject (sub) claim sai format
3. Thumbprint của OIDC provider không đúng
4. permissions: id-token: write chưa được set

Debug:
- In ra JWT token để kiểm tra claims
- So sánh sub claim với condition trong Trust Policy
```

```yaml
# Debug: In ra OIDC token (chỉ dùng khi debug!)
- name: Debug OIDC token
  run: |
    TOKEN=$(curl -s -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
      "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=sts.amazonaws.com")
    
    # Decode payload (phần giữa của JWT)
    echo $TOKEN | jq -r '.value' | cut -d. -f2 | base64 -d 2>/dev/null | jq .
```

### Lỗi: "The OIDC provider's HTTPS certificate could not be verified"

```bash
# Lấy thumbprint mới của GitHub OIDC provider
openssl s_client -servername token.actions.githubusercontent.com \
  -connect token.actions.githubusercontent.com:443 < /dev/null 2>/dev/null | \
  openssl x509 -fingerprint -noout -sha1 | \
  tr -d ':'
```

### Lỗi: "permissions: id-token not configured"

```yaml
# PHẢI có ở job level hoặc workflow level
jobs:
  deploy:
    permissions:
      id-token: write   # ← BẮT BUỘC
      contents: read
```

---

## Câu Hỏi Phỏng Vấn

### Q: Giải thích OIDC trong GitHub Actions và tại sao tốt hơn dùng static credentials?

**Trả lời đầy đủ:**

OIDC (OpenID Connect) là giao thức xác thực cho phép GitHub Actions xác thực với cloud providers mà không cần lưu long-lived credentials (thông tin đăng nhập tồn tại lâu dài).

**Cơ chế:**
1. Khi workflow job bắt đầu, GitHub OIDC provider tạo một JWT token ngắn hạn (short-lived) chứa thông tin về workflow: repo, branch, environment, actor
2. Workflow gửi JWT token này đến cloud provider (AWS STS, GCP Workload Identity, Azure AD)
3. Cloud provider xác minh token với GitHub's OIDC endpoint, kiểm tra conditions (repo, branch, environment)
4. Nếu hợp lệ, cloud provider trả về temporary credentials chỉ sống trong khoảng 1 giờ

**Lợi ích so với static credentials:**
- **Không có secrets nào bị lộ lâu dài** — nếu bị intercepted, token hết hạn ngay sau job
- **Zero rotation burden** — không cần nhớ rotate credentials hàng quý
- **Fine-grained control** — IAM role conditions có thể restrict đến từng branch, environment, thậm chí từng tag
- **Audit trail tốt hơn** — cloud audit logs ghi rõ: workflow nào, từ repo nào, branch nào đã access

### Q: Trong AWS Trust Policy, `sub` claim có thể có những format nào?

```
repo:OWNER/REPO:ref:refs/heads/BRANCH        → Specific branch
repo:OWNER/REPO:ref:refs/tags/TAG            → Specific tag
repo:OWNER/REPO:environment:ENVIRONMENT      → Specific environment
repo:OWNER/REPO:pull_request                 → Any PR
repo:OWNER/REPO:ref:refs/heads/*             → Any branch (dùng StringLike)
repo:OWNER/*:*                               → All repos in org (StringLike)
```

### Q: Tại sao phải set `permissions: id-token: write`?

GitHub Actions theo mặc định không cấp quyền tạo OIDC token vì lý do bảo mật. `permissions: id-token: write` khai báo rõ ràng rằng workflow này cần đặc quyền tạo token để xác thực với external services. Nguyên tắc: phải explicit opt-in cho mọi quyền đặc biệt.

---

**Điều Hướng:**
- ← [2-variables.md](2-variables.md)
- → [4-vault-integration.md](4-vault-integration.md)

**Cập Nhật Lần Cuối:** 2026-05-11
