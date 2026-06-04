# 4 — AWS X-Ray — Distributed Tracing (Theo Dõi Phân Tán)

> AWS X-Ray — Dịch Vụ Theo Dõi Phân Tán — cho phép trace (theo dõi) hành trình của một request xuyên qua nhiều services, giúp tìm bottleneck (điểm nghẽn) và debug trong hệ thống microservices

## 📚 Mục Lục

1. [Tại Sao Cần Distributed Tracing](#tại-sao-cần-distributed-tracing)
2. [Kiến Trúc X-Ray](#kiến-trúc-x-ray)
3. [Cấu Trúc Dữ Liệu: Trace, Segment, Subsegment](#cấu-trúc-dữ-liệu)
4. [X-Ray SDK — Tích Hợp Vào Ứng Dụng](#x-ray-sdk)
5. [Service Map — Bản Đồ Dịch Vụ](#service-map)
6. [Sampling Rules — Quy Tắc Lấy Mẫu](#sampling-rules)
7. [X-Ray Groups và Analytics](#x-ray-groups-và-analytics)
8. [Integration Với AWS Services](#integration-với-aws-services)
9. [X-Ray Insights — Phát Hiện Sự Cố Tự Động](#x-ray-insights)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Distributed Tracing

### Vấn Đề Với Microservices Không Có Tracing

```
User báo cáo: "Checkout bị chậm, mất 8 giây!"

Bạn có:
  ✓ CloudWatch Logs của mỗi service riêng lẻ
  ✓ CloudWatch Metrics của từng service

Bạn KHÔNG biết:
  ✗ Request đi qua những services nào?
  ✗ Service nào gây ra 8 giây đó?
  ✗ API Gateway tốn bao lâu? Lambda? DynamoDB? Payment API?
  ✗ Lỗi ở service nào trong chuỗi?
```

### Với X-Ray Distributed Tracing

```
User báo cáo: "Checkout bị chậm, mất 8 giây!"

X-Ray Service Map cho thấy:
  API Gateway        → 15ms ✓
  Lambda-checkout    → 50ms ✓
  DynamoDB GetItem   → 12ms ✓
  SQS SendMessage    → 8ms  ✓
  HTTP → payment-api → 7,850ms ✗ ← ROOT CAUSE!

Vấn đề ngay lập tức: external payment API đang chậm
Giải pháp: timeout ngắn hơn, retry, circuit breaker
```

---

## Kiến Trúc X-Ray

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AWS X-RAY ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  YOUR APPLICATION                                                   │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  X-Ray SDK (tích hợp trong code)                             │  │
│  │  → Tạo Segments, Subsegments                                 │  │
│  │  → Propagate Trace ID qua HTTP headers (X-Amzn-Trace-Id)    │  │
│  └──────────────────────────────┬────────────────────────────────┘  │
│                                 │ UDP port 2000 (localhost)          │
│  ┌──────────────────────────────▼────────────────────────────────┐  │
│  │  X-Ray DAEMON (Tiến Trình Nền X-Ray)                         │  │
│  │  → Buffer trace segments locally                             │  │
│  │  → Batch gửi về X-Ray API mỗi 1 giây                        │  │
│  │  (Chạy trên EC2, ECS sidecar, hoặc built-in Lambda)         │  │
│  └──────────────────────────────┬────────────────────────────────┘  │
│                                 │ HTTPS                              │
│  ┌──────────────────────────────▼────────────────────────────────┐  │
│  │                    AWS X-RAY SERVICE                          │  │
│  │  ┌─────────────────┐  ┌──────────────┐  ┌──────────────────┐ │  │
│  │  │  Trace Storage  │  │ Service Map  │  │    Analytics     │ │  │
│  │  │  (30 ngày)      │  │  Generator   │  │    Engine        │ │  │
│  │  └─────────────────┘  └──────────────┘  └──────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### X-Ray Daemon vs ADOT

| Phương Thức              | Mô Tả                                        | Khi Dùng                     |
| ------------------------ | -------------------------------------------- | ---------------------------- |
| **X-Ray Daemon**         | Tiến trình nhỏ nhận trace từ SDK qua UDP     | EC2, ECS sidecar, Lambda     |
| **X-Ray SDK trực tiếp**  | SDK gửi thẳng đến X-Ray API                  | Serverless, đơn giản         |
| **ADOT** (AWS Distro for OpenTelemetry — Phân Phối AWS Cho OpenTelemetry) | Agent tổng quát, hỗ trợ nhiều backend | EKS, multi-cloud, vendor-neutral |

---

## Cấu Trúc Dữ Liệu

### Trace — Vết Theo Dõi

```
TRACE (Vết) = toàn bộ hành trình của 1 request từ đầu đến cuối

Trace ID: 1-5759e988-bd862e3fe1be46a994272793
Thời gian: 8,240ms (tổng thời gian)

Cấu trúc:
  └── SEGMENT: API Gateway (origin)
      └── SEGMENT: Lambda-checkout-handler
          ├── SUBSEGMENT: DynamoDB GetItem    [12ms]
          ├── SUBSEGMENT: SQS SendMessage     [8ms]
          └── SUBSEGMENT: HTTP payment-api    [7,850ms] ← chậm
              └── SUBSEGMENT: payment-api internal...
```

### Segment — Đoạn

Segment là công việc của **một service** trong một request. Mỗi service tạo ra một Segment:

```json
{
  "name": "order-service",
  "id": "70de5b6f19ff9a70",
  "trace_id": "1-5759e988-bd862e3fe1be46a994272793",
  "start_time": 1461096053.37518,
  "end_time": 1461096053.4042,
  "in_progress": false,
  "annotations": {
    "orderId": "12345",
    "userId": "user-789"
  },
  "metadata": {
    "orderDetails": {"items": 3, "total": 150.00}
  },
  "http": {
    "request": {"method": "POST", "url": "https://api.example.com/checkout"},
    "response": {"status": 200}
  }
}
```

### Subsegment — Đoạn Con

Subsegment theo dõi **một hoạt động cụ thể** bên trong một service — gọi database, HTTP call, v.v.:

```python
import boto3
from aws_xray_sdk.core import xray_recorder

@xray_recorder.capture('get_user_from_db')
def get_user(user_id: str):
    # X-Ray tự động tạo subsegment "get_user_from_db"
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('Users')
    return table.get_item(Key={'userId': user_id})
```

### Annotations vs Metadata

```python
# ANNOTATIONS (Chú Thích) — có thể filter/search trong X-Ray Console
xray_recorder.current_segment().put_annotation('orderId', '12345')
xray_recorder.current_segment().put_annotation('userId', 'user-789')
xray_recorder.current_segment().put_annotation('environment', 'prod')

# METADATA (Siêu Dữ Liệu) — chỉ xem chi tiết, không filter được
xray_recorder.current_segment().put_metadata(
    'order_details',
    {'items': 3, 'total': 150.00, 'currency': 'VND'}
)
```

**Phân biệt quan trọng:**
- Annotations → index được, có thể dùng để filter traces trong Console
- Metadata → không index, chỉ xem khi drill-down vào trace cụ thể

---

## X-Ray SDK

### Python SDK Setup

```python
# Cài đặt
# pip install aws-xray-sdk

from aws_xray_sdk.core import xray_recorder, patch_all
from aws_xray_sdk.ext.flask.middleware import XRayMiddleware

# Cấu hình
xray_recorder.configure(
    service='order-service',
    daemon_address='127.0.0.1:2000',  # X-Ray Daemon
    sampling=True,
)

# Patch AWS SDK và HTTP libraries để auto-instrument
patch_all()  # patches boto3, requests, etc.

# Flask middleware
from flask import Flask
app = Flask(__name__)
XRayMiddleware(app, xray_recorder)
```

### Lambda — Tự Động (Managed)

Lambda tự động bật X-Ray nếu được cấu hình — không cần Daemon:

```bash
# Bật Active Tracing cho Lambda
aws lambda update-function-configuration \
  --function-name order-handler \
  --tracing-config Mode=Active
```

```python
# Trong Lambda code — dùng SDK để thêm custom subsegments
from aws_xray_sdk.core import xray_recorder

def lambda_handler(event, context):
    # X-Ray Segment tự động được tạo bởi Lambda runtime

    with xray_recorder.in_subsegment('validate-input') as subsegment:
        subsegment.put_annotation('orderId', event['order_id'])
        validate_order(event)

    with xray_recorder.in_subsegment('process-payment') as subsegment:
        result = process_payment(event)
        subsegment.put_annotation('paymentStatus', result['status'])

    return {"statusCode": 200}
```

### Node.js SDK

```javascript
const AWSXRay = require('aws-xray-sdk');
const AWS = AWSXRay.captureAWS(require('aws-sdk'));
// AWS SDK calls tự động được trace

// Custom subsegment
const segment = AWSXRay.getSegment();
const subseg = segment.addNewSubsegment('custom-operation');
subseg.addAnnotation('key', 'value');
// ... thực hiện công việc ...
subseg.close();
```

### Propagate Trace ID Giữa Services

X-Ray tự động propagate Trace ID qua HTTP header `X-Amzn-Trace-Id`:

```
Request từ Client:
  X-Amzn-Trace-Id: Root=1-5759e988-bd862e3fe1be46a994272793;Sampled=1

Service A nhận header → tiếp tục trace với cùng Trace ID
Service A gọi Service B → tự động truyền header
Service B → tiếp tục cùng trace
```

Nếu dùng SDK đã `patch_all()` hoặc `captureHTTPs()` → propagation tự động.

---

## Service Map — Bản Đồ Dịch Vụ

Service Map (Bản Đồ Dịch Vụ) là đồ thị visualize các services và dependencies của chúng, với health status và latency.

### Đọc Service Map

```
Service Map Node thông tin:
  ┌──────────────────────────────┐
  │ 🟢 order-service             │
  │ Requests/min: 1,234          │
  │ Avg Latency: 245ms           │
  │ Error Rate: 0.02%            │
  └──────────────────────────────┘

Node màu:
  🟢 Green  → OK, hoạt động bình thường
  🟡 Yellow → Có lỗi (4xx errors từ client)
  🔴 Red    → Có lỗi nghiêm trọng (5xx, fault)

Cạnh (Edge) trong đồ thị:
  Mũi tên → chiều gọi (caller → callee)
  Độ dày  → volume traffic
  Màu     → health status của edge
```

### Dùng Service Map Để Debug

```
Quy trình debug với Service Map:

1. Tìm node 🔴 Red hoặc 🟡 Yellow
2. Click vào node đó
3. Xem list traces có lỗi
4. Click vào trace cụ thể
5. Xem Trace Timeline — segment nào chậm/lỗi
6. Xem segment detail — annotations, metadata, HTTP info
7. Link sang CloudWatch Logs nếu cần thêm chi tiết
```

---

## Sampling Rules — Quy Tắc Lấy Mẫu

Sampling (Lấy Mẫu) kiểm soát tỷ lệ requests được trace — balance giữa visibility và chi phí.

### Default Sampling Rule

```
Mặc định X-Ray:
  - 5% tất cả requests được trace
  - Bổ sung 1 request/giây (reservoir) luôn được trace
  - Đảm bảo có ít nhất 1 trace/giây dù traffic thấp
```

### Custom Sampling Rules

```bash
# Tạo rule: trace 100% checkout requests (quan trọng)
aws xray create-sampling-rule --cli-input-json '{
  "SamplingRule": {
    "RuleName": "CheckoutHighSampling",
    "RulePriority": 1,
    "FixedRate": 1.0,
    "ReservoirSize": 100,
    "ServiceName": "order-service",
    "ServiceType": "AWS::Lambda::Function",
    "Host": "*",
    "HTTPMethod": "POST",
    "URLPath": "/checkout",
    "ResourceARN": "*",
    "Version": 1
  }
}'

# Rule: trace 1% health check endpoints (không quan trọng)
aws xray create-sampling-rule --cli-input-json '{
  "SamplingRule": {
    "RuleName": "HealthCheckLowSampling",
    "RulePriority": 10,
    "FixedRate": 0.01,
    "ReservoirSize": 0,
    "ServiceName": "*",
    "ServiceType": "*",
    "Host": "*",
    "HTTPMethod": "GET",
    "URLPath": "/health",
    "ResourceARN": "*",
    "Version": 1
  }
}'
```

### Reservoir — Bể Dự Trữ

```
Reservoir Size = số requests/giây luôn được trace (bất kể FixedRate)
Fixed Rate     = tỷ lệ % trace sau khi reservoir đầy

Ví dụ:
  ReservoirSize = 10, FixedRate = 0.05 (5%)
  
  Traffic: 5 req/giây  → 5 traced (reservoir chưa đầy, trace hết)
  Traffic: 50 req/giây → 10 (reservoir) + 2 (5% của 40 còn lại) = 12 traced
  Traffic: 500 req/giây → 10 (reservoir) + 24.5 (5% của 490) ≈ 34 traced
```

---

## X-Ray Groups và Analytics

### X-Ray Groups — Nhóm Trace

Groups filter traces để phân tích riêng theo điều kiện:

```bash
# Tạo Group cho traces có lỗi
aws xray create-group \
  --group-name "ProductionErrors" \
  --filter-expression "fault = true OR error = true"

# Tạo Group cho checkout flow chậm
aws xray create-group \
  --group-name "SlowCheckout" \
  --filter-expression "responsetime > 2 AND service(\"order-service\")"
```

### Filter Expressions — Biểu Thức Lọc

```
# Lọc theo service
service("order-service") { fault = true }

# Lọc theo response time
responsetime > 2

# Lọc theo annotation
annotation.orderId = "12345"

# Lọc theo HTTP status
http.status = 500

# Kết hợp
service("order-service") AND responsetime > 1 AND annotation.environment = "prod"
```

### X-Ray Analytics Console

```
Trong Console → X-Ray → Analytics:

1. Response Time Distribution:
   → Biểu đồ phân phối latency
   → Tìm P50, P90, P99
   → Phát hiện bimodal distribution (2 đỉnh = 2 nhóm requests khác nhau)

2. Root Cause Analysis:
   → X-Ray tự động nhóm traces có fault tương tự
   → Gợi ý possible root causes
   → Thống kê theo service/URL/response code
```

---

## Integration Với AWS Services

### API Gateway

```yaml
# Bật tracing trong API Gateway
RestApiResource:
  Type: AWS::ApiGateway::Stage
  Properties:
    TracingEnabled: true
    # X-Ray trace header tự động được thêm vào downstream calls
```

### ECS — X-Ray Daemon Sidecar

```json
{
  "containerDefinitions": [
    {
      "name": "order-service",
      "image": "my-app:latest",
      "environment": [
        {"name": "AWS_XRAY_DAEMON_ADDRESS", "value": "xray-daemon:2000"}
      ]
    },
    {
      "name": "xray-daemon",
      "image": "amazon/aws-xray-daemon:latest",
      "portMappings": [
        {"containerPort": 2000, "protocol": "udp"}
      ],
      "cpu": 32,
      "memoryReservation": 256
    }
  ]
}
```

### EC2 — Cài X-Ray Daemon

```bash
# Cài đặt X-Ray Daemon
wget https://s3.us-east-2.amazonaws.com/aws-xray-assets.us-east-2/xray-daemon/aws-xray-daemon-3.x.rpm
sudo rpm -U aws-xray-daemon-3.x.rpm

# Khởi động
sudo service xray start

# Cấu hình /etc/amazon/xray/cfg.yaml
LocalMode: false
Region: "ap-southeast-1"
Socket:
  UDPAddress: "127.0.0.1:2000"
```

### EKS — ADOT Operator

```bash
# Cài ADOT (AWS Distro for OpenTelemetry) Operator qua Helm
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm install opentelemetry-operator open-telemetry/opentelemetry-operator

# Tạo OpenTelemetryCollector CR (Custom Resource)
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: xray-collector
spec:
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
    exporters:
      awsxray:
        region: ap-southeast-1
    service:
      pipelines:
        traces:
          receivers: [otlp]
          exporters: [awsxray]
EOF
```

---

## X-Ray Insights — Phát Hiện Sự Cố Tự Động

X-Ray Insights (phải bật thủ công cho mỗi Group) tự động phát hiện anomaly trong trace data:

```
Insights có thể phát hiện:
  ✓ Đột ngột tăng error rate
  ✓ Latency regression (thụt lùi về hiệu năng)
  ✓ Fault patterns trong specific service
  ✓ Traffic pattern bất thường

Insights tạo:
  → SNS notification khi phát hiện issue mới
  → Timeline của sự cố từ lúc bắt đầu đến kết thúc
  → Impact analysis: bao nhiêu % traces bị ảnh hưởng
```

### Bật X-Ray Insights

```bash
aws xray update-group \
  --group-name "ProductionErrors" \
  --insights-configuration InsightsEnabled=true,NotificationsEnabled=true
```

---

## Câu Hỏi Phỏng Vấn

**Q: X-Ray khác CloudWatch Logs như thế nào?**

> X-Ray theo dõi hành trình của request xuyên qua nhiều services (distributed tracing — theo dõi phân tán), cho thấy latency từng bước và dependency giữa services. CloudWatch Logs lưu trữ log events văn bản từng service riêng lẻ. Trong thực tế dùng kết hợp: X-Ray dẫn bạn đến service/trace có vấn đề → CloudWatch Logs cho bạn xem chi tiết lỗi trong service đó.

**Q: Segment khác Subsegment như thế nào?**

> Segment là đơn vị lớn nhất, đại diện cho công việc của một service (mỗi service tạo ra 1 Segment). Subsegment là công việc nhỏ hơn bên trong service: gọi DynamoDB, gọi HTTP API, logic tùy chỉnh. SDK tự động tạo Subsegments cho AWS SDK calls và HTTP requests khi đã `patch_all()`.

**Q: Tại sao không trace 100% requests?**

> Chi phí! X-Ray tính tiền theo số traces ($5/million traces). Với traffic 1,000 RPS → 86.4 triệu requests/ngày → quá đắt nếu trace 100%. Sampling 5% (default) giảm cost 20x trong khi vẫn đủ data để phân tích patterns. Trace 100% chỉ cho critical flows (checkout, payment) với custom sampling rules.

**Q: Annotations vs Metadata trong X-Ray?**

> Annotations là key-value có thể dùng để filter và search traces trong Console (indexed). Metadata là key-value phức tạp hơn (nested objects), chỉ xem khi xem trace detail, không filter được. Rule: data cần filter → Annotations; data debug chi tiết → Metadata.

**Q: Làm sao truyền trace ID qua nhiều services?**

> X-Ray dùng HTTP header `X-Amzn-Trace-Id: Root=...; Sampled=1`. Khi dùng X-Ray SDK đã `patch_all()` hoặc cấu hình AWS SDK tích hợp, header này được tự động thêm vào mọi outgoing HTTP call. Service nhận request đọc header và tiếp tục trace với cùng Trace ID → tất cả segments từ các services xuất hiện cùng trong 1 trace.

---

**← Trước:** [3-cloudwatch-alarms.md](./3-cloudwatch-alarms.md) | **Tiếp theo →** [5-observability-checklist.md](./5-observability-checklist.md)

**Cập Nhật Lần Cuối:** 2026-05-15 | **Phiên Bản:** 1.0
