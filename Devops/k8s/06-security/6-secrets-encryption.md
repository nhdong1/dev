# Secrets Encryption — Mã Hoá Bí Mật Kubernetes

> Hướng dẫn chi tiết về Encryption at Rest (Mã Hoá Tại Nơi Lưu Trữ) cho etcd trong Kubernetes: cấu hình EncryptionConfiguration, tích hợp KMS (Key Management Service — Dịch Vụ Quản Lý Khoá) với AWS KMS, GCP KMS, và HashiCorp Vault KMS Plugin.

## Mục Lục

1. [Vấn Đề Bảo Mật Secret Mặc Định](#vấn-đề-bảo-mật-secret-mặc-định)
2. [Encryption at Rest — Mã Hoá Tại Etcd](#encryption-at-rest--mã-hoá-tại-etcd)
3. [Cấu Hình EncryptionConfiguration](#cấu-hình-encryptionconfiguration)
4. [KMS Plugin — Mã Hoá Với Key Bên Ngoài](#kms-plugin--mã-hoá-với-key-bên-ngoài)
5. [Managed Cluster — Encryption Tự Động](#managed-cluster--encryption-tự-động)
6. [Envelope Encryption — Mã Hoá Phong Bì](#envelope-encryption--mã-hoá-phong-bì)
7. [Kiểm Tra và Xác Minh Encryption](#kiểm-tra-và-xác-minh-encryption)
8. [Quản Lý Key Rotation](#quản-lý-key-rotation)
9. [Kết Hợp Với Sealed Secrets và External Secrets](#kết-hợp-với-sealed-secrets-và-external-secrets)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Vấn Đề Bảo Mật Secret Mặc Định

Secret trong Kubernetes mặc định **không được mã hoá** — chỉ được base64 encode:

```bash
# Secret YAML:
apiVersion: v1
kind: Secret
metadata:
  name: db-password
data:
  password: c3VwZXJTZWNyZXRAMTIz    # base64("superSecret@123")

# Ai có thể đọc Secret này?
# 1. Bất kỳ user nào có quyền: kubectl get secret db-password -o yaml
# 2. Admin có quyền truy cập etcd backup
# 3. Attacker đánh cắp etcd snapshot
```

**Xác minh Secret lưu plaintext trong etcd:**

```bash
# Truy cập etcd trực tiếp để đọc Secret (trên control plane node)
ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  /registry/secrets/production/db-password

# Kết quả: thấy plaintext JSON — Secret không được mã hoá
# {"kind":"Secret","apiVersion":"v1","data":{"password":"c3VwZXJTZWNyZXRAMTIz"},...}
```

**Các vector tấn công:**

```
Attacker truy cập etcd data:
├── Đánh cắp etcd backup file (thường lưu trên S3/GCS)
├── Truy cập etcd pod trực tiếp nếu không có auth mạnh
├── Truy cập snapshot từ volume backup
└── Man-in-the-middle giữa control plane components (nếu không dùng TLS)
```

---

## Encryption at Rest — Mã Hoá Tại Etcd

**Encryption at Rest** mã hoá data trước khi ghi vào etcd — ngay cả admin đọc etcd trực tiếp cũng không thấy plaintext.

### Luồng Xử Lý

```
Không có Encryption at Rest:
  kubectl create secret  →  API Server  →  etcd: PLAINTEXT

Có Encryption at Rest:
  kubectl create secret  →  API Server  →  Mã hoá với key  →  etcd: CIPHERTEXT
  kubectl get secret     →  API Server  ←  Giải mã với key  ←  etcd: CIPHERTEXT
```

### Providers Hỗ Trợ

| Provider | Thuật Toán | Hiệu Năng | Bảo Mật | Ghi Chú |
| -------- | ---------- | --------- | ------- | ------- |
| `identity` | Không mã hoá | Cao | Thấp | Fallback, đọc được data cũ |
| `aescbc` | AES-CBC 256-bit | Cao | Trung bình | Key lưu trên disk |
| `aesgcm` | AES-GCM 256-bit | Rất cao | Tốt | Cần rotate key thường xuyên |
| `secretbox` | XSalsa20 + Poly1305 | Cao | Tốt | Ít phổ biến hơn |
| `kms` | AES-GCM + external key | Trung bình | **Cao nhất** | Key lưu trong KMS bên ngoài |
| `kms` v2 | AES-GCM 256-bit + DEK cache | Cao | **Cao nhất** | K8s 1.27+, khuyến nghị |

---

## Cấu Hình EncryptionConfiguration

### Bước 1: Tạo File Cấu Hình

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  # Mã hoá Secret (quan trọng nhất)
  - resources:
      - secrets
    providers:
      # Provider đầu tiên = provider dùng để VIẾT mới
      - aesgcm:
          keys:
            - name: key1
              secret: dGhpcy1pcy1hLTMyLWJ5dGUtc2VjcmV0LWtleS0h  # base64(32-byte key)
      # identity ở cuối: cho phép đọc Secret cũ chưa mã hoá
      - identity: {}

  # Tuỳ chọn: mã hoá cả ConfigMap
  - resources:
      - configmaps
    providers:
      - aesgcm:
          keys:
            - name: key1
              secret: dGhpcy1pcy1hLTMyLWJ5dGUtc2VjcmV0LWtleS0h
      - identity: {}
```

**Tạo 32-byte key:**

```bash
# Tạo key ngẫu nhiên 32 bytes, encode base64
head -c 32 /dev/urandom | base64
# Ví dụ output: dGhpcy1pcy1hLTMyLWJ5dGUtc2VjcmV0LWtleS0h
```

### Bước 2: Cấu Hình kube-apiserver

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
    - name: kube-apiserver
      image: registry.k8s.io/kube-apiserver:v1.28.0
      command:
        - kube-apiserver
        - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
        # Các flag khác...

      # Mount file cấu hình vào container
      volumeMounts:
        - name: encryption-config
          mountPath: /etc/kubernetes/encryption-config.yaml
          readOnly: true

  volumes:
    - name: encryption-config
      hostPath:
        path: /etc/kubernetes/encryption-config.yaml
        type: File
```

### Bước 3: Restart API Server

```bash
# Với static Pod (kubeadm), chỉ cần sửa file manifest — kubelet tự restart
# Kiểm tra API Server đang chạy
kubectl get pods -n kube-system kube-apiserver-master

# Xem log để đảm bảo không có lỗi
kubectl logs -n kube-system kube-apiserver-master | grep encryption
```

### Bước 4: Re-encrypt Secret Cũ

```bash
# Secret cũ vẫn lưu plaintext — cần trigger re-write để mã hoá
kubectl get secrets --all-namespaces -o json | kubectl replace -f -

# Hoặc theo namespace
kubectl get secrets -n production -o json | kubectl replace -f -

# Xác nhận mã hoá thành công (đọc từ etcd trực tiếp)
ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  /registry/secrets/production/db-password | hexdump -C | head -5

# Kết quả mong đợi: thấy "k8s:enc:aesgcm:v1:" ở đầu — đã mã hoá
```

---

## KMS Plugin — Mã Hoá Với Key Bên Ngoài

**KMS Plugin** là lựa chọn bảo mật cao nhất: key mã hoá không lưu trên control plane — được quản lý bởi dịch vụ KMS bên ngoài (AWS KMS, GCP KMS, HashiCorp Vault).

### Envelope Encryption với KMS

```
Lần đầu ghi Secret:
1. API Server tạo DEK (Data Encryption Key — Khoá Mã Hoá Dữ Liệu) ngẫu nhiên
2. Mã hoá Secret bằng DEK
3. Gửi DEK đến KMS — KMS mã hoá DEK bằng KEK (Key Encryption Key)
4. Lưu vào etcd: { encrypted_secret, encrypted_DEK }

Lần sau đọc Secret:
1. Lấy encrypted_secret + encrypted_DEK từ etcd
2. Gửi encrypted_DEK đến KMS — KMS giải mã DEK
3. Dùng DEK giải mã Secret
4. Trả về plaintext cho caller
```

### AWS KMS Plugin

```bash
# Tạo KMS key trong AWS
aws kms create-key \
  --description "Kubernetes etcd encryption key" \
  --key-usage ENCRYPT_DECRYPT

# Tạo alias dễ nhớ
aws kms create-alias \
  --alias-name alias/k8s-etcd-encryption \
  --target-key-id <key-id>

# Ghi lại ARN: arn:aws:kms:us-east-1:123456789012:key/abc123-...
```

```yaml
# encryption-config.yaml với KMS v2 (K8s 1.27+)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - kms:
          apiVersion: v2
          name: aws-kms-plugin
          endpoint: unix:///var/run/kmsplugin/socket.sock
          timeout: 3s
          cachesize: 1000          # cache DEK để giảm call KMS
      - identity: {}
```

```yaml
# DaemonSet chạy AWS KMS Plugin
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: aws-encryption-provider
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: aws-encryption-provider
  template:
    spec:
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
      containers:
        - name: aws-encryption-provider
          image: amazon/aws-encryption-provider:latest
          args:
            - --key=arn:aws:kms:us-east-1:123456789012:key/abc123
            - --region=us-east-1
            - --listen=/var/run/kmsplugin/socket.sock
          volumeMounts:
            - name: plugin-socket-dir
              mountPath: /var/run/kmsplugin
      volumes:
        - name: plugin-socket-dir
          hostPath:
            path: /var/run/kmsplugin
            type: DirectoryOrCreate
```

### HashiCorp Vault KMS Plugin

```hcl
# vault.hcl — cấu hình Vault Transit engine (Công Cụ Quá Cảnh)
path "transit/encrypt/k8s-etcd" {
  capabilities = ["create", "update"]
}

path "transit/decrypt/k8s-etcd" {
  capabilities = ["create", "update"]
}

path "transit/keys/k8s-etcd" {
  capabilities = ["read"]
}
```

```yaml
# encryption-config.yaml với Vault KMS Plugin
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - kms:
          apiVersion: v2
          name: vault-kms-plugin
          endpoint: unix:///opt/vault-kms-plugin/socket.sock
          timeout: 5s
      - identity: {}
```

---

## Managed Cluster — Encryption Tự Động

### Amazon EKS

```bash
# Bật Secret encryption khi tạo cluster mới
aws eks create-cluster \
  --name my-cluster \
  --kubernetes-version 1.28 \
  --role-arn arn:aws:iam::123456789012:role/eks-cluster-role \
  --resources-vpc-config subnetIds=...,securityGroupIds=... \
  --encryption-config '[{
    "resources": ["secrets"],
    "provider": {
      "keyArn": "arn:aws:kms:us-east-1:123456789012:key/abc123"
    }
  }]'

# Bật cho cluster đã có (không thể tắt sau khi bật)
aws eks associate-encryption-config \
  --cluster-name my-cluster \
  --encryption-config '[{
    "resources": ["secrets"],
    "provider": {
      "keyArn": "arn:aws:kms:us-east-1:123456789012:key/abc123"
    }
  }]'

# Kiểm tra trạng thái
aws eks describe-cluster --name my-cluster \
  --query "cluster.encryptionConfig"
```

### Google GKE

```bash
# Bật Application-layer Secrets Encryption với Cloud KMS
gcloud kms keyrings create k8s-keyring \
  --location us-central1

gcloud kms keys create k8s-etcd-key \
  --keyring k8s-keyring \
  --location us-central1 \
  --purpose encryption

# Tạo cluster với database encryption
gcloud container clusters create my-cluster \
  --region us-central1 \
  --database-encryption-key \
    projects/my-project/locations/us-central1/keyRings/k8s-keyring/cryptoKeys/k8s-etcd-key

# Bật cho cluster hiện có
gcloud container clusters update my-cluster \
  --region us-central1 \
  --database-encryption-key \
    projects/my-project/locations/us-central1/keyRings/k8s-keyring/cryptoKeys/k8s-etcd-key
```

### Azure AKS

```bash
# Azure AKS mã hoá Secret at rest theo mặc định
# Dùng Customer Managed Key (CMK) cho kiểm soát tốt hơn:

# Tạo Key Vault và key
az keyvault create --name my-kv --resource-group my-rg
az keyvault key create --vault-name my-kv --name k8s-etcd-key --protection software

# Tạo disk encryption set
az disk-encryption-set create \
  --name my-des \
  --resource-group my-rg \
  --source-vault my-kv \
  --key-url <key-url>

# Tạo cluster với CMK
az aks create \
  --name my-cluster \
  --resource-group my-rg \
  --node-disk-encryption-set-id <des-id>
```

---

## Envelope Encryption — Mã Hoá Phong Bì

**Envelope Encryption** (Mã Hoá Phong Bì) là pattern chuẩn cho encryption at scale:

```
DEK (Data Encryption Key — Khoá Mã Hoá Dữ Liệu):
  - Tạo ngẫu nhiên cho mỗi Secret
  - Dùng AES-256-GCM mã hoá Secret data
  - Bản thân DEK được mã hoá bằng KEK

KEK (Key Encryption Key — Khoá Mã Khoá):
  - Lưu trong KMS (AWS KMS, GCP KMS, Vault)
  - Không bao giờ rời khỏi KMS
  - KMS chỉ expose encrypt/decrypt operation
  - Có thể rotate mà không cần re-encrypt toàn bộ data
```

**Lợi ích của pattern này:**
- KEK rotate: chỉ cần re-encrypt DEK (nhỏ) — không cần re-encrypt toàn bộ Secret
- Hiệu năng: KMS call ít hơn (cache DEK)
- Audit: mọi decrypt operation đều có audit trail trong KMS

---

## Kiểm Tra và Xác Minh Encryption

```bash
# Tạo Secret test
kubectl create secret generic encryption-test \
  --from-literal=key=value-to-encrypt \
  -n default

# Đọc trực tiếp từ etcd
ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  /registry/secrets/default/encryption-test

# Nếu đã mã hoá đúng, output bắt đầu bằng:
# k8s:enc:aesgcm:v1:key1:... (binary data)
# Nếu chưa mã hoá: thấy JSON plaintext

# Script kiểm tra tất cả Secret
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  for secret in $(kubectl get secrets -n $ns -o jsonpath='{.items[*].metadata.name}'); do
    result=$(ETCDCTL_API=3 etcdctl get \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/server.crt \
      --key=/etc/kubernetes/pki/etcd/server.key \
      /registry/secrets/$ns/$secret | head -c 20)
    if [[ ! "$result" == *"k8s:enc"* ]]; then
      echo "KHÔNG MÃ HOÁ: $ns/$secret"
    fi
  done
done
```

---

## Quản Lý Key Rotation

### Rotate Encryption Key (Local Key)

```yaml
# Bước 1: Thêm key mới vào ĐẦULIST, giữ key cũ ở sau
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aesgcm:
          keys:
            - name: key2                  # key MỚI — dùng để write
              secret: <new-32-byte-key-base64>
            - name: key1                  # key CŨ — giữ để đọc secret cũ
              secret: <old-32-byte-key-base64>
      - identity: {}
```

```bash
# Bước 2: Apply config mới, restart API Server

# Bước 3: Re-encrypt mọi Secret với key mới
kubectl get secrets --all-namespaces -o json | kubectl replace -f -

# Bước 4: Sau khi verify xong, xoá key cũ khỏi config
```

### Key Rotation Với KMS

```bash
# AWS KMS: tạo key version mới (automatic rotation)
aws kms enable-key-rotation \
  --key-id arn:aws:kms:us-east-1:123456789012:key/abc123

# Hoặc rotate thủ công
aws kms rotate-key-on-demand \
  --key-id arn:aws:kms:us-east-1:123456789012:key/abc123

# Với KMS, không cần re-encrypt Secret — KMS tự quản lý key version
```

---

## Kết Hợp Với Sealed Secrets và External Secrets

**Encryption at Rest là một lớp bảo mật, không phải giải pháp toàn diện.** Chiến lược bảo mật Secret đầy đủ:

```
Vấn đề                          Giải Pháp
────────────────────────────────────────────────────────────
Secret lưu plaintext trong etcd → Encryption at Rest với KMS
Secret hardcode trong Git       → Sealed Secrets (mã hoá trong Git)
Secret phân tán khó quản lý    → External Secrets từ Vault/AWS SM
Secret lộ qua kubectl get       → RBAC restrict quyền get secret
Auto-rotation secret            → External Secrets + Vault/AWS SM
Audit trail chi tiết            → KMS audit log + K8s Audit Log
```

```yaml
# Chiến lược hoàn chỉnh cho production:

# 1. Encryption at Rest: Secret trong etcd được mã hoá bằng AWS KMS
# 2. External Secrets: Secret từ AWS Secrets Manager tự động đồng bộ vào cluster
# 3. RBAC: chỉ SA cần thiết mới có quyền get/list secret
# 4. Audit: mọi get/list secret được log vào K8s Audit Log
# 5. Monitoring: alert khi có secret access bất thường

# Với GitOps workflow, thêm:
# 6. Sealed Secrets hoặc không commit secret vào Git
#    (dùng External Secrets thì không cần commit secret)
```

---

## Câu Hỏi Phỏng Vấn

**Secret Kubernetes có thực sự được mã hoá không?**

> Mặc định: **không** — Secret chỉ được base64 encode (không phải mã hoá). Để thực sự mã hoá, cần bật **Encryption at Rest** bằng `EncryptionConfiguration` trong kube-apiserver. Khi bật, Secret được mã hoá bằng AES-256 trước khi ghi vào etcd. Tốt nhất là dùng **KMS provider** (AWS KMS, GCP KMS) để key không bao giờ lưu trên control plane. Bổ sung thêm: RBAC restrict quyền get secret, Kubernetes Audit Log track truy cập.

**Encryption at Rest bảo vệ điều gì? Không bảo vệ điều gì?**

> **Bảo vệ:** etcd data bị đánh cắp trực tiếp (backup file, snapshot), admin truy cập etcd bằng etcdctl, file system của control plane bị truy cập. **Không bảo vệ:** Secret vẫn decrypt khi trả về qua API Server — ai có quyền `kubectl get secret` vẫn đọc được; traffic giữa API Server và client (cần TLS — mặc định đã có); application memory sau khi mount secret vào container.

**Envelope Encryption là gì? Tại sao tốt hơn mã hoá trực tiếp bằng master key?**

> Envelope Encryption dùng hai tầng key: DEK (Data Encryption Key) ngẫu nhiên cho từng Secret, và KEK (Key Encryption Key) trong KMS để mã hoá DEK. Lợi ích so với mã hoá trực tiếp bằng master key: (1) **Key rotation hiệu quả** — chỉ cần re-encrypt DEK nhỏ, không cần re-encrypt toàn bộ data; (2) **Hiệu năng** — DEK cache locally, không phải gọi KMS mỗi read/write; (3) **Bảo mật** — KEK không bao giờ rời khỏi KMS, chỉ expose encrypt/decrypt API; (4) **Audit granularity** — biết ai decrypt DEK nào lúc nào.

**Làm thế nào để migrate cluster từ không mã hoá sang có mã hoá mà không downtime?**

> Quy trình zero-downtime: (1) Tạo `EncryptionConfiguration` với provider mới đứng đầu và `identity: {}` ở cuối; (2) Update kube-apiserver manifest — apiserver restart nhanh (Static Pod tự restart). Trong thời gian này apiserver tạm không serve nhưng thường dưới 30 giây; (3) Sau khi apiserver lên, chạy `kubectl get secrets --all-namespaces -o json | kubectl replace -f -` để re-encrypt Secret cũ; (4) Verify bằng etcdctl get và kiểm tra prefix `k8s:enc`; (5) Khi tất cả Secret đã mã hoá, có thể xoá `identity: {}` khỏi config để từ chối đọc plaintext (thận trọng — đảm bảo mọi Secret đã re-encrypt).
