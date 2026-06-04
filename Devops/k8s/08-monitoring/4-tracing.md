# Distributed Tracing với Jaeger và Tempo

> Hướng dẫn chi tiết về Distributed Tracing (Theo Dõi Phân Tán) trong Kubernetes: khái niệm Span, Trace, TraceContext, OpenTelemetry (OTel) — chuẩn thống nhất cho instrumentation, cài đặt Jaeger và Grafana Tempo, auto-instrumentation với OTel Operator, chiến lược sampling (lấy mẫu), và correlate traces với logs qua TraceID.

## Mục Lục

1. [Khái Niệm Distributed Tracing](#khái-niệm-distributed-tracing)
2. [OpenTelemetry — Chuẩn Thống Nhất](#opentelemetry--chuẩn-thống-nhất)
3. [Cài Đặt Jaeger](#cài-đặt-jaeger)
4. [Cài Đặt Grafana Tempo](#cài-đặt-grafana-tempo)
5. [Auto-instrumentation với OTel Operator](#auto-instrumentation-với-otel-operator)
6. [Sampling Strategy — Chiến Lược Lấy Mẫu](#sampling-strategy--chiến-lược-lấy-mẫu)
7. [Correlate Traces với Logs](#correlate-traces-với-logs)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)
9. [Checklist Production](#checklist-production)

---

## Khái Niệm Distributed Tracing

### Ví Dụ Microservice Thực Tế

```
User gửi request POST /api/orders
│
▼
┌─────────────────────────────────────────────────────────────────┐
│              TRACE: "Create Order" — traceID: abc123            │
│                    Tổng thời gian: 350ms                        │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ SPAN: api-gateway  [0ms ──────────────────────── 350ms]│    │
│  │       spanID: s001  parentID: (root)                   │    │
│  └──────────────────────┬─────────────────────────────────┘    │
│                         │ gọi order-service                     │
│  ┌──────────────────────▼─────────────────────────────────┐    │
│  │ SPAN: order-service [20ms ──────────────── 320ms]      │    │
│  │       spanID: s002  parentID: s001                     │    │
│  └──────┬───────────────────────────────────┬─────────────┘    │
│         │ gọi inventory-service             │ gọi database     │
│  ┌──────▼──────────────┐    ┌───────────────▼─────────────┐    │
│  │ SPAN: inventory     │    │ SPAN: postgres query         │    │
│  │ [25ms ─── 80ms]     │    │ [30ms ──────────── 290ms]   │    │
│  │ spanID: s003        │    │ spanID: s004                 │    │
│  │ parentID: s002      │    │ parentID: s002               │    │
│  └─────────────────────┘    └─────────────────────────────┘    │
│                                                                 │
│  Nhìn vào trace → thấy ngay postgres query chiếm 260ms/350ms  │
│  → đây là bottleneck cần tối ưu                                │
└─────────────────────────────────────────────────────────────────┘
```

### Các Khái Niệm Cốt Lõi

**Trace (Vết Theo Dõi):** Toàn bộ hành trình của một request qua tất cả service. Mỗi trace có `traceID` duy nhất.

**Span (Đoạn):** Một đơn vị công việc trong trace. Ví dụ: một HTTP call, một DB query, một gRPC call. Span có:
- `spanID` — ID duy nhất của span
- `parentSpanID` — span cha (để build tree structure)
- `traceID` — thuộc trace nào
- `startTime` và `endTime` — để tính duration
- `tags/attributes` — metadata (HTTP method, status code, DB query...)
- `events/logs` — sự kiện trong span (ví dụ: "cache miss", "retry attempt 1")
- `status` — OK, ERROR, UNSET

**TraceContext (Bối Cảnh Vết):** Header HTTP được truyền từ service này sang service khác để connect span thành trace. Standard: W3C TraceContext (`traceparent` header).

```
traceparent: 00-abc123def456-s001-01
             ^  ^             ^    ^
             version  traceID  spanID  flags (01=sampled)
```

**Baggage (Hành Lý):** Key-value metadata được propagate cùng với trace context qua toàn bộ call chain. Dùng để truyền thông tin như `userId`, `tenantId` xuyên suốt các service mà không cần sửa từng API signature.

---

## OpenTelemetry — Chuẩn Thống Nhất

**OpenTelemetry (OTel)** là CNCF project hợp nhất OpenTracing và OpenCensus, cung cấp chuẩn thống nhất cho instrumentation của traces, metrics và logs.

```
┌──────────────────────────────────────────────────────────────────┐
│                   OPENTELEMETRY ARCHITECTURE                     │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                APPLICATION LAYER                         │   │
│  │                                                          │   │
│  │  App Code                                                │   │
│  │      │ instrument với OTel SDK                          │   │
│  │      ▼                                                   │   │
│  │  OTel SDK (Go / Java / Python / Node.js...)              │   │
│  │      │ export via OTLP (OTel Protocol)                   │   │
│  └──────┼───────────────────────────────────────────────────┘   │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              OTEL COLLECTOR (Agent / Gateway)            │   │
│  │                                                          │   │
│  │  Receivers:  OTLP, Jaeger, Zipkin, Prometheus            │   │
│  │      │                                                   │   │
│  │  Processors: batch, filter, attribute, sampling          │   │
│  │      │                                                   │   │
│  │  Exporters: Jaeger, Tempo, Prometheus, Loki              │   │
│  └──────┬───────────────────────────────────────────────────┘   │
│         │                                                        │
│    ┌────┴────┬──────────┐                                        │
│    ▼         ▼          ▼                                        │
│  Jaeger   Tempo    Prometheus                                    │
│ (traces) (traces)  (metrics)                                    │
└──────────────────────────────────────────────────────────────────┘
```

### OTel SDK — Instrumentation Code

```go
// main.go — khởi tạo OTel SDK trong Go application
package main

import (
    "context"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/trace"
    "go.opentelemetry.io/otel/sdk/resource"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
)

func initTracer(ctx context.Context) (*trace.TracerProvider, error) {
    // Exporter gửi trace đến OTel Collector (OTLP gRPC)
    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("otel-collector.monitoring.svc.cluster.local:4317"),
        otlptracegrpc.WithInsecure(),
    )

    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithSampler(trace.TraceIDRatioBased(0.1)), // sample 10%
        trace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName("order-service"),
            semconv.ServiceVersion("v1.2.3"),
            semconv.DeploymentEnvironment("production"),
        )),
    )

    otel.SetTracerProvider(tp)
    return tp, nil
}

// Trong handler — tạo span
func createOrder(ctx context.Context, order Order) error {
    tracer := otel.Tracer("order-service")

    ctx, span := tracer.Start(ctx, "createOrder")
    defer span.End()

    // Thêm attribute vào span
    span.SetAttributes(
        attribute.String("order.id", order.ID),
        attribute.Int("order.items", len(order.Items)),
    )

    // Gọi DB — span con tự động được tạo bởi OTel driver
    if err := db.CreateOrder(ctx, order); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return err
    }

    return nil
}
```

---

## Cài Đặt Jaeger

### Jaeger All-in-One (Môi Trường Dev)

```yaml
# jaeger-all-in-one.yaml — deployment đơn giản cho dev/staging
# Tất cả component trong một Pod, lưu data in-memory
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/all-in-one:1.55
          env:
            - name: COLLECTOR_OTLP_ENABLED
              value: "true"   # nhận trace qua OTLP protocol
            - name: MEMORY_MAX_TRACES
              value: "50000"  # giới hạn trace trong memory (dev only)
          ports:
            - containerPort: 16686   # Jaeger UI
            - containerPort: 4317    # OTLP gRPC
            - containerPort: 4318    # OTLP HTTP
            - containerPort: 14268   # Jaeger HTTP (legacy)
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 1
              memory: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger
  namespace: monitoring
spec:
  selector:
    app: jaeger
  ports:
    - name: ui
      port: 16686
    - name: otlp-grpc
      port: 4317
    - name: otlp-http
      port: 4318
```

### Jaeger Production với Elasticsearch

```bash
# Cài Jaeger Operator trước
kubectl create namespace observability
kubectl apply -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.55.0/jaeger-operator.yaml -n observability

# Sau đó tạo Jaeger instance với Elasticsearch backend
```

```yaml
# jaeger-production.yaml — Jaeger với Elasticsearch backend
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-production
  namespace: observability
spec:
  strategy: production    # production mode: tách riêng collector, query, ingester

  collector:
    replicas: 3
    resources:
      requests:
        cpu: 500m
        memory: 512Mi

  query:
    replicas: 2
    options:
      query:
        base-path: /jaeger   # nếu expose qua ingress với prefix

  storage:
    type: elasticsearch
    options:
      es:
        server-urls: https://elasticsearch.monitoring.svc.cluster.local:9200
        index-prefix: jaeger
        tls:
          enabled: true
    secretName: jaeger-elasticsearch-secret   # username/password

  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
    hosts:
      - jaeger.example.com
```

---

## Cài Đặt Grafana Tempo

**Grafana Tempo** là backend tracing nhẹ hơn Jaeger: không cần Elasticsearch, lưu trace trực tiếp lên object storage (S3/GCS), tích hợp tốt với Loki và Prometheus trong Grafana ecosystem.

```bash
# Cài Tempo qua Helm
helm repo add grafana https://grafana.github.io/helm-charts
helm install tempo grafana/tempo-distributed \
  --namespace monitoring \
  --values tempo-values.yaml
```

### tempo-values.yaml

```yaml
# tempo-values.yaml
tempo:
  reportingEnabled: false   # tắt telemetry gửi về Grafana

  # Storage backend
  storage:
    trace:
      backend: s3
      s3:
        bucket: my-company-tempo-traces
        region: us-east-1
        endpoint: s3.amazonaws.com

  # Retention — bao lâu giữ trace
  compactor:
    compaction:
      block_retention: 720h   # 30 ngày

  # Ingester — nhận trace từ OTel Collector
  ingester:
    replicas: 3
    resources:
      requests:
        cpu: 500m
        memory: 1Gi

  # Distributor — nhận trace, route đến ingester
  distributor:
    replicas: 2
    config:
      receivers:
        otlp:
          protocols:
            grpc:
              endpoint: 0.0.0.0:4317
            http:
              endpoint: 0.0.0.0:4318
        jaeger:
          protocols:
            thrift_http:
              endpoint: 0.0.0.0:14268

# Grafana Tempo tích hợp với Grafana qua datasource type "tempo"
# Không cần UI riêng — xem trace trực tiếp trong Grafana Explore
```

### OTel Collector — Nhận Trace và Forward sang Tempo

```yaml
# otel-collector-config.yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  mode: DaemonSet   # một collector trên mỗi node (agent mode)
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      batch:
        timeout: 1s
        send_batch_size: 1024
      memory_limiter:
        limit_mib: 400
        spike_limit_mib: 100
      # Thêm K8s metadata vào span (namespace, pod, node)
      k8sattributes:
        auth_type: serviceAccount
        passthrough: false
        extract:
          metadata:
            - k8s.namespace.name
            - k8s.pod.name
            - k8s.node.name
            - k8s.deployment.name

    exporters:
      otlp/tempo:
        endpoint: tempo-distributor.monitoring.svc.cluster.local:4317
        tls:
          insecure: true
      # Cũng export metrics từ trace sang Prometheus (span metrics)
      prometheusremotewrite:
        endpoint: http://prometheus.monitoring.svc.cluster.local:9090/api/v1/write

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, k8sattributes, batch]
          exporters: [otlp/tempo]
```

---

## Auto-instrumentation với OTel Operator

**OTel Operator** cho phép inject OpenTelemetry instrumentation vào Pod mà **không cần sửa code** — chỉ cần thêm annotation vào Deployment.

### Cài Đặt OTel Operator

```bash
# Cài cert-manager trước (OTel Operator yêu cầu)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Cài OTel Operator
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm install opentelemetry-operator open-telemetry/opentelemetry-operator \
  --namespace monitoring \
  --set "manager.collectorImage.repository=otel/opentelemetry-collector-contrib" \
  --wait
```

### Instrumentation CRD — Cấu Hình Auto-inject

```yaml
# instrumentation.yaml — cấu hình auto-instrumentation
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: auto-instrumentation
  namespace: production
spec:
  # Địa chỉ OTel Collector nhận trace từ sidecar
  exporter:
    endpoint: http://otel-collector.monitoring.svc.cluster.local:4318

  propagators:
    - tracecontext   # W3C TraceContext header
    - baggage        # W3C Baggage header
    - b3             # Zipkin B3 header (backward compat)

  sampler:
    type: parentbased_traceidratio
    argument: "0.1"   # sample 10% trace

  # Cấu hình cho từng ngôn ngữ
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:1.32.0
    env:
      - name: OTEL_INSTRUMENTATION_JDBC_ENABLED
        value: "true"
      - name: OTEL_INSTRUMENTATION_SPRING_WEB_ENABLED
        value: "true"

  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.43b0

  nodejs:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:0.46.0
```

### Thêm Annotation vào Deployment để Kích Hoạt Auto-inject

```yaml
# web-api-deployment.yaml — thêm annotation để OTel Operator inject sidecar
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-api
  namespace: production
spec:
  template:
    metadata:
      annotations:
        # Annotation này kích hoạt auto-instrumentation
        # OTel Operator inject init container và env var vào Pod
        instrumentation.opentelemetry.io/inject-java: "auto-instrumentation"
        # Hoặc:
        # instrumentation.opentelemetry.io/inject-python: "auto-instrumentation"
        # instrumentation.opentelemetry.io/inject-nodejs: "auto-instrumentation"
    spec:
      containers:
        - name: web-api
          image: myapp:1.0.0
          # Không cần thêm bất kỳ OTel code nào — Operator tự inject!
```

---

## Sampling Strategy — Chiến Lược Lấy Mẫu

Sample 100% trace là **không thực tế** cho production: chi phí lưu trữ cao, ảnh hưởng performance app. Cần chiến lược sampling phù hợp.

```
┌──────────────────────────────────────────────────────────────────┐
│              HEAD-BASED vs TAIL-BASED SAMPLING                   │
│                                                                  │
│  HEAD-BASED SAMPLING                 TAIL-BASED SAMPLING        │
│  (quyết định ở đầu request)         (quyết định sau khi xong)  │
│                                                                  │
│  Request vào → 10% sample? → Y       Request vào → ghi tất cả  │
│                                       Request xong              │
│  ┌──────────────────────────┐         ├── nếu ERROR → giữ lại   │
│  │ Đơn giản, overhead thấp  │         ├── nếu latency > 1s → giữ │
│  │ Không đảm bảo giữ lại    │         └── ngược lại → xóa 90%   │
│  │ trace có lỗi/latency cao │                                    │
│  └──────────────────────────┘  ┌──────────────────────────────┐  │
│                                │ Giữ 100% trace quan trọng    │  │
│                                │ Phức tạp hơn, cần buffer     │  │
│                                │ (OTel Collector tail sampler) │  │
│                                └──────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### Head-based Sampling (Đơn Giản, Phổ Biến)

```yaml
# Trong Instrumentation CRD hoặc OTel SDK config
sampler:
  type: parentbased_traceidratio
  argument: "0.05"   # sample 5% trace ngẫu nhiên

# parentbased: nếu parent span đã được sampled → con cũng sample
# traceidratio: sample ngẫu nhiên theo tỉ lệ
```

### Tail-based Sampling với OTel Collector

```yaml
# OTel Collector config với tail sampling processor
processors:
  tail_sampling:
    decision_wait: 10s      # đợi 10s trước khi quyết định sample
    num_traces: 50000       # giữ trong memory tối đa 50000 trace
    expected_new_traces_per_sec: 100
    policies:
      # Luôn giữ trace có lỗi
      - name: errors-policy
        type: status_code
        status_code:
          status_codes: [ERROR]

      # Luôn giữ trace latency > 1 giây
      - name: slow-traces-policy
        type: latency
        latency:
          threshold_ms: 1000

      # Giữ 5% trace bình thường (random sample)
      - name: random-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 5

      # Luôn giữ trace từ user VIP
      - name: vip-user-policy
        type: string_attribute
        string_attribute:
          key: user.tier
          values: [premium, enterprise]
```

---

## Correlate Traces với Logs

**Correlate (Liên Kết)** trace với log giúp đi từ metric spike → tìm trace liên quan → xem log chi tiết mà không cần đoán mò.

### Inject TraceID vào Log

```go
// Go — inject traceID vào log bằng zerolog
import (
    "go.opentelemetry.io/otel/trace"
    "github.com/rs/zerolog"
)

func logWithTrace(ctx context.Context, logger zerolog.Logger) zerolog.Logger {
    span := trace.SpanFromContext(ctx)
    if span.IsRecording() {
        spanCtx := span.SpanContext()
        return logger.With().
            Str("traceID", spanCtx.TraceID().String()).
            Str("spanID", spanCtx.SpanID().String()).
            Logger()
    }
    return logger
}

// Trong handler
func handleRequest(ctx context.Context, w http.ResponseWriter, r *http.Request) {
    log := logWithTrace(ctx, logger)
    log.Info().Str("method", r.Method).Msg("request received")
    // Log sẽ chứa: {"level":"info","traceID":"abc123def456...","spanID":"s001","method":"POST",...}
}
```

### Cấu Hình Grafana để Link Log → Trace

```yaml
# Grafana datasource Loki với derived field
# Khi click vào traceID trong log → nhảy sang Grafana Tempo
datasources:
  - name: Loki
    type: loki
    jsonData:
      derivedFields:
        - name: TraceID
          matcherRegex: '"traceID":"(\w+)"'
          url: "${__value.raw}"
          datasourceUid: tempo     # uid của Tempo datasource
          urlDisplayLabel: "View Trace in Tempo"

        # Cũng link từ log sang Jaeger nếu dùng Jaeger thay Tempo
        # - name: TraceID-Jaeger
        #   matcherRegex: '"traceID":"(\w+)"'
        #   url: "https://jaeger.example.com/trace/${__value.raw}"
        #   urlDisplayLabel: "View in Jaeger"
```

### Exemplars — Link Metric → Trace

**Exemplars** là tính năng Prometheus/Grafana cho phép đính kèm `traceID` vào một metric sample cụ thể. Khi thấy spike trên Prometheus graph, click vào điểm spike → nhảy trực tiếp sang trace tương ứng.

```go
// Trong Go — ghi exemplar vào histogram metric
histogram.With(prometheus.Labels{"status": "200"}).
    ObserveWithExemplar(
        duration.Seconds(),
        prometheus.Labels{"traceID": span.SpanContext().TraceID().String()},
    )
```

---

## Câu Hỏi Phỏng Vấn

**Phân biệt Span, Trace, TraceContext? Chúng liên quan nhau như thế nào?**

> **Trace** là toàn bộ hành trình của một request, bao gồm tất cả span liên quan, được nhận dạng bởi `traceID`. **Span** là một đơn vị công việc trong trace (một HTTP call, một DB query), có `spanID`, `parentSpanID` để xây dựng cây span, và `startTime`/`endTime` để tính latency. **TraceContext** (W3C standard) là cơ chế truyền `traceID` và `parentSpanID` qua HTTP header `traceparent` giữa các service — đây là "sợi dây" nối các span rời rạc thành một trace thống nhất. Ví dụ thực tế: `api-gateway` tạo span gốc (root span), thêm `traceparent` header vào request đến `order-service`; `order-service` đọc header này, tạo span con với `parentSpanID` = spanID của `api-gateway`.

**Head-based vs tail-based sampling — nên dùng cái nào?**

> **Head-based sampling** quyết định sample hay không ngay khi request bắt đầu, dựa trên tỉ lệ cố định hoặc rule đơn giản. Ưu điểm: đơn giản, không cần buffer, overhead thấp. Nhược điểm: có thể bỏ qua trace quan trọng (lỗi, latency cao) nếu chúng rơi vào phần không được sample. **Tail-based sampling** đợi đến khi trace hoàn thành mới quyết định — giữ 100% trace có lỗi hoặc latency cao, bỏ qua trace bình thường. Nhược điểm: cần buffer toàn bộ trace trong memory (tốn RAM), phức tạp hơn. Thực tế trong production: bắt đầu với head-based 5-10% kết hợp rule giữ 100% ERROR trace (có thể làm bằng cách set `AlwaysSample` trong SDK cho request lỗi). Dùng tail-based khi có yêu cầu cao hơn (SRE team muốn phân tích 100% slow request).

**OpenTelemetry thay thế Jaeger hay bổ sung?**

> OpenTelemetry là **instrumentation standard** (cách instrument code để generate trace, metric, log), còn Jaeger và Tempo là **backends** (nơi lưu trữ và query trace). OTel không thay thế Jaeger/Tempo mà giải quyết vấn đề vendor lock-in: trước OTel, mỗi backend có SDK riêng (Jaeger SDK, Zipkin SDK...) — đổi backend phải sửa code. Với OTel: instrument code một lần bằng OTel SDK, sau đó có thể export sang Jaeger, Tempo, Zipkin, Datadog, Honeycomb... chỉ cần thay cấu hình exporter. OTel Collector thêm một lớp nữa: nhận trace từ OTel SDK, transform, fan-out sang nhiều backend cùng lúc.

**Auto-instrumentation với OTel Operator hoạt động như thế nào?**

> OTel Operator watch Deployment có annotation `instrumentation.opentelemetry.io/inject-*`. Khi Pod mới được tạo, Operator's mutating webhook intercept Pod creation và inject: (1) **Init container** copy OTel agent (JAR file, Python package, Node.js module) vào shared volume; (2) **Environment variable** set JAVA_TOOL_OPTIONS, PYTHONPATH, NODE_OPTIONS để app tự load agent; (3) **OTEL_EXPORTER_OTLP_ENDPOINT** trỏ đến OTel Collector. App không cần bất kỳ OTel code nào — agent tự instrument HTTP client, gRPC, database driver... qua bytecode injection (Java) hoặc monkey patching (Python/Node). Hạn chế: không thể custom span attribute chi tiết; app cần restart khi thay đổi cấu hình instrumentation; một số framework đặc biệt không được auto-instrument tốt.

---

## Checklist Production

### Jaeger / Tempo

- [ ] Backend tracing được cài đặt và có persistent storage (S3, Elasticsearch) — không dùng in-memory
- [ ] OTel Collector deploy dạng DaemonSet (agent mode) hoặc Deployment (gateway mode)
- [ ] Trace retention được cấu hình phù hợp (7-30 ngày thường đủ)
- [ ] Grafana Tempo datasource được cấu hình và test thành công

### Instrumentation

- [ ] Ít nhất các service critical path đã được instrument (manual hoặc auto)
- [ ] TraceID được inject vào log của tất cả service đã instrument
- [ ] Service name được set chuẩn trong OTel resource (`service.name`)
- [ ] HTTP header propagation được test: trace connect qua service boundary

### Sampling

- [ ] Sampling rate không phải 100% (gây tốn storage)
- [ ] ERROR trace luôn được giữ (100% sample cho error)
- [ ] Slow trace (latency > threshold) luôn được giữ
- [ ] Sampling rate được monitor: không quá thấp (miss quan trọng) không quá cao (tốn tiền)

### Correlate

- [ ] Grafana datasource Loki có derived field TraceID link sang Tempo
- [ ] Prometheus exemplar được cấu hình cho histogram metric quan trọng
- [ ] Test end-to-end: từ Grafana metric spike → log → trace hoạt động
