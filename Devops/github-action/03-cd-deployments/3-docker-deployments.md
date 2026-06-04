# 3 — Docker Deployments (Triển Khai với Docker)

> Docker cho phép đóng gói ứng dụng và tất cả dependencies (phụ thuộc) vào một container image (hình ảnh container) bất biến — chạy giống nhau trên mọi môi trường từ laptop đến production.

## 🎯 Docker Workflow trong CI/CD

```
Code Push
    ↓
Build Image          ← docker build
    ↓
Run Tests            ← docker run --rm myapp test
    ↓
Security Scan        ← Trivy, Snyk scan image
    ↓
Push to Registry     ← docker push ghcr.io/myorg/myapp:sha-abc123
    ↓
Deploy Container     ← Kubernetes / ECS / Cloud Run pull image & run
```

---

## 🏗️ Dockerfile Best Practices

### Multi-stage Build (Build Nhiều Giai Đoạn)

```dockerfile
# Stage 1: Builder — chứa tất cả dev tools
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production=false    # Cài cả devDependencies để build
COPY . .
RUN npm run build

# Stage 2: Runner — chỉ chứa runtime cần thiết
FROM node:20-alpine AS runner
WORKDIR /app

# Non-root user (người dùng không có quyền root) — bảo mật
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Chỉ copy artifacts từ builder, không copy source code
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json .

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

**Lợi ích Multi-stage:**
- Final image nhỏ hơn đáng kể (không chứa build tools, test files)
- Bảo mật hơn — không có source code, dev deps trong production image
- Build cache hiệu quả — mỗi stage cache riêng

### Image Tagging (Đánh Dấu Phiên Bản Image)

```
Tránh dùng:  myapp:latest       — không biết đây là version nào
Nên dùng:    myapp:sha-abc1234  — truy ra được commit chính xác
             myapp:v1.2.3       — semantic version rõ ràng
             myapp:main-abc123  — branch + commit
```

---

## 📦 Container Registries (Kho Lưu Trữ Container)

### GHCR — GitHub Container Registry

Tích hợp sẵn với GitHub Actions, không cần cấu hình thêm cho public repos.

```yaml
jobs:
  build-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write    # Cần để push lên GHCR

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}    # Tự động, không cần tạo

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

### AWS ECR — Elastic Container Registry

```yaml
      - name: Configure AWS credentials (OIDC — không cần long-lived keys)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Log in to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push to ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: myapp
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
```

### Docker Hub

```yaml
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}    # Access token, không phải password

      - name: Build and push to Docker Hub
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: myorg/myapp:${{ github.sha }}
```

---

## 🚀 Docker Metadata Action — Tự Động Tạo Tags

`docker/metadata-action` tự động tạo tags và labels chuẩn từ git context:

```yaml
      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: |
            ghcr.io/${{ github.repository }}
            123456789.dkr.ecr.us-east-1.amazonaws.com/myapp
          tags: |
            # Tag với commit SHA (luôn có)
            type=sha,prefix=sha-,format=short

            # Tag với tên branch (main → :main)
            type=ref,event=branch

            # Tag với PR number (pr-123)
            type=ref,event=pr,prefix=pr-

            # Semantic versioning từ git tag (v1.2.3 → :1.2.3, :1.2, :1)
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{major}}

            # :latest chỉ khi push lên main
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

**Output example khi push tag `v1.2.3` lên main:**
```
ghcr.io/myorg/myapp:sha-abc1234
ghcr.io/myorg/myapp:main
ghcr.io/myorg/myapp:1.2.3
ghcr.io/myorg/myapp:1.2
ghcr.io/myorg/myapp:1
ghcr.io/myorg/myapp:latest
```

---

## ⚡ Build Cache (Bộ Đệm Build) — Tăng Tốc CI

### GitHub Actions Cache

```yaml
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build with cache
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha           # Lấy cache từ GitHub Actions cache
          cache-to: type=gha,mode=max    # Lưu cache — mode=max cache tất cả layers
```

### Registry Cache (Cache trên Registry)

```yaml
      - name: Build with registry cache
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/myorg/myapp:${{ github.sha }}
          cache-from: type=registry,ref=ghcr.io/myorg/myapp:buildcache
          cache-to: type=registry,ref=ghcr.io/myorg/myapp:buildcache,mode=max
```

### So Sánh Cache Strategies

| Strategy | Tốc Độ Đọc | Chi Phí Lưu Trữ | Phù Hợp |
|---|---|---|---|
| `type=gha` | Nhanh | Dùng Actions storage | Dự án nhỏ-vừa |
| `type=registry` | Trung bình | Registry storage | Nhiều workflows |
| `type=local` | Rất nhanh | Runner disk | Self-hosted runners |
| `type=inline` | Chậm | Trong image layer | Không khuyến nghị |

---

## 🛡️ Image Security Scanning (Quét Bảo Mật Image)

### Trivy — Quét Vulnerabilities

```yaml
      - name: Build image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false    # Chưa push, scan trước
          tags: myapp:${{ github.sha }}
          load: true     # Load vào Docker daemon để scan

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: table
          exit-code: 1               # Fail workflow nếu tìm thấy CRITICAL/HIGH
          ignore-unfixed: true       # Bỏ qua lỗi chưa có bản vá
          vuln-type: os,library
          severity: CRITICAL,HIGH

      - name: Push image (chỉ sau khi scan pass)
        if: success()
        run: |
          docker tag myapp:${{ github.sha }} ghcr.io/myorg/myapp:${{ github.sha }}
          docker push ghcr.io/myorg/myapp:${{ github.sha }}
```

### Trivy với SARIF Output (Tích Hợp GitHub Security Tab)

```yaml
      - name: Run Trivy và upload kết quả lên GitHub Security
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif
```

### Docker Scout (GitHub-native scanning)

```yaml
      - name: Docker Scout CVEs
        uses: docker/scout-action@v1
        with:
          command: cves
          image: ghcr.io/myorg/myapp:${{ github.sha }}
          only-severities: critical,high
          exit-code: true
```

---

## 🏗️ Multi-Architecture Build (Build Nhiều Kiến Trúc CPU)

Cần thiết khi deploy trên ARM (ví dụ: AWS Graviton, Apple Silicon):

```yaml
      - name: Set up QEMU (giả lập nhiều CPU architectures)
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build multi-arch image
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64    # Intel x86 + ARM
          push: true
          tags: ghcr.io/myorg/myapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## 🔄 Deploy Container sau khi Build

### Deploy lên Kubernetes

```yaml
  deploy:
    needs: build-push
    environment: production
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup kubectl
        uses: azure/setup-kubectl@v4

      - name: Configure kubeconfig
        run: |
          echo "${{ secrets.KUBECONFIG_BASE64 }}" | base64 -d > ~/.kube/config

      - name: Deploy to Kubernetes
        run: |
          # Cập nhật image trong deployment
          kubectl set image deployment/myapp \
            myapp=ghcr.io/${{ github.repository }}:${{ github.sha }} \
            --record

          # Đợi rollout hoàn thành
          kubectl rollout status deployment/myapp --timeout=5m

      - name: Verify pods are running
        run: |
          kubectl get pods -l app=myapp
          # Đảm bảo không có pod CrashLoopBackOff
          FAILED=$(kubectl get pods -l app=myapp \
            --field-selector=status.phase=Failed -o name | wc -l)
          [ "$FAILED" = "0" ] || exit 1
```

### Deploy lên AWS ECS

```yaml
      - name: Render ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: myapp
          image: ${{ needs.build-push.outputs.image-uri }}

      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: myapp-service
          cluster: production
          wait-for-service-stability: true
```

### Deploy lên Google Cloud Run

```yaml
      - name: Deploy to Cloud Run
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: myapp
          region: us-central1
          image: us-central1-docker.pkg.dev/my-project/myapp:${{ github.sha }}
          flags: --min-instances=1 --max-instances=10 --memory=512Mi
```

---

## 🔁 Docker Compose cho Integration Tests

```yaml
jobs:
  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start services với Docker Compose
        run: docker compose -f docker-compose.test.yml up -d

      - name: Wait for services to be healthy
        run: |
          timeout 60 bash -c 'until docker compose exec -T db pg_isready; do sleep 2; done'

      - name: Run integration tests
        run: docker compose exec -T app npm run test:integration

      - name: Collect logs on failure
        if: failure()
        run: docker compose logs

      - name: Tear down services
        if: always()
        run: docker compose -f docker-compose.test.yml down -v
```

```yaml
# docker-compose.test.yml
services:
  app:
    build: .
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/testdb
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: testdb
    healthcheck:
      test: ["CMD", "pg_isready"]
      interval: 5s
      timeout: 5s
      retries: 5
```

---

## 📋 Complete Docker CI/CD Pipeline

```yaml
name: Docker CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ── 1. Build & Test ────────────────────────────────────
  build-test:
    name: Build & Test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      security-events: write    # Cho SARIF upload

    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tags: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Log in to GHCR
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build image (và push nếu không phải PR)
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          load: ${{ github.event_name == 'pull_request' }}    # Load locally khi PR

      - name: Security scan với Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH

      - name: Upload security scan results
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif

  # ── 2. Deploy Staging ──────────────────────────────────
  deploy-staging:
    name: Deploy → Staging
    needs: build-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_STAGING_ROLE_ARN }}
          aws-region: us-east-1
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster staging \
            --service myapp \
            --force-new-deployment
          aws ecs wait services-stable \
            --cluster staging \
            --services myapp

  # ── 3. Deploy Production ───────────────────────────────
  deploy-production:
    name: Deploy → Production
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_PROD_ROLE_ARN }}
          aws-region: us-east-1
      - name: Deploy to ECS Production
        run: |
          aws ecs update-service \
            --cluster production \
            --service myapp \
            --force-new-deployment
          aws ecs wait services-stable \
            --cluster production \
            --services myapp
      - name: Health check
        run: curl -f https://myapp.com/health
```

---

## ❌ Lỗi Phổ Biến

### 1. Build context quá lớn

```
# .dockerignore — bắt buộc phải có
node_modules/
.git/
*.log
dist/
coverage/
.env
.env.*
**/*.test.ts
**/*.spec.ts
README.md
```

### 2. Chạy container với root user

```dockerfile
# ❌ SAI — chạy với root
FROM node:20-alpine
CMD ["node", "app.js"]

# ✅ ĐÚNG — tạo non-root user
FROM node:20-alpine
RUN addgroup -S app && adduser -S app -G app
USER app
CMD ["node", "app.js"]
```

### 3. Lưu secrets trong image layers

```dockerfile
# ❌ SAI — secret được lưu vào layer!
RUN curl -H "Authorization: Bearer $API_KEY" https://api.example.com/download

# ✅ ĐÚNG — dùng BuildKit secrets
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=api_key \
    curl -H "Authorization: Bearer $(cat /run/secrets/api_key)" https://api.example.com/download
```

---

## 🔗 Điều Hướng

| ← Trước | Hiện Tại | Tiếp Theo → |
|---|---|---|
| [2-deployment-strategies.md](2-deployment-strategies.md) | 3-docker-deployments.md | [4-kubernetes-deployments.md](4-kubernetes-deployments.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-11
