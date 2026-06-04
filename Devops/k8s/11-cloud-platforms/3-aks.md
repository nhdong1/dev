# Azure AKS — Azure Kubernetes Service

> Hướng dẫn toàn diện về Azure AKS (Azure Kubernetes Service — Dịch Vụ Kubernetes Azure): kiến trúc, Azure AD integration, node pool, Windows container, Workload Identity và vận hành production trên Azure.

## Mục Lục

1. [Tổng Quan AKS](#tổng-quan-aks)
2. [Kiến Trúc AKS](#kiến-trúc-aks)
3. [Node Pool — Nhóm Node](#node-pool--nhóm-node)
4. [Azure AD Integration — Tích Hợp Azure Active Directory](#azure-ad-integration--tích-hợp-azure-active-directory)
5. [Workload Identity — Định Danh Workload](#workload-identity--định-danh-workload)
6. [Windows Container — Container Windows](#windows-container--container-windows)
7. [Networking trên AKS](#networking-trên-aks)
8. [Load Balancer và Ingress trên AKS](#load-balancer-và-ingress-trên-aks)
9. [Storage trên AKS](#storage-trên-aks)
10. [Bảo Mật AKS](#bảo-mật-aks)
11. [Tích Hợp Azure Services](#tích-hợp-azure-services)
12. [CI/CD với Azure DevOps](#cicd-với-azure-devops)
13. [Tối Ưu Chi Phí AKS](#tối-ưu-chi-phí-aks)
14. [Vận Hành và Upgrade](#vận-hành-và-upgrade)
15. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan AKS

**AKS (Azure Kubernetes Service — Dịch Vụ Kubernetes Azure)** là managed Kubernetes service trên Microsoft Azure — ra mắt 2018. Azure đặc biệt mạnh về tích hợp với hệ sinh thái Microsoft: Active Directory, Azure DevOps, .NET, Windows Server.

### Chi Phí Cơ Bản

```
Control Plane:  Miễn phí (không mất phí như EKS $0.10/giờ)
Worker Node:    Chi phí Azure VM thông thường
Uptime SLA:     99.95% với Availability Zones
                99.9%  không có Availability Zones

Lưu ý: Chi phí VM trên Azure thường cao hơn GCP/AWS một chút
        nhưng bù lại Control Plane miễn phí giống GKE
```

### Điểm Mạnh của AKS

```
✅ Control Plane miễn phí
✅ Tích hợp Azure Active Directory sâu nhất trong 3 managed provider
✅ Hỗ trợ Windows Container tốt nhất
✅ Azure DevOps tích hợp native
✅ Azure Monitor + Container Insights mạnh
✅ Virtual Node (Serverless bằng Azure Container Instances)
✅ KEDA (Kubernetes Event-Driven Autoscaling) được sáng tạo bởi Microsoft
```

---

## Kiến Trúc AKS

### Control Plane (Mặt Điều Khiển)

AKS Control Plane chạy trong subscription ẩn của Azure — miễn phí và được Azure quản lý hoàn toàn.

```
Azure Subscription của bạn           Azure Managed Subscription
┌─────────────────────────┐          ┌──────────────────────────┐
│  Resource Group: my-rg  │          │  AKS Control Plane       │
│                         │ ←Private→│  (Azure quản lý)         │
│  Node VM 1              │  Link    │  API Server              │
│  Node VM 2              │          │  etcd                    │
│  Node VM 3              │          │  Scheduler               │
│  Load Balancer          │          │  Controller Manager      │
│  VNet / Subnet          │          │                          │
└─────────────────────────┘          └──────────────────────────┘
```

### Private Cluster (Cluster Riêng Tư)

```bash
# Tạo AKS private cluster — API Server chỉ truy cập trong VNet
az aks create \
  --resource-group my-rg \
  --name my-cluster \
  --enable-private-cluster \
  --private-dns-zone system \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --network-plugin azure \
  --generate-ssh-keys
```

### Availability Zones (Vùng Khả Dụng)

```bash
# Tạo cluster với node phân bổ qua 3 AZ — HA cao nhất
az aks create \
  --resource-group my-rg \
  --name my-cluster \
  --zones 1 2 3 \                  # Zone 1, 2, 3 trong region
  --node-count 3 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 10
```

---

## Node Pool — Nhóm Node

AKS cho phép nhiều node pool trong cùng cluster — mỗi pool có thể có OS, VM size, taint/label riêng.

### System Node Pool vs User Node Pool

```
System Node Pool:
  - Chạy các component cốt lõi của K8s: CoreDNS, metrics-server
  - Bắt buộc phải có ít nhất 1 system node pool
  - Taint: CriticalAddonsOnly=true:NoSchedule (workload user không lên đây)
  - Khuyến nghị: không chạy workload ứng dụng trên system pool

User Node Pool:
  - Chạy workload ứng dụng của bạn
  - Có thể scale to zero
  - Có thể dùng Spot VM
  - Có thể chạy Windows OS
```

### Quản Lý Node Pool

```bash
# Thêm node pool Linux cho workload thông thường
az aks nodepool add \
  --resource-group my-rg \
  --cluster-name my-cluster \
  --name linuxpool \
  --os-type Linux \
  --node-vm-size Standard_D4s_v3 \
  --node-count 3 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 10 \
  --node-taints workload=app:NoSchedule \
  --labels env=production

# Thêm node pool Windows cho .NET workload
az aks nodepool add \
  --resource-group my-rg \
  --cluster-name my-cluster \
  --name winpool \
  --os-type Windows \
  --node-vm-size Standard_D4s_v3 \
  --node-count 2 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 5

# Thêm node pool Spot cho batch job
az aks nodepool add \
  --resource-group my-rg \
  --cluster-name my-cluster \
  --name spotpool \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \              # -1 = trả theo giá spot thị trường
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 20 \
  --node-taints kubernetes.azure.com/scalesetpriority=spot:NoSchedule
```

### Node Pool Upgrade Mode

```bash
# Upgrade node pool với surge (tạo node mới trước)
az aks nodepool update \
  --resource-group my-rg \
  --cluster-name my-cluster \
  --name nodepool1 \
  --max-surge 1 \                    # Tạo thêm 1 node khi upgrade
  --node-soak-duration 0
```

---

## Azure AD Integration — Tích Hợp Azure Active Directory

### AKS-managed Azure AD (Phiên Bản Mới)

Cho phép dùng Azure AD user và group trực tiếp trong K8s RBAC — không cần cài thêm gì.

```bash
# Tạo cluster với AKS-managed Azure AD
az aks create \
  --resource-group my-rg \
  --name my-cluster \
  --enable-aad \
  --enable-azure-rbac \              # Dùng Azure RBAC thay vì K8s RBAC (hoặc kết hợp)
  --aad-admin-group-object-ids <GROUP_OBJECT_ID>

# Lấy credentials — Azure AD token sẽ được dùng để xác thực
az aks get-credentials \
  --resource-group my-rg \
  --name my-cluster
```

### Kết Hợp Azure RBAC với K8s RBAC

**Azure RBAC Role Assignment (Gán Role Azure RBAC):**

```bash
# Gán quyền Azure Kubernetes Service Cluster User Role
az role assignment create \
  --role "Azure Kubernetes Service Cluster User Role" \
  --assignee <USER_OR_GROUP_ID> \
  --scope /subscriptions/<SUB_ID>/resourceGroups/my-rg/providers/Microsoft.ContainerService/managedClusters/my-cluster

# Gán quyền cụ thể hơn theo namespace
az role assignment create \
  --role "Azure Kubernetes Service RBAC Writer" \
  --assignee <USER_ID> \
  --scope /subscriptions/<SUB_ID>/resourceGroups/my-rg/providers/Microsoft.ContainerService/managedClusters/my-cluster/namespaces/development
```

**K8s RBAC kết hợp Azure AD Group:**

```yaml
# ClusterRoleBinding với Azure AD Group
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: devops-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: Group
    name: <AZURE_AD_GROUP_OBJECT_ID>   # Object ID của Azure AD Group
```

```yaml
# RoleBinding giới hạn namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-edit
  namespace: development
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: Group
    name: <DEV_TEAM_GROUP_OBJECT_ID>
```

---

## Workload Identity — Định Danh Workload

**AKS Workload Identity** (thay thế AAD Pod Identity — Pod Identity Azure AD) cho phép Pod truy cập Azure services bằng Managed Identity — không cần secret hay connection string.

### Cơ Chế Hoạt Động

```
Pod
 │ (1) Gọi Azure API (Key Vault, Storage, SQL Database...)
 ↓
Azure Identity SDK trong Pod
 │ (2) Đọc OIDC token từ projected volume
 ↓
Azure STS (Security Token Service)
 │ (3) Xác thực token qua OIDC endpoint của AKS cluster
 │ (4) Kiểm tra federated credential linking giữa KSA và Managed Identity
 ↓
Azure STS trả về Access Token của Managed Identity
 │
 ↓
Pod gọi Azure API — không cần connection string
```

### Cấu Hình Workload Identity

**Bước 1: Bật OIDC Issuer và Workload Identity**

```bash
# Bật OIDC issuer
az aks update \
  --resource-group my-rg \
  --name my-cluster \
  --enable-oidc-issuer \
  --enable-workload-identity

# Lấy OIDC issuer URL
OIDC_ISSUER=$(az aks show \
  --resource-group my-rg \
  --name my-cluster \
  --query "oidcIssuerProfile.issuerUrl" \
  --output tsv)
```

**Bước 2: Tạo Azure Managed Identity**

```bash
# Tạo User-Assigned Managed Identity (Định Danh Được Quản Lý)
az identity create \
  --resource-group my-rg \
  --name my-app-identity

# Lấy client ID và principal ID
CLIENT_ID=$(az identity show \
  --resource-group my-rg \
  --name my-app-identity \
  --query clientId --output tsv)

PRINCIPAL_ID=$(az identity show \
  --resource-group my-rg \
  --name my-app-identity \
  --query principalId --output tsv)

# Gán quyền cho Managed Identity (ví dụ: đọc Key Vault secrets)
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $PRINCIPAL_ID \
  --scope /subscriptions/<SUB_ID>/resourceGroups/my-rg/providers/Microsoft.KeyVault/vaults/my-keyvault
```

**Bước 3: Tạo Federated Credential**

```bash
# Liên kết Managed Identity với K8s ServiceAccount
az identity federated-credential create \
  --name my-federated-credential \
  --identity-name my-app-identity \
  --resource-group my-rg \
  --issuer $OIDC_ISSUER \
  --subject system:serviceaccount:my-namespace:my-ksa \
  --audiences api://AzureADTokenExchange
```

**Bước 4: Tạo và Annotate ServiceAccount**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-ksa
  namespace: my-namespace
  annotations:
    azure.workload.identity/client-id: "<CLIENT_ID_CỦA_MANAGED_IDENTITY>"
  labels:
    azure.workload.identity/use: "true"    # Label bắt buộc
```

**Bước 5: Pod tự nhận Token**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: my-namespace
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: my-ksa
  containers:
    - name: app
      image: my-app:latest
      # Azure SDK tự detect AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_FEDERATED_TOKEN_FILE
      # được inject bởi mutating webhook của Workload Identity
```

---

## Windows Container — Container Windows

AKS hỗ trợ Windows container tốt nhất trong ba managed provider — phù hợp workload .NET Framework, SQL Server Agent, legacy Windows application.

### Yêu Cầu và Giới Hạn

```
Yêu cầu:
- Cluster phải dùng Azure CNI (không dùng kubenet)
- Windows node pool là User Node Pool (không phải System)
- Base image: Windows Server 2019 hoặc 2022

Giới hạn:
- Không hỗ trợ Linux-only feature: hostNetwork, hostPID
- Windows container không hỗ trợ một số security context Linux
- Ít công cụ debug hơn Linux container
```

### Tạo Windows Node Pool

```bash
# Cluster phải tạo với Windows username/password khi muốn hỗ trợ Windows
az aks create \
  --resource-group my-rg \
  --name my-cluster \
  --network-plugin azure \
  --windows-admin-username azureuser \
  --windows-admin-password MyPass123! \
  --generate-ssh-keys

# Thêm Windows node pool
az aks nodepool add \
  --resource-group my-rg \
  --cluster-name my-cluster \
  --os-type Windows \
  --name winpool \
  --node-count 2 \
  --node-vm-size Standard_D4s_v3
```

### Deploy Windows Container

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dotnet-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: dotnet-app
  template:
    metadata:
      labels:
        app: dotnet-app
    spec:
      nodeSelector:
        "kubernetes.io/os": windows          # Schedule lên Windows node
      tolerations:
        - key: "os"
          operator: Equal
          value: "windows"
          effect: NoSchedule
      containers:
        - name: dotnet-app
          image: mcr.microsoft.com/dotnet/aspnet:8.0-windowsservercore-ltsc2022
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 2
              memory: 2Gi
```

### Mixed Cluster (Cluster Kết Hợp Linux và Windows)

```yaml
# Linux Pod
spec:
  nodeSelector:
    "kubernetes.io/os": linux

# Windows Pod
spec:
  nodeSelector:
    "kubernetes.io/os": windows
```

---

## Networking trên AKS

### Azure CNI (Container Network Interface Gốc Azure) vs Kubenet

**Kubenet:**

```
- Pod nhận IP từ pod CIDR riêng (khác với VNet CIDR)
- Azure tạo route table để route traffic giữa node và Pod
- Giới hạn: 400 node tối đa
- Dùng khi: tiết kiệm IP VNet, cluster nhỏ, không cần tích hợp VNet sâu
```

**Azure CNI (Khuyến Nghị):**

```
- Mỗi Pod nhận IP trực tiếp từ VNet subnet
- Không cần route table — tích hợp native với VNet
- Yêu cầu nhiều IP hơn (cần plan subnet lớn)
- Hỗ trợ Windows container (Kubenet không hỗ trợ)
- Dùng khi: cần Windows container, VNet peering, kết nối on-premise
```

**Azure CNI Overlay (Cân Bằng Giữa Hai Loại):**

```bash
# Azure CNI Overlay: Pod dùng IP từ private range riêng
# nhưng vẫn tích hợp tốt hơn kubenet
az aks create \
  --resource-group my-rg \
  --name my-cluster \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --pod-cidr 192.168.0.0/16
```

### Tạo Cluster với Azure CNI

```bash
az aks create \
  --resource-group my-rg \
  --name my-cluster \
  --network-plugin azure \
  --vnet-subnet-id /subscriptions/<SUB>/resourceGroups/my-rg/providers/Microsoft.Network/virtualNetworks/my-vnet/subnets/aks-subnet \
  --docker-bridge-address 172.17.0.1/16 \
  --service-cidr 10.2.0.0/24 \
  --dns-service-ip 10.2.0.10
```

### Network Policy

```bash
# Bật Network Policy khi tạo cluster (không thể thêm sau)
az aks create \
  --resource-group my-rg \
  --name my-cluster \
  --network-plugin azure \
  --network-policy calico    # hoặc azure (Azure Network Policy Manager)
```

---

## Load Balancer và Ingress trên AKS

### Azure Load Balancer (Cân Bằng Tải Azure)

AKS tự động tạo Azure Standard Load Balancer khi bạn tạo Service type LoadBalancer.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"    # Internal LB
    service.beta.kubernetes.io/azure-pip-name: my-public-ip             # Dùng static IP
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

### AGIC — Application Gateway Ingress Controller

**AGIC (Application Gateway Ingress Controller — Bộ Điều Khiển Ingress Application Gateway)** tích hợp K8s Ingress với Azure Application Gateway (WAF — Web Application Firewall tích hợp).

```bash
# Bật AGIC add-on
az aks enable-addons \
  --resource-group my-rg \
  --name my-cluster \
  --addons ingress-appgw \
  --appgw-name my-appgw \
  --appgw-subnet-cidr 10.225.0.0/16
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    appgw.ingress.kubernetes.io/waf-policy-for-path: /subscriptions/.../wafPolicies/my-waf
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: my-tls-secret
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

### NGINX Ingress Controller trên AKS

```bash
# Cài NGINX Ingress qua Helm (phổ biến nhất)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```

---

## Storage trên AKS

### Azure Disk (ReadWriteOnce)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS             # Premium SSD — LRS (Locally Redundant Storage)
  cachingMode: ReadOnly
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Retain
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-disk-pvc
spec:
  accessModes:
    - ReadWriteOnce                # Azure Disk chỉ mount được 1 node
  storageClassName: managed-premium
  resources:
    requests:
      storage: 30Gi
```

### Azure Files (ReadWriteMany — Nhiều Node Đọc/Ghi)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurefile-premium
provisioner: file.csi.azure.com
parameters:
  skuName: Premium_LRS
  protocol: nfs                    # NFS protocol — hiệu năng tốt hơn SMB
allowVolumeExpansion: true
mountOptions:
  - hard
  - nfsvers=4.1
```

### Azure Blob Storage với CSI Driver

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azureblob-fuse
provisioner: blob.csi.azure.com
parameters:
  skuName: Standard_LRS
  protocol: fuse2                  # BlobFuse2 — mount Azure Blob như filesystem
```

---

## Bảo Mật AKS

### Microsoft Defender for Containers

```bash
# Bật Defender for Containers — phát hiện mối đe dọa runtime
az security pricing create \
  --name Containers \
  --tier Standard

# Defender tự động cài DaemonSet sensor trên mỗi node
# Phát hiện: anomalous process execution, crypto mining, shell script injection
```

### Azure Policy for AKS

Áp dụng chính sách bảo mật tự động cho toàn cluster — tương tự OPA Gatekeeper.

```bash
# Bật Azure Policy add-on
az aks enable-addons \
  --resource-group my-rg \
  --name my-cluster \
  --addons azure-policy

# Gán Policy initiative cho cluster
az policy assignment create \
  --name restrict-privileged-containers \
  --policy-set-definition /providers/Microsoft.Authorization/policySetDefinitions/a8640138-9b0a-4a28-b8cb-1666c838647d \
  --scope /subscriptions/<SUB_ID>/resourceGroups/my-rg
```

### Azure Key Vault Integration (Tích Hợp Lưu Trữ Khóa)

```bash
# Bật Secret Store CSI Driver với Azure Key Vault provider
az aks enable-addons \
  --resource-group my-rg \
  --name my-cluster \
  --addons azure-keyvault-secrets-provider \
  --enable-secret-rotation \
  --rotation-poll-interval 2m
```

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault-secrets
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "<MANAGED_IDENTITY_CLIENT_ID>"
    keyvaultName: my-keyvault
    tenantId: "<TENANT_ID>"
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
          objectVersion: ""
        - |
          objectName: tls-cert
          objectType: cert
          objectVersion: ""
  secretObjects:
    - secretName: db-credentials
      type: Opaque
      data:
        - objectName: db-password
          key: password
```

```yaml
# Pod dùng Key Vault secrets
spec:
  volumes:
    - name: secrets-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: azure-keyvault-secrets
  containers:
    - name: app
      volumeMounts:
        - name: secrets-store
          mountPath: /mnt/secrets-store
          readOnly: true
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

---

## Tích Hợp Azure Services

### Azure Monitor và Container Insights

```bash
# Bật Container Insights
az aks enable-addons \
  --resource-group my-rg \
  --name my-cluster \
  --addons monitoring \
  --workspace-resource-id /subscriptions/<SUB>/resourceGroups/my-rg/providers/Microsoft.OperationalInsights/workspaces/my-workspace

# Hoặc bật Managed Prometheus + Azure Managed Grafana
az aks update \
  --resource-group my-rg \
  --name my-cluster \
  --enable-azure-monitor-metrics \
  --azure-monitor-workspace-resource-id <WORKSPACE_ID> \
  --grafana-resource-id <GRAFANA_ID>
```

### Azure Container Registry (ACR — Registry Container Azure)

```bash
# Attach ACR vào AKS — node tự pull image từ ACR không cần imagePullSecret
az aks update \
  --resource-group my-rg \
  --name my-cluster \
  --attach-acr myacr

# Kiểm tra
az aks check-acr \
  --resource-group my-rg \
  --name my-cluster \
  --acr myacr
```

### Azure Service Bus / Event Hub với KEDA

**KEDA (Kubernetes Event-Driven Autoscaling — Tự Động Mở Rộng Dựa Trên Sự Kiện)** được Microsoft tạo ra — tích hợp native với Azure Event Hub, Service Bus, Storage Queue.

```bash
# Bật KEDA add-on
az aks update \
  --resource-group my-rg \
  --name my-cluster \
  --enable-keda
```

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: azure-servicebus-scaler
spec:
  scaleTargetRef:
    name: message-processor
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: my-queue
        namespace: my-servicebus-namespace
        messageCount: "10"             # 1 replica xử lý 10 message
      authenticationRef:
        name: azure-servicebus-trigger-auth
```

---

## CI/CD với Azure DevOps

### Azure Pipelines Deploy lên AKS

```yaml
# azure-pipelines.yml
trigger:
  - main

stages:
  - stage: Build
    jobs:
      - job: BuildAndPush
        pool:
          vmImage: ubuntu-latest
        steps:
          - task: Docker@2
            displayName: Build and push image
            inputs:
              command: buildAndPush
              containerRegistry: myACRServiceConnection
              repository: my-app
              tags: |
                $(Build.BuildId)
                latest

  - stage: Deploy
    dependsOn: Build
    jobs:
      - deployment: DeployToAKS
        environment: production
        pool:
          vmImage: ubuntu-latest
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@0
                  displayName: Deploy to AKS
                  inputs:
                    action: deploy
                    kubernetesServiceConnection: myAKSServiceConnection
                    namespace: production
                    manifests: |
                      k8s/deployment.yaml
                      k8s/service.yaml
                    containers: |
                      myacr.azurecr.io/my-app:$(Build.BuildId)
```

### GitOps với Flux trên AKS

```bash
# Bật Flux add-on (GitOps)
az aks enable-addons \
  --resource-group my-rg \
  --name my-cluster \
  --addons gitops

# Tạo Flux configuration
az k8s-configuration flux create \
  --resource-group my-rg \
  --cluster-name my-cluster \
  --cluster-type managedClusters \
  --name my-flux-config \
  --namespace cluster-config \
  --scope cluster \
  --url https://github.com/my-org/my-gitops-repo \
  --branch main \
  --kustomization name=infra path=./clusters/production prune=true
```

---

## Tối Ưu Chi Phí AKS

### Azure Spot Virtual Machine

```bash
az aks nodepool add \
  --resource-group my-rg \
  --cluster-name my-cluster \
  --name spotpool \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \              # Trả theo giá thị trường
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 30
```

### Azure Reserved VM Instances (VM Instance Dự Trữ)

```
1-year Reserved:  tiết kiệm ~40%
3-year Reserved:  tiết kiệm ~64%

Áp dụng cho: VM Size và Region cụ thể
Phù hợp: System Node Pool và workload baseline ổn định
```

### Start/Stop Cluster

```bash
# Dừng cluster ngoài giờ làm việc (dev/staging) — tiết kiệm chi phí VM
az aks stop \
  --resource-group my-rg \
  --name my-cluster

# Khởi động lại
az aks start \
  --resource-group my-rg \
  --name my-cluster
```

---

## Vận Hành và Upgrade

### Kết Nối Cluster

```bash
# Lấy credentials
az aks get-credentials \
  --resource-group my-rg \
  --name my-cluster

# Kiểm tra nodes
kubectl get nodes -o wide

# Xem phiên bản cluster
az aks show \
  --resource-group my-rg \
  --name my-cluster \
  --query kubernetesVersion
```

### Upgrade Cluster

```bash
# Xem phiên bản K8s có sẵn
az aks get-upgrades \
  --resource-group my-rg \
  --name my-cluster

# Upgrade Control Plane (từng phiên bản một)
az aks upgrade \
  --resource-group my-rg \
  --name my-cluster \
  --kubernetes-version 1.30.0 \
  --control-plane-only             # Chỉ upgrade Control Plane

# Upgrade node pool
az aks nodepool upgrade \
  --resource-group my-rg \
  --cluster-name my-cluster \
  --name nodepool1 \
  --kubernetes-version 1.30.0 \
  --max-surge 1                    # Tạo thêm 1 node khi upgrade
```

### Auto-upgrade

```bash
# Bật auto-upgrade
az aks update \
  --resource-group my-rg \
  --name my-cluster \
  --auto-upgrade-channel patch      # patch: tự upgrade patch version
                                    # rapid / stable / node-image: các level khác

# Cấu hình maintenance window
az aks maintenanceconfiguration add \
  --resource-group my-rg \
  --cluster-name my-cluster \
  --name default \
  --weekday Saturday \
  --start-hour 22                   # 22:00 UTC thứ Bảy
```

### Backup với Velero

```bash
# Cài Velero với Azure Blob Storage backend
velero install \
  --provider azure \
  --plugins velero/velero-plugin-for-microsoft-azure:v1.9.0 \
  --bucket my-velero-backups \
  --secret-file ./credentials-velero \
  --backup-location-config resourceGroup=my-rg,storageAccount=myvelero,subscriptionId=<SUB_ID>

# Tạo backup namespace production
velero backup create prod-backup \
  --include-namespaces production \
  --storage-location default
```

---

## Câu Hỏi Phỏng Vấn

**Q: Tại sao chọn AKS thay vì EKS hay GKE?**

> Tôi chọn AKS khi: (1) Hệ thống enterprise đang dùng Azure với Active Directory — tích hợp Azure AD với K8s RBAC là native và đơn giản nhất trên AKS. (2) Cần chạy Windows container (.NET Framework, SQL Server Agent) — AKS hỗ trợ tốt nhất. (3) Đội dùng Azure DevOps — pipeline tích hợp với AKS deployment trực tiếp. (4) Control Plane miễn phí — tiết kiệm chi phí so với EKS.

**Q: Workload Identity trên AKS khác AAD Pod Identity thế nào?**

> AAD Pod Identity (thế hệ cũ) dùng iptables để intercept metadata server request, cài thêm MIC và NMI DaemonSet — phức tạp, có vấn đề hiệu năng và race condition. Workload Identity (thế hệ mới) dùng OIDC federation chuẩn — K8s phát JWT token, Azure STS xác thực, trả về access token của Managed Identity. Đơn giản hơn, an toàn hơn, performance tốt hơn vì không có iptables intercept. Microsoft đã deprecate AAD Pod Identity, khuyến nghị migrate sang Workload Identity.

**Q: Azure CNI và Kubenet khác nhau thế nào? Khi nào dùng mỗi loại?**

> Kubenet: Pod có IP từ pod CIDR riêng, Azure tạo route table để forward — đơn giản, tiết kiệm IP VNet nhưng giới hạn 400 node. Azure CNI: Pod nhận IP thực từ VNet subnet — yêu cầu nhiều IP hơn nhưng tích hợp native với VNet, hỗ trợ Windows container, không có overhead route table. Tôi dùng Azure CNI cho production khi cần Windows container, VNet peering với on-premise, hoặc cần kết nối trực tiếp vào Pod từ VNet. Kubenet cho cluster nhỏ, dev/staging, không cần Windows.

**Q: Làm thế nào để deploy .NET Framework application lên AKS?**

> (1) Tạo cluster với `--network-plugin azure` và thêm Windows node pool. (2) Build Docker image từ base image `mcr.microsoft.com/dotnet/framework/aspnet:4.8-windowsservercore-ltsc2022`. (3) Push image lên Azure Container Registry. (4) Deploy với `nodeSelector: kubernetes.io/os: windows` để Pod schedule lên Windows node. (5) Expose qua Azure Load Balancer hoặc Application Gateway. Lưu ý: Windows container thường lớn hơn (2–5 GB) — pull time lâu hơn, nên pre-pull image hoặc dùng node với image đã cache.

**Q: KEDA là gì và khi nào dùng trên AKS?**

> KEDA (Kubernetes Event-Driven Autoscaling) scale Pod dựa trên event từ message queue, cron, metric từ external system — thay vì chỉ dựa vào CPU/memory như HPA. Trên AKS, KEDA tích hợp native với Azure Service Bus, Event Hub, Storage Queue, Azure Monitor. Dùng khi: có consumer service cần scale theo độ dài queue (ví dụ: email worker scale lên khi có 1000 email cần gửi, scale xuống 0 khi queue rỗng). KEDA có thể scale to 0 — tiết kiệm hơn HPA vì HPA minimum là 1 replica.

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
