# Secret — Quản Lý Dữ Liệu Nhạy Cảm Kubernetes

> Hướng dẫn chi tiết về Secret: tạo, các loại Secret, cách mount vào Pod, bảo mật với Encryption at Rest (Mã Hoá Tại Nơi Lưu Trữ), và best practice quản lý dữ liệu nhạy cảm.

## Mục Lục

1. [Secret Là Gì?](#secret-là-gì)
2. [Các Loại Secret](#các-loại-secret)
3. [Tạo Secret](#tạo-secret)
4. [Mount Secret Vào Pod](#mount-secret-vào-pod)
5. [Encryption at Rest — Mã Hoá Tại Nơi Lưu Trữ](#encryption-at-rest--mã-hoá-tại-nơi-lưu-trữ)
6. [Secret Trong Docker Registry](#secret-trong-docker-registry)
7. [Best Practice](#best-practice)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Secret Là Gì?

**Secret** là tài nguyên Kubernetes dùng để lưu trữ dữ liệu nhạy cảm như password, API key, token OAuth, certificate TLS, SSH key. Giống ConfigMap về cấu trúc, nhưng Kubernetes xử lý Secret với một số biện pháp bảo vệ bổ sung:

| Bảo Vệ | ConfigMap | Secret |
| ------- | --------- | ------ |
| base64 encode | Không | ✅ Có |
| Không in ra khi `kubectl describe` | Không | ✅ Có (hiện `REDACTED`) |
| Lưu trong tmpfs (RAM) trên node | Không | ✅ Có (tránh ghi ra disk) |
| Encryption at Rest (etcd) | Không | ✅ Có thể bật |
| RBAC nên restrict chặt hơn | Không cần | ✅ Khuyến nghị |

> **Hiểu đúng về base64:** base64 là **encoding** (mã hoá định dạng), không phải **encryption** (mã hoá bảo mật). Bất kỳ ai có quyền `kubectl get secret -o yaml` đều đọc được giá trị gốc bằng `echo "..." | base64 -d`. Secret mặc định **không an toàn** nếu không cấu hình thêm.

---

## Các Loại Secret

### `Opaque` — Tuỳ Ý (Mặc Định)

Loại Secret phổ biến nhất — lưu dữ liệu tuỳ ý dạng key-value:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
data:
  # Giá trị phải được base64 encode
  username: cG9zdGdyZXM=         # base64("postgres")
  password: c3VwZXJTZWNyZXRAMTIz # base64("superSecret@123")
  api-key: c2stYWJjMTIz          # base64("sk-abc123")
```

Tạo giá trị base64:
```bash
echo -n "postgres" | base64        # cG9zdGdyZXM=
echo -n "superSecret@123" | base64 # c3VwZXJTZWNyZXRAMTIz

# Decode để kiểm tra
echo "cG9zdGdyZXM=" | base64 -d   # postgres
```

Hoặc dùng `stringData` (Kubernetes tự encode):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  # Viết plaintext — Kubernetes tự convert sang base64 khi lưu
  username: postgres
  password: superSecret@123
  api-key: sk-abc123
```

### `kubernetes.io/tls` — TLS Certificate

Lưu certificate và private key cho TLS termination:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-myapp
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: |
    LS0tLS1CRUdJTi... # base64(certificate PEM)
  tls.key: |
    LS0tLS1CRUdJTi... # base64(private key PEM)
```

Tạo từ file cert:
```bash
kubectl create secret tls tls-myapp \
  --cert=tls.crt \
  --key=tls.key \
  -n production
```

Dùng trong Ingress:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
spec:
  tls:
    - hosts:
        - myapp.example.com
      secretName: tls-myapp     # tham chiếu Secret TLS
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

### `kubernetes.io/dockerconfigjson` — Registry Credentials

Thông tin xác thực kéo image từ private registry (kho lưu trữ riêng tư):

```bash
# Tạo secret để pull image từ AWS ECR (Elastic Container Registry)
kubectl create secret docker-registry ecr-secret \
  --docker-server=123456789.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region us-east-1) \
  -n production

# Tạo từ file .docker/config.json
kubectl create secret generic registry-creds \
  --from-file=.dockerconfigjson=/root/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson
```

Dùng trong Pod:
```yaml
spec:
  imagePullSecrets:
    - name: ecr-secret    # Pod dùng secret này để pull image
  containers:
    - name: app
      image: 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
```

### `kubernetes.io/service-account-token` — Token ServiceAccount

Token được tạo tự động cho ServiceAccount (từ K8s 1.24+ không tự tạo — dùng projected volume):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production

---
# Token được mount tự động vào Pod dùng ServiceAccount này
# Tại: /var/run/secrets/kubernetes.io/serviceaccount/token
```

### `kubernetes.io/ssh-auth` — SSH Key

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-ssh-key
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: |
    LS0tLS1CRUdJTi... # base64(private SSH key)
```

### `kubernetes.io/basic-auth` — Thông Tin Xác Thực Cơ Bản

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: mypassword
```

---

## Tạo Secret

### Từ Dòng Lệnh

```bash
# Từ literal value
kubectl create secret generic db-credentials \
  --from-literal=username=postgres \
  --from-literal=password='superSecret@123' \
  -n production

# Từ file (tên file là key)
kubectl create secret generic app-certs \
  --from-file=ca.crt \
  --from-file=server.crt \
  --from-file=server.key \
  -n production

# Từ file với key tuỳ chỉnh
kubectl create secret generic app-certs \
  --from-file=certificate=server.crt \
  --from-file=private-key=server.key

# Từ file .env (định dạng KEY=VALUE)
kubectl create secret generic app-secrets \
  --from-env-file=.env.production
```

### Xem Secret

```bash
# Liệt kê (không hiện giá trị)
kubectl get secret -n production

# Xem YAML (data là base64 — không decode)
kubectl get secret db-credentials -o yaml

# Decode giá trị một key
kubectl get secret db-credentials \
  -o jsonpath='{.data.password}' | base64 -d

# Xem tất cả key và decode (script)
kubectl get secret db-credentials -o json | \
  jq -r '.data | to_entries[] | "\(.key): \(.value | @base64d)"'
```

---

## Mount Secret Vào Pod

Cách mount Secret vào Pod giống ConfigMap — hỗ trợ cả biến môi trường và volume:

### Phương Pháp 1: envFrom — Toàn Bộ Secret Thành Env Var

```yaml
spec:
  containers:
    - name: app
      image: myapp:v1
      envFrom:
        - secretRef:
            name: db-credentials     # tất cả key → biến môi trường
```

Container thấy: `username=postgres`, `password=superSecret@123`

### Phương Pháp 2: env[].valueFrom — Từng Key Cụ Thể

```yaml
spec:
  containers:
    - name: app
      image: myapp:v1
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        - name: EXTERNAL_API_KEY
          valueFrom:
            secretKeyRef:
              name: api-keys
              key: stripe-api-key
              optional: true    # không lỗi nếu key không tồn tại
```

### Phương Pháp 3: Volume Mount — Mount Thành File

Phù hợp cho certificate, SSH key, file cấu hình chứa secret:

```yaml
spec:
  volumes:
    - name: db-creds-vol
      secret:
        secretName: db-credentials
        defaultMode: 0400          # chỉ owner đọc được (an toàn hơn 0644)
        items:                     # tuỳ chọn: chỉ mount một số key
          - key: password
            path: db-password      # tên file trong container

  containers:
    - name: app
      image: myapp:v1
      volumeMounts:
        - name: db-creds-vol
          mountPath: /etc/secrets
          readOnly: true
```

Container thấy file `/etc/secrets/db-password` chứa plaintext password.

### Ví Dụ Thực Tế: TLS Certificate Cho Application

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  template:
    spec:
      volumes:
        - name: tls-certs
          secret:
            secretName: tls-myapp
            defaultMode: 0400

      containers:
        - name: app
          image: myapp:v1
          ports:
            - containerPort: 8443
          volumeMounts:
            - name: tls-certs
              mountPath: /etc/ssl/certs
              readOnly: true
          env:
            - name: TLS_CERT_PATH
              value: /etc/ssl/certs/tls.crt
            - name: TLS_KEY_PATH
              value: /etc/ssl/certs/tls.key
```

---

## Encryption at Rest — Mã Hoá Tại Nơi Lưu Trữ

Mặc định, Secret lưu trong etcd dưới dạng **base64 plaintext**. Bất kỳ ai có quyền truy cập etcd (admin cluster, backup file) đều đọc được. Để bảo mật thật sự, cần bật Encryption at Rest.

### Cách Cấu Hình EncryptionConfiguration

Tạo file cấu hình trên control plane node:

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets              # mã hoá Secret
      - configmaps           # tuỳ chọn: mã hoá cả ConfigMap
    providers:
      # Provider 1: AES-GCM (khuyến nghị — nhanh và an toàn)
      - aescbc:
          keys:
            - name: key1
              secret: c2VjcmV0LWtleS0zMi1ieXRlcy1iYXNlNjQ=  # base64(32-byte key)

      # Provider 2: KMS (Key Management Service) — bảo mật nhất
      # - kms:
      #     name: aws-kms
      #     endpoint: unix:///tmp/socketfile.sock
      #     cachesize: 1000
      #     timeout: 3s

      # Cuối cùng luôn có identity (đọc được secret chưa mã hoá)
      - identity: {}
```

Kích hoạt trong kube-apiserver:
```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
    - command:
        - kube-apiserver
        - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

### Xác Nhận Mã Hoá Đang Hoạt Động

```bash
# Tạo Secret mới và kiểm tra trong etcd
kubectl create secret generic test-encrypt \
  --from-literal=key=value

# Đọc trực tiếp từ etcd (cần truy cập etcd)
ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  /registry/secrets/default/test-encrypt

# Nếu thấy "k8s:enc:aescbc:v1:" ở đầu → đã mã hoá thành công
# Nếu thấy plaintext → chưa mã hoá
```

### Mã Hoá Lại Secret Cũ

```bash
# Sau khi bật Encryption at Rest, cần re-encrypt Secret cũ
# (Secret cũ vẫn lưu plaintext trong etcd)
kubectl get secrets --all-namespaces -o json | \
  kubectl replace -f -
```

### Managed Cluster — Encryption Tự Động

```
AWS EKS:   Secrets mã hoá bằng AWS KMS — bật trong cluster config
           aws eks create-cluster --secrets-encryption-key-arn <kms-key-arn>

GKE:       Application-layer secrets encryption với Cloud KMS
           gcloud container clusters create --database-encryption-key <key>

AKS:       Encryption at Rest mặc định với Azure Managed Key
           Có thể dùng Customer Managed Key (CMK) từ Azure Key Vault
```

---

## Secret Trong Docker Registry

Khi dùng private registry, Pod cần secret để pull image. Có hai cách cung cấp:

### Cách 1: Gắn imagePullSecrets Vào Pod/Deployment

```yaml
spec:
  imagePullSecrets:
    - name: registry-secret
  containers:
    - name: app
      image: private-registry.example.com/myapp:v1
```

### Cách 2: Gắn Vào ServiceAccount (Khuyến Nghị — Áp Dụng Cho Mọi Pod Dùng SA)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
imagePullSecrets:
  - name: registry-secret   # mọi Pod dùng SA này đều tự động có pull secret
```

### Tự Động Refresh ECR Token

ECR token hết hạn sau 12 giờ. Dùng CronJob để refresh:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ecr-token-refresher
spec:
  schedule: "0 */6 * * *"   # mỗi 6 giờ
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: ecr-token-refresher-sa  # SA có quyền ecr:GetAuthorizationToken
          containers:
            - name: refresher
              image: amazon/aws-cli:latest
              command:
                - /bin/sh
                - -c
                - |
                  TOKEN=$(aws ecr get-login-password --region us-east-1)
                  kubectl create secret docker-registry ecr-secret \
                    --docker-server=$ECR_REGISTRY \
                    --docker-username=AWS \
                    --docker-password=$TOKEN \
                    --dry-run=client -o yaml | kubectl apply -f -
```

---

## Best Practice

### Quản Lý RBAC Cho Secret

```yaml
# Chỉ cho phép ServiceAccount cụ thể đọc Secret
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["db-credentials", "api-keys"]  # chỉ secret cụ thể
    verbs: ["get"]   # chỉ get, không list/watch/create/delete

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-secret-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: myapp-sa
    namespace: production
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### Rotation — Xoay Vòng Secret

```bash
# 1. Tạo Secret mới với credentials mới
kubectl create secret generic db-credentials-v2 \
  --from-literal=username=postgres \
  --from-literal=password='newStrongPassword@456'

# 2. Cập nhật Deployment dùng Secret mới
kubectl patch deployment myapp \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","envFrom":[{"secretRef":{"name":"db-credentials-v2"}}]}]}}}}'

# 3. Xác nhận rollout hoàn tất
kubectl rollout status deployment/myapp

# 4. Xoá Secret cũ
kubectl delete secret db-credentials
```

### Checklist Bảo Mật

- [ ] **Không lưu Secret trong Git plaintext** — dùng Sealed Secrets hoặc External Secrets
- [ ] **Bật Encryption at Rest** cho etcd trong production cluster
- [ ] **RBAC chặt:** chỉ pod cần thiết mới có quyền `get` secret — không dùng `list/watch` trừ khi cần
- [ ] **Dùng `stringData`** khi tạo YAML để dễ đọc, Kubernetes tự encode
- [ ] **Đặt `readOnly: true`** khi mount Secret vào volume
- [ ] **Dùng `defaultMode: 0400`** cho file Secret (chỉ owner đọc được)
- [ ] **Rotate định kỳ** — tối thiểu 90 ngày, tốt hơn là dùng auto-rotation qua Vault/AWS
- [ ] **Không dùng Secret làm biến môi trường** cho dữ liệu cực kỳ nhạy cảm — volume mount an toàn hơn (không lộ trong `ps` hoặc crash dump)
- [ ] **Monitor truy cập Secret** qua Kubernetes Audit Log (Nhật Ký Kiểm Tra)

---

## Câu Hỏi Phỏng Vấn

**Secret có thực sự bảo mật không? Kubernetes làm gì để bảo vệ Secret?**

> Secret **không an toàn tự động** — cần cấu hình thêm để bảo vệ thật sự. Kubernetes cung cấp một số lớp bảo vệ: (1) base64 encoding (không phải encryption); (2) lưu trong tmpfs (RAM) trên node thay vì disk; (3) Secret chỉ gửi đến node cần thiết (node chạy Pod dùng Secret đó); (4) hỗ trợ **Encryption at Rest** với etcd nếu cấu hình EncryptionConfiguration; (5) RBAC phân quyền truy cập Secret. Để bảo mật production, cần: bật Encryption at Rest + RBAC chặt + Audit log + không commit Secret lên Git.

**Tại sao không nên truyền Secret qua biến môi trường (env var)?**

> Biến môi trường có rủi ro bảo mật cao hơn file volume: (1) Có thể bị lộ qua lệnh `ps aux` (hiển thị process list) trên node; (2) Thường bị in ra trong crash dump, error log, stack trace; (3) Các thư viện bên thứ ba có thể vô tình log biến môi trường; (4) Không thể rotate mà không restart Pod; (5) Truyền vào child process mặc định. Mount Secret qua volume tạo file với permission hạn chế (0400), chỉ process cần thiết mới đọc được, và có thể rotate mà không restart nếu ứng dụng hỗ trợ hot-reload.

**Khi Secret thay đổi, container có nhận được giá trị mới không?**

> Giống ConfigMap: nếu mount qua **volume** thì file tự cập nhật sau 1–2 phút, nhưng ứng dụng phải tự reload. Nếu mount qua **biến môi trường** thì **không bao giờ tự cập nhật** — phải restart Pod. Đây là lý do auto-rotation của Vault/External Secrets thường kết hợp với cơ chế restart Pod hoặc Reloader (sidecar watch thay đổi Secret và trigger restart tự động).

**IRSA (IAM Roles for Service Accounts) là gì? Tại sao tốt hơn lưu AWS credentials trong Secret?**

> IRSA (IAM Roles for Service Accounts — Vai Trò IAM Cho ServiceAccount) là cơ chế cho phép Pod trên EKS assume IAM role mà **không cần lưu access key/secret key**. Pod được gắn ServiceAccount, ServiceAccount được annotate với IAM role ARN, EKS tự động issue temporary credential qua OIDC (OpenID Connect). Tốt hơn lưu credentials trong Secret vì: (1) Không có static credential để lộ; (2) Credential tự rotate (expire sau 1 giờ); (3) Phạm vi quyền giới hạn theo IAM policy; (4) Audit log đầy đủ qua CloudTrail; (5) Không cần rotation thủ công.
