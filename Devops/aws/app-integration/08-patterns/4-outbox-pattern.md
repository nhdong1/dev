# Outbox Pattern — Đảm Bảo Nhất Quán DB và Messaging

> Outbox Pattern (Mẫu Hộp Thư Đi) giải quyết vấn đề dual-write (ghi đôi): làm sao đảm bảo "đã lưu DB thì chắc chắn có message, đã có message thì chắc chắn đã lưu DB".

---

## 📚 Mục Lục

1. [Vấn Đề Dual-Write](#vấn-đề-dual-write)
2. [Outbox Pattern Là Gì?](#outbox-pattern-là-gì)
3. [Hai Cách Triển Khai](#hai-cách-triển-khai)
4. [Polling-Based Outbox](#polling-based-outbox)
5. [CDC-Based Outbox — Change Data Capture](#cdc-based-outbox--change-data-capture)
6. [Triển Khai Trên AWS](#triển-khai-trên-aws)
7. [Inbox Pattern — Nhận An Toàn](#inbox-pattern--nhận-an-toàn)
8. [Ví Dụ Code Thực Tế](#ví-dụ-code-thực-tế)
9. [Trade-offs Và Giới Hạn](#trade-offs-và-giới-hạn)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Vấn Đề Dual-Write

### Kịch Bản Nguy Hiểm

```
Order Service nhận order mới:
1. INSERT INTO orders (id, status) VALUES ('123', 'CREATED')  ← DB write
2. sns.publish(topic='order-events', message='order.created') ← Publish event

Điều gì xảy ra nếu crash xảy ra?
```

**Trường Hợp 1: Crash giữa bước 1 và bước 2**

```
1. INSERT INTO orders ✅ (đã lưu)
   [CRASH]
2. sns.publish ❌ (không bao giờ xảy ra)

Kết quả:
- Database có order 'CREATED'
- Downstream services (Inventory, Payment) KHÔNG bao giờ nhận được event
- Order "mất tích" trong hệ thống, không được xử lý
```

**Trường Hợp 2: Publish thành công nhưng DB rollback**

```
1. INSERT INTO orders ✅ (trong transaction)
2. sns.publish ✅ (đã gửi event ra ngoài)
3. Transaction ROLLBACK vì lý do khác ❌

Kết quả:
- Database KHÔNG có order (đã rollback)
- Downstream services đã nhận event 'order.created'
- Inventory đã reserve stock cho đơn hàng không tồn tại
```

### Tại Sao Không Thể Dùng Distributed Transaction?

```
Giải pháp lý tưởng: Wrap cả DB write và message publish trong một transaction
Thực tế: SQS/SNS/EventBridge KHÔNG hỗ trợ distributed transaction với DB

Không thể làm:
BEGIN TRANSACTION with DB + SQS;  ← Không tồn tại
  INSERT INTO orders ...;
  SQS.send_message(...);
COMMIT;
```

---

## Outbox Pattern Là Gì?

**Ý tưởng cốt lõi:** Thay vì gửi message trực tiếp đến message broker, ghi message vào một **outbox table (bảng hộp thư đi)** trong CÙNG database transaction. Sau đó, một **message relay (bộ chuyển tiếp tin nhắn)** đọc từ outbox table và gửi lên message broker.

### Flow Hoạt Động

```
┌───────────────────────────────────────────┐
│              Service Process              │
│                                           │
│  BEGIN TRANSACTION                        │
│    INSERT INTO orders (...)     ┐         │
│    INSERT INTO outbox (event)   ┤ ATOMIC  │
│  COMMIT                         ┘         │
│                                           │
└───────────────────────────────────────────┘
                    │
                    │ (transaction committed)
                    ▼
┌───────────────────────────────────────────┐
│            Outbox Table (DB)              │
│  ┌──────┬──────────────┬─────────────┐   │
│  │ id   │ event_type   │ payload     │   │
│  │ 001  │ order.created│ {...}       │   │
│  └──────┴──────────────┴─────────────┘   │
└───────────────────────────────────────────┘
                    │
                    │ (Relay reads & publishes)
                    ▼
┌───────────────────────────────────────────┐
│         Message Relay Process             │
│  1. SELECT * FROM outbox WHERE sent=false │
│  2. Publish to SNS/SQS/EventBridge        │
│  3. UPDATE outbox SET sent=true           │
└───────────────────────────────────────────┘
                    │
                    ▼
            SNS / SQS / EventBridge
```

### Tại Sao An Toàn?

```
Kịch bản 1: Crash sau DB transaction commit
  → Outbox table đã có event → Relay sẽ gửi khi restart
  → Không mất event ✅

Kịch bản 2: Crash trước DB transaction commit
  → Transaction rollback → outbox KHÔNG có event
  → Consistency (nhất quán): DB không có order, outbox không có event ✅

Kịch bản 3: Relay crash sau publish nhưng trước UPDATE sent=true
  → Relay retry → đọc lại outbox → publish lại
  → Downstream nhận duplicate → cần Idempotency ở consumer ✅
```

---

## Hai Cách Triển Khai

### 1. Polling-Based Outbox

Relay định kỳ poll (lấy) từ outbox table và publish.

```
Relay process runs every N seconds:
  SELECT * FROM outbox WHERE sent = false ORDER BY created_at LIMIT 100;
  FOR EACH event:
    Publish to message broker;
    UPDATE outbox SET sent = true WHERE id = event.id;
```

**Ưu điểm:**
- Đơn giản, dễ implement
- Hoạt động với mọi database

**Nhược điểm:**
- Latency (độ trễ) phụ thuộc polling interval (khoảng thời gian polling)
- Database pressure (tải DB) từ constant polling
- Scaling phức tạp — nhiều relay process cần distributed lock (khóa phân tán)

### 2. CDC-Based Outbox — Change Data Capture

CDC (Change Data Capture — Thu Nạp Thay Đổi Dữ Liệu) đọc database transaction log (nhật ký giao dịch) trực tiếp và publish events.

```
Database Transaction Log (Write-Ahead Log — WAL):
  ← DynamoDB Streams, RDS/Aurora binlog, PostgreSQL WAL

CDC Tool (Debezium, AWS DMS):
  Reads WAL → Publishes to Kafka/EventBridge/SQS
```

**Ưu điểm:**
- Low latency (độ trễ thấp) — near real-time
- Không cần polling → giảm DB load
- Reliable (đáng tin cậy) — không miss changes

**Nhược điểm:**
- Phức tạp hơn để setup
- Phụ thuộc vào CDC tooling
- Cần access vào database transaction log

---

## Polling-Based Outbox

### Schema Outbox Table

```sql
CREATE TABLE outbox_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,  -- 'Order', 'Payment', ...
    aggregate_id VARCHAR(255) NOT NULL,    -- order_id, payment_id, ...
    event_type VARCHAR(100) NOT NULL,      -- 'order.created', 'payment.completed', ...
    payload JSONB NOT NULL,                -- Nội dung event
    created_at TIMESTAMP DEFAULT NOW(),
    sent_at TIMESTAMP,                     -- NULL nếu chưa gửi
    sent BOOLEAN DEFAULT FALSE,
    retry_count INT DEFAULT 0,
    error_message TEXT
);

-- Index để Relay query nhanh
CREATE INDEX idx_outbox_unsent ON outbox_events(created_at)
    WHERE sent = FALSE;
```

### Relay Process

```python
import boto3
import psycopg2
import json
import time

def relay_loop(conn, sns_client, topic_arn, interval_seconds=1):
    while True:
        try:
            process_batch(conn, sns_client, topic_arn)
        except Exception as e:
            print(f"Relay error: {e}")
        time.sleep(interval_seconds)

def process_batch(conn, sns_client, topic_arn):
    with conn.cursor() as cur:
        # Lock (khóa) batch để tránh concurrent relay (relay đồng thời)
        cur.execute("""
            SELECT id, event_type, payload, aggregate_id
            FROM outbox_events
            WHERE sent = FALSE
            ORDER BY created_at
            LIMIT 100
            FOR UPDATE SKIP LOCKED
        """)
        events = cur.fetchall()

        for event_id, event_type, payload, aggregate_id in events:
            try:
                # Publish đến SNS
                sns_client.publish(
                    TopicArn=topic_arn,
                    Message=json.dumps(payload),
                    MessageAttributes={
                        'event_type': {
                            'DataType': 'String',
                            'StringValue': event_type
                        }
                    },
                    MessageDeduplicationId=str(event_id)  # Idempotent publish
                )

                # Đánh dấu đã gửi
                cur.execute("""
                    UPDATE outbox_events
                    SET sent = TRUE, sent_at = NOW()
                    WHERE id = %s
                """, (event_id,))

            except Exception as e:
                cur.execute("""
                    UPDATE outbox_events
                    SET retry_count = retry_count + 1, error_message = %s
                    WHERE id = %s
                """, (str(e), event_id))

        conn.commit()
```

---

## CDC-Based Outbox — Change Data Capture

### DynamoDB Streams + Lambda (AWS Native)

```
DynamoDB Table (orders + outbox) 
    → DynamoDB Streams (luồng thay đổi)
    → Lambda trigger 
    → EventBridge / SNS / SQS
```

```python
# Lambda được trigger bởi DynamoDB Streams
def lambda_handler(event, context):
    for record in event['Records']:
        # Chỉ xử lý INSERT vào outbox table
        if record['eventName'] != 'INSERT':
            continue
        
        new_image = record['dynamodb'].get('NewImage', {})
        
        # Kiểm tra đây có phải outbox record không
        if new_image.get('record_type', {}).get('S') != 'OUTBOX':
            continue

        event_type = new_image['event_type']['S']
        payload = json.loads(new_image['payload']['S'])
        outbox_id = new_image['id']['S']

        # Publish đến EventBridge
        eventbridge.put_events(Entries=[{
            'Source': 'order-service',
            'DetailType': event_type,
            'Detail': json.dumps(payload),
            'EventBusName': 'order-bus'
        }])

        # Lambda tự retry nếu fail → Lambda phải idempotent
        # DynamoDB Streams đảm bảo at-least-once delivery
```

### Aurora/RDS với AWS Database Migration Service (DMS)

```
RDS PostgreSQL/MySQL
    → AWS DMS (Database Migration Service — Dịch Vụ Di Chuyển Cơ Sở Dữ Liệu)
    → Kinesis Data Streams
    → Lambda
    → EventBridge
```

---

## Triển Khai Trên AWS

### Option 1: PostgreSQL + SQS (Polling)

```
PostgreSQL (RDS Aurora)
    outbox_events table
          │
    Lambda (scheduled every 1s)
    EventBridge Scheduler rule
          │
    AWS SQS / SNS
```

### Option 2: DynamoDB + DynamoDB Streams (CDC)

```
DynamoDB single table:
  PK: ORDER#123, SK: OUTBOX#<uuid>
  record_type: 'OUTBOX'
  event_type: 'order.created'
  payload: {...}
  sent: false
      │
DynamoDB Streams (Change Data Capture)
      │
Lambda (triggered by stream)
      │
EventBridge / SNS / SQS
```

**DynamoDB Single-Table Design cho Outbox:**

```python
# Ghi order + outbox event trong một transaction
dynamodb = boto3.resource('dynamodb')

def create_order_with_outbox(order_data):
    # DynamoDB TransactWrite — atomic write tới nhiều item
    dynamodb.meta.client.transact_write(
        TransactItems=[
            # Item 1: Order record
            {
                'Put': {
                    'TableName': 'orders-table',
                    'Item': {
                        'PK': f"ORDER#{order_data['order_id']}",
                        'SK': 'METADATA',
                        'record_type': 'ORDER',
                        'status': 'CREATED',
                        'customer_id': order_data['customer_id'],
                        'amount': order_data['amount'],
                        'created_at': datetime.utcnow().isoformat()
                    },
                    'ConditionExpression': 'attribute_not_exists(PK)'
                }
            },
            # Item 2: Outbox event (trong cùng transaction)
            {
                'Put': {
                    'TableName': 'orders-table',
                    'Item': {
                        'PK': f"ORDER#{order_data['order_id']}",
                        'SK': f"OUTBOX#{str(uuid.uuid4())}",
                        'record_type': 'OUTBOX',
                        'event_type': 'order.created',
                        'payload': json.dumps(order_data),
                        'created_at': datetime.utcnow().isoformat(),
                        'sent': False
                    }
                }
            }
        ]
    )
```

---

## Inbox Pattern — Nhận An Toàn

**Inbox Pattern (Mẫu Hộp Thư Đến)** là đối xứng với Outbox — dùng ở consumer side để đảm bảo idempotency:

```
SNS/SQS → Consumer Service
               │
               │ Lưu message vào inbox_events table
               │ (trong cùng DB transaction với business logic)
               ▼
         inbox_events table
         ┌──────────────────────────────┐
         │ message_id (unique)           │
         │ event_type                    │
         │ payload                       │
         │ processed_at                  │
         └──────────────────────────────┘
```

**Kết hợp Outbox (gửi) + Inbox (nhận) = Guaranteed exactly-once processing.**

---

## Ví Dụ Code Thực Tế

### Service Layer Với Outbox

```python
class OrderService:
    def __init__(self, db_session, outbox_repo):
        self.db = db_session
        self.outbox = outbox_repo

    def create_order(self, order_data: dict) -> Order:
        with self.db.begin():  # Bắt đầu database transaction
            # Tạo order trong DB
            order = Order(
                id=str(uuid.uuid4()),
                customer_id=order_data['customer_id'],
                amount=order_data['amount'],
                status='CREATED'
            )
            self.db.add(order)

            # Ghi outbox event TRONG CÙNG TRANSACTION
            outbox_event = OutboxEvent(
                aggregate_id=order.id,
                aggregate_type='Order',
                event_type='order.created',
                payload=json.dumps({
                    'order_id': order.id,
                    'customer_id': order.customer_id,
                    'amount': float(order.amount),
                    'created_at': datetime.utcnow().isoformat()
                })
            )
            self.db.add(outbox_event)

            # Commit cả hai trong một transaction
            # Nếu bất kỳ bước nào fail → cả hai rollback

        return order
        # Sau khi return, Relay sẽ đọc outbox và gửi event
```

---

## Trade-offs Và Giới Hạn

### Ưu Điểm

```
✅ Đảm bảo at-least-once delivery của events (ít nhất một lần)
✅ Không mất event khi service crash
✅ Không gửi event sai khi transaction rollback
✅ Decouples (tách rời) business logic khỏi message broker
✅ Có thể retry events thất bại
```

### Nhược Điểm

```
❌ Tăng độ phức tạp — cần thêm outbox table và relay process
❌ Relay là thêm một component cần maintain
❌ Latency: event không được publish ngay tức thì (polling delay)
❌ Outbox table cần cleanup (xóa records cũ đã gửi)
❌ Consumer vẫn cần idempotency (Outbox chỉ đảm bảo at-least-once, không exactly-once)
```

### Khi Nào KHÔNG Cần Outbox Pattern

```
Không cần khi:
- Event đơn thuần là notification, không critical (không quan trọng)
- Có thể chấp nhận mất event (analytics, metrics)
- Dùng saga orchestration — Step Functions quản lý state externally
- Message broker hỗ trợ XA transaction (hiếm gặp)
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Outbox Pattern giải quyết vấn đề gì?

**Trả lời:**
> Dual-write problem (vấn đề ghi đôi): khi cần ghi vào DB và gửi message, nếu crash xảy ra giữa hai bước, hệ thống không nhất quán — DB có data nhưng không có event, hoặc có event nhưng DB không có data.
>
> Outbox Pattern giải quyết bằng cách ghi event vào outbox table trong cùng database transaction với business data. Vì cùng DB transaction, hai bước là atomic — cả hai commit hoặc cả hai rollback. Một relay process sau đó đọc outbox và publish đến message broker.

### Câu 2: Triển khai Outbox Pattern trên DynamoDB như thế nào?

**Trả lời:**
> DynamoDB hỗ trợ TransactWrite — ghi atomic đến nhiều item trong cùng request. Tôi tạo order item và outbox item trong cùng TransactWrite. DynamoDB Streams tự động capture changes → trigger Lambda → Lambda publish lên EventBridge.
>
> Lambda phải idempotent vì DynamoDB Streams là at-least-once. Dùng event ID từ stream làm idempotency key, lưu vào DynamoDB với conditional write để tránh double publish.

### Câu 3: Outbox đảm bảo exactly-once không?

**Trả lời:**
> Không — Outbox chỉ đảm bảo at-least-once. Relay có thể publish event rồi crash trước khi đánh dấu sent=true → retry lại → consumer nhận duplicate.
>
> Để đạt hiệu quả exactly-once: kết hợp Outbox (ở producer) + Idempotency (ở consumer). Consumer kiểm tra event ID đã xử lý chưa — nếu có thì skip. Kết hợp hai pattern này là "effectively exactly-once".

---

**Cập Nhật Lần Cuối:** 2026-05-18  
**Tags:** outbox-pattern, dual-write, CDC, DynamoDB-Streams, at-least-once, transactional-messaging
