# Dead Letter Queue (DLQ) — Hàng Đợi Thư Chết

> DLQ là nơi "cách ly" các tin nhắn lỗi, ngăn chúng block queue chính mãi mãi và giúp debug vấn đề production.

## 🎯 DLQ Là Gì?

**Dead Letter Queue — DLQ** (Hàng Đợi Thư Chết) là một SQS queue đặc biệt nhận các tin nhắn mà consumer **không thể xử lý thành công** sau N lần thử.

### Vấn Đề DLQ Giải Quyết

```
Không có DLQ:
┌────────────────────────────────────────────┐
│  Queue: [M1-POISON] [M2] [M3] [M4]         │
│                                             │
│  Consumer nhận M1 → xử lý thất bại         │
│  M1 visible lại → Consumer nhận M1 → lỗi   │
│  M1 visible lại → Consumer nhận M1 → lỗi   │
│  M1 visible lại → ... → vòng lặp vô tận!   │
│                                             │
│  Hệ quả:                                    │
│  - M2, M3, M4 bị trì hoãn (không bị block  │
│    về technical nhưng Consumer bị bận với   │
│    M1)                                      │
│  - CPU, logs tràn ngập error                │
│  - Khó phân biệt lỗi thật và poison message│
└────────────────────────────────────────────┘

Có DLQ:
┌────────────────────────────────────────────┐
│  Main Queue: [M1-POISON] [M2] [M3] [M4]    │
│                                             │
│  Consumer nhận M1 → thất bại (lần 1)       │
│  Consumer nhận M1 → thất bại (lần 2)       │
│  Consumer nhận M1 → thất bại (lần 3)       │
│  maxReceiveCount = 3 → M1 → DLQ ✅         │
│                                             │
│  Main Queue: [M2] [M3] [M4] ← tiếp tục    │
│  DLQ: [M1-POISON] ← chờ điều tra          │
└────────────────────────────────────────────┘
```

**Poison Message** (Tin Nhắn Độc Hại) — tin nhắn luôn gây lỗi khi xử lý, ví dụ: dữ liệu malformed (sai định dạng), schema không tương thích, logic lỗi.

---

## ⚙️ Cấu Hình DLQ

### Bước 1: Tạo DLQ Trước

```bash
# Tạo DLQ (cũng là SQS queue thông thường)
aws sqs create-queue \
  --queue-name order-queue-dlq \
  --attributes MessageRetentionPeriod=1209600  # 14 ngày (tối đa)
```

### Bước 2: Liên Kết DLQ Với Queue Chính

```bash
# Lấy ARN của DLQ
DLQ_ARN=$(aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456/order-queue-dlq \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' \
  --output text)

# Cấu hình Redrive Policy cho queue chính
aws sqs set-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456/order-queue \
  --attributes "{
    \"RedrivePolicy\": \"{\\\"deadLetterTargetArn\\\": \\\"${DLQ_ARN}\\\", \\\"maxReceiveCount\\\": \\\"3\\\"}\"
  }"
```

### Tham Số Redrive Policy (Chính Sách Chuyển Hướng)

| Tham Số | Ý Nghĩa | Giá Trị Khuyến Nghị |
|---|---|---|
| `deadLetterTargetArn` | ARN của DLQ | ARN đúng queue DLQ |
| `maxReceiveCount` | Số lần nhận tối đa trước khi chuyển sang DLQ | 3–5 |

**Chọn maxReceiveCount:**
- **3** — lỗi thường do transient error (lỗi tạm thời): network, DB timeout → 3 lần đủ
- **5** — nếu external dependencies (phụ thuộc bên ngoài) có thể chậm
- **1** — nếu tin nhắn không thể retry (không cần retry): acknowledge-only pattern
- **Không nên > 10** — quá nhiều retry làm tăng latency cho M2, M3, M4

---

## 🔍 DLQ Với Standard vs FIFO Queue

| Tiêu Chí | Standard DLQ | FIFO DLQ |
|---|---|---|
| **Queue chính** | Standard Queue | FIFO Queue |
| **DLQ phải là** | Standard Queue | FIFO Queue |
| **Giữ Message Group** | N/A | Có — vẫn giữ MessageGroupId |
| **Thứ tự trong DLQ** | Không đảm bảo | Đảm bảo FIFO trong group |

**Quy tắc bắt buộc:** DLQ phải **cùng loại** với queue chính (Standard→Standard, FIFO→FIFO) và phải ở **cùng AWS Region và Account**.

---

## 📊 Monitoring DLQ (Giám Sát DLQ)

Thiết lập CloudWatch Alarm (Cảnh Báo CloudWatch) cho DLQ là **bắt buộc** trong production:

### Tạo Alarm Cho DLQ

```bash
# Cảnh báo khi DLQ có bất kỳ tin nhắn nào
aws cloudwatch put-metric-alarm \
  --alarm-name "DLQ-Has-Messages" \
  --alarm-description "Có tin nhắn trong DLQ — cần điều tra ngay" \
  --namespace AWS/SQS \
  --metric-name ApproximateNumberOfMessagesVisible \
  --dimensions Name=QueueName,Value=order-queue-dlq \
  --statistic Maximum \
  --period 60 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456:ops-alerts-topic \
  --treat-missing-data notBreaching
```

### Metrics Cần Theo Dõi

```
DLQ Metrics:
├── ApproximateNumberOfMessagesVisible  → bao nhiêu tin nhắn đang chờ trong DLQ
├── ApproximateAgeOfOldestMessage       → tin nhắn cũ nhất bao lâu (giây)
└── NumberOfMessagesSent (to DLQ)       → tốc độ tin nhắn vào DLQ (báo hiệu vấn đề)

Main Queue Metrics (liên quan):
└── ApproximateNumberOfMessagesNotVisible  → đang được xử lý (in-flight)
```

---

## 🔄 DLQ Redrive — Gửi Lại Tin Nhắn Từ DLQ

Sau khi fix bug, bạn muốn xử lý lại các tin nhắn trong DLQ.

### Cách 1: SQS Redrive (Console hoặc API) — AWS Managed

```bash
# Bắt đầu redrive (gửi lại) từ DLQ về source queue
aws sqs start-message-move-task \
  --source-arn arn:aws:sqs:us-east-1:123456:order-queue-dlq \
  --destination-arn arn:aws:sqs:us-east-1:123456:order-queue \
  --max-number-of-messages-per-second 10  # Giới hạn tốc độ để tránh overwhelm

# Kiểm tra trạng thái redrive
aws sqs list-message-move-tasks \
  --source-arn arn:aws:sqs:us-east-1:123456:order-queue-dlq
```

### Cách 2: Manual Consumer + Re-enqueue

```python
import boto3
import json
import logging

sqs = boto3.client('sqs', region_name='us-east-1')

DLQ_URL = "https://sqs.us-east-1.amazonaws.com/123456/order-queue-dlq"
MAIN_QUEUE_URL = "https://sqs.us-east-1.amazonaws.com/123456/order-queue"

def redrive_dlq_messages(batch_size: int = 10, dry_run: bool = True):
    """
    Đọc tin nhắn từ DLQ và gửi lại vào main queue.
    
    dry_run=True: chỉ in ra, không thực sự gửi — dùng để xem trước
    dry_run=False: thực hiện redrive thật
    """
    processed = 0
    errors = 0
    
    while True:
        response = sqs.receive_message(
            QueueUrl=DLQ_URL,
            MaxNumberOfMessages=min(batch_size, 10),
            WaitTimeSeconds=5,
            AttributeNames=['All'],
            MessageAttributeNames=['All']
        )
        
        messages = response.get('Messages', [])
        if not messages:
            print(f"DLQ trống. Đã redrive: {processed}, lỗi: {errors}")
            break
        
        for message in messages:
            message_id = message['MessageId']
            body = message['Body']
            
            # Log thông tin để audit trail (vết kiểm toán)
            approx_receive_count = message.get('Attributes', {}).get('ApproximateReceiveCount', 'N/A')
            logging.info(f"DLQ message: {message_id}, receive_count: {approx_receive_count}")
            
            if dry_run:
                print(f"[DRY RUN] Would redrive: {message_id}")
                print(f"  Body: {body[:100]}...")
                processed += 1
                continue
            
            try:
                # Gửi lại vào main queue
                sqs.send_message(
                    QueueUrl=MAIN_QUEUE_URL,
                    MessageBody=body,
                    MessageAttributes=message.get('MessageAttributes', {})
                )
                
                # Xóa khỏi DLQ
                sqs.delete_message(
                    QueueUrl=DLQ_URL,
                    ReceiptHandle=message['ReceiptHandle']
                )
                
                processed += 1
                print(f"Redriven: {message_id}")
                
            except Exception as e:
                errors += 1
                logging.error(f"Failed to redrive {message_id}: {e}")
    
    return {"processed": processed, "errors": errors}
```

---

## 🛡️ Patterns Xử Lý DLQ Trong Production

### Pattern 1: DLQ Inspector (Thanh Tra DLQ)

Lambda trigger khi có message trong DLQ → tự phân tích và alert chi tiết:

```python
import boto3
import json

sns = boto3.client('sns', region_name='us-east-1')
ALERT_TOPIC_ARN = "arn:aws:sns:us-east-1:123456:ops-alerts"

def lambda_handler(event, context):
    """
    Lambda được trigger bởi SQS DLQ event source mapping.
    Phân tích lỗi và gửi alert chi tiết.
    """
    for record in event['Records']:
        message_id = record['messageId']
        body = json.loads(record['body'])
        attributes = record.get('attributes', {})
        
        # Thông tin lỗi
        receive_count = attributes.get('ApproximateReceiveCount', 'N/A')
        sent_timestamp = attributes.get('SentTimestamp', 'N/A')
        
        # Gửi alert chi tiết
        alert_message = f"""
⚠️ DLQ Alert — Tin Nhắn Thất Bại

Message ID: {message_id}
Số lần thử: {receive_count}
Thời gian gửi: {sent_timestamp}
Nội dung:
{json.dumps(body, indent=2, ensure_ascii=False)}

Action cần làm:
1. Kiểm tra CloudWatch Logs của main consumer
2. Xác định nguyên nhân lỗi
3. Fix bug nếu cần
4. Dùng SQS Redrive để xử lý lại
        """
        
        sns.publish(
            TopicArn=ALERT_TOPIC_ARN,
            Subject=f"[DLQ ALERT] Message {message_id} failed",
            Message=alert_message
        )
    
    # Quan trọng: return bình thường để xóa khỏi DLQ
    # Nếu muốn giữ lại DLQ để audit thêm, throw exception
    return {"statusCode": 200}
```

### Pattern 2: Retry Với Exponential Backoff (Tăng Dần Theo Hàm Mũ)

```python
import boto3
import json
import time
import math

sqs = boto3.client('sqs', region_name='us-east-1')

def get_retry_delay(attempt: int, base_delay: float = 1.0, max_delay: float = 300.0) -> float:
    """
    Exponential backoff với jitter (độ ngẫu nhiên).
    attempt=1 → ~1s, attempt=2 → ~2s, attempt=3 → ~4s, ...
    """
    delay = min(base_delay * (2 ** (attempt - 1)), max_delay)
    # Jitter: thêm ngẫu nhiên ±25% để tránh thundering herd (đàn sét đánh)
    import random
    jitter = delay * 0.25 * (2 * random.random() - 1)
    return max(0, delay + jitter)

def process_with_retry(queue_url: str, max_attempts: int = 3):
    """Consumer với retry và delay tăng dần."""
    while True:
        response = sqs.receive_message(
            QueueUrl=queue_url,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20,
            AttributeNames=['ApproximateReceiveCount']
        )
        
        for message in response.get('Messages', []):
            message_id = message['MessageId']
            receive_count = int(message['Attributes'].get('ApproximateReceiveCount', 1))
            body = json.loads(message['Body'])
            
            try:
                # Nếu là retry, thêm delay
                if receive_count > 1:
                    delay = get_retry_delay(receive_count - 1)
                    print(f"Retry #{receive_count} cho {message_id}, chờ {delay:.1f}s")
                    time.sleep(delay)
                
                process_business_logic(body)
                
                sqs.delete_message(
                    QueueUrl=queue_url,
                    ReceiptHandle=message['ReceiptHandle']
                )
                print(f"Xử lý thành công: {message_id}")
                
            except Exception as e:
                print(f"Lỗi lần {receive_count}: {message_id} — {e}")
                # Không xóa → Visibility Timeout hết → visible lại → retry
                # Sau maxReceiveCount lần → vào DLQ tự động
```

---

## 📋 Checklist DLQ Production

```
Thiết lập DLQ:
├── ✅ Tạo DLQ trước khi tạo main queue
├── ✅ DLQ cùng loại với main queue (Standard/FIFO)
├── ✅ Đặt MessageRetentionPeriod DLQ = 14 ngày (tối đa)
├── ✅ maxReceiveCount phù hợp (3-5 thường đủ)
└── ✅ Kiểm tra ARN đúng trong Redrive Policy

Monitoring:
├── ✅ CloudWatch Alarm khi DLQ > 0 messages
├── ✅ Alarm gửi đến SNS topic → email/Slack/PagerDuty
├── ✅ Dashboard hiển thị số message trong DLQ
└── ✅ Log message_id khi consumer thất bại

Quy Trình Xử Lý Khi Có Alert:
├── 1. Xem message trong DLQ (không xóa)
├── 2. Kiểm tra CloudWatch Logs → xác định nguyên nhân
├── 3. Fix bug (nếu là code bug)
├── 4. Deploy fix
└── 5. Dùng SQS Redrive để xử lý lại

Code Consumer:
├── ✅ Log message_id và lỗi khi xử lý thất bại
├── ✅ KHÔNG xóa message khi xử lý lỗi
├── ✅ Implement idempotency nếu dùng Standard Queue
└── ✅ Set Visibility Timeout đủ dài
```

---

## ❓ Câu Hỏi Phỏng Vấn

### Q: Tại sao cần DLQ? Không thể dùng try-catch trong consumer?

**A:** Try-catch xử lý lỗi runtime tạm thời, nhưng không giải quyết **poison message** — tin nhắn luôn gây lỗi dù retry bao nhiêu lần (ví dụ: dữ liệu malformed, schema mismatch). Không có DLQ, poison message sẽ block consumer mãi mãi và làm đầy logs lỗi. DLQ tách biệt "tin nhắn bình thường" khỏi "tin nhắn cần điều tra", giúp cả hai được xử lý đúng cách.

### Q: maxReceiveCount nên đặt là bao nhiêu?

**A:** Phụ thuộc vào loại lỗi kỳ vọng:
- **3** — phù hợp cho hầu hết hệ thống; đủ cho transient errors (lỗi tạm thời) như network timeout
- **1** — dùng khi poison message rõ ràng ngay từ lần đầu (ví dụ: validation lỗi ngay)
- **5-10** — khi consumer gọi external service có thể slow hoặc có SLA dài
- Không đặt quá cao vì retry làm tăng latency cho các message khác

### Q: Khi nào nên tự động redrive từ DLQ, khi nào cần manual?

**A:**
- **Tự động redrive** — khi lỗi là transient (network, timeout) và không cần fix code
- **Manual redrive sau fix** — khi lỗi do code bug; phải fix bug trước, deploy, rồi redrive
- **Không redrive** — khi tin nhắn là invalid data từ upstream (source xấu); cần fix source

### Q: DLQ có thể có DLQ không?

**A:** Không — DLQ không thể có DLQ của riêng nó. Nếu Lambda đọc DLQ mà lỗi, Lambda có **Lambda Destinations** hoặc bạn có thể dùng EventBridge để bắt Lambda failure.

---

## 🗺️ Điều Hướng

- ◀ Quay lại: [2-visibility-timeout.md](./2-visibility-timeout.md)
- ▶ Tiếp theo: [4-long-polling.md](./4-long-polling.md) — Long Polling vs Short Polling
