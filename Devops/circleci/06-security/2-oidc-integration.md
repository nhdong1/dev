# OIDC — OpenID Connect: Xác Thực Không Cần Static Credentials

> OIDC — OpenID Connect là giao thức xác thực hiện đại cho phép CircleCI xác thực với các cloud providers (AWS, GCP, Azure) mà **không cần lưu trữ long-lived credentials** — thông tin xác thực tồn tại vĩnh viễn. Thay vào đó, CircleCI nhận được short-lived token — mã thông tin ngắn hạn tự động hết hạn sau mỗi lần dùng.

## 📚 Mục Lục

1. [OIDC là gì?](#oidc-là-gì)
2. [Tại Sao OIDC Tốt Hơn Static Keys?](#tại-sao-oidc-tốt-hơn-static-keys)
3. [Cơ Chế Hoạt Động](#cơ-chế-hoạt-động)
4. [Tích Hợp với AWS](#tích-hợp-với-aws)
5. [Tích Hợp với GCP](#tích-hợp-với-gcp)
6. [Tích Hợp với Azure](#tích-hợp-với-azure)
7. [OIDC Token Claims](#oidc-token-claims)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## OIDC là gì?

**OIDC — OpenID Connect** là một lớp xác thực xây dựng trên nền **OAuth 2.0**. Trong ngữ cảnh CI/CD:

```
OIDC Identity Provider (IdP) — Nhà Cung Cấp Danh Tính: CircleCI
OIDC Relying Party (RP) — Bên Phụ Thuộc: AWS / GCP / Azure

Luồng hoạt động:
  1. CircleCI tạo JWT token — JSON Web Token có chữ ký số
  2. Pipeline gửi token này đến AWS/GCP/Azure
  3. Cloud provider xác minh chữ ký với CircleCI OIDC endpoint
  4. Cloud provider cấp short-lived credentials
  5. Credentials tự hết hạn sau ~1 giờ
```

### Khái Niệm Quan Trọng

| Thuật Ngữ | Giải Thích |
|-----------|-----------|
| **JWT** — JSON Web Token | Token có cấu trúc JSON, được ký số, chứa claims |
| **Claims** — Tuyên Bố | Thông tin trong JWT: ai đang yêu cầu, từ repo nào, branch nào |
| **OIDC Provider URL** | Endpoint CircleCI công bố để cloud providers xác minh chữ ký |
| **Web Identity Role** | IAM Role AWS chỉ cho phép được assume qua OIDC |
| **Workload Identity** | GCP tương đương với AWS Web Identity Role |
| **Short-lived Credentials** | Credentials hết hạn tự động (thường 1 giờ) |

---

## Tại Sao OIDC Tốt Hơn Static Keys?

### So Sánh Trực Tiếp

```
┌─────────────────────────────────────────────────────────────────┐
│                 STATIC CREDENTIALS (Cũ)                         │
├─────────────────────────────────────────────────────────────────┤
│  Lưu trữ: AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY trong       │
│           CircleCI Context                                       │
│  Thời hạn: Không bao giờ hết hạn (trừ khi manually rotate)    │
│  Rủi ro: Nếu key bị lộ → kẻ tấn công có quyền truy cập        │
│           mãi mãi cho đến khi phát hiện và revoke               │
│  Rotate: Phải làm thủ công, dễ quên, có downtime nếu làm sai   │
│  Audit: Khó biết ai/khi nào dùng key này                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      OIDC (Hiện Đại)                            │
├─────────────────────────────────────────────────────────────────┤
│  Lưu trữ: Chỉ cần AWS_ROLE_ARN (không phải secret)             │
│  Thời hạn: Token hết hạn sau 1 giờ, tự động renew               │
│  Rủi ro: Nếu token bị lộ → hết hạn trong giờ tiếp theo         │
│  Rotate: Không cần rotate — không có static key nào để rotate   │
│  Audit: AWS CloudTrail ghi đầy đủ với context CircleCI           │
└─────────────────────────────────────────────────────────────────┘
```

### Lợi Ích Cụ Thể

1. **Zero Secrets to Rotate** — Không cần xoay vòng: Không có long-lived key → không có rotation risk — rủi ro từ quên xoay vòng
2. **Blast Radius Reduction** — Giảm Bán Kính Thiệt Hại: Token hết hạn tự động sau 1 giờ
3. **Fine-grained Trust** — Tin Cậy Chi Tiết: AWS Role chỉ trust pipeline từ repo cụ thể, branch cụ thể
4. **Better Audit Trail** — Nhật Ký Tốt Hơn: Cloud provider biết chính xác pipeline nào đã request credentials

---

## Cơ Chế Hoạt Động

### Sơ Đồ Luồng OIDC

```
┌──────────────┐    1. Pipeline starts     ┌──────────────────┐
│  CircleCI    │ ─────────────────────────► │  CircleCI OIDC   │
│  Pipeline    │                            │  Provider        │
│              │ ◄───────────────────────── │  (IdP)           │
│              │    2. JWT Token            └──────────────────┘
│              │       (CIRCLE_OIDC_TOKEN)
│              │
│              │    3. STS AssumeRole       ┌──────────────────┐
│              │       + JWT Token          │   AWS STS        │
│              │ ─────────────────────────► │  (Security Token │
│              │                            │   Service)       │
│              │ ◄───────────────────────── │                  │
│              │    4. Short-lived creds    └──────────────────┘
│              │       (15min - 1h)
│              │
│              │    5. API calls with       ┌──────────────────┐
│              │       temp credentials     │   AWS Services   │
│              │ ─────────────────────────► │   (S3, ECR,      │
└──────────────┘                            │    ECS, etc.)    │
                                            └──────────────────┘
```

### CircleCI OIDC Provider URL

CircleCI cung cấp OIDC discovery endpoint — điểm cuối khám phá OIDC:

```
https://oidc.circleci.com/org/<organization-id>

Ví dụ:
https://oidc.circleci.com/org/550e8400-e29b-41d4-a716-446655440000

Tại endpoint này có:
  /.well-known/openid-configuration  → OIDC metadata
  /.well-known/jwks                  → Public keys để verify JWT
```

---

## Tích Hợp với AWS

### Bước 1: Tạo OIDC Identity Provider trong AWS

```bash
# Lấy organization ID của CircleCI
# Dashboard → Organization Settings → Organization ID (UUID)
ORG_ID="550e8400-e29b-41d4-a716-446655440000"

# Tạo OIDC Identity Provider
aws iam create-open-id-connect-provider \
  --url "https://oidc.circleci.com/org/${ORG_ID}" \
  --client-id-list "https://oidc.circleci.com/org/${ORG_ID}" \
  --thumbprint-list "9e99a48a9960b14926bb7f3b02e22da2b0ab7280"
  
# Lấy thumbprint:
# openssl s_client -connect oidc.circleci.com:443 -showcerts < /dev/null 2>/dev/null \
#   | openssl x509 -fingerprint -sha1 -noout | cut -d= -f2 | tr -d ':'
```

### Bước 2: Tạo IAM Role với Trust Policy — Chính Sách Tin Cậy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789:oidc-provider/oidc.circleci.com/org/<ORG_ID>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "oidc.circleci.com/org/<ORG_ID>:sub": "org/<ORG_ID>/project/<PROJECT_ID>/user/*"
        }
      }
    }
  ]
}
```

**Điều kiện phổ biến để restrict trust — hạn chế quyền tin cậy:**

```json
{
  "Condition": {
    "StringEquals": {
      "oidc.circleci.com/org/<ORG_ID>:sub": 
        "org/<ORG_ID>/project/<PROJECT_ID>/user/<USER_ID>"
    }
  }
}
```

```json
{
  "Condition": {
    "StringLike": {
      "oidc.circleci.com/org/<ORG_ID>:sub": 
        "org/<ORG_ID>/project/<PROJECT_ID>/user/*"
    }
  }
}
```

### Bước 3: Attach IAM Policy vào Role

```bash
# Ví dụ: Role chỉ có quyền push lên ECR
aws iam attach-role-policy \
  --role-name CircleCI-ECR-Push \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser
```

### Bước 4: Dùng trong CircleCI Config

```yaml
version: 2.1

orbs:
  aws-cli: circleci/aws-cli@4.1.0

jobs:
  deploy-to-aws:
    docker:
      - image: cimg/base:stable
    environment:
      AWS_DEFAULT_REGION: ap-southeast-1
    steps:
      - checkout
      - aws-cli/setup:
          role_arn: $AWS_ROLE_ARN          # Từ Context hoặc Project vars
          role_session_name: circleci-${CIRCLE_BUILD_NUM}
          # aws-cli orb tự xử lý OIDC token exchange — trao đổi token OIDC
      
      - run:
          name: Xác nhận identity
          command: aws sts get-caller-identity
      
      - run:
          name: Push lên ECR
          command: |
            ECR_URL="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
            aws ecr get-login-password | \
              docker login --username AWS --password-stdin $ECR_URL
            docker push $ECR_URL/my-app:${CIRCLE_SHA1}

workflows:
  deploy:
    jobs:
      - deploy-to-aws:
          context: aws-oidc-config    # Chỉ chứa AWS_ROLE_ARN và AWS_ACCOUNT_ID
          filters:
            branches:
              only: main
```

### Dùng OIDC Thủ Công (không qua orb)

```yaml
jobs:
  manual-oidc:
    docker:
      - image: cimg/aws:2023.09
    steps:
      - run:
          name: Exchange OIDC token lấy AWS credentials
          command: |
            # CIRCLE_OIDC_TOKEN được CircleCI tự inject vào job
            # khi pipeline dùng context hoặc khi OIDC được kích hoạt
            
            CREDS=$(aws sts assume-role-with-web-identity \
              --role-arn "$AWS_ROLE_ARN" \
              --role-session-name "circleci-${CIRCLE_WORKFLOW_ID}" \
              --web-identity-token "$CIRCLE_OIDC_TOKEN" \
              --duration-seconds 3600 \
              --query 'Credentials' \
              --output json)
            
            # Lưu credentials vào file để dùng trong các steps tiếp theo
            echo "export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r '.AccessKeyId')" >> $BASH_ENV
            echo "export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r '.SecretAccessKey')" >> $BASH_ENV
            echo "export AWS_SESSION_TOKEN=$(echo $CREDS | jq -r '.SessionToken')" >> $BASH_ENV
            source $BASH_ENV
      
      - run:
          name: Dùng AWS credentials
          command: aws s3 ls
```

---

## Tích Hợp với GCP

### Bước 1: Tạo Workload Identity Pool — Nhóm Danh Tính Workload

```bash
PROJECT_ID="my-gcp-project"
ORG_ID="550e8400-e29b-41d4-a716-446655440000"  # CircleCI Organization ID

# Tạo Workload Identity Pool
gcloud iam workload-identity-pools create "circleci-pool" \
  --project="$PROJECT_ID" \
  --location="global" \
  --display-name="CircleCI Pool"

# Lấy pool name
POOL_NAME=$(gcloud iam workload-identity-pools describe "circleci-pool" \
  --project="$PROJECT_ID" \
  --location="global" \
  --format="value(name)")
```

### Bước 2: Thêm OIDC Provider vào Pool

```bash
gcloud iam workload-identity-pools providers create-oidc "circleci-provider" \
  --project="$PROJECT_ID" \
  --location="global" \
  --workload-identity-pool="circleci-pool" \
  --display-name="CircleCI OIDC Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.org_id=assertion.iss" \
  --issuer-uri="https://oidc.circleci.com/org/${ORG_ID}"
```

### Bước 3: Grant Service Account Access — Cấp Quyền Tài Khoản Dịch Vụ

```bash
SERVICE_ACCOUNT="circleci-deployer@${PROJECT_ID}.iam.gserviceaccount.com"

# Tạo service account nếu chưa có
gcloud iam service-accounts create "circleci-deployer" \
  --project="$PROJECT_ID"

# Grant Cloud Run deploy permission
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$SERVICE_ACCOUNT" \
  --role="roles/run.developer"

# Bind workload identity → service account
gcloud iam service-accounts add-iam-policy-binding "$SERVICE_ACCOUNT" \
  --project="$PROJECT_ID" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/${POOL_NAME}/attribute.org_id/https://oidc.circleci.com/org/${ORG_ID}"
```

### Bước 4: Dùng trong CircleCI Config

```yaml
version: 2.1

orbs:
  gcp-cli: circleci/gcp-cli@3.2.0

jobs:
  deploy-to-gcp:
    docker:
      - image: cimg/gcp:2023.09
    steps:
      - checkout
      - run:
          name: Xác thực GCP qua OIDC
          command: |
            # Tạo credentials config file cho gcloud
            gcloud iam workload-identity-pools create-cred-config \
              "$WORKLOAD_IDENTITY_PROVIDER" \
              --service-account="$GCP_SERVICE_ACCOUNT" \
              --credential-source-file="/proc/1/fd/0" \
              --output-file="$HOME/gcp-credentials.json"
            
            # Đặt credentials
            gcloud auth login --cred-file="$HOME/gcp-credentials.json"
            gcloud config set project $GCP_PROJECT_ID
      
      - run:
          name: Deploy lên Cloud Run
          command: |
            gcloud run deploy my-service \
              --image "gcr.io/${GCP_PROJECT_ID}/my-app:${CIRCLE_SHA1}" \
              --region asia-southeast1 \
              --platform managed

workflows:
  deploy:
    jobs:
      - deploy-to-gcp:
          context: gcp-oidc-config    # Chứa WORKLOAD_IDENTITY_PROVIDER, GCP_SERVICE_ACCOUNT
          filters:
            branches:
              only: main
```

---

## Tích Hợp với Azure

### Bước 1: Tạo Federated Identity Credential — Thông Tin Xác Thực Liên Kết

```bash
# Đăng nhập Azure CLI
az login

TENANT_ID="your-tenant-id"
SUBSCRIPTION_ID="your-subscription-id"
APP_NAME="circleci-oidc-app"
ORG_ID="550e8400-e29b-41d4-a716-446655440000"

# Tạo App Registration
az ad app create --display-name "$APP_NAME"
APP_ID=$(az ad app list --display-name "$APP_NAME" --query "[0].appId" -o tsv)

# Tạo Service Principal
az ad sp create --id "$APP_ID"

# Tạo Federated Identity Credential
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters "{
    \"name\": \"circleci-credential\",
    \"issuer\": \"https://oidc.circleci.com/org/${ORG_ID}\",
    \"subject\": \"org/${ORG_ID}/project/<PROJECT_ID>/user/*\",
    \"audiences\": [\"https://oidc.circleci.com/org/${ORG_ID}\"]
  }"

# Gán role cho Service Principal
az role assignment create \
  --assignee "$APP_ID" \
  --role "Contributor" \
  --scope "/subscriptions/${SUBSCRIPTION_ID}"
```

### Bước 2: Dùng trong CircleCI Config

```yaml
jobs:
  deploy-to-azure:
    docker:
      - image: mcr.microsoft.com/azure-cli
    steps:
      - run:
          name: Đăng nhập Azure qua OIDC
          command: |
            az login \
              --service-principal \
              --username "$AZURE_CLIENT_ID" \
              --tenant "$AZURE_TENANT_ID" \
              --federated-token "$CIRCLE_OIDC_TOKEN"
      
      - run:
          name: Deploy lên Azure Container Apps
          command: |
            az containerapp update \
              --name my-app \
              --resource-group my-rg \
              --image myregistry.azurecr.io/my-app:${CIRCLE_SHA1}

workflows:
  deploy:
    jobs:
      - deploy-to-azure:
          context: azure-oidc-config    # AZURE_CLIENT_ID, AZURE_TENANT_ID
```

---

## OIDC Token Claims

**Claims** — Tuyên Bố là các thuộc tính được nhúng vào JWT token. CircleCI cung cấp nhiều claims hữu ích để tạo điều kiện trust chặt chẽ:

### Các Claims Chuẩn (Standard Claims)

| Claim | Giá Trị Mẫu | Ý Nghĩa |
|-------|-------------|---------|
| `iss` | `https://oidc.circleci.com/org/<org-id>` | Issuer — Bên phát hành token |
| `sub` | `org/<org-id>/project/<proj-id>/user/<user-id>` | Subject — Chủ thể |
| `aud` | `https://oidc.circleci.com/org/<org-id>` | Audience — Đối tượng |
| `iat` | `1716019200` | Issued At — Thời điểm phát hành |
| `exp` | `1716022800` | Expiration — Thời điểm hết hạn |

### Các Claims Mở Rộng của CircleCI

| Claim | Giá Trị Mẫu | Ý Nghĩa |
|-------|-------------|---------|
| `oidc.circleci.com/project-id` | `<uuid>` | Project ID |
| `oidc.circleci.com/context-ids` | `["<uuid>"]` | Danh sách context IDs |
| `oidc.circleci.com/vcs-origin` | `github.com/myorg/myrepo` | VCS origin |
| `oidc.circleci.com/vcs-ref` | `refs/heads/main` | Branch/tag reference |

### Dùng Claims để Restrict Trust — Hạn Chế Quyền Tin Cậy

```json
// AWS Trust Policy ví dụ — Chỉ trust pipeline từ nhánh main của repo cụ thể
{
  "Condition": {
    "StringEquals": {
      "oidc.circleci.com/org/<ORG_ID>:oidc.circleci.com/vcs-ref": 
        "refs/heads/main",
      "oidc.circleci.com/org/<ORG_ID>:oidc.circleci.com/project-id": 
        "<specific-project-id>"
    }
  }
}
```

### Decode Token để Debug

```bash
# Trong pipeline, decode OIDC token để xem claims
echo $CIRCLE_OIDC_TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | jq .

# Kết quả ví dụ:
# {
#   "iss": "https://oidc.circleci.com/org/550e8400-...",
#   "sub": "org/550e8400-.../project/abc123.../user/def456...",
#   "oidc.circleci.com/vcs-ref": "refs/heads/main",
#   "oidc.circleci.com/vcs-origin": "github.com/myorg/myapp",
#   "exp": 1716022800
# }
```

---

## Best Practices

### 1. Principle of Least Privilege — Nguyên Tắc Quyền Tối Thiểu

```bash
# KHÔNG TỐT: Role có quyền admin toàn bộ AWS
aws iam attach-role-policy \
  --role-name CircleCI-Role \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# TỐT: Role chỉ có quyền cụ thể cho task đó
# Ví dụ: Role chỉ push ECR + deploy ECS
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "arn:aws:ecr:ap-southeast-1:123456789:repository/my-app"
    },
    {
      "Effect": "Allow",
      "Action": ["ecs:UpdateService"],
      "Resource": "arn:aws:ecs:ap-southeast-1:123456789:service/prod/my-app"
    }
  ]
}
```

### 2. Restrict by VCS Ref — Hạn Chế Theo Tham Chiếu VCS

```json
// Chỉ cho phép nhánh main và release/* sử dụng production role
{
  "Condition": {
    "StringLike": {
      "oidc.circleci.com/org/<ORG_ID>:oidc.circleci.com/vcs-ref": [
        "refs/heads/main",
        "refs/heads/release/*"
      ]
    }
  }
}
```

### 3. Tạo Role Riêng Cho Từng Môi Trường

```
AWS Roles:
  circleci-staging-ecr-push   → Trust policy: mọi user, mọi branch
  circleci-staging-ecs-deploy → Trust policy: mọi user, nhánh develop/*
  circleci-prod-ecr-push      → Trust policy: mọi user, nhánh main
  circleci-prod-ecs-deploy    → Trust policy: mọi user, nhánh main
```

### 4. Giám Sát OIDC Token Usage — Theo Dõi Sử Dụng Token

```bash
# AWS CloudTrail query — truy vấn nhật ký để xem ai đã dùng role
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity \
  --query 'Events[*].{Time:EventTime,User:Username,Source:RequestParameters}' \
  --output table
```

---

## Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: OIDC là gì và tại sao nó tốt hơn static API keys?**

> **A:** OIDC — OpenID Connect là giao thức xác thực cho phép CircleCI chứng minh danh tính với cloud providers mà không cần lưu trữ long-lived credentials. Thay vào đó, CircleCI nhận một JWT token ngắn hạn (hết hạn sau ~1 giờ) để trao đổi lấy temporary credentials từ cloud provider. Ưu điểm: không có static key để rotate hay bị lộ; nếu token bị lộ, nó tự hết hạn; audit trail tốt hơn vì cloud provider biết chính xác pipeline nào đã request.

**Q: Khi dùng OIDC với AWS, CircleCI cần lưu trữ gì trong Context?**

> **A:** Chỉ cần lưu `AWS_ROLE_ARN` (và `AWS_DEFAULT_REGION`) — đây không phải secrets vì ARN là thông tin công khai. Không cần lưu `AWS_ACCESS_KEY_ID` hay `AWS_SECRET_ACCESS_KEY`. CircleCI tự động inject `CIRCLE_OIDC_TOKEN` vào pipeline, sau đó dùng token này để call AWS STS AssumeRoleWithWebIdentity và nhận temporary credentials.

### Câu Hỏi Nâng Cao

**Q: Làm thế nào để restrict OIDC trust chỉ cho một nhánh cụ thể?**

> **A:** Trong AWS Trust Policy, sử dụng Condition với claim `oidc.circleci.com/vcs-ref`. Ví dụ: `"StringEquals": {"oidc.circleci.com/org/<ORG_ID>:oidc.circleci.com/vcs-ref": "refs/heads/main"}` — điều này đảm bảo chỉ pipeline chạy trên nhánh main mới có thể assume role production, ngay cả khi attacker tạo branch khác và cố gắng request token.

**Q: CIRCLE_OIDC_TOKEN được inject khi nào?**

> **A:** `CIRCLE_OIDC_TOKEN` được inject tự động khi job sử dụng Context. Nếu job không có context, token không được tạo. Đây là thiết kế có chủ ý — buộc developer phải khai báo rõ ràng rằng job này cần xác thực với cloud provider.

**Q: Kể một vấn đề thực tế bạn đã giải quyết với OIDC.**

> **A (ví dụ mẫu):** "Trước đây team chúng tôi có AWS access key không được rotate trong 2 năm, khi một developer rời công ty chúng tôi không chắc key đó có bị lộ không. Tôi đề xuất chuyển sang OIDC: (1) tạo IAM Role với trust policy chỉ cho phép CircleCI org của chúng tôi, (2) restrict theo project ID và nhánh main, (3) xóa access key cũ. Kết quả: zero credentials cần quản lý, audit trail tốt hơn, và một rotation incident được loại bỏ hoàn toàn."

---

## 🔗 Điều Hướng

| ← Trước | Vị Trí | Tiếp → |
|---------|--------|--------|
| [1-contexts.md](./1-contexts.md) | **2-oidc-integration.md** | [3-ip-ranges.md](./3-ip-ranges.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Độ Khó:** ⭐⭐⭐ Nâng Cao
**Thời Gian Đọc:** ~60 phút
