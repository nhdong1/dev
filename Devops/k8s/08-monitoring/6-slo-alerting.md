# SLO, SLI và AlertManager

> Hướng dẫn chi tiết về SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ), SLO (Service Level Objective — Mục Tiêu Mức Dịch Vụ), SLI (Service Level Indicator — Chỉ Số Mức Dịch Vụ), Error Budget (Ngân Sách Lỗi) và Burn Rate (Tốc Độ Tiêu Hao): định nghĩa SLI thực tế, tính toán error budget, cấu hình Prometheus alert rule cho SLO, và AlertManager architecture — route, inhibition, silencing, receiver configuration.

## Mục Lục

1. [SLA, SLO, SLI — Phân Biệt và Ví Dụ Thực Tế](#sla-slo-sli--phân-biệt-và-ví-dụ-thực-tế)
2. [Định Nghĩa SLI](#định-nghĩa-sli)
3. [Error Budget và Burn Rate](#error-budget-và-burn-rate)
4. [Prometheus Alert Rules cho SLO](#prometheus-alert-rules-cho-slo)
5. [AlertManager — Kiến Trúc và Cấu Hình](#alertmanager--kiến-trúc-và-cấu-hình)
6. [Receiver Configuration](#receiver-configuration)
7. [Runbook URL trong Alert](#runbook-url-trong-alert)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)
9. [Checklist Production AlertManager](#checklist-production-alertmanager)

---

## SLA, SLO, SLI — Phân Biệt và Ví Dụ Thực Tế

```
┌────────────────────────────────────────────────────────────────────┐
│                    SLA / SLO / SLI HIERARCHY                      │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ SLA — Service Level Agreement (Thỏa Thuận Mức Dịch Vụ)      │  │
│  │                                                              │  │
│  │ Hợp đồng pháp lý giữa provider và khách hàng.               │  │
│  │ Vi phạm SLA → hoàn tiền, phạt hợp đồng.                     │  │
│  │                                                              │  │
│  │ Ví dụ: "Chúng tôi cam kết uptime 99.9% mỗi tháng.           │  │
│  │         Nếu vi phạm, hoàn lại 10% cước tháng đó."           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │ chứa                                    │
│  ┌───────────────────────▼──────────────────────────────────────┐  │
│  │ SLO — Service Level Objective (Mục Tiêu Mức Dịch Vụ)        │  │
│  │                                                              │  │
│  │ Mục tiêu nội bộ, nghiêm ngặt hơn SLA để có buffer.          │  │
│  │ Vi phạm SLO → team engineering cần hành động.               │  │
│  │                                                              │  │
│  │ Ví dụ: "API availability ≥ 99.95% trong 30 ngày."           │  │
│  │         "P99 latency < 500ms."                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │ đo bằng                                 │
│  ┌───────────────────────▼──────────────────────────────────────┐  │
│  │ SLI — Service Level Indicator (Chỉ Số Mức Dịch Vụ)          │  │
│  │                                                              │  │
│  │ Metric đo lường thực tế để so sánh với SLO.                 │  │
│  │                                                              │  │
│  │ Ví dụ: "Tỉ lệ request trả về HTTP 2xx trong 5 phút."        │  │
│  │         "P99 latency của endpoint /api/orders."              │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

### Ví Dụ Thực Tế

| Khái Niệm | Ví Dụ API Gateway | Ví Dụ Payment Service |
| --------- | ----------------- | --------------------- |
| **SLA** | "99.9% uptime, vi phạm → credit" | "99.95% uptime, vi phạm → 20% refund" |
| **SLO** | "99.95% request thành công/tháng" | "99.99% transaction thành công/tháng" |
| **SLI** | `success_rate = HTTP 2xx / total request` | `success_tx_rate = commit / (commit + rollback)` |
| **Error Budget** | 0.05% × 43200 phút = 21.6 phút/tháng | 0.01% × 43200 phút = 4.3 phút/tháng |

---

## Định Nghĩa SLI

### SLI cho Availability (Tính Khả Dụng)

```promql
# SLI: tỉ lệ request thành công (status 2xx và 3xx)
# Không tính lỗi do client (4xx là lỗi người dùng, không phải server)

# Tử số: request thành công
sum(rate(http_requests_total{
  job="web-api",
  status!~"5.."
}[5m]))

# Mẫu số: tất cả request (trừ 429 Too Many Rate Limit nếu muốn)
sum(rate(http_requests_total{job="web-api"}[5m]))

# SLI ratio (0–1)
sum(rate(http_requests_total{job="web-api", status!~"5.."}[5m]))
/
sum(rate(http_requests_total{job="web-api"}[5m]))
```

### SLI cho Latency (Độ Trễ)

```promql
# SLI: tỉ lệ request có latency < threshold (ví dụ < 500ms)
sum(rate(http_request_duration_seconds_bucket{
  job="web-api",
  le="0.5"   # bucket ≤ 500ms
}[5m]))
/
sum(rate(http_request_duration_seconds_count{job="web-api"}[5m]))

# P99 latency tuyệt đối
histogram_quantile(0.99,
  sum by (le) (
    rate(http_request_duration_seconds_bucket{job="web-api"}[5m])
  )
)
```

### SLI cho Error Rate (Tỉ Lệ Lỗi)

```promql
# Error rate — tỉ lệ lỗi server-side
sum(rate(http_requests_total{job="web-api", status=~"5.."}[5m]))
/
sum(rate(http_requests_total{job="web-api"}[5m]))
```

### SLI cho Throughput (Thông Lượng)

```promql
# Request/giây hiện tại
sum(rate(http_requests_total{job="web-api"}[5m]))

# SLI throughput: tỉ lệ thời gian throughput đạt ngưỡng tối thiểu
# (ví dụ: ít nhất 100 req/s)
sum(rate(http_requests_total{job="web-api"}[5m])) >= 100
```

---

## Error Budget và Burn Rate

### Tính Error Budget (Ngân Sách Lỗi)

```
Công thức:
Error Budget = (1 - SLO target) × rolling window

Ví dụ:
SLO: 99.95% availability trong 30 ngày
Error Budget = (1 - 0.9995) × 30 × 24 × 60 phút
             = 0.0005 × 43200 phút
             = 21.6 phút downtime được phép trong 30 ngày

Hoặc tính bằng số request:
Nếu throughput là 1000 req/s:
Error Budget requests = 0.0005 × 1000 × 30 × 24 × 3600
                      = 0.0005 × 2,592,000,000
                      = 1,296,000 request lỗi được phép
```

### Burn Rate — Tốc Độ Tiêu Hao Error Budget

**Burn rate** (tốc độ tiêu hao) cho biết error budget đang bị tiêu thụ nhanh gấp bao nhiêu lần tốc độ bình thường.

```
Burn Rate = actual error rate / expected error rate
          = actual error rate / (1 - SLO)

Ví dụ:
SLO = 99.95% → expected error rate = 0.05%

Nếu actual error rate = 1%:
Burn rate = 1% / 0.05% = 20x

Burn rate 20x nghĩa là: budget 30 ngày sẽ hết trong 30/20 = 1.5 ngày
→ Đây là sự cố nghiêm trọng cần phản ứng ngay!

Nếu actual error rate = 0.1%:
Burn rate = 0.1% / 0.05% = 2x
→ Budget 30 ngày sẽ hết trong 30/2 = 15 ngày
→ Cần điều tra và fix trong vài giờ tới
```

### Alert Theo Burn Rate (Google SRE Pattern)

| Burn Rate | Budget tiêu hết sau | Severity | Hành Động |
| --------- | ------------------- | -------- | --------- |
| 14.4x | ~2 giờ | Critical | Gọi on-call ngay, incident response |
| 6x | ~5 giờ | Critical | Alert team, fix trong 1-2 giờ |
| 3x | ~10 giờ | Warning | Điều tra, fix trong vài giờ |
| 1x | = window | Info | Bình thường, không alert |

---

## Prometheus Alert Rules cho SLO

### Multi-window Multi-burn-rate Alert (Pattern Google SRE)

```yaml
# slo-alert-rules.yaml — PrometheusRule cho SLO alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: web-api-slo-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    # Recording Rules — tính SLI trước để alert nhanh
    - name: web_api_slo_recording
      interval: 30s
      rules:
        # Error rate 5 phút
        - record: job:http_error_rate:ratio_rate5m
          expr: |
            sum(rate(http_requests_total{job="web-api", status=~"5.."}[5m]))
            /
            sum(rate(http_requests_total{job="web-api"}[5m]))

        # Error rate 30 phút
        - record: job:http_error_rate:ratio_rate30m
          expr: |
            sum(rate(http_requests_total{job="web-api", status=~"5.."}[30m]))
            /
            sum(rate(http_requests_total{job="web-api"}[30m]))

        # Error rate 1 giờ
        - record: job:http_error_rate:ratio_rate1h
          expr: |
            sum(rate(http_requests_total{job="web-api", status=~"5.."}[1h]))
            /
            sum(rate(http_requests_total{job="web-api"}[1h]))

        # Error rate 6 giờ
        - record: job:http_error_rate:ratio_rate6h
          expr: |
            sum(rate(http_requests_total{job="web-api", status=~"5.."}[6h]))
            /
            sum(rate(http_requests_total{job="web-api"}[6h]))

    # Alert Rules — dùng recording rule đã tính sẵn
    - name: web_api_slo_alerts
      rules:
        # CRITICAL: Burn rate 14.4x — budget hết trong ~2 giờ
        # Cần phản ứng ngay: 5m + 1h window đều cao
        - alert: WebAPIErrorBudgetCritical
          expr: |
            job:http_error_rate:ratio_rate5m{job="web-api"} > (14.4 * 0.0005)
            and
            job:http_error_rate:ratio_rate1h{job="web-api"} > (14.4 * 0.0005)
          for: 2m
          labels:
            severity: critical
            slo: availability
            burn_rate: "14.4"
          annotations:
            summary: "SLO Critical: Web API error budget đang cạn kiệt"
            description: |
              Error rate 5m: {{ $value | humanizePercentage }}
              Burn rate 14.4x — error budget sẽ hết trong ~2 giờ.
              Hành động ngay!
            runbook_url: "https://wiki.example.com/runbooks/slo-critical"

        # CRITICAL: Burn rate 6x — budget hết trong ~5 giờ
        - alert: WebAPIErrorBudgetHigh
          expr: |
            job:http_error_rate:ratio_rate30m{job="web-api"} > (6 * 0.0005)
            and
            job:http_error_rate:ratio_rate6h{job="web-api"} > (6 * 0.0005)
          for: 15m
          labels:
            severity: critical
            slo: availability
            burn_rate: "6"
          annotations:
            summary: "SLO High: Web API error budget tiêu hao nhanh"
            description: |
              Burn rate 6x — error budget sẽ hết trong ~5 giờ.
              Điều tra và xử lý trong 1-2 giờ tới.
            runbook_url: "https://wiki.example.com/runbooks/slo-high"

        # WARNING: Burn rate 3x — budget hết trong ~10 giờ
        - alert: WebAPIErrorBudgetWarning
          expr: |
            job:http_error_rate:ratio_rate1h{job="web-api"} > (3 * 0.0005)
            and
            job:http_error_rate:ratio_rate6h{job="web-api"} > (3 * 0.0005)
          for: 1h
          labels:
            severity: warning
            slo: availability
            burn_rate: "3"
          annotations:
            summary: "SLO Warning: Web API error budget tiêu hao cao hơn bình thường"
            description: "Burn rate 3x — cần điều tra trong vài giờ tới."

        # Alert: Latency SLO vi phạm
        - alert: WebAPILatencySLO
          expr: |
            histogram_quantile(0.99,
              sum by (le) (
                rate(http_request_duration_seconds_bucket{job="web-api"}[5m])
              )
            ) > 0.5
          for: 10m
          labels:
            severity: warning
            slo: latency
          annotations:
            summary: "P99 latency vượt SLO 500ms"
            description: "P99 latency hiện tại: {{ $value | humanizeDuration }}"
```

---

## AlertManager — Kiến Trúc và Cấu Hình

```
┌──────────────────────────────────────────────────────────────────┐
│                  ALERTMANAGER ARCHITECTURE                       │
│                                                                  │
│  Prometheus / Grafana                                            │
│       │ fire alert                                               │
│       ▼                                                          │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                  ALERTMANAGER                              │  │
│  │                                                            │  │
│  │  ┌──────────────────────────────────────────────────────┐  │  │
│  │  │  DISPATCHER                                          │  │  │
│  │  │  ├── Deduplication (khử trùng lặp cùng alert)       │  │  │
│  │  │  ├── Grouping (gom alert cùng label thành 1 message) │  │  │
│  │  │  └── Route matching (tìm receiver phù hợp)           │  │  │
│  │  └──────────────────┬───────────────────────────────────┘  │  │
│  │                     │                                       │  │
│  │  ┌──────────────────▼───────────────────────────────────┐  │  │
│  │  │  INHIBITION (Ức Chế)                                 │  │  │
│  │  │  Nếu alert A đang fire → tắt alert B liên quan       │  │  │
│  │  │  Ví dụ: Node down → không gửi alert về Pod trên node │  │  │
│  │  └──────────────────┬───────────────────────────────────┘  │  │
│  │                     │                                       │  │
│  │  ┌──────────────────▼───────────────────────────────────┐  │  │
│  │  │  SILENCING (Tắt Tiếng)                               │  │  │
│  │  │  Tắt alert cụ thể trong khoảng thời gian            │  │  │
│  │  │  (maintenance window, planned downtime)              │  │  │
│  │  └──────────────────┬───────────────────────────────────┘  │  │
│  │                     │                                       │  │
│  │  ┌──────────────────▼───────────────────────────────────┐  │  │
│  │  │  RECEIVERS                                           │  │  │
│  │  │  Slack / PagerDuty / OpsGenie / Email / Webhook      │  │  │
│  │  └──────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### alertmanager.yaml — Cấu Hình Đầy Đủ

```yaml
# alertmanager.yaml
# Cấu hình cho AlertManager trong production

global:
  # Default resolve timeout — sau bao lâu alert được coi là resolved
  resolve_timeout: 5m

  # Slack webhook mặc định
  slack_api_url: "${SLACK_WEBHOOK_URL}"   # set qua Secret/env

route:
  # Receiver mặc định nếu không match rule nào
  receiver: slack-default

  # Label để gom alert thành group
  group_by: ["alertname", "cluster", "service", "namespace"]

  # Đợi 30s để gom các alert cùng group vào một message
  group_wait: 30s

  # Sau khi gửi group đầu tiên, gom alert mới thêm vào trong 5 phút
  group_interval: 5m

  # Nhắc lại alert đang active sau 4 giờ
  repeat_interval: 4h

  routes:
    # CRITICAL alert → PagerDuty + Slack #alerts-critical
    - receiver: pagerduty-and-slack-critical
      matchers:
        - name: severity
          value: critical
      group_wait: 10s             # critical cần phản ứng nhanh → group_wait ngắn hơn
      repeat_interval: 1h         # nhắc mỗi giờ cho critical
      continue: false             # không match các route sau

    # WARNING alert → Slack #alerts-warning (không disturb on-call)
    - receiver: slack-warning
      matchers:
        - name: severity
          value: warning
      group_wait: 2m
      repeat_interval: 6h

    # Alert của team database → Slack channel riêng của team
    - receiver: slack-database-team
      matchers:
        - name: team
          value: database
      continue: true   # true = tiếp tục match rule sau (gửi đến 2 nơi)

    # Alert về infrastructure → #infra channel
    - receiver: slack-infra
      matchers:
        - name: component
          matchType: "=~"
          value: "node|disk|network"

# Inhibition rules — tắt alert con khi alert cha đã fire
inhibit_rules:
  # Nếu Node down → tắt alert về Pod trên node đó
  - source_matchers:
      - name: alertname
        value: NodeDown
    target_matchers:
      - name: alertname
        value: PodCrashLooping
    equal: [cluster, node]

  # Nếu cluster-level critical alert → tắt service-level warning
  - source_matchers:
      - name: severity
        value: critical
    target_matchers:
      - name: severity
        value: warning
    equal: [cluster, namespace]
```

---

## Receiver Configuration

### Slack với Custom Template

```yaml
# alertmanager.yaml — receivers section
receivers:
  - name: slack-default
    slack_configs:
      - api_url: "${SLACK_WEBHOOK_URL_DEFAULT}"
        channel: "#alerts-general"
        send_resolved: true    # gửi thông báo khi alert resolved

        # Template tùy chỉnh — rõ ràng và có action
        title: |
          [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}]
          {{ .CommonLabels.alertname }} — {{ .CommonLabels.cluster }}

        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Severity:* {{ .Labels.severity | toUpper }}
          *Namespace:* {{ .Labels.namespace }}
          *Description:* {{ .Annotations.description }}
          {{ if .Annotations.runbook_url }}*Runbook:* <{{ .Annotations.runbook_url }}|Link>{{ end }}
          *Started:* {{ .StartsAt | since }}
          ---
          {{ end }}

        # Color theo severity
        color: |
          {{ if eq .Status "resolved" }}good
          {{ else if eq .CommonLabels.severity "critical" }}danger
          {{ else }}warning{{ end }}

        # Chỉ hiện nút action nếu có runbook URL
        actions:
          - type: button
            text: "View Runbook :book:"
            url: "{{ (index .Alerts 0).Annotations.runbook_url }}"
          - type: button
            text: "Silence 1h :mute:"
            url: '{{ template "silenceURL" . }}'

  - name: pagerduty-and-slack-critical
    pagerduty_configs:
      - routing_key: "${PAGERDUTY_INTEGRATION_KEY}"
        description: "{{ .CommonAnnotations.summary }}"
        severity: "{{ .CommonLabels.severity }}"
        client: "AlertManager"
        client_url: "http://alertmanager.example.com"
        details:
          cluster: "{{ .CommonLabels.cluster }}"
          namespace: "{{ .CommonLabels.namespace }}"
          description: "{{ .CommonAnnotations.description }}"
          runbook: "{{ .CommonAnnotations.runbook_url }}"
        # PagerDuty link — đính kèm link sang Grafana dashboard
        links:
          - href: "{{ .CommonAnnotations.runbook_url }}"
            text: Runbook
          - href: "https://grafana.example.com/d/k8s-overview"
            text: "Grafana Dashboard"

    slack_configs:
      - api_url: "${SLACK_WEBHOOK_URL_CRITICAL}"
        channel: "#alerts-critical"
        send_resolved: true
        title: ":red_circle: CRITICAL: {{ .CommonLabels.alertname }}"

  - name: slack-warning
    slack_configs:
      - api_url: "${SLACK_WEBHOOK_URL_WARNING}"
        channel: "#alerts-warning"
        send_resolved: true
        title: ":warning: WARNING: {{ .CommonLabels.alertname }}"

  - name: opsgenie-critical
    opsgenie_configs:
      - api_key: "${OPSGENIE_API_KEY}"
        api_url: "https://api.opsgenie.com/"
        message: "{{ .CommonAnnotations.summary }}"
        description: "{{ .CommonAnnotations.description }}"
        priority: |
          {{ if eq .CommonLabels.severity "critical" }}P1
          {{ else if eq .CommonLabels.severity "warning" }}P3
          {{ else }}P5{{ end }}
        tags: "kubernetes,{{ .CommonLabels.cluster }},{{ .CommonLabels.namespace }}"
        details:
          runbook: "{{ .CommonAnnotations.runbook_url }}"
```

### Cấu Hình qua Kubernetes Secret (AlertManager trong kube-prometheus-stack)

```yaml
# alertmanager-secret.yaml — lưu config AlertManager trong Secret
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-kube-prometheus-stack-alertmanager
  namespace: monitoring
type: Opaque
stringData:
  alertmanager.yaml: |
    # Nội dung alertmanager.yaml ở trên
    global:
      resolve_timeout: 5m
    route:
      ...
```

```bash
# Hoặc cập nhật AlertManager config không cần restart
kubectl create secret generic alertmanager-config \
  --from-file=alertmanager.yaml \
  --namespace monitoring \
  --dry-run=client -o yaml | kubectl apply -f -

# Kiểm tra config hiện tại (port-forward AlertManager UI)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093
# Mở http://localhost:9093/#/status
```

---

## Runbook URL trong Alert

**Runbook (Sổ Tay Xử Lý)** là tài liệu hướng dẫn on-call engineer cách xử lý alert cụ thể. Đính kèm runbook URL vào alert giảm thời gian MTTR (Mean Time To Recovery — Thời Gian Trung Bình Để Phục Hồi).

### Cấu Trúc Runbook Tốt

```markdown
# Runbook: WebAPIHighErrorRate

## Triệu Chứng
Alert fires khi error rate > 5% trong 5 phút.

## Tác Động
User nhận lỗi 500 khi dùng API.

## Điều Tra

### Bước 1: Xem log
Truy cập Grafana Loki:
- Query: `{namespace="production", app="web-api"} |= "ERROR" | json`
- Tìm error message phổ biến nhất

### Bước 2: Xem trace
- Lấy traceID từ log
- Mở Grafana Tempo để xem span breakdown

### Bước 3: Kiểm tra dependency
- Database: `kubectl exec -n production <db-pod> -- psql -c "SELECT * FROM pg_stat_activity"`
- External API: check status page của third-party service

## Xử Lý

### Nếu do DB overload:
- Scale replicas: `kubectl scale deployment web-api -n production --replicas=5`
- Hoặc enable circuit breaker nếu có

### Nếu do code bug sau deploy:
- Rollback: `kubectl rollout undo deployment/web-api -n production`

## Escalate
Nếu không xử lý được trong 30 phút → ping #oncall-l2
```

### Thêm Runbook URL vào PrometheusRule

```yaml
# Trong PrometheusRule annotations
annotations:
  summary: "Web API error rate vượt 5%"
  description: "Error rate: {{ $value | humanizePercentage }}"
  runbook_url: "https://wiki.example.com/runbooks/web-api-high-error-rate"
  # Link sang Grafana dashboard tương ứng
  dashboard_url: "https://grafana.example.com/d/web-api-main?var-namespace=production"
```

---

## Câu Hỏi Phỏng Vấn

**Phân biệt SLA, SLO, SLI? Tại sao SLO thường nghiêm ngặt hơn SLA?**

> **SLI (Service Level Indicator)** là metric đo thực tế (ví dụ: error rate, P99 latency). **SLO (Service Level Objective)** là mục tiêu nội bộ team engineering phải đạt được với SLI đó (ví dụ: error rate < 0.05%). **SLA (Service Level Agreement)** là hợp đồng pháp lý với khách hàng — vi phạm có hậu quả kinh tế (hoàn tiền, phạt). SLO thường nghiêm ngặt hơn SLA (ví dụ SLO 99.95% vs SLA 99.9%) để tạo **buffer**: nếu vi phạm SLO, team biết và xử lý trước khi vi phạm SLA đến mức khách hàng bị ảnh hưởng. SLO là cam kết nội bộ, SLA là cam kết bên ngoài. Vi phạm SLO là cơ hội học hỏi; vi phạm SLA là mất tiền và mất khách hàng.

**Error Budget là gì và nó thay đổi cách team làm việc như thế nào?**

> Error Budget = (1 - SLO) × time window = thời lượng lỗi cho phép. Ví dụ SLO 99.9%/tháng = 43.2 phút downtime được phép. Error budget thay đổi tư duy: thay vì "zero downtime" không thực tế, team có một ngân sách lỗi cụ thể để cân bằng giữa reliability và feature velocity. Khi budget còn nhiều → team có thể deploy nhiều, release nhanh hơn. Khi budget gần hết → team freeze release, tập trung reliability. Ứng dụng thực tế: SRE team ở Google dùng error budget như "thỏa thuận" với development team — nếu dev team làm giảm reliability (tiêu budget), SRE có quyền block release cho đến khi reliability recover.

**AlertManager inhibition khác silencing thế nào?**

> **Silencing** là tắt alert cụ thể trong khoảng thời gian định trước — dùng cho maintenance window (biết trước sẽ có downtime). Silencing là hành động chủ động, thủ công, có thời hạn. **Inhibition** là quy tắc tự động: khi alert A đang fire, tắt alert B liên quan — dùng để giảm noise khi có incident lớn. Ví dụ: Node down → inhibit alert CrashLoopBackOff của Pod trên node đó (vì đương nhiên Pod sẽ crash khi node down, không cần thêm alert). Inhibition là tự động, luôn active theo rule cố định. Khác nhau then chốt: silencing cần người tạo thủ công trước mỗi maintenance; inhibition là rule tĩnh được cấu hình một lần và áp dụng tự động.

**Multi-window multi-burn-rate alerting là gì? Tại sao tốt hơn threshold đơn giản?**

> Threshold đơn giản (ví dụ: alert khi error rate > 1%) có 2 vấn đề: (1) **False positive** — spike ngắn 30 giây vẫn trigger alert dù không ảnh hưởng nhiều đến SLO; (2) **Slow burn miss** — lỗi nhỏ liên tục (0.2% trong nhiều ngày) không trigger nhưng vẫn tiêu hết budget. Multi-window multi-burn-rate dùng nhiều cặp time window (5m+1h, 30m+6h) để detect cả fast burn (spike ngắn mạnh) lẫn slow burn (lỗi nhỏ kéo dài). Ví dụ: burn rate 14.4x phát hiện qua window 5m+1h (cả hai đều cao) → tránh false positive từ spike 30 giây vì window 1h không đủ cao; đồng thời burn rate 3x phát hiện qua window 1h+6h → phát hiện slow burn mà threshold đơn giản bỏ qua.

---

## Checklist Production AlertManager

### Cấu Hình Cơ Bản

- [ ] AlertManager có ít nhất 2 replica để tránh mất alert
- [ ] AlertManager config được version control (không chỉnh tay trên UI)
- [ ] Route tree được test với `amtool config routes test` cho các scenario alert
- [ ] `resolve_timeout` được set phù hợp (5m là thường đủ)
- [ ] `group_wait` đủ ngắn cho critical (10-30s) và dài hơn cho warning (1-2m)

### Receiver và Notification

- [ ] Slack webhook và PagerDuty key lưu trong Kubernetes Secret, không hardcode
- [ ] Notification template được test với alert thực — verify format đúng
- [ ] `send_resolved: true` được bật — on-call biết khi nào incident resolve
- [ ] Test end-to-end: tạo test alert và verify notification nhận được

### Inhibition và Silencing

- [ ] Inhibition rule cho Node down → Pod alert được cấu hình
- [ ] Maintenance window procedure được document — ai tạo silence, khi nào
- [ ] Inhibition rule được test — verify alert bị tắt đúng trong scenario

### SLO và Error Budget

- [ ] SLO được định nghĩa rõ ràng cho tất cả service quan trọng
- [ ] Recording rule tính error rate cho SLO được cấu hình và verify
- [ ] Multi-burn-rate alert (14.4x, 6x, 3x) được cấu hình theo pattern Google SRE
- [ ] Error budget dashboard trong Grafana hiển thị % budget còn lại theo thời gian
- [ ] Runbook URL được đặt trong annotation của mỗi alert rule
- [ ] Runbook tài liệu đầy đủ bước điều tra và xử lý cho từng alert
