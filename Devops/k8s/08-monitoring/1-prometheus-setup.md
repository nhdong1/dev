# Prometheus — Cài Đặt và Scrape Config

> Hướng dẫn chi tiết về Prometheus (hệ thống giám sát time-series mã nguồn mở): kiến trúc TSDB (Time Series Database — Cơ Sở Dữ Liệu Chuỗi Thời Gian), mô hình pull-based, cài đặt với kube-prometheus-stack, cấu hình ServiceMonitor, viết PromQL (Prometheus Query Language) từ cơ bản đến nâng cao, Recording Rules, Remote Write và chiến lược long-term storage với Thanos/Mimir.

## Mục Lục

1. [Kiến Trúc Prometheus](#kiến-trúc-prometheus)
2. [Cài Đặt với kube-prometheus-stack](#cài-đặt-với-kube-prometheus-stack)
3. [ServiceMonitor và PodMonitor](#servicemonitor-và-podmonitor)
4. [PromQL — Prometheus Query Language](#promql--prometheus-query-language)
5. [Recording Rules](#recording-rules)
6. [Remote Write — Long-term Storage](#remote-write--long-term-storage)
7. [Retention và Storage Sizing](#retention-và-storage-sizing)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)
9. [Checklist Production](#checklist-production)

---

## Kiến Trúc Prometheus

```
┌──────────────────────────────────────────────────────────────────┐
│                    PROMETHEUS ARCHITECTURE                       │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │ App /metrics │  │ Node Exporter│  │  kube-state-metrics    │ │
│  │ (HTTP endpoint│  │ (node metric)│  │  (K8s object state)    │ │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬─────────────┘ │
│         │                 │                       │               │
│         └─────────────────┴───────────────────────┘               │
│                           │ HTTP GET /metrics                     │
│                           │ (scrape interval: 15–60s)             │
│                           ▼                                       │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                   PROMETHEUS SERVER                        │  │
│  │                                                            │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │  │
│  │  │  Retrieval   │  │     TSDB     │  │   HTTP Server   │  │  │
│  │  │  (Scraper)   │→ │  (Storage)   │← │  (Query API)    │  │  │
│  │  └──────────────┘  └──────────────┘  └────────┬────────┘  │  │
│  │                                                │           │  │
│  │  ┌──────────────┐  ┌──────────────┐           │           │  │
│  │  │  Rule Eval   │  │ Alertmanager │           │           │  │
│  │  │  (recording  │  │  (sending    │           │           │  │
│  │  │   + alerting)│  │   alerts)    │           │           │  │
│  │  └──────────────┘  └──────────────┘           │           │  │
│  └────────────────────────────────────────────────┼───────────┘  │
│                                                   │               │
│                                            ┌──────▼──────┐        │
│                                            │   Grafana   │        │
│                                            │  (PromQL UI)│        │
│                                            └─────────────┘        │
└──────────────────────────────────────────────────────────────────┘
```

### TSDB — Time Series Database

Prometheus lưu metric theo định dạng:

```
<metric_name>{<label_name>="<label_value>", ...} <value> [<timestamp>]

Ví dụ:
http_requests_total{method="GET", status="200", handler="/api/v1/users"} 1234 1715000000
node_cpu_seconds_total{cpu="0", mode="idle"} 4590.23 1715000000
```

**Đặc điểm TSDB:**
- Lưu data theo block 2 giờ trong RAM (Head Block), flush xuống disk định kỳ
- Nén dữ liệu hiệu quả với Gorilla encoding (~1.37 bytes/sample)
- Mỗi time series được index bởi fingerprint của label set
- Write Ahead Log (WAL — Nhật Ký Ghi Trước) bảo vệ khỏi crash

### Scrape Model — Pull-based

```
Service Discovery (Kubernetes API)
│  phát hiện target (Pod, Service, Endpoint)
▼
Prometheus Retrieval
│  gọi HTTP GET <target>/metrics mỗi scrapeInterval
▼
Parse text format → time series
│  lưu vào TSDB head block
▼
TSDB compaction (2h) → disk block
```

---

## Cài Đặt với kube-prometheus-stack

**kube-prometheus-stack** (trước đây là prometheus-operator) là Helm chart tích hợp đầy đủ:
Prometheus + Alertmanager + Grafana + Node Exporter + kube-state-metrics + các CRD (ServiceMonitor, PodMonitor, PrometheusRule...).

### Thêm Helm Repo và Cài Đặt

```bash
# Thêm Helm repository của Prometheus community
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Tạo namespace cho monitoring stack
kubectl create namespace monitoring

# Cài đặt với custom values
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version 58.0.0 \
  --values prometheus-values.yaml \
  --wait
```

### values.yaml Đầy Đủ

```yaml
# prometheus-values.yaml
# Cấu hình kube-prometheus-stack cho môi trường production

prometheus:
  prometheusSpec:
    # Retention — thời gian lưu metric trên local disk
    retention: 15d
    retentionSize: "50GB"   # giới hạn thêm theo dung lượng

    # Storage — dùng PVC thay vì emptyDir
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3   # AWS EBS gp3, hoặc standard-rwo trên GKE
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi

    # Resource cho Prometheus pod
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 2
        memory: 8Gi

    # Cho phép Prometheus tìm ServiceMonitor ở tất cả namespace
    serviceMonitorSelectorNilUsesHelmValues: false
    serviceMonitorNamespaceSelector: {}   # tất cả namespace
    serviceMonitorSelector: {}            # tất cả ServiceMonitor

    podMonitorSelectorNilUsesHelmValues: false
    podMonitorNamespaceSelector: {}
    podMonitorSelector: {}

    # Scrape interval mặc định
    scrapeInterval: 30s
    evaluationInterval: 30s

    # Thêm external label để phân biệt khi dùng Thanos/federation
    externalLabels:
      cluster: production-us-east-1
      environment: production

    # Remote write sang Thanos Receiver hoặc Mimir
    remoteWrite:
      - url: "http://thanos-receive.monitoring.svc.cluster.local:19291/api/v1/receive"
        queue_config:
          max_samples_per_send: 10000
          batch_send_deadline: 5s

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          resources:
            requests:
              storage: 10Gi
    resources:
      requests:
        cpu: 100m
        memory: 256Mi

grafana:
  enabled: true
  adminPassword: "change-me-in-production"   # dùng Secret trong production

  persistence:
    enabled: true
    storageClassName: gp3
    size: 10Gi

  # Cấu hình datasource tự động
  additionalDataSources:
    - name: Loki
      type: loki
      url: http://loki-gateway.monitoring.svc.cluster.local
      version: 1
      isDefault: false

# Node Exporter — thu thập metric từng Node
nodeExporter:
  enabled: true

# kube-state-metrics — metric về K8s object state
kubeStateMetrics:
  enabled: true

# Cài đặt rules alerting mặc định (CPU, memory, disk, pod crash...)
defaultRules:
  create: true
  rules:
    alertmanager: true
    etcd: true
    general: true
    kubeApiserverAvailability: true
    kubeApiserverBurnrate: true
    kubelet: true
    kubePrometheusNodeRecording: true
    kubernetesApps: true
    kubernetesResources: true
    kubernetesStorage: true
    kubernetesSystem: true
    node: true
    prometheus: true
```

### Kiểm Tra Sau Cài Đặt

```bash
# Kiểm tra tất cả Pod đang chạy
kubectl get pods -n monitoring

# Kiểm tra ServiceMonitor đã được tạo
kubectl get servicemonitors -n monitoring

# Xem target đang được scrape (port-forward để truy cập UI)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# Mở http://localhost:9090/targets

# Xem alert rule
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# Mở http://localhost:9090/rules
```

---

## ServiceMonitor và PodMonitor

### ServiceMonitor — Giám Sát qua Service

**ServiceMonitor** là CRD (Custom Resource Definition — Định Nghĩa Tài Nguyên Tùy Chỉnh) cho phép Prometheus tự động tìm và scrape Service theo label selector. Đây là cách khuyến nghị thay vì cấu hình scrape tĩnh.

```yaml
# servicemonitor-example.yaml
# Giám sát ứng dụng web-api qua Service

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: web-api-monitor
  namespace: production           # namespace của ServiceMonitor
  labels:
    release: kube-prometheus-stack   # label để Prometheus nhận ServiceMonitor này
spec:
  # Tìm Service trong namespace nào
  namespaceSelector:
    matchNames:
      - production
      - staging

  # Label selector để tìm Service
  selector:
    matchLabels:
      app: web-api
      monitoring: "true"

  endpoints:
    - port: metrics              # tên port trong Service spec
      interval: 30s             # scrape mỗi 30 giây
      scrapeTimeout: 10s        # timeout cho mỗi scrape
      path: /metrics            # đường dẫn metrics endpoint
      scheme: http

      # Nếu app dùng Basic Auth hoặc Bearer Token
      # bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token

      # TLS config nếu app expose HTTPS
      # tlsConfig:
      #   caFile: /etc/prometheus/secrets/ca.crt
      #   insecureSkipVerify: false

      # Relabeling — thêm/đổi label trước khi lưu vào TSDB
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_name]
          targetLabel: pod
        - sourceLabels: [__meta_kubernetes_namespace]
          targetLabel: namespace
```

### Service Tương Ứng (Phải Có Port Đặt Tên)

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-api
  namespace: production
  labels:
    app: web-api
    monitoring: "true"   # label được ServiceMonitor selector dùng
spec:
  selector:
    app: web-api
  ports:
    - name: http        # port cho traffic
      port: 80
      targetPort: 8080
    - name: metrics     # port cho metrics — ServiceMonitor trỏ vào đây
      port: 9090
      targetPort: 9090
```

### PodMonitor — Giám Sát Trực Tiếp Pod

**PodMonitor** scrape trực tiếp từ Pod thay vì qua Service — hữu ích khi Pod không có Service, hoặc cần scrape từng Pod riêng (StatefulSet).

```yaml
# podmonitor-example.yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: batch-worker-monitor
  namespace: production
  labels:
    release: kube-prometheus-stack
spec:
  namespaceSelector:
    matchNames:
      - production
  selector:
    matchLabels:
      app: batch-worker
  podMetricsEndpoints:
    - port: metrics
      interval: 60s     # batch worker — scrape ít thường xuyên hơn
      path: /metrics
```

---

## PromQL — Prometheus Query Language

### Cú Pháp Cơ Bản

```promql
# Lấy giá trị hiện tại của metric
http_requests_total

# Filter theo label
http_requests_total{status="500"}
http_requests_total{status=~"5.."}          # regex: status bắt đầu bằng 5
http_requests_total{status!~"2.."}          # loại trừ status 2xx
http_requests_total{namespace="production", job="web-api"}

# Range vector — lấy giá trị trong khoảng thời gian
http_requests_total[5m]                     # 5 phút gần nhất

# Offset — so sánh với quá khứ
http_requests_total offset 1h              # giá trị 1 giờ trước
```

### Hàm Thường Dùng

```promql
# rate() — tốc độ thay đổi trung bình của counter trong khoảng thời gian
# Dùng cho: request/s, error/s, byte/s
rate(http_requests_total{status=~"5.."}[5m])

# irate() — tốc độ tức thời (2 sample gần nhất)
# Dùng khi: cần phát hiện spike ngắn, nhưng bị noise hơn rate()
irate(http_requests_total[5m])

# increase() — tổng tăng trong khoảng thời gian (= rate * duration)
increase(http_requests_total[1h])           # tổng request trong 1 giờ

# sum() — tổng hợp nhiều time series
sum(rate(http_requests_total[5m]))          # tổng request/s từ tất cả instance

# sum by() — tổng hợp nhóm theo label
sum by (status) (rate(http_requests_total[5m]))   # request/s theo từng status code
sum by (namespace, pod) (container_cpu_usage_seconds_total)

# avg, max, min
avg(container_memory_usage_bytes{namespace="production"})
max by (node) (node_cpu_seconds_total{mode="idle"})

# topk() — top N time series cao nhất
topk(5, sum by (pod) (rate(http_requests_total[5m])))   # top 5 pod nhiều request

# histogram_quantile() — tính percentile từ histogram
# P99 latency trong 5 phút
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# P50, P95, P99 theo job
histogram_quantile(0.95,
  sum by (job, le) (
    rate(http_request_duration_seconds_bucket{job="web-api"}[5m])
  )
)

# predict_linear() — dự báo giá trị trong tương lai
# Disk sẽ đầy sau bao lâu?
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 4 * 3600)  # dự báo 4h
```

### Query Thực Tế Hữu Ích

```promql
# Error rate (tỉ lệ lỗi) — % request 5xx trên tổng
sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
sum(rate(http_requests_total[5m]))

# CPU utilization của từng Pod
sum by (pod, namespace) (
  rate(container_cpu_usage_seconds_total{container!=""}[5m])
)

# Memory working set của Pod (không tính cache)
sum by (pod, namespace) (
  container_memory_working_set_bytes{container!=""}
)

# Pod restart nhiều — phát hiện CrashLoopBackOff
increase(kube_pod_container_status_restarts_total[1h]) > 5

# Node CPU usage tổng
1 - avg by (instance) (
  rate(node_cpu_seconds_total{mode="idle"}[5m])
)

# Disk usage %
1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)

# Request P99 latency cao hơn 1 giây
histogram_quantile(0.99,
  sum by (le, job) (rate(http_request_duration_seconds_bucket[5m]))
) > 1
```

---

## Recording Rules

**Recording Rules (Quy Tắc Ghi)** tính toán trước (pre-compute) các biểu thức PromQL phức tạp và lưu kết quả vào metric mới. Lợi ích:
- Dashboard load nhanh hơn (không tính PromQL phức tạp mỗi lần)
- Giảm tải query cho TSDB
- Alert nhanh hơn vì rule đơn giản hơn

```yaml
# prometheus-recording-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: web-api-recording-rules
  namespace: monitoring
  labels:
    release: kube-prometheus-stack   # label để Prometheus nhận rule này
spec:
  groups:
    - name: web_api_request_rates
      interval: 30s    # tính lại mỗi 30 giây
      rules:
        # Tính request rate mỗi 5 phút, lưu vào metric mới
        - record: job:http_requests_total:rate5m
          expr: |
            sum by (job, status) (
              rate(http_requests_total[5m])
            )

        # Error rate tổng
        - record: job:http_error_rate:ratio5m
          expr: |
            sum(rate(http_requests_total{status=~"5.."}[5m]))
              /
            sum(rate(http_requests_total[5m]))

        # P99 latency đã được tính sẵn
        - record: job:http_request_duration_seconds:p99_5m
          expr: |
            histogram_quantile(0.99,
              sum by (job, le) (
                rate(http_request_duration_seconds_bucket[5m])
              )
            )

    - name: node_resource_rules
      rules:
        # CPU utilization mỗi node
        - record: instance:node_cpu_utilisation:ratio
          expr: |
            1 - avg by (instance) (
              rate(node_cpu_seconds_total{mode="idle"}[5m])
            )

        # Memory utilization mỗi node
        - record: instance:node_memory_utilisation:ratio
          expr: |
            1 - (
              node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
            )
```

**Quy ước đặt tên recording rule:**
```
<aggregation_level>:<metric_name>:<operations>
Ví dụ: job:http_requests_total:rate5m
        namespace:container_memory_usage:sum
        instance:node_cpu_utilisation:ratio
```

---

## Remote Write — Long-term Storage

**Remote Write (Ghi Từ Xa)** là cơ chế Prometheus gửi metric sang backend khác để lưu trữ lâu dài. Prometheus mặc định chỉ giữ 15 ngày; Remote Write giúp lưu metric 1–2 năm cho capacity planning.

### Cấu Hình Remote Write sang Thanos

```yaml
# Trong PrometheusSpec (values.yaml hoặc Prometheus CRD)
prometheus:
  prometheusSpec:
    remoteWrite:
      - url: "http://thanos-receive.monitoring.svc.cluster.local:19291/api/v1/receive"
        # Queue config để tránh mất data khi backend chậm
        queue_config:
          capacity: 10000             # số sample trong queue
          max_shards: 200             # số goroutine ghi song song
          min_shards: 1
          max_samples_per_send: 10000
          batch_send_deadline: 5s     # gửi batch sau tối đa 5 giây
          min_backoff: 30ms
          max_backoff: 5s
        # Chỉ gửi metric khớp với relabel config
        writeRelabelConfigs:
          - sourceLabels: [__name__]
            regex: "^(http_|container_|node_|kube_).*"
            action: keep             # chỉ giữ metric có prefix trên
```

### Kiến Trúc Thanos với Remote Write

```
┌─────────────────────────────────────────────────────────────┐
│                  THANOS ARCHITECTURE                        │
│                                                             │
│  Prometheus A ──remote write──┐                            │
│  Prometheus B ──remote write──┤                            │
│  Prometheus C ──remote write──┘                            │
│                               │                            │
│                               ▼                            │
│                    Thanos Receive                           │
│                    (ingestor)                              │
│                         │                                  │
│                         ▼                                  │
│                    Object Storage                          │
│               (S3 / GCS / Azure Blob)                     │
│                         │                                  │
│              ┌──────────┴──────────┐                      │
│              ▼                     ▼                      │
│       Thanos Store             Thanos Compact              │
│     (query từ S3)             (compact + downsample)      │
│              │                                            │
│              ▼                                            │
│       Thanos Query                                        │
│    (global query view)                                   │
│              │                                            │
│              ▼                                            │
│           Grafana                                         │
│   (datasource: Thanos Query)                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Retention và Storage Sizing

### Tính Toán Dung Lượng Prometheus

```
Công thức ước tính:
disk_space = retention_days × ingested_samples_per_second × bytes_per_sample

bytes_per_sample ≈ 1.5 bytes (với Prometheus TSDB compression)

Ví dụ:
- Ingestion rate: 100,000 samples/giây
- Retention: 15 ngày
- disk_space = 15 × 86400 × 100000 × 1.5 bytes
             = 15 × 86400 × 150000 bytes
             = 194,400,000,000 bytes ≈ 181 GB
```

### Kiểm Tra Ingestion Rate Thực Tế

```bash
# Query Prometheus để xem ingestion rate hiện tại
# Mở http://localhost:9090 (sau khi port-forward)

# Số samples đang được ingest mỗi giây
rate(prometheus_tsdb_head_samples_appended_total[5m])

# Số time series đang active
prometheus_tsdb_head_series

# Dung lượng WAL hiện tại
prometheus_tsdb_wal_storage_size_bytes

# Head block size (RAM đang dùng)
prometheus_tsdb_head_chunks_storage_size_bytes
```

### Cấu Hình Retention

```yaml
# Trong Prometheus CRD hoặc values.yaml
prometheusSpec:
  retention: 15d           # giữ 15 ngày
  retentionSize: "50GB"    # hoặc giới hạn theo dung lượng (chọn cái đạt trước)

  # Giảm số lượng metric để tiết kiệm storage
  # Dùng metric relabeling để drop metric không cần
  metricRelabelings:
    - sourceLabels: [__name__]
      regex: "go_gc_.*|go_memstats_.*"   # drop Go runtime metric chi tiết
      action: drop
```

---

## Câu Hỏi Phỏng Vấn

**Prometheus pull model hoạt động như thế nào? Ưu điểm so với push model?**

> Prometheus chủ động gọi HTTP GET đến endpoint `/metrics` của từng target theo interval cấu hình (thường 15–30 giây). Target phải expose metric ở format text (hoặc protobuf). Ưu điểm của pull model: (1) **Phát hiện target down** — nếu target không trả lời scrape, Prometheus đặt metric `up=0` ngay lập tức; với push model, bạn không biết target đã stop push vì lý do gì. (2) **Service discovery tự động** — Prometheus dùng Kubernetes API để tự phát hiện Pod, Service mới mà không cần cấu hình tay; target không cần biết Prometheus ở đâu. (3) **Kiểm soát tải** — Prometheus kiểm soát tần suất scrape; với push, target có thể flood dữ liệu. Hạn chế: không phù hợp với short-lived job (batch job chạy < 1 phút) → dùng Pushgateway để job push metric vào rồi Prometheus scrape Pushgateway.

**PromQL: Giải thích sự khác biệt giữa `rate()` và `irate()`?**

> `rate()` tính tốc độ tăng trung bình của counter trong toàn bộ range window (ví dụ 5 phút). Nó dùng điểm đầu và điểm cuối của window, smooth out các spike ngắn. `irate()` tính tốc độ tức thời dựa trên 2 sample cuối cùng trong window, rất nhạy với spike. Thực tế: dùng `rate()` cho dashboard và alert (ổn định hơn, ít false positive). Dùng `irate()` khi muốn phát hiện spike ngắn trong Explore/debug. Lưu ý: không dùng `irate()` với window ngắn hơn scrape interval × 2 vì có thể trả về `NaN`.

**Recording rule là gì và khi nào cần dùng?**

> Recording rule tính toán trước biểu thức PromQL phức tạp và lưu kết quả như một metric mới. Cần dùng khi: (1) Dashboard query chậm do biểu thức nặng (histogram_quantile qua nhiều ngàn time series) — pre-compute rút ngắn query time từ giây xuống millisecond. (2) Alert rule dùng biểu thức phức tạp — pre-compute giảm CPU Prometheus khi evaluate. (3) Tính toán aggregate nhiều bước (ví dụ tính P99 rồi so sánh với SLO) — lưu intermediate result. Quy tắc: luôn pre-compute metric nào được query thường xuyên (dashboard load mỗi lần refresh), và metric nào cần alert nhanh.

**Thanos khác Prometheus đơn lẻ như thế nào?**

> Prometheus đơn lẻ bị giới hạn bởi một node — thường phù hợp cho cluster đến ~1 triệu active time series và retention 15–30 ngày. Thanos giải quyết 3 vấn đề: (1) **Long-term storage** — Thanos Sidecar hoặc Remote Write gửi metric lên object storage (S3, GCS) → lưu metric 1–2 năm với chi phí thấp. (2) **Global query** — Thanos Query aggregates nhiều Prometheus instance (multi-cluster, HA pair) thành một query view duy nhất, deduplication tự động. (3) **High availability** — có thể chạy 2 Prometheus replica giống hệt nhau, Thanos dedup khi query. Overhead: cần thêm Thanos component (Sidecar, Store, Query, Compact) và object storage setup.

---

## Checklist Production

### Cài Đặt

- [ ] kube-prometheus-stack cài đặt qua Helm với values.yaml được version control
- [ ] Prometheus dùng PVC (PersistentVolumeClaim — Yêu Cầu Lưu Trữ Liên Tục) thay vì emptyDir
- [ ] Resource requests/limits được đặt cho Prometheus, Alertmanager, Grafana
- [ ] External label được đặt (cluster, environment) để phân biệt trong multi-cluster setup

### Scrape Config

- [ ] ServiceMonitor tạo cho tất cả app quan trọng
- [ ] Verify target trạng thái `UP` trên Prometheus UI `/targets`
- [ ] Scrape interval phù hợp (30s cho app, 60s cho infra metric ít thay đổi)
- [ ] TLS/auth được cấu hình cho endpoint nhạy cảm

### PromQL và Recording Rules

- [ ] Recording rule cho tất cả query nặng trong dashboard
- [ ] Recording rule cho SLO metric (error rate, latency P99)
- [ ] Test recording rule output bằng `promtool test rules`
- [ ] Không dùng cardinality (số lượng unique label value) quá cao — tránh label chứa user ID, URL path không normalize

### Retention và Storage

- [ ] Retention phù hợp với yêu cầu (15–30 ngày local, 1 năm+ với Thanos/Mimir)
- [ ] Remote write cấu hình và verify data xuất hiện trên backend
- [ ] Alert khi Prometheus disk usage > 80%
- [ ] Monitor ingestion rate: `rate(prometheus_tsdb_head_samples_appended_total[5m])`
