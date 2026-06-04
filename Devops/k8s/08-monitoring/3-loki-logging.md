# Loki — Tổng Hợp Log

> Hướng dẫn chi tiết về Loki (hệ thống tổng hợp log — log aggregation system) của Grafana Labs: kiến trúc Distributor/Ingester/Querier/Ruler, so sánh với ELK Stack, cài đặt với Helm và Promtail, viết LogQL từ cơ bản đến nâng cao, structured logging, multi-tenancy (đa người thuê), storage backend và tích hợp Grafana để xem log kèm metric trên cùng timeline.

## Mục Lục

1. [Kiến Trúc Loki](#kiến-trúc-loki)
2. [So Sánh Loki vs ELK Stack](#so-sánh-loki-vs-elk-stack)
3. [Cài Đặt Loki + Promtail với Helm](#cài-đặt-loki--promtail-với-helm)
4. [LogQL — Loki Query Language](#logql--loki-query-language)
5. [Structured Logging và Label Indexing](#structured-logging-và-label-indexing)
6. [Multi-tenancy](#multi-tenancy)
7. [Storage Backend và Retention](#storage-backend-và-retention)
8. [Tích Hợp Grafana](#tích-hợp-grafana)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)
10. [Checklist Production](#checklist-production)

---

## Kiến Trúc Loki

```
┌──────────────────────────────────────────────────────────────────┐
│                      LOKI ARCHITECTURE                           │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                 │
│  │  App Pod A │  │  App Pod B │  │  App Pod C │                 │
│  │  (stdout)  │  │  (stdout)  │  │  (stdout)  │                 │
│  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘                 │
│         └───────────────┼───────────────┘                        │
│                         │ tail log files                         │
│                         ▼                                        │
│  ┌──────────────────────────────────────┐                        │
│  │         PROMTAIL (DaemonSet)         │                        │
│  │  ├── discover Pod via K8s API        │                        │
│  │  ├── tail /var/log/containers/*.log  │                        │
│  │  ├── add labels: namespace, pod...   │                        │
│  │  └── push to Loki Distributor        │                        │
│  └───────────────────┬──────────────────┘                        │
│                      │ HTTP push                                 │
│                      ▼                                           │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │                  LOKI WRITE PATH                          │   │
│  │                                                           │   │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐   │   │
│  │  │ Distributor │───→│  Ingester   │───→│   Storage   │   │   │
│  │  │ (validate,  │    │ (buffer in  │    │  Backend    │   │   │
│  │  │  hash ring) │    │  memory,    │    │ (S3 / GCS)  │   │   │
│  │  │             │    │  WAL)       │    │             │   │   │
│  │  └─────────────┘    └─────────────┘    └─────────────┘   │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │                  LOKI READ PATH                           │   │
│  │                                                           │   │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐   │   │
│  │  │   Querier   │←───│Query Frontend│←───│   Grafana   │   │   │
│  │  │ (query      │    │ (cache,     │    │ (LogQL UI)  │   │   │
│  │  │  storage +  │    │  split,     │    │             │   │   │
│  │  │  ingester)  │    │  retry)     │    │             │   │   │
│  │  └─────────────┘    └─────────────┘    └─────────────┘   │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────┐                            │
│  │     Ruler (Alert từ Log)        │                            │
│  │  evaluate LogQL alert rule      │                            │
│  │  → AlertManager                 │                            │
│  └──────────────────────────────────┘                            │
└──────────────────────────────────────────────────────────────────┘
```

### Các Thành Phần Chính

**Distributor:** Nhận log từ Promtail/Fluentd, validate format, phân phối đến Ingester theo consistent hashing. Stateless — có thể scale ngang.

**Ingester:** Buffer log trong memory (WAL — Write Ahead Log), flush xuống storage backend theo chunk (thường 1MB hoặc sau 1 giờ). Stateful — cần graceful shutdown để không mất data.

**Querier:** Xử lý LogQL query, query cả Ingester (data mới) và storage backend (data cũ), merge kết quả.

**Query Frontend:** Cache kết quả query, split query range lớn thành nhiều query nhỏ song song (sharding), retry khi Querier lỗi.

**Ruler:** Evaluate LogQL alert rule và metric rule (tương tự Prometheus recording rule nhưng từ log), gửi alert sang AlertManager.

---

## So Sánh Loki vs ELK Stack

| Tiêu Chí | Loki | ELK Stack (Elasticsearch + Logstash + Kibana) |
| -------- | ---- | --------------------------------------------- |
| **Indexing** | Chỉ index label (không index nội dung log) | Full-text index toàn bộ nội dung log |
| **Storage cost** | Thấp (gzip compress, object storage) | Cao (Elasticsearch index tốn ~3-5x dung lượng raw log) |
| **Query speed** | Chậm hơn với full-text search (brute force grep) | Rất nhanh với full-text search (inverted index) |
| **Query language** | LogQL (tương tự PromQL) | Lucene query, KQL (Kibana Query Language) |
| **Tích hợp K8s** | Native với Grafana, ServiceMonitor pattern | Cần Beats, Filebeat, hoặc Fluentd |
| **Multi-tenancy** | Sẵn có qua X-Scope-OrgID | Cần cấu hình phức tạp (index per tenant) |
| **Operational complexity** | Thấp hơn (single binary cho small setup) | Cao (Elasticsearch cluster management khó) |
| **Log parsing** | Query time (chậm hơn) | Index time (nhanh hơn cho structured search) |
| **Phù hợp** | Kubernetes, cloud-native, kết hợp Grafana | Enterprise, full-text search, legacy system |

> **Kết luận:** Loki tốt hơn khi: chi phí là ưu tiên, đã dùng Grafana, workload là container log. ELK tốt hơn khi: cần full-text search nhanh, log structured phức tạp, team đã quen ELK, compliance yêu cầu full-text index.

---

## Cài Đặt Loki + Promtail với Helm

### Cài Loki với Helm

```bash
# Thêm Grafana Helm repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Cài Loki Stack (Loki + Promtail) — simple scalable mode
helm install loki grafana/loki \
  --namespace monitoring \
  --values loki-values.yaml \
  --wait
```

### loki-values.yaml

```yaml
# loki-values.yaml
# Loki cấu hình simple scalable mode (phù hợp production trung bình)

loki:
  auth_enabled: false   # tắt multi-tenancy cho setup đơn giản

  commonConfig:
    replication_factor: 1   # môi trường dev; production nên dùng 3

  storage:
    type: s3
    s3:
      endpoint: s3.amazonaws.com
      region: us-east-1
      bucketnames: my-loki-logs   # S3 bucket phải tạo trước
      # Dùng IAM role (IRSA) thay vì access key
      # access_key_id: ""
      # secret_access_key: ""

  schemaConfig:
    configs:
      - from: 2024-01-01
        store: tsdb           # TSDB index (tốt hơn boltdb-shipper)
        object_store: s3
        schema: v12
        index:
          prefix: loki_index_
          period: 24h

  limits_config:
    retention_period: 744h      # 31 ngày
    ingestion_rate_mb: 16       # MB/s per tenant
    ingestion_burst_size_mb: 32
    max_query_parallelism: 32
    max_streams_per_user: 10000
    split_queries_by_interval: 30m   # split query dài thành chunks 30 phút

  compactor:
    working_directory: /var/loki/data/retention
    shared_store: s3
    compaction_interval: 10m
    retention_enabled: true    # bật auto delete theo retention_period
    retention_delete_delay: 2h

# Promtail — DaemonSet để thu thập log từ Node
promtail:
  enabled: true
  config:
    clients:
      - url: http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push
        tenant_id: default    # nếu auth_enabled: false, giá trị này bị bỏ qua

    # Pipeline — xử lý log trước khi gửi
    snippets:
      pipelineStages:
        # Stage 1: Parse CRI log format (containerd)
        - cri: {}

        # Stage 2: Parse JSON log nếu app ghi JSON
        - json:
            expressions:
              level: level
              msg: message
              traceID: traceID

        # Stage 3: Thêm label từ field JSON
        - labels:
            level:
            traceID:

        # Stage 4: Lọc log không cần thiết (giảm volume)
        - drop:
            expression: '.*healthcheck.*'
            drop_counter_reason: health_check_drop

        # Stage 5: Thêm timestamp từ log thay vì dùng thời gian nhận
        - timestamp:
            source: time
            format: RFC3339Nano
```

### Kiểm Tra Sau Cài Đặt

```bash
# Xem Pod Loki và Promtail đang chạy
kubectl get pods -n monitoring -l app.kubernetes.io/name=loki
kubectl get pods -n monitoring -l app.kubernetes.io/name=promtail

# Test query từ command line (port-forward Loki)
kubectl port-forward -n monitoring svc/loki-gateway 3100:80

# Query log 15 phút gần nhất từ namespace production
curl -G "http://localhost:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={namespace="production"}' \
  --data-urlencode 'start='$(date -d '15 minutes ago' +%s)000000000 \
  --data-urlencode 'end='$(date +%s)000000000 \
  --data-urlencode 'limit=100'

# Xem Promtail target đang được scrape
kubectl port-forward -n monitoring <promtail-pod> 3101:3101
# Mở http://localhost:3101/targets
```

---

## LogQL — Loki Query Language

### Log Stream Selector (Chọn Log Stream)

```logql
# Label matcher — cú pháp cơ bản
{namespace="production"}
{namespace="production", pod=~"web-api-.*"}     # regex match
{namespace!="kube-system"}                       # loại trừ namespace
{job="web-api", level="error"}                   # filter nhiều label

# Xem tất cả log của pod cụ thể
{pod="web-api-7d9f8b6-xk2qp"}
```

### Log Filter Expression (Lọc Nội Dung)

```logql
# Filter chứa chuỗi (case sensitive)
{namespace="production"} |= "ERROR"
{namespace="production"} |= "database connection refused"

# Loại trừ dòng chứa chuỗi
{namespace="production"} != "healthcheck"

# Regex match
{namespace="production"} |~ "ERROR|FATAL|PANIC"
{namespace="production"} !~ "DEBUG|TRACE"

# Kết hợp nhiều filter
{namespace="production", job="web-api"}
  |= "ERROR"
  != "expected error"
  |~ "user_id=[0-9]+"
```

### Parse Expression (Phân Tích Cấu Trúc Log)

```logql
# Parse JSON log
{namespace="production"}
  | json
  | level="error"
  | line_format "{{.msg}} [traceID={{.traceID}}]"

# Parse logfmt (key=value format)
{namespace="production"}
  | logfmt
  | status_code >= 500

# Parse với pattern (cho log không có cấu trúc)
{namespace="production"}
  | pattern "<_> <method> <path> <status> <duration>"
  | status >= "500"

# Parse với regexp
{namespace="production"}
  | regexp `(?P<ip>\d+\.\d+\.\d+\.\d+) - - \[(?P<timestamp>[^\]]+)\] "(?P<method>\w+) (?P<path>[^ ]+)`
  | method="POST"
```

### Metric Query (Tạo Metric Từ Log)

```logql
# Đếm số lỗi mỗi giây (rate)
sum(rate({namespace="production"} |= "ERROR" [5m])) by (pod)

# Đếm số request theo status code từ log
sum by (status) (
  rate({namespace="production"}
    | json
    | __error__=""    # bỏ qua dòng không parse được
  [5m])
)

# Top 5 pod có nhiều log lỗi nhất
topk(5,
  sum by (pod) (
    rate({namespace="production"} |= "ERROR" [5m])
  )
)

# Tính P99 latency từ log (nếu log có field duration)
histogram_quantile(0.99,
  sum by (le, pod) (
    rate({namespace="production"}
      | json
      | unwrap duration [5m]
    )
  )
)

# Bytes per second
sum(bytes_rate({namespace="production"}[5m])) by (pod)
```

---

## Structured Logging và Label Indexing

### Structured Logging với JSON Format

Loki **không full-text index** nội dung log. Để search hiệu quả, app nên ghi log theo **JSON format** với các field nhất quán.

```go
// Ví dụ Go với zerolog — JSON structured logging
log.Error().
    Str("traceID", traceID).
    Str("userID", userID).
    Str("method", r.Method).
    Str("path", r.URL.Path).
    Int("status", 500).
    Dur("duration", time.Since(start)).
    Err(err).
    Msg("request failed")

// Output JSON:
// {"level":"error","traceID":"abc123","userID":"u-456","method":"POST",
//  "path":"/api/orders","status":500,"duration":1.234,"error":"db timeout",
//  "message":"request failed","time":"2024-01-15T10:30:00Z"}
```

### Label Strategy — Chiến Lược Đặt Label

**Label là index của Loki** — chỉ nên đặt label cho field có **cardinality thấp** (ít giá trị khác nhau).

```yaml
# TỐTT — cardinality thấp, ổn định
labels:
  namespace: production      # ít namespace
  app: web-api               # ít app
  level: error               # chỉ có debug/info/warn/error/fatal
  environment: prod          # staging / prod

# XẤU — cardinality cao, gây explosion
labels:
  user_id: "user-12345"      # triệu user → triệu stream → Loki crash
  request_id: "req-uuid"     # unique mỗi request → không bao giờ dùng làm label
  ip: "1.2.3.4"              # quá nhiều IP khác nhau
```

**Quy tắc label:**
- Label nên có < 100 giá trị khác nhau
- Cardinality cao → dùng parsed field trong query, không dùng label
- Mỗi stream = unique combination của tất cả label → cardinality quá cao → Loki bị chậm

---

## Multi-tenancy

**Multi-tenancy (Đa Người Thuê)** cho phép nhiều team/project gửi và query log trong cùng Loki cluster mà không thấy dữ liệu của nhau.

```yaml
# Bật multi-tenancy trong Loki config
loki:
  auth_enabled: true   # bật multi-tenancy

  # Config giới hạn per-tenant
  limits_config:
    ingestion_rate_mb: 10    # default cho tất cả tenant
    max_streams_per_user: 5000

  # Override per-tenant (nếu cần)
  # Cần Loki Enterprise hoặc cấu hình via API
```

### Gửi Log với Tenant ID

```yaml
# Cấu hình Promtail để gửi log với tenant ID
# Mỗi namespace → tenant riêng
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    pipeline_stages:
      - labels:
          namespace: __meta_kubernetes_namespace
    # Gửi log của namespace "team-a" vào tenant "team-a"
    relabel_configs:
      - source_labels: [__meta_kubernetes_namespace]
        target_label: __tenant_id__   # Promtail dùng header X-Scope-OrgID

clients:
  - url: http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push
    # tenant_id sẽ được set từ __tenant_id__ label tự động
```

### Query Log của Tenant Cụ Thể

```bash
# Query với tenant header
curl -H "X-Scope-OrgID: team-a" \
  -G "http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/query_range" \
  --data-urlencode 'query={namespace="team-a"}'
```

---

## Storage Backend và Retention

### S3 Storage — Cấu Hình Đầy Đủ

```yaml
# AWS S3 với IRSA (IAM Role for Service Account — Vai Trò IAM cho Service Account)
loki:
  storage:
    type: s3
    s3:
      region: us-east-1
      bucketnames: my-company-loki-prod

  serviceAccount:
    annotations:
      eks.amazonaws.com/role-arn: "arn:aws:iam::123456789012:role/loki-s3-role"
```

```json
// IAM Policy cho Loki S3 role
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-company-loki-prod",
        "arn:aws:s3:::my-company-loki-prod/*"
      ]
    }
  ]
}
```

### Log Retention Policy

```yaml
# Retention config trong Loki
loki:
  limits_config:
    retention_period: 744h   # 31 ngày cho tất cả tenant

  compactor:
    retention_enabled: true
    retention_delete_delay: 2h    # xóa sau 2h so với retention expire
    retention_delete_worker_count: 150
    compaction_interval: 10m

  # Per-tenant retention (yêu cầu auth_enabled: true)
  # Cấu hình via Loki API hoặc helm values
  tenant_limits:
    team-a:
      retention_period: 2160h   # 90 ngày cho team A
    team-b:
      retention_period: 168h    # 7 ngày cho team B (ít yêu cầu hơn)
```

---

## Tích Hợp Grafana

### Correlate Log với Metric trên Cùng Timeline

Grafana cho phép xem log và metric trong cùng giao diện, liên kết với nhau qua timestamp và TraceID.

```yaml
# Cấu hình Loki datasource trong Grafana với derived fields
# File: grafana/provisioning/datasources/loki.yaml
datasources:
  - name: Loki
    type: loki
    url: http://loki-gateway.monitoring.svc.cluster.local
    jsonData:
      maxLines: 1000
      derivedFields:
        # Tự động tạo link từ traceID trong log sang Tempo
        - name: TraceID
          matcherRegex: '"traceID":"(\w+)"'   # regex tìm traceID trong JSON log
          url: "$${__value.raw}"               # URL link đến trace (dùng Tempo uid)
          datasourceUid: tempo                  # uid của datasource Tempo
          urlDisplayLabel: "View Trace"
```

### Annotations — Đánh Dấu Deploy Event

```yaml
# Trong Grafana dashboard JSON — thêm annotation từ Loki
"annotations": {
  "list": [
    {
      "datasource": {
        "type": "loki",
        "uid": "loki"
      },
      "enable": true,
      "expr": "{namespace=\"production\"} |= \"deployment completed\"",
      "name": "Deployments",
      "iconColor": "blue",
      "titleFormat": "Deploy: {{pod}}"
    }
  ]
}
```

### Grafana Explore — Ad-hoc Log Query

```bash
# Quy trình điều tra sự cố trong Grafana Explore:

# 1. Thấy error spike trên Prometheus dashboard lúc 2:15 AM
# 2. Mở Explore, chọn datasource Loki
# 3. Query log cùng thời điểm:
{namespace="production", app="web-api"} |= "ERROR" | json | level="error"

# 4. Tìm thấy log có traceID → click link đến Tempo
# 5. Xem trace breakdown → tìm được DB query chậm

# Split view: Metric + Log cùng thời điểm
# Click "Split" trong Explore → xem Prometheus và Loki song song
```

---

## Câu Hỏi Phỏng Vấn

**Tại sao Loki không index nội dung log? Ảnh hưởng thế nào đến query?**

> Loki chỉ index label (metadata) của log stream, không index nội dung từng dòng log. Quyết định này có mục đích: index nội dung như Elasticsearch tốn rất nhiều storage và CPU (thường 3-5x dung lượng raw log) — với log volume lớn trong Kubernetes, chi phí cực kỳ cao. Loki thay vào đó nén raw log bằng gzip (~10:1 ratio) và lưu trên object storage rẻ (S3). Hậu quả về query: full-text search chậm hơn vì Loki phải grep qua raw log data. Để giảm thiểu: (1) dùng label selector tốt để thu hẹp stream cần scan trước khi filter; (2) structured logging (JSON) + parse trong query; (3) recording rule từ log metric để precompute số liệu thường dùng. Trong thực tế, với label selector tốt (namespace + app + level), query response time vẫn dưới 1-2 giây cho lượng log hợp lý.

**Giải thích label cardinality vấn đề trong Loki?**

> Mỗi unique combination của label trong Loki tạo ra một **log stream** riêng biệt. Nếu dùng label có cardinality cao (ví dụ user_id, request_id), số stream tăng theo số lượng giá trị đó. Ví dụ: 1 triệu user × 10 namespace = 10 triệu stream. Loki phải giữ metadata của tất cả active stream trong memory → OOM. Ingester trở nên không ổn định, query chậm do phải merge quá nhiều stream. Giải pháp: chỉ dùng label cho field cardinality thấp và ổn định (namespace, app, environment, level). Field cardinality cao (user_id, session_id, traceID) nên để trong nội dung log và query bằng parse + filter.

**LogQL metric query dùng khi nào?**

> Metric query trong LogQL (ví dụ `rate()`, `count_over_time()`, `bytes_rate()`) chuyển đổi log stream thành time series — cho phép tạo alert và dashboard từ log mà không cần Prometheus. Dùng khi: (1) App không expose Prometheus metric nhưng ghi log có cấu trúc → extract metric từ log (ví dụ đếm HTTP status code từ access log); (2) Cần alert dựa trên nội dung log (ví dụ: alert khi log chứa "CRITICAL" vượt 10 lần/phút); (3) Đo latency từ log field duration. Hạn chế: chậm hơn Prometheus cho metric thường xuyên query — không thay thế Prometheus nhưng là fallback tốt khi metric không có sẵn.

---

## Checklist Production

### Cài Đặt

- [ ] Loki cài đặt với storage backend S3/GCS (không dùng filesystem cho production)
- [ ] Promtail deploy dạng DaemonSet trên tất cả Node
- [ ] Kiểm tra Promtail target: tất cả namespace production đang được scrape
- [ ] Loki resource (CPU/memory) đủ cho ingestion rate thực tế

### Log Management

- [ ] App ghi log dạng JSON structured (không ghi log dạng text thuần)
- [ ] Label strategy được document — chỉ dùng label cardinality thấp
- [ ] Retention được cấu hình phù hợp với yêu cầu (30-90 ngày)
- [ ] Log volume giám sát: `rate({job="promtail"}[5m])` — alert khi volume bất thường

### Multi-tenancy

- [ ] Nếu nhiều team dùng chung Loki — bật auth_enabled và cấu hình tenant
- [ ] Prometheus metric cho Loki được scrape để monitor sức khỏe Loki
- [ ] Ingestion rate limit được set per-tenant để tránh một team flood Loki

### Tích Hợp

- [ ] Grafana datasource Loki được cấu hình với derived field TraceID → Tempo
- [ ] Test query log trong Grafana Explore thành công
- [ ] Alert rule từ log (nếu dùng Loki Ruler) được test end-to-end
- [ ] TraceID được inject vào log bởi OpenTelemetry SDK để correlate trace-log
