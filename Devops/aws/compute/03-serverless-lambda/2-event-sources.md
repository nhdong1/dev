# 🔗 Event Sources — Nguồn Sự Kiện & Tích Hợp Lambda

> Lambda có thể được kích hoạt bởi hàng chục AWS services. Mỗi event source có cấu trúc event riêng, cơ chế retry khác nhau, và best practices cụ thể. Hiểu rõ từng loại là chìa khóa để thiết kế serverless architecture đúng.

## 📚 Mục Lục

1. [Phân Loại Event Sources](#phân-loại-event-sources)
2. [API Gateway — HTTP API & REST API](#api-gateway--http-api--rest-api)
3. [SQS — Simple Queue Service](#sqs--simple-queue-service)
4. [SNS — Simple Notification Service](#sns--simple-notification-service)
5. [S3 — Object Storage Events](#s3--object-storage-events)
6. [DynamoDB Streams — Luồng Thay Đổi DB](#dynamodb-streams--luồng-thay-đổi-db)
7. [EventBridge — Event Bus](#eventbridge--event-bus)
8. [Kinesis Data Streams](#kinesis-data-streams)
9. [ALB — Application Load Balancer](#alb--application-load-balancer)
10. [So Sánh & Chọn Event Source](#so-sánh--chọn-event-source)

---

## 🗂️ Phân Loại Event Sources

Lambda nhận events theo 3 cơ chế:

```
┌────────────────────────────────────────────────────────────────┐
│                    Invocation Models                           │
│                  (Mô Hình Gọi Hàm)                            │
├────────────────────────────────────────────────────────────────┤
│  SYNCHRONOUS (Đồng Bộ)      │ Caller chờ response             │
│  ─────────────────────────  │ ─────────────────────────────── │
│  - API Gateway               │ Error handling: caller tự xử    │
│  - ALB                       │ Retry: caller tự retry          │
│  - Lambda invoke()           │ Timeout: caller timeout         │
├──────────────────────────────┼─────────────────────────────────┤
│  ASYNCHRONOUS (Bất Đồng Bộ) │ Lambda nhận event, xử lý sau   │
│  ─────────────────────────  │ ─────────────────────────────── │
│  - S3 Events                 │ Error handling: DLQ + retry     │
│  - SNS                       │ Retry: 0-2 lần tự động          │
│  - EventBridge               │ Timeout: 6 giờ event age        │
├──────────────────────────────┼─────────────────────────────────┤
│  POLL-BASED (Thăm Dò)       │ Lambda polling từ source        │
│  ─────────────────────────  │ ─────────────────────────────── │
│  - SQS                       │ Error handling: message ở queue │
│  - DynamoDB Streams          │ Retry: đến visibility timeout   │
│  - Kinesis Streams           │ Batch: xử lý nhiều records/lần  │
└──────────────────────────────┴─────────────────────────────────┘
```

---

## 🌐 API Gateway — HTTP API & REST API

### REST API vs HTTP API

| Tính Năng                    | REST API              | HTTP API              |
| ---------------------------- | --------------------- | --------------------- |
| Giá (per million requests)   | ~$3.50                | ~$1.00 (rẻ hơn 71%)  |
| Latency                      | ~6ms overhead          | ~1ms overhead         |
| WebSocket                    | ✅                    | ❌                    |
| Request/Response transform   | ✅ Mapping Templates  | ❌ Không có           |
| Usage Plans / API Keys       | ✅                    | ❌                    |
| Private endpoints            | ✅                    | ✅                    |
| JWT Authorizer               | ✅                    | ✅ (native support)   |
| **Khi nào dùng**             | Legacy, full features | Mới, đơn giản, nhanh  |

### Event Structure từ REST API (Cấu Trúc Sự Kiện)

```json
{
  "version": "1.0",
  "resource": "/users/{userId}",
  "path": "/users/123",
  "httpMethod": "GET",
  "headers": {
    "Content-Type": "application/json",
    "Authorization": "Bearer eyJ..."
  },
  "queryStringParameters": {
    "include": "profile"
  },
  "pathParameters": {
    "userId": "123"
  },
  "requestContext": {
    "requestId": "req-abc-123",
    "identity": {
      "sourceIp": "192.168.1.1"
    }
  },
  "body": null,
  "isBase64Encoded": false
}
```

### Lambda Handler cho API Gateway

```python
import json
from decimal import Decimal

class DecimalEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, Decimal):
            return float(obj)
        return super().default(obj)

def handler(event, context):
    method = event['httpMethod']
    path = event['path']
    
    # Parse body
    body = {}
    if event.get('body'):
        body = json.loads(event['body'])
    
    # Route theo method và path
    if method == 'GET' and path.startswith('/users/'):
        user_id = event['pathParameters']['userId']
        return get_user_response(user_id)
    elif method == 'POST' and path == '/users':
        return create_user_response(body)
    else:
        return {
            'statusCode': 404,
            'body': json.dumps({'error': 'Route not found'})
        }

def get_user_response(user_id):
    # Trả về response đúng format cho API Gateway
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'  # CORS
        },
        'body': json.dumps({'userId': user_id, 'name': 'John'}, cls=DecimalEncoder)
    }
```

### Authorizers (Bộ Xác Thực)

#### JWT Authorizer (HTTP API)

```
Client → API GW → JWT Authorizer → Lambda
                      │
                   Verify JWT token
                   Extract claims
                   Attach to $context
```

#### Lambda Authorizer (REST API)

```python
def authorizer_handler(event, context):
    token = event['authorizationToken']
    method_arn = event['methodArn']
    
    try:
        # Verify token
        claims = verify_jwt(token)
        effect = 'Allow'
        principal = claims['sub']
    except Exception:
        effect = 'Deny'
        principal = 'unauthorized'
    
    return {
        'principalId': principal,
        'policyDocument': {
            'Version': '2012-10-17',
            'Statement': [{
                'Action': 'execute-api:Invoke',
                'Effect': effect,
                'Resource': method_arn
            }]
        },
        'context': {  # Gửi thêm data cho downstream Lambda
            'userId': claims.get('sub'),
            'role': claims.get('role')
        }
    }
```

---

## 📨 SQS — Simple Queue Service (Dịch Vụ Hàng Đợi Đơn Giản)

### Cách Lambda Tích Hợp Với SQS

Lambda **polling** (thăm dò) SQS queue — không phải SQS push vào Lambda:

```
SQS Queue ←── Lambda Polling ──── Lambda Service
   │                                     │
   │ (Lambda nhận batch của messages)    │
   └─────────────────────────────────────┘
                    │
                    ▼ Invoke
              Lambda Function
                    │
         ┌──────────┴──────────┐
         │ Success             │ Failure
         ▼                     ▼
   Delete messages       Keep in queue
   from queue            (retry later)
```

### Event Structure từ SQS (Cấu Trúc Sự Kiện)

```json
{
  "Records": [
    {
      "messageId": "msg-001",
      "receiptHandle": "AQEBwJ...",
      "body": "{\"orderId\": \"ORD-123\", \"amount\": 99.99}",
      "attributes": {
        "ApproximateReceiveCount": "1",
        "SentTimestamp": "1704067200000",
        "SenderId": "123456789"
      },
      "messageAttributes": {
        "MessageType": {
          "stringValue": "OrderCreated",
          "dataType": "String"
        }
      },
      "eventSource": "aws:sqs",
      "eventSourceARN": "arn:aws:sqs:us-east-1:123456789:orders-queue",
      "awsRegion": "us-east-1"
    }
  ]
}
```

### SQS Handler với Partial Batch Failure (Xử Lý Lỗi Một Phần)

```python
import json
import logging

logger = logging.getLogger()

def handler(event, context):
    """
    Xử lý SQS messages với partial batch failure.
    Nếu một số messages thất bại, chỉ giữ lại chúng để retry.
    """
    failed_message_ids = []
    
    for record in event['Records']:
        message_id = record['messageId']
        
        try:
            body = json.loads(record['body'])
            process_message(body)
            logger.info(f"Successfully processed: {message_id}")
        except Exception as e:
            logger.error(f"Failed to process {message_id}: {str(e)}")
            failed_message_ids.append(message_id)
    
    # Trả về danh sách messages thất bại để Lambda retry
    # (Cần bật ReportBatchItemFailures trong Event Source Mapping)
    return {
        "batchItemFailures": [
            {"itemIdentifier": msg_id}
            for msg_id in failed_message_ids
        ]
    }

def process_message(body):
    order_id = body['orderId']
    amount = body['amount']
    # ... business logic
    logger.info(f"Processing order {order_id} for ${amount}")
```

### Cấu Hình Event Source Mapping SQS

```bash
aws lambda create-event-source-mapping \
  --function-name process-orders \
  --event-source-arn arn:aws:sqs:us-east-1:123456789:orders-queue \
  --batch-size 10 \
  --maximum-batching-window-in-seconds 5 \
  --function-response-types ReportBatchItemFailures
```

### SQS Standard vs FIFO

| Tính Năng                | SQS Standard              | SQS FIFO                   |
| ------------------------ | ------------------------- | -------------------------- |
| Throughput               | Unlimited                 | 300 msg/s (3000 với batching) |
| Thứ tự messages          | Best-effort (không đảm bảo)| Đảm bảo FIFO              |
| Exactly-once delivery    | At-least-once             | Exactly-once               |
| Lambda concurrency       | Scale tự do               | Giới hạn theo message group|
| Khi dùng với Lambda      | Bulk processing, resilient | Order-critical workflows   |

---

## 📢 SNS — Simple Notification Service (Dịch Vụ Thông Báo)

### SNS Push (Đẩy) vs SQS Pull (Kéo)

```
SNS → Lambda (Push — SNS gọi Lambda trực tiếp)
SQS → Lambda (Pull — Lambda polling từ SQS)

Fan-out Pattern:
                  ┌─── Lambda A (Email notification)
SNS Topic ────────┼─── Lambda B (SMS notification)
                  ├─── SQS Queue → Lambda C (Process & store)
                  └─── HTTP Endpoint (Webhook)
```

### Event Structure từ SNS

```json
{
  "Records": [
    {
      "EventVersion": "1.0",
      "EventSubscriptionArn": "arn:aws:sns:us-east-1:123456789:alerts:sub-id",
      "EventSource": "aws:sns",
      "Sns": {
        "SignatureVersion": "1",
        "Timestamp": "2024-01-01T00:00:00.000Z",
        "MessageId": "msg-id-123",
        "Message": "{\"alarm\": \"CPU_HIGH\", \"value\": 95}",
        "MessageAttributes": {
          "severity": {
            "Type": "String",
            "Value": "critical"
          }
        },
        "Type": "Notification",
        "Subject": "CloudWatch Alert",
        "TopicArn": "arn:aws:sns:us-east-1:123456789:alerts"
      }
    }
  ]
}
```

### SNS Handler

```python
import json

def handler(event, context):
    for record in event['Records']:
        sns_message = record['Sns']
        
        # Parse message body
        message = json.loads(sns_message['Message'])
        subject = sns_message.get('Subject', 'No Subject')
        message_attrs = sns_message.get('MessageAttributes', {})
        
        severity = message_attrs.get('severity', {}).get('Value', 'info')
        
        if severity == 'critical':
            send_pagerduty_alert(message)
        else:
            send_slack_notification(message)
        
        print(f"Processed SNS message: {sns_message['MessageId']}")
```

---

## 🪣 S3 — Object Storage Events (Sự Kiện Lưu Trữ Object)

### Các Loại S3 Events

| Event Type                    | Khi Xảy Ra                        |
| ----------------------------- | ---------------------------------- |
| `s3:ObjectCreated:Put`        | Upload file lên S3                 |
| `s3:ObjectCreated:Post`       | Upload qua presigned POST URL      |
| `s3:ObjectCreated:Copy`       | Copy object từ bucket khác         |
| `s3:ObjectCreated:*`          | Bất kỳ loại tạo object nào        |
| `s3:ObjectRemoved:Delete`     | Xóa object                        |
| `s3:ObjectRestore:Completed`  | Restore từ Glacier xong           |

### Event Structure từ S3

```json
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventTime": "2024-01-01T00:00:00.000Z",
      "eventName": "ObjectCreated:Put",
      "s3": {
        "s3SchemaVersion": "1.0",
        "bucket": {
          "name": "my-uploads-bucket",
          "arn": "arn:aws:s3:::my-uploads-bucket"
        },
        "object": {
          "key": "uploads/user-123/profile-photo.jpg",
          "size": 1048576,
          "eTag": "abc123def456",
          "versionId": "v001"
        }
      }
    }
  ]
}
```

### S3 Handler — Image Resize (Thay Đổi Kích Thước Ảnh)

```python
import boto3
from PIL import Image
import io
import os
import urllib.parse

s3 = boto3.client('s3')

THUMBNAIL_SIZES = [(150, 150), (300, 300), (600, 600)]
OUTPUT_BUCKET = os.environ['OUTPUT_BUCKET']

def handler(event, context):
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        # URL decode key (S3 URL-encodes special characters)
        key = urllib.parse.unquote_plus(record['s3']['object']['key'])
        
        # Chỉ xử lý ảnh
        if not is_image(key):
            print(f"Skipping non-image: {key}")
            continue
        
        try:
            process_image(bucket, key)
        except Exception as e:
            print(f"Error processing {key}: {str(e)}")
            raise

def process_image(bucket, key):
    # Download ảnh gốc
    response = s3.get_object(Bucket=bucket, Key=key)
    image_data = response['Body'].read()
    
    image = Image.open(io.BytesIO(image_data))
    
    for width, height in THUMBNAIL_SIZES:
        thumbnail = image.copy()
        thumbnail.thumbnail((width, height))
        
        # Upload thumbnail
        buffer = io.BytesIO()
        thumbnail.save(buffer, format=image.format)
        buffer.seek(0)
        
        thumbnail_key = f"thumbnails/{width}x{height}/{key}"
        s3.put_object(
            Bucket=OUTPUT_BUCKET,
            Key=thumbnail_key,
            Body=buffer,
            ContentType=response['ContentType']
        )
        
        print(f"Created thumbnail: {thumbnail_key}")

def is_image(key):
    extensions = ['.jpg', '.jpeg', '.png', '.gif', '.webp']
    return any(key.lower().endswith(ext) for ext in extensions)
```

### Cảnh Báo: S3 → Lambda → S3 Loop (Vòng Lặp Vô Hạn)

```
NGUY HIỂM: Tránh loop này!
S3 Bucket → Lambda → S3 Bucket (cùng bucket + trigger trùng prefix)
                           │
                    Trigger lại Lambda
                           │
                    Lambda chạy lại...
                    (Tốn tiền vô hạn!)

GIẢI PHÁP:
1. Dùng prefix/suffix filter khác nhau:
   - Trigger: prefix="uploads/", suffix=".jpg"
   - Output:  prefix="thumbnails/"

2. Dùng bucket riêng cho input và output

3. Kiểm tra eventName để bỏ qua events không mong muốn
```

---

## 🌊 DynamoDB Streams — Luồng Thay Đổi DB

### Cách Hoạt Động

DynamoDB Streams ghi lại mọi thay đổi (insert/update/delete) vào stream:

```
DynamoDB Table
      │
      │ (Every change → Stream shard)
      ▼
DynamoDB Stream ←── Lambda Polling
      │                    │
      │  (Batch records)   │
      └────────────────────┘
                    │
                    ▼
              Lambda Function
                    │
              Process changes
              (CDC — Change Data Capture)
```

### Stream View Types (Loại Dữ Liệu Trong Stream)

| View Type                | Nội Dung                                      |
| ------------------------ | --------------------------------------------- |
| `KEYS_ONLY`              | Chỉ primary key của item thay đổi             |
| `NEW_IMAGE`              | Toàn bộ item sau khi thay đổi                 |
| `OLD_IMAGE`              | Toàn bộ item trước khi thay đổi               |
| `NEW_AND_OLD_IMAGES`     | Cả trước và sau (đầy đủ nhất, tốn storage hơn)|

### Event Structure từ DynamoDB Streams

```json
{
  "Records": [
    {
      "eventID": "event-001",
      "eventVersion": "1.1",
      "eventSource": "aws:dynamodb",
      "awsRegion": "us-east-1",
      "eventName": "INSERT",
      "dynamodb": {
        "ApproximateCreationDateTime": 1704067200,
        "Keys": {
          "userId": {"S": "user-123"},
          "createdAt": {"N": "1704067200"}
        },
        "NewImage": {
          "userId": {"S": "user-123"},
          "name": {"S": "John Doe"},
          "email": {"S": "john@example.com"},
          "status": {"S": "active"}
        },
        "OldImage": null,
        "StreamViewType": "NEW_AND_OLD_IMAGES",
        "SequenceNumber": "100000000001234",
        "SizeBytes": 128
      }
    },
    {
      "eventName": "MODIFY",
      "dynamodb": {
        "Keys": {"userId": {"S": "user-456"}},
        "NewImage": {"userId": {"S": "user-456"}, "status": {"S": "inactive"}},
        "OldImage": {"userId": {"S": "user-456"}, "status": {"S": "active"}}
      }
    },
    {
      "eventName": "REMOVE",
      "dynamodb": {
        "Keys": {"userId": {"S": "user-789"}},
        "OldImage": {"userId": {"S": "user-789"}, "name": {"S": "Jane Doe"}}
      }
    }
  ]
}
```

### DynamoDB Streams Handler — CDC (Change Data Capture — Nắm Bắt Thay Đổi Dữ Liệu)

```python
import json
import boto3
from boto3.dynamodb.types import TypeDeserializer

deserializer = TypeDeserializer()
elasticsearch = boto3.client('es')  # Sync sang Elasticsearch

def handler(event, context):
    for record in event['Records']:
        event_name = record['eventName']  # INSERT, MODIFY, REMOVE
        
        if event_name == 'INSERT':
            new_item = deserialize(record['dynamodb']['NewImage'])
            index_document(new_item, 'create')
            
        elif event_name == 'MODIFY':
            old_item = deserialize(record['dynamodb']['OldImage'])
            new_item = deserialize(record['dynamodb']['NewImage'])
            
            # Chỉ xử lý khi status thay đổi
            if old_item.get('status') != new_item.get('status'):
                handle_status_change(old_item, new_item)
            
            index_document(new_item, 'update')
            
        elif event_name == 'REMOVE':
            deleted_item = deserialize(record['dynamodb']['OldImage'])
            index_document(deleted_item, 'delete')

def deserialize(dynamo_item):
    return {k: deserializer.deserialize(v) for k, v in dynamo_item.items()}

def index_document(item, action):
    print(f"Elasticsearch {action}: userId={item['userId']}")
    # ... index vào Elasticsearch/OpenSearch
```

---

## 🚌 EventBridge — Event Bus (Xe Buýt Sự Kiện)

### EventBridge vs SNS vs SQS

| Tính Năng            | EventBridge          | SNS                   | SQS                   |
| -------------------- | -------------------- | --------------------- | --------------------- |
| Routing              | Rules dựa trên content | Simple fan-out       | Queue-based           |
| Filtering            | JSON pattern matching | Message attributes    | Chỉ queue level       |
| Targets              | 20+ AWS services     | SQS, Lambda, HTTP     | Lambda (chủ yếu)      |
| Schema Registry      | ✅ Có                | ❌ Không              | ❌ Không              |
| Cross-account        | ✅ Native            | ✅ Native             | ✅ Native             |
| Archive & Replay     | ✅ Có                | ❌ Không              | ❌ Không              |
| Khi dùng             | Complex routing, audit| Simple notifications  | Queue & rate limiting |

### EventBridge Rule (Quy Tắc) với Lambda

```json
{
  "source": ["custom.myapp"],
  "detail-type": ["OrderCreated"],
  "detail": {
    "amount": [{"numeric": [">=", 100]}],
    "status": ["PENDING"]
  }
}
```

### EventBridge Handler

```python
import json

def handler(event, context):
    """
    EventBridge event có cấu trúc chuẩn:
    {
        "version": "0",
        "id": "uuid",
        "detail-type": "OrderCreated",
        "source": "custom.myapp",
        "account": "123456789",
        "time": "2024-01-01T00:00:00Z",
        "region": "us-east-1",
        "detail": { ... }  ← Business data ở đây
    }
    """
    detail_type = event['detail-type']
    source = event['source']
    detail = event['detail']
    
    print(f"Received event: {detail_type} from {source}")
    
    if detail_type == 'OrderCreated':
        process_new_order(detail)
    elif detail_type == 'OrderCancelled':
        process_cancellation(detail)
    
def process_new_order(detail):
    order_id = detail['orderId']
    amount = detail['amount']
    # ... xử lý đơn hàng mới
```

---

## 🌊 Kinesis Data Streams (Luồng Dữ Liệu Kinesis)

### Kinesis vs SQS

| Tính Năng             | Kinesis Data Streams   | SQS                       |
| --------------------- | ---------------------- | ------------------------- |
| Retention             | 1-365 ngày             | 1-14 ngày                 |
| Throughput            | Per-shard (1 MB/s)     | Gần unlimited             |
| Ordering              | Per-shard ordering     | Best-effort (Standard)    |
| Replay               | ✅ Có thể replay        | ❌ Không                  |
| Consumers             | Multiple consumers      | One consumer per message  |
| Khi dùng              | Real-time analytics    | Task queue, decoupling    |

### Kinesis Handler với Parallel Processing

```python
import base64
import json

def handler(event, context):
    failed_records = []
    
    for record in event['Records']:
        try:
            # Kinesis data được base64-encoded
            payload = base64.b64decode(record['kinesis']['data']).decode('utf-8')
            data = json.loads(payload)
            
            shard_id = record['eventID'].split(':')[0]
            sequence = record['kinesis']['sequenceNumber']
            
            process_kinesis_record(data, shard_id)
            
        except Exception as e:
            print(f"Error processing record: {str(e)}")
            failed_records.append({
                "itemIdentifier": record['kinesis']['sequenceNumber']
            })
    
    return {"batchItemFailures": failed_records}
```

---

## ⚖️ ALB — Application Load Balancer (Cân Bằng Tải Ứng Dụng)

### ALB vs API Gateway

| Tính Năng                    | ALB                      | API Gateway              |
| ---------------------------- | ------------------------ | ------------------------ |
| Giá                          | Rẻ hơn cho high traffic  | Rẻ hơn cho low traffic   |
| WebSocket                    | ✅                       | ✅ (REST API)            |
| gRPC                         | ✅                       | ❌                       |
| Content-based routing        | ✅ (host/path/header)    | ✅ (path/method)         |
| Fixed IP                     | ✅ (qua Network LB)      | ❌                       |
| Khi dùng                     | High volume, gRPC, fixed IP | API management, OAuth  |

### ALB Event Structure

```json
{
  "requestContext": {
    "elb": {
      "targetGroupArn": "arn:aws:elasticloadbalancing:..."
    }
  },
  "httpMethod": "GET",
  "path": "/api/users",
  "queryStringParameters": {"page": "1"},
  "headers": {
    "x-forwarded-for": "1.2.3.4",
    "host": "api.example.com"
  },
  "body": null,
  "isBase64Encoded": false
}
```

---

## 📊 So Sánh & Chọn Event Source

### Decision Tree (Cây Quyết Định)

```
Bạn cần gì?
│
├─ HTTP API (REST/GraphQL)?
│   ├─ Nhiều tính năng (OAuth, API keys, throttling)? → API Gateway REST API
│   ├─ Đơn giản, nhanh, rẻ?                         → API Gateway HTTP API
│   └─ Volume rất cao, gRPC, fixed IP?               → ALB
│
├─ Xử lý messages/jobs?
│   ├─ Thứ tự quan trọng, exactly-once?              → SQS FIFO
│   ├─ Bulk processing, resilience?                  → SQS Standard
│   └─ Real-time streaming, replay, multiple readers?→ Kinesis
│
├─ Notifications/broadcasts?
│   ├─ Fan-out đơn giản?                             → SNS
│   └─ Routing phức tạp theo content?                → EventBridge
│
├─ Phản ứng với thay đổi database?
│   ├─ DynamoDB?                                     → DynamoDB Streams
│   └─ RDS/Aurora?                                   → RDS Event Notifications
│
└─ Phản ứng với file upload?
    └─ S3 Events → Lambda
```

### Bảng So Sánh Tổng Hợp

| Event Source        | Model       | Retry | Batch | Ordering | Khi Dùng                    |
| ------------------- | ----------- | ----- | ----- | -------- | --------------------------- |
| API Gateway         | Sync        | No    | No    | N/A      | HTTP APIs, real-time        |
| ALB                 | Sync        | No    | No    | N/A      | HTTP, gRPC, high volume     |
| SQS Standard        | Poll-based  | Yes   | Yes   | No       | Queues, task processing     |
| SQS FIFO            | Poll-based  | Yes   | Yes   | Yes      | Ordered processing          |
| SNS                 | Async       | Yes   | No    | No       | Notifications, fan-out      |
| S3 Events           | Async       | Yes   | No    | No       | File processing             |
| DynamoDB Streams    | Poll-based  | Yes   | Yes   | Per-shard| CDC, cross-service sync     |
| Kinesis             | Poll-based  | Yes   | Yes   | Per-shard| Real-time analytics         |
| EventBridge         | Async       | Yes   | No    | No       | Event routing, integration  |

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa SQS trigger và SNS trigger cho Lambda?**
> SQS: Lambda polling (kéo) từ queue, xử lý message theo batch, message ở trong queue cho đến khi xử lý thành công. SNS: SNS push (đẩy) vào Lambda, không có batch, Lambda phải xử lý ngay. SQS phù hợp cho task queue có retry tốt hơn; SNS phù hợp cho fan-out notification.

**Q: Làm thế nào tránh vòng lặp vô hạn với S3 trigger?**
> Dùng prefix/suffix filter khác nhau giữa input và output. Ví dụ: trigger trên prefix "uploads/", xuất kết quả vào prefix "processed/". Hoặc dùng 2 bucket riêng biệt. Kiểm tra eventName trong handler để bỏ qua events không mong đợi.

**Q: Khi nào chọn API Gateway REST API vs HTTP API?**
> HTTP API: Mới, đơn giản, rẻ hơn 71%, latency thấp hơn — dùng cho hầu hết API mới. REST API: Cần usage plans/API keys, request/response transformation (mapping templates), WebSocket, hoặc tích hợp với non-Lambda backends.

**Q: DynamoDB Streams dùng để làm gì?**
> CDC (Change Data Capture) — phản ứng với thay đổi trong DynamoDB để: sync dữ liệu sang Elasticsearch/OpenSearch, gửi notification khi record thay đổi, invalidate cache, audit log, replication cross-region.

**Q: Partial batch failure trong SQS là gì?**
> Khi một Lambda function xử lý batch SQS messages, mặc định nếu handler throw exception thì toàn bộ batch sẽ retry. Với partial batch failure (ReportBatchItemFailures), handler trả về danh sách messageIds thất bại — chỉ những messages đó bị retry, không phải cả batch.

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Trạng Thái:** ✅ Hoàn thành
**Tiếp Theo:** [3-layers-extensions.md](./3-layers-extensions.md) — Lambda Layers & Extensions
