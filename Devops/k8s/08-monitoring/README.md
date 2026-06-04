# Monitoring & Observability — Giám Sát và Quan Sát Kubernetes

> Tổng quan về hệ thống giám sát (Monitoring) và quan sát (Observability) trong Kubernetes: từ 3 trụ cột Metrics (số liệu), Logs (nhật ký), Traces (vết theo dõi phân tán) đến toàn bộ stack Prometheus — Grafana — Loki — Jaeger — AlertManager trong môi trường production.

## Mục Lục

1. [Ba Trụ Cột Observability](#ba-trụ-cột-observability)
2. [Kiến Trúc Tổng Thể Monitoring Stack](#kiến-trúc-tổng-thể-monitoring-stack)
3. [Bản Đồ Quyết Định](#bản-đồ-quyết-định)
4. [Các Thành Phần Chính](#các-thành-phần-chính)
5. [So Sánh Các Giải Pháp Monitoring](#so-sánh-các-giải-pháp-monitoring)
6. [Ma Trận Tình Huống vs Giải Pháp](#ma-trận-tình-huống-vs-giải-pháp)
7. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)
8. [Checklist Production Monitoring](#checklist-production-monitoring)

---

## Ba Trụ Cột Observability

**Observability (Quan Sát)** là khả năng hiểu trạng thái bên trong hệ thống từ dữ liệu đầu ra mà không cần biết trước lỗi sẽ xảy ra ở đâu. Ba trụ cột tạo nên observability đầy đủ:

```
┌─────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY PILLARS                        │
│                   (Ba Trụ Cột Quan Sát)                        │
│                                                                 │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────────┐  │
│  │   METRICS   │   │    LOGS     │   │       TRACES        │  │
│  │  (Số Liệu)  │   │ (Nhật Ký)  │   │  (Vết Theo Dõi)     │  │
│  │             │   │             │   │                     │  │
│  │ cpu=80%     │   │ ERROR: conn │   │ req → svc-A → DB    │  │
│  │ req/s=1200  │   │   refused   │   │  [50ms] [120ms]     │  │
│  │ error=0.2%  │   │ INFO: done  │   │ TraceID: abc123     │  │
│  │             │   │             │   │                     │  │
│  │ Prometheus  │   │    Loki     │   │  Jaeger / Tempo     │  │
│  │  Datadog    │   │  ELK Stack  │   │  OpenTelemetry      │  │
│  └─────────────┘   └─────────────┘   └─────────────────────┘  │
│                                                                 │
│  "Hệ thống đang như thế nào?"                                  │
│  "Chuyện gì đã xảy ra?"                                        │
│  "Tại sao request này bị chậm?"                                │
└─────────────────────────────────────────────────────────────────┘
```

### Metrics (Số Liệu)

**Metrics** là dữ liệu số được thu thập theo thời gian (time series — chuỗi thời gian). Đặc điểm:
- **Hiệu quả lưu trữ:** Chỉ lưu số, không lưu text → hàng triệu metric/giây vẫn quản lý được
- **Dễ alert:** Đặt ngưỡng cố định, tính toán tỉ lệ, dự báo xu hướng
- **Hạn chế:** Không có context — biết CPU=90% nhưng không biết *tại sao*
- **Dùng khi:** Monitor health tổng thể, đặt SLO (Service Level Objective — Mục Tiêu Mức Dịch Vụ), capacity planning

### Logs (Nhật Ký)

**Logs** là bản ghi sự kiện xảy ra trong hệ thống. Đặc điểm:
- **Giàu context:** Chứa thông tin chi tiết về từng sự kiện, stack trace, request ID
- **Debug hiệu quả:** Tìm root cause khi biết thời điểm xảy ra lỗi
- **Hạn chế:** Tốn storage, khó aggregate thành insight tổng thể, tìm kiếm chậm nếu không index tốt
- **Dùng khi:** Debug lỗi cụ thể, audit trail, phân tích event sequence

### Traces (Vết Theo Dõi Phân Tán)

**Distributed Tracing (Theo Dõi Phân Tán)** theo dõi hành trình của một request qua nhiều service. Đặc điểm:
- **End-to-end visibility:** Xem request đi qua service nào, mỗi bước mất bao lâu
- **Tìm bottleneck:** Phát hiện service nào gây latency cao trong chuỗi call
- **Hạn chế:** Cần instrument (cài đặt mã theo dõi) vào code hoặc dùng auto-instrumentation, overhead nhất định
- **Dùng khi:** Microservice architecture, debug latency, hiểu dependency giữa service

---

## Kiến Trúc Tổng Thể Monitoring Stack

```
┌─────────────────────────────────────────────────────────────────────┐
│                     KUBERNETES CLUSTER                              │
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │  App A   │  │  App B   │  │  App C   │  │   System Layer   │   │
│  │ /metrics │  │ /metrics │  │ /metrics │  │ Node / kubelet   │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────┬──────────┘   │
│       │              │              │                │               │
│       └──────────────┴──────────────┴────────────────┘               │
│                             │ scrape (pull)                         │
│                             ▼                                        │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    COLLECT LAYER                             │   │
│  │                                                              │   │
│  │  ┌────────────┐  ┌──────────────┐  ┌─────────────────────┐  │   │
│  │  │ Prometheus │  │   Promtail   │  │   OTel Collector    │  │   │
│  │  │  (metric)  │  │    (log)     │  │     (trace)         │  │   │
│  │  └─────┬──────┘  └──────┬───────┘  └──────────┬──────────┘  │   │
│  └────────┼────────────────┼─────────────────────┼─────────────┘   │
│           │                │                      │                 │
│           ▼                ▼                      ▼                 │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    STORE LAYER                               │   │
│  │                                                              │   │
│  │  ┌────────────┐  ┌────────────┐  ┌───────────────────────┐  │   │
│  │  │  Thanos /  │  │    Loki    │  │  Jaeger / Tempo       │  │   │
│  │  │   Mimir    │  │  (log DB)  │  │  (trace backend)      │  │   │
│  │  │ (long-term)│  └────────────┘  └───────────────────────┘  │   │
│  │  └────────────┘                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│           │                │                      │                 │
│           └────────────────┴──────────────────────┘                 │
│                             │                                        │
│                             ▼                                        │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                   VISUALIZE & ALERT                          │   │
│  │                                                              │   │
│  │              ┌──────────────────┐                           │   │
│  │              │     Grafana      │                           │   │
│  │              │ (dashboard, alert│                           │   │
│  │              │  unified UI)     │                           │   │
│  │              └────────┬─────────┘                           │   │
│  │                       │                                      │   │
│  │              ┌────────┴─────────┐                           │   │
│  │              │  AlertManager    │                           │   │
│  │              │ (route, dedup,   │                           │   │
│  │              │  silence)        │                           │   │
│  │              └────────┬─────────┘                           │   │
│  └───────────────────────┼─────────────────────────────────────┘   │
└───────────────────────────┼─────────────────────────────────────────┘
                            │
                ┌───────────┴───────────┐
                │                       │
         ┌──────▼──────┐        ┌───────▼──────┐
         │    Slack    │        │  PagerDuty   │
         │  (notify)   │        │  (on-call)   │
         └─────────────┘        └──────────────┘
```

---

## Bản Đồ Quyết Định

```
Hệ thống có vấn đề — bắt đầu điều tra từ đâu?
│
├── Câu hỏi: "Hệ thống có đang bình thường không?"
│   └── → Dùng METRICS (Prometheus + Grafana)
│       ├── CPU, memory, error rate, latency tổng thể
│       ├── SLO dashboard để xem error budget còn bao nhiêu
│       └── Alert khi ngưỡng bị vượt
│
├── Câu hỏi: "Chuyện gì đã xảy ra lúc 2:00 sáng?"
│   └── → Dùng LOGS (Loki + Grafana)
│       ├── Filter theo thời gian + namespace + pod
│       ├── Tìm ERROR, WARN, stack trace
│       └── Correlate với metric spike cùng thời điểm
│
├── Câu hỏi: "Tại sao request này mất 3 giây?"
│   └── → Dùng TRACES (Jaeger / Tempo)
│       ├── Tìm trace theo TraceID hoặc khoảng thời gian
│       ├── Xem breakdown latency qua từng service
│       └── Tìm span nào chiếm thời gian nhiều nhất
│
├── Câu hỏi: "Khi nào thì hết tài nguyên?"
│   └── → Dùng METRICS với capacity planning query
│       ├── predict_linear() trong PromQL
│       └── Grafana trending dashboard
│
└── Câu hỏi: "Ai đã thay đổi gì lúc deploy?"
    └── → Dùng LOGS (audit log) + Deployment event
        ├── kube-apiserver audit log
        └── Loki filter theo user, resource, verb
```

---

## Các Thành Phần Chính

### Prometheus — Thu Thập Metric

**Prometheus** là hệ thống monitoring time-series mã nguồn mở, sử dụng mô hình **pull-based** (chủ động đến lấy metric từ target). Prometheus scrape metric từ endpoint `/metrics` mỗi 15–60 giây.

```
Prometheus TSDB (Time Series Database — Cơ Sở Dữ Liệu Chuỗi Thời Gian)
├── Lưu metric dạng: metric_name{label1="val1", label2="val2"} value timestamp
├── Block-based storage trên disk
├── Mặc định retention: 15 ngày
└── Query bằng PromQL (Prometheus Query Language)
```

Tham khảo chi tiết: [1-prometheus-setup.md](./1-prometheus-setup.md)

---

### Grafana — Visualize và Alert

**Grafana** là nền tảng visualize (hiển thị trực quan) mã nguồn mở, kết nối với nhiều datasource (Prometheus, Loki, Tempo, Elasticsearch...) và hiển thị trên cùng một giao diện.

```
Grafana Features:
├── Dashboard với nhiều panel (graph, table, stat, heatmap...)
├── Unified Alerting — alert từ bất kỳ datasource nào
├── Annotation — đánh dấu sự kiện deploy lên timeline
├── Template Variable — lọc theo namespace, pod, node...
└── Dashboard as Code — quản lý dashboard qua JSON/ConfigMap
```

Tham khảo chi tiết: [2-grafana-dashboards.md](./2-grafana-dashboards.md)

---

### Loki — Tổng Hợp Log

**Loki** là hệ thống tổng hợp log (log aggregation) được Grafana Labs phát triển, lấy cảm hứng từ Prometheus. Khác với ELK Stack, Loki **không index nội dung log** mà chỉ index label → lưu trữ rẻ hơn nhiều.

```
Loki Architecture:
├── Promtail / Fluentd / OTel Collector → thu thập log từ Pod
├── Distributor → nhận và phân phối log đến Ingester
├── Ingester → ghi log vào storage backend (S3, GCS...)
├── Querier → xử lý query LogQL
└── Ruler → evaluate alert rule từ log
```

Tham khảo chi tiết: [3-loki-logging.md](./3-loki-logging.md)

---

### Jaeger / Tempo — Distributed Tracing

**Jaeger** là hệ thống distributed tracing (theo dõi phân tán) mã nguồn mở từ Uber. **Grafana Tempo** là backend tracing nhẹ hơn, tích hợp tốt với Loki và Prometheus trong Grafana ecosystem.

```
Tracing Flow:
App (instrumented) → OTel Collector → Jaeger / Tempo
                                           │
                                     Query từ Grafana
                                     hoặc Jaeger UI
```

Tham khảo chi tiết: [4-tracing.md](./4-tracing.md)

---

### kube-state-metrics và metrics-server

- **metrics-server:** Cung cấp metric real-time CPU/memory cho HPA và `kubectl top`. Không lưu trữ lịch sử.
- **kube-state-metrics:** Cung cấp metric về trạng thái Kubernetes object (Deployment, Pod, Node...) từ kube-apiserver. Prometheus scrape rồi lưu.
- **cAdvisor:** Tích hợp trong kubelet, cung cấp resource metric ở cấp container.

Tham khảo chi tiết: [5-kube-state-metrics.md](./5-kube-state-metrics.md)

---

### AlertManager — Quản Lý Cảnh Báo

**AlertManager** nhận alert từ Prometheus (hoặc Grafana), xử lý deduplication (khử trùng lặp), grouping (nhóm), silencing (tắt tạm thời), inhibition (ức chế) rồi route đến receiver (Slack, PagerDuty, email...).

Tham khảo chi tiết: [6-slo-alerting.md](./6-slo-alerting.md)

---

## So Sánh Các Giải Pháp Monitoring

| Tiêu Chí | Prometheus + Grafana | Datadog | ELK Stack |
| -------- | ------------------- | ------- | --------- |
| **Mô hình** | Self-hosted, pull-based | SaaS, agent-based | Self-hosted, push-based |
| **Chi phí** | Miễn phí (infrastructure cost) | Đắt tiền theo host/metric | Infrastructure cost + Elasticsearch |
| **Metrics** | Rất mạnh, PromQL linh hoạt | Rất mạnh, UI thân thiện | Có nhưng không phải thế mạnh |
| **Logs** | Cần Loki thêm vào | Tích hợp sẵn | Rất mạnh (Elasticsearch full-text) |
| **Traces** | Cần Tempo/Jaeger | Tích hợp sẵn APM | Cần Elastic APM thêm vào |
| **Alert** | AlertManager + Grafana | Tích hợp sẵn | Watcher (tính năng) |
| **Kubernetes tích hợp** | Native, kube-prometheus-stack | Agent cài DaemonSet | Beats/Filebeat |
| **Khả năng scale** | Cần Thanos/Mimir cho multi-cluster | Auto-scale (SaaS) | Cần cluster Elasticsearch |
| **Học** | Cần học PromQL, LogQL | UI dễ dùng hơn | Cần học Lucene query |
| **Phù hợp** | Team có kỹ năng, budget hạn chế | Enterprise, muốn all-in-one | Log-heavy workload, full-text search |

> **Lựa chọn phổ biến nhất trong Kubernetes production:** kube-prometheus-stack (Prometheus + Grafana + AlertManager) + Loki + Tempo/Jaeger. Đây là "LGTM Stack" (Loki, Grafana, Tempo, Mimir) của Grafana Labs.

---

## Ma Trận Tình Huống vs Giải Pháp

| Tình Huống | Dấu Hiệu | Công Cụ | Giải Pháp | File |
| ---------- | --------- | ------- | --------- | ---- |
| API response time tăng đột ngột | P99 latency > 2s | Prometheus + Grafana | Xem latency histogram, tìm service bị ảnh hưởng | [1-prometheus-setup.md](./1-prometheus-setup.md) |
| Pod OOMKilled lặp lại | container restart count tăng | kube-state-metrics | Xem memory metric, điều chỉnh limit | [5-kube-state-metrics.md](./5-kube-state-metrics.md) |
| Không biết tại sao request chậm | Metric bình thường nhưng user phàn nàn | Jaeger / Tempo | Tìm trace, xem span breakdown | [4-tracing.md](./4-tracing.md) |
| Tìm lỗi sau sự cố đêm qua | Lỗi đã xảy ra, cần forensic | Loki | Query log theo timestamp + error level | [3-loki-logging.md](./3-loki-logging.md) |
| Alert quá nhiều, team bị alert fatigue | Slack bị spam alert | AlertManager | Grouping, silencing, inhibition rules | [6-slo-alerting.md](./6-slo-alerting.md) |
| Không biết SLO đang ở mức nào | Khách hàng hỏi uptime | Prometheus + Error Budget | Tính error rate, so với SLO target | [6-slo-alerting.md](./6-slo-alerting.md) |
| Dashboard không có dữ liệu | Grafana hiển thị "No data" | Grafana + Prometheus | Kiểm tra datasource, scrape config, label | [2-grafana-dashboards.md](./2-grafana-dashboards.md) |
| Node đang dùng bao nhiêu tài nguyên | Không biết capacity còn bao nhiêu | kube-state-metrics + Prometheus | Node resource dashboard | [5-kube-state-metrics.md](./5-kube-state-metrics.md) |
| Muốn alert khi deployment fail | Rollout stuck | kube-state-metrics alert | Alert trên kube_deployment_status_condition | [5-kube-state-metrics.md](./5-kube-state-metrics.md) |
| Log từ nhiều Pod bị phân tán | Phải vào từng Pod xem log | Loki + Promtail | Aggregate log, query tập trung | [3-loki-logging.md](./3-loki-logging.md) |
| Muốn correlate log với metric | Thấy spike metric, muốn xem log cùng thời điểm | Grafana (Loki + Prometheus) | Explore mode, Grafana annotation | [2-grafana-dashboards.md](./2-grafana-dashboards.md) |
| Multi-cluster monitoring | Nhiều cluster cần xem chung | Thanos / Mimir | Remote write, global query view | [1-prometheus-setup.md](./1-prometheus-setup.md) |

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu Hỏi Cơ Bản

**Observability khác monitoring thế nào?**

> **Monitoring (Giám Sát)** là theo dõi các metric và alert đã biết trước — bạn biết mình cần xem gì. **Observability (Quan Sát)** là khả năng hỏi bất kỳ câu hỏi nào về hệ thống dựa trên dữ liệu đầu ra — kể cả câu hỏi chưa từng nghĩ đến. Hệ thống có observability cao cho phép debug lỗi mới mà không cần deploy thêm code monitoring. Trong thực tế: monitoring tốt nhưng observability kém là biết CPU=90% nhưng không biết tại sao; observability đầy đủ là trace được request từ frontend qua 5 microservice đến DB và biết chính xác query nào chiếm 80% thời gian.

**Tại sao Prometheus dùng pull model thay vì push model?**

> Pull model có nhiều ưu điểm trong Kubernetes: (1) **Service discovery tự động** — Prometheus tự phát hiện target mới qua Kubernetes API thay vì target phải biết địa chỉ Prometheus; (2) **Kiểm soát scrape rate** — Prometheus kiểm soát tần suất lấy data, không bị target flood dữ liệu; (3) **Phát hiện target down** — nếu target không trả lời scrape, Prometheus biết ngay target đó down (push model không biết); (4) **Đơn giản hóa firewall** — chỉ cần Prometheus truy cập được target, không cần target biết Prometheus ở đâu. Hạn chế: không phù hợp cho job ngắn hạn (chạy rồi biến mất trước khi được scrape) → dùng Pushgateway cho trường hợp này.

**3 loại metric của Prometheus là gì?**

> (1) **Counter** — chỉ tăng, không giảm (ví dụ: tổng số request, tổng số lỗi). Query bằng `rate()` để tính tốc độ tăng. (2) **Gauge** — tăng hoặc giảm (ví dụ: CPU usage, số Pod hiện tại, queue size). Query trực tiếp để lấy giá trị hiện tại. (3) **Histogram** — ghi phân phối giá trị theo bucket (ví dụ: latency phân phối vào bucket <100ms, <500ms, <1s...). Dùng `histogram_quantile()` để tính P50, P95, P99 latency. (4) **Summary** — tương tự histogram nhưng tính quantile ở phía client thay vì server → ít linh hoạt hơn, khó aggregate nhiều instance.

### Câu Hỏi Nâng Cao

**Khi nào dùng metric, khi nào dùng log, khi nào dùng trace?**

> Ba nguồn dữ liệu bổ trợ nhau, không thay thế nhau. **Metric** là điểm khởi đầu — alert khi metric vượt ngưỡng, dashboard để xem trend, SLO để đo chất lượng dịch vụ. Khi metric báo hiệu có vấn đề, **log** giúp đào sâu hơn — tìm chính xác error message, stack trace, context của lỗi. Khi biết lỗi xảy ra nhưng không hiểu tại sao (latency cao nhưng không có error log), **trace** giúp xem request đi qua service nào và bước nào tốn thời gian. Workflow lý tưởng: Alert từ metric → xem log cùng thời điểm → trace để hiểu root cause. Grafana hỗ trợ workflow này bằng cách liên kết metric spike với log và trace trong cùng giao diện.

**Prometheus có thể scale như thế nào cho hệ thống lớn?**

> Prometheus đơn lẻ bị giới hạn bởi RAM và disk của một node — thường phù hợp cho cluster nhỏ đến trung bình (~1 triệu active time series). Để scale lên: (1) **Federation** — Prometheus cấp cao scrape aggregate từ các Prometheus cấp thấp, nhưng phức tạp và không giải quyết hoàn toàn; (2) **Thanos** — operator thêm long-term storage (S3/GCS), global query view nhiều cluster, deduplication. Thêm sidecar vào Prometheus pod, không cần thay đổi cách Prometheus hoạt động; (3) **Grafana Mimir** — giải pháp mới hơn, horizontally scalable, compatible với Prometheus API, thiết kế cho multi-tenant. Trong production lớn, Thanos hoặc Mimir là lựa chọn tiêu chuẩn.

---

## Checklist Production Monitoring

### Metrics Collection

- [ ] kube-prometheus-stack đã cài đặt và chạy ổn định
- [ ] Tất cả Node đều có Node Exporter scrape thành công
- [ ] kube-state-metrics đang báo cáo trạng thái Deployment, Pod, Node
- [ ] metrics-server cài đặt và `kubectl top` hoạt động
- [ ] ServiceMonitor hoặc PodMonitor đã cấu hình cho tất cả app quan trọng
- [ ] Prometheus retention được set phù hợp (15-30 ngày) hoặc dùng Thanos cho long-term

### Logging

- [ ] Promtail (hoặc Fluentd) deploy trên tất cả Node dạng DaemonSet
- [ ] Loki đang nhận log từ tất cả namespace production
- [ ] Log retention được cấu hình phù hợp (30-90 ngày)
- [ ] Structured logging (JSON format) được áp dụng cho tất cả app
- [ ] Index label tối thiểu: namespace, pod, container, level

### Tracing

- [ ] OpenTelemetry Collector deploy trong cluster
- [ ] Ít nhất service quan trọng nhất đã được instrument
- [ ] Trace sampling được cấu hình hợp lý (không 100% để tiết kiệm)
- [ ] TraceID được đưa vào log để correlate trace-log

### Alerting

- [ ] AlertManager đã cấu hình route đến Slack/PagerDuty
- [ ] Alert rule cơ bản: node down, pod CrashLooping, high error rate, disk full
- [ ] Runbook URL được thêm vào mỗi alert
- [ ] Test alert đã được kiểm tra end-to-end
- [ ] Silencing và inhibition được cấu hình để giảm noise

### Dashboard

- [ ] Grafana đã import các dashboard chuẩn (Node Exporter, K8s Overview)
- [ ] Dashboard SLO/Error Budget có cho các service quan trọng
- [ ] Datasource Prometheus và Loki đã cấu hình và test thành công
- [ ] Grafana authentication được bảo mật (OIDC hoặc ít nhất basic auth mạnh)

---

**Tài Liệu Liên Quan:**

| File | Nội Dung |
| ---- | -------- |
| [1-prometheus-setup.md](./1-prometheus-setup.md) | Cài đặt Prometheus, ServiceMonitor, PromQL, Recording Rules, Remote Write |
| [2-grafana-dashboards.md](./2-grafana-dashboards.md) | Grafana datasource, dashboard as code, alert rule, notification channel |
| [3-loki-logging.md](./3-loki-logging.md) | Loki architecture, Promtail, LogQL, multi-tenancy, storage backend |
| [4-tracing.md](./4-tracing.md) | OpenTelemetry, Jaeger, Tempo, auto-instrumentation, sampling strategy |
| [5-kube-state-metrics.md](./5-kube-state-metrics.md) | cAdvisor vs metrics-server vs kube-state-metrics, important metric list |
| [6-slo-alerting.md](./6-slo-alerting.md) | SLA/SLO/SLI, Error Budget, burn rate, AlertManager config, receiver |
