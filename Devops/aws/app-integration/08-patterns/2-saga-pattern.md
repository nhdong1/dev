# Saga Pattern — Quản Lý Giao Dịch Phân Tán

> Saga Pattern là giải pháp quản lý distributed transactions (giao dịch phân tán) qua nhiều microservice mà không cần 2PC (Two-Phase Commit — Cam Kết Hai Giai Đoạn).

---

## 📚 Mục Lục

1. [Vấn Đề Cần Giải Quyết](#vấn-đề-cần-giải-quyết)
2. [Saga Pattern Là Gì?](#saga-pattern-là-gì)
3. [Choreography vs Orchestration](#choreography-vs-orchestration)
4. [Choreography — Kiến Trúc Vũ Đạo](#choreography--kiến-trúc-vũ-đạo)
5. [Orchestration — Kiến Trúc Điều Phối](#orchestration--kiến-trúc-điều-phối)
6. [So Sánh Hai Phương Pháp](#so-sánh-hai-phương-pháp)
7. [Compensating Transactions — Giao Dịch Bù Trừ](#compensating-transactions--giao-dịch-bù-trừ)
8. [Triển Khai Trên AWS](#triển-khai-trên-aws)
9. [Ví Dụ Code Thực Tế](#ví-dụ-code-thực-tế)
10. [Cạm Bẫy Thường Gặp](#cạm-bẫy-thường-gặp)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Vấn Đề Cần Giải Quyết

### Giao Dịch Trong Monolith Truyền Thống

Trong monolith (ứng dụng nguyên khối), ACID transaction (giao dịch ACID) chạy trong một database duy nhất:

```sql
BEGIN TRANSACTION;
  UPDATE inventory SET stock = stock - 1 WHERE product_id = 123;
  INSERT INTO orders (customer_id, product_id) VALUES (456, 123);
  UPDATE payment_accounts SET balance = balance - 100 WHERE customer_id = 456;
COMMIT; -- Tất cả hoặc không có gì
```

### Vấn Đề Trong Microservices

Mỗi service có database riêng — không thể dùng ACID transaction liên service:

```
Order Service (PostgreSQL)  →  Inventory Service (MySQL)  →  Payment Service (DynamoDB)
       ↓                              ↓                              ↓
  orders table              inventory table                accounts table
```

**Nếu Payment thất bại sau khi Inventory đã giảm:**

```
1. Order created ✅
2. Inventory decreased ✅  ← Đã xảy ra
3. Payment failed ❌       ← Lỗi ở đây

→ Hệ thống không nhất quán: inventory đã giảm nhưng không có đơn hàng hợp lệ
```

### Tại Sao Không Dùng 2PC (Two-Phase Commit)?

2PC giải quyết bài toán này nhưng có vấn đề lớn:

- **Blocking protocol (Giao thức chặn):** Tất cả participant phải chờ coordinator
- **Single point of failure (Điểm lỗi đơn):** Nếu coordinator chết, toàn hệ thống bị block
- **Không phù hợp microservices:** Mỗi service cần implement 2PC protocol
- **Performance kém:** Latency cao vì cần 2 round-trip giữa tất cả service

---

## Saga Pattern Là Gì?

**Saga** là chuỗi các local transaction (giao dịch cục bộ), mỗi transaction publish event hoặc gửi message để trigger transaction tiếp theo. Nếu một bước thất bại, saga chạy **compensating transactions (giao dịch bù trừ)** để undo (hoàn tác) các bước đã thành công trước đó.

```
T1 → T2 → T3 → T4  (happy path — đường đi thuận lợi)
              ↓
           FAILURE
              ↓
C3 → C2 → C1       (compensation — bù trừ ngược lại)
```

Trong đó:
- **Tn** = Local transaction thứ n
- **Cn** = Compensating transaction (bù trừ) cho Tn

**Tính nhất quán của Saga:**
- **Eventual consistency (Nhất quán cuối cùng)** — không phải strong consistency (nhất quán tuyệt đối)
- Trong quá trình chạy, hệ thống có thể ở trạng thái trung gian
- Sau khi saga hoàn thành (thành công hoặc bù trừ xong), hệ thống nhất quán trở lại

---

## Choreography vs Orchestration

Có hai cách triển khai Saga Pattern:

| | Choreography (Vũ Đạo) | Orchestration (Điều Phối) |
|---|---|---|
| **Ai điều khiển?** | Các service tự điều phối qua events | Một orchestrator trung tâm |
| **Giao tiếp** | Event-based (dựa trên sự kiện) | Command-based (dựa trên lệnh) |
| **Coupling (Ghép nối)** | Loose coupling (ghép nối lỏng) | Tập trung ở orchestrator |
| **Khả năng debug** | Khó — phải trace qua nhiều service | Dễ — state tập trung |
| **Phức tạp** | Tăng khi thêm service | Orchestrator phức tạp hơn |

---

## Choreography — Kiến Trúc Vũ Đạo

Mỗi service lắng nghe event và publish event mới — không có trung tâm điều phối.

### Ví Dụ: Order Processing Flow

```
                    ┌─────────────────────────────────┐
                    │         EventBridge              │
                    │     (Event Bus — Xe Buýt)        │
                    └────────────┬────────────────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
        ▼                        ▼                        ▼
┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│ Order Service │        │  Inventory   │        │   Payment    │
│              │        │   Service    │        │   Service    │
│ 1. Create    │        │              │        │              │
│    order     │        │ 2. Reserve   │        │ 3. Process   │
│              │ ──────►│    stock     │ ──────►│    payment   │
│ Listen:      │        │              │        │              │
│ payment.done │        │ Listen:      │        │ Listen:      │
│              │        │ order.placed │        │ stock.reserved│
└──────────────┘        └──────────────┘        └──────────────┘

Events flow:
1. Order Service → publishes: order.placed
2. Inventory Service → listens: order.placed → publishes: stock.reserved
3. Payment Service → listens: stock.reserved → publishes: payment.completed
4. Order Service → listens: payment.completed → publishes: order.confirmed
```

### Compensation trong Choreography

```
Bước 3 (Payment) thất bại:
Payment Service → publishes: payment.failed

Inventory Service → listens: payment.failed → RELEASES stock (trả lại hàng)
                  → publishes: stock.released

Order Service → listens: stock.released → CANCELS order (hủy đơn)
             → publishes: order.cancelled
```

### Code Ví Dụ: Choreography với EventBridge

```python
# inventory_service.py

import boto3
import json

eventbridge = boto3.client('events')

def handle_order_placed(event):
    order_id = event['detail']['order_id']
    product_id = event['detail']['product_id']
    quantity = event['detail']['quantity']

    # Thực hiện local transaction
    success = reserve_stock(product_id, quantity)

    if success:
        # Publish success event (sự kiện thành công)
        publish_event('stock.reserved', {
            'order_id': order_id,
            'product_id': product_id,
            'quantity': quantity
        })
    else:
        # Publish failure event (sự kiện thất bại) → trigger compensation
        publish_event('stock.reservation.failed', {
            'order_id': order_id,
            'reason': 'INSUFFICIENT_STOCK'
        })

def publish_event(event_type, detail):
    eventbridge.put_events(Entries=[{
        'Source': 'inventory-service',
        'DetailType': event_type,
        'Detail': json.dumps(detail),
        'EventBusName': 'order-processing-bus'
    }])
```

---

## Orchestration — Kiến Trúc Điều Phối

Một orchestrator trung tâm (thường là AWS Step Functions) điều phối toàn bộ saga — biết trạng thái hiện tại, quyết định bước tiếp theo.

### Ví Dụ: Order Processing với Step Functions

```json
{
  "Comment": "Order Processing Saga",
  "StartAt": "CreateOrder",
  "States": {
    "CreateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123:function:create-order",
      "Next": "ReserveInventory",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "OrderFailed"
      }]
    },
    "ReserveInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123:function:reserve-inventory",
      "Next": "ProcessPayment",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "CancelOrder"
      }]
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123:function:process-payment",
      "Next": "ConfirmOrder",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "ReleaseInventory"
      }]
    },
    "ConfirmOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123:function:confirm-order",
      "End": true
    },
    "ReleaseInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123:function:release-inventory",
      "Next": "CancelOrder"
    },
    "CancelOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123:function:cancel-order",
      "Next": "OrderFailed"
    },
    "OrderFailed": {
      "Type": "Fail",
      "Error": "OrderProcessingFailed",
      "Cause": "Saga compensation completed"
    }
  }
}
```

### Flow Trực Quan

```
Start
  │
  ▼
CreateOrder ──FAIL──► OrderFailed
  │
  ▼
ReserveInventory ──FAIL──► CancelOrder → OrderFailed
  │
  ▼
ProcessPayment ──FAIL──► ReleaseInventory → CancelOrder → OrderFailed
  │
  ▼
ConfirmOrder
  │
  ▼
End (Success)
```

---

## So Sánh Hai Phương Pháp

### Choreography — Ưu Điểm

```
✅ Loose coupling — service không biết nhau
✅ Dễ thêm service mới (chỉ cần subscribe thêm event)
✅ Không có single point of failure
✅ Phù hợp simple saga với ít bước
```

### Choreography — Nhược Điểm

```
❌ Khó debug — phải trace event qua nhiều service
❌ Khó hiểu toàn bộ flow (không có diagram tổng quan)
❌ Cyclic dependency (phụ thuộc vòng) có thể xảy ra
❌ Khó đảm bảo tất cả compensation đã chạy
❌ Testing phức tạp
```

### Orchestration — Ưu Điểm

```
✅ Toàn bộ flow rõ ràng, dễ visualize (Step Functions có diagram)
✅ Dễ debug — execution history (lịch sử thực thi) đầy đủ
✅ Dễ đảm bảo compensation chạy đúng
✅ Phù hợp complex saga nhiều bước và điều kiện
✅ Tích hợp retry và error handling tự động
```

### Orchestration — Nhược Điểm

```
❌ Orchestrator là central component — tăng coupling
❌ Orchestrator có thể trở thành bottleneck
❌ Chi phí Step Functions Standard Workflow (theo transition)
❌ Cần deploy và maintain thêm orchestrator service
```

### Khi Nào Chọn Cái Nào?

```
Choreography phù hợp khi:
- Saga đơn giản, ít bước (2–4 bước)
- Teams độc lập, service boundaries rõ ràng
- Không muốn central dependency
- Event-driven culture đã có sẵn

Orchestration phù hợp khi:
- Saga phức tạp, nhiều bước (5+ bước)
- Cần visibility tốt (audit, debug)
- Complex compensation logic
- Cần timeout, retry phức tạp
```

---

## Compensating Transactions — Giao Dịch Bù Trừ

### Nguyên Tắc Thiết Kế

**Compensating transaction phải:**
1. **Idempotent (bất biến):** Có thể chạy nhiều lần, kết quả như nhau
2. **Không thể fail:** Nếu compensation fail → hệ thống không bao giờ nhất quán
3. **Không giống undo:** Compensation là forward action (hành động tiến), không phải rollback thực sự

### Ví Dụ Compensation

| Transaction | Compensating Transaction |
|---|---|
| Reserve inventory (giữ hàng) | Release inventory (trả lại hàng) |
| Create order (tạo đơn) | Cancel order (hủy đơn) |
| Charge payment (tính phí) | Refund payment (hoàn tiền) |
| Book flight (đặt chỗ bay) | Cancel booking (hủy đặt chỗ) |
| Send email | Không thể undo — log "cancellation email sent" |

### Các Loại Trạng Thái Trong Saga

```
Pending (Đang Chờ) → một bước đang thực hiện
Completed (Hoàn Thành) → bước thành công
Compensating (Đang Bù Trừ) → đang chạy compensation
Compensated (Đã Bù Trừ) → compensation hoàn thành
Failed (Thất Bại) → không thể compensation
```

---

## Triển Khai Trên AWS

### Choreography với EventBridge + SQS

```
Architecture:
- EventBridge Custom Bus: nhận events từ tất cả service
- SQS Queue per service: buffer events, retry, DLQ
- EventBridge Rule: route đúng event đến đúng queue
- Lambda per service: xử lý event và publish event tiếp theo

Ưu điểm: 
- SQS đảm bảo at-least-once delivery
- DLQ bắt compensation events thất bại
- EventBridge Archive cho replay nếu cần
```

### Orchestration với Step Functions

```
Architecture:
- Step Functions Standard Workflow: dài hạn, durable execution
- Lambda cho mỗi bước: thực hiện local transaction
- DynamoDB: lưu saga state (nếu cần)
- EventBridge để trigger Step Functions

Khi nào dùng Standard vs Express:
- Standard: saga dài hạn, cần durable (bền), audit trail
- Express: saga ngắn (<5 phút), high volume, chi phí thấp hơn
```

### Best Practice — Lưu Saga State

```python
# Lưu correlation ID (định danh tương quan) để trace toàn bộ saga

import boto3
import uuid

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('saga-executions')

def start_saga(order_data):
    saga_id = str(uuid.uuid4())

    # Lưu saga state
    table.put_item(Item={
        'saga_id': saga_id,
        'status': 'STARTED',
        'created_at': datetime.utcnow().isoformat(),
        'order_id': order_data['order_id'],
        'steps_completed': [],
        'current_step': 'CreateOrder'
    })

    return saga_id
```

---

## Ví Dụ Code Thực Tế

### Lambda Handler: Reserve Inventory (Idempotent)

```python
import boto3
import json

dynamodb = boto3.resource('dynamodb')
inventory_table = dynamodb.Table('inventory')
saga_table = dynamodb.Table('saga-executions')

def lambda_handler(event, context):
    saga_id = event['saga_id']
    order_id = event['order_id']
    product_id = event['product_id']
    quantity = event['quantity']

    # Idempotency check (kiểm tra bất biến)
    # Nếu đã reserve rồi → trả về kết quả cũ
    existing = saga_table.get_item(Key={'saga_id': saga_id})
    if 'Item' in existing:
        steps = existing['Item'].get('steps_completed', [])
        if 'ReserveInventory' in steps:
            return {'status': 'ALREADY_RESERVED', 'saga_id': saga_id}

    # Thực hiện local transaction
    try:
        response = inventory_table.update_item(
            Key={'product_id': product_id},
            UpdateExpression='SET reserved = reserved + :qty, available = available - :qty',
            ConditionExpression='available >= :qty',
            ExpressionAttributeValues={':qty': quantity},
            ReturnValues='UPDATED_NEW'
        )

        # Cập nhật saga state
        saga_table.update_item(
            Key={'saga_id': saga_id},
            UpdateExpression='SET steps_completed = list_append(steps_completed, :step)',
            ExpressionAttributeValues={':step': ['ReserveInventory']}
        )

        return {'status': 'SUCCESS', 'saga_id': saga_id}

    except dynamodb.meta.client.exceptions.ConditionalCheckFailedException:
        raise Exception('INSUFFICIENT_STOCK')
```

---

## Cạm Bẫy Thường Gặp

### 1. Forget Compensation (Quên Bù Trừ)

```
❌ Sai: Chỉ xử lý happy path, không thiết kế compensation
✅ Đúng: Thiết kế compensation ngay khi thiết kế transaction
```

### 2. Non-idempotent Compensation

```
❌ Sai: Compensation gọi external API có thể duplicate charge
✅ Đúng: Idempotency key trong mọi API call, kiểm tra trạng thái trước khi hành động
```

### 3. Missing Correlation ID (Thiếu Định Danh Tương Quan)

```
❌ Sai: Events không có saga_id → không thể trace toàn bộ flow
✅ Đúng: Mọi event và log phải có saga_id / correlation_id
```

### 4. Long-running Saga Không Có Timeout

```
❌ Sai: Saga chờ mãi mãi nếu một service không bao giờ respond
✅ Đúng: Đặt timeout cho mỗi bước và toàn bộ saga
         Step Functions: HeartbeatSeconds, TimeoutSeconds
```

### 5. Không Xử Lý Partial Failure Của Compensation

```
❌ Sai: Compensation thất bại → hệ thống kẹt trong trạng thái không nhất quán
✅ Đúng:
   - Compensation phải không thể fail (idempotent, retry vô hạn)
   - Alert và manual intervention (can thiệp thủ công) cho edge cases
   - "Saga Admin" để manually complete hoặc rollback
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Saga Pattern khác 2PC như thế nào?

**Trả lời:**
> 2PC (Two-Phase Commit) sử dụng global lock (khóa toàn cục) — tất cả participant phải lock resource đến khi commit, gây blocking và single point of failure ở coordinator.
>
> Saga không dùng global transaction — thay vào đó là chuỗi local transaction + compensating transaction. Mỗi service tự commit local transaction ngay lập tức, không block nhau.
>
> Đánh đổi: Saga chỉ đảm bảo eventual consistency (nhất quán cuối cùng), không phải strong consistency như 2PC. Trong quá trình chạy saga, hệ thống có thể ở trạng thái trung gian — cần thiết kế business logic chấp nhận điều này.

### Câu 2: Khi nào chọn Choreography, khi nào chọn Orchestration?

**Trả lời:**
> Choreography phù hợp khi saga đơn giản (ít bước), các team/service hoàn toàn độc lập, và muốn tránh central dependency. Nhược điểm là khó debug khi có lỗi.
>
> Orchestration (Step Functions) phù hợp khi saga phức tạp, cần visibility rõ ràng, có điều kiện phân nhánh phức tạp, hoặc cần audit trail đầy đủ. Step Functions tự động lưu execution history — debug rất dễ.
>
> Trong thực tế tôi thấy orchestration thường được ưu tiên cho production vì dễ maintain và debug hơn.

### Câu 3: Compensating transaction phải thỏa mãn điều kiện gì?

**Trả lời:**
> Compensating transaction phải idempotent — có thể chạy nhiều lần mà kết quả giống nhau. Lý do: nếu compensation fail và retry, không được tạo ra side effect (tác dụng phụ) thêm.
>
> Ngoài ra, compensation phải được thiết kế để không bao giờ fail — vì nếu compensation cũng fail, hệ thống kẹt mãi trong trạng thái không nhất quán. Trong thực tế điều này không phải lúc nào cũng đảm bảo được, nên cần có monitoring, alerting, và quy trình manual intervention.

---

**Cập Nhật Lần Cuối:** 2026-05-18  
**Tags:** saga, choreography, orchestration, distributed-transaction, step-functions, eventbridge
