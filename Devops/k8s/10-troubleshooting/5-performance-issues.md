# Performance Issues — Xử Lý Vấn Đề Hiệu Năng Kubernetes

> Hướng dẫn chẩn đoán và khắc phục các vấn đề hiệu năng: CPU throttling, high latency, memory pressure, resource starvation, và các vấn đề hiệu năng ứng dụng trên Kubernetes.

## Mục Lục

1. [Tổng Quan Về Resource Model Kubernetes](#tổng-quan-resource-model)
2. [CPU Throttling (Bóp Nghẹt CPU)](#cpu-throttling)
3. [Memory Pressure và OOM](#memory-pressure-và-oom)
4. [High Latency (Độ Trễ Cao)](#high-latency)
5. [Resource Starvation (Đói Tài Nguyên)](#resource-starvation)
6. [Node-Level Performance Issues](#node-level-performance-issues)
7. [Network Performance Issues](#network-performance-issues)
8. [Công Cụ Phân Tích Hiệu Năng](#công-cụ-phân-tích-hiệu-năng)

---

## Tổng Quan Resource Model

### Request vs Limit — Sự Khác Biệt Quan Trọng

```
┌────────────────────────────────────────────────────────────────┐
│  requests — Lượng tài nguyên được đảm bảo (reserved)          │
│  limits   — Mức tối đa container được phép dùng               │
│                                                                │
│  CPU:    vượt limit → bị throttle (bóp), không bị kill        │
│  Memory: vượt limit → bị OOMKilled (bị kill ngay lập tức)     │
└────────────────────────────────────────────────────────────────┘
```

### QoS Class (Lớp Chất Lượng Dịch Vụ)

```yaml
# Guaranteed — request == limit (bảo vệ tốt nhất, khó bị evict)
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"

# Burstable — có request nhưng request < limit (trung bình)
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
  limits:
    cpu: "1000m"
    memory: "512Mi"

# BestEffort — không có request lẫn limit (dễ bị evict nhất)
resources: {}
```

### Xem Resource Hiện Tại

```bash
# Xem resource đang dùng theo Pod
kubectl top pods -n <namespace>
kubectl top pods -n <namespace> --containers

# Xem resource đang dùng theo Node
kubectl top nodes

# Xem resource được allocate trên Node
kubectl describe node <name> | grep -A20 "Allocated resources:"
```

---

## CPU Throttling

### Định Nghĩa

CPU throttling xảy ra khi container **vượt quá CPU limit** — kernel Linux (CFS Scheduler — Completely Fair Scheduler) sẽ dừng container trong một khoảng thời gian rồi mới cho chạy tiếp. Kết quả: ứng dụng phản hồi chậm dù CPU usage trên dashboard trông vẫn bình thường.

### Phát Hiện CPU Throttling

#### Cách 1: Xem Metric Prometheus

```bash
# Metric quan trọng nhất
container_cpu_cfs_throttled_periods_total   # Số chu kỳ bị throttle
container_cpu_cfs_periods_total             # Tổng số chu kỳ

# Công thức tính throttle rate
rate(container_cpu_cfs_throttled_periods_total[5m])
/ rate(container_cpu_cfs_periods_total[5m]) * 100

# Alert khi throttle rate > 25%
```

#### Cách 2: Kiểm Tra Trực Tiếp Trên Node

```bash
# SSH vào node
# Tìm container ID
crictl ps | grep <container-name>

# Xem cgroup stats
cat /sys/fs/cgroup/cpu/kubepods/.../cpu.stat
# throttled_time N   ← Thời gian bị throttle (nanoseconds)
```

#### Cách 3: kubectl top

```bash
kubectl top pods -n <ns> --containers
# Nếu CPU usage liên tục gần bằng limit → có khả năng bị throttle
```

### Nguyên Nhân và Cách Xử Lý

#### 1. CPU Limit Quá Thấp

```bash
# Xem limit hiện tại
kubectl get pod <name> -o jsonpath='{.spec.containers[0].resources.limits.cpu}'

# Tăng limit
kubectl set resources deployment <name> \
  --limits=cpu=1000m --requests=cpu=250m -n <ns>
```

**Lưu ý về đơn vị CPU:**
- `1` hoặc `1000m` = 1 CPU core
- `500m` = 0.5 CPU core (milliCPU)
- `100m` = 0.1 CPU core

#### 2. Burst Load Không Dự Đoán Được

```bash
# Giải pháp: Dùng HPA để scale ra nhiều Pod hơn
kubectl autoscale deployment <name> \
  --min=2 --max=10 \
  --cpu-percent=70 \
  -n <ns>

# Giải pháp khác: Tăng request để Pod được schedule trên node ít tải hơn
```

#### 3. CPU Limit Cao Hơn CPU Request Quá Nhiều (Noisy Neighbor Problem)

Khi nhiều Pod cùng burst lên sử dụng nhiều CPU, các Pod share node sẽ tranh giành CPU và cùng bị throttle.

```bash
# Kiểm tra ratio request/limit
kubectl get pods -n <ns> -o json | jq '.items[].spec.containers[].resources'

# Best practice: limit không nên gấp quá 2-3 lần request cho production
resources:
  requests:
    cpu: "500m"
  limits:
    cpu: "1000m"   # Tối đa 2x request
```

### Ảnh Hưởng Của CPU Throttling Đến Java/JVM

Java GC (Garbage Collection — Thu Gom Rác) đặc biệt nhạy cảm với CPU throttling:
- GC cần CPU burst ngắn để chạy
- Nếu bị throttle trong GC → stop-the-world pause kéo dài
- Kết quả: latency spike đột ngột

```yaml
# Cho JVM workload — tăng request để đảm bảo GC có đủ CPU
resources:
  requests:
    cpu: "1000m"     # Đủ CPU cho GC
  limits:
    cpu: "2000m"
  limits:
    memory: "2Gi"
```

---

## Memory Pressure và OOM

### Phân Biệt Các Loại OOM

```
1. Container OOM (OOMKilled):
   - Container vượt quá memory limit của chính nó
   - K8s kill container đó, không ảnh hưởng Pod khác

2. Node OOM (Node Memory Pressure):
   - Node không còn memory → kubelet evict Pod
   - Thứ tự evict: BestEffort → Burstable → Guaranteed

3. JVM OOM (OutOfMemoryError):
   - Lỗi trong JVM, không nhất thiết vượt limit
   - Ứng dụng crash, container exit với code 1 (không phải 137)
```

### Phát Hiện Memory Vấn Đề

```bash
# Xem memory usage hiện tại
kubectl top pods -n <ns> --containers --sort-by=memory

# Xem memory limit vs usage
kubectl get pod <name> -o jsonpath='{.spec.containers[0].resources}'
kubectl top pod <name> --containers

# Xem lịch sử OOMKilled
kubectl get events -n <ns> | grep -i "OOM\|kill"
kubectl describe pod <name> | grep -A5 "Last State"

# Prometheus metric
container_memory_working_set_bytes    # Memory thực sự đang dùng
container_memory_rss                  # RSS memory (resident set size)
kube_pod_container_status_last_terminated_reason  # Lý do container bị tắt
```

### Thiết Lập Memory Budget Đúng

```bash
# Bước 1: Chạy load test và quan sát peak memory
kubectl top pod <name> --containers -n <ns>
# Quan sát trong 30 phút với traffic thực tế

# Bước 2: Set request = baseline usage, limit = peak * 1.2
# Ví dụ: baseline 200Mi, peak 400Mi
resources:
  requests:
    memory: "200Mi"
  limits:
    memory: "480Mi"   # 400Mi * 1.2
```

### Dùng VPA Để Tìm Request/Limit Phù Hợp

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
  namespace: default
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"    # Off = chỉ recommend, không tự apply
```

```bash
# Xem VPA recommendation
kubectl describe vpa my-app-vpa
# Sẽ thấy: Recommendation > Container > Target: cpu/memory
```

---

## High Latency

### Phân Tích Nguồn Gốc Latency

```
Latency cao có thể do:
│
├── Ứng dụng (application latency)
│   ├── GC pause (Java, Go, .NET)
│   ├── Database query chậm
│   ├── External API timeout
│   └── Thread pool exhausted (hết thread)
│
├── Kubernetes (platform latency)
│   ├── CPU throttling
│   ├── Node resource contention (tranh giành tài nguyên node)
│   ├── HPA chưa kịp scale
│   └── readinessProbe period quá dài
│
└── Mạng (network latency)
    ├── kube-proxy iptables rules chậm (nhiều Service)
    ├── CNI overhead
    ├── DNS lookup chậm
    └── Ingress controller bottleneck
```

### Debug Latency Trên Kubernetes

```bash
# Kiểm tra CPU throttling (nguyên nhân phổ biến nhất)
# Metric: rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m])

# Kiểm tra resource contention trên node
kubectl top nodes
kubectl describe node <name> | grep -A5 "Allocated resources"

# Xem pending Pod (HPA chưa kịp scale)
kubectl get pods -n <ns> | grep Pending

# Kiểm tra HPA status
kubectl get hpa -n <ns>
kubectl describe hpa <name> -n <ns>
# Xem: Current/Desired/Min/Max replicas, scaling conditions
```

### Phát Hiện Latency Spike (Đột Biến Latency)

```bash
# Dùng kubectl để xem log thời gian phản hồi
kubectl logs <pod> -n <ns> | grep -E "[0-9]+ms|[0-9]+s" | tail -50

# Dùng Prometheus + Grafana
# metric: http_request_duration_seconds_bucket (histogram)
# Tính P99 latency:
# histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

### Tối Ưu HPA Để Giảm Latency

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 3      # Tăng min để luôn có dự phòng
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60    # Scale sớm hơn (mặc định 80%)
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30   # Scale up nhanh hơn (mặc định 300s)
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300  # Scale down chậm (an toàn hơn)
```

---

## Resource Starvation

### Định Nghĩa

Resource Starvation xảy ra khi tổng resource request của tất cả Pod vượt quá tài nguyên thực tế trên cluster, hoặc một số Pod chiếm quá nhiều tài nguyên và làm các Pod khác bị thiếu.

### Phát Hiện Resource Starvation

```bash
# Xem overcommitment trên từng node
kubectl describe nodes | grep -A10 "Allocated resources"
# Lưu ý: CPU có thể overcommit an toàn, Memory thì không

# Tìm Pod không có resource request (BestEffort)
kubectl get pods -A -o json | jq '.items[] | select(.spec.containers[].resources.requests == null) | .metadata.name'

# Tìm Pod dùng nhiều hơn request (candidates gây starvation)
kubectl top pods -A --sort-by=cpu | head -20
```

### ResourceQuota — Giới Hạn Theo Namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    pods: "50"
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    persistentvolumeclaims: "20"
    requests.storage: "200Gi"
```

### LimitRange — Giá Trị Mặc Định Và Giới Hạn Per-Container

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:           # Limit mặc định nếu không khai báo
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:    # Request mặc định
        cpu: "100m"
        memory: "128Mi"
      max:               # Giới hạn tối đa
        cpu: "4"
        memory: "4Gi"
      min:               # Yêu cầu tối thiểu
        cpu: "50m"
        memory: "64Mi"
```

---

## Node-Level Performance Issues

### Noisy Neighbor Problem (Vấn Đề Hàng Xóm Ồn Ào)

Khi một Pod trên node dùng quá nhiều tài nguyên và ảnh hưởng đến các Pod khác trên cùng node.

```bash
# Tìm Pod đang dùng nhiều CPU/Memory nhất trên một node
kubectl top pods -A --sort-by=cpu | grep <node-name>

# Di chuyển Pod "noisy" sang node riêng bằng nodeAffinity
# Hoặc dùng Pod Priority và PriorityClass

# Kiểm tra node có đang ở dưới pressure không
kubectl describe node <name> | grep -A5 "Conditions"
```

### Node Kernel Vấn Đề

```bash
# Xem kernel logs trên node
dmesg | grep -E "oom|OOM|throttl|error" | tail -50

# Xem system load
uptime
# load average: 0.5, 0.7, 0.8 → 3 giá trị: 1 phút, 5 phút, 15 phút
# Load > số CPU core = system overloaded

# Xem I/O wait
iostat -x 1 5
# %iowait cao → disk I/O là bottleneck
```

### CPU Pinning (Ghim CPU) Với Guaranteed QoS

Với Pod có QoS Guaranteed và request là integer CPU, kubelet có thể pin CPU để tránh context switching:

```yaml
# Pod Guaranteed có integer CPU
resources:
  requests:
    cpu: "2"     # 2 cores nguyên, không phải 2000m
    memory: "1Gi"
  limits:
    cpu: "2"
    memory: "1Gi"
```

Cần bật CPU Manager policy trên kubelet:
```
--cpu-manager-policy=static
```

---

## Network Performance Issues

### Vấn Đề kube-proxy Với Nhiều Service

Khi cluster có hàng nghìn Service, iptables ruleset trở nên rất lớn, làm chậm network latency.

```bash
# Đếm số Service trong cluster
kubectl get svc -A | wc -l

# Nếu > 5000 Service → nên chuyển sang IPVS mode hoặc Cilium eBPF

# Xem mode kube-proxy
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
```

### DNS Performance

```bash
# Xem DNS query latency (nếu có Prometheus)
# metric: coredns_dns_request_duration_seconds

# Tăng cache TTL của CoreDNS
kubectl edit configmap coredns -n kube-system
# Sửa cache directive:
# cache 30 → cache 300

# Dùng dnsConfig để giảm lookup
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "1"    # Giảm từ 5 xuống 1 — ít retry DNS hơn
      - name: timeout
        value: "5"
```

### Pod Startup Latency (Độ Trễ Khởi Động Pod)

```bash
# Xem thời gian từ khi Pod được tạo đến khi Ready
kubectl get pod <name> -o jsonpath='{.status.startTime}'
kubectl describe pod <name> | grep -E "Start Time|Ready"

# Nguyên nhân thường gặp:
# 1. Image pull chậm → dùng image đã cache sẵn trên node
# 2. Init container chậm
# 3. Readiness probe initialDelay quá dài
# 4. Volume mount chậm (EBS attach có thể mất 20-30s)

# Tối ưu:
# - Dùng imagePullPolicy: IfNotPresent (không pull lại nếu đã có)
# - Giảm initialDelaySeconds của readiness probe
# - Pre-pull image trên node (DaemonSet để pull image sẵn)
```

---

## Công Cụ Phân Tích Hiệu Năng

### kubectl top — Xem Nhanh

```bash
kubectl top pods -n <ns> --sort-by=cpu
kubectl top pods -n <ns> --sort-by=memory
kubectl top nodes --sort-by=cpu
```

### Prometheus Queries (PromQL) Hữu Ích

```promql
# CPU throttle rate (%)
rate(container_cpu_cfs_throttled_periods_total{container!=""}[5m])
/ rate(container_cpu_cfs_periods_total{container!=""}[5m]) * 100

# Memory usage so với limit (%)
container_memory_working_set_bytes{container!=""}
/ container_spec_memory_limit_bytes{container!=""} * 100

# Pod restart rate
rate(kube_pod_container_status_restarts_total[1h])

# Node CPU usage (%)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Node memory usage (%)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

### Popeye — Quét Cluster

```bash
# Chạy Popeye để phát hiện vấn đề cấu hình
kubectl run popeye --image=derailed/popeye --rm -it \
  --serviceaccount=popeye \
  -- scan

# Kết quả sẽ highlight:
# - Pod không có resource request/limit
# - Pod không có readiness probe
# - Service không có endpoint
# - Deployment với chỉ 1 replica
```

### kube-score — Phân Tích Manifest

```bash
kube-score score my-deployment.yaml

# Sẽ phát hiện:
# - Thiếu resource request/limit
# - Thiếu liveness/readiness probe
# - Không có PodDisruptionBudget
# - SecurityContext chưa được set
```

### Profiling Ứng Dụng

```bash
# Go pprof
kubectl port-forward pod/<go-pod> 6060:6060 -n <ns>
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Java heap dump
kubectl exec <java-pod> -n <ns> -- jmap -dump:format=b,file=/tmp/heap.hprof 1
kubectl cp <ns>/<java-pod>:/tmp/heap.hprof ./heap.hprof

# Node.js
kubectl exec <node-pod> -n <ns> -- node --prof /app/index.js
```

---

## Tóm Tắt — Performance Troubleshooting

| Triệu Chứng                   | Metric Kiểm Tra                          | Cách Xử Lý                              |
| ----------------------------- | ---------------------------------------- | ---------------------------------------- |
| Response chậm, CPU OK         | CPU throttle rate                        | Tăng CPU limit                           |
| OOMKilled liên tục            | `container_memory_working_set_bytes`     | Tăng memory limit, fix memory leak       |
| Latency spike định kỳ         | GC pause metrics                         | Tăng CPU request cho JVM, tune GC        |
| Latency tăng khi load cao     | HPA status, pending pods                 | Giảm targetUtilization HPA, tăng minReplicas |
| Pod bị evict                  | `kubectl describe node` → Conditions    | Thêm memory limit, tăng node capacity    |
| Toàn cluster chậm             | `kubectl top nodes`                      | Kiểm tra noisy neighbor, thêm node       |

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
