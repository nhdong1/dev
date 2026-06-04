# Sealed Secrets — Bí Mật Được Niêm Phong Cho GitOps

> Hướng dẫn sử dụng Sealed Secrets (Bitnami) để mã hoá Secret an toàn bằng mã hoá bất đối xứng (asymmetric encryption), cho phép lưu secret trên Git mà không lộ nội dung — nền tảng cho GitOps workflow bảo mật.

## Mục Lục

1. [Vấn Đề GitOps Với Secret](#vấn-đề-gitops-với-secret)
2. [Sealed Secrets Là Gì?](#sealed-secrets-là-gì)
3. [Cài Đặt](#cài-đặt)
4. [Tạo và Áp Dụng SealedSecret](#tạo-và-áp-dụng-sealedsecret)
5. [Scope — Phạm Vi Mã Hoá](#scope--phạm-vi-mã-hoá)
6. [Quản Lý Khoá — Key Management](#quản-lý-khoá--key-management)
7. [Tích Hợp Với ArgoCD và Flux](#tích-hợp-với-argocd-và-flux)
8. [So Sánh Sealed Secrets vs External Secrets](#so-sánh-sealed-secrets-vs-external-secrets)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Vấn Đề GitOps Với Secret

GitOps là mô hình vận hành lưu toàn bộ manifest Kubernetes trên Git và dùng tool như ArgoCD / Flux để đồng bộ cluster. Vấn đề:

```
GitOps Workflow Lý Tưởng:
Git Repository
  ├── deployment.yaml    ← an toàn để commit
  ├── service.yaml       ← an toàn để commit
  └── secret.yaml        ← ❌ KHÔNG BAO GIỜ commit Secret plaintext!

Nếu commit Secret:
  - Lịch sử Git lưu mãi mãi — xoá file không xoá history
  - Bất kỳ ai clone repo đều thấy secret
  - Nếu repo là public → credential bị lộ toàn thế giới
```

**Các giải pháp phổ biến:**

| Giải Pháp | Ưu Điểm | Nhược Điểm |
| --------- | ------- | ---------- |
| Bỏ Secret khỏi Git | Đơn giản | Không GitOps-native, phải quản lý thủ công |
| External Secrets (Vault, AWS) | Auto-rotation, audit | Phụ thuộc hệ thống ngoài, phức tạp |
| **Sealed Secrets** | Commit an toàn trên Git | Rotation phải re-seal, phụ thuộc controller |
| SOPS (Secrets OPerationS) | Linh hoạt, nhiều backend | Phức tạp hơn, cần key management |

---

## Sealed Secrets Là Gì?

**Sealed Secrets** (của Bitnami/VMware) là giải pháp mã hoá Secret bằng **mã hoá bất đối xứng** (asymmetric encryption — dùng cặp public/private key):

```
Cách hoạt động:
──────────────
1. Sealed Secrets Controller chạy trong cluster
   └── Có private key (chỉ controller biết)
   └── Public key chia sẻ công khai

2. Developer lấy public key:
   kubeseal --fetch-cert > pub-key.pem

3. Developer mã hoá Secret:
   Secret (nhạy cảm) + Public Key → SealedSecret (an toàn commit Git)

4. Áp dụng SealedSecret lên cluster:
   kubectl apply -f sealed-db-credentials.yaml

5. Controller giải mã:
   SealedSecret + Private Key → Secret (trong cluster)

6. Pod dùng Secret như thường.
```

**Tính chất bảo mật:**
- Chỉ cluster có private key mới giải mã được
- Hacker có SealedSecret + public key vẫn **không** giải mã được
- Mỗi cluster có cặp key riêng — SealedSecret của cluster A không dùng được ở cluster B

---

## Cài Đặt

### Cài Sealed Secrets Controller

```bash
# Cài bằng Helm (khuyến nghị)
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update

helm install sealed-secrets \
  sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --set fullnameOverride=sealed-secrets-controller

# Xác nhận controller đang chạy
kubectl get pods -n kube-system -l app.kubernetes.io/name=sealed-secrets
# NAME                                        READY   STATUS    RESTARTS
# sealed-secrets-controller-7f9b6d5c4-xk2pl  1/1     Running   0

# Xem log controller
kubectl logs -n kube-system -l app.kubernetes.io/name=sealed-secrets
```

### Cài kubeseal CLI

`kubeseal` là công cụ dòng lệnh để mã hoá Secret thành SealedSecret:

```bash
# macOS
brew install kubeseal

# Linux (tải binary)
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest | jq -r '.tag_name')
curl -L "https://github.com/bitnami-labs/sealed-secrets/releases/download/${KUBESEAL_VERSION}/kubeseal-$(uname -s | tr '[:upper:]' '[:lower:]')-amd64" \
  -o kubeseal
chmod +x kubeseal
mv kubeseal /usr/local/bin/

# Xác nhận
kubeseal --version
```

---

## Tạo và Áp Dụng SealedSecret

### Quy Trình Cơ Bản

```bash
# Bước 1: Tạo Secret thông thường (chưa apply vào cluster)
kubectl create secret generic db-credentials \
  --from-literal=username=postgres \
  --from-literal=password='superSecret@123' \
  --dry-run=client \          # QUAN TRỌNG: dry-run, không apply thật
  -o yaml > /tmp/db-secret.yaml

# Bước 2: Mã hoá thành SealedSecret
kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --format yaml \
  < /tmp/db-secret.yaml \
  > sealed-db-credentials.yaml

# Bước 3: Xoá file Secret gốc (nhạy cảm)
rm /tmp/db-secret.yaml

# Bước 4: Kiểm tra SealedSecret (an toàn để xem)
cat sealed-db-credentials.yaml
```

### Nội Dung SealedSecret Sau Khi Seal

```yaml
# sealed-db-credentials.yaml — AN TOÀN ĐỂ COMMIT GIT
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: production
  creationTimestamp: null
spec:
  encryptedData:
    # Giá trị được mã hoá — không thể đọc được nếu không có private key
    password: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...long-encrypted-string...
    username: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...another-encrypted-string...
  template:
    metadata:
      name: db-credentials
      namespace: production
    type: Opaque
```

### Apply SealedSecret Lên Cluster

```bash
# Apply (controller tự giải mã và tạo Secret)
kubectl apply -f sealed-db-credentials.yaml

# Xác nhận SealedSecret được tạo
kubectl get sealedsecret -n production
# NAME             AGE   STATUS   SYNCED
# db-credentials   30s   True

# Xác nhận Secret được tạo từ SealedSecret
kubectl get secret db-credentials -n production
# NAME             TYPE     DATA   AGE
# db-credentials   Opaque   2      25s

# Xem giá trị Secret (base64 decode)
kubectl get secret db-credentials -n production \
  -o jsonpath='{.data.password}' | base64 -d
```

### Seal Từ File Secret YAML

```yaml
# secret-template.yaml (để tạo SealedSecret — đừng commit file này)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
stringData:
  DB_PASSWORD: "myDatabasePassword"
  API_KEY: "sk-abc123xyz"
  JWT_SECRET: "myJwtSecretKey256bits"
  SMTP_PASSWORD: "emailServicePassword"
```

```bash
kubeseal --format yaml < secret-template.yaml > sealed-app-secrets.yaml
# Commit sealed-app-secrets.yaml lên Git ✅
# ĐỪNG commit secret-template.yaml ❌
```

---

## Scope — Phạm Vi Mã Hoá

Sealed Secrets hỗ trợ ba phạm vi (scope) mã hoá, ảnh hưởng đến khả năng di chuyển SealedSecret giữa namespace:

### `strict` — Nghiêm Ngặt (Mặc Định)

SealedSecret chỉ giải mã được đúng namespace và tên như khai báo:

```bash
kubeseal \
  --scope strict \         # mặc định — không cần khai báo tường minh
  --format yaml \
  < secret.yaml > sealed-secret.yaml

# SealedSecret chỉ dùng được ở đúng namespace "production" và tên "db-credentials"
# Nếu apply sang namespace "staging" → lỗi giải mã
```

### `namespace-wide` — Toàn Namespace

SealedSecret giải mã được ở bất kỳ tên nào trong namespace khai báo:

```bash
kubeseal \
  --scope namespace-wide \
  --format yaml \
  < secret.yaml > sealed-secret.yaml

# Có thể dùng ở namespace "production" với bất kỳ tên Secret nào
# Không dùng được ở namespace khác
```

### `cluster-wide` — Toàn Cluster

SealedSecret giải mã được ở bất kỳ namespace và tên nào:

```bash
kubeseal \
  --scope cluster-wide \
  --format yaml \
  < secret.yaml > sealed-secret.yaml

# Dùng được ở mọi namespace — phù hợp cho shared secret (TLS cert, registry credentials)
```

**Lưu ý bảo mật:** Chỉ dùng `cluster-wide` khi thực sự cần — scope nhỏ nhất là an toàn nhất.

---

## Quản Lý Khoá — Key Management

### Xem và Backup Key Hiện Tại

```bash
# Lấy public key (chia sẻ được — dùng để seal)
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  > pub-key.pem

# Xem private key (NHẠY CẢM — backup cẩn thận)
kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key=active \
  -o yaml > sealed-secrets-keys-backup.yaml

# LƯU FILE NÀY Ở NƠI AN TOÀN — MẤT KEY LÀ MẤT KHẢ NĂNG GIẢI MÃ!
```

### Key Rotation — Xoay Vòng Khoá

Sealed Secrets tự động tạo key mới mỗi 30 ngày (mặc định), giữ lại key cũ để giải mã SealedSecret cũ:

```bash
# Xem tất cả key (active và old)
kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key

# Controller tự động dùng key mới nhất để giải mã SealedSecret mới
# SealedSecret cũ vẫn được giải mã bằng key cũ

# Force tạo key mới ngay (cho tình huống khẩn cấp)
kubectl label secret -n kube-system \
  <old-key-name> \
  sealedsecrets.bitnami.com/sealed-secrets-key=compromised

# Sau khi tạo key mới, phải re-seal toàn bộ SealedSecret với key mới
```

### Khôi Phục Controller Từ Backup

Kịch bản: cluster bị xoá, cần restore Sealed Secrets:

```bash
# Bước 1: Cài Sealed Secrets Controller mới
helm install sealed-secrets ...

# Bước 2: Khôi phục key từ backup
kubectl apply -f sealed-secrets-keys-backup.yaml

# Bước 3: Restart controller để load key
kubectl rollout restart deployment/sealed-secrets-controller -n kube-system

# Bước 4: Apply SealedSecret từ Git — controller giải mã bằng key đã restore
kubectl apply -f sealed-app-secrets.yaml
```

### Seal Với Public Key Offline

Khi không có kubectl context tới cluster (CI/CD pipeline, developer offline):

```bash
# Bước 1: Admin export public key và lưu vào repo
kubeseal --fetch-cert > certs/sealed-secrets-pub-key.pem

# Bước 2: Developer dùng public key để seal (không cần kết nối cluster)
kubeseal \
  --cert certs/sealed-secrets-pub-key.pem \
  --format yaml \
  < /tmp/secret.yaml > sealed-secret.yaml
```

---

## Tích Hợp Với ArgoCD và Flux

### Với ArgoCD

ArgoCD tự nhận ra `SealedSecret` CRD và deploy như resource thông thường. Controller trong cluster giải mã khi apply:

```yaml
# Git repository structure (toàn bộ safe để commit)
k8s-manifests/
  production/
    deployment.yaml
    service.yaml
    sealed-db-credentials.yaml     ← SealedSecret an toàn
    sealed-api-keys.yaml           ← SealedSecret an toàn

# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/myorg/k8s-manifests
    path: production
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Với Flux

```yaml
# Flux tự động apply SealedSecret từ Git
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 5m
  path: "./production"
  prune: true
  sourceRef:
    kind: GitRepository
    name: k8s-manifests
  # Flux biết cách handle SealedSecret qua controller trong cluster
```

---

## So Sánh Sealed Secrets vs External Secrets

| Tiêu Chí | Sealed Secrets | External Secrets Operator |
| -------- | -------------- | ------------------------- |
| **Cơ chế** | Mã hoá bất đối xứng, lưu trên Git | Đồng bộ từ provider bên ngoài |
| **Phụ thuộc** | Chỉ controller trong cluster | Cần provider (AWS, Vault, GCP...) |
| **GitOps** | ✅ Secret có thể lưu trên Git | ⚠️ Secret không lưu trên Git (kém native hơn) |
| **Auto-rotation** | ❌ Phải re-seal thủ công | ✅ Tự đồng bộ theo `refreshInterval` |
| **Audit trail** | K8s audit log | Audit log phong phú từ provider |
| **Setup phức tạp** | Thấp | Trung bình đến cao |
| **Chi phí** | Không (chỉ controller) | Phí provider (AWS Secrets Manager: ~$0.40/secret/tháng) |
| **Multi-cluster** | Mỗi cluster có key riêng | Dễ chia sẻ secret qua nhiều cluster |
| **Secret rotation** | Thủ công + re-seal + re-commit | Tự động |
| **Offline capability** | ✅ Seal offline với public key | ❌ Cần kết nối provider |

**Khi nào dùng Sealed Secrets:**
- Team nhỏ, không có Vault hay AWS Secrets Manager
- Muốn GitOps-native — toàn bộ state trên Git
- Secret thay đổi không thường xuyên
- Budget hạn chế

**Khi nào dùng External Secrets:**
- Đã có Vault hoặc AWS Secrets Manager trong tổ chức
- Cần auto-rotation (compliance yêu cầu rotate định kỳ)
- Nhiều cluster chia sẻ cùng secret
- Team lớn cần audit trail chi tiết

---

## Câu Hỏi Phỏng Vấn

**Sealed Secrets giải quyết vấn đề gì và hoạt động như thế nào?**

> Sealed Secrets giải quyết bài toán: làm thế nào commit secret vào Git trong GitOps workflow mà không lộ nội dung. Cơ chế: Controller trong cluster nắm giữ **private key** (bí mật). Public key được chia sẻ công khai. Developer dùng `kubeseal` mã hoá Secret bằng public key tạo ra `SealedSecret` — file này an toàn để commit Git vì chỉ controller mới giải mã được bằng private key. Mã hoá là **hybrid**: RSA (4096-bit) mã hoá symmetric key, symmetric key mã hoá nội dung Secret thật.

**Điều gì xảy ra nếu mất private key của Sealed Secrets Controller?**

> **Thảm hoạ** — mất private key đồng nghĩa với việc không thể giải mã bất kỳ SealedSecret nào trên Git. Cần re-seal toàn bộ secret bằng key mới, điều này đòi hỏi truy cập vào giá trị secret gốc. Vì vậy **backup private key là bắt buộc**: `kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key=active -o yaml` và lưu ở nơi an toàn (không phải Git). Đây là khác biệt quan trọng so với External Secrets — ESO không có vấn đề này vì secret sống ở provider bên ngoài.

**scope trong Sealed Secrets là gì? Khi nào dùng cluster-wide?**

> `scope` quyết định ngữ cảnh (namespace + tên) mà SealedSecret có thể giải mã: `strict` (mặc định) — chỉ đúng namespace và tên; `namespace-wide` — bất kỳ tên nào trong namespace; `cluster-wide` — bất kỳ namespace và tên nào. Dùng `cluster-wide` cho shared resource cần dùng nhiều namespace: wildcard TLS certificate, Docker registry credential. Tuy nhiên `cluster-wide` SealedSecret có thể bị copy sang namespace khác và vẫn hoạt động — ít an toàn hơn, chỉ dùng khi thực sự cần.

**Làm thế nào để rotate secret khi dùng Sealed Secrets?**

> Không có auto-rotation — đây là nhược điểm lớn nhất. Quy trình thủ công: (1) Tạo Secret mới với credential mới; (2) Seal bằng `kubeseal` tạo SealedSecret mới; (3) Commit và push SealedSecret mới lên Git; (4) ArgoCD/Flux apply vào cluster; (5) Controller giải mã và cập nhật K8s Secret; (6) Pod tự reload nếu dùng volume, hoặc restart nếu dùng env var. Với Sealed Secrets, rotation phải được trigger thủ công và cần giám sát expiry của secret. Đây là lý do tổ chức cần auto-rotation thường chọn External Secrets + Vault thay vì Sealed Secrets.
