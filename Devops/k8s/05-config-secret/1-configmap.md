# ConfigMap — Bản Đồ Cấu Hình Kubernetes

> Hướng dẫn toàn diện về ConfigMap (Bản Đồ Cấu Hình): tạo, cập nhật, và các cách tiêm cấu hình vào Pod qua biến môi trường hoặc volume mount.

## Mục Lục

1. [ConfigMap Là Gì?](#configmap-là-gì)
2. [Tạo ConfigMap](#tạo-configmap)
3. [Mount ConfigMap Vào Pod](#mount-configmap-vào-pod)
4. [Cập Nhật ConfigMap](#cập-nhật-configmap)
5. [ConfigMap Immutable](#configmap-immutable)
6. [Best Practice](#best-practice)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## ConfigMap Là Gì?

**ConfigMap** là tài nguyên Kubernetes lưu trữ dữ liệu cấu hình phi nhạy cảm dưới dạng key-value. Mục tiêu là **tách cấu hình ra khỏi image container** — cùng một image có thể chạy với cấu hình khác nhau tuỳ môi trường.

```
image: myapp:v1.2.3  (bất biến, không chứa config)
       +
ConfigMap: app-config-prod  (LOG_LEVEL=warn, DB_HOST=prod-db)
ConfigMap: app-config-dev   (LOG_LEVEL=debug, DB_HOST=dev-db)
       =
Cùng image, hành vi khác nhau theo môi trường
```

**Phạm vi:** ConfigMap thuộc về một **namespace** — Pod chỉ có thể dùng ConfigMap trong cùng namespace.

**Giới hạn dung lượng:** 1 MiB (1,048,576 bytes). Nếu cần lưu dữ liệu lớn hơn, dùng volume hoặc database.

---

## Tạo ConfigMap

### Cách 1: Khai Báo YAML (Khuyến Nghị)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
  labels:
    app: myapp
    version: v1
data:
  # Key-value đơn giản (biến môi trường)
  APP_ENV: "production"
  LOG_LEVEL: "warn"
  DB_HOST: "postgres.production.svc.cluster.local"
  DB_PORT: "5432"
  MAX_CONNECTIONS: "100"
  FEATURE_FLAG_DARK_MODE: "true"

  # Nội dung file (multi-line string dùng | hoặc >)
  application.yml: |
    server:
      port: 8080
      compression:
        enabled: true
    logging:
      level:
        root: WARN
        com.myapp: INFO
    spring:
      datasource:
        url: jdbc:postgresql://postgres:5432/mydb

  nginx.conf: |
    upstream app {
      server localhost:8080;
    }
    server {
      listen 80;
      location / {
        proxy_pass http://app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
      }
      location /health {
        return 200 "ok";
      }
    }
```

### Cách 2: Tạo Từ Dòng Lệnh

```bash
# Từ literal (giá trị trực tiếp)
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=warn \
  --from-literal=DB_HOST=postgres.internal

# Từ một file (tên file trở thành key)
kubectl create configmap nginx-config \
  --from-file=nginx.conf                   # key = "nginx.conf"

# Từ file với key tuỳ chỉnh
kubectl create configmap nginx-config \
  --from-file=config=nginx.conf            # key = "config"

# Từ toàn bộ thư mục (mỗi file là một key)
kubectl create configmap app-configs \
  --from-file=./config-dir/

# Từ file env (định dạng KEY=VALUE, mỗi dòng là một key)
kubectl create configmap env-config \
  --from-env-file=.env.production
```

### Xem ConfigMap

```bash
# Liệt kê ConfigMap trong namespace
kubectl get configmap -n production

# Xem nội dung chi tiết (hiện toàn bộ giá trị)
kubectl get configmap app-config -o yaml

# Mô tả (không hiện giá trị data)
kubectl describe configmap app-config

# Lấy giá trị một key cụ thể
kubectl get configmap app-config -o jsonpath='{.data.LOG_LEVEL}'
```

---

## Mount ConfigMap Vào Pod

### Phương Pháp 1: envFrom — Tiêm Toàn Bộ Làm Biến Môi Trường

Tiêm tất cả key trong ConfigMap thành biến môi trường trong container:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:v1.2.3
          envFrom:
            - configMapRef:
                name: app-config       # toàn bộ key trong app-config → env var
            - configMapRef:
                name: feature-flags    # nhiều ConfigMap được gộp lại
                optional: true         # không lỗi nếu ConfigMap chưa tồn tại
```

Container sẽ thấy: `APP_ENV=production`, `LOG_LEVEL=warn`, `DB_HOST=postgres.internal`...

**Lưu ý:** Key trong ConfigMap phải là tên biến môi trường hợp lệ (chỉ chữ, số, dấu gạch dưới). Key như `nginx.conf` sẽ bị bỏ qua hoặc gây lỗi.

### Phương Pháp 2: env[].valueFrom — Tiêm Từng Key Có Chọn Lọc

Chỉ lấy một số key cụ thể, có thể đặt tên biến khác:

```yaml
containers:
  - name: app
    image: myapp:v1.2.3
    env:
      - name: ENVIRONMENT          # tên biến trong container
        valueFrom:
          configMapKeyRef:
            name: app-config       # tên ConfigMap
            key: APP_ENV           # key cần lấy
      - name: DATABASE_HOST
        valueFrom:
          configMapKeyRef:
            name: app-config
            key: DB_HOST
            optional: false        # sẽ lỗi nếu key không tồn tại (mặc định)
      - name: SIDECAR_LOG_LEVEL
        valueFrom:
          configMapKeyRef:
            name: sidecar-config
            key: LOG_LEVEL
            optional: true         # không lỗi nếu key/ConfigMap không tồn tại
```

### Phương Pháp 3: Volume Mount — Mount Thành File

Phương pháp này mount ConfigMap thành file trong filesystem container. **Hỗ trợ hot-reload** — file tự cập nhật khi ConfigMap thay đổi (sau 1–2 phút).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      volumes:
        - name: app-config-vol
          configMap:
            name: app-config             # ConfigMap cần mount
            defaultMode: 0644            # quyền file (octal)
            items:                       # chỉ mount một số key (tuỳ chọn)
              - key: application.yml     # key trong ConfigMap
                path: application.yml   # tên file trong container
              - key: nginx.conf
                path: nginx/nginx.conf  # có thể đặt trong thư mục con

      containers:
        - name: app
          image: myapp:v1.2.3
          volumeMounts:
            - name: app-config-vol
              mountPath: /etc/config    # thư mục chứa file config
              readOnly: true            # khuyến nghị readOnly cho config
```

Kết quả — container thấy:
```
/etc/config/application.yml   ← nội dung key "application.yml"
/etc/config/nginx/nginx.conf  ← nội dung key "nginx.conf"
```

### Phương Pháp 4: Mount Vào File Cụ Thể (subPath)

Mount một key ConfigMap vào một file cụ thể, không tạo thư mục:

```yaml
volumes:
  - name: nginx-config
    configMap:
      name: app-config

containers:
  - name: nginx
    image: nginx:1.25
    volumeMounts:
      - name: nginx-config
        mountPath: /etc/nginx/nginx.conf  # mount thẳng vào file
        subPath: nginx.conf               # key trong ConfigMap
        readOnly: true
```

**Cảnh báo với subPath:** File mount qua `subPath` **không tự cập nhật** khi ConfigMap thay đổi — hành vi giống env var. Dùng khi cần mount vào đường dẫn cụ thể mà không ảnh hưởng file khác trong thư mục đó.

---

## Cập Nhật ConfigMap

### Cập Nhật Bằng kubectl

```bash
# Chỉnh sửa trực tiếp trong editor
kubectl edit configmap app-config -n production

# Apply từ file YAML mới
kubectl apply -f app-config-v2.yaml

# Patch một key cụ thể
kubectl patch configmap app-config \
  -p '{"data":{"LOG_LEVEL":"debug"}}'

# Thay thế hoàn toàn (replace — cẩn thận!)
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=debug \
  --dry-run=client -o yaml | kubectl replace -f -
```

### Điều Gì Xảy Ra Sau Khi Cập Nhật?

```
ConfigMap thay đổi
       │
       ├── Pod dùng env var / envFrom?
       │   └── ❌ Không tự cập nhật — phải restart Pod
       │       kubectl rollout restart deployment/myapp
       │
       └── Pod dùng volume mount?
           └── ✅ Tự cập nhật sau ~1–2 phút (kubelet sync period)
               Tuy nhiên ứng dụng phải tự reload file!
               Dùng inotify, signal handler, hoặc polling
```

### Restart Pod Để Nhận Config Mới (Khi Dùng Env Var)

```bash
# Rolling restart — không downtime
kubectl rollout restart deployment/myapp -n production

# Xem tiến trình restart
kubectl rollout status deployment/myapp -n production

# Rollback nếu có vấn đề
kubectl rollout undo deployment/myapp -n production
```

---

## ConfigMap Immutable

Từ Kubernetes 1.21+, ConfigMap hỗ trợ `immutable: true`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2    # đặt version vào tên
immutable: true           # không thể sửa data sau khi tạo
data:
  APP_ENV: "production"
  LOG_LEVEL: "warn"
```

**Lợi ích:**
- Bảo vệ khỏi thay đổi vô tình gây ảnh hưởng Pod đang chạy
- **Cải thiện hiệu năng cluster:** kubelet không cần watch ConfigMap immutable → giảm tải API Server và etcd đáng kể với cluster lớn (hàng nghìn Pod)

**Quy trình thay đổi config với immutable:**
```
1. Tạo ConfigMap mới: app-config-v3
2. Cập nhật Deployment reference: configMapRef.name = app-config-v3
3. Deployment rolling update tự động
4. Xoá ConfigMap cũ: kubectl delete configmap app-config-v2
```

---

## Best Practice

### Tổ Chức ConfigMap

```yaml
# Tách theo mục đích — không gộp tất cả vào một ConfigMap lớn
app-config          # cấu hình ứng dụng
nginx-config        # cấu hình nginx
feature-flags       # feature flag riêng biệt (thay đổi thường xuyên)
monitoring-config   # cấu hình prometheus scrape

# Đặt tên có version khi dùng immutable
app-config-v1, app-config-v2, ...
```

### Không Làm (Anti-Pattern)

```yaml
# ❌ Sai: đặt secret trong ConfigMap
data:
  DB_PASSWORD: "mysecretpassword"  # dùng Secret thay thế!
  API_KEY: "sk-abc123"

# ❌ Sai: dữ liệu quá lớn trong ConfigMap
data:
  large-dataset.json: |
    { ... 500KB data ... }   # dùng PV hoặc object storage thay thế

# ❌ Sai: không đặt namespace tường minh
metadata:
  name: app-config
  # namespace bị bỏ qua → phụ thuộc kubectl context hiện tại
```

### Nên Làm

```bash
# ✅ Validate YAML trước khi apply
kubectl apply --dry-run=server -f app-config.yaml

# ✅ Xem diff trước khi apply thay đổi
kubectl diff -f app-config.yaml

# ✅ Dùng label để quản lý ConfigMap theo app
kubectl get configmap -l app=myapp -n production

# ✅ Backup ConfigMap quan trọng
kubectl get configmap app-config -o yaml > backup/app-config-$(date +%Y%m%d).yaml
```

---

## Câu Hỏi Phỏng Vấn

**Sự khác nhau giữa `envFrom` và `env[].valueFrom`?**

> `envFrom` tiêm **toàn bộ** key trong ConfigMap/Secret thành biến môi trường — thuận tiện nhưng ít kiểm soát, có thể gây xung đột tên biến nếu nhiều ConfigMap có key trùng. `env[].valueFrom` chỉ lấy **từng key cụ thể** và cho phép đặt tên biến tuỳ ý trong container — linh hoạt hơn, kiểm soát rõ ràng hơn, phù hợp khi chỉ cần một số key hoặc cần đổi tên.

**Volume mount ConfigMap có tự cập nhật khi ConfigMap thay đổi không?**

> **Có**, nhưng có điều kiện: (1) Cần khoảng **1–2 phút** để kubelet phát hiện thay đổi và cập nhật file trong container — đây là `--sync-frequency` mặc định của kubelet. (2) Ứng dụng phải **tự reload** — Kubernetes chỉ cập nhật file, không restart container. (3) Ngoại lệ: nếu mount qua `subPath`, file **không bao giờ tự cập nhật** — phải restart Pod. Đây là hành vi cần nhớ kỹ trong phỏng vấn.

**Tại sao nên dùng immutable ConfigMap trong production?**

> Hai lý do chính: (1) **An toàn** — ngăn thay đổi vô tình làm hỏng ứng dụng đang chạy trong production; với immutable, muốn thay đổi phải tạo ConfigMap mới và cập nhật Deployment có chủ ý. (2) **Hiệu năng** — kubelet không cần thiết lập watch connection lên API Server cho ConfigMap immutable, tiết kiệm kết nối và CPU đáng kể khi cluster có hàng nghìn Pod mỗi Pod xem nhiều ConfigMap.

**Làm thế nào để một Pod dùng ConfigMap từ namespace khác?**

> **Không thể trực tiếp** — ConfigMap bị giới hạn trong namespace, Pod chỉ dùng được ConfigMap trong cùng namespace. Giải pháp: (1) Copy ConfigMap sang namespace cần dùng (thủ công hoặc dùng tool như `config-syncer`); (2) Dùng External Secrets Operator để đồng bộ từ nguồn trung tâm sang nhiều namespace; (3) Thiết kế lại để dùng chung namespace nếu hợp lý. Đây là lý do tại sao hệ thống cấu hình tập trung như Vault hoặc AWS Parameter Store được ưa chuộng trong multi-tenant cluster.
