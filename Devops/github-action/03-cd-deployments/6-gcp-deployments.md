# 6 — GCP Deployments (Triển Khai trên Google Cloud Platform)

> GCP — Google Cloud Platform — là cloud provider của Google. GitHub Actions tích hợp chặt với GCP qua Workload Identity Federation (liên kết danh tính workload), GKE — Google Kubernetes Engine — và Cloud Run.

## 🔐 Xác Thực với GCP — Workload Identity Federation

Workload Identity Federation — Liên Kết Danh Tính Workload — là cách GCP cho phép GitHub Actions xác thực mà không cần service account key (khóa tài khoản dịch vụ) tĩnh.

### Cách hoạt động

```
GitHub Actions                    GCP
     │                             │
     │  1. Request OIDC token      │
     │────────────────────────────>│
     │                             │
     │  2. Present token to STS    │
     │  (Security Token Service)   │
     │────────────────────────────>│
     │                             │
     │  3. STS validates token     │
     │     với GitHub OIDC         │
     │                             │
     │  4. Issue short-lived       │
     │     access token            │
     │<────────────────────────────│
     │                             │
     │  5. Dùng token để gọi       │
     │     GCP APIs                │
     │────────────────────────────>│
```

### Thiết Lập Workload Identity Federation (Một Lần)

```bash
# 1. Tạo Workload Identity Pool
gcloud iam workload-identity-pools create github-actions-pool \
  --project=my-project \
  --location=global \
  --display-name="GitHub Actions Pool"

# 2. Tạo Workload Identity Provider
gcloud iam workload-identity-pools providers create-oidc github-provider \
  --project=my-project \
  --location=global \
  --workload-identity-pool=github-actions-pool \
  --display-name="GitHub Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com"

# 3. Tạo Service Account
gcloud iam service-accounts create github-actions-sa \
  --project=my-project \
  --display-name="GitHub Actions Service Account"

# 4. Cấp quyền cho Service Account
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:github-actions-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/container.developer"    # Cho GKE

# 5. Bind Workload Identity
gcloud iam service-accounts add-iam-policy-binding \
  github-actions-sa@my-project.iam.gserviceaccount.com \
  --project=my-project \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/123456/locations/global/workloadIdentityPools/github-actions-pool/attribute.repository/myorg/myrepo"
```

### Sử Dụng trong GitHub Actions

```yaml
jobs:
  deploy:
    permissions:
      id-token: write    # Bắt buộc để lấy OIDC token
      contents: read

    steps:
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/123456/locations/global/workloadIdentityPools/github-actions-pool/providers/github-provider
          service_account: github-actions-sa@my-project.iam.gserviceaccount.com
          # Không cần GCP_SA_KEY hay credentials JSON!
```

---

## 📦 Artifact Registry (Kho Lưu Trữ Artifact)

Artifact Registry — Kho Lưu Trữ Artifact — thay thế cho Container Registry (GCR) của Google, hỗ trợ Docker images, npm, Maven, Python packages.

### Push Image lên Artifact Registry

```yaml
jobs:
  build-push:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    outputs:
      image-uri: us-central1-docker.pkg.dev/my-project/my-repo/myapp:${{ github.sha }}

    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}

      - name: Configure Docker để push lên Artifact Registry
        run: |
          gcloud auth configure-docker us-central1-docker.pkg.dev --quiet

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            us-central1-docker.pkg.dev/my-project/my-repo/myapp:${{ github.sha }}
            us-central1-docker.pkg.dev/my-project/my-repo/myapp:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## ☁️ Cloud Run — Serverless Container Deployment

Cloud Run — Dịch Vụ Container Không Máy Chủ — chạy containers mà không cần quản lý servers. Tự động scale lên 0 khi không có traffic (tiết kiệm chi phí).

### Deploy đơn giản lên Cloud Run

```yaml
jobs:
  deploy-cloudrun:
    needs: build-push
    environment: production
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}

      - name: Deploy to Cloud Run
        id: deploy
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: myapp
          region: us-central1
          image: ${{ needs.build-push.outputs.image-uri }}
          flags: |
            --min-instances=1
            --max-instances=100
            --memory=512Mi
            --cpu=1
            --concurrency=80
            --timeout=300s
            --port=8080
          env_vars: |
            APP_ENV=production
            LOG_LEVEL=warn
          secrets: |
            DATABASE_URL=projects/my-project/secrets/prod-database-url/versions/latest
            API_KEY=projects/my-project/secrets/prod-api-key/versions/latest

      - name: Get Cloud Run URL
        run: |
          echo "Service URL: ${{ steps.deploy.outputs.url }}"

      - name: Verify deployment
        run: |
          curl -f ${{ steps.deploy.outputs.url }}/health
```

### Cloud Run Traffic Splitting (Canary cho Cloud Run)

```yaml
      - name: Deploy mới nhưng chưa gửi traffic
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: myapp
          region: us-central1
          image: ${{ needs.build-push.outputs.image-uri }}
          flags: --no-traffic    # Deploy nhưng không nhận traffic

      - name: Test revision mới (không có traffic thực)
        run: |
          REVISION_URL=$(gcloud run revisions describe \
            --region us-central1 \
            --service myapp \
            --format 'value(status.url)' \
            $(gcloud run revisions list \
              --region us-central1 \
              --service myapp \
              --format 'value(metadata.name)' \
              --limit 1))
          curl -f "$REVISION_URL/health"

      - name: Route 10% traffic sang revision mới
        run: |
          NEW_REVISION=$(gcloud run revisions list \
            --region us-central1 \
            --service myapp \
            --format 'value(metadata.name)' \
            --limit 1)

          gcloud run services update-traffic myapp \
            --region us-central1 \
            --to-revisions $NEW_REVISION=10

      - name: Monitor và promote to 100%
        run: |
          echo "Monitoring for 5 minutes..."
          sleep 300

          # Kiểm tra error rate từ Cloud Monitoring
          # (bỏ qua query phức tạp, minh họa concept)
          echo "Promoting to 100%..."

          gcloud run services update-traffic myapp \
            --region us-central1 \
            --to-latest

      - name: Rollback nếu cần
        if: failure()
        run: |
          PREVIOUS_REVISION=$(gcloud run revisions list \
            --region us-central1 \
            --service myapp \
            --format 'value(metadata.name)' \
            --limit 2 | tail -1)

          gcloud run services update-traffic myapp \
            --region us-central1 \
            --to-revisions $PREVIOUS_REVISION=100
```

---

## ☸️ GKE — Google Kubernetes Engine

GKE — Google Kubernetes Engine — Dịch Vụ Kubernetes Được Quản Lý của Google.

### Deploy lên GKE

```yaml
jobs:
  deploy-gke:
    needs: build-push
    environment: production
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    env:
      GKE_CLUSTER: production-cluster
      GKE_ZONE: us-central1-a
      PROJECT_ID: my-project

    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}

      - name: Get GKE credentials
        uses: google-github-actions/get-gke-credentials@v2
        with:
          cluster_name: ${{ env.GKE_CLUSTER }}
          location: ${{ env.GKE_ZONE }}
          project_id: ${{ env.PROJECT_ID }}

      - name: Setup Helm
        uses: azure/setup-helm@v4

      - name: Deploy với Helm
        run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace production \
            --create-namespace \
            --set image.repository=us-central1-docker.pkg.dev/${{ env.PROJECT_ID }}/my-repo/myapp \
            --set image.tag=${{ github.sha }} \
            --values ./charts/myapp/values-production.yaml \
            --wait --timeout 10m --atomic

      - name: Verify deployment
        run: |
          kubectl get pods -n production -l app=myapp
          kubectl rollout status deployment/myapp -n production
          curl -f https://myapp.com/health
```

### GKE Autopilot (Serverless Kubernetes)

GKE Autopilot — Kubernetes Tự Động — tự động quản lý nodes, không cần cấu hình node pools:

```yaml
      - name: Get GKE Autopilot credentials
        uses: google-github-actions/get-gke-credentials@v2
        with:
          cluster_name: autopilot-cluster
          location: us-central1    # Region thay vì zone
          project_id: my-project

      - name: Deploy (Autopilot tự điều chỉnh resources)
        run: |
          kubectl apply -f k8s/deployment.yaml -n production
          kubectl rollout status deployment/myapp -n production --timeout=10m
          # Autopilot có thể cần thêm thời gian để provision nodes
```

---

## 🔒 Secret Manager Integration

Thay vì lưu secrets vào GitHub Secrets, dùng Google Secret Manager và truy cập trong runtime:

```yaml
      - name: Access Secret Manager
        id: secrets
        uses: google-github-actions/get-secretmanager-secrets@v2
        with:
          secrets: |
            database_url:my-project/prod-database-url
            api_key:my-project/prod-api-key/2          # Version cụ thể

      - name: Deploy với secrets từ Secret Manager
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: myapp
          region: us-central1
          image: ${{ needs.build-push.outputs.image-uri }}
          env_vars: |
            DATABASE_URL=${{ steps.secrets.outputs.database_url }}
            API_KEY=${{ steps.secrets.outputs.api_key }}
```

---

## 🌐 Firebase Hosting (Static Sites)

```yaml
jobs:
  deploy-firebase:
    environment: production
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Build
        run: |
          npm ci
          npm run build
        env:
          VITE_API_URL: ${{ vars.API_URL }}

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}

      - name: Deploy to Firebase Hosting
        uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: ${{ secrets.GITHUB_TOKEN }}
          firebaseServiceAccount: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
          channelId: live
          projectId: my-firebase-project
```

---

## 📋 GCP IAM Permissions Cần Thiết

```
Service Account cần các roles sau:

Cloud Run Deploy:
  roles/run.admin                      — manage Cloud Run services
  roles/iam.serviceAccountUser         — act as service accounts

GKE Deploy:
  roles/container.developer            — deploy to GKE
  roles/artifactregistry.reader        — pull images

Artifact Registry Push:
  roles/artifactregistry.writer        — push images

Secret Manager:
  roles/secretmanager.secretAccessor   — read secrets
```

---

## 🔗 Điều Hướng

| ← Trước | Hiện Tại | Tiếp Theo → |
|---|---|---|
| [5-aws-deployments.md](5-aws-deployments.md) | 6-gcp-deployments.md | [7-azure-deployments.md](7-azure-deployments.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-11
