# Idempotency — Tính Bất Biến Trong Distributed Systems

> Idempotency (Tính Bất Biến) đảm bảo rằng xử lý cùng một tin nhắn nhiều lần cho kết quả giống như xử lý một lần — thiết yếu trong mọi distributed system.

---

## 📚 Mục Lục

1. [Tại Sao Cần Idempotency?](#tại-sao-cần-idempotency)
2. [Delivery Guarantees — Đảm Bảo Giao Vận](#delivery-guarantees--đảm-bảo-giao-vận)
3. [Idempotency Key — Khóa Bất Biến](#idempotency-key--khóa-bất-biến)
4. [Triển Khai Idempotency](#triển-khai-idempotency)
5. [Idempotency Trong AWS Services](#idempotency-trong-aws-services)
6. [Deduplication — Loại Trùng](#deduplication--loại-trùng)
7. [Ví Dụ Code Thực Tế](#ví-dụ-code-thực-tế)
8. [Cạm Bẫy Thường Gặp](#cạm-bẫy-thường-gặp)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Idempotency?

### Kịch Bản Không Có Idempotency

```
1. Order Service gửi "process payment $100" đến SQS
2. Payment Lambda nhận message, bắt đầu charge thẻ
3. Charge thành công, nhưng Lambda crash trước khi delete message
4. SQS Visibility Timeout hết → message reappears (xuất hiện lại)
5. Payment Lambda nhận lại message → charge thẻ LẦN 2
6. Customer bị charge $200 thay vì $100 ❌
```

### Nguyên Nhân Tin Nhắn Bị Gửi Lặp

| Nguyên Nhân | Mô Tả |
|---|---|
| **Network retry (Thử lại mạng)** | Client retry do timeout — server đã nhận và xử lý lần đầu |
| **SQS at-least-once delivery** | Consumer crash sau khi xử lý nhưng trước khi delete message |
| **Lambda retry** | Lambda tự động retry khi có lỗi |
| **Step Functions retry** | Orchestrator retry bước bị lỗi |
| **Manual replay (Phát lại thủ công)** | Engineer replay event để fix bug |
| **EventBridge delivery retry** | EventBridge retry target khi target unavailable |

### Định Nghĩa Idempotent Operation

Một operation (thao tác) gọi là **idempotent** nếu:

```
f(f(x)) = f(x)

Ví dụ:
- SET status = 'PAID'        → Idempotent ✅ (set nhiều lần vẫn như nhau)
- balance = balance - 100    → KHÔNG idempotent ❌ (mỗi lần gọi trừ thêm)
- INSERT INTO payments...    → KHÔNG idempotent ❌ (thêm nhiều row trùng)
- PUT /payment/{id} {amount} → Idempotent ✅ (PUT với ID cụ thể)
- POST /payment {amount}     → KHÔNG idempotent ❌ (tạo mới mỗi lần)
```

---

## Delivery Guarantees — Đảm Bảo Giao Vận

### At-least-once Delivery (Giao Vận Ít Nhất Một Lần)

```
Đảm bảo: Message được giao ÍT NHẤT một lần
Rủi ro: Message có thể bị giao NHIỀU HƠN một lần

Dịch vụ AWS dùng at-least-once:
- SQS Standard Queue
- SNS Standard Topic  
- EventBridge
- Kinesis Data Streams

→ Consumer PHẢI implement idempotency
```

### At-most-once Delivery (Giao Vận Tối Đa Một Lần)

```
Đảm bảo: Message được giao TỐI ĐA một lần
Rủi ro: Message có thể bị MẤT

Dịch vụ dùng at-most-once:
- Fire-and-forget messaging
- UDP protocol

→ Chấp nhận mất message, không cần idempotency
  Dùng cho: metrics collection, real-time game state
```

### Exactly-once Delivery (Giao Vận Đúng Một Lần)

```
Đảm bảo: Message được giao ĐÚNG MỘT LẦN, không hơn không kém
Thực tế: Rất khó đạt được thực sự

Dịch vụ AWS hỗ trợ:
- SQS FIFO Queue (MessageDeduplicationId)
- SNS FIFO Topic

Lưu ý: "Exactly-once delivery" trong SQS FIFO thực chất là
        "deduplicated delivery" trong cửa sổ 5 phút
```

### Tóm Lại

```
At-least-once + Idempotent consumer = Effectively exactly-once
(Giao vận ít nhất một lần + Consumer bất biến = Hiệu quả đúng một lần)
```

**Đây là cách thực tế nhất để đạt exactly-once semantics (ngữ nghĩa đúng một lần).**

---

## Idempotency Key — Khóa Bất Biến

### Idempotency Key Là Gì?

Idempotency Key là định danh duy nhất (unique identifier) gắn với một logical operation (thao tác logic). Khi nhận request/message với cùng key → trả về kết quả cũ thay vì xử lý lại.

### Nơi Tạo Idempotency Key

```
1. Client tạo trước khi gửi request:
   idempotency_key = UUID()
   POST /payment
   Headers: Idempotency-Key: idempotency_key
   
2. Từ nội dung business:
   idempotency_key = f"payment-{order_id}-{customer_id}"
   
3. Từ message metadata:
   idempotency_key = sqs_message_id  (MessageId của SQS)
   idempotency_key = event.id        (EventBridge event ID)
```

### Thiết Kế Idempotency Key Tốt

```
✅ Duy nhất cho mỗi logical operation
✅ Stable (ổn định) — không thay đổi khi retry
✅ Có ý nghĩa business (dễ debug)
✅ Đủ ngắn để store trong DB

Ví dụ tốt:
- "charge-order-{order_id}-attempt-{attempt_number}"
- "reserve-{product_id}-for-{order_id}"
- SQS MessageDeduplicationId

Ví dụ xấu:
- Timestamp đơn thuần (có thể trùng)
- Random UUID mỗi lần retry (mất ý nghĩa)
```

---

## Triển Khai Idempotency

### Cách 1: Database Unique Constraint (Ràng Buộc Duy Nhất)

```sql
-- Tạo bảng với unique constraint trên idempotency_key
CREATE TABLE payment_operations (
    id SERIAL PRIMARY KEY,
    idempotency_key VARCHAR(255) UNIQUE NOT NULL,
    order_id VARCHAR(255) NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- INSERT sẽ fail nếu key đã tồn tại
INSERT INTO payment_operations (idempotency_key, order_id, amount, status)
VALUES ('charge-order-123-attempt-1', '123', 100.00, 'COMPLETED')
ON CONFLICT (idempotency_key) DO NOTHING;
```

**Ưu điểm:** Đơn giản, database đảm bảo atomicity.  
**Nhược điểm:** Mỗi check cần database query.

### Cách 2: Check-and-Store Pattern (Kiểm Tra Và Lưu)

```python
import boto3
import json
from datetime import datetime, timedelta

dynamodb = boto3.resource('dynamodb')
idempotency_table = dynamodb.Table('idempotency-records')

def idempotent_process_payment(idempotency_key, order_id, amount):
    # Bước 1: Kiểm tra idempotency key đã tồn tại chưa
    existing = idempotency_table.get_item(
        Key={'idempotency_key': idempotency_key}
    )

    if 'Item' in existing:
        # Đã xử lý rồi → trả về kết quả cũ
        stored = existing['Item']
        if stored['status'] == 'COMPLETED':
            return stored['result']
        elif stored['status'] == 'IN_PROGRESS':
            # Đang xử lý bởi instance khác (concurrent request)
            raise Exception('PROCESSING_IN_PROGRESS')
        # Nếu status == 'FAILED' → có thể retry

    # Bước 2: Đánh dấu IN_PROGRESS (đang xử lý) trước khi thực hiện
    # Conditional write (ghi có điều kiện) để tránh race condition
    try:
        idempotency_table.put_item(
            Item={
                'idempotency_key': idempotency_key,
                'status': 'IN_PROGRESS',
                'created_at': datetime.utcnow().isoformat(),
                'ttl': int((datetime.utcnow() + timedelta(hours=24)).timestamp())
            },
            ConditionExpression='attribute_not_exists(idempotency_key)'
        )
    except dynamodb.meta.client.exceptions.ConditionalCheckFailedException:
        raise Exception('CONCURRENT_REQUEST')

    # Bước 3: Thực hiện thao tác thực sự
    try:
        result = charge_payment(order_id, amount)

        # Bước 4: Lưu kết quả và đánh dấu COMPLETED
        idempotency_table.update_item(
            Key={'idempotency_key': idempotency_key},
            UpdateExpression='SET #s = :status, #r = :result',
            ExpressionAttributeNames={'#s': 'status', '#r': 'result'},
            ExpressionAttributeValues={
                ':status': 'COMPLETED',
                ':result': json.dumps(result)
            }
        )
        return result

    except Exception as e:
        # Đánh dấu FAILED để cho phép retry
        idempotency_table.update_item(
            Key={'idempotency_key': idempotency_key},
            UpdateExpression='SET #s = :status, error_message = :err',
            ExpressionAttributeNames={'#s': 'status'},
            ExpressionAttributeValues={
                ':status': 'FAILED',
                ':err': str(e)
            }
        )
        raise
```

### Cách 3: AWS Lambda Powertools Idempotency

AWS Lambda Powertools (Công Cụ Mạnh Cho Lambda) cung cấp built-in idempotency decorator (trang trí):

```python
from aws_lambda_powertools.utilities.idempotency import (
    idempotent, 
    IdempotencyConfig,
    DynamoDBPersistenceLayer
)

# Cấu hình persistence layer (tầng lưu trữ)
persistence_layer = DynamoDBPersistenceLayer(
    table_name="idempotency-table"
)

config = IdempotencyConfig(
    event_key_jmespath="body.order_id",  # Lấy key từ event
    expires_after_seconds=3600           # TTL 1 giờ
)

@idempotent(config=config, persistence_store=persistence_layer)
def lambda_handler(event, context):
    # Lambda Powertools tự xử lý idempotency
    # Code bình thường ở đây
    order_id = event['body']['order_id']
    result = process_order(order_id)
    return result
```

---

## Idempotency Trong AWS Services

### SQS — Message Deduplication

```
SQS Standard Queue:
- At-least-once delivery → consumer phải implement idempotency
- Dùng MessageId (ID Tin Nhắn) của SQS làm idempotency key

SQS FIFO Queue:
- MessageDeduplicationId: deduplication window (cửa sổ loại trùng) 5 phút
- ContentBasedDeduplication: tự động tính hash từ message body
- Nếu gửi 2 message cùng MessageDeduplicationId trong 5 phút → chỉ nhận 1
```

```python
# Gửi message với MessageDeduplicationId
sqs = boto3.client('sqs')
response = sqs.send_message(
    QueueUrl='https://sqs.us-east-1.amazonaws.com/123/my-queue.fifo',
    MessageBody=json.dumps(order_event),
    MessageGroupId='order-processing',
    MessageDeduplicationId=f"order-{order_id}-v1"  # Unique, stable
)
```

### Lambda — Event Source Mapping

```
Lambda với SQS trigger:
- Lambda nhận batch (lô) message, xử lý, delete nếu thành công
- Nếu Lambda throw exception → message reappears sau visibility timeout
- Lambda tự động retry (mặc định không giới hạn)

→ Lambda handler PHẢI idempotent

Lambda với EventBridge:
- EventBridge retry khi Lambda trả về error
- Default retry: 2 lần trong 24 giờ (Standard), exponential backoff
```

### DynamoDB — Conditional Writes

```python
# Conditional write — chỉ insert nếu chưa tồn tại
try:
    table.put_item(
        Item={'id': order_id, 'status': 'CREATED', ...},
        ConditionExpression='attribute_not_exists(id)'
    )
    # Thành công → lần đầu xử lý
except ConditionalCheckFailedException:
    # Đã tồn tại → đây là duplicate, ignore
    pass
```

### API Gateway — Idempotency Header

```
REST API best practice:
- Client gửi header: Idempotency-Key: <unique-uuid>
- Server lưu kết quả của request này
- Retry với cùng key → trả về cached result (kết quả đã lưu)

AWS API Gateway không có built-in idempotency
→ Implement trong Lambda handler hoặc dùng Lambda Powertools
```

---

## Deduplication — Loại Trùng

### Deduplication vs Idempotency

| | Deduplication (Loại Trùng) | Idempotency (Bất Biến) |
|---|---|---|
| **Ai xử lý** | Infrastructure (hạ tầng) | Application (ứng dụng) |
| **Cơ chế** | Drop duplicate message | Process nhưng trả kết quả cũ |
| **Ví dụ** | SQS FIFO MessageDeduplicationId | Lambda check idempotency key |
| **Giới hạn** | Cửa sổ thời gian có hạn | Tuỳ TTL của idempotency store |

### SQS FIFO Deduplication

```
MessageDeduplicationId workflow (luồng xử lý):
1. Producer gửi message với MessageDeduplicationId = "order-123-v1"
2. SQS lưu ID trong deduplication store (cửa sổ 5 phút)
3. Producer gửi lại (retry) với cùng MessageDeduplicationId
4. SQS phát hiện trùng → SỰ KIỆN NÀY ĐƯỢC THÊM VÀO QUEUE NHƯNG KHÔNG ĐƯỢC GIAO ĐẾN CONSUMER
5. Consumer chỉ nhận message một lần

Lưu ý:
- Cửa sổ 5 phút — sau đó có thể giao lại
- ContentBasedDeduplication: hash MD5 của message body làm deduplication ID
```

---

## Ví Dụ Code Thực Tế

### SQS Consumer Idempotent (Hoàn Chỉnh)

```python
import boto3
import json
import hashlib
from datetime import datetime, timedelta

dynamodb = boto3.resource('dynamodb')
processed_table = dynamodb.Table('processed-messages')

def lambda_handler(event, context):
    for record in event['Records']:
        message_id = record['messageId']
        body = json.loads(record['body'])

        try:
            process_message_idempotent(message_id, body)
        except Exception as e:
            # Nếu lỗi → message không bị xóa, sẽ retry
            print(f"Error processing {message_id}: {e}")
            raise

def process_message_idempotent(message_id, body):
    # Kiểm tra đã xử lý chưa
    existing = processed_table.get_item(Key={'message_id': message_id})

    if 'Item' in existing:
        print(f"Message {message_id} already processed, skipping")
        return existing['Item']['result']

    # Xử lý message
    result = do_business_logic(body)

    # Đánh dấu đã xử lý
    processed_table.put_item(
        Item={
            'message_id': message_id,
            'processed_at': datetime.utcnow().isoformat(),
            'result': json.dumps(result),
            'ttl': int((datetime.utcnow() + timedelta(days=7)).timestamp())
        }
    )

    return result
```

### Payment Service Idempotent

```python
def charge_customer(idempotency_key: str, customer_id: str, amount: float):
    """
    Idempotent payment charge.
    Same idempotency_key → same result, no double charge.
    """

    # Kiểm tra trong idempotency store
    cached = get_from_store(idempotency_key)
    if cached:
        return cached  # Trả kết quả cũ, không charge lại

    # Gọi payment gateway
    charge_result = payment_gateway.charge(
        customer_id=customer_id,
        amount=amount,
        idempotency_key=idempotency_key  # Gửi key đến gateway để gateway cũng idempotent
    )

    # Lưu vào idempotency store
    save_to_store(idempotency_key, charge_result, ttl_hours=24)

    return charge_result
```

---

## Cạm Bẫy Thường Gặp

### 1. Idempotency Chỉ Ở Một Tầng

```
❌ Sai: Chỉ implement idempotency ở API layer nhưng bên trong gọi
        external payment API không có idempotency_key
        
→ Nếu external API không nhận idempotency_key, có thể charge 2 lần

✅ Đúng: Propagate (lan truyền) idempotency key qua toàn bộ call chain
         Mọi external API call phải có idempotency_key
```

### 2. TTL Quá Ngắn

```
❌ Sai: Idempotency records TTL = 1 giờ, nhưng retry có thể xảy ra 
        sau 24 giờ do manual replay
        
✅ Đúng: TTL phải > thời gian retry window tối đa
         SQS maxReceiveCount × visibility timeout × retention period
         Thường: 24 giờ đến 7 ngày
```

### 3. Race Condition (Điều Kiện Chạy Đua)

```
❌ Sai:
   Thread 1: Check → not exists → start processing
   Thread 2: Check → not exists → start processing (cùng lúc!)
   → Xử lý 2 lần dù có check

✅ Đúng: Dùng conditional write (ghi có điều kiện):
   DynamoDB: ConditionExpression='attribute_not_exists(id)'
   Database: UNIQUE constraint + ON CONFLICT DO NOTHING
   Redis: SET NX EX
```

### 4. Partial Success (Thành Công Một Phần)

```
❌ Sai: Đánh dấu COMPLETED trước khi hoàn thành tất cả bước
        → Nếu crash sau đó, các bước còn lại bị bỏ qua mãi mãi

✅ Đúng: Chỉ đánh dấu COMPLETED khi TẤT CẢ bước hoàn thành
         Dùng status machine: PENDING → IN_PROGRESS → COMPLETED/FAILED
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Tại sao SQS consumer phải idempotent?

**Trả lời:**
> SQS Standard Queue đảm bảo at-least-once delivery — không phải exactly-once. Message có thể được giao nhiều hơn một lần vì nhiều lý do: consumer crash sau khi xử lý nhưng trước khi delete message, visibility timeout quá ngắn, network partition.
>
> Nếu consumer không idempotent: một message "charge payment $100" bị giao 2 lần → customer bị charge $200.
>
> Cách implement: dùng SQS MessageId làm idempotency key, lưu vào DynamoDB với TTL. Trước khi xử lý, kiểm tra key đã tồn tại chưa. Nếu có → skip (bỏ qua) và trả về kết quả cũ.

### Câu 2: Phân biệt at-least-once, at-most-once, và exactly-once?

**Trả lời:**
> - **At-least-once (Ít nhất một lần):** Đảm bảo message đến đích nhưng có thể bị duplicate. SQS Standard, SNS Standard dùng cơ chế này. Consumer cần idempotent.
> - **At-most-once (Tối đa một lần):** Message chỉ giao nhiều nhất một lần, có thể mất. Dùng khi mất message chấp nhận được (metrics, analytics).
> - **Exactly-once (Đúng một lần):** Khó đạt được thực sự. SQS FIFO hỗ trợ deduplication trong 5 phút. Thực tế: at-least-once + idempotent consumer = effectively exactly-once.

### Câu 3: Implement idempotency như thế nào trong Lambda + SQS?

**Trả lời:**
> 1. Tạo DynamoDB table `idempotency-records` với `message_id` làm partition key và TTL attribute.
> 2. Trong Lambda handler: trước khi xử lý, check DynamoDB xem MessageId đã tồn tại chưa.
> 3. Nếu có → return cached result, không xử lý lại.
> 4. Nếu chưa → xử lý, sau đó write vào DynamoDB với conditional write (attribute_not_exists) để tránh race condition.
> 5. Dùng TTL = 7 ngày (hoặc lớn hơn SQS message retention period).
>
> Hoặc đơn giản hơn: dùng AWS Lambda Powertools với @idempotent decorator — tự xử lý DynamoDB storage.

---

**Cập Nhật Lần Cuối:** 2026-05-18  
**Tags:** idempotency, at-least-once, exactly-once, deduplication, SQS, Lambda
