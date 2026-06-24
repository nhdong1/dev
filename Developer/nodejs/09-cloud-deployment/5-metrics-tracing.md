# Metrics & Tracing — Prometheus, Grafana, OpenTelemetry

> Metrics (Chỉ Số) và Tracing (Theo Dõi Phân Tán) là hai pillars còn lại của observability — metrics cho biết **hệ thống đang như thế nào**, traces cho biết **request đi qua đâu và mất bao lâu**.

## Mục Lục

1. [Three Pillars of Observability](#three-pillars-of-observability)
2. [Prometheus Metrics](#prometheus-metrics)
3. [prom-client trong Node.js](#prom-client-trong-nodejs)
4. [Custom Business Metrics](#custom-business-metrics)
5. [Grafana Dashboards](#grafana-dashboards)
6. [OpenTelemetry — Unified Instrumentation](#opentelemetry--unified-instrumentation)
7. [Distributed Tracing với Jaeger](#distributed-tracing-với-jaeger)
8. [SLI, SLO và Alerting](#sli-slo-và-alerting)
9. [Production Setup](#production-setup)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Three Pillars of Observability

```
                    OBSERVABILITY
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
      LOGS           METRICS          TRACES
   "What happened"  "How is it"    "Where did it go"
         │               │               │
    Pino/Loki      Prometheus       OpenTelemetry
                   Grafana          Jaeger/Zipkin
```

| Pillar | Trả Lời Câu Hỏi | Ví Dụ |
| ------ | --------------- | ----- |
| **Logs** | Chuyện gì đã xảy ra? | `login failed for user X` |
| **Metrics** | Hệ thống healthy không? Trend? | `error_rate = 0.5%`, `p95_latency = 120ms` |
| **Traces** | Request đi qua services nào? Bottleneck ở đâu? | `API → Auth → DB: 45ms + 12ms + 89ms` |

Ba pillars bổ sung cho nhau — incident investigation thường bắt đầu từ alert (metrics), drill down traces, rồi đọc logs.

---

## Prometheus Metrics

Prometheus (Hệ Thống Giám Sát Mã Nguồn Mở) thu thập metrics theo mô hình **pull** — scrape (quét) endpoints định kỳ.

### Metric Types

| Type | Mô Tả | Ví Dụ Node.js |
| ---- | ----- | ------------- |
| **Counter** | Chỉ tăng, reset khi restart | `http_requests_total`, `errors_total` |
| **Gauge** | Tăng/giảm tự do | `active_connections`, `memory_usage_bytes` |
| **Histogram** | Phân phối values vào buckets | `http_request_duration_seconds` |
| **Summary** | Quantiles (p50, p95, p99) | Tương tự histogram, client-side |

```
Counter:     0 ──► 1 ──► 2 ──► 3 ──► ... (never decreases)
Gauge:       5 ──► 3 ──► 7 ──► 2 ──► ... (up and down)
Histogram:   requests grouped into buckets:
             le="0.1" → 800
             le="0.5" → 950
             le="1.0" → 990
             le="+Inf" → 1000
```

---

## prom-client trong Node.js

```typescript
import { Registry, Counter, Histogram, Gauge, collectDefaultMetrics } from 'prom-client';

const register = new Registry();

// Default Node.js metrics: heap, event loop, GC
collectDefaultMetrics({ register, prefix: 'nodejs_' });

// HTTP request counter
const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  registers: [register],
});

// Request duration histogram
const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
  registers: [register],
});

// Active connections gauge
const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
  registers: [register],
});

export { register, httpRequestsTotal, httpRequestDuration, activeConnections };
```

### Express Middleware

```typescript
import { httpRequestsTotal, httpRequestDuration } from './metrics';

export function metricsMiddleware(req: Request, res: Response, next: NextFunction) {
  const end = httpRequestDuration.startTimer();

  res.on('finish', () => {
    const route = req.route?.path || req.path;
    const labels = {
      method: req.method,
      route,
      status_code: res.statusCode.toString(),
    };

    httpRequestsTotal.inc(labels);
    end(labels);
  });

  next();
}

// Expose /metrics endpoint
app.get('/metrics', async (_req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

### Bảo Vệ /metrics Endpoint

```typescript
app.get('/metrics', (req, res, next) => {
  // Chỉ cho phép internal network
  const clientIp = req.ip;
  if (!isInternalIp(clientIp)) {
    return res.status(403).send('Forbidden');
  }
  next();
}, async (_req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

---

## Custom Business Metrics

```typescript
// Orders created
const ordersCreated = new Counter({
  name: 'orders_created_total',
  help: 'Total orders created',
  labelNames: ['payment_method'],
  registers: [register],
});

// Queue depth
const jobQueueDepth = new Gauge({
  name: 'job_queue_depth',
  help: 'Number of jobs waiting in queue',
  labelNames: ['queue_name'],
  registers: [register],
});

// Trong business logic
async function createOrder(order: Order) {
  const result = await orderRepository.save(order);
  ordersCreated.inc({ payment_method: order.paymentMethod });
  return result;
}
```

---

## Grafana Dashboards

### PromQL Queries Thường Dùng

```promql
# Request rate (requests per second)
rate(http_requests_total[5m])

# Error rate (%)
sum(rate(http_requests_total{status_code=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m])) * 100

# p95 latency
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)

# Event loop lag (từ default metrics)
nodejs_eventloop_lag_p99_seconds

# Memory usage
nodejs_heap_size_used_bytes / nodejs_heap_size_total_bytes * 100
```

### Dashboard Panels Khuyến Nghị

| Panel | Metric | Mục Đích |
| ----- | ------ | -------- |
| Request Rate | `rate(http_requests_total[5m])` | Traffic volume |
| Error Rate | 5xx / total | Service health |
| Latency p50/p95/p99 | histogram_quantile | User experience |
| Event Loop Lag | nodejs_eventloop_lag | Node.js specific health |
| Memory Usage | heap used / total | Memory leak detection |
| Active Connections | gauge | Connection pool health |

---

## OpenTelemetry — Unified Instrumentation

OpenTelemetry (OTel — Chuẩn Mở Cho Telemetry) cung cấp unified API cho traces, metrics, và logs.

```typescript
// instrumentation.ts — import TRƯỚC app code
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { Resource } from '@opentelemetry/resources';
import { ATTR_SERVICE_NAME } from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  resource: new Resource({
    [ATTR_SERVICE_NAME]: 'nodejs-api',
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://localhost:4318/v1/traces',
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-fs': { enabled: false },
    }),
  ],
});

sdk.start();
```

```bash
# Chạy app với auto-instrumentation
node --require ./instrumentation.js dist/main.js
```

Auto-instrumentation tự động trace: HTTP requests, Express routes, PostgreSQL queries, Redis calls.

### Manual Spans

```typescript
import { trace, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('order-service');

async function processOrder(orderId: string) {
  return tracer.startActiveSpan('processOrder', async (span) => {
    try {
      span.setAttribute('order.id', orderId);

      const order = await tracer.startActiveSpan('fetchOrder', async (childSpan) => {
        const result = await orderRepo.findById(orderId);
        childSpan.end();
        return result;
      });

      await paymentService.charge(order);
      span.setStatus({ code: SpanStatusCode.OK });
      return order;
    } catch (err) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: (err as Error).message });
      span.recordException(err as Error);
      throw err;
    } finally {
      span.end();
    }
  });
}
```

---

## Distributed Tracing với Jaeger

```
Client Request
      │
      ▼
┌─────────────┐  span: HTTP GET /orders/123  (120ms)
│  API Gateway│
└──────┬──────┘
       │
       ▼
┌─────────────┐  span: getOrder  (95ms)
│  Order Svc  │
└──────┬──────┘
       │
   ┌───┴───┐
   ▼       ▼
┌──────┐ ┌──────┐
│ DB   │ │Cache │  span: SELECT (45ms)  span: GET (2ms)
└──────┘ └──────┘
```

### Docker Compose — Local Stack

```yaml
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"   # Jaeger UI
      - "4318:4318"     # OTLP HTTP receiver

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
```

### Trace Context Propagation

OTel tự động propagate trace context qua HTTP headers (`traceparent`, `tracestate`). Khi gọi service khác:

```typescript
// Auto-instrumented HTTP client tự động inject headers
const response = await fetch('http://payment-service/charge', {
  method: 'POST',
  body: JSON.stringify(payment),
});
// Headers: traceparent: 00-abc123-def456-01
```

---

## SLI, SLO và Alerting

| Khái Niệm | Định Nghĩa | Ví Dụ |
| --------- | ---------- | ----- |
| **SLI** (Service Level Indicator — Chỉ Số Mức Dịch Vụ) | Metric đo chất lượng | Request latency p95, error rate |
| **SLO** (Service Level Objective — Mục Tiêu Mức Dịch Vụ) | Target cho SLI | p95 < 200ms, availability 99.9% |
| **SLA** (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ) | Contract với khách hàng | 99.9% uptime hoặc refund |
| **Error Budget** | 100% - SLO | 99.9% SLO = 0.1% error budget |

### Alert Rules (Prometheus)

```yaml
# alerts.yml
groups:
  - name: nodejs-api
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.01
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Error rate > 1% for 2 minutes"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          ) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "p95 latency > 500ms"

      - alert: EventLoopLag
        expr: nodejs_eventloop_lag_p99_seconds > 0.1
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "Event loop lag p99 > 100ms"
```

---

## Production Setup

```
┌─────────────────────────────────────────────────────────┐
│                  PRODUCTION OBSERVABILITY                │
│                                                         │
│  Node.js App                                            │
│  ├── /metrics ──► Prometheus (scrape 15s)               │
│  ├── stdout logs ──► Promtail ──► Loki                  │
│  └── OTel SDK ──► OTLP Collector ──► Jaeger             │
│                                                         │
│  Grafana ◄── Prometheus + Loki + Jaeger (datasources)   │
│  Alertmanager ◄── Prometheus alerts ──► PagerDuty/Slack │
└─────────────────────────────────────────────────────────┘
```

### Checklist

- [ ] `/metrics` endpoint với prom-client default + custom metrics
- [ ] HTTP request duration histogram với labels (method, route, status)
- [ ] OpenTelemetry auto-instrumentation cho HTTP và DB
- [ ] Grafana dashboard với RED metrics (Rate, Errors, Duration)
- [ ] Alert rules cho error rate, latency, event loop lag
- [ ] Correlation giữa traces và logs (traceId trong log fields)

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Đáp Án Ngắn |
| ------- | ----------- |
| Counter vs Gauge? | Counter chỉ tăng; Gauge tăng/giảm tự do |
| Histogram vs Summary? | Histogram: server-side quantiles; Summary: client-side quantiles |
| RED method là gì? | Rate, Errors, Duration — 3 metrics cốt lõi cho services |
| OpenTelemetry khác Prometheus? | OTel = instrumentation standard; Prometheus = metrics storage/query |
| Distributed tracing giải quyết gì? | Visualize request flow qua microservices, tìm bottleneck |
| SLI vs SLO vs SLA? | SLI = metric; SLO = target; SLA = contract với penalty |
