# S3 Event Notifications — Thông Báo Sự Kiện

> S3 Event Notifications (Thông Báo Sự Kiện S3) cho phép S3 tự động gửi thông báo đến các dịch vụ khác khi có sự kiện xảy ra trong bucket (upload, xóa, replication, v.v.). Là nền tảng cho event-driven architecture (kiến trúc hướng sự kiện) với chi phí thấp và độ trễ cực nhỏ.

## 📚 Mục Lục

1. [Event Notifications Là Gì?](#1-event-notifications-là-gì)
2. [Các Loại Sự Kiện](#2-các-loại-sự-kiện)
3. [Destinations — Đích Đến](#3-destinations--đích-đến)
4. [Cấu Hình Event Notifications](#4-cấu-hình-event-notifications)
5. [S3 với EventBridge — So Sánh Cách Tiếp Cận](#5-s3-với-eventbridge--so-sánh-cách-tiếp-cận)
6. [Kiến Trúc Thực Tế](#6-kiến-trúc-thực-tế)
7. [Xử Lý Lỗi Và Retry](#7-xử-lý-lỗi-và-retry)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Event Notifications Là Gì?

Khi object trong S3 bucket có sự kiện (upload, xóa, restore từ Glacier, v.v.), S3 gửi notification (thông báo) đến destination (đích) đã cấu hình.

### Luồng Cơ Bản

```
     Client
       │
       │ PUT object
       ▼
  S3 Bucket ──── Event xảy ra ──── S3 phát thông báo ──▶ SQS Queue
                                                     └──▶ SNS Topic
                                                     └──▶ Lambda Function
                                                     └──▶ EventBridge
```

### Đặc Điểm Chính

- **Near-realtime (Gần Thời Gian Thực):** Thông báo thường trong vài giây sau khi sự kiện
- **At-least-once delivery (Gửi Ít Nhất Một Lần):** Có thể nhận duplicate notifications — consumers cần idempotent (lũy đẳng)
- **No cost for configuration (Không Phí Cấu Hình):** Chỉ tính phí ở destination (SQS, SNS, Lambda)
- **Per-bucket configuration:** Mỗi bucket có cấu hình notification riêng

---

## 2. Các Loại Sự Kiện

### 2.1 ObjectCreated Events — Sự Kiện Tạo Object

| Event | Mô Tả |
| ----- | ------ |
| `s3:ObjectCreated:Put` | Upload bằng PUT request |
| `s3:ObjectCreated:Post` | Upload qua HTML form POST |
| `s3:ObjectCreated:Copy` | Copy từ object khác |
| `s3:ObjectCreated:CompleteMultipartUpload` | Hoàn thành multipart upload |
| `s3:ObjectCreated:*` | Tất cả loại tạo object |

### 2.2 ObjectRemoved Events — Sự Kiện Xóa Object

| Event | Mô Tả |
| ----- | ------ |
| `s3:ObjectRemoved:Delete` | Xóa object (không versioning hoặc delete specific version) |
| `s3:ObjectRemoved:DeleteMarkerCreated` | Tạo delete marker (versioning bucket) |
| `s3:ObjectRemoved:*` | Tất cả loại xóa |

### 2.3 ObjectRestore Events — Sự Kiện Restore Từ Glacier

| Event | Mô Tả |
| ----- | ------ |
| `s3:ObjectRestore:Post` | Bắt đầu yêu cầu restore |
| `s3:ObjectRestore:Completed` | Restore hoàn thành, có thể tải object |
| `s3:ObjectRestore:Delete` | Bản restore tạm thời bị xóa |

### 2.4 Replication Events — Sự Kiện Sao Chép

| Event | Mô Tả |
| ----- | ------ |
| `s3:Replication:OperationFailedReplication` | Object không được replicate (lỗi) |
| `s3:Replication:OperationMissedThreshold` | Object chưa được replicate đúng hạn (RTC breach) |
| `s3:Replication:OperationReplicatedAfterThreshold` | Replicate sau khi quá hạn |
| `s3:Replication:OperationNotTracked` | Object không được track replication |

### 2.5 Các Sự Kiện Khác

| Event | Mô Tả |
| ----- | ------ |
| `s3:ObjectTagging:Put` | Thêm hoặc cập nhật tag |
| `s3:ObjectTagging:Delete` | Xóa tag |
| `s3:ObjectAcl:Put` | Thay đổi ACL object |
| `s3:LifecycleTransition` | Object chuyển sang storage class khác |
| `s3:IntelligentTiering` | Object chuyển trong Intelligent-Tiering |

---

## 3. Destinations — Đích Đến

### 3.1 Amazon SQS — Simple Queue Service (Hàng Đợi Đơn Giản)

```
Khi dùng:
- Xử lý bất đồng bộ, không cần trả lời ngay
- Nhiều consumers (người tiêu dùng) xử lý cùng lúc (không dùng FIFO)
- Muốn retry tự động khi consumer fail
- Hàng đợi làm buffer (vùng đệm) khi có spike traffic

Cấu hình SQS Queue Policy để S3 được phép gửi:
{
  "Effect": "Allow",
  "Principal": { "Service": "s3.amazonaws.com" },
  "Action": "sqs:SendMessage",
  "Resource": "arn:aws:sqs:us-east-1:123456789012:my-queue",
  "Condition": {
    "ArnLike": {
      "aws:SourceArn": "arn:aws:s3:::my-bucket"
    }
  }
}
```

### 3.2 Amazon SNS — Simple Notification Service (Dịch Vụ Thông Báo)

```
Khi dùng:
- Fan-out (phát tán): Một event gửi đến nhiều subscribers
- Email/SMS alert khi có sự kiện đặc biệt
- Kết hợp SNS → SQS cho fan-out + reliable delivery

SNS Queue Policy tương tự SQS — cho phép s3.amazonaws.com publish.

Fan-out pattern:
              ┌──▶ SQS Queue A (processing worker)
S3 Event ──▶ SNS ──▶ SQS Queue B (audit logger)
              └──▶ Lambda (real-time alert)
```

### 3.3 AWS Lambda Function

```
Khi dùng:
- Xử lý inline (trực tiếp) ngay khi có event
- Transform dữ liệu: resize ảnh, convert format
- Validate và enrich object metadata
- Trigger downstream pipeline

Lambda Resource Policy cho phép S3 invoke:
{
  "Effect": "Allow",
  "Principal": { "Service": "s3.amazonaws.com" },
  "Action": "lambda:InvokeFunction",
  "Resource": "arn:aws:lambda:us-east-1:123456789012:function:my-function",
  "Condition": {
    "ArnLike": {
      "AWS:SourceArn": "arn:aws:s3:::my-bucket"
    }
  }
}
```

### 3.4 Amazon EventBridge — Cầu Nối Sự Kiện

```
Khi dùng:
- Routing linh hoạt: filter event dựa trên content của JSON
- Kết nối với 200+ AWS services và SaaS applications
- Archiving và replay events
- Schema discovery (khám phá lược đồ) tự động

Khác với notification trực tiếp:
- Phải bật CloudTrail data events HOẶC bật "Send notifications to EventBridge" ở bucket
- Event detail phong phú hơn (full S3 API context)
- Có thể tạo event pattern phức tạp hơn
```

---

## 4. Cấu Hình Event Notifications

### 4.1 Qua AWS CLI

```bash
# Đặt notification configuration cho bucket
aws s3api put-bucket-notification-configuration \
  --bucket my-bucket \
  --notification-configuration file://notification.json
```

### 4.2 File notification.json — Ví Dụ Đầy Đủ

```json
{
  "LambdaFunctionConfigurations": [
    {
      "Id": "ProcessImageOnUpload",
      "LambdaFunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:resize-image",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {
          "FilterRules": [
            { "Name": "prefix", "Value": "uploads/" },
            { "Name": "suffix", "Value": ".jpg" }
          ]
        }
      }
    },
    {
      "Id": "AlertOnDelete",
      "LambdaFunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:delete-alert",
      "Events": ["s3:ObjectRemoved:*"],
      "Filter": {
        "Key": {
          "FilterRules": [
            { "Name": "prefix", "Value": "production/" }
          ]
        }
      }
    }
  ],
  "QueueConfigurations": [
    {
      "Id": "SendToProcessingQueue",
      "QueueArn": "arn:aws:sqs:us-east-1:123456789012:document-processing",
      "Events": ["s3:ObjectCreated:CompleteMultipartUpload"],
      "Filter": {
        "Key": {
          "FilterRules": [
            { "Name": "prefix", "Value": "documents/" },
            { "Name": "suffix", "Value": ".pdf" }
          ]
        }
      }
    }
  ],
  "TopicConfigurations": [
    {
      "Id": "ReplicationAlert",
      "TopicArn": "arn:aws:sns:us-east-1:123456789012:ops-alerts",
      "Events": ["s3:Replication:OperationFailedReplication"]
    }
  ],
  "EventBridgeConfiguration": {
    "EventBridgeEnabled": true
  }
}
```

### 4.3 Qua Terraform

```hcl
resource "aws_s3_bucket_notification" "bucket_notification" {
  bucket = aws_s3_bucket.example.id

  lambda_function {
    id                  = "ProcessImage"
    lambda_function_arn = aws_lambda_function.resize.arn
    events              = ["s3:ObjectCreated:*"]
    filter_prefix       = "uploads/"
    filter_suffix       = ".jpg"
  }

  queue {
    id        = "QueueDocuments"
    queue_arn = aws_sqs_queue.documents.arn
    events    = ["s3:ObjectCreated:CompleteMultipartUpload"]
    filter_prefix = "documents/"
  }

  eventbridge = true

  depends_on = [
    aws_lambda_permission.allow_s3,
    aws_sqs_queue_policy.allow_s3
  ]
}
```

### 4.4 Lambda Event Payload — Cấu Trúc JSON Nhận Được

```json
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventTime": "2026-05-15T10:00:00.000Z",
      "eventName": "ObjectCreated:Put",
      "s3": {
        "s3SchemaVersion": "1.0",
        "bucket": {
          "name": "my-bucket",
          "arn": "arn:aws:s3:::my-bucket"
        },
        "object": {
          "key": "uploads/photo.jpg",
          "size": 1024000,
          "eTag": "d41d8cd98f00b204e9800998ecf8427e",
          "versionId": "abc123"
        }
      }
    }
  ]
}
```

---

## 5. S3 với EventBridge — So Sánh Cách Tiếp Cận

| Tiêu Chí | S3 Native Notifications | S3 + EventBridge |
| -------- | ----------------------- | ---------------- |
| Độ trễ | Thấp hơn | Cao hơn một chút |
| Filtering | Chỉ theo prefix/suffix | Content-based filtering phức tạp |
| Fan-out | Tối đa 3 destinations | 5 targets mặc định, có thể tăng |
| Dead Letter Queue | SQS DLQ (tùy cấu hình) | EventBridge DLQ built-in |
| Retry | Phụ thuộc destination | Retry có cấu hình |
| Archiving events | Không | Có (replay events) |
| Schema registry | Không | Có |
| Cost | Không phí notification | $1/million events |
| 200+ integration | Không | Có |
| Monitoring | CloudWatch (cơ bản) | CloudWatch + EventBridge metrics |

### Khi Nào Dùng EventBridge?

```
Dùng EventBridge khi:
✅ Cần routing phức tạp dựa trên nội dung event
✅ Kết nối với SaaS (Salesforce, Zendesk, v.v.)
✅ Cần archive và replay events
✅ Multi-account event routing
✅ Cần schema validation

Dùng Native Notification khi:
✅ Cần độ trễ thấp nhất
✅ Use case đơn giản (upload → Lambda hoặc SQS)
✅ Muốn tiết kiệm chi phí
✅ Không cần tính năng nâng cao
```

---

## 6. Kiến Trúc Thực Tế

### Kiến Trúc 1: Image Processing Pipeline — Xử Lý Hình Ảnh

```
User upload ảnh → S3 (uploads/)
                     │
              s3:ObjectCreated:*
                     │
              Lambda: resize-function
              ├── Resize to thumbnail (128×128)
              ├── Resize to medium (800×600)
              └── Lưu vào S3 (processed/)
                     │
              s3:ObjectCreated (processed/)
                     │
              Lambda: cdn-invalidation
              └── CloudFront invalidation

Bucket: uploads → processed (tách biệt)
```

### Kiến Trúc 2: Document Indexing — Lập Chỉ Mục Tài Liệu

```
Upload PDF → S3 (documents/)
                │
         CompleteMultipartUpload event
                │
             SQS Queue (buffer)
                │
         Worker fleet (EC2/ECS)
         ├── Textract — trích xuất text từ PDF
         ├── Comprehend — phân tích ngôn ngữ
         └── OpenSearch — lập chỉ mục để search

Ưu điểm SQS ở giữa:
- Buffer khi nhiều upload cùng lúc
- Worker có thể scale independently
- Retry khi worker fail
```

### Kiến Trúc 3: Real-time Audit — Kiểm Toán Thời Gian Thực

```
Production bucket → s3:ObjectRemoved:* event
                          │
                      SNS Topic
                    ┌─────┴─────┐
                  SQS          Lambda
                   │            │
               Audit Log     PagerDuty Alert
               (S3/OpenSearch)  (nếu xóa object production)
```

### Kiến Trúc 4: Data Lake Ingestion — Nạp Dữ Liệu Data Lake

```
Partner gửi file CSV → S3 (raw/partner-name/date/)
                            │
                     s3:ObjectCreated
                            │
                       Lambda: validator
                       ├── Validate schema CSV
                       ├── Check data quality
                       ├── Nếu valid → copy sang s3://processed/
                       └── Nếu invalid → copy sang s3://quarantine/ + SNS alert
```

---

## 7. Xử Lý Lỗi Và Retry

### S3 → Lambda: Xử Lý Lỗi

S3 invoke Lambda **synchronously không** — S3 gửi notification và không chờ kết quả. Nếu Lambda fail:

```
S3 gửi notification → Lambda fail
→ S3 KHÔNG retry
→ Event bị mất

Giải pháp: Lambda Destinations
- Configure Lambda để khi fail → gửi sang SQS DLQ (Dead Letter Queue — Hàng Đợi Thư Chết)
- Monitor DLQ để detect failed events
- Reprocess từ DLQ khi đã fix lỗi
```

### S3 → SQS: Xử Lý Lỗi

```
S3 gửi message → SQS nhận message → Consumer xử lý
                                        │
                                     Fail → Message visible lại sau Visibility Timeout
                                          → Retry tự động
                                          → Sau N lần → DLQ
```

### Idempotency — Lũy Đẳng

At-least-once delivery nghĩa là một event có thể được gửi nhiều lần. Consumer phải xử lý idempotent:

```python
def lambda_handler(event, context):
    for record in event['Records']:
        object_key = record['s3']['object']['key']
        etag = record['s3']['object']['eTag']
        
        # Dùng ETag để kiểm tra đã xử lý chưa
        if already_processed(object_key, etag):
            print(f"Already processed {object_key}, skipping")
            continue
        
        process_object(object_key)
        mark_as_processed(object_key, etag)
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q1: S3 Event Notifications đảm bảo giao tin exactly-once không?**

> Không. S3 Event Notifications cung cấp at-least-once delivery — có thể nhận duplicate notifications cho cùng một event. Consumer phải thiết kế idempotent, nghĩa là xử lý cùng một event nhiều lần không gây ra tác dụng phụ. Ví dụ: dùng ETag + object key làm idempotency key, lưu vào DynamoDB để kiểm tra đã xử lý chưa.

**Q2: Khi nào dùng S3 → SQS thay vì S3 → Lambda trực tiếp?**

> Dùng SQS ở giữa khi: (1) Upload rate cao, cần buffer để tránh throttle Lambda, (2) Xử lý mất nhiều thời gian (Lambda max 15 phút, SQS giữ message 14 ngày), (3) Cần retry tự động khi xử lý fail, (4) Nhiều loại consumer (worker khác nhau) cùng xử lý cùng queue. Dùng Lambda trực tiếp khi cần xử lý nhanh (<15 giây), đơn giản, ít event.

**Q3: Tại sao không thể dùng cùng prefix làm cả nguồn và đích notification?**

> Sẽ gây infinite loop: Lambda xử lý event → write output vào cùng bucket/prefix → trigger event mới → Lambda lại chạy → vô hạn. Giải pháp: Source bucket và destination bucket tách biệt, hoặc dùng prefix khác nhau (uploads/ → processed/). S3 có cơ chế detect loop cơ bản nhưng không hoàn toàn ngăn được.

**Q4: So sánh S3 Event Notifications và S3 + EventBridge về độ phức tạp và tính năng?**

> S3 native notifications: Đơn giản, độ trễ thấp, không phí, hỗ trợ 3 destinations (Lambda, SQS, SNS). EventBridge: Phức tạp hơn, có thêm phí ($1/triệu event), nhưng hỗ trợ 200+ integration, content-based routing phức tạp, archive/replay events, và kết nối với SaaS. Chọn native cho use case đơn giản, EventBridge khi cần routing phức tạp hoặc kết nối nhiều service.

**Q5: Làm sao đảm bảo Lambda không bị invoke quá nhiều khi có S3 upload storm?**

> Dùng SQS làm buffer: S3 → SQS → Lambda. SQS tự động batch messages, Lambda đọc từ SQS theo batch (tối đa 10,000 message/batch với Long Polling). Lambda concurrency limit áp dụng tự nhiên. Thêm SQS queue depth alarm để scale Lambda concurrency khi cần. Không invoke Lambda trực tiếp từ S3 khi có khả năng traffic spike.

---

## 📋 Checklist Triển Khai

- [ ] Thêm resource policy vào SQS/SNS/Lambda để S3 có quyền gửi notification
- [ ] Kiểm tra ARN destination chính xác trong notification configuration
- [ ] Thiết kế consumer idempotent (chống duplicate)
- [ ] Cấu hình Dead Letter Queue cho Lambda và SQS
- [ ] Test với event thực tế (upload file nhỏ và kiểm tra notification đến đích)
- [ ] Monitor Lambda throttling và SQS queue depth
- [ ] Cân nhắc dùng prefix/suffix filter để tránh notification không cần thiết
- [ ] Tránh circular triggers (source và destination prefix không trùng nhau)

---

**Tiếp Theo:** [6-s3-batch-operations.md](./6-s3-batch-operations.md) — Thao tác hàng loạt trên hàng tỷ object
