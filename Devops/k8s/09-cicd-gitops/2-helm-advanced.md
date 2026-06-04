# Helm Advanced — Template, Hook, Test và Chart Dependency Nâng Cao

> Kỹ thuật Helm nâng cao: Go template syntax (cú pháp mẫu Go), helper function, lifecycle hook (móc vòng đời), chart testing, dependency management (quản lý phụ thuộc), và OCI registry.

## Mục Lục

1. [Go Template Syntax](#go-template-syntax)
2. [Helper Templates — _helpers.tpl](#helper-templates--_helperstpl)
3. [Helm Hooks — Móc Vòng Đời](#helm-hooks--móc-vòng-đời)
4. [Helm Test — Kiểm Thử Chart](#helm-test--kiểm-thử-chart)
5. [Chart Dependencies](#chart-dependencies)
6. [Library Charts](#library-charts)
7. [OCI Registry](#oci-registry)
8. [Best Practices Nâng Cao](#best-practices-nâng-cao)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Go Template Syntax

Helm sử dụng **Go template engine** (bộ xử lý mẫu Go) để render manifest. Hiểu cú pháp template là kỹ năng cốt lõi khi viết chart phức tạp.

### Biến và Built-in Objects (Đối Tượng Tích Hợp)

```yaml
# .Values — giá trị từ values.yaml và --set
{{ .Values.image.repository }}
{{ .Values.service.port }}

# .Chart — thông tin từ Chart.yaml
{{ .Chart.Name }}
{{ .Chart.Version }}
{{ .Chart.AppVersion }}

# .Release — thông tin về release hiện tại
{{ .Release.Name }}          # tên release: "my-app"
{{ .Release.Namespace }}     # namespace: "production"
{{ .Release.IsInstall }}     # true nếu là install lần đầu
{{ .Release.IsUpgrade }}     # true nếu là upgrade

# .Files — truy cập file trong chart (ngoài templates/)
{{ .Files.Get "config/app.conf" }}

# .Capabilities — thông tin về Kubernetes cluster
{{ .Capabilities.KubeVersion.Major }}
{{ .Capabilities.APIVersions.Has "apps/v1" }}
```

### Điều Kiện (Conditionals)

```yaml
# if / else if / else / end
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- else }}
# Không tạo Ingress
{{- end }}

# with — thay đổi scope (phạm vi biến .) tạm thời
{{- with .Values.resources }}
resources:
  requests:
    cpu: {{ .requests.cpu }}      # .requests thay vì .Values.resources.requests
    memory: {{ .requests.memory }}
{{- end }}

# Kiểm tra nil / empty
{{- if .Values.serviceAccount.name }}
serviceAccountName: {{ .Values.serviceAccount.name }}
{{- end }}
```

### Vòng Lặp (Range)

```yaml
# Lặp qua list
{{- range .Values.env }}
- name: {{ .name }}
  value: {{ .value | quote }}
{{- end }}

# Lặp qua map (key-value)
{{- range $key, $val := .Values.annotations }}
  {{ $key }}: {{ $val | quote }}
{{- end }}

# Lặp với index
{{- range $i, $host := .Values.ingress.hosts }}
- host: {{ $host }}
{{- end }}
```

### Pipeline và Hàm (Functions)

```yaml
# Pipeline: truyền kết quả của function này sang function tiếp theo
{{ .Values.image.tag | default "latest" | quote }}
# = quote(default("latest", .Values.image.tag))

# Các hàm phổ biến
{{ .Values.name | upper }}           # chuyển thành chữ hoa
{{ .Values.name | lower }}           # chuyển thành chữ thường
{{ .Values.name | title }}           # Title Case
{{ .Values.name | trim }}            # xóa whitespace đầu/cuối
{{ .Values.name | replace "-" "_" }} # thay thế ký tự
{{ .Values.name | trunc 63 }}        # cắt chuỗi tối đa 63 ký tự

# toYaml — chuyển object thành YAML block
resources:
  {{- toYaml .Values.resources | nindent 2 }}

# Ví dụ kết quả (với nindent 2):
#   requests:
#     cpu: "100m"
#   limits:
#     memory: "256Mi"

# quote — thêm dấu nháy kép
value: {{ .Values.env.DEBUG | quote }}   # → value: "true"

# default — giá trị mặc định nếu nil/empty
replicas: {{ .Values.replicaCount | default 1 }}

# required — báo lỗi nếu giá trị không được cung cấp
image: {{ required "image.repository là bắt buộc" .Values.image.repository }}
```

### Whitespace Control (Kiểm Soát Khoảng Trắng)

```yaml
# {{- ... }} — xóa newline/whitespace TRƯỚC action
# {{ ... -}} — xóa newline/whitespace SAU action
# {{- ... -}} — xóa cả hai phía

# KHÔNG có whitespace control:
labels:
  {{if .Values.env.production}}
  tier: production
  {{end}}
# Kết quả: có dòng trắng thừa

# CÓ whitespace control:
labels:
  {{- if .Values.env.production}}
  tier: production
  {{- end}}
# Kết quả: sạch, không có dòng trắng thừa
```

---

## Helper Templates — _helpers.tpl

**`_helpers.tpl`** là file chứa các named template (mẫu có tên) tái sử dụng trong toàn bộ chart. File bắt đầu bằng `_` nên Helm biết không render nó thành manifest.

### Định Nghĩa Helper

```yaml
# templates/_helpers.tpl

{{/*
Tên đầy đủ của release: "<release-name>-<chart-name>"
Tối đa 63 ký tự (giới hạn của DNS label trong K8s)
*/}}
{{- define "my-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Label chuẩn cho mọi resource trong chart
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ include "my-app.chart" . }}
{{ include "my-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels — dùng trong matchLabels và template.metadata.labels
*/}}
{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### Sử Dụng Helper Trong Template

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}          # gọi helper
  labels:
    {{- include "my-app.labels" . | nindent 4 }}   # gọi helper và indent 4 spaces
spec:
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
```

### Custom Helper Thực Tế

```yaml
{{/*
Tạo environment variables từ .Values.env (list) và .Values.envFrom (configmap/secret ref)
*/}}
{{- define "my-app.envVars" -}}
{{- range .Values.env }}
- name: {{ .name }}
  value: {{ .value | quote }}
{{- end }}
{{- range .Values.envFromSecrets }}
- name: {{ .name }}
  valueFrom:
    secretKeyRef:
      name: {{ .secretName }}
      key: {{ .key }}
{{- end }}
{{- end }}

# Sử dụng trong deployment.yaml:
env:
  {{- include "my-app.envVars" . | nindent 12 }}
```

---

## Helm Hooks — Móc Vòng Đời

**Hook (Móc)** là cơ chế cho phép can thiệp vào các điểm cụ thể trong vòng đời của release. Hook thực chất là Kubernetes Job hoặc Pod với annotation đặc biệt.

### Các Loại Hook

```
Vòng đời của helm install:
pre-install  → [Hook chạy] → install resources → post-install → [Hook chạy]

Vòng đời của helm upgrade:
pre-upgrade  → [Hook chạy] → upgrade resources → post-upgrade → [Hook chạy]

Vòng đời của helm rollback:
pre-rollback → [Hook chạy] → rollback         → post-rollback → [Hook chạy]

Vòng đời của helm uninstall:
pre-delete   → [Hook chạy] → delete resources → post-delete   → [Hook chạy]

Hook đặc biệt:
test         → chạy khi gọi "helm test"
```

### Ví Dụ: Database Migration Hook

```yaml
# templates/migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "my-app.fullname" . }}-db-migrate
  annotations:
    # Đây là hook pre-upgrade và pre-install
    "helm.sh/hook": pre-upgrade,pre-install
    # Xóa Job sau khi hoàn thành (không tích lũy Job cũ)
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
    # Thứ tự ưu tiên — số nhỏ chạy trước (mặc định: 0)
    "helm.sh/hook-weight": "-5"
spec:
  backoffLimit: 3   # thử lại tối đa 3 lần nếu thất bại
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["python", "manage.py", "migrate"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "my-app.fullname" . }}-secret
                  key: database-url
```

### Hook Delete Policy (Chính Sách Xóa Hook)

```yaml
# before-hook-creation — xóa resource hook cũ trước khi tạo hook mới
# hook-succeeded       — xóa sau khi hook thành công
# hook-failed          — xóa sau khi hook thất bại

# Thực tế hay dùng:
"helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
# Giữ Job lại nếu thất bại → dễ debug, tự động xóa nếu thành công
```

---

## Helm Test — Kiểm Thử Chart

**Helm Test** là cơ chế viết test tích hợp để verify deployment thành công sau khi install/upgrade. Test là Pod với annotation `helm.sh/hook: test`.

### Viết Test

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "my-app.fullname" . }}-test-connection
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args:
        - '--spider'                          # chỉ kiểm tra HTTP, không tải xuống
        - '--timeout=5'
        - 'http://{{ include "my-app.fullname" . }}:{{ .Values.service.port }}/health'
```

```yaml
# Test phức tạp hơn: kiểm tra database connection
# templates/tests/test-db.yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "my-app.fullname" . }}-test-db
  annotations:
    "helm.sh/hook": test
spec:
  restartPolicy: Never
  containers:
    - name: test
      image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
      command: ["python", "-c", "import psycopg2; psycopg2.connect('{{ .Values.database.url }}'); print('DB OK')"]
```

### Chạy Test

```bash
# Chạy test sau khi install
helm install my-app ./my-app
helm test my-app

# Xem log của test Pod
helm test my-app --logs

# Kết quả:
# NAME: my-app
# LAST DEPLOYED: ...
# NAMESPACE: default
# STATUS: deployed
# TEST SUITE:   my-app-test-connection
# Last Started: ...
# Last Completed: ...
# Phase: Succeeded    ← PASS
```

---

## Chart Dependencies

**Chart Dependencies (Phụ Thuộc Chart)** cho phép chart khai báo phụ thuộc vào chart khác. Ví dụ: chart của app phụ thuộc vào postgresql chart.

### Khai Báo Dependency Trong Chart.yaml

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "12.5.x"          # x = wildcard — lấy patch version mới nhất
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled   # chỉ cài khi postgresql.enabled=true trong values

  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled

  - name: common                # library chart
    version: "2.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    tags:
      - shared-lib
```

### Quản Lý Dependency

```bash
# Download tất cả dependency vào charts/ folder
helm dependency update ./my-app

# Kết quả tạo ra:
# charts/postgresql-12.5.6.tgz
# charts/redis-17.3.7.tgz
# Chart.lock (lock file — pin exact version)

# Kiểm tra dependency
helm dependency list ./my-app
```

### Override Values Của Sub-Chart

```yaml
# values.yaml của parent chart
# Override values của postgresql sub-chart
postgresql:
  enabled: true
  auth:
    username: myapp
    password: ""              # lấy từ existing secret
    existingSecret: myapp-db-secret
    secretKeys:
      adminPasswordKey: postgres-password
      userPasswordKey: password
  primary:
    resources:
      requests:
        cpu: "250m"
        memory: "256Mi"
  persistence:
    enabled: true
    size: 10Gi
    storageClass: "gp3"

redis:
  enabled: false             # tắt redis sub-chart
```

---

## Library Charts

**Library Chart (Chart Thư Viện)** là chart không tạo ra resource Kubernetes nào — chỉ cung cấp helper template tái sử dụng cho nhiều application chart.

```yaml
# Chart.yaml của library chart
apiVersion: v2
name: my-company-common
type: library             # ← khai báo là library
version: 1.0.0
description: Common Helm helpers for My Company charts
```

```yaml
# templates/_deployment.tpl trong library chart
{{- define "my-company-common.deployment" -}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-company-common.fullname" . }}
  labels:
    {{- include "my-company-common.labels" . | nindent 4 }}
    app.kubernetes.io/component: {{ .component | default "web" }}
spec:
  replicas: {{ .Values.replicaCount | default 1 }}
  ...
{{- end }}
```

```yaml
# Sử dụng trong application chart
# Chart.yaml
dependencies:
  - name: my-company-common
    version: "1.x.x"
    repository: "oci://registry.my-company.io/charts"

# templates/deployment.yaml — chỉ cần một dòng
{{- include "my-company-common.deployment" . }}
```

---

## OCI Registry

**OCI Registry (Sổ Đăng Ký OCI — Open Container Initiative)** cho phép lưu trữ Helm chart cùng chỗ với Docker image, trong cùng container registry. Đây là chuẩn mới được Helm 3.8+ hỗ trợ chính thức.

### Lợi Ích của OCI vs HTTP Repo

```
HTTP Helm Repository:        OCI Registry:
├── Cần index.yaml           ├── Không cần index.yaml
├── Public hoặc basic auth   ├── Dùng container registry auth
├── Phải host riêng          ├── Dùng chung ECR / GCR / ACR / GHCR
└── Helm-specific            └── Standard OCI artifact
```

### Workflow Với OCI

```bash
# === BUILD & PUSH ===
# Đóng gói chart thành .tgz
helm package ./my-app --version 1.2.3
# Tạo ra: my-app-1.2.3.tgz

# Login vào OCI registry
helm registry login ghcr.io --username $GITHUB_USER --password $GITHUB_TOKEN

# Push lên GitHub Container Registry
helm push my-app-1.2.3.tgz oci://ghcr.io/my-org/charts

# === PULL & INSTALL ===
# Install trực tiếp (không cần helm repo add)
helm install my-release oci://ghcr.io/my-org/charts/my-app --version 1.2.3

# Upgrade
helm upgrade my-release oci://ghcr.io/my-org/charts/my-app --version 1.3.0

# Pull về local để inspect
helm pull oci://ghcr.io/my-org/charts/my-app --version 1.2.3 --untar
```

---

## Best Practices Nâng Cao

### Template Best Practices

```yaml
# 1. Luôn dùng "required" cho giá trị bắt buộc
host: {{ required "Phải cung cấp .Values.ingress.host" .Values.ingress.host }}

# 2. Luôn quote string để tránh YAML type coercion
  value: {{ .Values.someFlag | quote }}    # "true" không phải boolean true

# 3. Dùng toYaml + nindent thay vì hardcode structure
  resources:
    {{- toYaml .Values.resources | nindent 4 }}

# 4. Tránh whitespace bằng {{- và -}}
  {{- if .Values.feature.enabled }}
  featureFlag: "enabled"
  {{- end }}

# 5. Validate value phức tạp với fail
{{- if and .Values.autoscaling.enabled (not .Values.resources.requests.cpu) }}
{{- fail "Khi bật autoscaling phải định nghĩa resources.requests.cpu" }}
{{- end }}
```

### Versionging và Release Strategy

```bash
# Semantic Versioning cho chart:
# MAJOR.MINOR.PATCH
# - MAJOR: breaking change (thay đổi gây phá vỡ compatibility)
# - MINOR: thêm feature mới, backward-compatible
# - PATCH: bugfix

# appVersion vs chart version:
# - chart version: tăng khi thay đổi chart structure/template
# - appVersion: tăng theo phiên bản của ứng dụng được đóng gói
# Hai version này độc lập nhau
```

---

## Câu Hỏi Phỏng Vấn

**Helm hook giải quyết vấn đề gì? Cho ví dụ thực tế?**

> Helm hook cho phép can thiệp vào vòng đời của release để chạy tác vụ chuẩn bị hoặc dọn dẹp. Ví dụ thực tế phổ biến nhất là **database migration**: trước khi upgrade app lên version mới có schema change, cần chạy migration script để update database schema. Dùng `pre-upgrade` hook với Job chạy `alembic upgrade head` (Python) hoặc `flyway migrate` (Java). Nếu migration thất bại, hook fail và Helm rollback — application không bao giờ chạy với schema chưa được migrate. Ví dụ khác: `post-install` hook để seed dữ liệu ban đầu, `pre-delete` hook để backup database trước khi uninstall.

**Khi nào nên dùng Library Chart thay vì copy-paste helper vào mỗi chart?**

> Library Chart phù hợp khi có nhiều application chart trong cùng organization và cần nhất quán về: label chuẩn (app.kubernetes.io/* labels), naming convention, security context mặc định, resource limit template, hoặc annotation tổ chức. Thay vì mỗi team tự viết `_helpers.tpl` với convention khác nhau, library chart tập trung standards vào một nơi. Khi cần update convention (ví dụ: thêm label mới cho compliance), chỉ update library chart và bump version — tất cả app chart sử dụng library tự động có thể upgrade lên. Không phù hợp nếu chỉ có 2-3 chart — overhead của quản lý library chart không đáng.

**Explain cách Helm xử lý secret trong values? Có vấn đề gì?**

> Helm truyền values dưới dạng plain text (văn bản thuần) qua CLI `--set` hoặc file `-f`. Values này được lưu trong Kubernetes Secret (dạng base64, không phải mã hóa thực sự) như là release history. Vấn đề: (1) Nếu lưu password trong values.yaml và commit lên Git → credentials bị lộ; (2) `helm get values my-app` hiển thị tất cả values kể cả secrets → ai có quyền đọc Secret trong namespace đều thấy được. Giải pháp tốt hơn: (1) **Helm Secrets plugin** + SOPS để mã hóa values trước khi commit Git; (2) **External Secrets Operator** — không lưu secret trong Helm values mà reference đến AWS Secrets Manager / Vault; (3) **Sealed Secrets** — mã hóa bằng public key, chỉ cluster controller mới giải mã được.
