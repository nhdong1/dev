# kube-state-metrics và metrics-server

> Hướng dẫn chi tiết về 3 nguồn metric trong Kubernetes: cAdvisor (metric container resource), metrics-server (metric real-time cho HPA và kubectl top), và kube-state-metrics (metric trạng thái Kubernetes object). Bao gồm danh sách metric quan trọng cho từng object, custom resource metric với custom metrics API, và chiến lược monitor hiệu quả.

## Mục Lục

1. [Ba Nguồn Metric trong Kubernetes](#ba-nguồn-metric-trong-kubernetes)
2. [cAdvisor — Container Advisor](#cadvisor--container-advisor)
3. [metrics-server — Real-time Resource Metric](#metrics-server--real-time-resource-metric)
4. [kube-state-metrics — Kubernetes Object State](#kube-state-metrics--kubernetes-object-state)
5. [Metric Quan Trọng Theo Object](#metric-quan-trọng-theo-object)
6. [Custom Metrics API](#custom-metrics-api)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)
8. [Checklist Production](#checklist-production)

---

## Ba Nguồn Metric trong Kubernetes

```
┌──────────────────────────────────────────────────────────────────┐
│              3 NGUỒN METRIC TRONG KUBERNETES                     │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    KUBERNETES NODE                      │    │
│  │                                                         │    │
│  │  ┌─────────────┐   ┌───────────────────────────────┐   │    │
│  │  │   kubelet   │   │         CONTAINERS            │   │    │
│  │  │             │   │  ┌──────────┐  ┌──────────┐   │   │    │
│  │  │  ┌────────┐ │   │  │Container │  │Container │   │   │    │
│  │  │  │cAdvisor│ │   │  │    A     │  │    B     │   │   │    │
│  │  │  │(built- │◄├───┤  └──────────┘  └──────────┘   │   │    │
│  │  │  │in)     │ │   └───────────────────────────────┘   │    │
│  │  │  └────┬───┘ │                                        │    │
│  │  └───────┼─────┘                                        │    │
│  └──────────┼───────────────────────────────────────────────    │
│             │ /metrics/cadvisor                                  │
│             ▼                                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              metrics-server                              │   │
│  │  ├── scrape kubelet /metrics/resource                    │   │
│  │  ├── expose Metrics API (short-term, no history)         │   │
│  │  └── HPA + kubectl top dùng metrics-server              │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              kube-state-metrics (KSM)                    │   │
│  │  ├── query kube-apiserver về trạng thái object           │   │
│  │  ├── expose /metrics (Prometheus scrape)                 │   │
│  │  └── Đo: "Deployment có bao nhiêu replica?"              │   │
│  │         "Pod đang ở phase nào?"                          │   │
│  │         "Node có condition Ready không?"                 │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Prometheus                                  │   │
│  │  ├── scrape kubelet /metrics/cadvisor → container metric │   │
│  │  ├── scrape kube-state-metrics → object state metric     │   │
│  │  └── store history, PromQL, alert                        │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### So Sánh 3 Nguồn Metric

| Tiêu Chí | cAdvisor | metrics-server | kube-state-metrics |
| -------- | -------- | -------------- | ------------------ |
| **Đo gì** | CPU, memory, network, disk của container/pod | CPU, memory real-time của pod và node | Trạng thái Kubernetes object (replicas, phase, condition) |
| **Nguồn dữ liệu** | Kernel cgroup, container runtime stats | kubelet /metrics/resource | kube-apiserver (watch/list) |
| **Lưu lịch sử** | Không (Prometheus scrape rồi lưu) | Không (chỉ giữ giá trị hiện tại) | Không (Prometheus scrape rồi lưu) |
| **Ai dùng** | Prometheus, HPA (qua adapter) | HPA, VPA, kubectl top | Prometheus, Grafana dashboard |
| **API** | kubelet /metrics/cadvisor (Prometheus format) | Metrics API (/apis/metrics.k8s.io) | /metrics (Prometheus format) |
| **Cài đặt** | Tích hợp sẵn trong kubelet | Cần cài riêng | Cần cài riêng (có trong kube-prometheus-stack) |

---

## cAdvisor — Container Advisor

**cAdvisor (Container Advisor — Tư Vấn Container)** được Google phát triển và tích hợp sẵn vào kubelet. cAdvisor thu thập metric resource ở cấp container bằng cách đọc từ Linux cgroup và container runtime.

### Metric Quan Trọng từ cAdvisor

```promql
# CPU usage của container (cores/giây)
rate(container_cpu_usage_seconds_total{container!=""}[5m])

# Memory working set (thực sự đang dùng, không tính cache có thể giải phóng)
container_memory_working_set_bytes{container!=""}

# Memory limit của container
container_spec_memory_limit_bytes{container!=""}

# Memory usage % (so với limit)
container_memory_working_set_bytes / container_spec_memory_limit_bytes

# Network receive bytes/giây
rate(container_network_receive_bytes_total[5m])

# Network transmit bytes/giây
rate(container_network_transmit_bytes_total[5m])

# Disk I/O read bytes/giây
rate(container_fs_reads_bytes_total[5m])

# Số lần restart container (OOMKill, crash...)
rate(container_restarts_total[1h])
```

---

## metrics-server — Real-time Resource Metric

**metrics-server** là implementation của Kubernetes Metrics API — cung cấp metric CPU/memory real-time cho HPA, VPA, và `kubectl top`. Không lưu lịch sử, chỉ giữ giá trị hiện tại.

### Cài Đặt metrics-server

```bash
# Cài metrics-server qua Helm
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm install metrics-server metrics-server/metrics-server \
  --namespace kube-system \
  --values metrics-server-values.yaml
```

### metrics-server-values.yaml

```yaml
# metrics-server-values.yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 300m
    memory: 256Mi

args:
  # Cho phép self-signed certificate (cần trong một số môi trường)
  - --kubelet-insecure-tls
  # Thay vì dùng node name, dùng IP (giải quyết vấn đề DNS)
  - --kubelet-preferred-address-types=InternalIP

# High availability — 2 replica với leader election
replicas: 2
```

### Kiểm Tra metrics-server

```bash
# Xem CPU/memory của Node
kubectl top nodes
# NAME          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# node-1        850m         21%    4294Mi          54%
# node-2        623m         15%    3891Mi          49%

# Xem CPU/memory của Pod trong namespace
kubectl top pods -n production
# NAME                          CPU(cores)   MEMORY(bytes)
# web-api-7d9f8b6-xk2qp         245m         312Mi
# web-api-7d9f8b6-abc12         189m         298Mi

# Xem metric theo container trong Pod
kubectl top pods -n production --containers
# POD                           NAME       CPU(cores)   MEMORY(bytes)
# web-api-7d9f8b6-xk2qp         web-api    245m         312Mi
# web-api-7d9f8b6-xk2qp         sidecar    12m          24Mi

# Kiểm tra Metrics API trực tiếp
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes | jq .
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods | jq .
```

---

## kube-state-metrics — Kubernetes Object State

**kube-state-metrics (KSM)** watch kube-apiserver và expose metric về trạng thái Kubernetes object dưới dạng Prometheus metric. Trả lời câu hỏi như: "Deployment có đang healthy không?", "Pod phase là gì?", "Node có Ready không?".

### Cài Đặt và Cấu Hình

```bash
# kube-state-metrics đã được cài sẵn khi dùng kube-prometheus-stack
# Kiểm tra:
kubectl get pods -n monitoring -l app.kubernetes.io/name=kube-state-metrics

# Xem metric raw (port-forward)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-kube-state-metrics 8080:8080
curl http://localhost:8080/metrics | grep kube_deployment
```

### Cấu Hình Lọc Resource (Giảm Cardinality)

```yaml
# Trong values.yaml của kube-prometheus-stack
kube-state-metrics:
  # Chỉ collect metric cho resource quan trọng
  collectors:
    - certificatesigningrequests
    - configmaps
    - cronjobs
    - daemonsets
    - deployments
    - endpoints
    - horizontalpodautoscalers
    - ingresses
    - jobs
    - leases
    - limitranges
    - namespaces
    - networkpolicies
    - nodes
    - persistentvolumeclaims
    - persistentvolumes
    - pods
    - replicasets
    - resourcequotas
    - secrets
    - services
    - statefulsets
    - storageclasses

  # Metric label allowlist — chỉ giữ label cần thiết
  # Giảm cardinality bằng cách không giữ tất cả K8s label
  metricLabelsAllowlist:
    - pods=[app,version,team]
    - deployments=[app,environment]
    - nodes=[kubernetes.io/role]
```

---

## Metric Quan Trọng Theo Object

### Deployment Metrics

```promql
# Số replica mong muốn (desired)
kube_deployment_spec_replicas{namespace="production", deployment="web-api"}

# Số replica đang available (available = ready)
kube_deployment_status_replicas_available{namespace="production"}

# Số replica đang updated (sau rolling update)
kube_deployment_status_replicas_updated{namespace="production"}

# Deployment đang bị paused
kube_deployment_spec_paused{namespace="production"}

# Alert: desired != available (Deployment không healthy)
kube_deployment_spec_replicas != kube_deployment_status_replicas_available

# Alert: Deployment bị stuck trong rolling update > 10 phút
(
  kube_deployment_status_replicas_updated != kube_deployment_spec_replicas
  and
  changes(kube_deployment_status_replicas_updated[10m]) == 0
)
```

### Pod Metrics

```promql
# Số Pod theo phase (Running, Pending, Failed, Succeeded)
count by (phase) (kube_pod_status_phase{namespace="production"})

# Pod đang ở trạng thái Pending lâu
(kube_pod_status_phase{phase="Pending"} == 1) * on(pod, namespace)
kube_pod_created > (time() - 300)   # pending > 5 phút

# Container restart count
sum by (pod, container) (
  increase(kube_pod_container_status_restarts_total{namespace="production"}[1h])
)

# Container đang OOMKilled
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1

# Pod không có requests — nguy hiểm cho scheduling
kube_pod_container_resource_requests{resource="cpu", container!=""} == 0

# Pod readiness
kube_pod_status_ready{condition="true"} == 0   # Pod không ready
```

### Node Metrics (từ kube-state-metrics + Node Exporter)

```promql
# Node condition (Ready, MemoryPressure, DiskPressure, PIDPressure)
kube_node_status_condition{condition="Ready", status="true"} == 0   # Node không Ready

# Node taint (node bị taint không nhận Pod bình thường)
kube_node_spec_taint{effect="NoSchedule"} == 1

# Allocatable vs capacity
kube_node_status_allocatable{resource="cpu"}    # CPU có thể cấp cho Pod
kube_node_status_capacity{resource="cpu"}       # CPU tổng của node

# CPU request đã được cấp / allocatable (%) — đo packing efficiency
sum by (node) (kube_pod_container_resource_requests{resource="cpu"})
/ sum by (node) (kube_node_status_allocatable{resource="cpu"})

# Node Exporter — CPU usage thực tế
1 - avg by (node) (
  rate(node_cpu_seconds_total{mode="idle"}[5m])
)
```

### PVC Metrics (PersistentVolumeClaim — Yêu Cầu Lưu Trữ Liên Tục)

```promql
# PVC đang ở trạng thái Bound (bình thường) hay Pending/Lost
kube_persistentvolumeclaim_status_phase{phase!="Bound"} == 1

# Dung lượng đã dùng / capacity (%)
# Kết hợp với kubelet metric
(
  kubelet_volume_stats_used_bytes /
  kubelet_volume_stats_capacity_bytes
) > 0.85   # cảnh báo khi > 85% đầy

# PVC không được dùng (không có Pod nào mount)
kube_persistentvolumeclaim_status_phase{phase="Bound"} == 1
unless on(persistentvolumeclaim, namespace)
kube_pod_spec_volumes_persistentvolumeclaims_info
```

### Job Metrics

```promql
# Job thành công
kube_job_status_succeeded{namespace="production"} == 1

# Job thất bại
kube_job_status_failed{namespace="production"} > 0

# Job đang chạy quá lâu (alert nếu Job chạy > 1 giờ)
(
  time() - kube_job_status_start_time{namespace="production"}
) > 3600
and
kube_job_status_active > 0

# CronJob lần cuối thành công
time() - kube_cronjob_status_last_successful_time{namespace="production"} > 3600
```

---

## Custom Metrics API

**Custom Metrics API** và **External Metrics API** cho phép HPA scale dựa trên metric tùy chỉnh (không chỉ CPU/memory).

```
┌──────────────────────────────────────────────────────────────────┐
│               CUSTOM METRICS ARCHITECTURE                        │
│                                                                  │
│  App expose /metrics                                             │
│       │                                                          │
│       ▼                                                          │
│  Prometheus scrape                                               │
│       │                                                          │
│       ▼                                                          │
│  Prometheus Adapter                                              │
│  ├── đọc metric từ Prometheus                                   │
│  ├── expose Custom Metrics API                                  │
│  │   /apis/custom.metrics.k8s.io/v1beta1                       │
│  └── expose External Metrics API                                │
│      /apis/external.metrics.k8s.io/v1beta1                     │
│       │                                                          │
│       ▼                                                          │
│  HPA query Custom Metrics API                                   │
│       │                                                          │
│       ▼                                                          │
│  Scale Deployment theo custom metric                            │
└──────────────────────────────────────────────────────────────────┘
```

### Cài Đặt Prometheus Adapter

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  --values prometheus-adapter-values.yaml
```

### prometheus-adapter-values.yaml

```yaml
# prometheus-adapter-values.yaml
prometheus:
  url: http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local
  port: 9090

rules:
  custom:
    # Tạo custom metric: http_requests_per_second
    # HPA có thể dùng metric này để scale
    - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace:
            resource: namespace
          pod:
            resource: pod
      name:
        matches: "^(.*)_total$"
        as: "${1}_per_second"   # http_requests_per_second
      metricsQuery: |
        sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)

    # Queue depth metric — cho KEDA hoặc HPA scale worker
    - seriesQuery: 'rabbitmq_queue_messages{namespace!="",service!=""}'
      resources:
        overrides:
          namespace:
            resource: namespace
          service:
            resource: service
      name:
        as: "rabbitmq_queue_messages"
      metricsQuery: |
        max(<<.Series>>{<<.LabelMatchers>>}) by (<<.GroupBy>>)
```

### HPA Dùng Custom Metric

```yaml
# hpa-custom-metric.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-api-rps-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-api
  minReplicas: 2
  maxReplicas: 30
  metrics:
    # CPU metric kết hợp với custom metric
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60

    # Custom metric: mỗi Pod xử lý tối đa 100 req/s
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
```

```bash
# Kiểm tra custom metric API
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq '.resources[].name'

# Xem giá trị metric cụ thể
kubectl get --raw \
  "/apis/custom.metrics.k8s.io/v1beta1/namespaces/production/pods/*/http_requests_per_second" \
  | jq .
```

---

## Câu Hỏi Phỏng Vấn

**Phân biệt cAdvisor, metrics-server và kube-state-metrics — cái nào dùng cho gì?**

> Ba nguồn metric phục vụ mục đích khác nhau và thường được dùng cùng nhau. **cAdvisor** (tích hợp trong kubelet) đo resource thực tế của container: CPU usage, memory usage, network I/O, disk I/O — tức là "container đang tiêu thụ bao nhiêu tài nguyên". Prometheus scrape cAdvisor để lưu lịch sử và alert. **metrics-server** là aggregator ngắn hạn, expose Metrics API cho HPA và `kubectl top` — chỉ giữ giá trị hiện tại, không có lịch sử. Không thể dùng Prometheus thay metrics-server cho HPA vì HPA cần Metrics API standard. **kube-state-metrics** đo trạng thái Kubernetes object từ góc nhìn control plane: Deployment có bao nhiêu replica, Pod ở phase nào, Node có Ready không — tức là "cluster đang ở trạng thái gì". Không liên quan đến resource usage.

**Tại sao cần kube-state-metrics mà không dùng `kubectl get deployment` trong monitoring?**

> `kubectl get` là lệnh one-shot, không phù hợp cho monitoring liên tục. kube-state-metrics watch kube-apiserver qua list/watch API (hiệu quả, không polling) và expose metric ở format Prometheus — cho phép lưu lịch sử theo thời gian, viết PromQL query phức tạp, và alert. Ví dụ: biết Deployment hiện tại có replicas mismatch thì có thể dùng `kubectl describe`, nhưng không thể tính "Deployment này đã bị mismatch trong bao lâu?" hoặc "Trung bình mỗi tuần có bao nhiêu Deployment fail rollout?". kube-state-metrics + Prometheus cho phép trả lời những câu hỏi đó và alert ngay khi mismatch xảy ra.

**HPA custom metric hoạt động như thế nào? Cần thành phần nào?**

> Để HPA dùng custom metric (ví dụ: số request/giây), cần 3 thành phần: (1) **App expose metric** — app phải có Prometheus metric endpoint (`/metrics`) với metric cần scale (ví dụ `http_requests_total`). (2) **Prometheus scrape** và lưu metric vào TSDB. (3) **Custom Metrics API adapter** — Prometheus Adapter đọc metric từ Prometheus và expose dưới dạng Kubernetes Custom Metrics API (`/apis/custom.metrics.k8s.io/v1beta1`). HPA query API này thay vì query Prometheus trực tiếp vì HPA theo Kubernetes API standard. Cấu hình Prometheus Adapter cần `rules` xác định: Prometheus series nào → Custom Metrics API resource nào → PromQL query nào để tính giá trị. Nếu thiếu adapter, HPA sẽ báo `TARGETS: <unknown>/100` và không scale.

---

## Checklist Production

### metrics-server

- [ ] metrics-server cài đặt và `kubectl top nodes` hoạt động
- [ ] metrics-server có ít nhất 2 replica để tránh downtime
- [ ] HPA trong cluster hiển thị TARGETS có giá trị (không phải `<unknown>`)
- [ ] metrics-server resource đủ — monitor memory của metrics-server pod

### kube-state-metrics

- [ ] kube-state-metrics được cài đặt và Prometheus đang scrape thành công
- [ ] Alert rule cho Deployment unhealthy (desired != available > 5 phút)
- [ ] Alert rule cho Pod CrashLoopBackOff (restart count > 5 trong 1h)
- [ ] Alert rule cho Node NotReady
- [ ] Alert rule cho PVC đầy > 85%
- [ ] Alert rule cho Job thất bại

### Custom Metrics

- [ ] Prometheus Adapter cài đặt nếu dùng HPA custom metric
- [ ] Custom Metrics API được test: `kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1`
- [ ] HPA custom metric hiển thị TARGETS có giá trị hợp lệ
- [ ] Alert khi Custom Metrics API không available (HPA sẽ ngừng scale)

### Dashboard và Alert

- [ ] Dashboard Node resources (CPU, memory, disk) từ kube-state-metrics + Node Exporter
- [ ] Dashboard Deployment health (replica status, rollout progress)
- [ ] Dashboard Pod lifecycle (restart count, OOMKill, phase distribution)
- [ ] Dashboard PVC usage trending
- [ ] Alert khi allocatable resource < 20% (cluster sắp hết chỗ)
