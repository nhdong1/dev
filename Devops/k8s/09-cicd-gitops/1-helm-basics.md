# Helm Basics — Quản Lý Package Kubernetes Cơ Bản

> Helm là trình quản lý package (package manager) cho Kubernetes, giúp đóng gói, cấu hình và triển khai ứng dụng theo cách có thể tái sử dụng, versioned (quản lý phiên bản) và dễ chia sẻ.

## Mục Lục

1. [Helm Là Gì và Tại Sao Cần Helm](#helm-là-gì-và-tại-sao-cần-helm)
2. [Các Khái Niệm Cốt Lõi](#các-khái-niệm-cốt-lõi)
3. [Cấu Trúc Helm Chart](#cấu-trúc-helm-chart)
4. [Helm Repository](#helm-repository)
5. [Các Lệnh Helm Thiết Yếu](#các-lệnh-helm-thiết-yếu)
6. [Quản Lý Values](#quản-lý-values)
7. [Vòng Đời Release](#vòng-đời-release)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)
9. [Cheat Sheet Lệnh Helm](#cheat-sheet-lệnh-helm)

---

## Helm Là Gì và Tại Sao Cần Helm

### Vấn Đề Khi Không Có Helm

Triển khai một ứng dụng microservice điển hình lên Kubernetes cần nhiều manifest YAML: Deployment, Service, Ingress, ConfigMap, Secret, HPA, ServiceAccount... Khi có nhiều môi trường (dev, staging, prod) với cấu hình khác nhau, việc maintain (duy trì) bộ YAML trở nên phức tạp:

```
Không có Helm:
app/
├── deployment-dev.yaml      ← copy gần giống deployment-prod.yaml
├── deployment-staging.yaml  ← copy với một vài giá trị khác
├── deployment-prod.yaml
├── service-dev.yaml
├── service-prod.yaml
└── ingress-dev.yaml
    ingress-prod.yaml

Vấn đề:
- DRY violation (Don't Repeat Yourself — Không lặp lại): 90% nội dung giống nhau
- Khi cần sửa một field (ví dụ: thêm label), phải sửa tất cả file
- Không có versioning — không biết prod đang chạy phiên bản nào
- Rollback thủ công, dễ lỗi
```

### Helm Giải Quyết Như Thế Nào

```
Với Helm:
my-app/                      ← một chart duy nhất
├── templates/               ← template với placeholder
│   ├── deployment.yaml      ← {{ .Values.image.tag }}
│   └── service.yaml
├── values.yaml              ← giá trị mặc định
├── values-dev.yaml          ← chỉ override giá trị khác
└── values-prod.yaml

Sử dụng:
helm install my-app ./my-app -f values-prod.yaml
helm upgrade my-app ./my-app --set image.tag=v1.2.3
helm rollback my-app 2         ← rollback về revision 2
helm history my-app            ← xem lịch sử tất cả revision
```

---

## Các Khái Niệm Cốt Lõi

### Chart (Biểu Đồ / Gói)

**Chart** là gói Helm — tập hợp file template, values mặc định và metadata mô tả một ứng dụng Kubernetes. Chart tương tự như `.deb` package trong Debian hay `npm` package trong Node.js.

```
Hai loại chart:
├── Application Chart — cài đặt ứng dụng cụ thể (nginx, postgresql, my-app)
└── Library Chart    — cung cấp helper template tái sử dụng, không install trực tiếp
```

### Release (Phiên Bản Đã Cài)

**Release** là một instance (thực thể) của chart đã được cài đặt vào cluster. Một chart có thể được cài đặt nhiều lần thành nhiều release khác nhau với tên và namespace khác nhau.

```
# Cùng một chart, hai release khác nhau
helm install wordpress-dev  bitnami/wordpress -n dev
helm install wordpress-prod bitnami/wordpress -n prod

# Mỗi release là độc lập, có revision history riêng
```

### Revision (Lần Sửa Đổi)

**Revision** là số thứ tự của mỗi lần thay đổi release. Mỗi lần `helm upgrade` tạo ra một revision mới. Helm lưu history để có thể rollback.

```
helm history my-app
REVISION  STATUS     CHART         DESCRIPTION
1         superseded my-app-1.0.0  Install complete
2         superseded my-app-1.0.1  Upgrade complete
3         deployed   my-app-1.1.0  Upgrade complete ← current
```

### Values (Giá Trị Cấu Hình)

**Values** là bộ tham số có thể tuỳ chỉnh khi install hoặc upgrade chart. Được lưu trong `values.yaml` (mặc định) và có thể override bằng `-f values-custom.yaml` hoặc `--set key=value`.

---

## Cấu Trúc Helm Chart

```
my-app/
├── Chart.yaml              ← bắt buộc: metadata của chart
├── values.yaml             ← bắt buộc: giá trị mặc định
├── charts/                 ← tùy chọn: sub-chart (phụ thuộc)
│   └── postgresql/         ← chart phụ thuộc (dependency)
├── templates/              ← bắt buộc: Kubernetes manifest template
│   ├── NOTES.txt           ← hướng dẫn hiển thị sau install
│   ├── _helpers.tpl        ← helper template (không tạo manifest)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   └── serviceaccount.yaml
└── .helmignore             ← file/folder bỏ qua khi package
```

### Chart.yaml — Metadata Bắt Buộc

```yaml
apiVersion: v2           # v2 cho Helm 3, v1 cho Helm 2 (deprecated)
name: my-app             # tên chart, phải khớp với tên thư mục
description: A Helm chart for deploying my-app
type: application        # "application" hoặc "library"

# Semantic Versioning (quản lý phiên bản ngữ nghĩa): MAJOR.MINOR.PATCH
version: 1.2.3           # phiên bản CHART — tăng khi thay đổi chart structure
appVersion: "2.0.1"      # phiên bản APP đang được đóng gói — để reference

maintainers:
  - name: Team DevOps
    email: devops@example.com

dependencies:            # chart phụ thuộc
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled   # chỉ cài khi postgresql.enabled=true
```

### values.yaml — Giá Trị Mặc Định

```yaml
# Số lượng replica mặc định
replicaCount: 1

image:
  repository: my-registry.io/my-app
  tag: "latest"            # nên dùng commit SHA trong production
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: false           # tắt mặc định, bật khi cần
  hostname: my-app.example.com
  tls: false

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

postgresql:
  enabled: true            # cài postgresql sub-chart
  auth:
    database: myapp_db
```

### Deployment Template Mẫu

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}   # dùng helper từ _helpers.tpl
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}       # lấy từ values.yaml
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}  # toYaml chuyển object thành YAML string
```

---

## Helm Repository

**Repository (Kho Lưu Trữ Chart)** là nơi lưu trữ và chia sẻ Helm chart. Có hai loại chính:

### HTTP/HTTPS Repository (Truyền Thống)

```bash
# Thêm repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable   https://charts.helm.sh/stable

# Cập nhật index từ tất cả repo đã add
helm repo update

# Tìm kiếm chart
helm search repo nginx
helm search repo postgresql --versions   # xem tất cả version

# Xem thông tin chi tiết
helm show values bitnami/postgresql      # xem values mặc định
helm show chart bitnami/postgresql       # xem Chart.yaml
```

### OCI Registry (Chuẩn Mới — Lưu Chart Trong Container Registry)

```bash
# Helm 3.8+ hỗ trợ OCI natively (không cần enable flag)

# Login vào OCI registry
helm registry login registry.example.com --username user --password pass

# Push chart lên OCI registry
helm package my-app                      # tạo my-app-1.2.3.tgz
helm push my-app-1.2.3.tgz oci://registry.example.com/charts

# Pull và install từ OCI
helm install my-release oci://registry.example.com/charts/my-app --version 1.2.3
```

### Repository Phổ Biến

```
Artifact Hub (artifacthub.io) — tập trung chart từ nhiều repo
├── Bitnami            — chart chất lượng cao cho PostgreSQL, Redis, Kafka...
├── Ingress NGINX      — NGINX Ingress Controller
├── Cert-Manager       — TLS certificate management
├── Prometheus         — kube-prometheus-stack
└── ArgoCD             — ArgoCD chính thức
```

---

## Các Lệnh Helm Thiết Yếu

### Cài Đặt (Install)

```bash
# Cú pháp cơ bản
helm install <release-name> <chart> [flags]

# Ví dụ thực tế
helm install my-nginx bitnami/nginx
helm install my-nginx bitnami/nginx -n production --create-namespace
helm install my-app ./my-app -f values-prod.yaml
helm install my-app ./my-app --set image.tag=v1.2.3 --set replicaCount=3

# Dry-run — xem manifest sẽ tạo ra mà không thực sự apply
helm install my-app ./my-app --dry-run --debug

# Chờ cho đến khi deployment healthy
helm install my-app ./my-app --wait --timeout 5m
```

### Nâng Cấp (Upgrade)

```bash
# Upgrade release đang chạy
helm upgrade my-app ./my-app
helm upgrade my-app ./my-app -f values-prod.yaml --set image.tag=v1.3.0

# --install: install nếu chưa có, upgrade nếu đã có (upsert)
helm upgrade --install my-app ./my-app -f values-prod.yaml

# Atomic: rollback tự động nếu upgrade thất bại
helm upgrade my-app ./my-app --atomic --timeout 5m

# Giữ nguyên values đã set trước đó khi upgrade
helm upgrade my-app ./my-app --reuse-values --set image.tag=v1.3.1
```

### Rollback

```bash
# Xem lịch sử revision
helm history my-app

# Rollback về revision trước
helm rollback my-app

# Rollback về revision cụ thể
helm rollback my-app 2

# Rollback với wait
helm rollback my-app 2 --wait --timeout 3m
```

### Kiểm Tra và Debug

```bash
# Xem trạng thái release
helm status my-app
helm status my-app -n production

# Xem values đang dùng (bao gồm computed values)
helm get values my-app           # values đã set bởi user
helm get values my-app --all     # kể cả default values

# Xem manifest đã render
helm get manifest my-app

# Render template local (không install)
helm template my-app ./my-app -f values-prod.yaml

# Kiểm tra chart có lỗi không
helm lint ./my-app
helm lint ./my-app -f values-prod.yaml   # lint với specific values
```

### Gỡ Cài Đặt (Uninstall)

```bash
helm uninstall my-app
helm uninstall my-app -n production

# Giữ history để có thể rollback sau (mặc định là xóa luôn history)
helm uninstall my-app --keep-history
```

---

## Quản Lý Values

### Thứ Tự Ưu Tiên Values (Từ Thấp Đến Cao)

```
1. values.yaml trong chart             ← thấp nhất, default
2. values.yaml của parent chart        ← nếu là sub-chart
3. -f values-override.yaml             ← file override
4. --set key=value                     ← cao nhất, ghi đè tất cả
```

### Override Theo Môi Trường

```bash
# Cách 1: Multiple -f files (file sau ghi đè file trước)
helm upgrade my-app ./my-app \
  -f values.yaml \          # base values
  -f values-prod.yaml \     # production override
  -f secrets.yaml           # secrets (không commit lên Git)

# Cách 2: --set cho giá trị đơn lẻ
helm upgrade my-app ./my-app \
  -f values-prod.yaml \
  --set image.tag=abc123def \     # commit SHA
  --set replicaCount=5

# Cách 3: --set-string cho string (tránh type conversion tự động)
helm upgrade my-app ./my-app \
  --set-string "annotations.timestamp=2026-05-10"
```

### values-prod.yaml Mẫu

```yaml
# Chỉ chứa giá trị KHÁC với values.yaml mặc định
# KHÔNG copy toàn bộ values.yaml

replicaCount: 3

image:
  tag: "v1.2.3"              # pin version cụ thể

resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2000m"
    memory: "2Gi"

ingress:
  enabled: true
  hostname: my-app.prod.example.com
  tls: true

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
```

---

## Vòng Đời Release

```
                   helm install
                       │
                       ▼
              ┌─────────────────┐
              │  Release v1     │
              │  revision: 1    │
              │  status: deployed│
              └────────┬────────┘
                       │
              helm upgrade (image mới)
                       │
                       ▼
              ┌─────────────────┐
              │  Release v1     │
              │  revision: 2    │
              │  status: deployed│
              └────────┬────────┘
                       │
              helm upgrade --atomic (thất bại)
                       │
                       ▼
              ┌─────────────────┐
              │  Release v1     │    Tự động rollback
              │  revision: 2    │ ←──────────────────
              │  status: deployed│    về revision 2
              └────────┬────────┘
                       │
              helm rollback (thủ công)
                       │
                       ▼
              ┌─────────────────┐
              │  Release v1     │
              │  revision: 4    │  (revision 4, quay về state của rev 1)
              │  status: deployed│
              └────────┬────────┘
                       │
              helm uninstall
                       │
                       ▼
                  (released)
```

---

## Câu Hỏi Phỏng Vấn

**`helm upgrade --install` khác `helm install` thế nào? Khi nào dùng?**

> `helm install` chỉ dùng để cài lần đầu — sẽ fail nếu release đã tồn tại. `helm upgrade --install` là idempotent (bất biến — chạy nhiều lần cho cùng kết quả): install nếu chưa có, upgrade nếu đã có. Đây là pattern chuẩn cho CI/CD pipeline vì pipeline không cần biết đây là deploy lần đầu hay lần thứ n — chạy một lệnh duy nhất là đủ. Ví dụ trong GitHub Actions: `helm upgrade --install my-app ./chart -f values-prod.yaml --atomic --wait`.

**Helm lưu release history ở đâu? Tại sao quan trọng?**

> Helm 3 lưu release history dưới dạng Kubernetes Secret trong cùng namespace với release. Mỗi revision là một Secret riêng với tên dạng `sh.helm.release.v1.<release-name>.v<revision>`. Secret chứa thông tin release được compress và base64-encode. Quan trọng vì: (1) `helm rollback` đọc từ đây để biết manifest cần apply cho version cũ; (2) `helm history` hiển thị audit trail; (3) `helm get` lấy values và manifest của release hiện tại. Mặc định Helm giữ 10 revision — có thể cấu hình qua `--history-max` hoặc `HELM_MAX_HISTORY` env var.

**`helm template` vs `helm install --dry-run` khác nhau thế nào?**

> `helm template` render manifest local mà không kết nối cluster — chỉ dùng thông tin trong chart và values. `helm install --dry-run` cũng render manifest nhưng **kết nối cluster** để validate (kiểm tra hợp lệ) manifest với Kubernetes API server — phát hiện được lỗi như invalid apiVersion, wrong field name. `helm template` dùng khi muốn xem output nhanh mà không cần access cluster, hoặc để pipe vào `kubectl apply`. `--dry-run` dùng khi muốn validation đầy đủ. Thực tế: dùng `helm lint` + `helm template | kubectl apply --dry-run=client` để check đầy đủ.

---

## Cheat Sheet Lệnh Helm

```bash
# === REPO ===
helm repo add <name> <url>
helm repo update
helm search repo <keyword>
helm show values <chart>

# === INSTALL / UPGRADE ===
helm install <release> <chart> -n <namespace> -f values.yaml
helm upgrade <release> <chart> -f values.yaml --set key=value
helm upgrade --install <release> <chart> -f values.yaml --atomic
helm install <release> <chart> --dry-run --debug

# === INSPECT ===
helm list -A                         # tất cả release mọi namespace
helm status <release>
helm history <release>
helm get values <release>
helm get values <release> --all
helm get manifest <release>

# === ROLLBACK ===
helm rollback <release> [revision]

# === DELETE ===
helm uninstall <release> -n <namespace>

# === LOCAL DEVELOPMENT ===
helm create <chart-name>             # tạo chart template mới
helm lint <chart-dir>
helm lint <chart-dir> -f values-prod.yaml
helm template <release> <chart-dir> -f values.yaml
helm package <chart-dir>             # tạo .tgz để push lên repo
helm dependency update               # download sub-chart
```
