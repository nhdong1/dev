# Long Polling vs Short Polling — Lấy Tin Nhắn Chờ Dài vs Chờ Ngắn

> Chọn đúng polling strategy (chiến lược thăm dò) giúp giảm chi phí đáng kể và cải thiện latency (độ trễ) của hệ thống.

## 🎯 Polling Là Gì?

SQS dùng mô hình **pull-based** — consumer chủ động "thăm dò" (poll) queue để lấy tin nhắn. Có hai chiến lược:

- **Short Polling** (Thăm Dò Ngắn) — hỏi ngay, trả lời ngay, kể cả khi không có tin nhắn
- **Long Polling** (Thăm Dò Dài) — chờ đến khi có tin nhắn hoặc hết timeout mới trả lời

---

## 🔴 Short Polling — Thăm Dò Ngắn

### Cách Hoạt Động

```
Consumer                        SQS Queue
   │                               │
   │──── ReceiveMessage() ─────────▶│
   │                               │  Kiểm tra 1 vài servers
   │◀─── Response: [] (rỗng) ──────│  (không phải toàn bộ)
   │                               │
   │  (chờ 1 giây)                 │
   │                               │
   │──── ReceiveMessage() ─────────▶│
   │◀─── Response: [] (rỗng) ──────│
   │                               │
   │  (chờ 1 giây)                 │
   │                               │
   │──── ReceiveMessage() ─────────▶│
   │◀─── Response: [M1] ───────────│  Lần này có tin nhắn!
```

### Vấn Đề Của Short Polling

**1. Chi phí cao (Wasted API Calls)**

```
Giả sử queue nhận 1 message / phút:
- Short polling: 60 request/phút × 60 phút = 3.600 request/giờ
  → Hầu hết là empty response (phản hồi rỗng) — lãng phí tiền
  
- Long polling (20s wait): 3 request/phút × 60 phút = 180 request/giờ
  → Giảm 95% số request!
```

**2. False Empty Response (Phản Hồi Rỗng Giả)**

Short Polling chỉ query **một phần** server của SQS (SQS phân tán trên nhiều server). Nếu tin nhắn nằm trên server chưa được query → trả về rỗng dù queue có tin nhắn!

```
SQS internal servers (ví dụ):
├── Server A: [M1, M2]
├── Server B: []
├── Server C: [M3]
└── Server D: []

Short Polling query Server B, D → trả về rỗng
(Dù thực tế queue có M1, M2, M3)
```

**3. Higher Latency (Độ Trễ Cao Hơn)**

Nếu consumer đang ở giữa polling interval, tin nhắn phải chờ đến lần poll tiếp theo.

### Khi Nào Dùng Short Polling

Short Polling (`WaitTimeSeconds=0`) hầu như không có use case tốt trong production. Dùng khi:
- Testing/debugging cần phản hồi ngay lập tức
- Consumer chỉ cần "fire-and-forget" không cần chờ

---

## 🟢 Long Polling — Thăm Dò Dài

### Cách Hoạt Động

```
Consumer                        SQS Queue
   │                               │
   │── ReceiveMessage(wait=20s) ───▶│
   │                               │  Query TẤT CẢ servers
   │                               │  (nếu không có tin nhắn,
   │                               │   chờ đến khi có hoặc hết 20s)
   │                               │
   │                               │  ... chờ ...  ... chờ ...
   │                               │
   │                               │  M1 xuất hiện!
   │◀─── Response: [M1] ───────────│  Trả về ngay
   │
   │── ReceiveMessage(wait=20s) ───▶│
   │                               │  ... chờ 20 giây ...
   │◀─── Response: [] ─────────────│  Hết 20s, trả về rỗng
```

### Lợi Ích Của Long Polling

| Lợi Ích | Mô Tả |
|---|---|
| **Giảm chi phí** | Ít API call hơn, đặc biệt khi queue thưa tin nhắn |
| **Giảm false empty** | Query toàn bộ servers → không bỏ sót tin nhắn |
| **Giảm latency** | Nhận tin nhắn ngay khi có, không cần đợi polling interval |
| **Giảm CPU/network** | Consumer không busy-loop liên tục |

### Cấu Hình Long Polling

**Cách 1: Cấu hình mặc định cho toàn queue (khuyến nghị)**

```bash
# Đặt Long Polling mặc định cho queue
aws sqs set-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456/my-queue \
  --attributes ReceiveMessageWaitTimeSeconds=20

# Kiểm tra cấu hình
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456/my-queue \
  --attribute-names ReceiveMessageWaitTimeSeconds
```

**Cách 2: Override per-request**

```python
response = sqs.receive_message(
    QueueUrl=queue_url,
    MaxNumberOfMessages=10,
    WaitTimeSeconds=20  # Override cho lần gọi này
)
```

### Giá Trị WaitTimeSeconds

| Giá Trị | Loại | Hành Vi |
|---|---|---|
| `0` | Short Polling | Trả về ngay (kể cả rỗng) |
| `1–19` | Long Polling ngắn | Chờ tối đa N giây |
| `20` | Long Polling tối đa | Chờ tối đa 20 giây — **khuyến nghị** |

**Khuyến nghị:** Luôn dùng `WaitTimeSeconds=20` trừ khi có lý do đặc biệt.

---

## 💻 Code Thực Tế

### Consumer Loop Cơ Bản Với Long Polling

```python
import boto3
import json
import logging
import signal
import sys

sqs = boto3.client('sqs', region_name='us-east-1')
logger = logging.getLogger(__name__)

class SQSConsumer:
    def __init__(self, queue_url: str, handler, max_messages: int = 10):
        self.queue_url = queue_url
        self.handler = handler
        self.max_messages = max_messages
        self.running = True
        
        # Graceful shutdown khi nhận SIGTERM / SIGINT
        signal.signal(signal.SIGTERM, self._shutdown)
        signal.signal(signal.SIGINT, self._shutdown)

    def _shutdown(self, signum, frame):
        logger.info("Shutting down consumer gracefully...")
        self.running = False

    def run(self):
        logger.info(f"Consumer started: {self.queue_url}")
        
        while self.running:
            try:
                response = sqs.receive_message(
                    QueueUrl=self.queue_url,
                    MaxNumberOfMessages=self.max_messages,
                    WaitTimeSeconds=20,              # Long polling
                    VisibilityTimeout=60,
                    AttributeNames=['All'],
                    MessageAttributeNames=['All']
                )
                
                messages = response.get('Messages', [])
                
                if not messages:
                    # Long polling đã chờ 20s, queue rỗng — poll lại
                    continue
                
                for message in messages:
                    self._process_message(message)
                    
            except Exception as e:
                logger.error(f"Error in consumer loop: {e}")
                # Không crash — tiếp tục loop
    
    def _process_message(self, message: dict):
        message_id = message['MessageId']
        receipt_handle = message['ReceiptHandle']
        
        try:
            body = json.loads(message['Body'])
            self.handler(body)
            
            # Xóa sau khi xử lý thành công
            sqs.delete_message(
                QueueUrl=self.queue_url,
                ReceiptHandle=receipt_handle
            )
            logger.info(f"Processed and deleted: {message_id}")
            
        except json.JSONDecodeError as e:
            # Malformed JSON — xóa luôn, không retry (sẽ vào DLQ không giải quyết được)
            logger.error(f"Invalid JSON in {message_id}: {e}. Deleting.")
            sqs.delete_message(
                QueueUrl=self.queue_url,
                ReceiptHandle=receipt_handle
            )
            
        except Exception as e:
            logger.error(f"Error processing {message_id}: {e}")
            # KHÔNG xóa → visibility timeout hết → retry → DLQ sau N lần

# Sử dụng
def my_handler(payload: dict):
    print(f"Processing order: {payload['orderId']}")
    # ... business logic ...

consumer = SQSConsumer(
    queue_url="https://sqs.us-east-1.amazonaws.com/123456/order-queue",
    handler=my_handler
)
consumer.run()
```

### Multi-threaded Consumer (Consumer Đa Luồng)

```python
import boto3
import json
import threading
import logging
from concurrent.futures import ThreadPoolExecutor

sqs = boto3.client('sqs', region_name='us-east-1')
logger = logging.getLogger(__name__)

def process_message(message: dict, queue_url: str):
    """Xử lý một message trong thread riêng."""
    message_id = message['MessageId']
    try:
        body = json.loads(message['Body'])
        # ... xử lý business logic ...
        
        sqs.delete_message(
            QueueUrl=queue_url,
            ReceiptHandle=message['ReceiptHandle']
        )
        logger.info(f"Thread {threading.current_thread().name}: processed {message_id}")
        
    except Exception as e:
        logger.error(f"Failed {message_id}: {e}")

def concurrent_consumer(queue_url: str, num_workers: int = 5):
    """
    Consumer sử dụng thread pool để xử lý song song.
    Phù hợp khi mỗi message cần nhiều thời gian xử lý.
    """
    with ThreadPoolExecutor(max_workers=num_workers) as executor:
        while True:
            response = sqs.receive_message(
                QueueUrl=queue_url,
                MaxNumberOfMessages=10,   # Lấy tối đa 10 message
                WaitTimeSeconds=20
            )
            
            for message in response.get('Messages', []):
                # Submit vào thread pool — không chờ kết quả
                executor.submit(process_message, message, queue_url)
```

---

## 🔄 So Sánh Chi Phí Thực Tế

```
Kịch bản: Queue nhận 100 message/giờ, consumer chạy 24/7

Short Polling (poll mỗi 1 giây):
  Requests/giờ = 3.600
  Empty responses = 3.600 - 100 = 3.500 (lãng phí)
  Chi phí/tháng = 3.600 × 24 × 30 / 1.000.000 × $0.40
                = ~$1.04/tháng (gần như toàn bộ là lãng phí)

Long Polling (WaitTimeSeconds=20):
  Requests/giờ ≈ 100 (gần như mỗi request có 1 message)
  + ~3 requests rỗng/giờ (khi queue thưa)
  = ~103 requests/giờ
  Chi phí/tháng = 103 × 24 × 30 / 1.000.000 × $0.40
                = ~$0.030/tháng

Tiết kiệm: ~97% chi phí API!
(Khoản tiết kiệm thực tế lớn hơn với hệ thống nhiều consumer)
```

---

## ⚖️ Long Polling và Lambda Trigger

Khi dùng **SQS → Lambda** (Event Source Mapping — Ánh Xạ Nguồn Sự Kiện), Lambda **tự động** dùng Long Polling bên dưới — bạn không cần cấu hình:

```
SQS Queue ──(Long Polling, AWS managed)──▶ Lambda Function
```

Tuy nhiên, bạn vẫn nên đặt `ReceiveMessageWaitTimeSeconds=20` trên queue để tiết kiệm chi phí khi có consumer tự quản lý bên ngoài Lambda.

---

## ❓ Câu Hỏi Phỏng Vấn

### Q: Long Polling khác Short Polling thế nào, nên dùng cái nào?

**A:** Short Polling trả về ngay kể cả queue rỗng; query chỉ một phần servers SQS nên có thể bỏ sót message. Long Polling chờ tới 20 giây nếu rỗng và query toàn bộ servers. Nên dùng Long Polling (`WaitTimeSeconds=20`) mặc định — giảm ~95% API calls và chi phí, giảm latency vì nhận message ngay khi có.

### Q: WaitTimeSeconds=20 có nghĩa là consumer chờ 20 giây không?

**A:** Chờ **tối đa** 20 giây. Nếu có message, SQS trả về ngay lập tức không cần đợi hết 20 giây. Nếu không có message nào sau 20 giây, trả về response rỗng.

### Q: Tại sao Short Polling có thể trả về rỗng dù queue có message?

**A:** SQS lưu trữ phân tán trên nhiều servers. Short Polling chỉ query một tập con servers (không phải tất cả). Nếu message nằm trên servers chưa được query trong round này → trả về rỗng. Long Polling query toàn bộ servers nên không có vấn đề này.

### Q: MaxNumberOfMessages=10 có đảm bảo nhận đúng 10 message không?

**A:** Không — đây là **tối đa** 10. SQS có thể trả về ít hơn kể cả khi queue có nhiều hơn. Lý do: SQS phân tán và không locking toàn bộ queue. Để xử lý hết queue, cần loop cho đến khi nhận được response rỗng.

---

## 🗺️ Điều Hướng

- ◀ Quay lại: [3-dead-letter-queue.md](./3-dead-letter-queue.md)
- ▶ Tiếp theo: [5-batch-processing.md](./5-batch-processing.md) — Xử Lý Theo Lô
