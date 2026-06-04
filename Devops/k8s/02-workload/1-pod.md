# Pod — Đơn Vị Triển Khai Nhỏ Nhất

> Pod là đơn vị tính toán có thể triển khai nhỏ nhất trong Kubernetes — bao gồm một hoặc nhiều container dùng chung mạng và lưu trữ, cùng thực hiện một mục đích chung.

## Mục Lục

1. [Pod Là Gì?](#pod-là-gì)
2. [Pod Lifecycle — Vòng Đời Pod](#pod-lifecycle--vòng-đời-pod)
3. [Multi-Container Pod — Pod Nhiều Container](#multi-container-pod--pod-nhiều-container)
4. [Init Container — Container Khởi Tạo](#init-container--container-khởi-tạo)
5. [Sidecar Container — Container Phụ Trợ](#sidecar-container--container-phụ-trợ)
6. [Resource Request và Limit](#resource-request-và-limit)
7. [Pod Manifest Đầy Đủ](#pod-manifest-đầy-đủ)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Pod Là Gì?

**Pod** là một nhóm một hoặc nhiều container chạy cùng nhau trên một node, chia sẻ:
- **Không gian mạng (network namespace)**: tất cả container trong Pod dùng cùng địa chỉ IP và cổng (port). Giao tiếp qua `localhost`.
- **Không gian lưu trữ (storage)**: có thể mount cùng Volume.
- **Vòng đời**: container trong Pod cùng tạo và cùng xoá.

```
┌────────────────────────── Pod ──────────────────────────┐
│                                                          │
│  ┌──────────────────┐   ┌──────────────────┐            │
│  │  Container chính │   │  Sidecar Container│            │
│  │  (app:8080)      │   │  (log-agent:0)    │            │
│  └──────────────────┘   └──────────────────┘            │
│                                                          │
│  Shared Network: IP = 10.244.1.5                         │
│  Shared Volume: /var/log/app                             │
└──────────────────────────────────────────────────────────┘
```

### Tại Sao Pod Không Nên Tạo Trực Tiếp?

Pod tạo trực tiếp (`kubectl run` hoặc `kubectl apply -f pod.yaml`) là **naked pod** (pod trần) — không được quản lý bởi controller. Khi node bị lỗi hoặc Pod crash:
- Không tự restart
- Không tự tạo lại trên node khác
- Không hỗ trợ rolling update

**Luôn dùng Deployment, StatefulSet hoặc DaemonSet** để quản lý Pod.

---

## Pod Lifecycle — Vòng Đời Pod

### Các Giai Đoạn (Phase)

| Phase | Ý Nghĩa |
| ----- | ------- |
| `Pending` | Pod đã được tạo nhưng chưa được lên lịch chạy trên node nào, hoặc đang tải image |
| `Running` | Pod đã được gán vào node, ít nhất một container đang chạy |
| `Succeeded` | Tất cả container đã kết thúc thành công (exit code 0), không restart |
| `Failed` | Tất cả container đã kết thúc, ít nhất một container thất bại (exit code ≠ 0) |
| `Unknown` | Không thể xác định trạng thái Pod, thường do mất kết nối với node |

### Trạng Thái Container (Container State)

Trong mỗi phase, mỗi container có thể ở một trong ba trạng thái:

| Trạng Thái | Ý Nghĩa |
| ---------- | ------- |
| `Waiting` | Container đang chờ khởi động (tải image, chờ init container) |
| `Running` | Container đang thực thi |
| `Terminated` | Container đã kết thúc (hoàn thành hoặc thất bại) |

### Restart Policy — Chính Sách Khởi Động Lại

`restartPolicy` áp dụng cho tất cả container trong Pod:

| Giá Trị | Hành Vi |
| ------- | ------- |
| `Always` (mặc định) | Luôn restart khi container dừng — dùng cho Deployment |
| `OnFailure` | Chỉ restart khi container thoát với lỗi — dùng cho Job |
| `Never` | Không bao giờ restart — dùng cho tác vụ chạy một lần |

### Luồng Lifecycle Chi Tiết

```
kubectl apply → API Server lưu vào etcd
                    ↓
              Scheduler chọn node
                    ↓
              kubelet trên node nhận Pod spec
                    ↓
              Init Containers chạy (tuần tự)
                    ↓
              Containers chính khởi động
                    ↓
              postStart hook (nếu có)
                    ↓
              Startup Probe → Liveness Probe + Readiness Probe chạy
                    ↓
              Pod ở trạng thái Running + Ready
                    ↓  (khi xoá Pod)
              preStop hook chạy
                    ↓
              SIGTERM gửi đến container
                    ↓
              terminationGracePeriodSeconds (chờ graceful shutdown)
                    ↓
              SIGKILL nếu vẫn chưa dừng
```

### Trạng Thái Lỗi Thường Gặp

| Trạng Thái | Nguyên Nhân | Cách Debug |
| ---------- | ----------- | ---------- |
| `CrashLoopBackOff` | Container liên tục crash và restart | `kubectl logs --previous` để xem log lần chạy trước |
| `ImagePullBackOff` | Không tải được image (sai tên, thiếu credential) | `kubectl describe pod` xem Events |
| `Pending` | Không có node phù hợp (thiếu tài nguyên, taint) | `kubectl describe pod` xem `Reason` |
| `OOMKilled` | Container vượt memory limit | Tăng `resources.limits.memory` |
| `Error` | Container thoát với exit code ≠ 0 | `kubectl logs` xem lỗi ứng dụng |

---

## Multi-Container Pod — Pod Nhiều Container

Chỉ đặt nhiều container vào cùng một Pod khi chúng **cần chia sẻ tài nguyên** (network, volume) và **vòng đời gắn chặt** với nhau.

### Ví Dụ: Web Server + Log Shipper

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logging
spec:
  volumes:
    - name: shared-logs
      emptyDir: {}                    # emptyDir: volume tạm thời, tồn tại theo vòng đời Pod

  containers:
    - name: web-app
      image: nginx:1.25
      ports:
        - containerPort: 80
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx   # nginx ghi log vào đây

    - name: log-shipper
      image: fluent/fluent-bit:2.1
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx   # fluent-bit đọc log từ đây
          readOnly: true
```

**Luồng dữ liệu:**
```
nginx → ghi log → /var/log/nginx (shared volume) → fluent-bit đọc → gửi đến Elasticsearch
```

---

## Init Container — Container Khởi Tạo

**Init Container** là container đặc biệt chạy **trước** các container chính trong Pod. Tất cả init container phải hoàn thành thành công mới đến lượt container chính.

### Đặc Điểm

- Chạy **tuần tự** (không chạy song song)
- Nếu một init container thất bại, Pod không tiến đến bước tiếp theo
- Được khai báo trong `spec.initContainers`
- Có thể dùng image và quyền khác với container chính

### Trường Hợp Sử Dụng

| Tình Huống | Init Container Làm Gì |
| ---------- | -------------------- |
| Chờ database sẵn sàng | `nc -z db-service 5432` hoặc chạy `pg_isready` |
| Chạy database migration | Chạy `flyway migrate` hoặc `alembic upgrade head` |
| Clone config từ Git | `git clone` config repository vào shared volume |
| Tạo cấu trúc thư mục | `mkdir -p /data/uploads && chmod 777 /data/uploads` |
| Kiểm tra điều kiện tiên quyết | Xác nhận Vault accessible, certificate đã tồn tại |

### Ví Dụ: Chờ Database Sẵn Sàng

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-server
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          until nc -z postgres-service 5432; do
            echo "Đang chờ PostgreSQL..."
            sleep 2
          done
          echo "PostgreSQL đã sẵn sàng!"

    - name: run-migration
      image: my-app:latest
      command: ["python", "manage.py", "migrate"]
      env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url

  containers:
    - name: api
      image: my-app:latest
      ports:
        - containerPort: 8000
```

---

## Sidecar Container — Container Phụ Trợ

**Sidecar** là container chạy **song song** với container chính, cung cấp chức năng bổ trợ mà không thay đổi code của container chính.

### Trường Hợp Sử Dụng Phổ Biến

| Sidecar | Chức Năng |
| ------- | --------- |
| **Log Shipper** (Fluentd, Filebeat) | Đọc log từ shared volume, gửi đến ELK Stack hoặc Loki |
| **Proxy** (Envoy, Istio proxy) | Quản lý lưu lượng vào/ra, TLS termination, circuit breaker |
| **Config Syncer** | Đồng bộ config từ Consul hoặc etcd vào file |
| **Metrics Exporter** | Export metric sang định dạng Prometheus |
| **Vault Agent** | Đọc secret từ HashiCorp Vault, ghi vào shared volume |

### Ví Dụ: Vault Agent Sidecar

```yaml
containers:
  - name: app
    image: my-app:latest
    volumeMounts:
      - name: secrets
        mountPath: /vault/secrets   # đọc secret từ đây

  - name: vault-agent
    image: hashicorp/vault:1.15
    args:
      - agent
      - -config=/vault/config/agent-config.hcl
    volumeMounts:
      - name: secrets
        mountPath: /vault/secrets   # ghi secret vào đây
      - name: vault-config
        mountPath: /vault/config

volumes:
  - name: secrets
    emptyDir:
      medium: Memory               # lưu trong RAM, không ghi ra disk
  - name: vault-config
    configMap:
      name: vault-agent-config
```

---

## Resource Request và Limit

Mỗi container nên khai báo tài nguyên để scheduler (bộ lập lịch) xếp Pod vào đúng node và tránh tranh chấp tài nguyên.

### Request vs Limit

| Khái Niệm | Ý Nghĩa | Tác Động |
| --------- | ------- | -------- |
| `requests` (yêu cầu) | Lượng tài nguyên tối thiểu cần thiết | Scheduler dùng để quyết định node nào có đủ tài nguyên |
| `limits` (giới hạn) | Lượng tài nguyên tối đa được dùng | CPU bị throttle; Memory vượt giới hạn → container bị OOMKilled |

### QoS Class — Lớp Đảm Bảo Chất Lượng

Kubernetes tự động gán lớp QoS cho Pod dựa trên cấu hình resource:

| QoS Class | Điều Kiện | Ưu Tiên Khi Node Thiếu Tài Nguyên |
| --------- | --------- | --------------------------------- |
| `Guaranteed` (đảm bảo) | `requests == limits` cho CPU và memory | Ưu tiên cao nhất, bị xoá cuối cùng |
| `Burstable` (co giãn) | `requests < limits` | Ưu tiên trung bình |
| `BestEffort` (cố gắng hết sức) | Không khai báo requests và limits | Bị xoá đầu tiên khi thiếu tài nguyên |

```yaml
resources:
  requests:
    memory: "256Mi"    # 256 Mebibyte
    cpu: "250m"        # 250 millicores = 0.25 CPU core
  limits:
    memory: "512Mi"
    cpu: "500m"
```

---

## Pod Manifest Đầy Đủ

Ví dụ manifest Pod đầy đủ với các tính năng thường dùng trong production:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: production-app
  namespace: default
  labels:
    app: production-app
    version: "1.0"
  annotations:
    deployment.kubernetes.io/revision: "3"

spec:
  # ─── Chính sách restart ───────────────────────────────
  restartPolicy: Always

  # ─── Thời gian chờ graceful shutdown (giây) ──────────
  terminationGracePeriodSeconds: 30

  # ─── Service Account ─────────────────────────────────
  serviceAccountName: production-app-sa

  # ─── Init Containers ─────────────────────────────────
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command: ["sh", "-c", "until nc -z db-service 5432; do sleep 1; done"]

  # ─── Containers chính ────────────────────────────────
  containers:
    - name: app
      image: my-app:1.2.3
      imagePullPolicy: IfNotPresent   # Không tải lại nếu image đã có trên node

      ports:
        - name: http
          containerPort: 8080
          protocol: TCP

      # Biến môi trường
      env:
        - name: APP_ENV
          value: production
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: db-password

      # Tài nguyên
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"
        limits:
          memory: "512Mi"
          cpu: "500m"

      # Health checks (xem health-probes.md để biết chi tiết)
      startupProbe:
        httpGet:
          path: /health
          port: 8080
        failureThreshold: 30
        periodSeconds: 10

      livenessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 15

      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10

      # Mount volume
      volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
        - name: tmp-dir
          mountPath: /tmp

      # Lifecycle hooks
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]   # chờ load balancer drain connection

  # ─── Volumes ─────────────────────────────────────────
  volumes:
    - name: config-volume
      configMap:
        name: app-config
    - name: tmp-dir
      emptyDir: {}

  # ─── Lập lịch ────────────────────────────────────────
  affinity:
    podAntiAffinity:                 # Phân tán Pod trên nhiều node
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values: ["production-app"]
            topologyKey: kubernetes.io/hostname

  # Security context ở cấp Pod
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
```

---

## Câu Hỏi Phỏng Vấn

### Q: Sự khác biệt giữa `requests` và `limits` trong resource?

**`requests`** là lượng tài nguyên Pod yêu cầu được đảm bảo — scheduler dùng con số này để quyết định Pod có vừa trên node không. **`limits`** là mức trần tối đa Pod được phép dùng — nếu vượt CPU thì bị throttle (giảm tốc), vượt memory thì bị `OOMKilled`. Không khai báo `limits` là rủi ro vì Pod có thể chiếm toàn bộ tài nguyên của node.

### Q: emptyDir và hostPath volume khác nhau thế nào?

| Loại | Vòng Đời | Vị Trí Lưu | Rủi Ro |
| ---- | --------- | ----------- | ------ |
| `emptyDir` | Theo Pod — xoá khi Pod xoá | RAM hoặc disk trên node | Không có dữ liệu sau khi Pod xoá |
| `hostPath` | Theo Node — tồn tại sau khi Pod xoá | Thư mục cụ thể trên node | Gắn chặt với node, bảo mật thấp |

### Q: Pod ở `Pending` do nguyên nhân gì?

1. Không có node đủ tài nguyên (`Insufficient cpu/memory`)
2. Node có taint mà Pod không có toleration phù hợp
3. `nodeSelector` hoặc `nodeAffinity` không khớp với bất kỳ node nào
4. Đang chờ PersistentVolumeClaim được bind
5. Image đang tải về (trạng thái `ContainerCreating`)

### Q: `terminationGracePeriodSeconds` hoạt động thế nào?

Khi Pod bị xoá:
1. Kubernetes gửi `SIGTERM` đến container
2. Đợi tối đa `terminationGracePeriodSeconds` giây (mặc định 30 giây)
3. Nếu container chưa dừng sau thời gian đó, Kubernetes gửi `SIGKILL`

`preStop` hook chạy trong cùng khoảng thời gian này. Tổng thời gian = `preStop` + thời gian app shutdown ≤ `terminationGracePeriodSeconds`.

---

**Xem Tiếp:** [deployment.md](./deployment.md) — Rolling Update, Rollback và Deployment Strategy
