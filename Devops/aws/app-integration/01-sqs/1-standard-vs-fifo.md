# Standard Queue vs FIFO Queue — So Sánh Chi Tiết

> Hiểu rõ sự khác biệt giữa Standard Queue và FIFO Queue (First-In, First-Out — Vào Trước Ra Trước) là câu hỏi phỏng vấn phổ biến nhất về SQS.

## 📌 Tóm Tắt Nhanh

| Tiêu Chí | Standard Queue | FIFO Queue |
|---|---|---|
| **Thứ tự tin nhắn** | Best-effort (nỗ lực tốt nhất, không đảm bảo) | Đảm bảo FIFO chặt chẽ |
| **Throughput** (Thông Lượng) | **Không giới hạn** | 300 msg/s; 3.000 msg/s với batching |
| **Delivery Guarantee** (Đảm Bảo Giao Vận) | At-least-once (ít nhất 1 lần) | Exactly-once (đúng 1 lần) |
| **Duplicate** (Tin Nhắn Trùng) | Có thể xảy ra | Deduplication tự động |
| **Tên Queue** | Bất kỳ | Phải kết thúc bằng `.fifo` |
| **Giá** | Thấp hơn | Cao hơn ~5% |
| **Message Group** | Không có | Hỗ trợ Message Group ID |

---

## 🔵 Standard Queue — Hàng Đợi Tiêu Chuẩn

### Đặc Điểm Cốt Lõi

**1. At-Least-Once Delivery (Giao Vận Ít Nhất Một Lần)**

SQS Standard **không đảm bảo** rằng mỗi tin nhắn chỉ được giao đúng 1 lần. Tin nhắn có thể được consumer nhận **nhiều hơn 1 lần** do:

- Lưu trữ phân tán (distributed storage) trên nhiều AZ
- Khi consumer 1 nhận message nhưng chưa xóa và visibility timeout hết → consumer 2 cũng nhận được
- Lỗi mạng tạm thời

```
Consumer A nhận message M1 → bắt đầu xử lý
    │
    ├── (Xử lý chậm, Visibility Timeout hết)
    │
Consumer B cũng nhận được M1 → cũng bắt đầu xử lý ← TRÙNG LẶP!
    │
Consumer A hoàn thành, gọi DeleteMessage
    │
Consumer B hoàn thành, gọi DeleteMessage ← Thành công (không lỗi)
    
Kết quả: M1 được xử lý 2 lần!
```

**Hệ quả:** Consumer phải **idempotent** (bất biến) — xử lý cùng tin nhắn nhiều lần không gây ra kết quả sai.

**2. Best-Effort Ordering (Thứ Tự Theo Nỗ Lực Tốt Nhất)**

Thứ tự tin nhắn **không được đảm bảo**. Tin nhắn gửi trước có thể được nhận sau:

```
Gửi vào: M1 → M2 → M3 → M4 → M5

Nhận ra: M3 → M1 → M5 → M2 → M4  ← Thứ tự không đoán được
```

**3. Unlimited Throughput (Thông Lượng Không Giới Hạn)**

Standard Queue có thể xử lý **không giới hạn** số tin nhắn per second — phù hợp cho hệ thống high-volume (khối lượng lớn).

### Khi Nào Dùng Standard Queue

✅ Throughput cao là ưu tiên  
✅ Thứ tự không quan trọng  
✅ Consumer có thể xử lý duplicate an toàn (idempotent)  
✅ Cost optimization (tối ưu chi phí) là yếu tố quan trọng  

**Ví dụ use case:**
- Gửi email marketing hàng loạt
- Log aggregation (thu thập log)
- Image/Video processing jobs
- Background task queues

---

## 🟢 FIFO Queue — Hàng Đợi FIFO (Vào Trước Ra Trước)

### Đặc Điểm Cốt Lõi

**1. Exactly-Once Processing (Xử Lý Đúng Một Lần)**

FIFO Queue dùng **Content-Based Deduplication** (Khử Trùng Dựa Trên Nội Dung) hoặc **Message Deduplication ID** (ID Khử Trùng Tin Nhắn) để đảm bảo tin nhắn không bị xử lý trùng:

```
Producer gửi M1 với MessageDeduplicationId = "order-001"
    │
    ▼
FIFO Queue nhận M1 → lưu deduplication ID trong 5 phút
    │
Producer gửi lại M1 (retry) với cùng ID "order-001"
    │
    ▼
FIFO Queue nhận ra duplicate → bỏ qua, KHÔNG thêm vào queue
    │
Consumer nhận M1 chỉ 1 lần ✅
```

**Hai cách deduplication:**

```python
# Cách 1: MessageDeduplicationId do producer cung cấp
sqs.send_message(
    QueueUrl=queue_url,
    MessageBody=json.dumps(order),
    MessageDeduplicationId="order-001-v1",  # Unique ID do bạn tạo
    MessageGroupId="orders"
)

# Cách 2: Content-Based Deduplication (bật trong queue settings)
# SQS tự tính SHA-256 hash của MessageBody làm deduplication ID
# Phù hợp khi message body đã unique
```

**2. Strict Ordering (Thứ Tự Nghiêm Ngặt) với Message Groups**

**Message Group ID** (ID Nhóm Tin Nhắn) cho phép kiểm soát thứ tự theo nhóm:

```
Queue FIFO: order-queue.fifo
│
├── Group "customer-A": M1 → M2 → M3  (xử lý theo thứ tự)
├── Group "customer-B": M4 → M5       (xử lý theo thứ tự)
└── Group "customer-C": M6            (xử lý theo thứ tự)
```

- Trong cùng 1 Message Group: thứ tự FIFO chặt chẽ
- Giữa các Message Group khác nhau: xử lý song song (parallel)

**Đây là thiết kế thông minh:** vừa đảm bảo thứ tự, vừa cho phép scale parallel processing.

**3. Throughput Limits (Giới Hạn Thông Lượng)**

| Kiểu Gửi | Giới Hạn |
|---|---|
| `SendMessage` thông thường | 300 msg/s |
| `SendMessageBatch` (10 msg/batch) | 3.000 msg/s |
| High throughput mode (opt-in) | 9.000 msg/s (batch), 900 msg/s (single) |

### Khi Nào Dùng FIFO Queue

✅ Thứ tự xử lý bắt buộc (ví dụ: các bước của một giao dịch)  
✅ Cần exactly-once delivery (giao vận đúng 1 lần)  
✅ Throughput không vượt ngưỡng giới hạn  
✅ Financial transactions (giao dịch tài chính)  

**Ví dụ use case:**
- Xử lý giao dịch ngân hàng (debit trước → credit sau)
- Inventory management (quản lý kho) — trừ hàng trước khi xác nhận đơn
- User action sequences (chuỗi hành động người dùng)
- Command processing trong CQRS architecture

---

## 🔍 So Sánh Chi Tiết

### Delivery Semantics (Ngữ Nghĩa Giao Vận)

```
Standard Queue — At-Least-Once:
┌─────────────────────────────────────────┐
│  Producer gửi: [M1] [M2] [M3]           │
│                                          │
│  Consumer nhận:                          │
│  Run 1: [M2] [M1] [M3]    ← đảo thứ tự  │
│  Run 2: [M1] [M3]         ← M2 mất?     │
│  Run 3: [M1] [M1] [M2]   ← M1 trùng!   │
└─────────────────────────────────────────┘

FIFO Queue — Exactly-Once:
┌─────────────────────────────────────────┐
│  Producer gửi: [M1] [M2] [M3]           │
│                                          │
│  Consumer nhận:                          │
│  Run 1: [M1] [M2] [M3]    ← đúng thứ tự │
│  (Không bao giờ: đảo thứ tự, trùng lặp) │
└─────────────────────────────────────────┘
```

### Throughput Comparison (So Sánh Thông Lượng)

```
Standard Queue:        ████████████████████████████  ∞ msg/s
FIFO (no batch):       ████  300 msg/s
FIFO (batch):          ████████████  3.000 msg/s
FIFO (high-throughput): ████████████████████  9.000 msg/s (batch)
```

### Message Attributes Đặc Thù FIFO

```json
{
  "MessageBody": "{\"orderId\": \"ORD-001\", \"amount\": 500}",
  "MessageGroupId": "customer-123",
  "MessageDeduplicationId": "ORD-001-place-order-2026-01-01"
}
```

- **MessageGroupId** — bắt buộc với FIFO; các tin nhắn cùng group được xử lý theo thứ tự
- **MessageDeduplicationId** — required nếu không bật Content-Based Deduplication; unique trong 5 phút

---

## 🛠️ Ví Dụ Code Thực Tế

### Gửi Tin Nhắn — Standard Queue

```python
import boto3
import json

sqs = boto3.client('sqs', region_name='us-east-1')
QUEUE_URL = "https://sqs.us-east-1.amazonaws.com/123456/email-queue"

def send_email_job(to_address: str, subject: str, body: str):
    message = {
        "to": to_address,
        "subject": subject,
        "body": body,
        "timestamp": "2026-01-01T10:00:00Z"
    }
    
    response = sqs.send_message(
        QueueUrl=QUEUE_URL,
        MessageBody=json.dumps(message),
        # DelaySeconds=0,  # Gửi ngay
        MessageAttributes={
            "ContentType": {
                "DataType": "String",
                "StringValue": "email"
            }
        }
    )
    
    print(f"Message sent: {response['MessageId']}")
    return response['MessageId']
```

### Gửi Tin Nhắn — FIFO Queue (Giao Dịch Ngân Hàng)

```python
import boto3
import json
import hashlib
import uuid

sqs = boto3.client('sqs', region_name='us-east-1')
FIFO_QUEUE_URL = "https://sqs.us-east-1.amazonaws.com/123456/transactions.fifo"

def process_bank_transaction(account_id: str, transaction_id: str, amount: float, txn_type: str):
    """
    Gửi giao dịch ngân hàng vào FIFO queue.
    
    - MessageGroupId = account_id: đảm bảo các giao dịch của cùng tài khoản
      được xử lý theo thứ tự
    - MessageDeduplicationId: tránh xử lý trùng khi producer retry
    """
    message = {
        "accountId": account_id,
        "transactionId": transaction_id,
        "amount": amount,
        "type": txn_type,  # "DEBIT" hoặc "CREDIT"
        "timestamp": "2026-01-01T10:00:00Z"
    }
    
    # Deduplication ID = transaction_id để tránh gửi trùng
    dedup_id = f"{transaction_id}-{txn_type}"
    
    response = sqs.send_message(
        QueueUrl=FIFO_QUEUE_URL,
        MessageBody=json.dumps(message),
        MessageGroupId=account_id,          # Nhóm theo tài khoản
        MessageDeduplicationId=dedup_id     # Unique per transaction
    )
    
    print(f"Transaction queued: {transaction_id}, SequenceNumber: {response['SequenceNumber']}")
    return response['MessageId']

# Ví dụ sử dụng
process_bank_transaction("ACC-001", "TXN-100", 1000.0, "DEBIT")
process_bank_transaction("ACC-001", "TXN-101", 500.0, "CREDIT")
# ACC-001 sẽ được xử lý: DEBIT TXN-100 trước, CREDIT TXN-101 sau — đúng thứ tự
```

### Consumer Idempotent Cho Standard Queue

```python
import boto3
import json
from functools import wraps

sqs = boto3.client('sqs', region_name='us-east-1')
dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
processed_table = dynamodb.Table('processed-messages')

def idempotent_handler(func):
    """Decorator đảm bảo handler chỉ xử lý mỗi message một lần."""
    @wraps(func)
    def wrapper(message_id: str, *args, **kwargs):
        # Kiểm tra đã xử lý chưa
        try:
            processed_table.put_item(
                Item={"messageId": message_id, "processedAt": "2026-01-01T10:00:00Z"},
                ConditionExpression="attribute_not_exists(messageId)"
            )
        except Exception as e:
            if "ConditionalCheckFailedException" in str(e):
                print(f"Message {message_id} đã được xử lý trước đó — bỏ qua")
                return  # Bỏ qua duplicate
            raise
        
        return func(message_id, *args, **kwargs)
    return wrapper

@idempotent_handler
def process_email(message_id: str, payload: dict):
    print(f"Gửi email đến {payload['to']} — {payload['subject']}")
    # Thực hiện gửi email thực tế...

def consumer_loop(queue_url: str):
    while True:
        response = sqs.receive_message(
            QueueUrl=queue_url,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20  # Long polling
        )
        
        for message in response.get('Messages', []):
            message_id = message['MessageId']
            body = json.loads(message['Body'])
            
            try:
                process_email(message_id, body)
                # Xóa sau khi xử lý thành công
                sqs.delete_message(
                    QueueUrl=queue_url,
                    ReceiptHandle=message['ReceiptHandle']
                )
            except Exception as e:
                print(f"Lỗi xử lý {message_id}: {e}")
                # Không xóa → tin nhắn visible lại sau Visibility Timeout
                # → sau N lần thất bại sẽ vào DLQ
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

### Q: Standard Queue at-least-once nghĩa là gì với consumer?

**A:** Consumer phải viết idempotent logic — xử lý cùng message ID nhiều lần phải cho kết quả như nhau. Thường dùng DynamoDB/Redis để lưu ID tin nhắn đã xử lý, hoặc dùng database `INSERT ... ON CONFLICT DO NOTHING`.

### Q: Khi nào nên chọn FIFO dù throughput bị giới hạn?

**A:** Khi thứ tự xử lý ảnh hưởng đến tính đúng đắn của hệ thống:
- Giao dịch tài chính (không thể credit trước debit)
- Hủy đơn hàng chỉ có thể sau khi tạo đơn
- State machine transitions (chuyển trạng thái máy trạng thái)
- Audit logs (nhật ký kiểm toán) cần thứ tự chronological (thời gian)

### Q: FIFO Queue có thể đạt throughput bao nhiêu?

**A:** Mặc định 300 TPS (transactions per second — giao dịch/giây), với batch là 3.000 TPS. Nếu bật **high-throughput FIFO mode**, đạt 9.000 TPS (batch) và 900 TPS (single message). Ngoài ra, dùng nhiều Message Group ID khác nhau giúp tăng parallelism (tính song song).

### Q: MessageGroupId và MessageDeduplicationId khác nhau thế nào?

**A:**
- **MessageGroupId** — kiểm soát *thứ tự*: các tin nhắn cùng group được xử lý theo FIFO, các group khác nhau xử lý song song
- **MessageDeduplicationId** — kiểm soát *trùng lặp*: nếu 2 tin nhắn có cùng ID trong 5 phút, tin thứ 2 bị bỏ qua

### Q: Content-Based Deduplication là gì?

**A:** Tính năng của FIFO Queue — SQS tự tính SHA-256 hash của `MessageBody` làm `MessageDeduplicationId`. Tiện lợi khi message body đã là unique (ví dụ: chứa UUID riêng biệt). Bật trong queue settings khi tạo queue.

---

## 🗺️ Điều Hướng

- ◀ Quay lại: [README.md](./README.md) — Tổng quan SQS
- ▶ Tiếp theo: [2-visibility-timeout.md](./2-visibility-timeout.md) — Visibility Timeout & Idempotency
