# GitHub Actions & Kubernetes — CI/CD Pipeline Thực Chiến

> Xây dựng pipeline CI/CD hoàn chỉnh với GitHub Actions: từ build và test code, scan image, đến deploy lên Kubernetes với OIDC authentication (xác thực OIDC), environment protection (bảo vệ môi trường) và secret management (quản lý bí mật) an toàn.

## Mục Lục

1. [Kiến Trúc Pipeline CI/CD](#kiến-trúc-pipeline-cicd)
2. [GitHub Actions Cơ Bản](#github-actions-cơ-bản)
3. [CI Pipeline — Build, Test, Scan](#ci-pipeline--build-test-scan)
4. [CD Pipeline — Deploy lên Kubernetes](#cd-pipeline--deploy-lên-kubernetes)
5. [OIDC Authentication — Không Cần Long-Lived Credentials](#oidc-authentication--không-cần-long-lived-credentials)
6. [Secret Management Trong Pipeline](#secret-management-trong-pipeline)
7. [Environments và Protection Rules](#environments-và-protection-rules)
8. [Reusable Workflows — Tái Sử Dụng Workflow](#reusable-workflows--tái-sử-dụng-workflow)
9. [Self-Hosted Runner Trong Kubernetes](#self-hosted-runner-trong-kubernetes)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kiến Trúc Pipeline CI/CD

```
Code Change
    │
    ├── Pull Request → CI Pipeline
    │                     │
    │                     ├── Lint & Format check
    │                     ├── Unit Tests
    │                     ├── Integration Tests
    │                     ├── SAST (Static Application Security Testing)
    │                     ├── Build Docker Image
    │                     ├── Image Vulnerability Scan (Trivy)
    │                     └── Push Image với tag: pr-<number>-<sha>
    │
    └── Merge to main → CD Pipeline
                            │
                            ├── Build & Push Image: main-<sha>
                            │
                            ├── Deploy to DEV (auto)
                            │   └── helm upgrade --install (hoặc update Git tag cho Flux/ArgoCD)
                            │
                            ├── Run Smoke Tests
                            │
                            └── Deploy to STAGING (auto)
                                    │
                                    └── Tag release → Deploy to PROD (manual approval)
```

---

## GitHub Actions Cơ Bản

### Cấu Trúc Workflow File

```yaml
# .github/workflows/ci.yaml
name: CI Pipeline                     # tên workflow hiển thị trên GitHub

on:                                   # sự kiện trigger
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:                  # trigger thủ công từ GitHub UI

env:                                  # biến môi trường dùng trong toàn workflow
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}  # my-org/my-app

jobs:
  test:                               # job ID
    name: Run Tests                   # tên hiển thị
    runs-on: ubuntu-latest            # loại runner
    timeout-minutes: 15               # timeout để tránh stuck

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.22"
          cache: true                 # cache module dependencies

      - name: Run tests
        run: go test ./... -v -race -coverprofile=coverage.out

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: coverage.out
```

### Contexts và Expressions (Ngữ Cảnh và Biểu Thức)

```yaml
steps:
  - name: Show context info
    run: |
      echo "Repository: ${{ github.repository }}"    # my-org/my-app
      echo "Branch: ${{ github.ref_name }}"          # main
      echo "SHA: ${{ github.sha }}"                  # abc123def456...
      echo "SHA short: ${{ github.sha }}"
      echo "Actor: ${{ github.actor }}"              # tên user trigger
      echo "Event: ${{ github.event_name }}"         # push, pull_request...
      echo "Run ID: ${{ github.run_id }}"
      echo "Run number: ${{ github.run_number }}"

  # Conditional step — chỉ chạy trên main branch
  - name: Deploy (main only)
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    run: echo "Deploying..."

  # Matrix strategy — chạy nhiều version song song
  strategy:
    matrix:
      go-version: ["1.21", "1.22"]
      os: [ubuntu-latest, macos-latest]
    fail-fast: false    # không dừng nếu một matrix fail
```

---

## CI Pipeline — Build, Test, Scan

### Pipeline CI Hoàn Chỉnh

```yaml
# .github/workflows/ci.yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # Job 1: Lint và Unit Test
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version-file: go.mod   # đọc version từ go.mod

      - name: Lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: latest

      - name: Unit Tests
        run: go test ./... -v -count=1 -coverprofile=coverage.out

      - name: Upload Test Results
        if: always()               # chạy kể cả khi test fail
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: coverage.out

  # Job 2: Build và Scan Image
  build:
    name: Build & Scan Image
    runs-on: ubuntu-latest
    needs: test                    # chờ test pass trước
    permissions:
      contents: read
      packages: write              # quyền push lên GitHub Container Registry
      security-events: write       # quyền upload SARIF scan results

    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GHCR (GitHub Container Registry)
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}  # token tự động, không cần tạo thủ công

      - name: Extract Docker metadata (label, tag)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            # Tag với branch-sha: main-abc123
            type=ref,event=branch,suffix=-${{ github.sha }}
            # Tag với semver nếu là git tag: v1.2.3
            type=semver,pattern={{version}}
            # Tag latest chỉ cho main branch
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and Push Image
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha      # cache layer trong GitHub Actions cache
          cache-to: type=gha,mode=max

      - name: Run Trivy Vulnerability Scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: "1"            # fail pipeline nếu có CRITICAL vulnerability

      - name: Upload Trivy Results to GitHub Security tab
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif
```

---

## CD Pipeline — Deploy lên Kubernetes

### CD Pipeline Với GitOps (Cập Nhật Image Tag Trong Config Repo)

```yaml
# .github/workflows/cd.yaml
name: CD — Update Image Tag

on:
  push:
    branches: [main]
    tags: ["v*"]

jobs:
  update-image-tag:
    name: Update Image Tag in Config Repo
    runs-on: ubuntu-latest
    needs: [build]               # chờ build job xong

    steps:
      - name: Checkout config repo (tách biệt với app repo)
        uses: actions/checkout@v4
        with:
          repository: my-org/my-cluster-config
          token: ${{ secrets.CONFIG_REPO_TOKEN }}  # PAT có quyền push config repo
          path: config-repo

      - name: Update image tag in Helm values
        run: |
          cd config-repo
          # Dùng yq (YAML query tool) để update tag trong values.yaml
          yq e ".image.tag = \"${{ github.sha }}\"" -i apps/my-app/values-staging.yaml

      - name: Commit and push
        run: |
          cd config-repo
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add apps/my-app/values-staging.yaml
          git commit -m "chore: update my-app to ${{ github.sha }}" || exit 0  # exit 0 nếu nothing to commit
          git push
```

### CD Pipeline Với Helm (Direct Deploy)

```yaml
# CD không dùng GitOps — deploy trực tiếp bằng Helm
# Cách này đơn giản hơn nhưng không có GitOps benefits
name: CD — Direct Helm Deploy

on:
  push:
    branches: [main]

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment: staging         # cần approval nếu environment có protection rule
    needs: [build]

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC — không cần long-lived key)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-staging
          aws-region: ap-southeast-1

      - name: Setup kubectl
        uses: azure/setup-kubectl@v4
        with:
          version: "v1.29.0"

      - name: Update kubeconfig for EKS
        run: |
          aws eks update-kubeconfig \
            --name staging-cluster \
            --region ap-southeast-1

      - name: Deploy with Helm
        run: |
          helm upgrade --install my-app ./helm/my-app \
            --namespace staging \
            --create-namespace \
            --set image.tag=${{ github.sha }} \
            --set image.repository=ghcr.io/${{ github.repository }} \
            --values helm/my-app/values-staging.yaml \
            --atomic \
            --timeout 5m \
            --wait
```

---

## OIDC Authentication — Không Cần Long-Lived Credentials

**OIDC (OpenID Connect — Kết Nối OpenID)** cho phép GitHub Actions xác thực với cloud provider (AWS, GCP, Azure) mà không cần lưu long-lived credentials (thông tin xác thực tồn tại lâu dài như access key) trong GitHub Secrets.

```
Mô hình truyền thống (nguy hiểm):
GitHub Secrets: AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY
→ Credential tồn tại mãi, bị lộ nếu repo bị compromise

Mô hình OIDC (an toàn):
GitHub Actions → request token ngắn hạn từ GitHub OIDC Provider
             → gửi token tới AWS STS (Security Token Service)
             → AWS xác thực token với GitHub public key
             → trả về temporary credentials (tồn tại 1 giờ)
```

### Cấu Hình OIDC Với AWS

```json
// Bước 1: Tạo IAM OIDC Provider trong AWS (làm một lần)
{
  "ProviderURL": "https://token.actions.githubusercontent.com",
  "ThumbprintList": ["6938fd4d98bab03faadb97b34396831e3780aea1"],
  "ClientIDList": ["sts.amazonaws.com"]
}
```

```json
// Bước 2: Tạo IAM Role với trust policy
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          // Giới hạn chỉ repo và branch cụ thể mới có thể assume role
          "token.actions.githubusercontent.com:sub": "repo:my-org/my-app:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

```yaml
# Bước 3: Dùng trong workflow
jobs:
  deploy:
    permissions:
      id-token: write      # BẮT BUỘC để request OIDC token
      contents: read

    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-deploy
          aws-region: ap-southeast-1
          # Không có access-key-id hay secret-access-key!
```

---

## Secret Management Trong Pipeline

### Hierarchy của Secret (Phân Cấp Bí Mật)

```
GitHub Secrets / Variables Hierarchy (từ cao đến thấp):
1. Environment secrets    → chỉ dùng trong job với environment cụ thể
2. Repository secrets     → dùng trong tất cả workflow trong repo
3. Organization secrets   → dùng trong nhiều repo trong organization
```

### Best Practices Cho Secret

```yaml
# KHÔNG BAO GIỜ:
env:
  DB_PASSWORD: "hardcoded-password"           # ❌ hardcode trong YAML
  API_KEY: ${{ vars.SOME_CONFIG }}           # ❌ dùng variable cho sensitive data

# NÊN LÀM:
steps:
  - name: Use secrets properly
    env:
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}  # ✓ từ GitHub Secrets
      API_KEY: ${{ secrets.API_KEY }}
    run: |
      # Secret được inject vào env var, không hiển thị trong log
      echo "Running with API key..."
      # ❌ KHÔNG in secret ra log: echo $API_KEY
      # ✓ GitHub tự động mask nếu secret accidentally bị log

  # Dùng OIDC thay vì secrets cho cloud credentials
  - name: AWS Authentication (OIDC - best practice)
    uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789:role/my-role  # không có credentials!
      aws-region: ap-southeast-1

  # Lấy secret từ AWS Secrets Manager trong pipeline
  - name: Get secrets from Vault
    uses: aws-actions/aws-secretsmanager-get-secrets@v2
    with:
      secret-ids: |
        MY_DB_PASSWORD,arn:aws:secretsmanager:ap-southeast-1:123:secret:prod/db
      parse-json-secrets: true
```

---

## Environments và Protection Rules

**Environments (Môi Trường)** trong GitHub Actions cho phép cấu hình protection rules (quy tắc bảo vệ) — yêu cầu approval trước khi deploy lên production.

```yaml
# Cấu hình trong GitHub UI: Settings > Environments

# Production environment protection rules:
# ├── Required reviewers: [senior-devops, tech-lead]
# ├── Wait timer: 5 minutes (cho thời gian cancel nếu deploy sai)
# ├── Deployment branches: only "main" and "v*" tags
# └── Environment secrets: PROD_DB_PASSWORD, PROD_KUBECONFIG

# Workflow dùng environment
jobs:
  deploy-production:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://my-app.prod.example.com   # URL hiển thị sau deploy
    # Job này sẽ DỪNG và chờ approval trước khi chạy

    steps:
      - name: Deploy to production
        run: |
          helm upgrade --install my-app ./helm/my-app \
            --values helm/my-app/values-prod.yaml \
            --set image.tag=${{ github.sha }} \
            --atomic --timeout 10m
```

---

## Reusable Workflows — Tái Sử Dụng Workflow

**Reusable Workflows (Workflow Tái Sử Dụng)** cho phép định nghĩa workflow một lần và gọi từ nhiều workflow khác — tránh DRY violation trong pipeline code.

```yaml
# .github/workflows/reusable-helm-deploy.yaml
name: Reusable Helm Deploy

on:
  workflow_call:          # trigger khi được gọi từ workflow khác
    inputs:
      environment:
        required: true
        type: string        # "staging" hoặc "production"
      image-tag:
        required: true
        type: string
      helm-values-file:
        required: false
        type: string
        default: "values.yaml"
    secrets:
      KUBECONFIG_DATA:
        required: true
    outputs:
      deploy-url:
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    outputs:
      url: ${{ steps.deploy.outputs.url }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup kubectl
        run: |
          echo "${{ secrets.KUBECONFIG_DATA }}" | base64 -d > ~/.kube/config

      - name: Helm Deploy
        id: deploy
        run: |
          helm upgrade --install my-app ./helm/my-app \
            --namespace ${{ inputs.environment }} \
            --values helm/my-app/${{ inputs.helm-values-file }} \
            --set image.tag=${{ inputs.image-tag }} \
            --atomic --wait
          echo "url=https://my-app.${{ inputs.environment }}.example.com" >> $GITHUB_OUTPUT
```

```yaml
# Gọi reusable workflow từ workflow chính
# .github/workflows/cd.yaml
jobs:
  deploy-staging:
    uses: ./.github/workflows/reusable-helm-deploy.yaml    # gọi workflow
    with:
      environment: staging
      image-tag: ${{ needs.build.outputs.image-tag }}
      helm-values-file: values-staging.yaml
    secrets:
      KUBECONFIG_DATA: ${{ secrets.STAGING_KUBECONFIG }}

  deploy-production:
    needs: deploy-staging
    uses: ./.github/workflows/reusable-helm-deploy.yaml
    with:
      environment: production
      image-tag: ${{ needs.build.outputs.image-tag }}
      helm-values-file: values-prod.yaml
    secrets:
      KUBECONFIG_DATA: ${{ secrets.PROD_KUBECONFIG }}
```

---

## Self-Hosted Runner Trong Kubernetes

**Self-Hosted Runner (Runner Tự Quản Lý)** giúp CI/CD chạy trong cluster nội bộ, không cần expose cluster ra internet. Phổ biến nhất là dùng **Actions Runner Controller (ARC)**.

```bash
# Cài Actions Runner Controller bằng Helm
helm install arc \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
  -n arc-systems --create-namespace

# Cài Runner Scale Set (tự động scale runner theo demand)
helm install arc-runner-set \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  -n arc-runners --create-namespace \
  --set githubConfigUrl="https://github.com/my-org" \
  --set githubConfigSecret.github_token="ghp_xxx"
```

```yaml
# Dùng self-hosted runner trong workflow
jobs:
  build-internal:
    runs-on: arc-runner-set    # chỉ định runner label
    steps:
      - uses: actions/checkout@v4
      - name: Build internal tool
        run: make build
        # Runner này chạy trong cluster, có thể:
        # - Truy cập internal service không expose ra internet
        # - Mount PVC để cache build artifact
        # - Dùng IAM Role của Node (IRSA) để access AWS service
```

---

## Câu Hỏi Phỏng Vấn

**Tại sao dùng OIDC thay vì lưu AWS credentials trong GitHub Secrets?**

> GitHub Secrets với AWS credentials (ACCESS_KEY_ID + SECRET_ACCESS_KEY) là long-lived credentials — nếu secret bị lộ do repo bị compromise, token hijacked, hay insider threat, attacker có quyền truy cập AWS vô thời hạn đến khi credentials bị rotate. OIDC giải quyết bằng cách phát hành short-lived token (token ngắn hạn) tự động expire sau 1 giờ. Ngay cả khi token bị chặn, nó cũng hết hạn rất nhanh. Thêm vào đó, OIDC trust policy có thể giới hạn chỉ repo cụ thể và branch cụ thể mới được assume role — không phải toàn bộ organization.

**Làm thế nào để đảm bảo image tag là immutable (bất biến) trong pipeline?**

> Không bao giờ dùng `latest` tag trong production deployment — `latest` luôn trỏ đến image mới nhất được push, không thể rollback chính xác. Thay vào đó dùng commit SHA (`abc123def`) làm tag: bất biến, 1-1 với commit, có thể trace ngược code. Trong GitHub Actions: `--set image.tag=${{ github.sha }}`. Hoặc dùng Semantic Versioning (quản lý phiên bản ngữ nghĩa) kết hợp với Git tag: `v1.2.3`. Thêm vào đó, cần đảm bảo registry có **image immutability** — chặn push lên tag đã tồn tại. AWS ECR có tùy chọn `imageTagMutability: IMMUTABLE`. GitHub Container Registry cũng hỗ trợ điều này.

**Pipeline của bạn xử lý secret rotation thế nào mà không cần update code?**

> Secrets nên được lưu trong secret management system (hệ thống quản lý bí mật) như AWS Secrets Manager, HashiCorp Vault, hay Azure Key Vault — không trong code hay GitHub Secrets tĩnh. Khi cần rotate, chỉ update secret trong Vault/AWS; ứng dụng đọc secret tại runtime mỗi lần start (không cache lâu). Trong pipeline CI/CD, dùng OIDC để lấy AWS credentials tạm thời, rồi dùng credentials đó để đọc secret từ Secrets Manager thay vì hardcode. Với Kubernetes deployment, External Secrets Operator tự động sync từ Vault/AWS và tạo Kubernetes Secret — khi secret rotate, operator update Secret trong cluster mà không cần redeploy app (với volume mount; với env var thì cần restart Pod).
