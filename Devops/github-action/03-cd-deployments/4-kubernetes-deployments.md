# 4 — Kubernetes Deployments (Triển Khai trên Kubernetes)

> Kubernetes — K8s — là nền tảng orchestration (điều phối) container phổ biến nhất. GitHub Actions có thể tương tác với K8s cluster thông qua `kubectl`, Helm, Kustomize và các GitOps tools như Argo CD.

## 🎯 Các Cách Deploy lên Kubernetes

```
GitHub Actions → Kubernetes

Option 1: kubectl apply     — đơn giản, trực tiếp
Option 2: Helm upgrade      — quản lý release, template linh hoạt
Option 3: Kustomize         — overlay-based, không cần template engine
Option 4: GitOps (Argo CD)  — declarative, pull-based, audit trail
```

---

## ⚙️ Kết Nối với Kubernetes Cluster

### Cách 1: kubeconfig từ Secret

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Setup kubectl
        uses: azure/setup-kubectl@v4
        with:
          version: 'v1.28.0'

      - name: Configure kubeconfig
        run: |
          mkdir -p ~/.kube
          # Lưu kubeconfig base64-encoded vào GitHub Secret
          echo "${{ secrets.KUBECONFIG_BASE64 }}" | base64 -d > ~/.kube/config
          chmod 600 ~/.kube/config

      - name: Verify connection
        run: kubectl cluster-info
```

### Cách 2: AWS EKS — aws eks update-kubeconfig

```yaml
      - name: Configure AWS credentials (OIDC — không cần long-lived keys)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_EKS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Update kubeconfig cho EKS cluster
        run: |
          aws eks update-kubeconfig \
            --name my-production-cluster \
            --region us-east-1
```

### Cách 3: GKE — gke-gcloud-auth-plugin

```yaml
      - name: Authenticate to Google Cloud (Workload Identity Federation)
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}

      - name: Get GKE credentials
        uses: google-github-actions/get-gke-credentials@v2
        with:
          cluster_name: my-cluster
          location: us-central1
```

### Cách 4: Azure AKS — azure/aks-set-context

```yaml
      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Set AKS context
        uses: azure/aks-set-context@v4
        with:
          resource-group: my-resource-group
          cluster-name: my-aks-cluster
```

---

## 1️⃣ kubectl — Triển Khai Trực Tiếp

### Cập Nhật Image (Rolling Update)

```yaml
      - name: Deploy với kubectl set image
        run: |
          kubectl set image deployment/myapp \
            myapp=ghcr.io/myorg/myapp:${{ github.sha }} \
            --namespace production

          kubectl rollout status deployment/myapp \
            --namespace production \
            --timeout=5m
```

### Apply Manifest Files

```yaml
      - name: Update image tag trong manifest
        run: |
          # Dùng sed để thay thế image tag trong YAML
          sed -i "s|IMAGE_TAG|${{ github.sha }}|g" k8s/deployment.yaml

      - name: Apply Kubernetes manifests
        run: |
          kubectl apply -f k8s/ --namespace production
          kubectl rollout status deployment/myapp --namespace production
```

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: ghcr.io/myorg/myapp:IMAGE_TAG    # Sẽ được sed thay thế
          ports:
            - containerPort: 3000
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
```

### Rollback với kubectl

```yaml
      - name: Rollback nếu health check thất bại
        if: failure()
        run: |
          kubectl rollout undo deployment/myapp --namespace production
          kubectl rollout status deployment/myapp \
            --namespace production \
            --timeout=3m
          echo "✅ Rollback hoàn thành"

      # Xem lịch sử deployment
      - name: Check rollout history
        run: |
          kubectl rollout history deployment/myapp --namespace production
```

---

## 2️⃣ Helm — Package Manager cho Kubernetes

Helm sử dụng "charts" (gói templates) để định nghĩa, cài đặt và nâng cấp Kubernetes applications.

```
Helm Concepts:
  Chart     — Gói template Kubernetes (giống apt package)
  Release   — Instance cụ thể của một chart đã install
  Values    — Cấu hình tùy chỉnh cho chart
  Repository — Nơi lưu trữ charts
```

### Basic Helm Deployment

```yaml
      - name: Setup Helm
        uses: azure/setup-helm@v4
        with:
          version: 'v3.13.0'

      - name: Add Helm repositories
        run: |
          helm repo add stable https://charts.helm.sh/stable
          helm repo update

      - name: Deploy / Upgrade với Helm
        run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace production \
            --create-namespace \
            --set image.tag=${{ github.sha }} \
            --set image.repository=ghcr.io/myorg/myapp \
            --set replicas=3 \
            --values ./charts/myapp/values-production.yaml \
            --wait \
            --timeout 5m \
            --atomic    # Rollback tự động nếu deploy thất bại
```

**`--atomic` flag:** Nếu deployment thất bại, Helm tự động rollback về release trước. Rất hữu ích cho production.

### Helm Values Files

```yaml
# charts/myapp/values-production.yaml
image:
  repository: ghcr.io/myorg/myapp
  tag: latest      # Override bằng --set image.tag=${{ github.sha }}
  pullPolicy: IfNotPresent

replicaCount: 3

resources:
  requests:
    memory: "256Mi"
    cpu: "200m"
  limits:
    memory: "512Mi"
    cpu: "1000m"

autoscaling:
  # HPA — Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

ingress:
  enabled: true
  hostname: myapp.com
  tls: true

env:
  LOG_LEVEL: "warn"
  APP_ENV: "production"
```

### Helm với OIDC và ECR

```yaml
      - name: Login to ECR for Helm charts
        run: |
          aws ecr get-login-password --region us-east-1 | \
            helm registry login \
              --username AWS \
              --password-stdin \
              123456789.dkr.ecr.us-east-1.amazonaws.com

      - name: Deploy chart từ ECR OCI registry
        run: |
          helm upgrade --install myapp \
            oci://123456789.dkr.ecr.us-east-1.amazonaws.com/charts/myapp \
            --version ${{ env.CHART_VERSION }} \
            --namespace production \
            --set image.tag=${{ github.sha }} \
            --wait --atomic
```

### Kiểm Tra Helm Release

```yaml
      - name: Verify Helm release
        run: |
          helm status myapp --namespace production
          helm history myapp --namespace production --max 5
          kubectl get pods -l "app.kubernetes.io/name=myapp" -n production
```

---

## 3️⃣ Kustomize — Overlay-Based Configuration

Kustomize cho phép tùy chỉnh manifests bằng overlays (lớp phủ) mà không cần templates. Tích hợp sẵn trong `kubectl apply -k`.

```
k8s/
├── base/                   ← Cấu hình chung cho tất cả envs
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/               ← Tùy chỉnh per-environment
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patch-replicas.yaml
    └── production/
        ├── kustomization.yaml
        └── patch-replicas.yaml
```

```yaml
# k8s/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

```yaml
# k8s/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
namespace: production
images:
  - name: ghcr.io/myorg/myapp
    newTag: REPLACE_IMAGE_TAG    # Sẽ được kustomize edit set thay thế
patches:
  - path: patch-replicas.yaml
```

```yaml
# k8s/overlays/production/patch-replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5    # Override replicas cho production
```

### GitHub Actions với Kustomize

```yaml
      - name: Update image tag với kustomize
        run: |
          cd k8s/overlays/production
          kustomize edit set image \
            ghcr.io/myorg/myapp=ghcr.io/myorg/myapp:${{ github.sha }}

      - name: Apply với kustomize
        run: |
          kustomize build k8s/overlays/production | kubectl apply -f -
          kubectl rollout status deployment/myapp --namespace production
```

---

## 4️⃣ GitOps với Argo CD

### GitOps — Nguyên Tắc Cốt Lõi

```
Traditional (Push-based):
  GitHub Actions → kubectl apply → Cluster
  (Actions trực tiếp push changes vào cluster)

GitOps (Pull-based):
  GitHub Actions → commit to Git → Argo CD watches Git → Argo CD applies to Cluster
  (Cluster tự kéo changes từ Git về)
```

**Lợi ích GitOps:**
- Cluster state luôn match với Git state (single source of truth)
- Mọi thay đổi đều có audit trail trong Git
- Dễ rollback (chỉ cần `git revert`)
- Không cần cluster credentials trong GitHub Actions

### Argo CD Image Updater

Thay vì GitHub Actions cập nhật K8s trực tiếp, Argo CD Image Updater tự phát hiện image mới và cập nhật:

```yaml
# Annotation trên Argo CD Application
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: |
      myapp=ghcr.io/myorg/myapp
    argocd-image-updater.argoproj.io/myapp.update-strategy: digest
    argocd-image-updater.argoproj.io/myapp.tag-match: ^sha-
```

### GitHub Actions + Argo CD Sync Trigger

```yaml
  deploy:
    needs: build-push
    runs-on: ubuntu-latest
    steps:
      - name: Update image tag trong GitOps repo
        uses: actions/checkout@v4
        with:
          repository: myorg/gitops-config    # Repo chứa Kubernetes manifests
          token: ${{ secrets.GITOPS_TOKEN }}

      - name: Update image tag và commit
        run: |
          cd apps/myapp/overlays/production
          kustomize edit set image \
            ghcr.io/myorg/myapp=ghcr.io/myorg/myapp:${{ github.sha }}

          git config user.email "ci@myorg.com"
          git config user.name "GitHub Actions"
          git add .
          git commit -m "chore: update myapp to ${{ github.sha }}"
          git push

      - name: Trigger Argo CD sync (không bắt buộc — Argo CD tự sync)
        run: |
          curl -s -X POST \
            -H "Authorization: Bearer ${{ secrets.ARGOCD_TOKEN }}" \
            https://argocd.myorg.com/api/v1/applications/myapp/sync

      - name: Wait for Argo CD sync to complete
        run: |
          for i in {1..30}; do
            STATUS=$(curl -s \
              -H "Authorization: Bearer ${{ secrets.ARGOCD_TOKEN }}" \
              https://argocd.myorg.com/api/v1/applications/myapp \
              | jq -r '.status.sync.status')

            if [ "$STATUS" = "Synced" ]; then
              echo "✅ Argo CD sync completed"
              break
            fi
            echo "Waiting for sync... ($i/30)"
            sleep 10
          done
```

---

## 🔍 Health Checks và Verification

### Kubernetes Health Check Patterns

```yaml
      - name: Comprehensive health check
        run: |
          NAMESPACE=production
          DEPLOYMENT=myapp

          # 1. Kiểm tra deployment rollout status
          kubectl rollout status deployment/$DEPLOYMENT \
            -n $NAMESPACE --timeout=5m

          # 2. Kiểm tra tất cả pods đang Running
          NOT_RUNNING=$(kubectl get pods -l app=$DEPLOYMENT \
            -n $NAMESPACE \
            --field-selector='status.phase!=Running' \
            -o name | wc -l)
          [ "$NOT_RUNNING" = "0" ] || {
            echo "❌ Một số pods không ở trạng thái Running"
            kubectl describe pods -l app=$DEPLOYMENT -n $NAMESPACE
            exit 1
          }

          # 3. Kiểm tra không có CrashLoopBackOff
          CRASHING=$(kubectl get pods -l app=$DEPLOYMENT \
            -n $NAMESPACE \
            -o jsonpath='{range .items[*]}{.status.containerStatuses[*].state.waiting.reason}{"\n"}{end}' \
            | grep -c "CrashLoopBackOff" || true)
          [ "$CRASHING" = "0" ] || {
            echo "❌ Phát hiện CrashLoopBackOff"
            exit 1
          }

          # 4. Kiểm tra số replicas mong muốn đang available
          AVAILABLE=$(kubectl get deployment/$DEPLOYMENT \
            -n $NAMESPACE \
            -o jsonpath='{.status.availableReplicas}')
          DESIRED=$(kubectl get deployment/$DEPLOYMENT \
            -n $NAMESPACE \
            -o jsonpath='{.spec.replicas}')
          [ "$AVAILABLE" = "$DESIRED" ] || {
            echo "❌ Chỉ có $AVAILABLE/$DESIRED replicas available"
            exit 1
          }

          echo "✅ Deployment healthy: $AVAILABLE/$DESIRED replicas running"
```

### Liveness vs Readiness Probes

```yaml
# deployment.yaml — Probes quan trọng cho zero-downtime
containers:
  - name: myapp
    # Readiness Probe — Pod chỉ nhận traffic khi probe pass
    readinessProbe:
      httpGet:
        path: /ready      # Endpoint riêng — kiểm tra app sẵn sàng nhận requests
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3

    # Liveness Probe — Kubernetes restart pod nếu probe fail liên tục
    livenessProbe:
      httpGet:
        path: /health     # Endpoint đơn giản — chỉ check app còn sống
        port: 3000
      initialDelaySeconds: 15
      periodSeconds: 10
      failureThreshold: 5

    # Startup Probe — Dành cho app khởi động chậm (không làm liveness kill sớm)
    startupProbe:
      httpGet:
        path: /health
        port: 3000
      failureThreshold: 30
      periodSeconds: 10
```

---

## 📋 Complete Kubernetes CD Pipeline

```yaml
name: Kubernetes CD

on:
  push:
    branches: [main]

jobs:
  build:
    name: Build & Push Image
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image: ghcr.io/${{ github.repository }}:${{ github.sha }}

    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

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
      - uses: azure/setup-kubectl@v4
      - uses: azure/setup-helm@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_STAGING_ROLE_ARN }}
          aws-region: us-east-1

      - name: Get EKS kubeconfig
        run: |
          aws eks update-kubeconfig \
            --name staging-cluster \
            --region us-east-1

      - name: Helm upgrade staging
        run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace staging \
            --create-namespace \
            --set image.tag=${{ github.sha }} \
            --values ./charts/myapp/values-staging.yaml \
            --wait --timeout 5m --atomic

      - name: Smoke tests
        run: |
          kubectl run smoke-test --image=curlimages/curl --restart=Never \
            --rm -it \
            --namespace staging \
            -- curl -f http://myapp.staging.svc.cluster.local/health

  deploy-production:
    name: Deploy → Production
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-kubectl@v4
      - uses: azure/setup-helm@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_PROD_ROLE_ARN }}
          aws-region: us-east-1

      - name: Get EKS kubeconfig
        run: |
          aws eks update-kubeconfig \
            --name production-cluster \
            --region us-east-1

      - name: Helm upgrade production
        run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace production \
            --set image.tag=${{ github.sha }} \
            --values ./charts/myapp/values-production.yaml \
            --wait --timeout 10m --atomic

      - name: Post-deploy health check
        run: |
          sleep 15
          curl -f https://myapp.com/health
          echo "✅ Production deployment successful"
```

---

## 🔗 Điều Hướng

| ← Trước | Hiện Tại | Tiếp Theo → |
|---|---|---|
| [3-docker-deployments.md](3-docker-deployments.md) | 4-kubernetes-deployments.md | [5-aws-deployments.md](5-aws-deployments.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-11
