# Kubernetes Objects — Tài Nguyên Kubernetes

> Trong Kubernetes, mọi thứ đều là **Object** (tài nguyên) — một bản ghi ý định được lưu trong etcd. Bạn mô tả trạng thái mong muốn bằng YAML, Kubernetes sẽ nỗ lực duy trì trạng thái đó.

## Mục Lục

1. [Cấu Trúc Object Cơ Bản](#cấu-trúc-object-cơ-bản)
2. [Workload Objects](#workload-objects)
3. [Networking Objects](#networking-objects)
4. [Storage Objects](#storage-objects)
5. [Configuration Objects](#configuration-objects)
6. [Security Objects](#security-objects)
7. [Namespace](#namespace)
8. [Label và Annotation](#label-và-annotation)
9. [Selector — Cơ Chế Liên Kết Object](#selector--cơ-chế-liên-kết-object)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Cấu Trúc Object Cơ Bản

Mọi Kubernetes Object đều có cùng cấu trúc YAML:

```yaml
apiVersion: apps/v1        # Group/Version: <group>/<version> hoặc <version> (core group)
kind: Deployment            # Loại tài nguyên
metadata:
  name: nginx-deployment    # Tên duy nhất trong namespace
  namespace: default        # Namespace chứa object (omit = default)
  labels:                   # Label (nhãn) — key:value tự định nghĩa
    app: nginx
    version: "1.0"
  annotations:              # Annotation (ghi chú) — metadata phi cấu trúc
    deployment.kubernetes.io/revision: "3"
    description: "Web server deployment"
spec:                       # Desired State — trạng thái mong muốn (BẠN viết)
  replicas: 3
  # ... chi tiết tuỳ loại object
status:                     # Current State — trạng thái thực tế (Kubernetes cập nhật)
  availableReplicas: 3
  readyReplicas: 3
  # ... do controller cập nhật, KHÔNG chỉnh tay
```

### Các Field Bắt Buộc

| Field          | Mô Tả                                                         |
| -------------- | ------------------------------------------------------------- |
| `apiVersion`   | Phiên bản API (xác định schema và tính năng hỗ trợ)          |
| `kind`         | Loại object (Pod, Deployment, Service...)                     |
| `metadata.name` | Tên object, duy nhất trong namespace                         |
| `spec`         | Trạng thái mong muốn — nội dung khác nhau tuỳ `kind`         |

---

## Workload Objects

### Pod — Đơn Vị Triển Khai Nhỏ Nhất

Pod là **nhóm một hoặc nhiều container** chia sẻ cùng network namespace và storage volume.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  labels:
    app: web
spec:
  # Init Container — chạy và hoàn thành trước các container chính
  initContainers:
    - name: wait-for-db
      image: busybox:1.35
      command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']

  containers:
    # Container chính
    - name: web
      image: nginx:1.25
      ports:
        - containerPort: 80
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: db_host
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
      livenessProbe:
        httpGet:
          path: /healthz
          port: 80
        initialDelaySeconds: 15
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /ready
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
      volumeMounts:
        - name: config-vol
          mountPath: /etc/nginx/conf.d

    # Sidecar Container — chạy song song container chính
    - name: log-collector
      image: fluentd:v1.16
      volumeMounts:
        - name: log-vol
          mountPath: /var/log

  volumes:
    - name: config-vol
      configMap:
        name: nginx-config
    - name: log-vol
      emptyDir: {}    # emptyDir: tạo mới khi Pod chạy, xoá khi Pod chết

  restartPolicy: Always   # Always | OnFailure | Never
```

**Pod lifecycle (Vòng đời Pod):**

```
Pending   → ContainerCreating → Running → Succeeded / Failed / Unknown
                                   │
                   ┌───────────────┴──────────────────┐
               Liveness fail                   Readiness fail
                   │                                   │
           Container restart               Xoá khỏi Service Endpoints
```

**Tại sao không tạo Pod trực tiếp?**
- Pod trực tiếp không tự hồi phục — nếu bị xoá thì mất
- Luôn dùng controller: Deployment, StatefulSet, DaemonSet, Job

---

### Deployment — Controller Cho Ứng Dụng Không Trạng Thái

Deployment quản lý ReplicaSet, đảm bảo đúng số Pod chạy và hỗ trợ rolling update:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: production
spec:
  replicas: 3                       # Số Pod mong muốn
  revisionHistoryLimit: 10          # Giữ lại 10 ReplicaSet cũ để rollback

  selector:
    matchLabels:
      app: nginx                    # Deployment quản lý Pod có label này

  strategy:
    type: RollingUpdate             # RollingUpdate | Recreate
    rollingUpdate:
      maxUnavailable: 1             # Tối đa 1 Pod không sẵn sàng trong quá trình update
      maxSurge: 1                   # Tối đa 1 Pod dư ra trong quá trình update

  template:                         # Template để tạo Pod — phần quan trọng nhất
    metadata:
      labels:
        app: nginx                  # Phải khớp với selector ở trên
        version: "1.25"
    spec:
      containers:
        - name: nginx
          image: nginx:1.25        # Thay đổi image để trigger rolling update
          ports:
            - containerPort: 80
```

**Lệnh Deployment thường dùng:**

```bash
# Rolling update — cập nhật image
kubectl set image deployment/nginx-deployment nginx=nginx:1.26

# Xem lịch sử rollout
kubectl rollout history deployment/nginx-deployment

# Rollback về phiên bản trước
kubectl rollout undo deployment/nginx-deployment

# Rollback về revision cụ thể
kubectl rollout undo deployment/nginx-deployment --to-revision=3

# Dừng rolling update tạm thời
kubectl rollout pause deployment/nginx-deployment

# Tiếp tục rolling update
kubectl rollout resume deployment/nginx-deployment

# Chờ đến khi deployment hoàn thành
kubectl rollout status deployment/nginx-deployment
```

**Cơ Chế Rolling Update:**
```
Trước: [v1][v1][v1]

Bước 1 (maxSurge=1): [v1][v1][v1][v2]     ← Tạo Pod v2 mới
Bước 2 (maxUnavailable=1): [v1][v1][v2]   ← Xoá Pod v1 khi v2 Ready
Bước 3: [v1][v2][v2]
Bước 4: [v2][v2][v2]                       ← Hoàn thành

→ Tổng thời gian downtime: 0 (zero-downtime deployment)
```

---

### ReplicaSet — Bộ Đảm Bảo Số Lượng Pod

ReplicaSet đảm bảo luôn có đúng N Pod đang chạy. Thường **không tạo trực tiếp** — Deployment quản lý ReplicaSet:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
```

```bash
# Xem ReplicaSet được tạo bởi Deployment
kubectl get replicaset -l app=nginx
```

---

### StatefulSet — Controller Cho Ứng Dụng Có Trạng Thái

StatefulSet dành cho workload cần **định danh ổn định** (tên Pod cố định) và **lưu trữ bền vững** (PVC riêng mỗi Pod):

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres-headless"    # Phải có Headless Service tương ứng
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data

  # Mỗi Pod nhận một PVC riêng — không dùng chung
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "fast-ssd"
        resources:
          requests:
            storage: 10Gi
```

**Đặc điểm StatefulSet:**

| Đặc Điểm                   | Mô Tả                                                        |
| -------------------------- | ------------------------------------------------------------ |
| **Tên Pod cố định**        | `postgres-0`, `postgres-1`, `postgres-2` — không thay đổi   |
| **Khởi động tuần tự**      | Pod-0 phải Ready trước khi Pod-1 được tạo                    |
| **Xoá ngược thứ tự**       | Pod-2 bị xoá trước Pod-1 trước Pod-0                         |
| **PVC riêng từng Pod**     | Mỗi Pod có PVC riêng, PVC không bị xoá khi Pod bị xoá       |
| **DNS ổn định**            | `postgres-0.postgres-headless.namespace.svc.cluster.local`   |

**Khi nào dùng StatefulSet?**
- Database (PostgreSQL, MySQL, MongoDB)
- Message broker (Kafka, RabbitMQ)
- Distributed cache (Redis Cluster)
- Bất kỳ ứng dụng nào cần biết mình là instance thứ mấy

---

### DaemonSet — Chạy Trên Mọi Node

DaemonSet đảm bảo **mỗi node** (hoặc node thoả điều kiện) chạy **đúng một Pod**:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-logger
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule            # Cho phép chạy trên control plane node
      containers:
        - name: fluentd
          image: fluentd:v1.16
          volumeMounts:
            - name: varlog
              mountPath: /var/log
      volumes:
        - name: varlog
          hostPath:
            path: /var/log              # Mount log folder của node vào container
```

**Use case phổ biến:**
- Log agent (Fluentd, Filebeat, Promtail)
- Monitoring agent (node-exporter)
- Network plugin (CNI agent: Calico, Cilium)
- Storage agent (CSI node driver)
- Security agent (Falco, Datadog Agent)

---

### Job và CronJob — Tác Vụ Một Lần và Định Kỳ

**Job** — chạy một tác vụ đến khi hoàn thành:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  completions: 1          # Cần bao nhiêu Pod hoàn thành thành công
  parallelism: 1          # Chạy bao nhiêu Pod song song
  backoffLimit: 4         # Số lần retry tối đa nếu Pod fail
  activeDeadlineSeconds: 300  # Timeout toàn bộ Job (300 giây)
  template:
    spec:
      containers:
        - name: migration
          image: my-app:1.0
          command: ["python", "manage.py", "migrate"]
      restartPolicy: OnFailure   # Job phải dùng OnFailure hoặc Never (không được Always)
```

**CronJob** — chạy Job theo lịch cron:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
spec:
  schedule: "0 2 * * *"           # Cú pháp cron: phút giờ ngày tháng thứ
  concurrencyPolicy: Forbid       # Allow | Forbid | Replace
  successfulJobsHistoryLimit: 3   # Giữ 3 Job thành công gần nhất
  failedJobsHistoryLimit: 1       # Giữ 1 Job thất bại gần nhất
  startingDeadlineSeconds: 60     # Bỏ qua nếu quá 60s không thể bắt đầu
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: report
              image: my-report:1.0
              command: ["python", "generate_report.py"]
          restartPolicy: OnFailure
```

---

## Networking Objects

### Service — Điểm Truy Cập Ổn Định Cho Pod

Service cung cấp IP ổn định và DNS cho nhóm Pod:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP           # ClusterIP | NodePort | LoadBalancer | ExternalName
  selector:
    app: nginx              # Chọn Pod có label app=nginx
  ports:
    - name: http
      port: 80              # Port của Service (client kết nối đến đây)
      targetPort: 8080      # Port của container trong Pod
      protocol: TCP
```

**Bốn loại Service:**

| Type               | Truy Cập Từ             | Use Case                                        |
| ------------------ | ----------------------- | ----------------------------------------------- |
| **ClusterIP**      | Chỉ trong cluster       | Internal service-to-service communication       |
| **NodePort**       | `NodeIP:NodePort`       | Expose cho external traffic (dev/test)          |
| **LoadBalancer**   | Cloud load balancer IP  | Expose production service trên cloud            |
| **ExternalName**   | DNS alias               | Alias sang external domain (db.example.com)     |

**Headless Service** (dùng với StatefulSet):

```yaml
spec:
  clusterIP: None     # clusterIP: None → DNS trả về danh sách IP Pod thay vì VIP
  selector:
    app: postgres
```

---

### Ingress — Bộ Định Tuyến HTTP/HTTPS

Ingress quản lý traffic HTTP/HTTPS vào cluster:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx           # Chỉ định Ingress Controller nào xử lý
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-cert     # Secret chứa TLS certificate
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

---

### NetworkPolicy — Chính Sách Mạng

Kiểm soát lưu lượng vào (ingress) và ra (egress) của Pod:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend               # Áp dụng cho Pod backend

  policyTypes:
    - Ingress                    # Kiểm soát traffic vào
    - Egress                     # Kiểm soát traffic ra

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend      # Chỉ cho phép Pod frontend gọi vào
      ports:
        - protocol: TCP
          port: 8080

  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database      # Chỉ cho phép gọi đến Pod database
      ports:
        - protocol: TCP
          port: 5432
    - to:                        # Cho phép DNS lookup
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
```

---

## Storage Objects

### PersistentVolume (PV — Volume Bền Vững)

PV là tài nguyên lưu trữ trong cluster, được admin tạo trước hoặc dynamic provisioning:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-001
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany              # RWX — nhiều node read/write cùng lúc
  persistentVolumeReclaimPolicy: Retain   # Retain | Recycle | Delete
  storageClassName: nfs-storage
  nfs:
    server: 192.168.1.100
    path: /exports/data
```

### PersistentVolumeClaim (PVC — Yêu Cầu Volume Bền Vững)

PVC là yêu cầu storage từ Pod — Kubernetes sẽ bind PVC với PV phù hợp:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data-pvc
spec:
  accessModes:
    - ReadWriteOnce              # RWO — một node read/write
  storageClassName: fast-ssd    # Phải khớp với StorageClass
  resources:
    requests:
      storage: 5Gi
```

---

## Configuration Objects

### ConfigMap — Lưu Cấu Hình

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Dạng key-value đơn giản
  DB_HOST: "postgres-service"
  DB_PORT: "5432"
  LOG_LEVEL: "info"

  # Dạng file content
  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://backend:8080;
      }
    }
```

**Mount ConfigMap vào Pod:**

```yaml
spec:
  containers:
    - name: app
      envFrom:
        - configMapRef:
            name: app-config           # Inject tất cả key thành biến môi trường
      volumeMounts:
        - name: config-vol
          mountPath: /etc/nginx/conf.d  # Mount file nginx.conf
  volumes:
    - name: config-vol
      configMap:
        name: app-config
        items:
          - key: nginx.conf
            path: default.conf          # Tên file sau khi mount
```

### Secret — Lưu Dữ Liệu Nhạy Cảm

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque                            # Opaque | kubernetes.io/tls | kubernetes.io/dockerconfigjson
data:
  username: cG9zdGdyZXM=              # base64("postgres")
  password: c3VwZXJzZWNyZXQ=          # base64("supersecret")
```

```bash
# Tạo Secret từ literal (không cần tự base64)
kubectl create secret generic db-credentials \
  --from-literal=username=postgres \
  --from-literal=password=supersecret

# Tạo TLS Secret từ cert file
kubectl create secret tls my-tls \
  --cert=tls.crt \
  --key=tls.key
```

---

## Security Objects

### ServiceAccount — Tài Khoản Dịch Vụ

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/my-role  # IRSA trên EKS
```

### Role và ClusterRole — Định Nghĩa Quyền

```yaml
# Role — chỉ có hiệu lực trong namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]              # "" = core API group
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list"]
```

```yaml
# ClusterRole — có hiệu lực toàn cluster
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-viewer
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
```

### RoleBinding — Gán Quyền Cho Subject

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: app-service-account
    namespace: production
  - kind: User
    name: "alice@example.com"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## Namespace

Namespace cung cấp phân vùng tài nguyên logic trong cluster:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
    team: backend
```

**Namespace mặc định:**

| Namespace         | Mục Đích                                              |
| ----------------- | ----------------------------------------------------- |
| `default`         | Namespace mặc định khi không chỉ định                |
| `kube-system`     | Thành phần hệ thống (CoreDNS, kube-proxy, metrics-server) |
| `kube-public`     | Dữ liệu public có thể đọc không cần xác thực         |
| `kube-node-lease` | Heartbeat lease object của kubelet                   |

**Tài nguyên có namespace và không có namespace:**

```bash
# Liệt kê tài nguyên có namespace
kubectl api-resources --namespaced=true

# Liệt kê tài nguyên không có namespace (cluster-scoped)
kubectl api-resources --namespaced=false
# → Node, PersistentVolume, ClusterRole, Namespace chính nó
```

**ResourceQuota — Giới Hạn Tài Nguyên Theo Namespace:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"
    services: "10"
    persistentvolumeclaims: "20"
```

---

## Label và Annotation

### Label — Nhãn Phục Vụ Selector

Label là cặp key-value được dùng để **identify và select** object:

```bash
# Thêm label
kubectl label pod nginx-pod env=production

# Xoá label
kubectl label pod nginx-pod env-

# Select theo label
kubectl get pods -l "app=nginx,env=production"
kubectl get pods -l "env in (production,staging)"
kubectl get pods -l "env notin (development)"
kubectl get pods -l "!deprecated"   # Không có label deprecated
```

**Label key conventions (quy ước đặt tên):**

| Key                            | Ý Nghĩa             | Ví Dụ           |
| ------------------------------ | ------------------- | --------------- |
| `app.kubernetes.io/name`       | Tên ứng dụng        | `nginx`         |
| `app.kubernetes.io/version`    | Phiên bản           | `1.25.0`        |
| `app.kubernetes.io/component`  | Thành phần          | `frontend`      |
| `app.kubernetes.io/part-of`    | Thuộc hệ thống nào  | `my-platform`   |
| `app.kubernetes.io/managed-by` | Ai quản lý          | `helm`          |

### Annotation — Ghi Chú Metadata

Annotation lưu metadata phi cấu trúc, **không dùng để select**, thường dùng cho:

```yaml
metadata:
  annotations:
    # Ghi chú cho con người
    description: "Nginx web server cho frontend"
    contact: "team-backend@example.com"

    # Công cụ và hệ thống đọc
    deployment.kubernetes.io/revision: "5"
    kubectl.kubernetes.io/last-applied-configuration: "{...}"

    # Cấu hình Ingress Controller
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # cert-manager
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

**Label vs Annotation:**

| Tiêu Chí             | Label                            | Annotation                        |
| -------------------- | -------------------------------- | --------------------------------- |
| **Dùng để select**   | Có (selector, nodeSelector...)   | Không                             |
| **Kích thước**       | Nhỏ (giới hạn 63 ký tự per key) | Lớn (không giới hạn thực tế)     |
| **Mục đích chính**   | Identify và group object         | Metadata, config cho tools        |

---

## Selector — Cơ Chế Liên Kết Object

Selector là cách các object "tìm" nhau trong Kubernetes:

```yaml
# Deployment tìm Pod để quản lý
spec:
  selector:
    matchLabels:
      app: nginx
    # hoặc dùng matchExpressions
    matchExpressions:
      - key: environment
        operator: In           # In | NotIn | Exists | DoesNotExist
        values:
          - production
          - staging

# Service tìm Pod để route traffic
spec:
  selector:
    app: nginx                 # Chỉ hỗ trợ equality-based (key: value)
```

**Chuỗi quan hệ thông qua Label:**

```
Deployment (selector: app=nginx)
    ↓ quản lý
ReplicaSet (selector: app=nginx, pod-template-hash=abc123)
    ↓ tạo và quản lý
Pod (labels: app=nginx, pod-template-hash=abc123)
    ↑ route traffic đến
Service (selector: app=nginx)
    ↑ expose qua HTTP
Ingress (backend: nginx-service)
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Sự khác biệt giữa Deployment và StatefulSet?

**Trả lời:** Deployment dành cho ứng dụng **không trạng thái** (stateless) — Pod có thể thay thế cho nhau, có thể scale tự do, mỗi Pod nhận tên ngẫu nhiên. StatefulSet dành cho ứng dụng **có trạng thái** (stateful) — Pod có tên cố định (pod-0, pod-1), khởi động/xoá tuần tự, mỗi Pod có PVC riêng, DNS ổn định. Dùng StatefulSet cho database, message broker, distributed cache.

### Q2: Tại sao không nên tạo Pod trực tiếp trong production?

**Trả lời:** Pod trực tiếp không có cơ chế tự hồi phục (self-healing) — nếu node crash hoặc Pod bị xoá, không có gì tạo lại nó. Deployment (hoặc StatefulSet) luôn đảm bảo có đúng số Pod chạy. Ngoài ra, Deployment hỗ trợ rolling update, rollback, và scaling — những tính năng thiết yếu cho production.

### Q3: ConfigMap và Secret khác nhau thế nào? Khi nào dùng cái nào?

**Trả lời:** ConfigMap lưu dữ liệu không nhạy cảm (URL, port, feature flag). Secret lưu dữ liệu nhạy cảm (password, API key, TLS cert) — được mã hoá base64 khi lưu vào YAML (không phải mã hoá thật sự), và có thể mã hoá thật sự trong etcd. Cả hai đều mount vào Pod qua env hoặc volume. Trên production: bật Secret encryption at rest trong etcd và dùng External Secrets Operator để lấy từ Vault hoặc AWS Secrets Manager.

### Q4: Label và Selector hoạt động ra sao? Tại sao quan trọng?

**Trả lời:** Label là cặp key-value gắn vào object. Selector là bộ lọc dùng label để tìm object. Đây là cơ chế kết nối các object trong K8s: Deployment dùng selector để tìm Pod mình quản lý, Service dùng selector để tìm Pod cần route traffic, NetworkPolicy dùng podSelector để tìm Pod cần áp policy. Nếu label sai, Service không route được, Deployment không quản lý Pod.

### Q5: Job khác CronJob thế nào? Khi nào dùng Job?

**Trả lời:** Job chạy một tác vụ đến khi **hoàn thành thành công** và kết thúc — dùng cho migration, batch processing một lần, initialization. CronJob tạo Job theo lịch cron định kỳ — dùng cho báo cáo hàng ngày, backup định kỳ, cleanup cũ. Lưu ý: Job phải dùng `restartPolicy: OnFailure` hoặc `Never` (không được `Always`).

---

**Xem tiếp:** [request-flow.md](./request-flow.md) để hiểu luồng xử lý đầy đủ từ kubectl đến Pod chạy.
