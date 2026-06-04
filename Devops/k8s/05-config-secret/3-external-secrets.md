# External Secrets Operator — Đồng Bộ Secret Từ Hệ Thống Bên Ngoài

> Hướng dẫn sử dụng External Secrets Operator (ESO — Nhà Vận Hành Bí Mật Bên Ngoài) để tự động đồng bộ secret từ AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager và các provider khác vào Kubernetes Secret.

## Mục Lục

1. [Vấn Đề ESO Giải Quyết](#vấn-đề-eso-giải-quyết)
2. [Kiến Trúc ESO](#kiến-trúc-eso)
3. [Cài Đặt ESO](#cài-đặt-eso)
4. [Tích Hợp AWS Secrets Manager](#tích-hợp-aws-secrets-manager)
5. [Tích Hợp HashiCorp Vault](#tích-hợp-hashicorp-vault)
6. [ClusterSecretStore — Dùng Chung Cho Nhiều Namespace](#clustersecretstore--dùng-chung-cho-nhiều-namespace)
7. [ExternalSecret Nâng Cao](#externalsecret-nâng-cao)
8. [Monitoring và Troubleshooting](#monitoring-và-troubleshooting)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Vấn Đề ESO Giải Quyết

**Vấn đề:** Secret K8s thuần có nhiều hạn chế trong môi trường enterprise:

```
Secret K8s Thuần — Vấn Đề
──────────────────────────
❌ Lưu base64 trong etcd — không đủ bảo mật cho regulated industry
❌ Không có auto-rotation — phải update thủ công khi đổi password
❌ Không có audit trail chi tiết — khó tuân thủ compliance (SOC2, PCI-DSS)
❌ Không tích hợp với hệ thống IAM tập trung của tổ chức
❌ Không thể chia sẻ secret dễ dàng giữa nhiều cluster
```

**Giải pháp:** Giữ secret trong hệ thống quản lý chuyên biệt (Vault, AWS Secrets Manager), ESO tự động kéo về và tạo K8s Secret:

```
Vault / AWS Secrets Manager / GCP Secret Manager
        ↓  (ESO đọc định kỳ)
External Secrets Operator
        ↓  (tạo/cập nhật)
Kubernetes Secret
        ↓  (mount như thường)
Pod / Container
```

---

## Kiến Trúc ESO

```
┌─────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                │
│                                                     │
│  ┌─────────────────┐     ┌──────────────────────┐  │
│  │   SecretStore   │     │   ExternalSecret      │  │
│  │  (credentials   │     │  (định nghĩa secret   │  │
│  │   kết nối       │     │   cần đồng bộ)        │  │
│  │   provider)     │     └──────────┬───────────┘  │
│  └────────┬────────┘                │               │
│           │                         │               │
│           └──────────┐  ┌───────────┘               │
│                      ▼  ▼                           │
│           ┌──────────────────────┐                  │
│           │  ESO Controller      │                  │
│           │  (reconcile loop)    │                  │
│           └──────────┬───────────┘                  │
│                      │  tạo/cập nhật                │
│                      ▼                              │
│           ┌──────────────────────┐                  │
│           │   Kubernetes Secret  │                  │
│           └──────────────────────┘                  │
│                                                     │
└─────────────────────────────────────────────────────┘
         ↑ đọc secret
┌────────────────────────┐
│  AWS Secrets Manager   │
│  HashiCorp Vault       │
│  GCP Secret Manager    │
│  Azure Key Vault       │
└────────────────────────┘
```

### Các CRD (Custom Resource Definition) Của ESO

| CRD | Phạm Vi | Mục Đích |
| --- | ------- | -------- |
| `SecretStore` | Namespace | Kết nối với provider trong một namespace |
| `ClusterSecretStore` | Cluster-wide | Kết nối dùng chung cho toàn cluster |
| `ExternalSecret` | Namespace | Định nghĩa secret cần đồng bộ |
| `ClusterExternalSecret` | Cluster-wide | Đồng bộ secret vào nhiều namespace |
| `PushSecret` | Namespace | Đẩy K8s Secret ra provider bên ngoài |

---

## Cài Đặt ESO

```bash
# Cài bằng Helm (khuyến nghị)
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set installCRDs=true

# Xác nhận cài đặt thành công
kubectl get pods -n external-secrets
# NAME                                               READY   STATUS    RESTARTS
# external-secrets-7d8f9b6c4-xk2pl                  1/1     Running   0
# external-secrets-cert-controller-6b9d7c5f4-p8qmn  1/1     Running   0
# external-secrets-webhook-84d6f5c9b-r7wlk           1/1     Running   0

kubectl get crd | grep external-secrets
# clustersecretstores.external-secrets.io
# externalsecrets.external-secrets.io
# secretstores.external-secrets.io
```

---

## Tích Hợp AWS Secrets Manager

### Bước 1: Cấp Quyền Cho ESO Truy Cập AWS

**Cách tốt nhất:** Dùng IRSA (IAM Roles for Service Accounts — Vai Trò IAM Cho ServiceAccount) thay vì access key:

```json
// IAM Policy cho ESO
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret",
        "secretsmanager:ListSecretVersionIds"
      ],
      "Resource": [
        "arn:aws:secretsmanager:us-east-1:123456789:secret:production/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath"
      ],
      "Resource": [
        "arn:aws:ssm:us-east-1:123456789:parameter/production/*"
      ]
    }
  ]
}
```

```bash
# Annotate ServiceAccount của ESO với IAM role
kubectl annotate serviceaccount external-secrets \
  -n external-secrets \
  eks.amazonaws.com/role-arn=arn:aws:iam::123456789:role/ESO-Role
```

### Bước 2: Tạo SecretStore

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager        # hoặc ParameterStore
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets   # ServiceAccount có IRSA annotation
```

Hoặc dùng access key (kém bảo mật hơn — chỉ dùng khi không có IRSA):
```yaml
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-credentials
            key: access-key-id
          secretAccessKeySecretRef:
            name: aws-credentials
            key: secret-access-key
```

### Bước 3: Tạo ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 15m             # tần suất đồng bộ (15 phút)

  secretStoreRef:
    name: aws-secrets-manager      # tên SecretStore
    kind: SecretStore

  target:
    name: db-credentials           # tên K8s Secret sẽ tạo
    creationPolicy: Owner          # ESO quản lý Secret này
    deletionPolicy: Retain         # không xoá Secret khi ExternalSecret bị xoá

  data:
    # Lấy từ AWS Secrets Manager JSON secret
    - secretKey: username          # key trong K8s Secret
      remoteRef:
        key: production/database   # tên secret trên AWS
        property: username         # field trong JSON secret

    - secretKey: password
      remoteRef:
        key: production/database
        property: password

    - secretKey: host
      remoteRef:
        key: production/database
        property: host

  # Hoặc dùng dataFrom để lấy toàn bộ JSON secret
  # dataFrom:
  #   - extract:
  #       key: production/database  # toàn bộ JSON key-value → K8s Secret
```

**Trên AWS Secrets Manager:**
```json
// Secret name: "production/database"
{
  "username": "postgres",
  "password": "superSecurePass@2024",
  "host": "prod-db.cluster.us-east-1.rds.amazonaws.com",
  "port": "5432"
}
```

### Tích Hợp AWS Parameter Store (Kho Tham Số)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-parameter-store
  namespace: production
spec:
  provider:
    aws:
      service: ParameterStore      # Parameter Store thay vì Secrets Manager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-params
  namespace: production
spec:
  refreshInterval: 30m
  secretStoreRef:
    name: aws-parameter-store
    kind: SecretStore
  target:
    name: app-params
  dataFrom:
    - find:
        path: /production/myapp    # lấy toàn bộ parameter dưới path này
        tags:
          Environment: production  # lọc theo tag (tuỳ chọn)
```

---

## Tích Hợp HashiCorp Vault

### Bước 1: Cấu Hình Vault Kubernetes Auth

```bash
# Trên Vault server — bật Kubernetes auth method
vault auth enable kubernetes

# Cấu hình Vault kết nối với Kubernetes cluster
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Tạo policy cho ESO
vault policy write eso-policy - <<EOF
path "secret/data/production/*" {
  capabilities = ["read"]
}
path "kv/data/production/*" {
  capabilities = ["read"]
}
EOF

# Tạo role binding ServiceAccount với policy
vault write auth/kubernetes/role/eso-role \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=eso-policy \
  ttl=1h
```

### Bước 2: Tạo SecretStore Cho Vault

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: "https://vault.internal:8200"
      path: "secret"               # Vault mount path (KV v2)
      version: "v2"                # KV version (v1 hoặc v2)
      caBundle: |                  # CA certificate của Vault (base64)
        LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...
      auth:
        kubernetes:
          mountPath: "kubernetes"  # auth method mount path
          role: "eso-role"         # Vault role đã tạo ở trên
          serviceAccountRef:
            name: external-secrets
```

### Bước 3: Tạo ExternalSecret Cho Vault

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-db-secret
  namespace: production
spec:
  refreshInterval: 1h

  secretStoreRef:
    name: vault-backend
    kind: SecretStore

  target:
    name: db-credentials

  data:
    - secretKey: password
      remoteRef:
        key: production/database       # đường dẫn secret trong Vault
        property: password             # field trong secret

    - secretKey: username
      remoteRef:
        key: production/database
        property: username

  # Dynamic secrets — Vault tạo credential tạm thời (TTL)
  # dataFrom:
  #   - sourceRef:
  #       generatorRef:
  #         apiVersion: generators.external-secrets.io/v1alpha1
  #         kind: VaultDynamicSecret
  #         name: vault-dynamic-db
```

### Vault Dynamic Secrets — Credential Tạm Thời

```yaml
# Vault tạo database credential tạm thời — tự expire sau TTL
apiVersion: generators.external-secrets.io/v1alpha1
kind: VaultDynamicSecret
metadata:
  name: vault-dynamic-db
  namespace: production
spec:
  path: database/creds/readonly-role   # Vault dynamic secrets path
  method: GET
  provider:
    server: "https://vault.internal:8200"
    auth:
      kubernetes:
        mountPath: kubernetes
        role: eso-role
        serviceAccountRef:
          name: external-secrets
```

---

## ClusterSecretStore — Dùng Chung Cho Nhiều Namespace

Khi nhiều namespace cần dùng chung một provider, dùng `ClusterSecretStore` thay vì tạo `SecretStore` riêng cho mỗi namespace:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-global          # không có namespace — cluster-wide
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets    # namespace của ESO ServiceAccount
```

ExternalSecret trong bất kỳ namespace nào có thể dùng ClusterSecretStore:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-secret
  namespace: team-a           # bất kỳ namespace nào
spec:
  secretStoreRef:
    name: aws-global
    kind: ClusterSecretStore  # dùng cluster-wide store
  target:
    name: my-secret
  data:
    - secretKey: api-key
      remoteRef:
        key: team-a/api-key
```

---

## ExternalSecret Nâng Cao

### Template — Biến Đổi Secret Trước Khi Tạo

```yaml
spec:
  target:
    name: connection-string
    template:
      type: Opaque
      data:
        # Kết hợp nhiều secret thành một connection string
        DATABASE_URL: "postgresql://{{ .username }}:{{ .password }}@{{ .host }}:5432/{{ .database }}"
        REDIS_URL: "redis://:{{ .redisPassword }}@{{ .redisHost }}:6379/0"

  data:
    - secretKey: username
      remoteRef:
        key: production/database
        property: username
    - secretKey: password
      remoteRef:
        key: production/database
        property: password
    - secretKey: host
      remoteRef:
        key: production/database
        property: host
    - secretKey: database
      remoteRef:
        key: production/database
        property: dbname
    - secretKey: redisPassword
      remoteRef:
        key: production/redis
        property: password
    - secretKey: redisHost
      remoteRef:
        key: production/redis
        property: host
```

### ClusterExternalSecret — Đồng Bộ Vào Nhiều Namespace

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterExternalSecret
metadata:
  name: shared-tls-cert
spec:
  namespaceSelector:
    matchLabels:
      requires-tls: "true"     # áp dụng cho mọi namespace có label này
  refreshTime: 1h
  externalSecretSpec:
    refreshInterval: 1h
    secretStoreRef:
      name: aws-global
      kind: ClusterSecretStore
    target:
      name: wildcard-tls       # tên Secret tạo trong mỗi namespace
    data:
      - secretKey: tls.crt
        remoteRef:
          key: production/wildcard-cert
          property: certificate
      - secretKey: tls.key
        remoteRef:
          key: production/wildcard-cert
          property: private_key
```

---

## Monitoring và Troubleshooting

### Xem Trạng Thái ExternalSecret

```bash
# Liệt kê ExternalSecret và trạng thái
kubectl get externalsecret -n production
# NAME             STORE                AGE   STATUS   READY
# db-credentials   aws-secrets-manager  5m    Valid    True
# api-keys         aws-secrets-manager  2m    Invalid  False

# Xem chi tiết lỗi
kubectl describe externalsecret db-credentials -n production

# Xem condition chi tiết
kubectl get externalsecret db-credentials -o jsonpath='{.status.conditions}' | jq .
# [
#   {
#     "type": "Ready",
#     "status": "True",
#     "reason": "SecretSynced",
#     "message": "Secret was synced",
#     "lastTransitionTime": "2026-05-10T10:00:00Z"
#   }
# ]
```

### Lỗi Thường Gặp

```bash
# Lỗi: "Could not find SecretStore"
# → Kiểm tra tên và namespace của SecretStore
kubectl get secretstore -n production

# Lỗi: "Access Denied" hoặc "Unauthorized"
# → Kiểm tra IRSA annotation hoặc credentials
kubectl describe serviceaccount external-secrets -n external-secrets

# Lỗi: "Secret not found in provider"
# → Kiểm tra tên secret trên AWS / Vault
aws secretsmanager describe-secret --secret-id production/database

# Force sync ngay lập tức (thêm annotation)
kubectl annotate externalsecret db-credentials \
  force-sync=$(date +%s) \
  --overwrite \
  -n production
```

### Metrics Prometheus

ESO export các metric quan trọng:
```
externalsecrets_sync_calls_total          # tổng số lần sync
externalsecrets_sync_call_errors_total    # số lần sync lỗi
externalsecrets_secret_without_owner_count # Secret không có ExternalSecret quản lý
```

---

## Câu Hỏi Phỏng Vấn

**External Secrets Operator khác gì so với Secret K8s thuần?**

> K8s Secret thuần yêu cầu lưu dữ liệu nhạy cảm trong cluster (etcd) và quản lý thủ công. External Secrets Operator (ESO) là bridge pattern — secret thật sự sống ở hệ thống bên ngoài (AWS Secrets Manager, Vault), ESO chỉ đồng bộ bản sao vào K8s Secret để Pod đọc. Lợi ích: (1) Một nguồn sự thật duy nhất cho secret; (2) Auto-rotation — khi Vault đổi credential, ESO đồng bộ tự động theo `refreshInterval`; (3) Audit log chi tiết qua provider; (4) Secret không cần lưu trong Git hay CI/CD pipeline.

**refreshInterval hoạt động như thế nào? Nên đặt bao lâu?**

> `refreshInterval` là chu kỳ ESO kiểm tra provider và cập nhật K8s Secret. ESO so sánh giá trị hiện tại với provider — nếu khác nhau mới update K8s Secret (tránh storm API calls). Chọn interval: (1) **15m–1h** cho secret thay đổi thường xuyên (API key, database password có auto-rotation); (2) **6h–24h** cho certificate TLS (thường rotate hàng tháng); (3) Tránh < 5m trên môi trường nhiều ExternalSecret vì tăng load lên provider. Sau khi K8s Secret update, Pod vẫn cần mechanism để reload (nếu dùng env var cần restart; nếu dùng volume cần ứng dụng hot-reload hoặc kết hợp Reloader).

**Làm thế nào để ESO truy cập AWS mà không cần lưu access key?**

> Dùng **IRSA (IAM Roles for Service Accounts)** trên EKS: (1) Tạo IAM role với trust policy cho OIDC provider của EKS cluster; (2) Annotate ServiceAccount của ESO với ARN của role: `eks.amazonaws.com/role-arn=arn:aws:iam::...`; (3) Khi ESO cần gọi AWS API, EKS inject temporary credentials qua projected service account token, ESO dùng credential này để gọi Secrets Manager. Không cần lưu bất kỳ static credential nào trong cluster — đây là zero-secret-to-manage cho ESO authentication.
