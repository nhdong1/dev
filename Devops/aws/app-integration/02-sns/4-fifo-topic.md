# FIFO Topic — Thứ Tự & Exactly-Once Delivery

> SNS FIFO Topic (First-In, First-Out — Vào Trước Ra Trước) đảm bảo tin nhắn được phát và nhận **theo đúng thứ tự** và **chính xác một lần** — dành cho các use case yêu cầu tính nhất quán cao.

## 📌 Tóm Tắt Nhanh

| Đặc Điểm | Standard Topic | FIFO Topic |
|---|---|---|
| **Thứ tự** | Best-effort (nỗ lực tốt nhất) | Đảm bảo FIFO chặt chẽ |
| **Throughput** (Thông Lượng) | 30 triệu msg/s | 300 msg/s (3.000 với batching) |
| **Deduplication** (Khử Trùng) | Không | ✅ Trong 5 phút |
| **Subscriber** | SQS, Lambda, HTTP, Email, SMS | **Chỉ SQS FIFO Queue** |
| **Tên topic** | Tùy ý | Phải kết thúc `.fifo` |
| **Message Group** | Không có | ✅ Bắt buộc |
| **Filter Policy** | ✅ | ✅ |

---

## 🎯 Khi Nào Cần FIFO Topic?

### Bài Toán Cần Thứ Tự

```
Standard Topic — Không đảm bảo thứ tự:
──────────────────────────────────────
Publisher gửi: [CREATE] → [UPDATE price=$100] → [CANCEL]

Consumer có thể nhận:
  [CANCEL] → [CREATE] → [UPDATE price=$100]  ← Hủy đơn chưa tồn tại!
  [UPDATE] → [CANCEL] → [CREATE]             ← Sai hoàn toàn!

FIFO Topic — Đảm bảo thứ tự:
─────────────────────────────
Publisher gửi: [CREATE] → [UPDATE price=$100] → [CANCEL]
Consumer nhận: [CREATE] → [UPDATE price=$100] → [CANCEL]  ← Luôn đúng thứ tự
```

### Bài Toán Exactly-Once (Đúng Một Lần)

```
Standard Topic — At-least-once:
──────────────────────────────────
Nếu publisher retry do timeout: [DEBIT $1000] được gửi 2 lần
Consumer xử lý 2 lần → tài khoản bị trừ $2000 ← Nghiêm trọng!

FIFO Topic — Exactly-once:
───────────────────────────
Publisher gửi [DEBIT $1000] với MessageDeduplicationId = "TXN-001"
Nếu retry với cùng ID → SNS nhận ra duplicate, bỏ qua
Consumer xử lý đúng 1 lần → tài khoản trừ $1000 ← Chính xác
```

---

## 🏗️ Kiến Trúc FIFO Topic

```
Publisher
    │ Publish với:
    │  - MessageGroupId (bắt buộc)
    │  - MessageDeduplicationId (hoặc ContentBasedDeduplication)
    ▼
[SNS FIFO Topic: order-events.fifo]
    │
    │ ← Chỉ hỗ trợ SQS FIFO subscriber
    │
    ├──▶ [SQS FIFO: payment.fifo]      ← Xử lý thanh toán theo thứ tự
    │         └──▶ [Payment Consumer]
    │
    └──▶ [SQS FIFO: inventory.fifo]    ← Cập nhật kho theo thứ tự
              └──▶ [Inventory Consumer]
```

### Tại Sao FIFO Topic Chỉ Hỗ Trợ SQS FIFO?

Để đảm bảo ordering (thứ tự) end-to-end (đầu cuối), **toàn bộ đường đi** phải hỗ trợ FIFO:

```
[SNS FIFO] ──▶ [SQS FIFO] ──▶ [Consumer]
   ↑                ↑               ↑
Giữ thứ tự      Giữ thứ tự    Xử lý theo
khi publish     trong queue   thứ tự nhận
```

Nếu dùng SQS Standard thay cho FIFO: SQS có thể đảo thứ tự → đảm bảo của SNS FIFO bị phá vỡ.

---

## ⚙️ Cấu Hình FIFO Topic

### Tạo FIFO Topic

```bash
# Tạo FIFO topic
aws sns create-topic \
  --name order-events.fifo \
  --attributes '{
    "FifoTopic": "true",
    "ContentBasedDeduplication": "false"
  }'

# Tạo FIFO topic với Content-Based Deduplication (tự tính hash từ message body)
aws sns create-topic \
  --name audit-log.fifo \
  --attributes '{
    "FifoTopic": "true",
    "ContentBasedDeduplication": "true"
  }'
```

### Tạo SQS FIFO Queue Và Subscribe

```bash
FIFO_TOPIC_ARN="arn:aws:sns:us-east-1:123456:order-events.fifo"

# Tạo SQS FIFO queue
aws sqs create-queue \
  --queue-name payment.fifo \
  --attributes '{
    "FifoQueue": "true",
    "ContentBasedDeduplication": "false"
  }'

PAYMENT_QUEUE_ARN=$(aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456/payment.fifo \
  --attribute-names QueueArn \
  --query "Attributes.QueueArn" --output text)

# Subscribe SQS FIFO queue vào SNS FIFO topic
aws sns subscribe \
  --topic-arn $FIFO_TOPIC_ARN \
  --protocol sqs \
  --notification-endpoint $PAYMENT_QUEUE_ARN \
  --attributes '{"RawMessageDelivery":"true"}'
```

---

## 🔑 Message Group ID — Kiểm Soát Thứ Tự

**MessageGroupId** (ID Nhóm Tin Nhắn) là cơ chế quan trọng nhất trong FIFO:

```
FIFO Topic: order-events.fifo
│
├── Group "order-ORD-001": [CREATE] → [PAY] → [SHIP] → [DELIVER]
│       ↑ Thứ tự đảm bảo trong cùng group
│
├── Group "order-ORD-002": [CREATE] → [PAY] → [CANCEL]
│       ↑ Xử lý song song với ORD-001
│
└── Group "order-ORD-003": [CREATE] → [PAY]
        ↑ Xử lý song song với ORD-001, ORD-002
```

**Ý nghĩa:**
- **Trong cùng một group**: thứ tự FIFO nghiêm ngặt — tin gửi trước nhận trước
- **Giữa các group khác nhau**: xử lý song song hoàn toàn

### Chiến Lược Chọn MessageGroupId

| Use Case | MessageGroupId Nên Là |
|---|---|
| Order processing per customer | `order-{orderId}` hoặc `customer-{customerId}` |
| Bank transactions per account | `account-{accountId}` |
| Inventory per product | `product-{productId}` |
| User action sequences | `user-{userId}` |
| Global single sequence | Một giá trị cố định (tất cả vào một group, không song song) |

---

## 🔄 Deduplication — Khử Trùng Tin Nhắn

### Cách 1: MessageDeduplicationId (Do Producer Cung Cấp)

```python
import boto3
import json

sns = boto3.client("sns", region_name="us-east-1")
FIFO_TOPIC_ARN = "arn:aws:sns:us-east-1:123456:order-events.fifo"

def publish_order_event_fifo(
    order_id: str,
    event_type: str,
    payload: dict,
    sequence: int   # Số thứ tự của event trong order lifecycle
) -> str:
    """
    Publish vào FIFO topic với deduplication.

    MessageGroupId = order_id: đảm bảo thứ tự events của cùng đơn hàng
    MessageDeduplicationId: ngăn publish trùng nếu producer retry
    """
    dedup_id = f"{order_id}-{event_type}-{sequence}"

    response = sns.publish(
        TopicArn=FIFO_TOPIC_ARN,
        Message=json.dumps({
            **payload,
            "orderId": order_id,
            "eventType": event_type,
            "sequence": sequence
        }),
        MessageGroupId=f"order-{order_id}",     # Nhóm theo đơn hàng
        MessageDeduplicationId=dedup_id         # Unique per event
    )

    print(f"Published seq={sequence} for order {order_id}: {response['MessageId']}")
    print(f"SequenceNumber: {response['SequenceNumber']}")
    return response["MessageId"]

# Publish lifecycle của đơn hàng ORD-001 theo thứ tự
publish_order_event_fifo("ORD-001", "ORDER_CREATED",  {"amount": 100}, sequence=1)
publish_order_event_fifo("ORD-001", "PAYMENT_CHARGED",{"chargeId": "CHG-001"}, sequence=2)
publish_order_event_fifo("ORD-001", "ORDER_SHIPPED",  {"trackingId": "TRK-001"}, sequence=3)
```

### Cách 2: Content-Based Deduplication (Tự Động)

```python
# Bật ContentBasedDeduplication trên topic
# SNS tự tính SHA-256 của message body làm deduplication ID
# Không cần cung cấp MessageDeduplicationId

sns.publish(
    TopicArn="arn:aws:sns:us-east-1:123456:audit-log.fifo",
    Message=json.dumps({
        "eventId": "EVT-001",       # Phải unique — body làm dedup key
        "action": "USER_LOGIN",
        "userId": "U-123",
        "timestamp": "2026-05-18T10:00:00Z"
    }),
    MessageGroupId="user-U-123"
    # Không cần MessageDeduplicationId
)
```

**Khi nào dùng từng cách:**

| Cách | Dùng Khi |
|---|---|
| **MessageDeduplicationId** | Producer kiểm soát được ID, cần dedup logic chính xác hơn |
| **Content-Based Deduplication** | Message body đã chứa unique ID (như `eventId`); muốn đơn giản |

---

## 🏭 Ví Dụ Thực Tế — Bank Transaction System

### Bài Toán

Hệ thống ngân hàng cần xử lý các giao dịch theo **đúng thứ tự** cho mỗi tài khoản:

```
Tài khoản ACC-001: [DEPOSIT $5000] → [WITHDRAW $2000] → [WITHDRAW $1000]

Nếu xử lý sai thứ tự:
  [WITHDRAW $2000] trước [DEPOSIT $5000] → Số dư âm → Giao dịch thất bại!

Nếu xử lý đúng thứ tự:
  [DEPOSIT $5000] → số dư = $5000
  [WITHDRAW $2000] → số dư = $3000
  [WITHDRAW $1000] → số dư = $2000  ← Đúng!
```

### Triển Khai

```python
import boto3
import json
import uuid
from datetime import datetime
from enum import Enum

class TransactionType(str, Enum):
    DEPOSIT = "DEPOSIT"
    WITHDRAW = "WITHDRAW"
    TRANSFER_OUT = "TRANSFER_OUT"
    TRANSFER_IN = "TRANSFER_IN"

class BankTransactionPublisher:
    """Publisher cho sự kiện giao dịch ngân hàng qua SNS FIFO."""

    def __init__(self, topic_arn: str):
        self.sns = boto3.client("sns", region_name="us-east-1")
        self.topic_arn = topic_arn

    def publish_transaction(
        self,
        account_id: str,
        transaction_id: str,
        txn_type: TransactionType,
        amount: float
    ) -> str:
        """
        Publish giao dịch vào FIFO topic.
        MessageGroupId = account_id: đảm bảo thứ tự giao dịch per tài khoản.
        MessageDeduplicationId = transaction_id: tránh xử lý giao dịch trùng.
        """
        payload = {
            "transactionId": transaction_id,
            "accountId": account_id,
            "type": txn_type,
            "amount": amount,
            "timestamp": datetime.utcnow().isoformat()
        }

        response = self.sns.publish(
            TopicArn=self.topic_arn,
            Message=json.dumps(payload),
            MessageGroupId=f"account-{account_id}",    # Thứ tự per tài khoản
            MessageDeduplicationId=transaction_id       # Unique per giao dịch
        )

        return response["MessageId"]

class BankTransactionConsumer:
    """Consumer xử lý giao dịch từ SQS FIFO queue."""

    def __init__(self, queue_url: str):
        self.sqs = boto3.client("sqs", region_name="us-east-1")
        self.queue_url = queue_url
        self.account_balances = {}  # Trong thực tế: dùng DynamoDB

    def run(self):
        print("Bank Transaction Consumer đang chạy...")

        while True:
            response = self.sqs.receive_message(
                QueueUrl=self.queue_url,
                MaxNumberOfMessages=1,   # FIFO: chỉ lấy 1 per lần để giữ thứ tự
                WaitTimeSeconds=20
            )

            for message in response.get("Messages", []):
                receipt = message["ReceiptHandle"]
                payload = json.loads(message["Body"])

                try:
                    self._process_transaction(payload)

                    self.sqs.delete_message(
                        QueueUrl=self.queue_url,
                        ReceiptHandle=receipt
                    )
                except Exception as e:
                    print(f"Transaction failed: {e}")
                    # Không xóa → retry, sau N lần → DLQ

    def _process_transaction(self, payload: dict):
        account_id = payload["accountId"]
        txn_type = payload["type"]
        amount = payload["amount"]

        balance = self.account_balances.get(account_id, 0)

        if txn_type in (TransactionType.DEPOSIT, TransactionType.TRANSFER_IN):
            new_balance = balance + amount
        elif txn_type in (TransactionType.WITHDRAW, TransactionType.TRANSFER_OUT):
            if balance < amount:
                raise ValueError(f"Insufficient funds: balance={balance}, amount={amount}")
            new_balance = balance - amount
        else:
            raise ValueError(f"Unknown transaction type: {txn_type}")

        self.account_balances[account_id] = new_balance
        print(f"Account {account_id}: {balance} → {new_balance} ({txn_type} {amount})")
```

---

## ⚡ Throughput FIFO Topic

| Mode | Throughput |
|---|---|
| Standard | 300 msg/s per topic |
| Batching (10 msg/batch) | 3.000 msg/s per topic |
| Multiple Message Groups | Tăng parallelism nhưng không tăng tổng throughput |

**So sánh với Standard Topic:**

```
Standard Topic:   ████████████████████████████  30.000.000 msg/s
FIFO Topic:       ████  300 msg/s (không batch)
FIFO (batch):     ████████  3.000 msg/s
```

**Khi nào throughput 300 msg/s là đủ?**

- Hầu hết banking và financial transaction systems
- Order management cho mid-size e-commerce
- Audit logging hệ thống nội bộ
- State machine event sourcing

---

## 🔧 Cấu Hình Nâng Cao

### Kết Hợp Filter Policy Với FIFO

FIFO Topic vẫn hỗ trợ Message Filtering:

```python
# Subscribe với filter: chỉ nhận DEPOSIT và TRANSFER_IN
sns.subscribe(
    TopicArn="arn:aws:sns:us-east-1:123456:bank-events.fifo",
    Protocol="sqs",
    Endpoint="arn:aws:sqs:us-east-1:123456:credit-events.fifo",
    Attributes={
        "RawMessageDelivery": "true",
        "FilterPolicy": json.dumps({
            "transactionType": ["DEPOSIT", "TRANSFER_IN"]
        })
    }
)

# Chỉ nhận WITHDRAW và TRANSFER_OUT
sns.subscribe(
    TopicArn="arn:aws:sns:us-east-1:123456:bank-events.fifo",
    Protocol="sqs",
    Endpoint="arn:aws:sqs:us-east-1:123456:debit-events.fifo",
    Attributes={
        "RawMessageDelivery": "true",
        "FilterPolicy": json.dumps({
            "transactionType": ["WITHDRAW", "TRANSFER_OUT"]
        })
    }
)
```

**Lưu ý:** Filter dựa trên MessageAttributes, không phải MessageGroupId.

### SequenceNumber (Số Thứ Tự)

SNS FIFO trả về `SequenceNumber` khi publish — dùng để verify thứ tự:

```python
response = sns.publish(
    TopicArn=fifo_topic_arn,
    Message=json.dumps(payload),
    MessageGroupId="order-ORD-001",
    MessageDeduplicationId="ORD-001-created-001"
)

sequence_number = response["SequenceNumber"]
message_id = response["MessageId"]
print(f"MessageId: {message_id}, SequenceNumber: {sequence_number}")
# SequenceNumber là monotonically increasing (tăng đơn điệu) trong cùng MessageGroup
```

---

## 📊 FIFO Topic vs Standard Topic — Khi Nào Dùng Gì?

| Tiêu Chí | Standard Topic | FIFO Topic |
|---|---|---|
| **Thứ tự quan trọng?** | Không | ✅ Có |
| **Exactly-once cần thiết?** | Không | ✅ Có |
| **Throughput cao?** | ✅ Không giới hạn | Giới hạn 300/3.000 msg/s |
| **Subscriber đa dạng?** | ✅ SQS, Lambda, Email, SMS... | Chỉ SQS FIFO |
| **Chi phí** | Thấp hơn | Cao hơn một chút |

### Decision Tree (Cây Quyết Định)

```
Bắt đầu
  │
  ├── Cần thứ tự nghiêm ngặt? ──[Có]──▶ Cần throughput > 3.000 msg/s?
  │                                          │
  │                                  [Có]──▶ Xem xét lại: có thể partition
  │                                  │       theo nhiều topic/group
  │                                  │
  │                                  [Không]──▶ Dùng SNS FIFO Topic
  │
  └──[Không]──▶ Cần exactly-once? ──[Có]──▶ Dùng SNS FIFO Topic
                                  │
                                  [Không]──▶ Dùng SNS Standard Topic
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

### Q: SNS FIFO Topic khác Standard Topic ở điểm nào quan trọng nhất?

**A:** Ba điểm chính: (1) **Thứ tự đảm bảo** — tin gửi trước nhận trước trong cùng MessageGroup; (2) **Exactly-once** — dùng deduplication ID để tránh xử lý trùng; (3) **Chỉ hỗ trợ SQS FIFO** subscriber — không hỗ trợ Lambda, Email, SMS trực tiếp.

### Q: MessageGroupId quyết định điều gì?

**A:** Quyết định phạm vi thứ tự. Các tin nhắn **cùng group** được xử lý theo FIFO chặt chẽ. Các group **khác nhau** xử lý song song. Đây là cơ chế tăng parallelism (tính song song) mà vẫn giữ thứ tự.

### Q: Nếu publisher fail và retry, FIFO Topic xử lý thế nào?

**A:** Nếu publisher gửi lại **cùng MessageDeduplicationId**, SNS nhận ra duplicate và **bỏ qua** — không thêm vào queue. Deduplication window (cửa sổ khử trùng) là 5 phút. Đây chính là cơ chế exactly-once.

### Q: SNS FIFO vs SNS Standard — chọn khi nào?

**A:** Dùng **FIFO** khi thứ tự ảnh hưởng đến tính đúng đắn (financial transactions, state machine events, audit logs). Dùng **Standard** khi cần throughput cao, subscriber đa dạng (Email, SMS, Lambda), hoặc thứ tự không quan trọng (notifications, analytics).

### Q: Throughput 300 msg/s có thể tăng không?

**A:** 300 msg/s là giới hạn của SNS FIFO. Để tăng parallelism, dùng **nhiều MessageGroupId** — mỗi group xử lý song song. Ví dụ: thay vì một group "all-transactions", dùng group per account-id → nhiều account xử lý song song, tổng throughput tăng. Batching (10 msg/batch) có thể đạt 3.000 msg/s.

---

## 🗺️ Điều Hướng

- ◀ Quay lại: [3-fanout-pattern.md](./3-fanout-pattern.md) — Fan-out Pattern
- ▶ Module tiếp theo: [../03-eventbridge/README.md](../03-eventbridge/README.md) — Amazon EventBridge
- 📋 Tổng quan SNS: [README.md](./README.md)
