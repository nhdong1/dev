# DynamoDB Streams & Lambda Integration

> DynamoDB Streams (Luồng Dữ Liệu DynamoDB) ghi lại mọi thay đổi dữ liệu theo thứ tự thời gian, cho phép xây dựng kiến trúc event-driven (hướng sự kiện) và tích hợp với AWS Lambda để xử lý real-time.

## 📚 Mục Lục

1. [DynamoDB Streams Là Gì?](#dynamodb-streams-là-gì)
2. [Cơ Chế Hoạt Động](#cơ-chế-hoạt-động)
3. [Stream View Types — Loại Dữ Liệu Stream](#stream-view-types--loại-dữ-liệu-stream)
4. [Lambda Integration — Tích Hợp Lambda](#lambda-integration--tích-hợp-lambda)
5. [Use Cases — Trường Hợp Sử Dụng](#use-cases--trường-hợp-sử-dụng)
6. [Xử Lý Lỗi & Retry](#xử-lý-lỗi--retry)
7. [Kinesis Data Streams cho DynamoDB](#kinesis-data-streams-cho-dynamodb)
8. [So Sánh Streams vs Kinesis](#so-sánh-streams-vs-kinesis)
9. [Best Practices — Thực Hành Tốt Nhất](#best-practices--thực-hành-tốt-nhất)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## DynamoDB Streams Là Gì?

DynamoDB Streams là tính năng ghi lại **mọi thay đổi dữ liệu** (INSERT, UPDATE, DELETE) trong bảng DynamoDB theo **thứ tự thời gian**, tổ chức thành một **ordered log** (nhật ký có thứ tự).

```
┌───────────────────────────────────────────────────────────────┐
│                       DynamoDB Table                          │
│                                                               │
│  INSERT item-A → ┐                                           │
│  UPDATE item-B → ├──► DynamoDB Streams ──► Consumers         │
│  DELETE item-C → ┘     (Ordered Log)       (Lambda, etc.)    │
│                                                               │
│              ┌─────────────────────────────┐                 │
│              │  Stream Record (Bản Ghi):   │                 │
│              │  - Event type (INSERT/...)  │                 │
│              │  - Old image (ảnh cũ)       │                 │
│              │  - New image (ảnh mới)      │                 │
│              │  - Timestamp                │                 │
│              │  - Sequence number          │                 │
│              └─────────────────────────────┘                 │
└───────────────────────────────────────────────────────────────┘
```

### Đặc Điểm Chính

| Đặc Điểm                              | Giá Trị                                              |
| ------------------------------------- | ---------------------------------------------------- |
| **Retention** (Thời Gian Lưu Trữ)    | **24 giờ** — sau đó bị xóa tự động                  |
| **Ordering** (Thứ Tự)                | Đảm bảo thứ tự **trong cùng partition**              |
| **Throughput** (Thông Lượng)         | Tự động scale theo table throughput                  |
| **Exactly-once delivery**            | Mỗi change record xuất hiện đúng một lần trong stream|
| **At-least-once processing**         | Lambda có thể được invoke nhiều lần (idempotent!)    |

---

## Cơ Chế Hoạt Động

### Stream Shards (Phân Mảnh Stream)

```
DynamoDB Table Partitions → Stream Shards (1:1 mapping)

Partition 1 → Shard 1 ──┐
Partition 2 → Shard 2 ──┤──► Lambda ESM (Event Source Mapping)
Partition 3 → Shard 3 ──┘    → Invoke Lambda per batch
...

Mỗi Shard:
- Sequence of stream records
- Đảm bảo thứ tự trong shard
- Lambda xử lý từng shard theo thứ tự (một batch tại một thời điểm)
```

### Stream Record Structure (Cấu Trúc Bản Ghi Stream)

```json
{
  "eventID": "1",
  "eventVersion": "1.0",
  "dynamodb": {
    "Keys": {
      "UserId": {"S": "user-001"},
      "OrderId": {"S": "ord-001"}
    },
    "NewImage": {
      "UserId": {"S": "user-001"},
      "OrderId": {"S": "ord-001"},
      "Status":  {"S": "PAID"},
      "TotalAmount": {"N": "150.00"}
    },
    "OldImage": {
      "UserId": {"S": "user-001"},
      "OrderId": {"S": "ord-001"},
      "Status":  {"S": "PENDING"},
      "TotalAmount": {"N": "150.00"}
    },
    "StreamViewType": "NEW_AND_OLD_IMAGES",
    "SequenceNumber": "111",
    "SizeBytes": 26
  },
  "awsRegion": "us-east-1",
  "eventName": "MODIFY",      // INSERT | MODIFY | REMOVE
  "eventSourceARN": "arn:aws:dynamodb:...",
  "eventSource": "aws:dynamodb"
}
```

---

## Stream View Types — Loại Dữ Liệu Stream

Chọn loại dữ liệu ghi vào stream khi enable Streams:

| View Type               | Dữ Liệu Trong Record                 | Dùng Khi                                        |
| ----------------------- | ------------------------------------ | ----------------------------------------------- |
| `KEYS_ONLY`             | Chỉ primary key của item thay đổi   | Chỉ cần biết item nào thay đổi, không cần data  |
| `NEW_IMAGE`             | Toàn bộ item sau khi thay đổi       | Cần trạng thái mới (audit log, replication)     |
| `OLD_IMAGE`             | Toàn bộ item trước khi thay đổi     | Cần trạng thái cũ (undo, change tracking)       |
| `NEW_AND_OLD_IMAGES`    | Cả trước và sau khi thay đổi        | So sánh thay đổi, phức tạp nhất, tốn storage    |

```python
# Enable Streams khi tạo hoặc cập nhật bảng
dynamodb.update_table(
    TableName='Orders',
    StreamSpecification={
        'StreamEnabled': True,
        'StreamViewType': 'NEW_AND_OLD_IMAGES'
    }
)
```

---

## Lambda Integration — Tích Hợp Lambda

### Event Source Mapping (Ánh Xạ Nguồn Sự Kiện)

Lambda không "poll" DynamoDB Streams — thay vào đó, AWS Lambda service tự động poll và invoke Lambda khi có records:

```
DynamoDB Streams ──► Lambda ESM (Event Source Mapping) ──► Lambda Function
                     │
                     ├── BatchSize: số records mỗi lần invoke (1-10,000)
                     ├── BisectBatchOnFunctionError: chia đôi batch khi lỗi
                     ├── StartingPosition: TRIM_HORIZON | LATEST
                     └── DestinationConfig: gửi failed records sang SQS/SNS
```

### Lambda Function Handler

```python
import json
from decimal import Decimal

def lambda_handler(event, context):
    for record in event['Records']:
        # Lấy event type
        event_name = record['eventName']  # INSERT | MODIFY | REMOVE

        # Parse DynamoDB types về Python types
        if event_name == 'INSERT':
            new_item = deserialize(record['dynamodb']['NewImage'])
            handle_insert(new_item)

        elif event_name == 'MODIFY':
            old_item = deserialize(record['dynamodb']['OldImage'])
            new_item = deserialize(record['dynamodb']['NewImage'])
            handle_modify(old_item, new_item)

        elif event_name == 'REMOVE':
            old_item = deserialize(record['dynamodb']['OldImage'])
            handle_delete(old_item)

    return {'statusCode': 200}


def deserialize(dynamodb_json):
    """Chuyển DynamoDB JSON format sang Python dict bình thường."""
    from boto3.dynamodb.types import TypeDeserializer
    deserializer = TypeDeserializer()
    return {k: deserializer.deserialize(v) for k, v in dynamodb_json.items()}


def handle_insert(item):
    """Xử lý order mới được tạo."""
    print(f"New order: {item['OrderId']} by user {item['UserId']}")
    # Gửi email xác nhận, cập nhật analytics, v.v.


def handle_modify(old_item, new_item):
    """Xử lý order được cập nhật."""
    if old_item['Status'] != new_item['Status']:
        print(f"Order {new_item['OrderId']} status: {old_item['Status']} → {new_item['Status']}")
        # Gửi notification (thông báo), cập nhật cache, v.v.
```

### Cấu Hình ESM (Event Source Mapping)

```python
import boto3
lambda_client = boto3.client('lambda')

# Lấy stream ARN
dynamodb = boto3.client('dynamodb')
table = dynamodb.describe_table(TableName='Orders')
stream_arn = table['Table']['LatestStreamArn']

# Tạo Event Source Mapping
lambda_client.create_event_source_mapping(
    EventSourceArn=stream_arn,
    FunctionName='process-order-changes',
    StartingPosition='LATEST',        # Bắt đầu từ records mới nhất
    BatchSize=100,                     # Xử lý 100 records/lần
    BisectBatchOnFunctionError=True,   # Chia đôi batch khi có lỗi
    DestinationConfig={
        'OnFailure': {
            'Destination': 'arn:aws:sqs:...:dead-letter-queue'
        }
    }
)
```

---

## Use Cases — Trường Hợp Sử Dụng

### 1. Cross-Region Replication (Sao Chép Xuyên Vùng)

```
Region A (us-east-1)          Region B (eu-west-1)
┌──────────────┐              ┌──────────────┐
│ DynamoDB     │  Streams     │ DynamoDB     │
│ Table A      │──────────►   │ Table B      │
└──────────────┘  Lambda      └──────────────┘
                  Replicator

(Lưu ý: DynamoDB Global Tables làm điều này tốt hơn,
 nhưng custom replication cho phép filter/transform data)
```

### 2. Cache Invalidation (Xóa Bộ Nhớ Đệm Hết Hạn)

```python
def lambda_handler(event, context):
    for record in event['Records']:
        if record['eventName'] in ('MODIFY', 'REMOVE'):
            keys = record['dynamodb']['Keys']
            product_id = keys['ProductId']['S']

            # Xóa cache khi product được cập nhật
            redis_client.delete(f"product:{product_id}")
            print(f"Cache invalidated for product: {product_id}")
```

### 3. Aggregation & Analytics (Tổng Hợp & Phân Tích)

```python
def lambda_handler(event, context):
    for record in event['Records']:
        if record['eventName'] == 'INSERT':
            new_item = deserialize(record['dynamodb']['NewImage'])

            # Gửi metrics sang CloudWatch
            cloudwatch.put_metric_data(
                Namespace='ECommerce',
                MetricData=[{
                    'MetricName': 'OrderCreated',
                    'Value': float(new_item['TotalAmount']),
                    'Unit': 'None'
                }]
            )

        elif record['eventName'] == 'MODIFY':
            old_item = deserialize(record['dynamodb']['OldImage'])
            new_item = deserialize(record['dynamodb']['NewImage'])
            # Track status transitions
            if new_item['Status'] == 'PAID' and old_item['Status'] == 'PENDING':
                send_to_analytics_pipeline(new_item)
```

### 4. Full-Text Search Sync (Đồng Bộ Tìm Kiếm Toàn Văn Bản)

```
DynamoDB (source of truth)
    │
    ├──► DynamoDB Streams
    │           │
    │    Lambda (sync)
    │           │
    └──►  OpenSearch / Elasticsearch
         (full-text search — tìm kiếm toàn văn bản)

Khi item được INSERT/UPDATE trong DynamoDB
→ Lambda tự động index sang OpenSearch
→ User có thể tìm kiếm full-text ngay lập tức
```

### 5. Audit Log (Nhật Ký Kiểm Toán)

```python
def lambda_handler(event, context):
    audit_records = []
    for record in event['Records']:
        audit_records.append({
            'TableName': 'Orders',
            'EventName': record['eventName'],
            'Keys': record['dynamodb']['Keys'],
            'Timestamp': record['dynamodb'].get('ApproximateCreationDateTime'),
            'OldData': record['dynamodb'].get('OldImage', {}),
            'NewData': record['dynamodb'].get('NewImage', {})
        })

    # Ghi vào S3 để lưu trữ dài hạn
    s3.put_object(
        Bucket='audit-logs-bucket',
        Key=f"orders/{context.aws_request_id}.json",
        Body=json.dumps(audit_records)
    )
```

---

## Xử Lý Lỗi & Retry

### Vấn Đề: Lambda Processing là At-Least-Once

```
DynamoDB Streams đảm bảo: each record xuất hiện đúng 1 lần trong stream
Lambda processing đảm bảo: at-least-once invoke

→ Lambda function có thể được gọi 2 lần với cùng record!
→ Handler PHẢI idempotent (an toàn khi chạy nhiều lần)
```

### Idempotency Pattern (Mẫu Idempotent)

```python
def handle_insert_idempotent(item):
    order_id = item['OrderId']

    # Kiểm tra đã xử lý chưa (dùng conditional write)
    try:
        processed_table.put_item(
            Item={'OrderId': order_id, 'ProcessedAt': datetime.utcnow().isoformat()},
            ConditionExpression='attribute_not_exists(OrderId)'
        )
        # Chỉ thực hiện business logic nếu chưa xử lý
        send_confirmation_email(item)

    except ClientError as e:
        if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
            print(f"Order {order_id} đã được xử lý — skip")
        else:
            raise
```

### Retry & Dead Letter Queue (Hàng Đợi Thư Chết)

```
Khi Lambda function throw exception:
1. Lambda ESM retry batch (cả batch bị retry, không chỉ record lỗi)
2. Nếu BisectBatchOnFunctionError=True: chia đôi batch, retry từng nửa
3. Sau MaximumRetryAttempts (mặc định không giới hạn): stuck forever!

→ Cần cấu hình DestinationConfig.OnFailure → SQS Dead Letter Queue
→ Records thất bại được gửi sang DLQ để xử lý thủ công
```

```python
# Lambda ESM với DLQ và retry limits
lambda_client.create_event_source_mapping(
    EventSourceArn=stream_arn,
    FunctionName='process-order-changes',
    StartingPosition='LATEST',
    BatchSize=10,
    MaximumRetryAttempts=3,              # Retry tối đa 3 lần
    BisectBatchOnFunctionError=True,     # Tìm record gây lỗi
    MaximumBatchingWindowInSeconds=30,   # Chờ 30s để gom batch
    DestinationConfig={
        'OnFailure': {
            'Destination': 'arn:aws:sqs:us-east-1:123:orders-dlq'
        }
    }
)
```

---

## Kinesis Data Streams cho DynamoDB

### Khi Nào Dùng Kinesis thay vì DynamoDB Streams?

Từ 2020, DynamoDB hỗ trợ stream data sang **Kinesis Data Streams** (Luồng Dữ Liệu Kinesis):

```
DynamoDB Table ──► Kinesis Data Streams ──► Nhiều consumers
                   (thay vì DynamoDB Streams)
```

**Ưu điểm Kinesis so với DynamoDB Streams:**

| Tiêu Chí              | DynamoDB Streams    | Kinesis Data Streams          |
| --------------------- | ------------------- | ----------------------------- |
| Retention             | 24 giờ              | 1 ngày đến 365 ngày           |
| Consumers             | 2 đồng thời         | Nhiều consumers (fan-out)     |
| Enhanced fan-out      | Không               | Có (dedicated throughput)     |
| Replay capability     | Giới hạn 24h        | Có thể replay đến 365 ngày    |
| Cost                  | Miễn phí (included) | Tính phí theo shard           |

---

## So Sánh Streams vs Kinesis

```
Dùng DynamoDB Streams khi:
✅ Tích hợp Lambda đơn giản
✅ Không cần retention > 24h
✅ Chỉ cần 1-2 consumers
✅ Không muốn thêm chi phí (Streams miễn phí)
✅ Cross-region replication đơn giản

Dùng Kinesis Data Streams khi:
✅ Cần retention dài hơn (replay, audit)
✅ Cần nhiều consumers đồng thời
✅ Cần replay data từ thời điểm cụ thể
✅ Kết hợp với data pipeline phức tạp (Kinesis Analytics, Firehose)
```

---

## Best Practices — Thực Hành Tốt Nhất

### 1. Lambda Handler

```python
# ✅ Xử lý từng record độc lập (không fail toàn batch vì 1 record)
def lambda_handler(event, context):
    failed_records = []
    for record in event['Records']:
        try:
            process_record(record)
        except Exception as e:
            print(f"Failed: {record['dynamodb']['Keys']}: {e}")
            failed_records.append(record)

    # Chỉ report failure nếu thực sự cần retry
    if failed_records:
        raise Exception(f"{len(failed_records)} records failed")
```

### 2. Batch Item Failures (Báo Lỗi Từng Item)

```python
# ✅ Dùng BatchItemFailures để chỉ retry records thất bại
def lambda_handler(event, context):
    batch_item_failures = []
    for record in event['Records']:
        try:
            process_record(record)
        except Exception as e:
            batch_item_failures.append({
                'itemIdentifier': record['dynamodb']['SequenceNumber']
            })

    return {'batchItemFailures': batch_item_failures}
    # Lambda ESM sẽ chỉ retry các records có trong batchItemFailures
```

### 3. Performance

```
✅ BatchSize lớn (100-10,000) cho throughput tốt
✅ MaximumBatchingWindowInSeconds để gom records
✅ Dùng async processing trong Lambda nếu có thể
✅ Monitor StreamReadThrottled metric trong CloudWatch
```

---

## Câu Hỏi Phỏng Vấn

**Q: DynamoDB Streams hoạt động như thế nào và khi nào dùng?**

A: DynamoDB Streams ghi lại mọi thay đổi dữ liệu (INSERT, UPDATE, DELETE) vào một ordered log (nhật ký có thứ tự), lưu trữ 24 giờ. Mỗi record trong stream chứa event type, old image, và new image (tùy cấu hình). Lambda được tích hợp qua Event Source Mapping để tự động xử lý records. Dùng Streams khi cần: real-time reaction to data changes (cache invalidation, search indexing, notifications), audit logging, hoặc cross-region replication.

**Q: Tại sao Lambda handler phải idempotent khi xử lý DynamoDB Streams?**

A: Lambda xử lý Streams theo cơ chế at-least-once — cùng một record có thể được xử lý nhiều lần do retry. Nếu Lambda throw exception, ESM (Event Source Mapping) retry toàn bộ batch. Idempotent handler đảm bảo xử lý cùng một record nhiều lần không gây side effects (tác dụng phụ) như gửi email trùng, charge tiền hai lần. Giải pháp: dùng conditional writes để track đã xử lý chưa, hoặc dùng BatchItemFailures để chỉ retry records thực sự thất bại.

**Q: Sự khác biệt giữa DynamoDB Streams và Kinesis Data Streams cho DynamoDB?**

A: DynamoDB Streams: miễn phí, retention 24h, tối đa 2 consumers, đơn giản để setup với Lambda. Kinesis Data Streams: có phí theo shard, retention 1-365 ngày, nhiều consumers với fan-out, hỗ trợ replay data từ thời điểm bất kỳ. Dùng Streams cho use cases đơn giản (cache invalidation, Lambda triggers). Dùng Kinesis khi cần retention dài, nhiều consumers, hoặc kết nối với data pipeline phức tạp.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Trạng Thái:** ✅ Hoàn Thành
