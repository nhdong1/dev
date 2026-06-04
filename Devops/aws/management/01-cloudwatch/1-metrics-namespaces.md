# CloudWatch Metrics, Namespaces, Dimensions & Statistics

> **Metrics** (Chỉ Số) là nền tảng của CloudWatch — dữ liệu số theo thời gian (time-series data) được tổ chức trong Namespaces, phân loại bằng Dimensions, và tính toán qua Statistics.

---

## 📚 Mục Lục

1. [Metric Là Gì?](#metric-là-gì)
2. [Namespaces — Không Gian Tên](#namespaces--không-gian-tên)
3. [Dimensions — Chiều Phân Loại](#dimensions--chiều-phân-loại)
4. [Statistics — Thống Kê Tổng Hợp](#statistics--thống-kê-tổng-hợp)
5. [Metric Resolution & Retention](#metric-resolution--retention)
6. [Metric Math — Tính Toán Metric](#metric-math--tính-toán-metric)
7. [Custom Metrics — Chỉ Số Tùy Chỉnh](#custom-metrics--chỉ-số-tùy-chỉnh)
8. [AWS Built-in Metrics Quan Trọng](#aws-built-in-metrics-quan-trọng)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Metric Là Gì?

**Metric** là một chuỗi **datapoints** (điểm dữ liệu) theo thời gian, mỗi datapoint gồm:
- **Timestamp** — Thời điểm ghi nhận (UTC)
- **Value** — Giá trị số (double)
- **Unit** — Đơn vị đo (`Percent`, `Bytes`, `Count`, `Milliseconds`, `None`...)

```
Ví dụ: EC2 CPUUtilization
  2026-05-17 10:00:00  →  35.2  %
  2026-05-17 10:01:00  →  42.7  %
  2026-05-17 10:02:00  →  88.5  %  ← spike
  2026-05-17 10:03:00  →  41.1  %
```

Mỗi metric được nhận dạng **duy nhất** bởi tổ hợp: `Namespace + MetricName + Dimensions`.

---

## Namespaces — Không Gian Tên

**Namespace** là container logic để tách biệt metrics giữa các dịch vụ khác nhau. Giống như "folder" để tổ chức metrics.

### Namespace Của AWS Services

| Namespace           | Dịch Vụ                      | Ví Dụ Metric                          |
| ------------------- | ----------------------------- | ------------------------------------- |
| `AWS/EC2`           | EC2 Instances                 | `CPUUtilization`, `NetworkIn`         |
| `AWS/RDS`           | RDS / Aurora                  | `DatabaseConnections`, `ReadIOPS`     |
| `AWS/Lambda`        | Lambda Functions              | `Duration`, `Errors`, `Throttles`     |
| `AWS/ECS`           | ECS Services                  | `CPUUtilization`, `MemoryUtilization` |
| `AWS/ApplicationELB`| Application Load Balancer     | `RequestCount`, `TargetResponseTime`  |
| `AWS/S3`            | S3 Buckets                    | `BucketSizeBytes`, `NumberOfObjects`  |
| `AWS/SQS`           | SQS Queues                    | `NumberOfMessagesSent`, `QueueDepth`  |
| `AWS/DynamoDB`      | DynamoDB Tables               | `ConsumedReadCapacityUnits`           |
| `AWS/ApiGateway`    | API Gateway                   | `4XXError`, `5XXError`, `Latency`     |
| `AWS/SNS`           | SNS Topics                    | `NumberOfMessagesPublished`           |
| `CWAgent`           | CloudWatch Agent (tùy chỉnh)  | `mem_used_percent`, `disk_used_percent`|

### Custom Namespace (Namespace Tùy Chỉnh)

Khi gửi custom metrics, bạn tự đặt namespace. Convention (quy ước) phổ biến:

```
MyCompany/MyApplication      ← theo company/app
Production/OrderService      ← theo environment/service
Platform/Payments/v2         ← theo domain/service/version
```

> **Lưu ý:** Tên namespace phân biệt hoa thường (case-sensitive). `AWS/` prefix được dành riêng cho AWS — không tạo namespace bắt đầu bằng `AWS/`.

---

## Dimensions — Chiều Phân Loại

**Dimension** là cặp key-value (khóa-giá trị) bổ sung để phân loại và lọc metrics. Dimension giúp phân biệt các instances khác nhau của cùng một metric.

### Ví Dụ Thực Tế

```
Metric: CPUUtilization (Namespace: AWS/EC2)
  Dimension: InstanceId = i-0abc123    → CPU của instance cụ thể
  Dimension: InstanceId = i-0def456    → CPU của instance khác

Metric: Latency (Namespace: AWS/ApplicationELB)
  Dimension: LoadBalancer = app/my-alb/abc123
  Dimension: TargetGroup  = targetgroup/my-targets/xyz

Metric: OrderCount (Namespace: MyApp/Orders)
  Dimensions: {Region=us-east-1, Environment=prod, PaymentMethod=credit}
```

### Quy Tắc Về Dimensions

- Tối đa **30 dimensions** trên một metric
- Metric **không có dimension** và metric **có dimension** là các metric **khác nhau** (không tổng hợp được tự động)
- Khi query, phải cung cấp đúng tổ hợp dimensions để nhận dữ liệu chính xác

### Dimension Aggregation (Tổng Hợp Theo Chiều)

Không phải tất cả dịch vụ đều hỗ trợ tổng hợp tự động. Ví dụ với EC2:

```
# Xem CPUUtilization của TẤT CẢ instances trong Auto Scaling Group:
Namespace: AWS/EC2
Dimension: AutoScalingGroupName = my-asg

# Xem CPUUtilization của MỘT instance cụ thể:
Namespace: AWS/EC2
Dimension: InstanceId = i-0abc123

# Hai query trên cho số liệu KHÁC nhau và không thể so sánh trực tiếp
```

---

## Statistics — Thống Kê Tổng Hợp

**Statistics** (Thống Kê) là cách tổng hợp (aggregate) nhiều datapoints trong một period thành một giá trị duy nhất.

### Các Loại Statistics

| Statistic     | Ý Nghĩa                           | Khi Nào Dùng                              |
| ------------- | --------------------------------- | ----------------------------------------- |
| **Average**   | Trung bình cộng                   | CPU utilization, memory usage             |
| **Sum**       | Tổng cộng                         | Request count, bytes transferred          |
| **Minimum**   | Giá trị nhỏ nhất                  | Free storage space, minimum latency       |
| **Maximum**   | Giá trị lớn nhất                  | Peak CPU, maximum response time           |
| **SampleCount**| Số datapoints                    | Debug xem có đủ data không                |
| **pNN.NN**    | Percentile (Phân Vị)             | P50, P90, P95, P99 latency                |

### Percentile Statistics (Thống Kê Phân Vị)

Percentile là thống kê quan trọng nhất cho **latency** (độ trễ) và **performance**:

```
P50  = Median — 50% request hoàn thành nhanh hơn giá trị này
P90  = 90% request hoàn thành nhanh hơn giá trị này
P95  = 95% request hoàn thành nhanh hơn giá trị này
P99  = 99% request hoàn thành nhanh hơn giá trị này
P99.9= "Three nines" — 99.9% request tốt hơn

Ví dụ thực tế:
  Average Latency: 120ms   ← Nghe có vẻ tốt
  P99 Latency:     3500ms  ← 1% user chờ 3.5 giây! → Vấn đề thực sự
```

> **Tại sao P99 quan trọng hơn Average?** Average bị che khuất bởi phần lớn request nhanh. P99 lộ ra tail latency — kinh nghiệm của 1% user "không may" — và thường là dấu hiệu của vấn đề hệ thống thực sự.

### Chọn Đúng Statistic Theo Metric

```
CPUUtilization     → Average (trung bình fleet)
                   → Maximum (phát hiện spike)

RequestCount       → Sum (tổng request trong period)

Latency / Duration → Average (baseline)
                   → p99 (tail latency, SLA monitoring)
                   → Maximum (worst case detection)

Errors             → Sum (tổng lỗi)
                   → Average (error rate %)

FreeStorageSpace   → Minimum (disk thấp nhất trong period)
```

---

## Metric Resolution & Retention

### Standard Resolution vs High Resolution

| Loại                        | Granularity (Độ Chi Tiết) | Use Case                              | Giá              |
| --------------------------- | -------------------------- | ------------------------------------- | ---------------- |
| **Standard Resolution**     | 1 phút (60 giây)           | Hầu hết use case                      | Miễn phí (AWS metrics) |
| **High Resolution**         | 1 giây                     | Real-time monitoring, trading systems | $0.30/metric/tháng     |

### Data Retention (Thời Gian Lưu Trữ)

CloudWatch tự động roll up (tổng hợp lên) và lưu theo các granularity khác nhau:

```
Granularity    │  Retention (Lưu Trữ)
───────────────┼─────────────────────────────
1 giây         │  3 giờ
1 phút (60s)   │  15 ngày
5 phút         │  63 ngày
1 giờ          │  455 ngày (15 tháng)
```

> **Thực tế:** Sau 15 ngày, bạn không thể xem dữ liệu từng phút — chỉ thấy dữ liệu 5-phút. Sau 63 ngày, chỉ còn hourly data. Kế hoạch điều tra sự cố dài hạn phải tính đến giới hạn này.

---

## Metric Math — Tính Toán Metric

**Metric Math** (Toán Học Metric) cho phép thực hiện phép tính trên nhiều metrics để tạo ra metric mới, không cần lưu trữ riêng.

### Ví Dụ Thực Tế

**Error Rate (Tỷ Lệ Lỗi) từ Error Count và Request Count:**
```
EXPRESSION: errors / requests * 100

Metric m1: RequestCount (Sum) = 1000 requests/phút
Metric m2: HTTPCode_Target_5XX_Count (Sum) = 15 errors/phút
Result:    15 / 1000 * 100 = 1.5% error rate
```

**So Sánh Latency Giữa Các Services:**
```
Metric m1: OrderService P99 latency
Metric m2: PaymentService P99 latency
Metric m3: InventoryService P99 latency

EXPRESSION: MAX([m1, m2, m3])  → P99 latency cao nhất trong hệ thống
```

**Tính Disk Utilization (% Sử Dụng Đĩa):**
```
Metric m1: FreeStorageSpace (Bytes) = 20 GB
Metric m2: AllocatedStorage (Bytes) = 100 GB

EXPRESSION: (1 - m1/m2) * 100 = 80% used
```

### Hàm Metric Math Phổ Biến

| Hàm              | Ý Nghĩa                                     | Ví Dụ                              |
| ---------------- | ------------------------------------------- | ---------------------------------- |
| `SUM(metrics)`   | Tổng các metrics                            | Tổng request từ nhiều ALB          |
| `AVG(metrics)`   | Trung bình các metrics                      | CPU trung bình fleet               |
| `MAX(metrics)`   | Giá trị lớn nhất                            | Latency cao nhất                   |
| `MIN(metrics)`   | Giá trị nhỏ nhất                            | Free space thấp nhất               |
| `RATE(metric)`   | Tốc độ thay đổi (per second)                | Requests/second từ cumulative count|
| `DIFF(metric)`   | Sự thay đổi so với datapoint trước          | Delta trong counter                |
| `IF(cond,a,b)`   | Conditional (điều kiện)                     | Gán giá trị theo ngưỡng            |
| `FILL(metric,v)` | Điền giá trị vào khoảng trống               | Fill 0 khi không có data           |

---

## Custom Metrics — Chỉ Số Tùy Chỉnh

### Cách Gửi Custom Metrics

**Phương pháp 1: AWS SDK / CLI**

```bash
# Ví dụ gửi metric qua AWS CLI
aws cloudwatch put-metric-data \
  --namespace "MyApp/Orders" \
  --metric-name "OrdersPerMinute" \
  --value 42 \
  --unit Count \
  --dimensions Environment=prod,Region=us-east-1 \
  --timestamp "2026-05-17T10:00:00Z"
```

```python
# Python SDK (boto3)
import boto3
cloudwatch = boto3.client('cloudwatch')

cloudwatch.put_metric_data(
    Namespace='MyApp/Orders',
    MetricData=[{
        'MetricName': 'OrderProcessingTime',
        'Value': 245.7,
        'Unit': 'Milliseconds',
        'Dimensions': [
            {'Name': 'Service', 'Value': 'OrderService'},
            {'Name': 'Environment', 'Value': 'prod'}
        ]
    }]
)
```

**Phương pháp 2: CloudWatch Agent (khuyên dùng cho EC2)**

Cấu hình trong file `amazon-cloudwatch-agent.json`:
```json
{
  "metrics": {
    "namespace": "CWAgent",
    "metrics_collected": {
      "mem": {
        "measurement": ["mem_used_percent"],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": ["disk_used_percent"],
        "resources": ["/", "/data"]
      }
    }
  }
}
```

**Phương pháp 3: Embedded Metric Format (EMF — Định Dạng Metric Nhúng)**

Ghi metric trực tiếp trong log (tối ưu cho Lambda):
```python
import json, time

def handler(event, context):
    start = time.time()
    # ... xử lý ...
    duration = (time.time() - start) * 1000

    # EMF log — CloudWatch tự động tạo metric từ log này
    print(json.dumps({
        "_aws": {
            "Timestamp": int(time.time() * 1000),
            "CloudWatchMetrics": [{
                "Namespace": "MyApp",
                "Dimensions": [["FunctionName"]],
                "Metrics": [
                    {"Name": "ProcessingDuration", "Unit": "Milliseconds"},
                    {"Name": "ItemsProcessed", "Unit": "Count"}
                ]
            }]
        },
        "FunctionName": context.function_name,
        "ProcessingDuration": duration,
        "ItemsProcessed": 42
    }))
```

### So Sánh Các Phương Pháp Gửi Custom Metrics

| Phương Pháp   | Phù Hợp Với                | Ưu Điểm                         | Nhược Điểm               |
| ------------- | -------------------------- | -------------------------------- | ------------------------ |
| SDK/CLI       | Ứng dụng bất kỳ            | Linh hoạt, kiểm soát hoàn toàn  | Thêm API call, latency   |
| CW Agent      | EC2, on-premises            | Cấu hình đơn giản, thu OS metrics| Cần cài agent            |
| EMF           | Lambda, containers          | Không thêm latency, log = metric | Cần parse format đúng    |
| StatsD/collectd| Hệ thống hiện có           | Tái dùng infrastructure cũ      | Cần CW Agent làm bridge  |

---

## AWS Built-in Metrics Quan Trọng

### EC2 Metrics (Quan Trọng Nhất)

| Metric                   | Unit      | Ngưỡng Cảnh Báo Thường Dùng |
| ------------------------ | --------- | ----------------------------- |
| `CPUUtilization`         | Percent   | > 80% (sustained 5 phút)     |
| `NetworkIn` / `NetworkOut`| Bytes    | Tùy theo instance type        |
| `DiskReadOps` / `DiskWriteOps`| Count| Phụ thuộc workload           |
| `StatusCheckFailed`      | Count     | > 0 (instance unhealthy)     |
| `StatusCheckFailed_Instance`| Count  | > 0 (OS-level issue)         |
| `StatusCheckFailed_System`| Count    | > 0 (hardware issue)         |

> **EC2 KHÔNG tự động gửi:** RAM, Disk Space, Process count. Phải dùng CloudWatch Agent.

### RDS / Aurora Metrics

| Metric                   | Unit      | Ghi Chú                              |
| ------------------------ | --------- | ------------------------------------- |
| `DatabaseConnections`    | Count     | Alert khi gần max_connections         |
| `FreeStorageSpace`       | Bytes     | Alert khi < 20% tổng storage          |
| `ReadLatency`            | Seconds   | P99 > 100ms là vấn đề                 |
| `WriteLatency`           | Seconds   | P99 > 50ms là vấn đề                  |
| `ReplicaLag`             | Seconds   | Alert khi > 30s cho critical workloads|
| `CPUUtilization`         | Percent   | Alert > 80%                           |
| `FreeableMemory`         | Bytes     | Alert khi thấp                        |

### Lambda Metrics

| Metric                   | Unit          | Ghi Chú                                   |
| ------------------------ | ------------- | ------------------------------------------ |
| `Invocations`            | Count         | Tổng số lần gọi                            |
| `Duration`               | Milliseconds  | Thời gian chạy — dùng P99 để monitor       |
| `Errors`                 | Count         | Lỗi unhandled exception                    |
| `Throttles`              | Count         | Khi vượt concurrency limit                 |
| `ConcurrentExecutions`   | Count         | Số Lambda đang chạy đồng thời              |
| `IteratorAge`            | Milliseconds  | Cho Kinesis/DynamoDB triggers — lag của consumer|

### Application Load Balancer (ALB) Metrics

| Metric                        | Unit      | Ghi Chú                               |
| ----------------------------- | --------- | ------------------------------------- |
| `RequestCount`                | Count     | Tổng request                          |
| `HTTPCode_Target_2XX_Count`   | Count     | Successful responses                  |
| `HTTPCode_Target_4XX_Count`   | Count     | Client errors                         |
| `HTTPCode_Target_5XX_Count`   | Count     | Server errors — critical metric       |
| `TargetResponseTime`          | Seconds   | Latency từ ALB đến target             |
| `HealthyHostCount`            | Count     | Số target healthy                     |
| `UnHealthyHostCount`          | Count     | > 0 = có target bị loại khỏi rotation |

---

## Câu Hỏi Phỏng Vấn

### Q1: Tại sao phải dùng Dimensions thay vì tạo nhiều Metrics khác nhau?

**Dimensions** cho phép filter và group (nhóm) metrics một cách linh hoạt trong cùng một metric:
- Một metric `RequestCount` với dimension `{Service, Environment}` cho phép query theo service, theo environment, hoặc cả hai
- Nếu tạo metric riêng (`OrderServiceProdRequests`, `OrderServiceStagingRequests`...) thì không thể aggregate hoặc compare linh hoạt
- Dimensions cũng giúp tối ưu chi phí — mỗi unique metric (namespace + name + dimensions set) tính phí riêng

### Q2: High-Resolution Metrics thực sự cần thiết không?

**Hiếm khi cần.** Standard (1 phút) đủ cho 95% use case. High-Resolution (1 giây) chỉ cần khi:
- Trading systems cần phát hiện spike trong seconds
- Auto Scaling cần phản ứng rất nhanh
- Real-time gaming hoặc streaming latency

Chi phí tương đương nhưng phức tạp hơn. Đánh giá kỹ trước khi bật.

### Q3: Sau bao lâu thì CloudWatch xóa metric data?

CloudWatch **không xóa** — nhưng sẽ roll up lên granularity thô hơn:
- High-resolution (1s) → chỉ giữ 3 giờ
- 1 phút → giữ 15 ngày rồi merge vào 5-phút
- 5 phút → giữ 63 ngày rồi merge vào 1-giờ
- 1 giờ → giữ 455 ngày

Sau 455 ngày, data không còn trong CloudWatch. Nếu cần lưu lâu hơn → export sang S3.

### Q4: Làm thế nào tính error rate mà không gửi thêm metric?

Dùng **Metric Math** kết hợp hai metrics đã có:

```
Metrics:
  m1 = HTTPCode_Target_5XX_Count (Sum)
  m2 = RequestCount (Sum)

Expression: (m1 / m2) * 100   → Error Rate (%)
```

Có thể đặt Alarm trực tiếp lên expression này — không cần gửi thêm custom metric nào.

### Q5: Sự khác biệt giữa `StatusCheckFailed_Instance` và `StatusCheckFailed_System`?

| Check                         | Phát Hiện                            | Cách Khắc Phục                    |
| ----------------------------- | ------------------------------------ | ---------------------------------- |
| `StatusCheckFailed_Instance`  | Vấn đề trong OS/software của instance| Stop/Start instance, sửa OS       |
| `StatusCheckFailed_System`    | Vấn đề hardware/network dưới AWS     | AWS tự xử lý; có thể cần recover  |
| `StatusCheckFailed`           | Bất kỳ cái nào ở trên               | Alert chung                        |

Khi `StatusCheckFailed_System = 1`, bạn có thể cấu hình EC2 Auto Recovery để AWS tự động di chuyển instance sang hardware tốt hơn.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
**Liên Kết:** [README.md](./README.md) | [2-alarms-composite.md](./2-alarms-composite.md)
