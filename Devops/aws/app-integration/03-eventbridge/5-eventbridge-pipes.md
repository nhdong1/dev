# EventBridge Pipes — Đường Ống Sự Kiện

> **EventBridge Pipes** (Đường Ống Sự Kiện) là tính năng ra mắt năm 2022, cho phép kết nối trực tiếp một **source** (nguồn) đến một **target** (đích) với các bước **filter** (lọc) và **enrichment** (làm giàu dữ liệu) tùy chọn ở giữa — tất cả trong một cấu hình, không cần code.

---

## 📚 Mục Lục

1. [Pipes Là Gì và Tại Sao Cần?](#1-pipes-là-gì-và-tại-sao-cần)
2. [Kiến Trúc Pipes](#2-kiến-trúc-pipes)
3. [Nguồn Sự Kiện (Sources)](#3-nguồn-sự-kiện-sources)
4. [Filtering — Lọc Sự Kiện](#4-filtering--lọc-sự-kiện)
5. [Enrichment — Làm Giàu Dữ Liệu](#5-enrichment--làm-giàu-dữ-liệu)
6. [Target — Đích Sự Kiện](#6-target--đích-sự-kiện)
7. [Ví Dụ Thực Tế](#7-ví-dụ-thực-tế)
8. [So Sánh Pipes vs Rule](#8-so-sánh-pipes-vs-rule)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Pipes Là Gì và Tại Sao Cần?

### Vấn Đề Trước Khi Có Pipes

Trước khi có EventBridge Pipes, để kết nối DynamoDB Streams đến một API endpoint, bạn cần:

```
DynamoDB Stream
    ↓
Lambda (polling, đọc records)
    ↓
Lambda (lọc records không cần thiết)
    ↓
Lambda (transform data format)
    ↓
Lambda (gọi API)
    ↓
API Gateway / HTTP endpoint
```

**Vấn đề:**
- Phải viết và maintain nhiều Lambda functions
- Chi phí Lambda invocation tích lũy
- Phức tạp hóa infrastructure (hạ tầng)
- Debug khó khi pipeline dài

### Pipes Giải Quyết Bằng Cách

```
DynamoDB Stream ──▶ [Filter] ──▶ [Enrichment] ──▶ API Gateway
                     Pipe cấu hình trong console, không cần code
```

Pipes là **low-code integration** (tích hợp ít code) giữa event source và target.

---

## 2. Kiến Trúc Pipes

### Cấu Trúc Chung

```
┌──────────────────────────────────────────────────────────┐
│                    EventBridge Pipe                      │
│                                                          │
│  ┌────────┐   ┌──────────┐   ┌────────────┐   ┌──────┐  │
│  │        │   │  Source  │   │Enrichment  │   │      │  │
│  │ Source │──▶│ Filter   │──▶│(Lambda/API │──▶│Target│  │
│  │        │   │(Lọc)     │   │Gateway/SF) │   │      │  │
│  └────────┘   └──────────┘   └────────────┘   └──────┘  │
│                                                          │
│  [Polling config] [Batch settings] [Concurrency]         │
└──────────────────────────────────────────────────────────┘
```

### Các Bước Trong Pipe

| Bước | Bắt Buộc | Mô Tả |
|---|---|---|
| **Source** | ✅ | Nguồn sự kiện (SQS, DynamoDB, Kinesis...) |
| **Source Filter** | ❌ | Lọc sự kiện ngay tại nguồn |
| **Enrichment** | ❌ | Làm giàu dữ liệu (Lambda, API GW, Step Fn) |
| **Target** | ✅ | Đích nhận sự kiện cuối cùng |
| **Target Input Transform** | ❌ | Biến đổi format trước khi gửi target |

---

## 3. Nguồn Sự Kiện (Sources)

Pipes hỗ trợ các nguồn có **polling mechanism** (cơ chế lấy dữ liệu chủ động):

| Source | Giao Thức | Polling Mode |
|---|---|---|
| **Amazon SQS** | Queue | Long polling |
| **Amazon DynamoDB Streams** | Stream | Shard iterator |
| **Amazon Kinesis Data Streams** | Stream | Shard iterator |
| **Amazon MQ (ActiveMQ/RabbitMQ)** | Queue/Topic | Consumer |
| **Amazon MSK (Managed Kafka)** | Topic | Consumer group |
| **Self-managed Kafka** | Topic | Consumer group |

### Cấu Hình Source SQS

```json
{
  "Source": "arn:aws:sqs:ap-southeast-1:123456789:orders-queue",
  "SourceParameters": {
    "SqsQueueParameters": {
      "BatchSize": 10,                    // Xử lý tối đa 10 messages/lần
      "MaximumBatchingWindowInSeconds": 5  // Đợi tối đa 5 giây để đủ batch
    }
  }
}
```

### Cấu Hình Source DynamoDB Streams

```json
{
  "Source": "arn:aws:dynamodb:ap-southeast-1:123456789:table/Orders/stream/2026-05-18T00:00:00.000",
  "SourceParameters": {
    "DynamoDBStreamParameters": {
      "StartingPosition": "LATEST",       // LATEST hoặc TRIM_HORIZON
      "BatchSize": 100,
      "MaximumRetryAttempts": 3,
      "OnPartialBatchItemFailure": "AUTOMATIC_BISECT",  // Tách batch khi lỗi
      "DeadLetterConfig": {
        "Arn": "arn:aws:sqs:...:dlq"
      }
    }
  }
}
```

### Cấu Hình Source Kinesis

```json
{
  "Source": "arn:aws:kinesis:ap-southeast-1:123456789:stream/order-events",
  "SourceParameters": {
    "KinesisStreamParameters": {
      "StartingPosition": "AT_TIMESTAMP",
      "StartingPositionTimestamp": "2026-05-18T00:00:00Z",
      "BatchSize": 100,
      "MaximumBatchingWindowInSeconds": 30,
      "ParallelizationFactor": 3  // Xử lý song song 3 shards
    }
  }
}
```

---

## 4. Filtering — Lọc Sự Kiện

**Source Filter** cho phép lọc sự kiện **trước khi gọi enrichment và target** — giảm chi phí vì không xử lý sự kiện không cần thiết.

### Filter Cú Pháp

```json
{
  "FilterCriteria": {
    "Filters": [
      {
        "Pattern": "{\"body\": {\"status\": [\"FAILED\", \"ERROR\"]}}"
      }
    ]
  }
}
```

### Filter Cho SQS Message Body

```json
// Chỉ xử lý đơn hàng có giá trị > 1 triệu VND
{
  "FilterCriteria": {
    "Filters": [
      {
        "Pattern": "{\"body\": {\"amount\": [{\"numeric\": [\">\", 1000000]}]}}"
      }
    ]
  }
}
```

### Filter Cho DynamoDB Streams

```json
// Chỉ xử lý khi có record mới được INSERT (không phải MODIFY/REMOVE)
{
  "FilterCriteria": {
    "Filters": [
      {
        "Pattern": "{\"eventName\": [\"INSERT\"]}"
      }
    ]
  }
}
```

### Filter Cho Kinesis

```json
// Chỉ xử lý events từ partition key "orders"
{
  "FilterCriteria": {
    "Filters": [
      {
        "Pattern": "{\"partitionKey\": [{\"prefix\": \"orders-\"}]}"
      }
    ]
  }
}
```

---

## 5. Enrichment — Làm Giàu Dữ Liệu

**Enrichment** cho phép gọi một service để bổ sung thêm thông tin vào sự kiện trước khi gửi target.

### Enrichment Sources Được Hỗ Trợ

| Enrichment | Mô Tả |
|---|---|
| **AWS Lambda** | Gọi Lambda để transform/enrich |
| **Amazon API Gateway REST API** | Gọi API endpoint |
| **Amazon API Gateway HTTP API** | Gọi HTTP endpoint |
| **AWS Step Functions Express Workflow** | Chạy workflow phức tạp |

### Enrichment Với Lambda

```python
# Lambda function dùng làm enrichment
def lambda_handler(event, context):
    """
    Nhận batch events từ Pipes, bổ sung thông tin khách hàng.
    Trả về enriched events.
    """
    enriched_events = []

    for record in event:
        # record là SQS message body
        order_data = json.loads(record['body'])
        customer_id = order_data['customerId']

        # Gọi Customer Service để lấy thông tin
        customer = get_customer_details(customer_id)

        # Bổ sung thông tin khách hàng vào event
        enriched_event = {
            **order_data,
            'customer': {
                'name': customer['name'],
                'email': customer['email'],
                'tier': customer['tier'],
                'totalOrders': customer['totalOrders']
            }
        }

        enriched_events.append(enriched_event)

    return enriched_events  # Pipes sẽ gửi array này đến target


def get_customer_details(customer_id: str) -> dict:
    """Lấy thông tin khách hàng từ DynamoDB."""
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('Customers')
    response = table.get_item(Key={'customerId': customer_id})
    return response.get('Item', {})
```

### Enrichment Với API Gateway

```json
{
  "Enrichment": "arn:aws:execute-api:ap-southeast-1:123456789:api-id/prod/GET/customer",
  "EnrichmentParameters": {
    "HttpParameters": {
      "PathParameterValues": ["$.customerId"],
      "HeaderParameters": {
        "Authorization": "Bearer ${TOKEN}"
      }
    },
    "InputTemplate": "{\"customerId\": \"<aws.pipes.event.customerId>\"}"
  }
}
```

---

## 6. Target — Đích Sự Kiện

Pipes hỗ trợ nhiều targets hơn EventBridge Rules:

| Target | Ghi Chú |
|---|---|
| **AWS Lambda** | Gọi trực tiếp, batch hoặc streaming |
| **Amazon SQS** | Gửi vào queue |
| **Amazon SNS** | Publish đến topic |
| **Amazon Kinesis Data Streams** | Ghi vào stream |
| **AWS Step Functions** | Khởi động workflow |
| **Amazon EventBridge Bus** | Forward đến event bus |
| **Amazon API Gateway REST** | Gọi REST endpoint |
| **Amazon API Gateway HTTP** | Gọi HTTP endpoint |
| **Amazon CloudWatch Logs** | Ghi log trực tiếp |
| **Amazon Redshift** | Ghi vào data warehouse |
| **Amazon SageMaker Pipeline** | Khởi động ML pipeline |

### Target Configuration với Input Transform

```json
{
  "Target": "arn:aws:sqs:ap-southeast-1:123456789:enriched-orders-queue",
  "TargetParameters": {
    "InputTemplate": "{\"orderId\": \"<aws.pipes.event.orderId>\", \"customerName\": \"<aws.pipes.event.customer.name>\", \"tier\": \"<aws.pipes.event.customer.tier>\", \"amount\": <aws.pipes.event.amount>}",
    "SqsQueueParameters": {
      "MessageGroupId": "<aws.pipes.event.customerId>",
      "MessageDeduplicationId": "<aws.pipes.event.orderId>"
    }
  }
}
```

---

## 7. Ví Dụ Thực Tế

### Ví Dụ 1: DynamoDB → EventBridge Bus

```
Kịch bản: Khi có thay đổi trong bảng Orders (DynamoDB), tự động publish sự kiện lên EventBridge.

Trước Pipes: Cần Lambda polling DynamoDB Streams + Lambda publish event
Với Pipes:   DynamoDB Streams → Filter (INSERT only) → EventBridge Bus
```

```python
import boto3

client = boto3.client('pipes')

response = client.create_pipe(
    Name='dynamo-to-eventbridge',
    Description='DynamoDB Orders stream → EventBridge orders-bus',
    RoleArn='arn:aws:iam::123456789:role/PipesRole',

    # Source: DynamoDB Streams
    Source='arn:aws:dynamodb:ap-southeast-1:123456789:table/Orders/stream/...',
    SourceParameters={
        'DynamoDBStreamParameters': {
            'StartingPosition': 'LATEST',
            'BatchSize': 10
        },
        'FilterCriteria': {
            'Filters': [
                {'Pattern': '{"eventName": ["INSERT"]}'}
            ]
        }
    },

    # Target: EventBridge Bus
    Target='arn:aws:events:ap-southeast-1:123456789:event-bus/orders-bus',
    TargetParameters={
        'InputTemplate': '{"source": "com.mycompany.orders", "detail-type": "Order Created", "detail": <aws.pipes.event>}'
    }
)

print(f"Pipe created: {response['Arn']}")
```

### Ví Dụ 2: SQS → Lambda Với Enrichment

```
Kịch bản: Xử lý đơn hàng từ SQS, bổ sung thông tin khách hàng, gửi đến Step Functions.

Pipeline:
SQS (orders-queue)
  ↓ Filter: chỉ đơn hàng PENDING
  ↓ Enrichment: Lambda gọi Customer Service API
  ↓ Target: Step Functions (OrderProcessingWorkflow)
```

```yaml
# CloudFormation template cho Pipe
Resources:
  OrderProcessingPipe:
    Type: AWS::Pipes::Pipe
    Properties:
      Name: order-processing-pipe
      RoleArn: !GetAtt PipesRole.Arn
      Source: !GetAtt OrdersQueue.Arn
      SourceParameters:
        SqsQueueParameters:
          BatchSize: 1  # Xử lý từng đơn hàng
          MaximumBatchingWindowInSeconds: 0
        FilterCriteria:
          Filters:
            - Pattern: '{"body": {"status": ["PENDING"]}}'
      Enrichment: !GetAtt CustomerEnrichmentFunction.Arn
      EnrichmentParameters:
        InputTemplate: '{"customerId": "<aws.pipes.event.body.customerId>"}'
      Target: !Ref OrderWorkflowStateMachine
      TargetParameters:
        StepFunctionStateMachineParameters:
          InvocationType: FIRE_AND_FORGET  # Async — không chờ kết quả
```

### Ví Dụ 3: Kinesis → CloudWatch Logs (Real-time Logging)

```
Kịch bản: Log tất cả sự kiện payment thất bại vào CloudWatch Logs để phân tích.

Kinesis (payment-events)
  ↓ Filter: status = FAILED
  ↓ Target: CloudWatch Log Group
```

```json
{
  "Name": "payment-failures-logging",
  "Source": "arn:aws:kinesis:...:stream/payment-events",
  "SourceParameters": {
    "KinesisStreamParameters": {
      "StartingPosition": "LATEST",
      "BatchSize": 50
    },
    "FilterCriteria": {
      "Filters": [
        {"Pattern": "{\"data\": {\"status\": [\"FAILED\"]}}"}
      ]
    }
  },
  "Target": "arn:aws:logs:ap-southeast-1:123456789:log-group:/payment/failures",
  "TargetParameters": {
    "InputTemplate": "{\"timestamp\": \"<aws.pipes.event.approximateArrivalTimestamp>\", \"paymentId\": \"<aws.pipes.event.data.paymentId>\", \"error\": \"<aws.pipes.event.data.errorCode>\"}"
  }
}
```

---

## 8. So Sánh Pipes vs Rule

| Tiêu Chí | EventBridge Rule | EventBridge Pipes |
|---|---|---|
| **Source** | Chỉ EventBridge Bus | SQS, DynamoDB, Kinesis, MQ, Kafka |
| **Polling** | Không — push model | Có — pull model từ queue/stream |
| **Enrichment** | Không | Có — Lambda, API GW, Step Fn |
| **Filter** | Có — event pattern | Có — source filter |
| **Batch** | Không | Có — configurable batch size |
| **Targets** | Tối đa 5 | 1 target |
| **Use Case** | Route event từ bus | Point-to-point với enrichment |
| **Pricing** | $1/triệu events | $0.40/triệu events (rẻ hơn) |

### Khi Nào Dùng Pipes?

```
✅ Dùng Pipes khi:
- Source là SQS, DynamoDB Streams, Kinesis, Kafka, MQ
- Cần enrichment (bổ sung dữ liệu) trước khi gửi target
- Cần filter ở source level để giảm chi phí
- Point-to-point integration đơn giản

✅ Dùng Rule khi:
- Source là EventBridge Event Bus
- Cần fan-out (1 event → nhiều targets)
- Cần scheduled execution (cron/rate)
- Cần routing phức tạp với nhiều targets
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: EventBridge Pipes khác gì so với Lambda làm trigger từ SQS?**

> Lambda trigger từ SQS: Cần viết Lambda code để handle polling, batching, error handling; mỗi message đi qua Lambda invocation. EventBridge Pipes: Cấu hình trong console, không cần code cho logic kết nối; chỉ cần viết code trong bước Enrichment nếu cần bổ sung dữ liệu. Pipes cũng hỗ trợ source filter để lọc trước khi gọi Enrichment — Lambda trigger không có bước filter này, phải filter trong code. Trade-off: Pipes ít linh hoạt hơn nhưng ít code hơn và rẻ hơn (không tốn Lambda invocation cho messages bị filter).

**Q: Enrichment trong Pipes khác gì với Transform trong Rule?**

> Transform (Input Transformer) trong Rule chỉ có thể rename fields và format strings — không gọi external service. Enrichment trong Pipes có thể gọi Lambda, API Gateway, Step Functions — nghĩa là có thể query database, gọi external API, chạy business logic để bổ sung thêm dữ liệu vào event. Ví dụ: nhận event có customerId, gọi Customer Service API để lấy thêm customerName, email, tier, rồi mới gửi đến target.

**Q: Pipes có thể thay thế hoàn toàn Lambda làm event processor không?**

> Không hoàn toàn. Pipes phù hợp cho point-to-point integration với enrichment đơn giản. Vẫn cần Lambda khi: (1) Business logic phức tạp ở consumer (không chỉ transform), (2) Cần fan-out đến nhiều targets, (3) Cần conditional routing phức tạp, (4) Cần state management. Pipes là công cụ bổ sung, không thay thế Lambda hoàn toàn.

---

## 📝 Checklist Module EventBridge Pipes

- [ ] Giải thích Pipes là gì và khác gì EventBridge Rule
- [ ] Kể tên các sources mà Pipes hỗ trợ
- [ ] Mô tả vai trò của Enrichment trong pipeline
- [ ] Biết khi nào dùng Pipes vs khi nào dùng Rule
- [ ] Viết được Pipe cơ bản với CloudFormation hoặc AWS CLI

---

**Liên Kết:** [4-archive-replay.md](4-archive-replay.md) | [README.md](README.md) | [INDEX.md](../INDEX.md)
