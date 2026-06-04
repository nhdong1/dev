# 3 — OIDC Federation (Liên Kết Định Danh OpenID Connect)

> OIDC — OpenID Connect (Kết Nối Định Danh Mở) — là giao thức xác thực hiện đại xây dựng trên nền OAuth 2.0, sử dụng JWT (JSON Web Token — Token Web JSON) để cho phép CI/CD pipelines, ứng dụng, và dịch vụ bên ngoài nhận temporary AWS credentials mà không cần lưu trữ long-term access keys.

---

## 📚 Mục Lục

1. [Tổng Quan OIDC](#tổng-quan-oidc)
2. [Luồng Xác Thực OIDC](#luồng-xác-thực-oidc)
3. [GitHub Actions + OIDC](#github-actions--oidc)
4. [GitLab CI/CD + OIDC](#gitlab-cicd--oidc)
5. [Google Accounts + OIDC](#google-accounts--oidc)
6. [Kubernetes Workloads + OIDC](#kubernetes-workloads--oidc)
7. [Cấu Hình OIDC Provider Trong IAM](#cấu-hình-oidc-provider-trong-iam)
8. [Bảo Mật OIDC](#bảo-mật-oidc)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan OIDC

### OIDC vs SAML — Khi Nào Dùng Cái Nào?

```
SAML 2.0:                              OIDC:
- XML-based, verbose                   - JSON-based, lightweight
- Enterprise / doanh nghiệp            - Modern apps, CI/CD, DevOps
- Active Directory, ADFS, Okta         - GitHub, Google, GitLab, Auth0
- Human users đăng nhập Console       - Machine-to-machine, automated
- Browser-based SSO                    - API-friendly, token-based
- Phức tạp cấu hình                    - Đơn giản hơn, developer-friendly

Quyết định nhanh:
- Nhân viên cần đăng nhập Console → SAML / IAM Identity Center
- GitHub Actions cần deploy lên AWS → OIDC
- Mobile app cần truy cập S3 → Cognito + OIDC
```

### JWT (JSON Web Token — Token Web JSON)

```
JWT là token nhỏ gọn gồm 3 phần, ngăn cách bởi dấu chấm:
<header>.<payload>.<signature>

Header (Base64URL encoded):
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "key-id-123"  // Key ID để verify signature
}

Payload (Base64URL encoded — chứa claims):
{
  "iss": "https://token.actions.githubusercontent.com",  // Issuer
  "sub": "repo:myorg/myrepo:ref:refs/heads/main",       // Subject
  "aud": "sts.amazonaws.com",                            // Audience
  "exp": 1716000000,                                     // Expiration
  "iat": 1715996400,                                     // Issued At
  "jti": "unique-token-id",                              // JWT ID
  
  // Custom claims từ IdP
  "repository": "myorg/myrepo",
  "ref": "refs/heads/main",
  "workflow": "deploy"
}

Signature: IdP ký bằng private key, AWS verify bằng public key
           lấy từ OIDC discovery endpoint của IdP
```

---

## Luồng Xác Thực OIDC

### Luồng Cơ Bản

```
OIDC Provider               AWS IAM / STS                 AWS Resources
(GitHub/Google/etc)              │                               │
      │                          │                               │
      │ 1. Issue JWT token        │                               │
      │    (signed by IdP)        │                               │
      │                          │                               │
      │ 2. Client gửi JWT ───────▶│                               │
      │    AssumeRoleWithWebIdentity                              │
      │                          │                               │
      │                       3. AWS fetch public keys           │
      │                          từ OIDC discovery URL           │
      │                       https://idp.example.com            │
      │                       /.well-known/openid-configuration   │
      │                          │                               │
      │                       4. Verify JWT signature            │
      │                          Verify claims (iss, aud, sub)    │
      │                          Check trust policy conditions    │
      │                          │                               │
      │                       5. AssumeRole ───────────────────▶│
      │                          │                               │
      │                       6. Return temporary credentials    │
      │                          AccessKey + SecretKey + Token   │
      │                          │                               │
      │                          ──────── API Calls ────────────▶│
```

### AssumeRoleWithWebIdentity API Call

```python
import boto3

sts = boto3.client('sts')

response = sts.assume_role_with_web_identity(
    RoleArn='arn:aws:iam::123456789012:role/GitHubActionsRole',
    RoleSessionName='github-deploy-session',
    WebIdentityToken='eyJhbGciOiJSUzI1NiJ9...',  # JWT từ IdP
    DurationSeconds=3600
)

credentials = response['Credentials']
# Dùng AccessKeyId, SecretAccessKey, SessionToken
```

---

## GitHub Actions + OIDC

### Tại Sao Không Dùng AWS Access Keys Trong GitHub Secrets?

```
Vấn đề với long-term access keys:
- Keys không hết hạn tự động — nếu lộ phải revoke thủ công
- Developer có thể vô tình commit keys vào code
- Keys rotate thủ công → dễ forget, tạo security gap
- Nếu repo bị fork, secrets có thể bị expose trong forked PRs

Giải pháp OIDC:
- Không có credentials nào được lưu trong GitHub
- Mỗi workflow run nhận JWT token tạm thời từ GitHub
- JWT expire sau vài phút
- AWS verify token trực tiếp với GitHub
- Nếu workflow kết thúc, credentials tự hết hạn
```

### Cấu Hình Bước 1: Tạo OIDC Provider Trong AWS

```bash
# Tạo OIDC Provider cho GitHub Actions
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1

# Lưu ý thumbprint:
# - Đây là thumbprint của GitHub's OIDC root CA certificate
# - AWS dùng để verify GitHub's TLS certificate
# - Có thể thay đổi — kiểm tra AWS docs định kỳ

# Verify provider được tạo
aws iam list-open-id-connect-providers
```

### Cấu Hình Bước 2: Tạo IAM Role Cho GitHub Actions

```json
// Trust Policy — chỉ cho phép workflow từ repo cụ thể
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
          // Chỉ accept token có audience là sts.amazonaws.com
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          // Chỉ cho phép từ main branch của repo cụ thể
          "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

```bash
# Tạo role
aws iam create-role \
  --role-name GitHubActionsDeployRole \
  --assume-role-policy-document file://github-trust-policy.json \
  --max-session-duration 3600

# Gắn permission policy
aws iam attach-role-policy \
  --role-name GitHubActionsDeployRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonECRFullAccess

# Hoặc tạo custom policy chặt hơn
aws iam put-role-policy \
  --role-name GitHubActionsDeployRole \
  --policy-name DeployPolicy \
  --policy-document file://deploy-policy.json
```

### Cấu Hình Bước 3: GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to AWS

on:
  push:
    branches: [main]

# Cấp quyền để GitHub tạo OIDC token
permissions:
  id-token: write   # Bắt buộc cho OIDC
  contents: read    # Để checkout code

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      # Bước quan trọng: Configure AWS credentials qua OIDC
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsDeployRole
          role-session-name: github-deploy-${{ github.run_id }}
          aws-region: us-east-1
          # Không cần aws-access-key-id hay aws-secret-access-key!

      # Sau bước trên, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY,
      # AWS_SESSION_TOKEN đã được set tự động trong environment

      - name: Deploy Lambda
        run: |
          aws lambda update-function-code \
            --function-name my-function \
            --zip-file fileb://function.zip

      - name: Push to ECR
        run: |
          aws ecr get-login-password | docker login --username AWS \
            --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
          docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
```

### Trust Policy Conditions Nâng Cao

```json
// Cho phép tất cả branches của một repo (không chỉ main)
{
  "Condition": {
    "StringLike": {
      "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:*"
    },
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
    }
  }
}

// Chỉ cho phép từ specific environment (GitHub Environments)
{
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:environment:production",
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
    }
  }
}

// Cho phép từ nhiều repos
{
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
    },
    "StringLike": {
      "token.actions.githubusercontent.com:sub": [
        "repo:myorg/frontend:*",
        "repo:myorg/backend:*",
        "repo:myorg/infrastructure:*"
      ]
    }
  }
}
```

---

## GitLab CI/CD + OIDC

### Cấu Hình GitLab OIDC

```bash
# Tạo OIDC Provider cho GitLab
# (thay gitlab.com bằng self-hosted GitLab URL nếu cần)
aws iam create-open-id-connect-provider \
  --url https://gitlab.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list <gitlab-cert-thumbprint>
```

```json
// Trust Policy cho GitLab CI
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/gitlab.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "gitlab.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          // Chỉ cho phép từ project cụ thể, branch main
          "gitlab.com:sub": "project_path:mygroup/myproject:ref_type:branch:ref:main"
        }
      }
    }
  ]
}
```

```yaml
# .gitlab-ci.yml
deploy:
  stage: deploy
  image: amazon/aws-cli:latest
  
  id_tokens:
    AWS_TOKEN:
      aud: sts.amazonaws.com
  
  script:
    - |
      # Dùng OIDC token để assume role
      CREDENTIALS=$(aws sts assume-role-with-web-identity \
        --role-arn arn:aws:iam::123456789012:role/GitLabDeployRole \
        --role-session-name gitlab-ci-$CI_JOB_ID \
        --web-identity-token $AWS_TOKEN \
        --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
        --output text)
      
      export AWS_ACCESS_KEY_ID=$(echo $CREDENTIALS | awk '{print $1}')
      export AWS_SECRET_ACCESS_KEY=$(echo $CREDENTIALS | awk '{print $2}')
      export AWS_SESSION_TOKEN=$(echo $CREDENTIALS | awk '{print $3}')
      
      # Deploy
      aws lambda update-function-code ...
  
  only:
    - main
```

---

## Google Accounts + OIDC

### Cho Phép Google Workspace Users Truy Cập AWS

```json
// Trust Policy cho Google OIDC
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "accounts.google.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          // Giới hạn audience để ngăn token từ Google app khác
          "accounts.google.com:aud": "your-google-client-id.apps.googleusercontent.com",
          // Chỉ cho phép email domain của công ty
          "accounts.google.com:hd": "mycompany.com"
        }
      }
    }
  ]
}
```

```python
# Python example: Google login → AWS credentials
from google.oauth2 import id_token
from google.auth.transport import requests as google_requests
import boto3

def get_aws_credentials_from_google(google_id_token: str) -> dict:
    # Verify Google ID token
    idinfo = id_token.verify_oauth2_token(
        google_id_token,
        google_requests.Request(),
        "your-google-client-id.apps.googleusercontent.com"
    )
    
    # Chỉ cho phép email từ domain của công ty
    if not idinfo['email'].endswith('@mycompany.com'):
        raise ValueError("Unauthorized domain")
    
    # Assume AWS role bằng Google token
    sts = boto3.client('sts', region_name='us-east-1')
    response = sts.assume_role_with_web_identity(
        RoleArn='arn:aws:iam::123456789012:role/GoogleWorkspaceAccess',
        RoleSessionName=idinfo['email'],
        WebIdentityToken=google_id_token,
        DurationSeconds=3600
    )
    
    return response['Credentials']
```

---

## Kubernetes Workloads + OIDC

### IRSA — IAM Roles for Service Accounts (IAM Roles Cho Service Accounts)

Kubernetes có OIDC provider riêng — AWS EKS tự động tạo OIDC provider cho cluster.

```bash
# Xem OIDC provider URL của EKS cluster
aws eks describe-cluster \
  --name my-cluster \
  --query "cluster.identity.oidc.issuer" \
  --output text
# Output: https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE

# Tạo OIDC provider trong IAM (nếu chưa có)
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve
```

```json
// Trust Policy cho Kubernetes Service Account
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          // Chỉ cho phép Service Account cụ thể trong namespace cụ thể
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub":
            "system:serviceaccount:production:my-app-service-account",
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:aud":
            "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

```yaml
# Kubernetes Service Account với annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-service-account
  namespace: production
  annotations:
    # Annotation này kích hoạt IRSA
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/MyAppRole

---
# Pod tự động nhận AWS credentials qua IRSA
# Không cần hardcode credentials trong container
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      serviceAccountName: my-app-service-account  # Dùng SA đã annotate
      containers:
      - name: app
        image: myapp:latest
        # AWS SDK tự động detect credentials từ
        # AWS_WEB_IDENTITY_TOKEN_FILE và AWS_ROLE_ARN env vars
        # được inject tự động bởi EKS mutating webhook
```

```bash
# Tạo IAM role với trust policy cho IRSA
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace production \
  --name my-app-service-account \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

---

## Cấu Hình OIDC Provider Trong IAM

### Tìm Thumbprint Của OIDC Provider

```bash
# Cách lấy thumbprint của OIDC provider
# (thay URL bằng OIDC provider của bạn)
OIDC_URL="https://token.actions.githubusercontent.com"

# Lấy JWKS URL từ discovery document
JWKS_URI=$(curl -s "${OIDC_URL}/.well-known/openid-configuration" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['jwks_uri'])")

# Lấy certificate thumbprint
THUMBPRINT=$(echo | openssl s_client \
  -servername $(echo $JWKS_URI | sed 's|https://||' | cut -d/ -f1) \
  -connect $(echo $JWKS_URI | sed 's|https://||' | cut -d/ -f1):443 2>/dev/null | \
  openssl x509 -fingerprint -noout -sha1 | \
  sed 's/SHA1 Fingerprint=//' | tr -d ':' | tr '[:upper:]' '[:lower:]')

echo "Thumbprint: $THUMBPRINT"
```

### Quản Lý OIDC Provider

```bash
# Tạo provider
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1

# Xem danh sách providers
aws iam list-open-id-connect-providers

# Xem chi tiết
aws iam get-open-id-connect-provider \
  --open-id-connect-provider-arn arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com

# Cập nhật thumbprint (khi cert thay đổi)
aws iam update-open-id-connect-provider-thumbprint \
  --open-id-connect-provider-arn arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com \
  --thumbprint-list new-thumbprint-here

# Xóa provider
aws iam delete-open-id-connect-provider \
  --open-id-connect-provider-arn arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com
```

---

## Bảo Mật OIDC

### Các Rủi Ro Và Giảm Thiểu

```
1. Quá Rộng Trust Policy (Overly Permissive Trust Policy):
   
   Rủi ro:
   Nếu sub condition chỉ check repo nhưng không check ref:
   "sub": "repo:myorg/myrepo:*"
   → Bất kỳ branch, tag, PR nào trong repo đều có thể deploy
   → Attacker fork/PR có thể trigger deployment
   
   Giảm thiểu:
   Dùng specific conditions:
   - Production: "sub": "repo:myorg/myrepo:ref:refs/heads/main"
   - Dùng GitHub Environments với protection rules

2. Stolen JWT Token:
   Rủi ro: Token bị intercept trong transit
   Giảm thiểu:
   - JWT có exp (expiration) ngắn (thường 5–10 phút)
   - Chỉ dùng HTTPS
   - jti (JWT ID) claim ngăn replay attacks nếu STS cache used tokens

3. Confused Deputy Attack (Tấn Công Đại Lý Nhầm Lẫn):
   Rủi ro: Attacker dùng token cho service A để truy cập service B
   Giảm thiểu:
   - Luôn verify aud (audience) claim trong trust policy
   - Condition: "aud": "sts.amazonaws.com" (không phải aud của app khác)

4. Compromised IdP:
   Rủi ro: Nếu GitHub/Google bị hack, attacker có thể tạo fake tokens
   Giảm thiểu:
   - Monitor AWS CloudTrail cho bất thường AssumeRoleWithWebIdentity
   - Dùng các conditions ngặt nghèo nhất có thể
   - Theo dõi security advisories của IdP
```

### CloudTrail Monitoring Cho OIDC

```bash
# Query CloudTrail để monitor OIDC federation
# (dùng CloudTrail Lake hoặc Athena)

SELECT
    eventTime,
    userIdentity.sessionContext.sessionIssuer.userName as role_name,
    requestParameters.roleSessionName as session_name,
    sourceIPAddress,
    requestParameters.webIdentityToken as token_preview
FROM cloudtrail_logs
WHERE eventName = 'AssumeRoleWithWebIdentity'
  AND eventTime > now() - interval '24' hour
ORDER BY eventTime DESC;
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Tại sao OIDC tốt hơn lưu AWS Access Keys trong GitHub Secrets?

```
Long-term access keys trong GitHub Secrets:
- Keys không hết hạn → window of opportunity lớn nếu bị lộ
- Developer có thể vô tình log keys trong CI output
- Phải manually rotate — risk của expired/forgotten rotation
- Nếu GitHub bị breach, keys bị lộ
- Audit trail yếu: chỉ biết "GitHub Action" dùng key, không biết workflow nào

OIDC:
- Không có credentials lưu trong GitHub
- JWT expire sau vài phút (thường 10 phút)
- Mỗi run nhận token riêng — không thể reuse
- Trust policy giới hạn chính xác repo/branch nào được trust
- CloudTrail ghi rõ: role session name, source repo, run ID
```

### Q2: Giải thích IAM OIDC thumbprint và tại sao cần thiết?

```
OIDC thumbprint là SHA-1 hash của root CA certificate trong chuỗi chứng chỉ TLS của IdP.

Tại sao cần:
1. AWS STS cần fetch public keys từ OIDC discovery URL của IdP
   để verify JWT signatures
2. Trước khi fetch, AWS cần verify TLS certificate của IdP endpoint
3. Thumbprint cho phép AWS verify TLS cert của IdP endpoint
   mà không phụ thuộc hoàn toàn vào CA bundle của AWS

Nguy cơ thumbprint cũ:
- IdP thay đổi root CA certificate → thumbprint cũ invalid
- AWS sẽ từ chối verify, tất cả OIDC federation bị break
- Solution: Update thumbprint trong OIDC provider definition

AWS gần đây đã cải thiện: Với một số providers (GitHub, Google),
AWS tự động pin thumbprints và verify qua trusted CAs.
```

### Q3: Làm thế nào bảo vệ production deployment khỏi bị trigger từ branch không phải main?

```
Dùng trust policy condition chặt chẽ:

// Chỉ cho phép từ main branch
"token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"

Kết hợp với GitHub Environments:
// Chỉ cho phép từ environment "production" (có branch protection rules)
"token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:environment:production"

GitHub Environment protection rules:
- Required reviewers: phải được approve trước khi deploy
- Wait timer: delay deployment sau approval
- Deployment branches: chỉ từ main branch

Kết quả: Chỉ workflow chạy trên main branch VÀ được approve
trong "production" environment mới có thể assume production role.
```

### Q4: IRSA (IAM Roles for Service Accounts) hoạt động như thế nào?

```
Flow kỹ thuật:

1. EKS cluster có OIDC provider
2. Mỗi Kubernetes Service Account được annotate với IAM Role ARN
3. EKS mutating webhook inject vào mỗi pod:
   - AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
   - AWS_ROLE_ARN=arn:aws:iam::123456789012:role/MyRole
   
4. Token file chứa JWT được Kubernetes tạo, signed bởi cluster's OIDC provider
5. AWS SDK tự động detect biến môi trường này, đọc JWT, gọi STS AssumeRoleWithWebIdentity
6. Mỗi pod nhận temporary credentials riêng — theo nguyên tắc least privilege

Tại sao tốt hơn node-level IAM role:
- Instance Profile cấp quyền cho toàn bộ node → mọi pod trên node đó
- IRSA cấp quyền per-pod, per-service-account → granular control
```

---

## 🔗 Điều Hướng

- **Trước:** [2-saml-federation.md](2-saml-federation.md) — SAML 2.0 Federation
- **Tiếp theo:** [4-cognito-user-pools.md](4-cognito-user-pools.md) — Cognito cho ứng dụng
- **Liên quan:** [5-cross-account-roles.md](5-cross-account-roles.md) — Cross-account access

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
