# 7 — Azure Deployments (Triển Khai trên Microsoft Azure)

> Azure là cloud platform của Microsoft. GitHub Actions tích hợp sâu với Azure thông qua `azure/login` action và bộ actions chính thức cho AKS — Azure Kubernetes Service, Azure Container Apps, App Service, và nhiều dịch vụ khác.

## 🔐 Xác Thực với Azure

### Cách 1: OIDC Federated Credentials (Khuyến Nghị)

Microsoft Entra ID (trước đây là Azure Active Directory) hỗ trợ OIDC Federated Credentials, cho phép GitHub Actions xác thực không cần client secret.

```bash
# 1. Tạo App Registration trong Azure Portal hoặc Azure CLI
az ad app create --display-name "github-actions-myrepo"

# 2. Tạo Service Principal
az ad sp create --id <app-id>

# 3. Thêm Federated Credential
az ad app federated-credential create \
  --id <app-id> \
  --parameters '{
    "name": "github-actions",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:myorg/myrepo:environment:production",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# 4. Cấp quyền cho Service Principal
az role assignment create \
  --assignee <service-principal-id> \
  --role "Contributor" \
  --scope /subscriptions/<subscription-id>/resourceGroups/my-rg
```

```yaml
jobs:
  deploy:
    permissions:
      id-token: write    # Bắt buộc cho OIDC
      contents: read

    steps:
      - name: Azure login via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          # Không cần AZURE_CLIENT_SECRET!
```

### Cách 2: Service Principal với Client Secret

```yaml
      - name: Azure login với service principal
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
          # AZURE_CREDENTIALS là JSON chứa: clientId, clientSecret, subscriptionId, tenantId
```

```json
// Định dạng AZURE_CREDENTIALS secret:
{
  "clientId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "clientSecret": "your-client-secret",
  "subscriptionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

---

## 📦 ACR — Azure Container Registry (Kho Container Azure)

ACR — Azure Container Registry — Kho Lưu Trữ Container của Azure.

```yaml
jobs:
  build-push-acr:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    outputs:
      image-uri: ${{ secrets.ACR_LOGIN_SERVER }}/myapp:${{ github.sha }}

    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Login to ACR
        run: |
          az acr login --name ${{ secrets.ACR_NAME }}

      - name: Build and push to ACR
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.ACR_LOGIN_SERVER }}/myapp:${{ github.sha }}
            ${{ secrets.ACR_LOGIN_SERVER }}/myapp:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      # Hoặc dùng ACR Tasks để build trực tiếp trên Azure
      - name: Build với ACR Tasks (tùy chọn)
        run: |
          az acr build \
            --registry ${{ secrets.ACR_NAME }} \
            --image myapp:${{ github.sha }} \
            --file Dockerfile .
```

---

## ☸️ AKS — Azure Kubernetes Service

AKS — Azure Kubernetes Service — Dịch Vụ Kubernetes Được Quản Lý của Azure.

### Deploy lên AKS

```yaml
jobs:
  deploy-aks:
    needs: build-push-acr
    environment: production
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Get AKS kubeconfig
        uses: azure/aks-set-context@v4
        with:
          resource-group: my-resource-group
          cluster-name: production-aks

      - name: Setup Helm
        uses: azure/setup-helm@v4

      - name: Deploy với Helm
        run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace production \
            --create-namespace \
            --set image.repository=${{ secrets.ACR_LOGIN_SERVER }}/myapp \
            --set image.tag=${{ github.sha }} \
            --values ./charts/myapp/values-production.yaml \
            --wait --timeout 10m --atomic

      - name: Verify AKS deployment
        run: |
          kubectl get pods -n production -l app=myapp
          kubectl rollout status deployment/myapp -n production
```

### AKS với Workload Identity (Thay Thế Pod Identity)

```yaml
      - name: Deploy với Workload Identity annotation
        run: |
          # Kubernetes Service Account cần annotation:
          # azure.workload.identity/client-id: <client-id>
          # Pods tự động lấy Azure token không cần secrets

          kubectl apply -f - <<EOF
          apiVersion: v1
          kind: ServiceAccount
          metadata:
            name: myapp-sa
            namespace: production
            annotations:
              azure.workload.identity/client-id: ${{ secrets.AZURE_CLIENT_ID }}
          EOF
```

---

## 🌐 Azure App Service — PaaS Deployment

Azure App Service — Dịch Vụ Ứng Dụng Azure — là PaaS (Platform as a Service — Nền Tảng Dịch Vụ) cho web apps, APIs, mobile backends.

### Deploy Web App

```yaml
jobs:
  deploy-appservice:
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

      - name: Build application
        run: |
          npm ci
          npm run build

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Azure App Service
        uses: azure/webapps-deploy@v3
        with:
          app-name: myapp-production
          slot-name: production     # Slot name, hoặc dùng staging slot
          package: ./dist
```

### Deployment Slots (Khe Triển Khai) — Blue/Green cho App Service

```yaml
      - name: Deploy lên staging slot trước
        uses: azure/webapps-deploy@v3
        with:
          app-name: myapp-production
          slot-name: staging        # Deploy vào slot staging, không phải production
          package: ./dist

      - name: Test staging slot
        run: |
          curl -f https://myapp-production-staging.azurewebsites.net/health

      - name: Swap slots (đổi staging sang production)
        run: |
          az webapp deployment slot swap \
            --name myapp-production \
            --resource-group my-rg \
            --slot staging \
            --target-slot production
          # Production bây giờ chạy code mới
          # Staging bây giờ chạy code cũ — rollback bằng cách swap lại

      - name: Verify production sau swap
        run: |
          curl -f https://myapp-production.azurewebsites.net/health

      - name: Rollback — swap lại nếu cần
        if: failure()
        run: |
          az webapp deployment slot swap \
            --name myapp-production \
            --resource-group my-rg \
            --slot staging \
            --target-slot production
```

### Deploy Container lên App Service

```yaml
      - name: Deploy container image lên App Service
        uses: azure/webapps-deploy@v3
        with:
          app-name: myapp-production
          images: ${{ secrets.ACR_LOGIN_SERVER }}/myapp:${{ github.sha }}
```

---

## 📱 Azure Container Apps — Serverless Container

Azure Container Apps — Ứng Dụng Container Không Máy Chủ — là managed service cho microservices và serverless containers. Scale to zero như Cloud Run.

```yaml
jobs:
  deploy-container-apps:
    needs: build-push-acr
    environment: production
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Azure Container Apps
        uses: azure/container-apps-deploy-action@v2
        with:
          resourceGroup: my-resource-group
          containerAppName: myapp
          imageToDeploy: ${{ secrets.ACR_LOGIN_SERVER }}/myapp:${{ github.sha }}
          containerAppEnvironment: production-env
          targetPort: 3000
          ingress: external

      - name: Get Container App URL
        run: |
          URL=$(az containerapp show \
            --name myapp \
            --resource-group my-resource-group \
            --query properties.configuration.ingress.fqdn \
            --output tsv)
          echo "Container App URL: https://$URL"
          curl -f "https://$URL/health"
```

### Traffic Splitting cho Container Apps

```yaml
      - name: Deploy new revision
        run: |
          az containerapp update \
            --name myapp \
            --resource-group my-resource-group \
            --image ${{ secrets.ACR_LOGIN_SERVER }}/myapp:${{ github.sha }} \
            --revision-suffix ${{ github.run_number }}

      - name: Route 20% traffic sang revision mới (Canary)
        run: |
          az containerapp ingress traffic set \
            --name myapp \
            --resource-group my-resource-group \
            --revision-weight \
              latest=20 \
              stable=80

      - name: Promote to 100% sau khi verify
        if: success()
        run: |
          az containerapp ingress traffic set \
            --name myapp \
            --resource-group my-resource-group \
            --revision-weight latest=100
```

---

## 🔒 Azure Key Vault Integration

Azure Key Vault — Kho Bí Mật Azure — lưu trữ và quản lý secrets, keys, certificates.

```yaml
      - name: Get secrets từ Azure Key Vault
        uses: azure/get-keyvault-secrets@v1
        with:
          keyvault: my-keyvault
          secrets: 'database-url, api-key, jwt-secret'
        id: keyvault-secrets

      - name: Deploy với Key Vault secrets
        uses: azure/webapps-deploy@v3
        with:
          app-name: myapp-production
          package: ./dist
        env:
          DATABASE_URL: ${{ steps.keyvault-secrets.outputs.database-url }}
          API_KEY: ${{ steps.keyvault-secrets.outputs.api-key }}
```

---

## 🚀 Azure Static Web Apps (Frontend)

```yaml
jobs:
  deploy-static-web-app:
    environment: production
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Build frontend
        run: |
          npm ci
          npm run build
        env:
          VITE_API_URL: ${{ vars.API_URL }}

      - name: Deploy to Azure Static Web Apps
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          action: upload
          app_location: /
          output_location: dist
```

---

## 📊 Complete Azure Deployment Pipeline

```yaml
name: Azure CD Pipeline

on:
  push:
    branches: [main]

jobs:
  build:
    name: Build & Push to ACR
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    outputs:
      image-uri: ${{ secrets.ACR_LOGIN_SERVER }}/myapp:${{ github.sha }}

    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - run: az acr login --name ${{ secrets.ACR_NAME }}
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.ACR_LOGIN_SERVER }}/myapp:${{ github.sha }}
            ${{ secrets.ACR_LOGIN_SERVER }}/myapp:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build
    environment: staging
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4
      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - uses: azure/aks-set-context@v4
        with:
          resource-group: staging-rg
          cluster-name: staging-aks
      - uses: azure/setup-helm@v4
      - run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace staging \
            --create-namespace \
            --set image.repository=${{ secrets.ACR_LOGIN_SERVER }}/myapp \
            --set image.tag=${{ github.sha }} \
            --values ./charts/myapp/values-staging.yaml \
            --wait --timeout 5m --atomic

  deploy-production:
    needs: deploy-staging
    environment: production
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4
      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - uses: azure/aks-set-context@v4
        with:
          resource-group: production-rg
          cluster-name: production-aks
      - uses: azure/setup-helm@v4
      - run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace production \
            --set image.repository=${{ secrets.ACR_LOGIN_SERVER }}/myapp \
            --set image.tag=${{ github.sha }} \
            --values ./charts/myapp/values-production.yaml \
            --wait --timeout 10m --atomic
      - run: curl -f https://myapp.com/health
```

---

## 📋 Azure Actions Reference

| Action | Mục Đích |
|---|---|
| `azure/login@v2` | Đăng nhập Azure (OIDC hoặc service principal) |
| `azure/aks-set-context@v4` | Lấy kubeconfig cho AKS cluster |
| `azure/setup-helm@v4` | Cài đặt Helm |
| `azure/setup-kubectl@v4` | Cài đặt kubectl |
| `azure/webapps-deploy@v3` | Deploy lên App Service |
| `azure/container-apps-deploy-action@v2` | Deploy lên Container Apps |
| `azure/get-keyvault-secrets@v1` | Đọc secrets từ Key Vault |
| `Azure/static-web-apps-deploy@v1` | Deploy Static Web Apps |
| `azure/arm-deploy@v1` | Deploy Azure Resource Manager template |

---

## 🔗 Điều Hướng

| ← Trước | Hiện Tại | Tiếp Theo → |
|---|---|---|
| [6-gcp-deployments.md](6-gcp-deployments.md) | 7-azure-deployments.md | [../04-secrets-variables/](../04-secrets-variables/) |

---

**Cập Nhật Lần Cuối:** 2026-05-11
