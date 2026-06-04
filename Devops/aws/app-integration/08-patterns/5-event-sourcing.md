# Event Sourcing — Lưu Trữ Theo Chuỗi Sự Kiện

> Event Sourcing (Nguồn Sự Kiện) là kiến trúc lưu trạng thái hệ thống không phải bằng giá trị hiện tại, mà bằng chuỗi tất cả sự kiện đã xảy ra — cho phép audit đầy đủ và tái tạo trạng thái tại bất kỳ thời điểm.

---

## 📚 Mục Lục

1. [Traditional vs Event Sourcing](#traditional-vs-event-sourcing)
2. [Các Khái Niệm Cốt Lõi](#các-khái-niệm-cốt-lõi)
3. [Event Store — Kho Sự Kiện](#event-store--kho-sự-kiện)
4. [Event Replay — Phát Lại Sự Kiện](#event-replay--phát-lại-sự-kiện)
5. [Snapshot — Ảnh Chụp Trạng Thái](#snapshot--ảnh-chụp-trạng-thái)
6. [Triển Khai Trên AWS](#triển-khai-trên-aws)
7. [Ví Dụ Code Thực Tế](#ví-dụ-code-thực-tế)
8. [Ưu Điểm Và Nhược Điểm](#ưu-điểm-và-nhược-điểm)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Traditional vs Event Sourcing

### Traditional State Storage (Lưu Trữ Trạng Thái Truyền Thống)

```
Database lưu trạng thái HIỆN TẠI:

Order Table:
┌──────────┬────────────┬──────────────┐
│ order_id │ status     │ total_amount │
├──────────┼────────────┼──────────────┤
│   123    │ SHIPPED    │    $250      │
└──────────┴────────────┴──────────────┘

Câu hỏi không thể trả lời:
- Order này đã ở trạng thái nào trước khi SHIPPED?
- Ai đã thay đổi status và lúc mấy giờ?
- Tại sao amount là $250, có discount không?
- Đã có bao nhiêu lần retry payment?
```

### Event Sourcing Storage (Lưu Trữ Theo Sự Kiện)

```
Event Store lưu TẤT CẢ SỰ KIỆN:

Events Table:
┌──────────┬──────────────────────┬────────────────────────────────────────┬─────────────┐
│ order_id │ event_type           │ payload                                │ occurred_at │
├──────────┼──────────────────────┼────────────────────────────────────────┼─────────────┤
│   123    │ order.created        │ {customer: 'Alice', items: [...]}      │ 10:00:00    │
│   123    │ discount.applied     │ {code: 'SAVE10', amount: 25}          │ 10:00:01    │
│   123    │ payment.attempted    │ {amount: 225, card: '****1234'}       │ 10:00:05    │
│   123    │ payment.failed       │ {reason: 'INSUFFICIENT_FUNDS'}        │ 10:00:06    │
│   123    │ payment.retried      │ {amount: 225, card: '****5678'}       │ 10:01:00    │
│   123    │ payment.completed    │ {transaction_id: 'TXN-999'}           │ 10:01:02    │
│   123    │ order.fulfilled      │ {warehouse: 'HAN', picker: 'Bob'}     │ 11:00:00    │
│   123    │ order.shipped        │ {tracking: 'VN123456', carrier: 'GHN'}│ 14:00:00    │
└──────────┴──────────────────────┴────────────────────────────────────────┴─────────────┘

Câu hỏi có thể trả lời:
- Trạng thái hiện tại: SHIPPED ✅
- Lịch sử đầy đủ: 8 sự kiện, rõ từng bước ✅
- Lý do: đã retry payment 1 lần, có discount SAVE10 ✅
- Audit: ai làm gì lúc mấy giờ ✅
```

---

## Các Khái Niệm Cốt Lõi

### Event (Sự Kiện)

```
Event là một fact (sự thật) đã xảy ra — không thể thay đổi (immutable):

{
  "event_id": "EVT-001",
  "event_type": "order.created",
  "aggregate_id": "ORDER-123",    ← ID của entity được event này mô tả
  "aggregate_type": "Order",
  "sequence_number": 1,           ← Thứ tự trong aggregate
  "occurred_at": "2026-05-18T10:00:00Z",
  "payload": {
    "customer_id": "CUST-456",
    "items": [{"product_id": "P1", "qty": 2}],
    "total": 250.00
  },
  "metadata": {
    "user_id": "user-789",        ← Ai tạo event
    "correlation_id": "REQ-999"   ← Trace ID
  }
}

Quy tắc đặt tên event: QUAKHỨ (past tense)
  ✅ order.created, payment.completed, item.shipped
  ❌ order.create, do.payment (hiện tại/mệnh lệnh)
```

### Aggregate (Tập Hợp)

```
Aggregate là một entity (thực thể) hoặc nhóm entity liên quan,
được biểu diễn bằng chuỗi events:

Order Aggregate:
  ┌─────────────────────────────────────────────┐
  │ Events: [order.created, payment.completed,   │
  │          order.fulfilled, order.shipped]      │
  │                                              │
  │ Current State = Reduce(events, initial_state)│
  │ → {status: 'SHIPPED', amount: 250}           │
  └─────────────────────────────────────────────┘
```

### Projection (Phép Chiếu)

```
Projection là quá trình "chiếu" (ánh xạ) events sang một view (góc nhìn) cụ thể:

Event stream (luồng sự kiện) → Projection → Read Model (mô hình đọc)

Ví dụ:
Events về orders → Projection "Orders by Customer" → 
  {customer_id: 'Alice', orders: [ORDER-123, ORDER-456]}

Cùng một event stream, nhiều projection khác nhau:
  → "Order Status Dashboard" view
  → "Revenue by Day" view
  → "Audit Log" view
```

---

## Event Store — Kho Sự Kiện

### Yêu Cầu Của Event Store

```
1. Append-only (Chỉ Thêm): không update, không delete events
2. Ordered (Có Thứ Tự): events trong một aggregate phải có thứ tự
3. Optimistic concurrency (Đồng Thời Lạc Quan): tránh write conflict
4. Query by aggregate: lấy tất cả events của một ORDER-123
5. Subscribe (Đăng Ký): notify downstream khi có event mới
```

### Schema Event Store với DynamoDB

```
Table: event-store
Partition Key: aggregate_id  (ví dụ: "ORDER-123")
Sort Key: sequence_number    (ví dụ: 1, 2, 3, ...)

┌──────────────┬─────────────────┬──────────────────────┬─────────┬─────────────┐
│ aggregate_id │ sequence_number │ event_type           │ payload │ occurred_at │
├──────────────┼─────────────────┼──────────────────────┼─────────┼─────────────┤
│ ORDER-123    │ 1               │ order.created        │ {...}   │ ...         │
│ ORDER-123    │ 2               │ payment.completed    │ {...}   │ ...         │
│ ORDER-123    │ 3               │ order.shipped        │ {...}   │ ...         │
│ ORDER-456    │ 1               │ order.created        │ {...}   │ ...         │
└──────────────┴─────────────────┴──────────────────────┴─────────┴─────────────┘
```

### Optimistic Concurrency — Tránh Conflict

```python
def append_event(aggregate_id, event, expected_version):
    dynamodb.put_item(
        TableName='event-store',
        Item={
            'aggregate_id': {'S': aggregate_id},
            'sequence_number': {'N': str(expected_version + 1)},
            'event_type': {'S': event['type']},
            'payload': {'S': json.dumps(event['payload'])},
            'occurred_at': {'S': datetime.utcnow().isoformat()}
        },
        # Chỉ insert nếu sequence_number này chưa tồn tại
        # → Tránh 2 process cùng ghi sequence_number giống nhau
        ConditionExpression='attribute_not_exists(sequence_number)'
    )
```

**Optimistic Concurrency (Đồng Thời Lạc Quan):**
```
Process A và Process B đều muốn append event vào ORDER-123 (hiện có 3 events):
- A đọc version=3, chuẩn bị ghi sequence=4
- B đọc version=3, chuẩn bị ghi sequence=4
- A ghi sequence=4 → thành công
- B cố ghi sequence=4 → ConditionExpression FAIL
- B reload events, tính lại logic, retry
```

---

## Event Replay — Phát Lại Sự Kiện

### Tái Tạo Aggregate State

```python
class OrderAggregate:
    def __init__(self):
        self.order_id = None
        self.status = None
        self.amount = 0
        self.version = 0

    def apply(self, event):
        """Áp dụng một event để cập nhật state."""
        handlers = {
            'order.created': self._on_order_created,
            'payment.completed': self._on_payment_completed,
            'order.shipped': self._on_order_shipped,
        }
        handler = handlers.get(event['event_type'])
        if handler:
            handler(event['payload'])
        self.version += 1

    def _on_order_created(self, payload):
        self.order_id = payload['order_id']
        self.status = 'CREATED'
        self.amount = payload['amount']

    def _on_payment_completed(self, payload):
        self.status = 'PAID'

    def _on_order_shipped(self, payload):
        self.status = 'SHIPPED'
        self.tracking_number = payload.get('tracking')


def load_aggregate(aggregate_id: str) -> OrderAggregate:
    """Tải aggregate từ event store bằng cách replay tất cả events."""
    events = get_events_from_store(aggregate_id)  # Query DynamoDB

    aggregate = OrderAggregate()
    for event in events:  # Replay từ đầu đến cuối
        aggregate.apply(event)

    return aggregate
```

### Temporal Query — Truy Vấn Theo Thời Gian

```python
def get_aggregate_at_time(aggregate_id: str, point_in_time: datetime) -> OrderAggregate:
    """Tái tạo trạng thái của aggregate tại một thời điểm cụ thể."""
    all_events = get_events_from_store(aggregate_id)
    
    # Chỉ replay events xảy ra TRƯỚC thời điểm cần
    events_before = [e for e in all_events 
                     if datetime.fromisoformat(e['occurred_at']) <= point_in_time]
    
    aggregate = OrderAggregate()
    for event in events_before:
        aggregate.apply(event)
    
    return aggregate

# Truy vấn: Order 123 ở trạng thái nào lúc 10:30?
state_at_1030 = get_aggregate_at_time('ORDER-123', datetime(2026, 5, 18, 10, 30))
```

---

## Snapshot — Ảnh Chụp Trạng Thái

### Vấn Đề Với Aggregate Nhiều Events

```
ORDER-123 sau 1 năm: 500 events
→ Mỗi lần load aggregate phải replay 500 events → Chậm!
```

### Giải Pháp: Snapshot

```
Snapshot là "bản lưu nhanh" của aggregate state tại một thời điểm.
Thay vì replay từ event đầu, replay từ snapshot gần nhất:

All events:  1 2 3 4 5 ... 200 [SNAPSHOT] 201 202 ... 500
                                     ↑
                             Load snapshot + replay từ 201 đến 500
                             → Chỉ cần replay 300 events thay vì 500
```

### Snapshot Strategy (Chiến Lược Chụp)

```python
SNAPSHOT_THRESHOLD = 50  # Tạo snapshot sau mỗi 50 events

def load_aggregate_with_snapshot(aggregate_id: str) -> OrderAggregate:
    # Tìm snapshot gần nhất
    snapshot = get_latest_snapshot(aggregate_id)
    
    aggregate = OrderAggregate()
    start_version = 0
    
    if snapshot:
        # Khôi phục từ snapshot
        aggregate.restore_from_snapshot(snapshot['state'])
        start_version = snapshot['version']
    
    # Chỉ replay events SAU snapshot
    remaining_events = get_events_from_store(
        aggregate_id, 
        after_version=start_version
    )
    
    for event in remaining_events:
        aggregate.apply(event)
    
    # Tạo snapshot mới nếu đủ ngưỡng
    if len(remaining_events) >= SNAPSHOT_THRESHOLD:
        save_snapshot(aggregate_id, aggregate.to_dict(), aggregate.version)
    
    return aggregate
```

---

## Triển Khai Trên AWS

### Option 1: DynamoDB + DynamoDB Streams

```
Architect:
┌─────────────────────────────────────────────────────┐
│                  Event Store                         │
│             (DynamoDB Table)                         │
│  PK: aggregate_id, SK: sequence_number               │
└────────────────────┬────────────────────────────────┘
                     │
              DynamoDB Streams
                     │
              ┌──────┴──────┐
              │             │
           Lambda        Lambda
         (Projector 1) (Projector 2)
              │             │
    ┌─────────┴─┐    ┌──────┴────────┐
    │ Read Model │    │  Read Model   │
    │(DynamoDB)  │    │ (OpenSearch)  │
    └────────────┘    └───────────────┘
```

### Option 2: Kinesis + S3 (Event Log)

```
Service → Kinesis Data Streams → Lambda → DynamoDB (hot store)
                                        → S3 Parquet (cold store / long-term)
                                        → OpenSearch (full-text search)

Kinesis: event log theo thời gian thực
S3 Parquet: lưu trữ dài hạn, query với Athena
```

### Option 3: EventBridge + S3 Archive

```
Service → EventBridge → S3 Archive (tự động)
                      → Lambda (real-time projections)
                      → Step Functions (sagas)

EventBridge Archive:
- Lưu mọi event theo S3
- Replay theo time range (khoảng thời gian)
- Query với S3 Select hoặc Athena
```

---

## Ví Dụ Code Thực Tế

### Event Store với DynamoDB

```python
import boto3
import json
import uuid
from datetime import datetime

dynamodb = boto3.resource('dynamodb')
event_store = dynamodb.Table('event-store')

def append_events(aggregate_id: str, events: list, expected_version: int):
    """Ghi nhiều events vào event store với optimistic concurrency."""
    with event_store.batch_writer() as batch:
        for i, event in enumerate(events):
            sequence = expected_version + i + 1
            batch.put_item(Item={
                'aggregate_id': aggregate_id,
                'sequence_number': sequence,
                'event_id': str(uuid.uuid4()),
                'event_type': event['type'],
                'payload': json.dumps(event['data']),
                'occurred_at': datetime.utcnow().isoformat(),
                'aggregate_version': sequence
            })

def get_events(aggregate_id: str, after_version: int = 0) -> list:
    """Lấy tất cả events của aggregate, từ after_version trở đi."""
    response = event_store.query(
        KeyConditionExpression='aggregate_id = :id AND sequence_number > :v',
        ExpressionAttributeValues={
            ':id': aggregate_id,
            ':v': after_version
        }
    )
    return response['Items']
```

### Projection Builder

```python
def build_order_status_projection(aggregate_id: str) -> dict:
    """Xây dựng projection 'Order Status' từ events."""
    events = get_events(aggregate_id)
    
    projection = {
        'order_id': aggregate_id,
        'status': None,
        'total_amount': 0,
        'payment_attempts': 0,
        'last_updated': None
    }
    
    event_handlers = {
        'order.created': lambda p, e: {**p, 
            'status': 'CREATED', 
            'total_amount': e['amount'],
            'customer_id': e['customer_id']
        },
        'payment.attempted': lambda p, e: {**p,
            'payment_attempts': p['payment_attempts'] + 1
        },
        'payment.completed': lambda p, e: {**p, 'status': 'PAID'},
        'order.shipped': lambda p, e: {**p, 
            'status': 'SHIPPED',
            'tracking_number': e.get('tracking')
        }
    }
    
    for event in events:
        handler = event_handlers.get(event['event_type'])
        if handler:
            projection = handler(projection, json.loads(event['payload']))
        projection['last_updated'] = event['occurred_at']
    
    return projection
```

---

## Ưu Điểm Và Nhược Điểm

### Ưu Điểm

```
✅ Audit log (Nhật ký kiểm tra) đầy đủ — không mất lịch sử
✅ Temporal queries (Truy vấn theo thời gian) — "hệ thống lúc T trông như thế nào?"
✅ Event replay (Phát lại) — sửa bug bằng cách replay với logic mới
✅ Multiple projections — cùng data, nhiều view khác nhau
✅ Decouples (Tách rời) write model và read model tự nhiên
✅ Dễ tích hợp với CQRS
✅ Debugging dễ hơn — có đầy đủ context
```

### Nhược Điểm

```
❌ Phức tạp hơn nhiều so với CRUD truyền thống
❌ Eventual consistency của projections — đọc có thể lag
❌ Schema evolution (Tiến hóa lược đồ) khó — events cũ có format cũ
❌ Event store tăng kích thước theo thời gian (append-only)
❌ Query hiện tại phức tạp hơn — phải qua projection
❌ Không phù hợp cho simple CRUD applications
```

### Schema Evolution — Thay Đổi Cấu Trúc Event

```
Vấn đề: 6 tháng sau muốn đổi tên field "amount" thành "total_amount"
Không thể sửa events cũ (immutable)

Giải pháp 1: Versioned events (Events có phiên bản)
  event_type: "order.created.v2" (thêm version suffix)
  Apply handler: 
    if event.version == 1: map amount → total_amount
    if event.version == 2: dùng total_amount trực tiếp

Giải pháp 2: Upcasters (Nâng Cấp Sự Kiện)
  Khi đọc events, transform từ old schema sang new schema on-the-fly
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Event Sourcing là gì, khác gì database truyền thống?

**Trả lời:**
> Database truyền thống lưu trạng thái hiện tại — mỗi update ghi đè giá trị cũ. Event Sourcing lưu toàn bộ lịch sử dưới dạng chuỗi events không thể thay đổi (immutable). Trạng thái hiện tại được tính bằng cách replay tất cả events từ đầu đến cuối.
>
> Ưu điểm chính: có đầy đủ audit trail, có thể tái tạo trạng thái tại bất kỳ thời điểm nào, dễ dàng thêm projections mới mà không cần migrate data.
>
> Nhược điểm: phức tạp hơn nhiều, schema evolution khó vì events cũ không thể sửa.

### Câu 2: Khi nào nên dùng Event Sourcing?

**Trả lời:**
> Nên dùng khi: cần audit log đầy đủ (tài chính, healthcare, legal), cần temporal queries, hoặc business logic phức tạp với nhiều state transitions.
>
> Không nên dùng khi: simple CRUD, không cần audit trail, team chưa có kinh nghiệm với pattern này. Event Sourcing thêm rất nhiều complexity — chỉ dùng khi thực sự cần thiết.

### Câu 3: Snapshot pattern giải quyết vấn đề gì?

**Trả lời:**
> Khi aggregate tích lũy nhiều events theo thời gian, việc replay toàn bộ từ đầu để lấy state hiện tại sẽ ngày càng chậm.
>
> Snapshot giải quyết bằng cách định kỳ lưu "ảnh chụp" của state. Khi load aggregate, thay vì replay từ event đầu tiên, load snapshot gần nhất rồi chỉ replay các events sau snapshot. Ví dụ: 10,000 events, snapshot tại event 9,950 → chỉ cần replay 50 events.

---

**Cập Nhật Lần Cuối:** 2026-05-18  
**Tags:** event-sourcing, event-store, projection, snapshot, aggregate, DynamoDB, Kinesis
