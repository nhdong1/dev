# CloudWatch Dashboards & Widgets

> **CloudWatch Dashboards** (Bảng Điều Khiển CloudWatch) là giao diện trực quan hóa tập trung, kết hợp metrics, logs, và alarms từ nhiều dịch vụ, region, và account AWS trong một màn hình vận hành duy nhất.

---

## 📚 Mục Lục

1. [Dashboard Là Gì?](#dashboard-là-gì)
2. [Các Loại Widget](#các-loại-widget)
3. [Thiết Kế Dashboard Hiệu Quả](#thiết-kế-dashboard-hiệu-quả)
4. [Cross-Account & Cross-Region Dashboard](#cross-account--cross-region-dashboard)
5. [Automatic Dashboards — Dashboard Tự Động](#automatic-dashboards--dashboard-tự-động)
6. [Dashboard Sharing & Embedding](#dashboard-sharing--embedding)
7. [Dashboard as Code](#dashboard-as-code)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Dashboard Là Gì?

**CloudWatch Dashboard** là trang tùy chỉnh gồm nhiều **widgets** (tiện ích) hiển thị dữ liệu giám sát theo thời gian thực. Dashboard giúp:

- **Situational Awareness** (Nhận Thức Tình Huống) — Nhìn toàn cảnh hệ thống trong 1 màn hình
- **Correlation** (Tương Quan) — Đặt CPU, latency, error rate cạnh nhau để thấy mối liên hệ
- **Communication** — Chia sẻ trạng thái với team không cần tài khoản AWS (public sharing)
- **Incident Response** (Phản Hồi Sự Cố) — Dashboard sự cố riêng để debug nhanh

### Pricing (Giá)

| Tier                     | Giới Hạn                    | Chi Phí                              |
| ------------------------ | --------------------------- | ------------------------------------ |
| Free Tier                | 3 dashboards, 50 metrics/dashboard | Miễn phí                      |
| Có tính phí              | Dashboard thứ 4 trở đi      | $3/dashboard/tháng                   |
| High-resolution widgets  | 1-second refresh            | Phí tăng theo metric cost            |

> **Mẹo:** 3 dashboard miễn phí đủ cho nhiều startup và team nhỏ. Organize tốt để dùng ít dashboard.

---

## Các Loại Widget

### 1. Line Widget (Biểu Đồ Đường)

Hiển thị metrics theo thời gian dưới dạng đường. Phổ biến nhất.

```
Phù hợp cho:
  - CPU utilization theo thời gian
  - Request rate trends (xu hướng)
  - Latency P50/P95/P99 so sánh
  - So sánh nhiều instances/services cùng lúc

Cấu hình quan trọng:
  - Y-axis: tự động hoặc fixed (cố định) range
  - Period: 1 phút, 5 phút, 1 giờ
  - Stacked: hiển thị tổng cộng các metrics
```

### 2. Number Widget (Số Đơn)

Hiển thị giá trị hiện tại của một metric — nhanh, dễ đọc.

```
Phù hợp cho:
  - Tổng request trong 1 giờ qua
  - Error count hiện tại
  - Active DB connections
  - Lambda concurrent executions

Có thể thêm:
  - Sparkline (biểu đồ mini bên dưới số)
  - Comparison (so sánh với period trước)
  - Color thresholds (đổi màu theo ngưỡng)
```

### 3. Gauge Widget (Đồng Hồ Đo)

Hiển thị giá trị trên thang đo hình vòng cung — trực quan cho phần trăm.

```
Phù hợp cho:
  - CPU utilization (0-100%)
  - Disk space used (0-100%)
  - Memory utilization (0-100%)
  - Error rate (0-100%)

Màu thường dùng:
  0–60%:   Xanh lá (OK)
  60–80%:  Vàng (Warning)
  80–100%: Đỏ (Critical)
```

### 4. Stacked Area Widget (Vùng Xếp Chồng)

Như Line chart nhưng điền màu phần dưới — tốt cho thấy tổng và thành phần.

```
Phù hợp cho:
  - Request breakdown theo status code (2xx vs 4xx vs 5xx)
  - Cost breakdown theo service
  - Traffic breakdown theo region
```

### 5. Bar Widget (Biểu Đồ Thanh)

Thanh ngang hoặc dọc, so sánh giá trị tại thời điểm cụ thể.

```
Phù hợp cho:
  - Throughput theo từng AZ (Availability Zone)
  - Error count theo từng service
  - Latency P99 theo từng API endpoint
```

### 6. Logs Table Widget (Bảng Nhật Ký)

Nhúng kết quả Logs Insights query vào dashboard — tự động refresh.

```
Phù hợp cho:
  - Hiển thị 10 lỗi gần nhất trong dashboard vận hành
  - Top slow requests theo thời gian thực
  - Recent security events từ CloudTrail

Cấu hình:
  - Log Group: chọn 1 hoặc nhiều groups
  - Query: Logs Insights query string
  - Refresh interval: 1 phút, 5 phút
```

**Ví dụ query cho Logs Table Widget:**
```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 10
```

### 7. Alarm Status Widget (Trạng Thái Cảnh Báo)

Hiển thị trạng thái (OK/ALARM/INSUFFICIENT_DATA) của nhiều alarms — nhìn nhanh "đèn giao thông".

```
Phù hợp cho:
  - Overview dashboard — thấy ngay service nào đang có vấn đề
  - Executive dashboard (Dashboard Điều Hành) — không cần chi tiết
  - NOC (Network Operations Center) screen

Hiển thị:
  - Xanh lá: OK
  - Đỏ: ALARM
  - Xám: INSUFFICIENT_DATA
```

### 8. Text Widget (Văn Bản)

Markdown text tĩnh — tiêu đề section, link, hướng dẫn sử dụng dashboard.

```
Phù hợp cho:
  - Tiêu đề và section headers
  - Link đến runbook (sách hướng dẫn vận hành)
  - Ngưỡng SLA reference
  - Liên hệ on-call
```

### 9. Explorer Widget (Khám Phá)

Dynamic widget — tự động hiển thị tất cả instances/resources khớp với tag hoặc filter.

```
Phù hợp cho:
  - Fleet monitoring (giám sát hàng trăm EC2)
  - Tự động thêm instance mới vào dashboard khi scale out
  - Environment-wide view không cần cấu hình từng instance
```

---

## Thiết Kế Dashboard Hiệu Quả

### Nguyên Tắc Thiết Kế

**1. Dashboard theo mục đích, không theo dịch vụ:**
```
❌ "EC2 Dashboard" — một dashboard cho tất cả EC2
✅ "Order Service Production" — một dashboard cho một service
✅ "Infrastructure Health" — executive overview
✅ "Incident Response Runbook" — debug dashboard
```

**2. Layout từ trên xuống theo mức độ trừu tượng:**
```
Hàng 1: Key business metrics (Orders/s, Revenue, Active users)
Hàng 2: Application health (Error rate, P99 Latency)
Hàng 3: Infrastructure (CPU, Memory, Network)
Hàng 4: Database (Connections, Read/Write IOPS, Replication lag)
Hàng 5: Recent errors (Logs Table widget)
```

**3. 3 loại dashboard nên có:**

| Dashboard              | Người Xem              | Cập Nhật     | Mục Tiêu                            |
| ---------------------- | ---------------------- | ------------ | ----------------------------------- |
| Executive Overview     | Management, Product    | Realtime     | Business metrics, overall health    |
| Service Operations     | Dev/Ops team           | Realtime     | Service-level debugging             |
| Incident Response      | On-call engineer       | Realtime     | Nhanh nhất có thể debug sự cố      |

**4. Consistent time range (Khoảng Thời Gian Nhất Quán):**
- Tất cả widget trong một dashboard nên dùng cùng time range
- CloudWatch hỗ trợ relative time (1h, 3h, 1d) hoặc absolute time range
- Trong sự cố: zoom vào absolute time window để correlate (tương quan) các metrics

### Mẫu Dashboard Vận Hành Chuẩn

```
┌─────────────────────────────────────────────────────────────────┐
│  Service: Order Service | Environment: Production | [3h ▼]      │
├──────────────┬──────────────┬──────────────┬────────────────────┤
│ Orders/min   │ Error Rate   │ P99 Latency  │ Active Alarms      │
│ [Number]     │ [Gauge]      │ [Number]     │ [Alarm Status]     │
│  1,234       │  0.1%  ●     │  245ms       │  ● OrderSvc OK     │
│              │              │              │  ● PaySvc  OK      │
│              │              │              │  ● RDS     OK      │
├──────────────┴──────────────┴──────────────┴────────────────────┤
│                    Request Rate & Error Rate                     │
│  [Line Chart — RequestCount + 5xx Count (dual axis)]            │
│  ████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░     │
├─────────────────────────────────────────────────────────────────┤
│           Latency Percentiles                                    │
│  [Line — P50, P95, P99 trên cùng chart]                        │
├──────────────────────────┬──────────────────────────────────────┤
│  EC2 CPU Utilization     │  RDS Connections & Replication Lag   │
│  [Line — per instance]   │  [Line — dual axis]                  │
├──────────────────────────┴──────────────────────────────────────┤
│  Recent Errors (Logs Table)                                     │
│  [timestamp]  [ERROR] OrderService: PaymentTimeout orderId=123  │
│  [timestamp]  [ERROR] OrderService: InventoryCheckFailed        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Cross-Account & Cross-Region Dashboard

### Cross-Region Dashboard (Dashboard Đa Vùng)

Một dashboard có thể hiển thị metrics từ nhiều AWS regions:

```
Dashboard (us-east-1):
  Widget 1: EC2 CPU (us-east-1) — US region
  Widget 2: EC2 CPU (eu-west-1) — Europe region
  Widget 3: EC2 CPU (ap-southeast-1) — Asia region
  Widget 4: ALB Latency (tất cả 3 regions — trên cùng chart)
```

Hữu ích cho: Global application health monitoring, disaster recovery (DR) comparison.

### Cross-Account Dashboard (Dashboard Đa Tài Khoản)

Hiển thị metrics từ nhiều AWS accounts trong một dashboard — cần **CloudWatch cross-account observability** được cấu hình.

**Kiến Trúc:**
```
Monitoring Account (Central Account)
       ↑
       │ (CloudWatch sharing role)
       │
├── Production Account (metrics shared)
├── Staging Account (metrics shared)
└── Development Account (metrics shared)

Dashboard trong Monitoring Account:
  → Xem metrics từ tất cả accounts
  → So sánh prod vs staging latency
  → Fleet-wide CPU utilization
```

**Cách Bật Cross-Account Sharing:**

```bash
# Trong source account (app account) — kích hoạt chia sẻ
aws cloudwatch put-dashboard-sharing-option \
  --dashboard-sharing-option IsEnabled=true,MonitoringAccountId=123456789012

# Trong monitoring account — tạo sink (nơi nhận)
aws oam create-sink --name "central-observability" --tags Environment=prod
aws oam create-link \
  --label-template "$AccountName" \
  --resource-types "AWS::CloudWatch::Metric" \
  --sink-identifier arn:aws:oam:us-east-1:123456789012:sink/abc123
```

> **OAM — Observability Access Manager** (Quản Lý Truy Cập Quan Sát): Service mới (2022) quản lý cross-account observability một cách có cấu trúc.

---

## Automatic Dashboards — Dashboard Tự Động

**CloudWatch Automatic Dashboards** được tạo tự động cho từng AWS service, hiển thị key metrics ngay khi bạn dùng service đó.

```
Automatic Dashboards có sẵn cho:
  EC2, Lambda, RDS, ECS, API Gateway, DynamoDB, S3,
  CloudFront, ElastiCache, Kinesis, SNS, SQS...

Cách truy cập:
  CloudWatch Console → Dashboards → Automatic Dashboards → [chọn service]
```

Automatic Dashboards **miễn phí** và không cần cấu hình. Tuy nhiên, chúng hiển thị metrics theo account-level và service-level — không thể customize theo business logic. Dùng làm điểm khởi đầu, sau đó tạo custom dashboard.

---

## Dashboard Sharing & Embedding

### Public Sharing (Chia Sẻ Công Khai)

Chia sẻ dashboard mà không yêu cầu người xem có AWS account:

```
Cách bật:
  Dashboard → Actions → Share dashboard → Enable sharing
  → Chọn: Public (anyone with link) hoặc Email-based SSO

Use case:
  - Executive team xem business metrics
  - Customer-facing status page (cần thiết kế cẩn thận)
  - On-call engineer trên điện thoại không có VPN
```

> **Bảo mật:** Public dashboard chỉ hiển thị data, không cho phép bất kỳ action nào. Tuy nhiên, cân nhắc thông tin nhạy cảm (số lượng orders, capacity) trước khi public.

### Embedding Dashboard (Nhúng Dashboard)

Nhúng dashboard vào web application nội bộ bằng iFrame:

```html
<!-- Nhúng CloudWatch dashboard vào internal tool -->
<iframe
  src="https://cloudwatch.amazonaws.com/dashboard.html?dashboard=MyDashboard&context=..."
  width="1000"
  height="800"
  frameborder="0">
</iframe>
```

---

## Dashboard as Code

### CloudFormation Template

```yaml
Resources:
  ProductionDashboard:
    Type: AWS::CloudWatch::Dashboard
    Properties:
      DashboardName: OrderService-Production
      DashboardBody: !Sub |
        {
          "widgets": [
            {
              "type": "metric",
              "x": 0, "y": 0, "width": 12, "height": 6,
              "properties": {
                "title": "Order Service - Request Rate",
                "metrics": [
                  ["AWS/ApplicationELB", "RequestCount",
                   "LoadBalancer", "${ALBName}",
                   {"stat": "Sum", "period": 60, "label": "Requests/min"}]
                ],
                "view": "timeSeries",
                "region": "${AWS::Region}",
                "period": 300
              }
            },
            {
              "type": "metric",
              "x": 12, "y": 0, "width": 12, "height": 6,
              "properties": {
                "title": "P99 Latency",
                "metrics": [
                  ["AWS/ApplicationELB", "TargetResponseTime",
                   "LoadBalancer", "${ALBName}",
                   {"stat": "p99", "label": "P99"}]
                ],
                "view": "timeSeries",
                "region": "${AWS::Region}"
              }
            }
          ]
        }
```

### Terraform

```hcl
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "OrderService-Production"

  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          title = "Request Rate"
          metrics = [
            ["AWS/ApplicationELB", "RequestCount",
             "LoadBalancer", var.alb_name,
             { "stat" = "Sum", "period" = 60 }]
          ]
          view   = "timeSeries"
          region = var.aws_region
        }
      }
    ]
  })
}
```

### AWS CDK (Python)

```python
from aws_cdk import aws_cloudwatch as cw

dashboard = cw.Dashboard(self, "OrderServiceDashboard",
    dashboard_name="OrderService-Production"
)

request_rate_widget = cw.GraphWidget(
    title="Request Rate",
    left=[
        cw.Metric(
            namespace="AWS/ApplicationELB",
            metric_name="RequestCount",
            dimensions_map={"LoadBalancer": alb.load_balancer_arn_suffix},
            statistic="Sum",
            period=Duration.minutes(1),
            label="Requests/min"
        )
    ],
    width=12,
    height=6
)

dashboard.add_widgets(request_rate_widget)
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Nên dùng bao nhiêu dashboard cho một hệ thống microservices?

Không có con số cố định, nhưng nguyên tắc:
- **1 Executive Dashboard** — Business KPIs tổng hợp toàn hệ thống
- **1 Dashboard per service** — Cho team phụ trách service đó
- **1 Incident Response Dashboard** — Gồm tất cả critical paths và dependencies
- **1 Infrastructure Dashboard** — EC2/ECS cluster health

Tránh tạo quá nhiều dashboard không có người xem — gây overhead bảo trì và chi phí ($3/dashboard).

### Q2: Cross-account Dashboard setup cần những gì?

1. **OAM Sink** (Bồn Chứa OAM) trong monitoring account — nơi nhận metric
2. **OAM Link** trong mỗi source account — kết nối và share metrics
3. **Resource type** khai báo: Metrics, Logs, Traces (chọn loại data share)
4. **IAM permissions** cho monitoring account đọc data từ source accounts

Sau khi setup, CloudWatch console trong monitoring account tự động hiển thị data từ tất cả linked accounts.

### Q3: Tại sao không nên dùng Average mà nên dùng P99 cho Latency widget?

Average latency bị dominated bởi request nhanh và che khuất tail latency — kinh nghiệm người dùng tệ nhất. P99 cho thấy 1% user "tệ nhất" đang trải qua gì. Với 1 triệu request/ngày, 1% là 10,000 user.

Trong dashboard production, show cả P50, P95, P99 để thấy distribution (phân phối).

### Q4: Explorer Widget khác gì với Widget thông thường?

**Explorer Widget** (Widget Khám Phá) là dynamic — tự động discover (khám phá) resources dựa trên tag, type, filter mà không cần config từng resource:

```
Tag filter: Environment=prod AND Team=payments
→ Tự động hiển thị CPU của TẤT CẢ EC2 instances có tags này
→ Khi thêm instance mới với tag đó, tự động vào widget
```

Widget thông thường phải hard-code (mã hóa cứng) từng metric và dimension. Explorer thích hợp cho fleet monitoring với dynamic scaling.

### Q5: CloudWatch Dashboard có thể thay thế Grafana không?

Cho AWS-only workloads: CloudWatch Dashboard đủ dùng và ít operational overhead (gánh nặng vận hành) hơn.

Grafana cần thiết khi:
- **Multi-cloud/on-premises** data sources cần visualize cùng nhau
- **Kubernetes** monitoring (Prometheus là standard, Grafana integrate native)
- **Advanced visualization** — Heatmap, complex transforms, plugins
- **Team đã quen Grafana** — Migration cost cao

Nhiều tổ chức dùng **Amazon Managed Grafana** — managed service, tích hợp CloudWatch và Prometheus, không cần tự host.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
**Liên Kết:** [3-logs-insights.md](./3-logs-insights.md) | [5-container-application-insights.md](./5-container-application-insights.md)
