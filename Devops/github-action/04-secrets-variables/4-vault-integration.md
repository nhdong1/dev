# 🏛️ HashiCorp Vault Integration — Tích Hợp Quản Lý Bí Mật Tập Trung

> HashiCorp Vault là giải pháp quản lý secrets tập trung (centralized secret management) dành cho enterprise. Khi tích hợp với GitHub Actions, Vault cung cấp dynamic secrets (bí mật động tự hủy), fine-grained access control (kiểm soát truy cập chi tiết) và audit log toàn diện.

---

## 📚 Mục Lục

1. [Tại Sao Dùng Vault?](#tại-sao-dùng-vault)
2. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
3. [JWT Authentication Method](#jwt-authentication-method)
4. [Setup Vault Cho GitHub Actions](#setup-vault-cho-github-actions)
5. [Workflow Tích Hợp Vault](#workflow-tích-hợp-vault)
6. [Dynamic Secrets — Bí Mật Động](#dynamic-secrets--bí-mật-động)
7. [Vault Policies — Chính Sách Vault](#vault-policies--chính-sách-vault)
8. [Kubernetes Vault Integration](#kubernetes-vault-integration)
9. [Best Practices](#best-practices)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Dùng Vault?

### So Sánh GitHub Secrets vs HashiCorp Vault

| Tiêu Chí | GitHub Secrets | HashiCorp Vault |
|---|---|---|
| **Dynamic Secrets** (Bí Mật Động) | ❌ | ✅ Tự tạo & hủy |
| **Cross-platform** | Chỉ GitHub | Mọi CI/CD system |
| **Secret versioning** | ❌ | ✅ Version history |
| **Fine-grained policies** | Cơ bản | ✅ Rất chi tiết |
| **Audit log** | Cơ bản | ✅ Toàn diện |
| **Lease & renewal** | ❌ | ✅ TTL tự động |
| **Database creds** | Manual | ✅ Tự tạo/thu hồi |
| **PKI (Certificate)** | ❌ | ✅ Có |
| **Độ phức tạp** | Thấp | Cao |
| **Chi phí** | Miễn phí | OSS miễn phí, Enterprise có phí |

### Khi Nào Nên Dùng Vault?

```
Dùng GitHub Secrets khi:
- Dự án nhỏ đến vừa
- Chỉ dùng GitHub Actions
- Ít secrets cần quản lý
- Không cần dynamic secrets

Dùng HashiCorp Vault khi:
- Enterprise với nhiều teams và platforms
- Cần dynamic secrets (DB creds tự thu hồi)
- Compliance yêu cầu centralized secret management
- Cần secret sharing giữa GitHub Actions, Jenkins, Kubernetes
- Cần secret versioning và rollback
```

---

## Kiến Trúc Tổng Quan

```
┌────────────────────────────────────────────────────────────┐
│               GitHub Actions + Vault Architecture          │
│                                                            │
│  GitHub                         HashiCorp Vault            │
│  ┌──────────────────┐            ┌────────────────────┐    │
│  │   Workflow Run   │            │                    │    │
│  │                  │            │  Secrets Engines:  │    │
│  │  1. Request      │───JWT────>│  - KV v2 (static)  │    │
│  │     Vault Token  │            │  - Database        │    │
│  │                  │<──Token───│  - AWS             │    │
│  │  2. Read Secrets │            │  - PKI             │    │
│  │     with Token   │──Token──> │                    │    │
│  │                  │<──Secrets─│  Auth Methods:     │    │
│  │  3. Use Secrets  │            │  - JWT/OIDC        │    │
│  │     in Steps     │            │  - AppRole         │    │
│  └──────────────────┘            └────────────────────┘    │
└────────────────────────────────────────────────────────────┘
```

---

## JWT Authentication Method

GitHub Actions dùng JWT token (giống OIDC) để xác thực với Vault. Đây là cách khuyến nghị — không cần lưu Vault credentials trong GitHub.

### Cách Hoạt Động

```
1. GitHub Actions tạo OIDC JWT token
2. Gửi JWT đến Vault's JWT auth endpoint
3. Vault xác minh JWT với GitHub's OIDC public keys
4. Vault kiểm tra bound_claims (điều kiện ràng buộc)
5. Nếu hợp lệ → Vault trả về token với policies đính kèm
6. Workflow dùng Vault token để đọc secrets
```

### Cấu Hình Vault — JWT Auth Method

```bash
# 1. Bật JWT auth method
vault auth enable jwt

# 2. Cấu hình với GitHub OIDC provider
vault write auth/jwt/config \
  oidc_discovery_url="https://token.actions.githubusercontent.com" \
  bound_issuer="https://token.actions.githubusercontent.com"

# 3. Tạo role cho GitHub Actions
vault write auth/jwt/role/github-actions \
  role_type="jwt" \
  user_claim="actor" \
  bound_claims_type="glob" \
  bound_claims='{
    "repository": "owner/repo",
    "ref": "refs/heads/main"
  }' \
  bound_audiences="https://github.com/owner" \
  ttl="1h" \
  max_ttl="4h" \
  policies="github-actions-policy"
```

---

## Setup Vault Cho GitHub Actions

### Bước 1: Tạo Secrets Trong Vault

```bash
# Bật KV v2 secrets engine (Key-Value version 2)
vault secrets enable -path=secret kv-v2

# Tạo secrets cho ứng dụng
vault kv put secret/myapp/production \
  db_password="super-secret-password" \
  api_key="my-api-key-value" \
  jwt_secret="random-jwt-secret"

# Tạo secrets cho staging
vault kv put secret/myapp/staging \
  db_password="staging-password" \
  api_key="staging-api-key" \
  jwt_secret="staging-jwt-secret"

# Kiểm tra
vault kv get secret/myapp/production
```

### Bước 2: Tạo Vault Policy

```hcl
# File: github-actions-policy.hcl
# Policy (Chính Sách) định nghĩa quyền truy cập

# Read-only access cho secrets của app
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}

# Không có quyền xóa hay update
path "secret/data/myapp/*" {
  denied_parameters = {
    "*" = []
  }
}

# Cho phép renew token của chính nó
path "auth/token/renew-self" {
  capabilities = ["update"]
}

# Cho phép revoke token sau khi dùng xong
path "auth/token/revoke-self" {
  capabilities = ["update"]
}
```

```bash
# Áp dụng policy
vault policy write github-actions-policy github-actions-policy.hcl
```

### Bước 3: Cấu Hình JWT Role Chi Tiết

```bash
# Role cho production deployment — chỉ main branch + production environment
vault write auth/jwt/role/production-deployer \
  role_type="jwt" \
  user_claim="actor" \
  bound_audiences="https://github.com/my-org" \
  bound_claims_type="glob" \
  bound_claims='{
    "repository": "my-org/my-repo",
    "ref": "refs/heads/main",
    "environment": "production"
  }' \
  ttl="30m" \
  max_ttl="1h" \
  policies="production-deploy-policy"

# Role cho staging — chỉ develop branch
vault write auth/jwt/role/staging-deployer \
  role_type="jwt" \
  user_claim="actor" \
  bound_audiences="https://github.com/my-org" \
  bound_claims='{
    "repository": "my-org/my-repo",
    "ref": "refs/heads/develop",
    "environment": "staging"
  }' \
  ttl="30m" \
  policies="staging-deploy-policy"
```

---

## Workflow Tích Hợp Vault

### Sử Dụng `hashicorp/vault-action`

```yaml
name: Deploy with Vault Secrets

on:
  push:
    branches: [main]

permissions:
  id-token: write   # Cần để tạo OIDC token cho Vault JWT auth
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Retrieve secrets from Vault
        uses: hashicorp/vault-action@v3
        with:
          url: https://vault.company.com
          method: jwt
          path: jwt
          role: production-deployer
          # Cú pháp: "path field | ENV_VAR_NAME"
          secrets: |
            secret/data/myapp/production db_password | DB_PASSWORD ;
            secret/data/myapp/production api_key | API_KEY ;
            secret/data/myapp/production jwt_secret | JWT_SECRET

      - name: Deploy application
        run: |
          echo "Deploying with Vault secrets..."
          # DB_PASSWORD, API_KEY, JWT_SECRET đã được set tự động
          ./scripts/deploy.sh \
            --db-url "postgresql://user:${DB_PASSWORD}@db.prod.com/myapp" \
            --api-key "${API_KEY}"
```

### Đọc Nhiều Secrets Từ Nhiều Paths

```yaml
- name: Read multiple secrets
  uses: hashicorp/vault-action@v3
  with:
    url: ${{ vars.VAULT_URL }}
    method: jwt
    path: jwt
    role: github-actions
    secrets: |
      secret/data/myapp/production db_password | PROD_DB_PASS ;
      secret/data/myapp/production db_user | PROD_DB_USER ;
      secret/data/aws/credentials access_key | AWS_ACCESS_KEY_ID ;
      secret/data/aws/credentials secret_key | AWS_SECRET_ACCESS_KEY ;
      pki/issue/myapp common_name=myapp.prod.com | TLS_CERT
```

### Vault với AppRole Auth (Phương Thức Xác Thực AppRole)

```yaml
# AppRole auth khi không thể dùng JWT/OIDC
- name: Get Vault token via AppRole
  env:
    VAULT_ADDR: ${{ vars.VAULT_URL }}
  run: |
    # role-id và secret-id được lưu trong GitHub Secrets
    VAULT_TOKEN=$(curl -s -X POST \
      "$VAULT_ADDR/v1/auth/approle/login" \
      -H "Content-Type: application/json" \
      -d '{
        "role_id": "${{ secrets.VAULT_ROLE_ID }}",
        "secret_id": "${{ secrets.VAULT_SECRET_ID }}"
      }' | jq -r '.auth.client_token')
    
    echo "VAULT_TOKEN=$VAULT_TOKEN" >> $GITHUB_ENV

- name: Read secrets
  uses: hashicorp/vault-action@v3
  with:
    url: ${{ vars.VAULT_URL }}
    token: ${{ env.VAULT_TOKEN }}
    secrets: |
      secret/data/myapp/production db_password | DB_PASSWORD
```

---

## Dynamic Secrets — Bí Mật Động

Dynamic Secrets là tính năng nổi bật của Vault — tự động tạo credentials có TTL ngắn và thu hồi sau khi hết hạn.

### Database Dynamic Secrets

```bash
# Cấu hình Vault database secrets engine
vault secrets enable database

vault write database/config/my-postgresql \
  plugin_name=postgresql-database-plugin \
  allowed_roles="github-actions" \
  connection_url="postgresql://{{username}}:{{password}}@db.example.com:5432/mydb" \
  username="vault-admin" \
  password="vault-admin-password"

# Tạo role với TTL 1 giờ
vault write database/roles/github-actions \
  db_name=my-postgresql \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"
```

```yaml
# Workflow dùng dynamic database credentials
- name: Get dynamic DB credentials
  uses: hashicorp/vault-action@v3
  with:
    url: ${{ vars.VAULT_URL }}
    method: jwt
    path: jwt
    role: github-actions
    secrets: |
      database/creds/github-actions username | DB_USERNAME ;
      database/creds/github-actions password | DB_PASSWORD

- name: Run database migration
  run: |
    # DB_USERNAME và DB_PASSWORD là credentials tạm thời, tự thu hồi sau 1 giờ
    DATABASE_URL="postgresql://$DB_USERNAME:$DB_PASSWORD@db.prod.com/mydb"
    ./migrate.sh "$DATABASE_URL"
```

### AWS Dynamic Secrets

```bash
# Cấu hình Vault AWS secrets engine
vault secrets enable aws

vault write aws/config/root \
  access_key="AKIA..." \
  secret_key="..." \
  region="us-east-1"

# Tạo role với permissions cụ thể
vault write aws/roles/github-actions-deploy \
  credential_type=iam_user \
  policy_document='{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["ecs:UpdateService", "ecr:*"],
      "Resource": "*"
    }]
  }' \
  default_ttl="1h"
```

```yaml
# Workflow lấy AWS credentials động từ Vault
- name: Get dynamic AWS credentials
  uses: hashicorp/vault-action@v3
  with:
    url: ${{ vars.VAULT_URL }}
    method: jwt
    path: jwt
    role: github-actions
    secrets: |
      aws/creds/github-actions-deploy access_key | AWS_ACCESS_KEY_ID ;
      aws/creds/github-actions-deploy secret_key | AWS_SECRET_ACCESS_KEY

- name: Deploy to AWS
  run: |
    aws ecs update-service --cluster prod --service my-svc --force-new-deployment
  env:
    AWS_DEFAULT_REGION: us-east-1
    # AWS_ACCESS_KEY_ID và AWS_SECRET_ACCESS_KEY đã được set bởi vault-action
```

---

## Vault Policies — Chính Sách Vault

### Policy Theo Môi Trường

```hcl
# production-deploy-policy.hcl
# Chỉ đọc production secrets

path "secret/data/myapp/production" {
  capabilities = ["read"]
}

# Lấy dynamic database credentials
path "database/creds/production-role" {
  capabilities = ["read"]
}

# Lấy dynamic AWS credentials
path "aws/creds/production-deployer" {
  capabilities = ["read"]
}

# Không cho phép truy cập staging hay development
path "secret/data/myapp/staging" {
  capabilities = []
}
```

```hcl
# staging-deploy-policy.hcl
path "secret/data/myapp/staging" {
  capabilities = ["read"]
}

path "database/creds/staging-role" {
  capabilities = ["read"]
}
```

### Kết Hợp Policy Với JWT Bound Claims

```bash
# Production role — rất strict: chỉ main branch + production environment
vault write auth/jwt/role/production \
  bound_claims='{
    "repository": "my-org/my-app",
    "ref": "refs/heads/main",
    "environment": "production",
    "workflow_ref": "my-org/my-app/.github/workflows/deploy-prod.yml@refs/heads/main"
  }' \
  policies="production-deploy-policy" \
  ttl="30m"

# CI role — rộng hơn: mọi branch nhưng chỉ đọc non-sensitive configs
vault write auth/jwt/role/ci-runner \
  bound_claims='{
    "repository": "my-org/my-app"
  }' \
  policies="ci-read-only-policy" \
  ttl="1h"
```

---

## Kubernetes Vault Integration

Khi self-hosted runners chạy trên Kubernetes, có thể dùng Vault Agent Injector — tự động inject secrets vào pods.

### Vault Agent Injector

```yaml
# Kubernetes pod spec với Vault annotations
apiVersion: v1
kind: Pod
metadata:
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "github-runner"
    vault.hashicorp.com/agent-inject-secret-config: "secret/data/myapp/production"
    vault.hashicorp.com/agent-inject-template-config: |
      {{ with secret "secret/data/myapp/production" }}
      DB_PASSWORD={{ .Data.data.db_password }}
      API_KEY={{ .Data.data.api_key }}
      {{ end }}
spec:
  containers:
    - name: github-runner
      image: myorg/github-runner:latest
      # Secrets sẽ được inject vào /vault/secrets/config
```

---

## Best Practices

### 1. Dùng JWT Auth Thay Vì Token-Based Auth

```yaml
# ✅ Tốt — JWT auth, không cần lưu credentials
uses: hashicorp/vault-action@v3
with:
  method: jwt
  role: my-role

# ❌ Tránh — Phải lưu Vault token trong GitHub Secrets
uses: hashicorp/vault-action@v3
with:
  token: ${{ secrets.VAULT_TOKEN }}  # Vẫn là long-lived credential
```

### 2. Luôn Đặt TTL Ngắn

```bash
# Vault role với TTL ngắn nhất có thể
vault write auth/jwt/role/github-actions \
  ttl="30m" \      # TTL của Vault token
  max_ttl="1h"     # Không thể renew quá 1 giờ
```

### 3. Revoke Token Sau Khi Dùng

```yaml
- name: Get secrets from Vault
  id: vault
  uses: hashicorp/vault-action@v3
  with:
    ...
    exportToken: true  # Export VAULT_TOKEN để có thể revoke sau

- name: Use secrets
  run: ./deploy.sh

- name: Revoke Vault token
  if: always()  # Chạy kể cả khi steps trước fail
  run: |
    curl -X POST \
      -H "X-Vault-Token: $VAULT_TOKEN" \
      "${{ vars.VAULT_URL }}/v1/auth/token/revoke-self"
```

### 4. Audit Log Monitoring

```bash
# Bật audit log trong Vault
vault audit enable file file_path=/vault/logs/audit.log

# Audit log ghi lại:
# - Ai (actor, workflow)
# - Đọc secret nào
# - Thời điểm nào
# - Từ IP nào
# - Kết quả (success/failure)
```

---

## Câu Hỏi Phỏng Vấn

### Q: Khi nào nên dùng Vault thay vì GitHub Secrets?

**Trả lời:**
Nên dùng Vault khi:
1. **Multi-platform:** Có nhiều CI/CD systems (GitHub Actions + Jenkins + Argo CD) và cần quản lý secrets tập trung
2. **Dynamic secrets:** Cần database credentials tự tạo và thu hồi (giảm attack window)
3. **Compliance:** SOC2, ISO27001 yêu cầu centralized secret management với audit trail toàn diện
4. **Scale:** Hàng trăm services với thousands of secrets
5. **Secret versioning:** Cần rollback về phiên bản secret trước

### Q: Dynamic secrets trong Vault là gì và lợi ích của nó?

**Trả lời:**
Dynamic secrets là secrets được Vault tạo ra on-demand (theo yêu cầu) với TTL ngắn. Ví dụ: mỗi khi CI pipeline chạy, Vault tạo một database user mới với password ngẫu nhiên, sau 1 giờ tự động thu hồi.

Lợi ích:
- Không có secret nào tồn tại lâu để bị đánh cắp
- Mỗi run có credentials riêng → dễ audit
- Nếu bị lộ, attacker chỉ có cửa sổ nhỏ để khai thác

---

**Điều Hướng:**
- ← [3-oidc.md](3-oidc.md)
- → [5-aws-secrets-manager.md](5-aws-secrets-manager.md)

**Cập Nhật Lần Cuối:** 2026-05-11
