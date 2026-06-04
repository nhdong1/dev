# 📈 Các Metrics Quan Trọng Theo Dịch Vụ

> Danh sách đầy đủ các CloudWatch Metrics (Số Liệu CloudWatch) cần giám sát cho mỗi dịch vụ AWS Application Integration, kèm ý nghĩa và ngưỡng khuyến nghị.

## 📚 Mục Lục

1. [Amazon SQS Metrics](#amazon-sqs-metrics)
2. [Amazon SNS Metrics](#amazon-sns-metrics)
3. [Amazon EventBridge Metrics](#amazon-eventbridge-metrics)
4. [AWS Step Functions Metrics](#aws-step-functions-metrics)
5. [Amazon Kinesis Metrics](#amazon-kinesis-metrics)
6. [AWS Lambda Metrics (Consumer)](#aws-lambda-metrics-consumer)
7. [Tạo Custom Metrics](#tạo-custom-metrics)
8. [Dashboard Mẫu](#dashboard-mẫu)

---

## Amazon SQS Metrics

### Metrics Quan Trọng Nhất

| Metric | Ý Nghĩa | Ngưỡng Cảnh Báo |
|---|---|---|
| `ApproximateNumberOfMessagesVisible` | Số tin nhắn đang chờ được xử lý trong queue | > 1000 (tùy use case) |
| `ApproximateNumberOfMessagesNotVisible` | Số tin nhắn đang được xử lý (trong Visibility Timeout — Thời Gian Ẩn) | Tăng đột biến = vấn đề |
| `ApproximateAgeOfOldestMessage` | Tuổi của tin nhắn cũ nhất tính bằng giây | > Message Retention / 2 |
| `NumberOfMessagesSent` | Số tin nhắn được gửi vào queue mỗi phút | — (dùng để baseline) |
| `NumberOfMessagesReceived` | Số tin nhắn consumer nhận được mỗi phút | — (so sánh với Sent) |
| `NumberOfMessagesDeleted` | Số tin nhắn đã xử lý xong và xoá khỏi queue | — |
| `NumberOfEmptyReceives` | Số lần poll (lấy tin nhắn) trả về rỗng | Cao = lãng phí, dùng Long Polling |

### Metric Quan Trọng Nhất: `ApproximateAgeOfOldestMessage`

**Tại sao đây là metric vàng của SQS:**

```
Ví dụ:
- Queue có 10,000 tin nhắn chờ
- Consumer đang xử lý 1,000 tin nhắn/phút
→ Ước tính sẽ mất 10 phút để xử lý hết

Nhưng nếu tin nhắn cũ nhất đã 2 giờ tuổi:
→ Consumer bị chặn hoặc không đủ nhanh
→ Cần tăng số lượng consumer (scale out)
```

**Cách tính ngưỡng cảnh báo:**

```
Message Retention Period (Chu Kỳ Lưu Giữ) mặc định = 4 ngày = 345,600 giây
Cảnh báo sớm: ApproximateAgeOfOldestMessage > 3,600 (1 giờ)
Cảnh báo khẩn: ApproximateAgeOfOldestMessage > 86,400 (1 ngày)
```

### Dead Letter Queue Metrics

```
DLQ không có metric built-in riêng —
Dùng ApproximateNumberOfMessagesVisible trên DLQ queue:
- = 0: Bình thường
- > 0: CÓ tin nhắn lỗi — cần xem xét ngay
- Tăng liên tục: Poison message (tin nhắn độc hại) đang hoạt động
```

Xem thêm: [4-dlq-monitoring.md](4-dlq-monitoring.md)

---

## Amazon SNS Metrics

### Metrics Quan Trọng

| Metric | Ý Nghĩa | Ghi Chú |
|---|---|---|
| `NumberOfMessagesPublished` | Số tin nhắn được publish (phát hành) lên topic mỗi phút | Đo lường throughput của producer |
| `NumberOfNotificationsDelivered` | Số tin nhắn đã giao thành công đến subscriber (người đăng ký) | |
| `NumberOfNotificationsFailed` | Số tin nhắn giao thất bại | Tỷ lệ > 1% cần điều tra |
| `NumberOfNotificationsFilteredOut` | Số tin nhắn bị lọc không giao | Do Message Filter không khớp |
| `NumberOfNotificationsRedrivenToDlq` | Số tin nhắn chuyển vào DLQ của subscription | Cần alarm ngay khi > 0 |
| `PublishSize` | Kích thước tin nhắn tính bằng byte | Tiếp cận giới hạn 256KB = vấn đề |

### Tỷ Lệ Giao Thành Công

```
Delivery Success Rate (Tỷ Lệ Giao Thành Công):
= NumberOfNotificationsDelivered / NumberOfMessagesPublished × 100%

Ngưỡng:
- > 99.9%: Bình thường
- 95–99.9%: Cần điều tra
- < 95%: Cảnh báo khẩn cấp
```

### SNS Delivery Status Logging

Bật **Delivery Status Logging** (Ghi Nhật Ký Trạng Thái Giao Vận) để ghi chi tiết mỗi lần giao:

```json
{
  "notification": {
    "messageMD5Sum": "abc123",
    "messageId": "uuid",
    "subscriptionArn": "arn:aws:sns:..."
  },
  "delivery": {
    "statusCode": 200,
    "dwellTimeMs": 15,
    "destination": "https://..."
  },
  "status": "SUCCESS"
}
```

---

## Amazon EventBridge Metrics

### Metrics Quan Trọng

| Metric | Ý Nghĩa | Khi Nào Cảnh Báo |
|---|---|---|
| `Invocations` | Số lần rule (quy tắc) được kích hoạt và gọi target | Giảm đột biến = rule bị tắt hoặc event không đến |
| `FailedInvocations` | Số lần gọi target thất bại | > 0 cần xem ngay |
| `ThrottledRules` | Số lần rule bị throttle (giới hạn tốc độ) | > 0 = đang gần giới hạn |
| `MatchedEvents` | Số event khớp với ít nhất một rule | So sánh với tổng event để biết tỷ lệ lọc |
| `DeadLetterInvocations` | Số event chuyển vào DLQ của EventBridge | Cần alarm ngay khi > 0 |

### EventBridge Pipe Metrics

Nếu dùng **EventBridge Pipes** (Đường Ống EventBridge):

| Metric | Ý Nghĩa |
|---|---|
| `ExecutionThrottled` | Pipe bị throttle — cần tăng concurrency limit |
| `ExecutionFailed` | Pipe thực thi thất bại — check enrichment/target |
| `ExecutionStarted` | Số lần pipe bắt đầu xử lý |

### Cross-Account Event Monitoring

Khi dùng Event Bus liên tài khoản (Cross-Account Event Bus):

```
Account A (Producer):
- Theo dõi PutEvents success/failure

Account B (Consumer):
- Theo dõi Invocations và FailedInvocations
- Phải có rule trên Custom Bus mới có metrics
```

---

## AWS Step Functions Metrics

### Metrics Quan Trọng

| Metric | Ý Nghĩa | Ngưỡng |
|---|---|---|
| `ExecutionsStarted` | Số lần thực thi state machine bắt đầu | Cơ sở để tính tỷ lệ |
| `ExecutionsSucceeded` | Số lần thực thi hoàn thành thành công | |
| `ExecutionsFailed` | Số lần thực thi thất bại vĩnh viễn | > 0 cần điều tra |
| `ExecutionsAborted` | Số lần thực thi bị huỷ (thủ công hoặc timeout) | |
| `ExecutionsTimedOut` | Số lần thực thi vượt quá timeout | > 0 = cần review timeout config |
| `ExecutionTime` | Thời gian hoàn thành một lần thực thi (ms) | Tùy SLA của workflow |
| `ExecutionThrottled` | Số lần thực thi bị throttle | > 0 = đang đạt rate limit |

### Tỷ Lệ Thành Công

```python
# Công thức tính Success Rate (Tỷ Lệ Thành Công)
success_rate = ExecutionsSucceeded / ExecutionsStarted * 100

# Ví dụ: 950 thành công / 1000 bắt đầu = 95%
# Nếu < 99% trong production → cần điều tra ngay
```

### Phân Tích Theo State (Trạng Thái)

Step Functions không cung cấp metric theo từng state mặc định. Dùng **X-Ray** hoặc **CloudWatch Logs** để phân tích:

```json
// Bật logging trong Step Functions
{
  "loggingConfiguration": {
    "level": "ALL",
    "includeExecutionData": true,
    "destinations": [{
      "cloudWatchLogsLogGroup": {
        "logGroupArn": "arn:aws:logs:..."
      }
    }]
  }
}
```

---

## Amazon Kinesis Metrics

### Kinesis Data Streams — Metrics Cơ Bản

| Metric | Ý Nghĩa | Ngưỡng Cảnh Báo |
|---|---|---|
| `GetRecords.IteratorAgeMilliseconds` | Tuổi của record cũ nhất consumer đang đọc (ms) | > 60,000 (1 phút) = lag |
| `GetRecords.Bytes` | Byte được đọc bởi consumer | Gần 2MB/s/shard = throttle |
| `PutRecord.Success` | Tỷ lệ thành công của PutRecord | < 99% = cần kiểm tra |
| `WriteProvisionedThroughputExceeded` | Số lần ghi bị từ chối vì vượt throughput | > 0 = cần thêm shard |
| `ReadProvisionedThroughputExceeded` | Số lần đọc bị từ chối vì vượt throughput | > 0 = dùng Enhanced Fan-Out |
| `IncomingRecords` | Số record được ghi vào stream mỗi phút | Tăng đột biến = event spike |
| `IncomingBytes` | Byte ghi vào stream mỗi phút | Gần giới hạn = cần shard thêm |

### Metric Vàng: `GetRecords.IteratorAgeMilliseconds`

**Iterator Age** (Tuổi Iterator) là khoảng thời gian từ khi record được ghi vào stream đến khi consumer đọc được. Đây là metric quan trọng nhất để đo **consumer lag** (độ trễ người tiêu dùng):

```
Iterator Age = 0ms: Consumer đang theo kịp real-time
Iterator Age = 5000ms (5 giây): Consumer đang lag nhẹ — theo dõi
Iterator Age = 60,000ms (1 phút): Consumer lag đáng kể — cần scale
Iterator Age = 3,600,000ms (1 giờ): Khủng hoảng — dữ liệu có thể mất nếu retention ngắn
```

**Tính toán cần thêm bao nhiêu shard:**

```
Công thức:
Shard cần thiết = max(IncomingRate / 1000, IncomingBytes / 1MB)

Ví dụ:
- Incoming: 3,000 records/giây, mỗi record 500 bytes
- Theo records: 3000 / 1000 = 3 shard
- Theo bytes: 3000 * 500 / 1,000,000 = 1.5 shard
→ Cần 3 shard (lấy giá trị lớn hơn)
```

### Kinesis Firehose Metrics

| Metric | Ý Nghĩa |
|---|---|
| `DeliveryToS3.Success` | Tỷ lệ giao thành công vào S3 |
| `DeliveryToS3.DataFreshness` | Độ trễ từ khi nhận đến khi ghi vào S3 (giây) |
| `ThrottledRecords` | Số record bị throttle — cần tăng limit |

---

## AWS Lambda Metrics (Consumer)

Lambda thường là consumer (người tiêu dùng) xử lý message từ SQS, Kinesis, EventBridge. Các metrics của Lambda liên quan đến integration:

| Metric | Ý Nghĩa | Ngưỡng |
|---|---|---|
| `Errors` | Số lần Lambda thực thi thất bại | > 0 = có vấn đề |
| `Throttles` | Số lần Lambda bị throttle | > 0 = cần tăng concurrency limit |
| `Duration` | Thời gian thực thi mỗi invocation (ms) | Gần timeout = cần tối ưu |
| `ConcurrentExecutions` | Số Lambda đang chạy đồng thời | Gần account limit = nguy hiểm |
| `IteratorAge` | (Với Kinesis trigger) Tuổi của batch hiện tại | > 60,000ms = lag |

### Kết Hợp SQS + Lambda

```
Kịch bản tối ưu:
1. SQS ApproximateNumberOfMessagesVisible tăng
   → CloudWatch Alarm kích hoạt
   → Lambda auto-scaling tăng concurrency
   → Số message giảm về 0

Nếu Lambda Errors tăng cùng lúc:
→ Consumer đang gặp lỗi, không phải do thiếu capacity
→ Check message format, dependencies
```

---

## Tạo Custom Metrics

Khi cần đo lường business logic không có sẵn trong AWS metrics, dùng **CloudWatch Custom Metrics** (Số Liệu Tùy Chỉnh):

### Ví Dụ: Đo Thời Gian Xử Lý Đơn Hàng

```python
import boto3
import time

cloudwatch = boto3.client('cloudwatch')

def process_order(order):
    start_time = time.time()
    
    # Xử lý đơn hàng...
    result = do_process(order)
    
    processing_time_ms = (time.time() - start_time) * 1000
    
    # Gửi custom metric
    cloudwatch.put_metric_data(
        Namespace='OrderService',  # Namespace (Không Gian Tên) tùy chỉnh
        MetricData=[
            {
                'MetricName': 'OrderProcessingTime',
                'Value': processing_time_ms,
                'Unit': 'Milliseconds',
                'Dimensions': [
                    {'Name': 'Environment', 'Value': 'production'},
                    {'Name': 'OrderType', 'Value': order['type']}
                ]
            },
            {
                'MetricName': 'OrdersProcessed',
                'Value': 1,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'Status', 'Value': 'Success' if result else 'Failed'}
                ]
            }
        ]
    )
```

### Namespace (Không Gian Tên) Khuyến Nghị

```
Quy ước đặt tên Namespace:
{CompanyName}/{ServiceName}/{Environment}

Ví dụ:
- MyCompany/OrderService/Production
- MyCompany/PaymentService/Staging
- MyCompany/NotificationService/Development
```

---

## Dashboard Mẫu

### Layout Dashboard Khuyến Nghị

```
+------------------+------------------+------------------+
| SQS Queue Depth  | Lambda Errors    | DLQ Messages     |
| (Line chart)     | (Bar chart)      | (Number widget)  |
+------------------+------------------+------------------+
| Kinesis Iterator | Step Functions   | SNS Delivery     |
| Age              | Success Rate     | Success Rate     |
| (Line chart)     | (Gauge)          | (Gauge)          |
+------------------+------------------+------------------+
| Lambda Duration  | EventBridge      | Custom: Orders   |
| P50/P95/P99      | Failed Invoc.    | Processed/min    |
| (Line chart)     | (Number widget)  | (Line chart)     |
+------------------+------------------+------------------+
```

### Tạo Dashboard Bằng AWS CLI

```bash
# Tạo dashboard JSON
aws cloudwatch put-dashboard \
  --dashboard-name "AppIntegration-Overview" \
  --dashboard-body file://dashboard.json

# Xem danh sách dashboard
aws cloudwatch list-dashboards

# Lấy nội dung dashboard
aws cloudwatch get-dashboard \
  --dashboard-name "AppIntegration-Overview"
```

---

## Tóm Tắt: Top 10 Metrics Phải Theo Dõi

| Thứ Tự | Service | Metric | Lý Do Quan Trọng |
|---|---|---|---|
| 1 | SQS | `ApproximateAgeOfOldestMessage` | Phát hiện consumer lag |
| 2 | SQS DLQ | `ApproximateNumberOfMessagesVisible` | Phát hiện tin nhắn lỗi |
| 3 | Kinesis | `GetRecords.IteratorAgeMilliseconds` | Đo độ trễ streaming |
| 4 | Kinesis | `WriteProvisionedThroughputExceeded` | Phát hiện thiếu shard |
| 5 | Lambda | `Errors` | Phát hiện consumer lỗi |
| 6 | Lambda | `Throttles` | Phát hiện thiếu concurrency |
| 7 | Step Functions | `ExecutionsFailed` | Phát hiện workflow lỗi |
| 8 | SNS | `NumberOfNotificationsFailed` | Phát hiện giao vận thất bại |
| 9 | EventBridge | `FailedInvocations` | Phát hiện target không phản hồi |
| 10 | Custom | `OrdersProcessedPerMinute` | Đo lường business outcome |

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Xem Tiếp:** [2-cloudwatch-alarms.md](2-cloudwatch-alarms.md) — Cách tạo alarm dựa trên các metrics này
