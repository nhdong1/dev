# Resource Management — Quản Lý Tài Nguyên

> Hướng dẫn chi tiết về quản lý tài nguyên trong Kubernetes: Resource Request và Limit (Yêu Cầu và Giới Hạn Tài Nguyên), QoS Classes (Lớp Chất Lượng Dịch Vụ), ResourceQuota (Hạn Mức Tài Nguyên Namespace), LimitRange (Phạm Vi Giới Hạn), và cách xử lý các vấn đề OOMKilled, CPU throttle.

## Mục Lục

1. [Request và Limit Là Gì?](#request-và-limit-là-gì)
2. [CPU Request và Limit](#cpu-request-và-limit)
3. [Memory Request và Limit](#memory-request-và-limit)
4. [QoS Classes — Lớp Chất Lượng Dịch Vụ](#qos-classes--lớp-chất-lượng-dịch-vụ)
5. [ResourceQuota — Hạn Mức Namespace](#resourcequota--hạn-mức-namespace)
6. [LimitRange — Giá Trị Mặc Định](#limitrange--giá-trị-mặc-định)
7. [Patterns Thực Chiến](#patterns-thực-chiến)
8. [Debug: OOMKilled và CPU Throttle](#debug-oomkilled-và-cpu-throttle)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Request và Limit Là Gì?

Resource Request và Limit là hai giá trị cấu hình trên mỗi container, kiểm soát cách Kubernetes và kernel phân bổ tài nguyên:

```
┌────────────────────────────────────────────────────────────┐
│                      Container                             │
│                                                            │
│  resources:                                                │
│    requests:    ─── Scheduler dùng để tìm node phù hợp    │
│      cpu: 250m     Kubelet đảm bảo container có ít nhất   │
│      memory: 256Mi  lượng này (cgroup guarantee)           │
│                                                            │
│    limits:      ─── Hard limit enforcement bởi kernel     │
│      cpu: 500m     CPU: throttle nếu vượt                  │
│      memory: 512Mi Memory: OOMKill nếu vượt               │
└────────────────────────────────────────────────────────────┘
```

**Tóm tắt:**
- `requests` — **scheduling và guarantee**: Scheduler chỉ đặt Pod lên node có đủ tài nguyên khả dụng (`allocatable - sum(all requests on node`). Kernel đảm bảo container luôn nhận được ít nhất lượng này.
- `limits` — **hard cap enforcement**: Kernel enforce giới hạn cứng. CPU bị throttle (giảm tốc), memory bị OOMKill (chết ngay).

---

## CPU Request và Limit

### Đơn Vị CPU

```
1 CPU = 1 core = 1000m (millicores — milli-CPU)

Ví dụ:
  cpu: "0.5"   = 500m = nửa core
  cpu: "1"     = 1000m = 1 core
  cpu: "250m"  = 1/4 core
  cpu: "2.5"   = 2500m = 2.5 core
```

### CPU Throttle — Cơ Chế Giới Hạn CPU

CPU limit được enforce bởi **Linux CFS (Completely Fair Scheduler — Bộ Lập Lịch Công Bằng Hoàn Toàn)** thông qua cgroup:

```
Mỗi 100ms period (CFS quota period):

cpu.cfs_quota_us = cpu_limit × 100ms
cpu.cfs_period_us = 100ms

Ví dụ: cpu limit = 500m
  → Container được dùng 50ms CPU trong mỗi 100ms
  → Nếu dùng hết 50ms trong 50ms đầu → throttle 50ms còn lại
  → Application thấy "CPU chậm" dù node còn nhiều CPU trống
```

**CPU throttle không giết process** — ứng dụng tiếp tục chạy nhưng chậm hơn. Đây là vấn đề nghiêm trọng cho latency-sensitive service.

```bash
# Kiểm tra CPU throttle
kubectl exec -it pod-name -- cat /sys/fs/cgroup/cpu/cpu.stat
# throttled_time: 50000000000   ← 50 giây bị throttle — rất nhiều!
# nr_throttled: 1234            ← số lần bị throttle
```

### Lý Do Không Nên Đặt CPU Limit Thấp Hơn Thực Tế

```
Ứng dụng cần 300ms CPU trong 1 request:
  CPU limit = 200m → quota = 20ms/100ms period
  → Request kéo dài: 300ms / 20ms × 100ms = 1500ms (1.5 giây!)
  Thực tế không có throttle: 300ms

CPU throttle có thể tăng latency lên 5–10x ngay cả khi node còn nhiều CPU trống.
```

---

## Memory Request và Limit

### Đơn Vị Memory

```
Mi = Mebibytes = 1024 × 1024 bytes  (binary)
M  = Megabytes = 1000 × 1000 bytes  (decimal)
Gi = Gibibytes = 1024 × 1024 × 1024 bytes
G  = Gigabytes = 1000 × 1000 × 1000 bytes

Thường dùng Mi và Gi trong Kubernetes.
```

### OOMKill — Cơ Chế Khi Vượt Memory Limit

Memory limit được enforce khác CPU limit — **không throttle, mà kill ngay**:

```
Container dùng > memory limit
        │
        ▼
Linux OOM Killer (Out-Of-Memory Killer) kích hoạt
        │
        ▼
Process trong container bị SIGKILL
        │
        ▼
Container restart (nếu restartPolicy: Always/OnFailure)
        │
        ▼
kubectl logs thấy: OOMKilled / Exit Code 137
```

### Lý Do Nên Đặt Memory Limit Gần Request

```
memory request = 256Mi, memory limit = 2Gi

Tình huống:
- 10 Pod trên node, mỗi Pod request 256Mi → Scheduler thấy node cần 2.5Gi
- Thực tế: mỗi Pod burst lên 1Gi → node cần 10Gi → Memory pressure!
- Node bị memory pressure → kubelet evict Pod BestEffort trước, rồi Burstable
- Hệ thống bất ổn, khó dự đoán

Khuyến nghị: memory limit ≤ 2× memory request
```

---

## QoS Classes — Lớp Chất Lượng Dịch Vụ

Kubernetes tự động phân Pod vào 3 QoS class dựa trên cách khai báo request/limit:

### Guaranteed — Đảm Bảo

**Điều kiện:** Mọi container đều có requests == limits (cả CPU lẫn memory)

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"     # == request
    memory: "512Mi" # == request
```

**Hành vi:** Ít bị evict nhất khi node gặp resource pressure. OOM killer ưu tiên kill Pod có QoS thấp hơn trước.

### Burstable — Có Thể Bùng Nổ

**Điều kiện:** Ít nhất một container có request hoặc limit (nhưng không phải Guaranteed)

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"     # != request → Burstable
    memory: "512Mi"
```

**Hành vi:** Bị evict sau BestEffort khi node pressure. Có thể burst lên limit khi node có tài nguyên trống.

### BestEffort — Nỗ Lực Tối Đa (Không Đảm Bảo)

**Điều kiện:** Không khai báo request lẫn limit

```yaml
resources: {}  # không có requests, không có limits → BestEffort
```

**Hành vi:** Bị evict đầu tiên khi node gặp pressure. Có thể dùng tài nguyên tuỳ thích khi node rảnh. Không phù hợp production.

### So Sánh QoS

```
Node Memory Pressure (áp lực bộ nhớ):
Priority evict (thấp trước):
    1. BestEffort
    2. Burstable (theo mức vượt request)
    3. Guaranteed (cuối cùng)

Khi nào bị evict:
    BestEffort:  sớm nhất, ngay khi node bắt đầu pressure
    Burstable:   khi memory usage vượt quá request của chính Pod đó
    Guaranteed:  chỉ khi cả cluster thực sự hết memory
```

---

## ResourceQuota — Hạn Mức Namespace

**ResourceQuota** đặt giới hạn tổng tài nguyên mà tất cả Pod trong một **namespace** được dùng. Ngăn một team/app chiếm hết tài nguyên cluster.

### Quota Tài Nguyên CPU và Memory

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Tổng resource requests của tất cả Pod trong namespace
    requests.cpu: "20"            # tổng CPU request tối đa 20 core
    requests.memory: 40Gi         # tổng memory request tối đa 40Gi

    # Tổng resource limits của tất cả Pod trong namespace
    limits.cpu: "40"              # tổng CPU limit tối đa 40 core
    limits.memory: 80Gi           # tổng memory limit tối đa 80Gi

    # Giới hạn số lượng object
    pods: "100"                   # tối đa 100 Pod
    services: "20"                # tối đa 20 Service
    secrets: "50"                 # tối đa 50 Secret
    configmaps: "50"              # tối đa 50 ConfigMap
    persistentvolumeclaims: "20"  # tối đa 20 PVC
```

### Quota Theo Object Count và Storage

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: production
spec:
  hard:
    requests.storage: 500Gi       # tổng PVC capacity tối đa
    gold.storageclass.storage.k8s.io/requests.storage: 100Gi  # quota per StorageClass
    count/deployments.apps: "20"  # tối đa 20 Deployment
    count/jobs.batch: "10"        # tối đa 10 Job đang chạy
```

### Kiểm Tra Quota Hiện Tại

```bash
# Xem quota và current usage
kubectl describe resourcequota production-quota -n production

# Output:
# Name:            production-quota
# Namespace:       production
# Resource         Used    Hard
# ────────         ────    ────
# limits.cpu       12      40
# limits.memory    24Gi    80Gi
# pods             45      100
# requests.cpu     6       20
# requests.memory  12Gi    40Gi

# Alert: nếu Used/Hard > 80% → cần tăng quota hoặc tối ưu resource
```

### Khi Quota Bị Vượt

```bash
# Pod tạo mới bị từ chối
kubectl apply -f my-deployment.yaml

# Error:
# Error from server (Forbidden): error when creating "my-deployment.yaml":
# pods "my-pod" is forbidden: exceeded quota: production-quota,
# requested: requests.cpu=500m, used: requests.cpu=19800m, limited: requests.cpu=20
```

---

## LimitRange — Giá Trị Mặc Định

**LimitRange** đặt **default request/limit** và kiểm soát giá trị min/max cho container, Pod, hoặc PVC trong một namespace.

### LimitRange Cho Container

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:                  # limit mặc định nếu container không khai báo
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:           # request mặc định nếu container không khai báo
        cpu: "100m"
        memory: "128Mi"
      max:                      # container không được có limit cao hơn
        cpu: "4"
        memory: "8Gi"
      min:                      # container không được có request thấp hơn
        cpu: "50m"
        memory: "64Mi"
```

**Lợi ích LimitRange:**
1. **Tự động điền default** cho Pod không khai báo resource (tránh BestEffort QoS)
2. **Ngăn khai báo quá thấp** gây OOMKill thường xuyên
3. **Ngăn khai báo quá cao** một Pod chiếm hết quota

### LimitRange Cho Pod

```yaml
spec:
  limits:
    - type: Pod
      max:
        cpu: "8"                # tổng CPU limit của tất cả container trong 1 Pod
        memory: "16Gi"          # tổng memory limit của tất cả container trong 1 Pod
```

### LimitRange Cho PersistentVolumeClaim

```yaml
spec:
  limits:
    - type: PersistentVolumeClaim
      max:
        storage: 100Gi          # PVC không được request quá 100Gi
      min:
        storage: 1Gi            # PVC phải ít nhất 1Gi
```

---

## Patterns Thực Chiến

### Pattern 1: Production Namespace — Strict Control

```yaml
---
# Quota nghiêm ngặt cho production
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "50"
    requests.memory: 100Gi
    limits.cpu: "100"
    limits.memory: 200Gi
    pods: "200"

---
# LimitRange đặt default hợp lý
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "8"
        memory: "16Gi"
      min:
        cpu: "10m"
        memory: "16Mi"
```

### Pattern 2: Development Namespace — Loose Control

```yaml
# Dev namespace ít nghiêm ngặt hơn
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
    - type: Container
      default:
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:
        cpu: "50m"
        memory: "64Mi"
      max:
        cpu: "2"
        memory: "4Gi"
```

### Pattern 3: Khai Báo Resource Đúng Cách

```yaml
containers:
  - name: web-api
    image: web-api:1.0
    resources:
      requests:
        cpu: "250m"             # baseline để scheduler tìm node
        memory: "256Mi"         # đặt bằng peak thực tế (tránh OOMKill)
      limits:
        cpu: "1000m"            # 4x request — cho phép burst (CPU không OOMKill)
        memory: "512Mi"         # 2x request — buffer an toàn (memory OOMKill!)

  - name: sidecar-proxy
    image: envoy:1.0
    resources:
      requests:
        cpu: "50m"              # sidecar nhỏ
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "128Mi"
```

**Quy tắc ngón tay cái:**
- Memory limit: 1.5x–2x memory request (memory OOMKill là fatal — cần buffer an toàn)
- CPU limit: 2x–4x CPU request (CPU throttle không fatal — cho phép burst thoải mái hơn)

---

## Debug: OOMKilled và CPU Throttle

### Phát Hiện OOMKilled

```bash
# Tìm Pod bị OOMKilled
kubectl get pods -A | grep -v Running

# Kiểm tra exit code
kubectl describe pod my-pod -n production
# Containers:
#   web-api:
#     State: Waiting
#       Reason: CrashLoopBackOff
#     Last State: Terminated
#       Reason: OOMKilled          ← đây rồi!
#       Exit Code: 137

# Xem memory usage trước khi chết
kubectl top pod my-pod -n production --containers

# Giải pháp: tăng memory limit
# Hoặc dùng VPA mode Off để lấy recommendation
```

### Phát Hiện CPU Throttle

```bash
# Xem container metrics chi tiết (cần kubectl exec)
kubectl exec -it my-pod -n production -- cat /sys/fs/cgroup/cpu/cpu.stat
# nr_periods: 100000
# nr_throttled: 15000        ← 15% số period bị throttle!
# throttled_time: 5000000000 ← 5 giây tổng thời gian throttle

# Hoặc dùng Prometheus query để tìm container bị throttle cao
# container_cpu_cfs_throttled_seconds_total / container_cpu_cfs_periods_total > 0.25
# → container bị throttle hơn 25% thời gian
```

### Điều Chỉnh Resource Đúng Sau Khi Debug

```bash
# Xem VPA recommendation (nếu có VPA)
kubectl describe vpa my-deployment-vpa -n production | grep -A 20 "Recommendation:"

# Xem usage thực tế 7 ngày qua (nếu có Prometheus)
# rate(container_cpu_usage_seconds_total{pod="my-pod"}[5m]) * 1000
# → peak CPU usage in millicores

# max_over_time(container_memory_working_set_bytes{pod="my-pod"}[7d]) / 1024 / 1024
# → peak memory usage in MiB
```

---

## Câu Hỏi Phỏng Vấn

**Sự khác nhau giữa `resources.requests` và `resources.limits`?**

> `requests` là **cam kết tối thiểu** — Scheduler dùng để tìm node có đủ tài nguyên khả dụng, và kernel đảm bảo container luôn nhận được ít nhất lượng này. `limits` là **trần tối đa** — kernel enforce cứng: CPU vượt limit thì throttle (giảm tốc), memory vượt limit thì OOMKill (kill ngay). Container có thể dùng nhiều hơn `requests` (burst) khi node còn tài nguyên trống, nhưng không bao giờ vượt `limits`. Nếu không đặt `limits`: CPU không giới hạn (có thể hưởng lợi từ node rảnh), memory không giới hạn (nguy hiểm — có thể chiếm hết node).

**3 QoS class trong Kubernetes là gì? Ảnh hưởng gì đến behavior cluster?**

> **Guaranteed:** requests == limits cho mọi container — ít bị evict nhất, kernel ưu tiên giữ lại khi node pressure. **Burstable:** có request/limit nhưng không Guaranteed — bị evict sau BestEffort, mức ưu tiên dựa trên mức vượt request. **BestEffort:** không có requests lẫn limits — bị evict đầu tiên, không có guarantee về tài nguyên. Ảnh hưởng thực tế: (1) Kubelet evict Pod BestEffort trước khi evict Burstable khi node memory pressure; (2) OOM killer ưu tiên kill BestEffort trước; (3) Cluster Autoscaler dựa vào `requests` (không phải usage thực) để tính node utilization — Pod BestEffort không có requests khiến CA tưởng node nhàn rỗi.

**ResourceQuota và LimitRange khác nhau thế nào?**

> **ResourceQuota** kiểm soát **tổng tài nguyên cấp độ namespace** — giới hạn tổng CPU/memory của tất cả Pod cộng lại, số lượng object (pods, services, secrets). Áp dụng cho cả namespace. **LimitRange** kiểm soát **từng container/pod/pvc** — đặt default request/limit khi không khai báo, giới hạn min/max cho từng container. Áp dụng cho từng object mới tạo. Dùng kết hợp: LimitRange đảm bảo mọi Pod có request (tránh BestEffort), ResourceQuota đảm bảo namespace không vượt tổng hạn mức. Nếu chỉ có ResourceQuota mà không có LimitRange, Pod không khai báo request sẽ bị Admission Controller từ chối tạo (vì ResourceQuota yêu cầu request phải khai báo tường minh khi quota được set).
