# Grafana — Dashboard và Alert

> Hướng dẫn chi tiết về Grafana (nền tảng visualize và alert mã nguồn mở): cài đặt và cấu hình datasource, quản lý dashboard as code với ConfigMap và Grafana Operator, các dashboard quan trọng cho Kubernetes, cấu hình Unified Alerting, notification channel (Slack, PagerDuty, email), và template variable để lọc theo namespace/pod/node.

## Mục Lục

1. [Kiến Trúc Grafana](#kiến-trúc-grafana)
2. [Cài Đặt và Cấu Hình Datasource](#cài-đặt-và-cấu-hình-datasource)
3. [Dashboard as Code](#dashboard-as-code)
4. [Dashboard Quan Trọng cho Kubernetes](#dashboard-quan-trọng-cho-kubernetes)
5. [Template Variable — Biến Lọc Dashboard](#template-variable--biến-lọc-dashboard)
6. [Unified Alerting — Alert Rule trong Grafana](#unified-alerting--alert-rule-trong-grafana)
7. [Notification Channel — Kênh Thông Báo](#notification-channel--kênh-thông-báo)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)
9. [Checklist Production](#checklist-production)

---

## Kiến Trúc Grafana

```
┌──────────────────────────────────────────────────────────────────┐
│                     GRAFANA ARCHITECTURE                        │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    DATASOURCES                             │  │
│  │                                                            │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │  │
│  │  │Prometheus│  │   Loki   │  │  Tempo   │  │Elasticsearch│  │
│  │  │(metrics) │  │  (logs)  │  │ (traces) │  │  (logs)   │  │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬──────┘  │  │
│  └───────┼─────────────┼─────────────┼─────────────┼──────────┘  │
│          └─────────────┴─────────────┴─────────────┘             │
│                                │ query                            │
│                                ▼                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    GRAFANA CORE                            │  │
│  │                                                            │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │  │
│  │  │  Dashboards  │  │   Alerting   │  │  Explore UI  │    │  │
│  │  │  (panels,    │  │  (rules,     │  │  (ad-hoc     │    │  │
│  │  │   variables) │  │   contacts)  │  │   query)     │    │  │
│  │  └──────────────┘  └──────┬───────┘  └──────────────┘    │  │
│  └─────────────────────────────┼──────────────────────────────┘  │
│                                │ alert fire                       │
│                                ▼                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │               NOTIFICATION CHANNELS                       │  │
│  │                                                            │  │
│  │      ┌──────────┐  ┌───────────┐  ┌──────────────┐       │  │
│  │      │  Slack   │  │ PagerDuty │  │    Email     │       │  │
│  │      └──────────┘  └───────────┘  └──────────────┘       │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Cài Đặt và Cấu Hình Datasource

### Cài Đặt Grafana Độc Lập (không qua kube-prometheus-stack)

```bash
# Thêm Helm repo Grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Cài Grafana
helm install grafana grafana/grafana \
  --namespace monitoring \
  --values grafana-values.yaml
```

### grafana-values.yaml Đầy Đủ

```yaml
# grafana-values.yaml

# Admin credentials — dùng Secret trong production
adminUser: admin
adminPassword: ""   # để trống và dùng existingSecret thay vì hardcode

# Hoặc dùng Secret có sẵn
admin:
  existingSecret: grafana-admin-secret
  userKey: admin-user
  passwordKey: admin-password

# Persistence — lưu dashboard, datasource, config qua restart
persistence:
  enabled: true
  storageClassName: gp3
  size: 10Gi

# Resource
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Grafana.ini — cấu hình chính
grafana.ini:
  server:
    root_url: https://grafana.example.com
  auth.generic_oauth:
    enabled: true
    name: Google
    client_id: ${GOOGLE_CLIENT_ID}
    client_secret: ${GOOGLE_CLIENT_SECRET}
    scopes: openid email profile
    auth_url: https://accounts.google.com/o/oauth2/auth
    token_url: https://oauth2.googleapis.com/token
    api_url: https://www.googleapis.com/oauth2/v1/userinfo
  analytics:
    reporting_enabled: false   # không gửi telemetry về Grafana Labs
  dashboards:
    default_home_dashboard_path: /var/lib/grafana/dashboards/overview.json

# Datasource tự động cấu hình
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090
        isDefault: true
        jsonData:
          timeInterval: 30s
          queryTimeout: 60s
          httpMethod: POST    # POST nhanh hơn GET cho query dài

      - name: Loki
        type: loki
        url: http://loki-gateway.monitoring.svc.cluster.local
        jsonData:
          maxLines: 1000
          derivedFields:
            # Tự động tạo link từ traceID trong log sang Tempo
            - datasourceUid: tempo
              matcherRegex: '"traceID":"(\w+)"'
              name: TraceID
              url: '$${__value.raw}'

      - name: Tempo
        type: tempo
        url: http://tempo.monitoring.svc.cluster.local:3100
        uid: tempo
        jsonData:
          tracesToLogs:
            datasourceUid: loki
            tags: ["pod", "namespace"]
            mappedTags: [{ key: "service.name", value: "app" }]
```

---

## Dashboard as Code

### Phương Pháp 1: ConfigMap với JSON Model

Grafana có thể tự động load dashboard từ ConfigMap khi deploy. Đây là cách đơn giản nhất.

```yaml
# grafana-dashboard-configmap.yaml
# Grafana sidecar container (grafana/grafana-sc-dashboard) watch ConfigMap
# có label grafana_dashboard: "1" và inject vào Grafana

apiVersion: v1
kind: ConfigMap
metadata:
  name: web-api-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"   # label này kích hoạt sidecar inject dashboard
data:
  web-api-dashboard.json: |
    {
      "__inputs": [],
      "__requires": [],
      "annotations": {
        "list": [
          {
            "datasource": "Prometheus",
            "enable": true,
            "expr": "changes(kube_deployment_status_replicas_updated{deployment=\"web-api\"}[2m]) > 0",
            "name": "Deployments",
            "titleFormat": "Deploy"
          }
        ]
      },
      "description": "Web API Service Dashboard",
      "panels": [
        {
          "id": 1,
          "title": "Request Rate (req/s)",
          "type": "timeseries",
          "targets": [
            {
              "expr": "sum(rate(http_requests_total{job=\"web-api\"}[5m])) by (status)",
              "legendFormat": "HTTP {{status}}"
            }
          ]
        },
        {
          "id": 2,
          "title": "Error Rate (%)",
          "type": "stat",
          "targets": [
            {
              "expr": "sum(rate(http_requests_total{job=\"web-api\",status=~\"5..\"}[5m])) / sum(rate(http_requests_total{job=\"web-api\"}[5m])) * 100"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "thresholds": {
                "steps": [
                  { "color": "green", "value": null },
                  { "color": "yellow", "value": 1 },
                  { "color": "red", "value": 5 }
                ]
              }
            }
          }
        },
        {
          "id": 3,
          "title": "P99 Latency (ms)",
          "type": "timeseries",
          "targets": [
            {
              "expr": "histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{job=\"web-api\"}[5m]))) * 1000",
              "legendFormat": "P99"
            }
          ]
        }
      ],
      "time": { "from": "now-1h", "to": "now" },
      "refresh": "30s",
      "tags": ["web-api", "production"],
      "title": "Web API Dashboard",
      "uid": "web-api-main",
      "version": 1
    }
```

### Phương Pháp 2: Grafana Operator

**Grafana Operator** là Kubernetes operator cho phép quản lý Grafana resource (datasource, dashboard, folder) qua CRD.

```yaml
# GrafanaDashboard CRD (dùng Grafana Operator)
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDashboard
metadata:
  name: web-api-overview
  namespace: monitoring
spec:
  instanceSelector:
    matchLabels:
      dashboards: "grafana"   # chọn Grafana instance nào để deploy dashboard
  folder: "Web API"
  resyncPeriod: 10m           # đồng bộ lại mỗi 10 phút
  url: "https://raw.githubusercontent.com/myorg/dashboards/main/web-api.json"
  # Hoặc dùng configMapRef:
  configMapRef:
    name: web-api-dashboard
    key: web-api-dashboard.json
```

### Phương Pháp 3: Grafana as Code với Grafonnet (Jsonnet)

```jsonnet
// web-api-dashboard.jsonnet
// Dùng Grafonnet library để tạo dashboard bằng code
local grafana = import 'grafonnet/grafana.libsonnet';
local dashboard = grafana.dashboard;
local graphPanel = grafana.graphPanel;
local prometheus = grafana.prometheus;

dashboard.new(
  'Web API Overview',
  uid='web-api-main',
  tags=['web-api'],
  refresh='30s',
)
.addPanel(
  graphPanel.new(
    'Request Rate',
    datasource='Prometheus',
  ).addTarget(
    prometheus.target(
      'sum(rate(http_requests_total{job="web-api"}[5m])) by (status)',
      legendFormat='HTTP {{status}}',
    )
  ),
  gridPos={ h: 8, w: 12, x: 0, y: 0 }
)
```

---

## Dashboard Quan Trọng cho Kubernetes

### Danh Sách Dashboard Nên Import từ Grafana.com

```bash
# Import dashboard từ Grafana.com bằng dashboard ID

# 1. Node Exporter Full — metric chi tiết mỗi Node
# ID: 1860 — https://grafana.com/grafana/dashboards/1860

# 2. Kubernetes Cluster Overview — tổng quan cluster
# ID: 7249 — https://grafana.com/grafana/dashboards/7249

# 3. Kubernetes Pod Resources — CPU/memory từng Pod
# ID: 6417 — https://grafana.com/grafana/dashboards/6417

# 4. Kubernetes Deployment — trạng thái Deployment
# ID: 8588 — https://grafana.com/grafana/dashboards/8588

# 5. Prometheus 2.0 Stats — sức khỏe Prometheus
# ID: 3662 — https://grafana.com/grafana/dashboards/3662

# 6. Loki Dashboard — tổng quan Loki logs
# ID: 13407 — https://grafana.com/grafana/dashboards/13407
```

### Import Tự Động qua values.yaml

```yaml
# Trong grafana values.yaml — import dashboard từ Grafana.com tự động
dashboards:
  default:
    node-exporter:
      gnetId: 1860        # Dashboard ID từ grafana.com
      revision: 37
      datasource: Prometheus

    kubernetes-overview:
      gnetId: 7249
      revision: 1
      datasource: Prometheus

    kubernetes-pods:
      gnetId: 6417
      revision: 1
      datasource: Prometheus
```

---

## Template Variable — Biến Lọc Dashboard

**Template Variable** cho phép người dùng chọn namespace, pod, node từ dropdown mà không cần tạo dashboard riêng cho từng môi trường.

```yaml
# Ví dụ cấu hình variable trong JSON dashboard model
"templating": {
  "list": [
    {
      "name": "namespace",
      "label": "Namespace",
      "type": "query",
      "datasource": "Prometheus",
      "query": "label_values(kube_pod_info, namespace)",
      "refresh": 2,           # refresh khi time range thay đổi
      "multi": false,
      "includeAll": true,
      "allValue": ".*"
    },
    {
      "name": "pod",
      "label": "Pod",
      "type": "query",
      "datasource": "Prometheus",
      "query": "label_values(kube_pod_info{namespace=~\"$namespace\"}, pod)",
      "refresh": 2,
      "multi": true,          # chọn nhiều pod cùng lúc
      "includeAll": true
    },
    {
      "name": "node",
      "label": "Node",
      "type": "query",
      "datasource": "Prometheus",
      "query": "label_values(kube_node_info, node)",
      "refresh": 1
    }
  ]
}
```

### Dùng Variable trong Panel Query

```promql
# Query dùng variable $namespace và $pod
sum by (pod) (
  rate(container_cpu_usage_seconds_total{
    namespace=~"$namespace",
    pod=~"$pod",
    container!=""
  }[5m])
)

# Memory của pod được chọn
container_memory_working_set_bytes{
  namespace=~"$namespace",
  pod=~"$pod",
  container!=""
}
```

---

## Unified Alerting — Alert Rule trong Grafana

Kể từ Grafana 9+, **Unified Alerting** cho phép tạo alert từ bất kỳ datasource nào (không chỉ Prometheus) ngay trong Grafana UI.

### Alert Rule qua CRD (Grafana Operator)

```yaml
# grafana-alert-rule.yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaAlertRuleGroup
metadata:
  name: web-api-alerts
  namespace: monitoring
spec:
  instanceSelector:
    matchLabels:
      dashboards: "grafana"
  folderRef:
    name: web-api-folder
  rules:
    - uid: web-api-high-error-rate
      title: "Web API Error Rate Cao"
      condition: C
      data:
        # Query A — lấy error rate
        - refId: A
          queryType: ""
          relativeTimeRange:
            from: 600
            to: 0
          datasourceUid: prometheus
          model:
            expr: |
              sum(rate(http_requests_total{job="web-api",status=~"5.."}[5m]))
              /
              sum(rate(http_requests_total{job="web-api"}[5m]))
            intervalMs: 1000
            maxDataPoints: 43200
            refId: A

        # Query C — điều kiện trigger alert
        - refId: C
          queryType: ""
          relativeTimeRange:
            from: 0
            to: 0
          datasourceUid: __expr__
          model:
            conditions:
              - evaluator:
                  params: [0.05]   # trigger khi error rate > 5%
                  type: gt
                operator:
                  type: and
                query:
                  params: [A]
                reducer:
                  type: last
                type: query
            refId: C
            type: classic_conditions

      noDataState: NoData
      execErrState: Error
      for: 5m         # alert phải đúng liên tục 5 phút trước khi fire
      labels:
        severity: critical
        team: backend
      annotations:
        summary: "Error rate Web API vượt 5%"
        description: "Error rate đang ở {{ $value | humanizePercentage }}, vượt ngưỡng 5%."
        runbook_url: "https://wiki.example.com/runbooks/web-api-high-error-rate"
```

### Alert Rule Prometheus (PrometheusRule CRD)

```yaml
# prometheus-alert-rules.yaml
# Alert rule trong Prometheus (không cần Grafana)
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: web-api-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: web-api
      interval: 30s
      rules:
        # Alert: error rate cao
        - alert: WebAPIHighErrorRate
          expr: |
            (
              sum(rate(http_requests_total{job="web-api",status=~"5.."}[5m]))
              /
              sum(rate(http_requests_total{job="web-api"}[5m]))
            ) > 0.05
          for: 5m
          labels:
            severity: critical
            service: web-api
          annotations:
            summary: "Web API error rate vượt 5%"
            description: "Error rate hiện tại: {{ $value | humanizePercentage }}. Kiểm tra log ngay."
            runbook_url: "https://wiki.example.com/runbooks/web-api-error-rate"

        # Alert: latency P99 cao
        - alert: WebAPIHighLatency
          expr: |
            histogram_quantile(0.99,
              sum by (le) (
                rate(http_request_duration_seconds_bucket{job="web-api"}[5m])
              )
            ) > 2
          for: 10m
          labels:
            severity: warning
            service: web-api
          annotations:
            summary: "Web API P99 latency vượt 2 giây"
            description: "P99 latency = {{ $value | humanizeDuration }}. Kiểm tra slow query, external dependency."

        # Alert: Pod CrashLoopBackOff
        - alert: PodCrashLooping
          expr: |
            increase(kube_pod_container_status_restarts_total{namespace="production"}[1h]) > 5
          for: 0m   # fire ngay lập tức
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ $labels.pod }} restart >5 lần trong 1 giờ"
            description: "Pod {{ $labels.namespace }}/{{ $labels.pod }}, container {{ $labels.container }} đang crash loop."
```

---

## Notification Channel — Kênh Thông Báo

### Cấu Hình Contact Point qua Grafana UI (hoặc API)

```yaml
# Cấu hình bằng Grafana provisioning (alerting/contactpoints.yaml)
# Mount vào container tại /etc/grafana/provisioning/alerting/

apiVersion: 1
contactPoints:
  - name: slack-critical
    receivers:
      - uid: slack-critical
        type: slack
        settings:
          url: "${SLACK_WEBHOOK_URL}"    # lấy từ env var, không hardcode
          channel: "#alerts-critical"
          username: "Grafana Alert"
          icon_emoji: ":red_circle:"
          title: |
            [{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}
          text: |
            *Summary:* {{ .CommonAnnotations.summary }}
            *Description:* {{ .CommonAnnotations.description }}
            *Severity:* {{ .CommonLabels.severity }}
            *Runbook:* {{ .CommonAnnotations.runbook_url }}

  - name: pagerduty-critical
    receivers:
      - uid: pagerduty-critical
        type: pagerduty
        settings:
          integrationKey: "${PAGERDUTY_INTEGRATION_KEY}"
          severity: "{{ .CommonLabels.severity }}"
          class: "{{ .CommonLabels.alertname }}"
          component: "{{ .CommonLabels.service }}"
          group: "kubernetes"
          summary: "{{ .CommonAnnotations.summary }}"

  - name: email-team
    receivers:
      - uid: email-team
        type: email
        settings:
          addresses: "oncall@example.com"
          subject: "[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}"
```

### Notification Policy (Route Alert đến Đúng Channel)

```yaml
# alerting/notification-policies.yaml
apiVersion: 1
policies:
  - orgId: 1
    receiver: slack-critical   # default receiver
    group_by: ["alertname", "cluster", "service"]
    group_wait: 30s            # đợi 30s để gom alert cùng group
    group_interval: 5m         # gửi update mỗi 5 phút cho group đang active
    repeat_interval: 4h        # nhắc lại sau 4 giờ nếu chưa resolve

    routes:
      # Alert critical → PagerDuty + Slack
      - receiver: pagerduty-critical
        matchers:
          - name: severity
            value: critical
        continue: true   # tiếp tục match rule sau (cũng gửi Slack)

      - receiver: slack-critical
        matchers:
          - name: severity
            value: critical

      # Alert warning → chỉ Slack channel warning
      - receiver: slack-warning
        matchers:
          - name: severity
            value: warning

      # Alert team database → Slack channel riêng
      - receiver: slack-database
        matchers:
          - name: team
            value: database
```

---

## Câu Hỏi Phỏng Vấn

**Dashboard as Code nghĩa là gì? Tại sao quan trọng?**

> Dashboard as Code là cách quản lý Grafana dashboard qua file (JSON, Jsonnet, YAML CRD) thay vì chỉnh sửa trực tiếp trên UI. Quan trọng vì: (1) **Version control** — track thay đổi dashboard qua Git, rollback khi dashboard bị xóa hoặc sửa nhầm; (2) **Reproducible deployment** — destroy rồi rebuild Grafana mà không mất dashboard; (3) **Review process** — đồng nghiệp review dashboard qua PR trước khi deploy; (4) **Multi-environment** — dùng cùng dashboard template cho staging và production, chỉ đổi datasource. Trong Kubernetes, implement bằng Grafana sidecar + ConfigMap (đơn giản) hoặc Grafana Operator + CRD (phức tạp hơn, powerful hơn).

**Grafana Unified Alerting khác gì so với Prometheus AlertManager?**

> Prometheus AlertManager nhận alert từ Prometheus (hoặc Grafana), xử lý routing, silencing, inhibition và gửi notification. Grafana Unified Alerting là alerting engine tích hợp trong Grafana, có thể tạo alert từ bất kỳ datasource nào (Prometheus, Loki, MySQL, Elasticsearch...) không cần qua Prometheus. Sự khác biệt chính: AlertManager mạnh hơn ở routing logic (route tree phức tạp, inhibition, silence theo label), còn Grafana Alerting tiện hơn vì tạo và quản lý alert ngay trong Grafana UI không cần YAML. Trong practice: nhiều team dùng cả hai — Prometheus tạo alert rule, Grafana gửi notification qua AlertManager — hoặc chuyển hoàn toàn sang Grafana Unified Alerting cho đơn giản.

**Template Variable giải quyết vấn đề gì trong dashboard?**

> Không có variable, bạn cần tạo dashboard riêng cho mỗi namespace/service/node — không scalable. Variable cho phép người dùng chọn từ dropdown (namespace, pod, node...) và tất cả panel trong dashboard tự động filter theo giá trị đã chọn. Query lấy danh sách giá trị từ Prometheus `label_values()` — luôn up-to-date khi có Pod mới. Multi-value variable cho phép chọn nhiều Pod cùng lúc để so sánh. Biến `$__interval` và `$__range` là variable đặc biệt Grafana tự điền dựa trên time range đang chọn — dùng trong query `rate(...[$__rate_interval])` để tránh hardcode khoảng thời gian.

---

## Checklist Production

### Cài Đặt và Bảo Mật

- [ ] Grafana không expose với anonymous access
- [ ] Dùng SSO (OIDC/SAML) thay vì local password
- [ ] Admin password lưu trong Kubernetes Secret, không hardcode trong values.yaml
- [ ] Grafana ingress có TLS (HTTPS)
- [ ] Audit log bật trong grafana.ini: `[log] level = info`

### Dashboard

- [ ] Tất cả dashboard quan trọng được version control (ConfigMap hoặc Git)
- [ ] Dashboard Node Exporter Full, K8s Overview đã import
- [ ] Dashboard cho từng service quan trọng có error rate, latency, request rate
- [ ] SLO dashboard hiển thị error budget còn bao nhiêu
- [ ] Annotation hiển thị deployment event trên timeline

### Alert

- [ ] Alert rule test được (amend to fire, verify notification nhận được)
- [ ] Contact point (Slack, PagerDuty) được cấu hình và test
- [ ] Notification policy route đúng severity đến đúng channel
- [ ] Alert có runbook URL link đến tài liệu xử lý
- [ ] Silence policy được test — verify alert không gửi trong maintenance window

### Datasource

- [ ] Datasource Prometheus và Loki được cấu hình và status `OK`
- [ ] Datasource Tempo được cấu hình với derived field TraceID → link sang trace
- [ ] Grafana có thể query Loki và Prometheus trong cùng Explore view
- [ ] Datasource được version control qua provisioning (không chỉnh tay trên UI)
