# 03 — CD & Deployments (Phân Phối Liên Tục & Triển Khai)

> CD — Continuous Delivery / Continuous Deployment — Phân Phối Liên Tục / Triển Khai Liên Tục là quá trình tự động đưa code từ repository đến môi trường production một cách an toàn, nhanh chóng và có kiểm soát.

## 📚 Mục Lục Topic Này

| File | Nội Dung | Độ Ưu Tiên |
|---|---|---|
| [1-environments.md](1-environments.md) | GitHub Environments, protection rules, manual approvals | ⭐⭐⭐ |
| [2-deployment-strategies.md](2-deployment-strategies.md) | Rolling, Blue/Green, Canary — chiến lược giảm rủi ro | ⭐⭐⭐ |
| [3-docker-deployments.md](3-docker-deployments.md) | Docker build, push registry, container deployment | ⭐⭐⭐ |
| [4-kubernetes-deployments.md](4-kubernetes-deployments.md) | kubectl, Helm, Kustomize, GitOps | ⭐⭐⭐ |
| [5-aws-deployments.md](5-aws-deployments.md) | ECS, EKS, Lambda, S3, CloudFront | ⭐⭐ |
| [6-gcp-deployments.md](6-gcp-deployments.md) | GKE, Cloud Run, Artifact Registry | ⭐⭐ |
| [7-azure-deployments.md](7-azure-deployments.md) | AKS, Azure Container Registry, App Service | ⭐⭐ |

---

## 🎯 CD là gì và tại sao quan trọng?

### Continuous Delivery vs Continuous Deployment

```
Continuous Integration (CI)
  └─> Code commit → Build → Test → Artifact ✅

Continuous Delivery (CD — Phân Phối Liên Tục)
  └─> CI ✅ → Deploy to Staging → Manual Approval → Deploy to Production

Continuous Deployment (CD — Triển Khai Liên Tục)
  └─> CI ✅ → Deploy to Staging → Automated Tests → Deploy to Production (auto)
```

**Sự khác biệt then chốt:**
- **Continuous Delivery**: Mỗi commit *có thể* được triển khai lên production, nhưng cần approval (phê duyệt) của người
- **Continuous Deployment**: Mỗi commit *tự động* được triển khai lên production nếu vượt qua tất cả tests

### Tại sao CD quan trọng?

| Vấn Đề Khi Không Có CD | Giải Pháp CD Mang Lại |
|---|---|
| Deploy thủ công, dễ sai sót | Quy trình chuẩn hóa, tự động hóa |
| Release lớn, rủi ro cao | Release nhỏ, thường xuyên, dễ rollback |
| Không biết code đang chạy version nào | Audit trail (nhật ký kiểm toán) rõ ràng |
| Thời gian deploy lâu, gây downtime | Zero-downtime deployments |
| Thiếu visibility (khả năng quan sát) | Dashboard trạng thái deployment |

---

## 🏗️ Kiến Trúc CD Pipeline Điển Hình

```
┌─────────────────────────────────────────────────────────┐
│                    CD PIPELINE                          │
│                                                         │
│  PR Merged    Build &      Deploy to    Deploy to       │
│  to main  →   Package  →   Staging   →  Production      │
│               Artifact     (auto)      (with approval)  │
│                                                         │
│  [CI Pass] → [Docker] → [Staging Env] → [Prod Env]      │
│              [Push]    [Smoke Tests]   [Health Check]   │
│              [Registry] [Auto]         [Manual Gate]    │
└─────────────────────────────────────────────────────────┘
```

### Các thành phần chính của CD Pipeline

```yaml
# Cấu trúc CD pipeline chuẩn
name: CD Pipeline

on:
  push:
    branches: [main]          # Trigger khi merge vào main

jobs:
  build:                       # Bước 1: Build và đóng gói
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

  deploy-staging:              # Bước 2: Deploy lên staging
    needs: build
    environment: staging       # Liên kết với GitHub Environment

  integration-tests:           # Bước 3: Smoke/Integration tests
    needs: deploy-staging

  deploy-production:           # Bước 4: Deploy lên production
    needs: integration-tests
    environment: production    # Có protection rules — cần approval
```

---

## 🌍 GitHub Environments — Môi Trường Deployment

GitHub Environments là tính năng cho phép bạn định nghĩa các môi trường deployment (development, staging, production) với các quy tắc bảo vệ riêng biệt.

### Tại sao cần Environments?

```
Không có Environments:
  push to main → deploy everywhere → ai biết được!

Có Environments:
  push to main
    → deploy staging  (auto, mọi commit)
    → wait for approval
    → deploy production (controlled, audited)
```

### Tạo Environment trên GitHub

```
Repository → Settings → Environments → New environment

Cấu hình cho Production:
  ✅ Required reviewers: [senior-engineer, team-lead]
  ✅ Wait timer: 5 minutes (buffer để cancel nếu cần)
  ✅ Deployment branches: main only
  ✅ Environment secrets: PROD_DB_URL, PROD_API_KEY
```

### Sử dụng Environment trong workflow

```yaml
jobs:
  deploy-production:
    environment:
      name: production
      url: https://myapp.com    # URL hiển thị trong GitHub UI
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        env:
          DB_URL: ${{ secrets.PROD_DB_URL }}    # Environment secret
        run: ./deploy.sh
```

---

## 🚀 Deployment Strategies Tóm Tắt

### 1. Rolling Deployment (Triển Khai Cuộn)

```
Old: [v1][v1][v1][v1]
     ↓
Mid: [v2][v2][v1][v1]   ← dần thay thế
     ↓
New: [v2][v2][v2][v2]
```

**Ưu điểm**: Đơn giản, ít tốn tài nguyên
**Nhược điểm**: Có thể có cả v1 và v2 cùng chạy

### 2. Blue/Green Deployment

```
Blue (Production):  [v1][v1][v1] ← traffic 100%
Green (Standby):    [v2][v2][v2]

Switch:
Blue (Standby):     [v1][v1][v1]
Green (Production): [v2][v2][v2] ← traffic 100%
```

**Ưu điểm**: Rollback (hoàn nguyên) tức thì, zero downtime
**Nhược điểm**: Cần gấp đôi tài nguyên

### 3. Canary Deployment (Triển Khai Thử Nghiệm)

```
Production:  [v1][v1][v1][v1][v1][v1][v1][v1][v1][v2]
              90%                                    10%
             ↓ (nếu ổn)
Production:  [v2][v2][v2][v2][v2][v2][v2][v2][v2][v2]
```

**Ưu điểm**: Giảm rủi ro, test với traffic thực
**Nhược điểm**: Phức tạp hơn, cần monitoring tốt

---

## 🐳 Container & Cloud Deployments

### Docker Workflow Cơ Bản

```yaml
- name: Build and push Docker image
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: |
      ghcr.io/${{ github.repository }}:${{ github.sha }}
      ghcr.io/${{ github.repository }}:latest
```

### Kubernetes Deployment

```yaml
- name: Deploy to Kubernetes
  run: |
    kubectl set image deployment/myapp \
      myapp=ghcr.io/myorg/myapp:${{ github.sha }}
    kubectl rollout status deployment/myapp
```

### Cloud Provider Actions Phổ Biến

| Cloud | Action | Mục Đích |
|---|---|---|
| AWS | `aws-actions/configure-aws-credentials` | Xác thực AWS qua OIDC |
| AWS | `aws-actions/amazon-ecs-deploy-task-definition` | Deploy lên ECS |
| GCP | `google-github-actions/auth` | Xác thực GCP qua Workload Identity |
| GCP | `google-github-actions/deploy-cloudrun` | Deploy lên Cloud Run |
| Azure | `azure/login` | Đăng nhập Azure |
| Azure | `azure/aks-set-context` | Kết nối AKS cluster |

---

## 🔒 Security trong CD

### OIDC — OpenID Connect (Xác Thực Không Cần Long-lived Credentials)

Thay vì lưu AWS_ACCESS_KEY_ID vào secrets (tồn tại mãi mãi, có thể bị lộ), dùng OIDC để lấy temporary credentials (thông tin xác thực tạm thời):

```yaml
permissions:
  id-token: write    # Bắt buộc để dùng OIDC
  contents: read

- name: Configure AWS credentials via OIDC
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
    aws-region: us-east-1
    # Không cần AWS_ACCESS_KEY_ID hay AWS_SECRET_ACCESS_KEY!
```

### Least Privilege (Quyền Tối Thiểu) cho Deployment

```yaml
permissions:
  contents: read         # Chỉ đọc code
  packages: write        # Push Docker images lên GHCR
  id-token: write        # OIDC token
  deployments: write     # Cập nhật deployment status
  # Không cần actions, issues, pull-requests, etc.
```

---

## 📊 Rollback — Hoàn Nguyên Khi Có Sự Cố

### Manual Rollback qua GitHub UI

```
Repository → Actions → Workflows → CD Pipeline
→ Chọn commit trước đó
→ Re-run all jobs
```

### Automated Rollback trong Workflow

```yaml
deploy-production:
  steps:
    - name: Deploy
      id: deploy
      run: ./deploy.sh ${{ env.IMAGE_TAG }}

    - name: Health check
      id: health
      run: ./health-check.sh https://myapp.com

    - name: Rollback on failure
      if: failure() && steps.deploy.outcome == 'success'
      run: ./rollback.sh ${{ env.PREVIOUS_TAG }}
```

### Kubernetes Rollback

```bash
# Xem lịch sử deployment
kubectl rollout history deployment/myapp

# Rollback về revision trước
kubectl rollout undo deployment/myapp

# Rollback về revision cụ thể
kubectl rollout undo deployment/myapp --to-revision=3
```

---

## 📋 CD Pipeline Template Đầy Đủ

```yaml
name: CD — Build, Test, Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ── 1. Build Docker Image ──────────────────────────────
  build:
    name: Build & Push Image
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ── 2. Deploy to Staging ───────────────────────────────
  deploy-staging:
    name: Deploy → Staging
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_STAGING_ROLE_ARN }}
          aws-region: us-east-1

      - name: Deploy to ECS Staging
        run: |
          aws ecs update-service \
            --cluster staging-cluster \
            --service myapp \
            --force-new-deployment

      - name: Wait for deployment
        run: |
          aws ecs wait services-stable \
            --cluster staging-cluster \
            --services myapp

  # ── 3. Smoke Tests ─────────────────────────────────────
  smoke-tests:
    name: Smoke Tests
    needs: deploy-staging
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Run smoke tests
        run: |
          curl -f https://staging.myapp.com/health
          npm run test:smoke -- --env staging

  # ── 4. Deploy to Production ────────────────────────────
  deploy-production:
    name: Deploy → Production
    needs: smoke-tests
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_PROD_ROLE_ARN }}
          aws-region: us-east-1

      - name: Deploy to ECS Production
        run: |
          aws ecs update-service \
            --cluster prod-cluster \
            --service myapp \
            --force-new-deployment

      - name: Verify deployment health
        run: |
          aws ecs wait services-stable \
            --cluster prod-cluster \
            --services myapp
          curl -f https://myapp.com/health

      - name: Notify on success
        if: success()
        run: |
          echo "✅ Deployed ${{ github.sha }} to production"
```

---

## 🔗 Điều Hướng

| ← Trước | Hiện Tại | Tiếp Theo → |
|---|---|---|
| [02-ci-pipeline/](../02-ci-pipeline/) | 03-cd-deployments/ | [04-secrets-variables/](../04-secrets-variables/) |

**Trong topic này:**
- [1. Environments](1-environments.md) — Bắt đầu tại đây
- [2. Deployment Strategies](2-deployment-strategies.md)
- [3. Docker Deployments](3-docker-deployments.md)
- [4. Kubernetes Deployments](4-kubernetes-deployments.md)
- [5. AWS Deployments](5-aws-deployments.md)
- [6. GCP Deployments](6-gcp-deployments.md)
- [7. Azure Deployments](7-azure-deployments.md)

---

**Cập Nhật Lần Cuối:** 2026-05-11
