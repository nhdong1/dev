# Batch Processing — Xử Lý Theo Lô Với SQS

> Xử lý theo lô (batch) giúp tăng throughput (thông lượng), giảm chi phí API và tối ưu hiệu năng consumer.

## 🎯 Batch Processing Là Gì?

Thay vì gửi/nhận/xóa từng tin nhắn một, **Batch Processing** cho phép xử lý **nhiều tin nhắn trong một API call** duy nhất.

```
Không có batching:
  10 tin nhắn = 10 API calls (Send) + 10 API calls (Delete) = 20 calls

Có batching:
  10 tin nhắn = 1 API call (SendBatch) + 1 API call (DeleteBatch) = 2 calls
  
→ Giảm 90% số API calls, tiết kiệm chi phí tương ứng
```

---

## 📦 Ba API Batch Quan Trọng

| API | Mô Tả | Tối Đa |
|---|---|---|
| `SendMessageBatch` | Gửi nhiều tin nhắn cùng lúc | 10 messages/call |
| `ReceiveMessage` với `MaxNumberOfMessages` | Nhận nhiều tin nhắn | 10 messages/call |
| `DeleteMessageBatch` | Xóa nhiều tin nhắn cùng lúc | 10 messages/call |
| `ChangeMessageVisibilityBatch` | Thay đổi visibility của nhiều message | 10 messages/call |

**Giới hạn quan trọng:**
- Tối đa **10 messages per batch**
- Tổng kích thước payload batch tối đa **256 KB** (không phải 256 KB mỗi message)

---

## 📤 SendMessageBatch — Gửi Theo Lô

```python
import boto3
import json
import uuid
from typing import List, Dict, Any

sqs = boto3.client('sqs', region_name='us-east-1')
QUEUE_URL = "https://sqs.us-east-1.amazonaws.com/123456/order-queue"

def send_batch(messages: List[Dict[str, Any]]) -> Dict:
    """
    Gửi tối đa 10 tin nhắn trong 1 API call.
    
    messages: list of dicts với keys: body, delay_seconds (optional)
    """
    entries = []
    for i, msg in enumerate(messages[:10]):  # Tối đa 10
        entry = {
            "Id": str(i),              # ID duy nhất trong batch (không phải MessageId)
            "MessageBody": json.dumps(msg["body"])
        }
        
        if "delay_seconds" in msg:
            entry["DelaySeconds"] = msg["delay_seconds"]
        
        if "deduplication_id" in msg:  # Chỉ cho FIFO
            entry["MessageDeduplicationId"] = msg["deduplication_id"]
        
        if "group_id" in msg:          # Chỉ cho FIFO
            entry["MessageGroupId"] = msg["group_id"]
            
        entries.append(entry)
    
    response = sqs.send_message_batch(
        QueueUrl=QUEUE_URL,
        Entries=entries
    )
    
    # Kiểm tra kết quả từng message
    successful = response.get("Successful", [])
    failed = response.get("Failed", [])
    
    if failed:
        print(f"WARNING: {len(failed)} messages failed to send:")
        for failure in failed:
            print(f"  ID {failure['Id']}: {failure['Code']} — {failure['Message']}")
    
    print(f"Sent {len(successful)}/{len(entries)} messages successfully")
    return response


def send_large_batch(messages: List[Dict]) -> None:
    """
    Gửi danh sách lớn bằng cách chia thành các chunk (khối) 10 message.
    """
    CHUNK_SIZE = 10
    total = len(messages)
    sent = 0
    
    for i in range(0, total, CHUNK_SIZE):
        chunk = messages[i:i + CHUNK_SIZE]
        send_batch(chunk)
        sent += len(chunk)
        print(f"Progress: {sent}/{total}")


# Ví dụ sử dụng
orders = [
    {"body": {"orderId": f"ORD-{i:03d}", "amount": i * 10}, "delay_seconds": 0}
    for i in range(1, 51)  # 50 đơn hàng
]

send_large_batch(orders)
# Sẽ tạo 5 API calls (50 / 10 = 5) thay vì 50 calls riêng lẻ
```

---

## 📥 ReceiveMessage Với MaxNumberOfMessages

`ReceiveMessage` không phải batch API riêng biệt mà là tham số trong receive thông thường:

```python
def receive_batch(queue_url: str, batch_size: int = 10) -> List[dict]:
    """
    Nhận tối đa batch_size tin nhắn (tối đa = 10).
    
    Chú ý: SQS có thể trả về ít hơn batch_size kể cả khi queue có nhiều message.
    """
    response = sqs.receive_message(
        QueueUrl=queue_url,
        MaxNumberOfMessages=min(batch_size, 10),  # Tối đa 10
        WaitTimeSeconds=20,
        VisibilityTimeout=300,        # 5 phút cho batch processing
        AttributeNames=['All'],
        MessageAttributeNames=['All']
    )
    
    return response.get('Messages', [])
```

---

## 🗑️ DeleteMessageBatch — Xóa Theo Lô

Đây là API quan trọng nhất để tối ưu — luôn dùng delete batch thay vì delete từng cái:

```python
def delete_batch(queue_url: str, messages: List[dict]) -> None:
    """Xóa nhiều tin nhắn sau khi xử lý thành công."""
    
    if not messages:
        return
    
    entries = [
        {
            "Id": str(i),
            "ReceiptHandle": message['ReceiptHandle']
        }
        for i, message in enumerate(messages)
    ]
    
    response = sqs.delete_message_batch(
        QueueUrl=queue_url,
        Entries=entries
    )
    
    failed = response.get('Failed', [])
    if failed:
        # Log để điều tra — message sẽ visible lại sau timeout
        for failure in failed:
            print(f"Delete failed for ID {failure['Id']}: {failure['Code']}")
    
    print(f"Deleted {len(entries) - len(failed)}/{len(entries)} messages")
```

---

## 🔄 Full Batch Processing Pipeline (Đường Ống Xử Lý Theo Lô)

Pattern hoàn chỉnh: nhận batch → xử lý song song → xóa thành công, giữ lại thất bại:

```python
import boto3
import json
import logging
from typing import List, Tuple, Dict
from concurrent.futures import ThreadPoolExecutor, as_completed

sqs = boto3.client('sqs', region_name='us-east-1')
logger = logging.getLogger(__name__)

def process_single_message(message: dict) -> Tuple[bool, dict]:
    """
    Xử lý một message. Trả về (success, message).
    """
    try:
        body = json.loads(message['Body'])
        # --- Business Logic Ở Đây ---
        order_id = body.get('orderId')
        logger.info(f"Processing order: {order_id}")
        # Gọi DB, external service, etc.
        # ----------------------------
        return True, message
    except Exception as e:
        logger.error(f"Error processing {message['MessageId']}: {e}")
        return False, message

def batch_consumer(queue_url: str, max_workers: int = 5):
    """
    Consumer batch đầy đủ:
    1. Nhận batch tin nhắn
    2. Xử lý song song
    3. Xóa batch thành công
    4. Giữ lại tin nhắn thất bại để retry
    """
    
    while True:
        # Bước 1: Nhận batch
        messages = receive_batch(queue_url, batch_size=10)
        
        if not messages:
            logger.debug("Queue empty, polling again...")
            continue
        
        logger.info(f"Received {len(messages)} messages")
        
        # Bước 2: Xử lý song song
        successful_messages = []
        failed_messages = []
        
        with ThreadPoolExecutor(max_workers=max_workers) as executor:
            futures = {
                executor.submit(process_single_message, msg): msg
                for msg in messages
            }
            
            for future in as_completed(futures):
                success, message = future.result()
                if success:
                    successful_messages.append(message)
                else:
                    failed_messages.append(message)
        
        # Bước 3: Xóa batch các message thành công
        if successful_messages:
            delete_batch(queue_url, successful_messages)
        
        # Bước 4: Không làm gì với failed_messages
        # → Visibility Timeout tự hết → message visible lại → retry
        # → Sau maxReceiveCount lần → vào DLQ
        if failed_messages:
            logger.warning(
                f"{len(failed_messages)} messages will be retried: "
                + ", ".join(m['MessageId'] for m in failed_messages)
            )
        
        logger.info(
            f"Batch complete: {len(successful_messages)} success, "
            f"{len(failed_messages)} failed"
        )
```

---

## ⚡ Lambda Batch Processing (Xử Lý Theo Lô Với Lambda)

Khi dùng SQS làm **Event Source** (Nguồn Sự Kiện) cho Lambda:

### Cấu Hình Event Source Mapping

```bash
aws lambda create-event-source-mapping \
  --function-name order-processor \
  --event-source-arn arn:aws:sqs:us-east-1:123456:order-queue \
  --batch-size 10 \
  --maximum-batching-window-in-seconds 5 \
  --function-response-types ReportBatchItemFailures
```

| Tham Số | Ý Nghĩa | Giá Trị Khuyến Nghị |
|---|---|---|
| `batch-size` | Số message tối đa / Lambda invocation | 1–10.000 (SQS Standard), 1–10 (FIFO) |
| `maximum-batching-window-in-seconds` | Chờ thêm N giây để gom đủ batch | 0–300 giây |
| `function-response-types ReportBatchItemFailures` | Lambda báo cáo từng message thành công/thất bại | **Bắt buộc** cho partial failures |

### Lambda Handler Với Partial Batch Failure (Lỗi Một Phần Trong Lô)

```python
import json
import logging

logger = logging.getLogger(__name__)

def lambda_handler(event, context):
    """
    Xử lý batch SQS messages với partial failure support.
    
    Không bật ReportBatchItemFailures:
      - Nếu 1 message lỗi → toàn bộ batch được retry (kể cả các message đã thành công!)
      - Rất tệ: N-1 message được xử lý 2 lần
    
    Bật ReportBatchItemFailures:
      - Lambda báo cáo chính xác message nào lỗi
      - Chỉ message lỗi được retry → các message thành công không bị xử lý lại
    """
    
    failed_message_ids = []
    
    for record in event['Records']:
        message_id = record['messageId']
        
        try:
            body = json.loads(record['body'])
            process_order(body)
            logger.info(f"Processed: {message_id}")
            
        except Exception as e:
            logger.error(f"Failed: {message_id} — {e}")
            failed_message_ids.append(message_id)
            # KHÔNG raise exception — tiếp tục xử lý các message còn lại
    
    # Báo cáo các message thất bại — chỉ những này sẽ được retry
    # Message thành công tự động được xóa khỏi queue
    return {
        "batchItemFailures": [
            {"itemIdentifier": msg_id}
            for msg_id in failed_message_ids
        ]
    }
    # Trả về [] hoặc không có batchItemFailures → tất cả thành công

def process_order(body: dict):
    """Business logic xử lý đơn hàng."""
    order_id = body.get('orderId')
    if not order_id:
        raise ValueError(f"Missing orderId in message body")
    logger.info(f"Processing order {order_id}")
    # ... thực hiện business logic ...
```

---

## 💰 Tối Ưu Chi Phí Với Batching

### Tính Toán Chi Phí

```
Kịch bản: 1 triệu messages/ngày

Không batching:
  SendMessage:   1.000.000 calls
  DeleteMessage: 1.000.000 calls
  Total:         2.000.000 calls/ngày
  Chi phí:       2 × $0.40/triệu = $0.80/ngày = $24/tháng

Có batching (batch 10):
  SendMessageBatch:   100.000 calls (1.000.000 / 10)
  DeleteMessageBatch: 100.000 calls
  Total:              200.000 calls/ngày
  Chi phí:            0.2 × $0.40/triệu = $0.08/ngày = $2.4/tháng

Tiết kiệm: 90% chi phí API! ($21.6/tháng)
```

### MaximumBatchingWindowInSeconds — Gom Thêm Cho Đủ

```
Không có batching window:
  Message đến lúc 10:00:00 → Lambda trigger ngay với 1 message
  Message đến lúc 10:00:01 → Lambda trigger ngay với 1 message
  ...
  10 triggers riêng lẻ trong 10 giây

Có batching window = 5 giây:
  Messages đến từ 10:00:00 đến 10:00:05
  → Lambda chờ 5 giây
  → Gom được 10 messages
  → 1 Lambda invocation duy nhất

Chi phí Lambda: giảm 10x invocations
```

---

## 🛡️ Xử Lý Lỗi Trong Batch

### Partial Batch Failure vs All-or-Nothing

```
All-or-Nothing (KHÔNG dùng ReportBatchItemFailures):

Batch: [M1✅, M2✅, M3❌, M4✅, M5✅]
Lambda lỗi ở M3 → raise exception
→ Toàn bộ batch được retry
→ M1, M2, M4, M5 được xử lý 2 lần! ← Xấu nếu không idempotent

Partial Batch Failure (dùng ReportBatchItemFailures):

Batch: [M1✅, M2✅, M3❌, M4✅, M5✅]
Lambda xử lý tất cả, collect failed_ids = [M3]
Return batchItemFailures = [M3]
→ Chỉ M3 được retry
→ M1, M2, M4, M5 được xóa khỏi queue ← Đúng!
```

**Kết luận:** Luôn bật `ReportBatchItemFailures` khi dùng Lambda + SQS.

---

## 📋 Checklist Batch Processing

```
Gửi (Send):
├── ✅ Dùng SendMessageBatch thay vì SendMessage khi gửi nhiều message
├── ✅ Chia danh sách lớn thành chunks 10 message
├── ✅ Kiểm tra response.Failed để xử lý message gửi thất bại
└── ✅ Tổng payload batch ≤ 256 KB

Nhận (Receive):
├── ✅ MaxNumberOfMessages=10
├── ✅ WaitTimeSeconds=20 (Long Polling)
└── ✅ VisibilityTimeout đủ dài cho batch processing

Xóa (Delete):
├── ✅ Dùng DeleteMessageBatch thay vì DeleteMessage
├── ✅ Chỉ xóa message đã xử lý thành công
└── ✅ Xử lý response.Failed của delete batch

Lambda:
├── ✅ Bật ReportBatchItemFailures
├── ✅ Không raise exception cho từng message — chỉ collect failures
├── ✅ Return batchItemFailures đúng format
└── ✅ Consumer idempotent vì Lambda có thể retry
```

---

## ❓ Câu Hỏi Phỏng Vấn

### Q: Batch size tối đa của SQS là bao nhiêu?

**A:** 10 messages per API call cho cả Send, Receive, và Delete. Nhưng với Lambda Event Source Mapping, `batch-size` có thể lên tới 10.000 messages — tuy nhiên đây là số message Lambda nhận trong một invocation, bên dưới AWS vẫn gọi nhiều ReceiveMessage calls.

### Q: Tại sao phải dùng ReportBatchItemFailures với Lambda?

**A:** Không có tính năng này, nếu 1 trong 10 message lỗi, Lambda throw exception, toàn bộ batch bị retry — 9 message thành công bị xử lý lại. Với ReportBatchItemFailures, Lambda chỉ báo cáo đúng message nào lỗi, còn lại được xóa an toàn.

### Q: Có nên luôn dùng batch size tối đa (10)?

**A:** Không nhất thiết. Trade-off:
- **Batch lớn (10):** Chi phí thấp hơn, throughput cao hơn, nhưng nếu Lambda timeout toàn bộ batch có thể không được xử lý
- **Batch nhỏ (1-3):** Latency thấp hơn cho từng message, dễ debug, ít risk khi timeout

Với SQS Standard không quan trọng thứ tự: dùng batch 10. Với FIFO hoặc cần latency thấp: dùng batch nhỏ hơn.

### Q: MaximumBatchingWindowInSeconds ảnh hưởng gì đến latency?

**A:** Đây là trade-off: window lớn → batch đầy hơn → chi phí Lambda thấp hơn, nhưng mỗi message phải chờ tới W giây trước khi được xử lý. Với use case real-time, dùng window=0; với use case batch overnight, window=300 (5 phút) hợp lý.

---

## 🗺️ Điều Hướng

- ◀ Quay lại: [4-long-polling.md](./4-long-polling.md)
- ▶ Lên module SQS: [README.md](./README.md)
- ▶ Module tiếp theo: [../02-sns/README.md](../02-sns/README.md) — Amazon SNS
