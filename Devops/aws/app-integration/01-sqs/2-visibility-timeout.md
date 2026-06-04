# Visibility Timeout & Idempotency — Thời Gian Ẩn và Tính Bất Biến

> Visibility Timeout là cơ chế bảo vệ tin nhắn khỏi bị xử lý trùng lặp. Hiểu đúng cơ chế này là nền tảng để xây dựng consumer (người tiêu dùng) đáng tin cậy.

## 🎯 Visibility Timeout Là Gì?

**Visibility Timeout** (Thời Gian Ẩn Tin Nhắn) là khoảng thời gian một tin nhắn bị **ẩn** khỏi queue sau khi consumer nhận (receive) nó.

### Mục Đích

```
Không có Visibility Timeout:
┌─────────────────────────────────────────┐
│  Queue: [M1] [M2] [M3]                  │
│                                          │
│  Consumer A nhận M1                      │
│  Consumer B cũng nhận M1 ngay lập tức!  │ ← VẤN ĐỀ
│  → M1 bị xử lý 2 lần                    │
└─────────────────────────────────────────┘

Có Visibility Timeout (30 giây):
┌─────────────────────────────────────────┐
│  Queue: [M1] [M2] [M3]                  │
│                                          │
│  Consumer A nhận M1                      │
│  → M1 ẩn trong 30 giây                  │
│  Consumer B KHÔNG thấy M1               │ ← An toàn
│                                          │
│  Consumer A xử lý xong, delete M1 ✅    │
│  HOẶC                                    │
│  Timeout hết → M1 visible lại           │
│  → Consumer B có thể nhận và retry       │
└─────────────────────────────────────────┘
```

---

## ⏱️ Cơ Chế Hoạt Động

### Timeline Đầy Đủ

```
T=0s    Producer gửi M1 vào queue
        Queue: [M1 VISIBLE]

T=5s    Consumer nhận M1 (ReceiveMessage)
        Queue: [M1 INVISIBLE — 30s]
        ReceiptHandle = "XYZ-abc-123"

T=10s   Consumer đang xử lý M1...
        Queue: [M1 INVISIBLE — 20s còn lại]

        === Kịch bản A: Xử lý thành công ===

T=15s   Consumer gọi DeleteMessage(ReceiptHandle="XYZ-abc-123")
        Queue: [] ← M1 bị xóa vĩnh viễn ✅

        === Kịch bản B: Consumer bị crash ===

T=35s   Visibility Timeout hết (30s đã qua)
        Queue: [M1 VISIBLE lại]
        Consumer khác có thể nhận và xử lý lại ✅

        === Kịch bản C: Xử lý cần nhiều thời gian hơn ===

T=25s   Consumer nhận ra cần > 30s để xử lý
        Gọi ChangeMessageVisibility để extend timeout
        Queue: [M1 INVISIBLE — thêm 60s] ← Gia hạn thành công
```

---

## ⚙️ Cấu Hình Visibility Timeout

### Giá Trị

| Thông Số | Giá Trị |
|---|---|
| **Mặc định** | 30 giây |
| **Tối thiểu** | 0 giây |
| **Tối đa** | 12 giờ (43.200 giây) |
| **Khuyến nghị** | Gấp 6 lần thời gian xử lý trung bình |

### Cấu Hình Khi Tạo Queue

```bash
# Tạo queue với Visibility Timeout 5 phút
aws sqs create-queue \
  --queue-name order-processing-queue \
  --attributes VisibilityTimeout=300

# Cập nhật Visibility Timeout cho queue hiện có
aws sqs set-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456/order-queue \
  --attributes VisibilityTimeout=300
```

### Cấu Hình Khi Nhận Tin Nhắn (Override Per-Message)

```python
import boto3

sqs = boto3.client('sqs', region_name='us-east-1')

# Override visibility timeout cho lần receive này
response = sqs.receive_message(
    QueueUrl=queue_url,
    MaxNumberOfMessages=5,
    VisibilityTimeout=120,      # 2 phút cho batch này
    WaitTimeSeconds=20
)
```

---

## 🔄 ChangeMessageVisibility — Gia Hạn Thời Gian Ẩn

Khi consumer biết sẽ cần nhiều thời gian hơn dự kiến, dùng **ChangeMessageVisibility** để gia hạn.

### Khi Nào Cần Gia Hạn?

```
Tình huống thực tế:
- Xử lý video lớn: 5 phút bình thường, nhưng file này 500MB cần 15 phút
- Gọi external API (API bên ngoài): response chậm bất thường
- Database transaction (giao dịch cơ sở dữ liệu) phức tạp
```

### Implement Heartbeat (Nhịp Tim) Pattern

```python
import boto3
import threading
import time
import json

sqs = boto3.client('sqs', region_name='us-east-1')

class VisibilityExtender:
    """
    Tự động gia hạn Visibility Timeout trong nền.
    Dừng lại khi xử lý hoàn thành.
    """
    
    def __init__(self, queue_url: str, receipt_handle: str,
                 extend_interval: int = 25, extend_by: int = 60):
        self.queue_url = queue_url
        self.receipt_handle = receipt_handle
        self.extend_interval = extend_interval  # Gia hạn mỗi 25 giây
        self.extend_by = extend_by              # Gia hạn thêm 60 giây mỗi lần
        self._stop_event = threading.Event()
        self._thread = None

    def start(self):
        self._thread = threading.Thread(target=self._extend_loop, daemon=True)
        self._thread.start()

    def stop(self):
        self._stop_event.set()
        if self._thread:
            self._thread.join()

    def _extend_loop(self):
        while not self._stop_event.wait(timeout=self.extend_interval):
            try:
                sqs.change_message_visibility(
                    QueueUrl=self.queue_url,
                    ReceiptHandle=self.receipt_handle,
                    VisibilityTimeout=self.extend_by
                )
                print(f"Visibility extended by {self.extend_by}s")
            except Exception as e:
                print(f"Failed to extend visibility: {e}")
                break

def process_large_video(message: dict):
    """Xử lý video lớn, có thể mất nhiều thời gian."""
    receipt_handle = message['ReceiptHandle']
    body = json.loads(message['Body'])
    
    # Khởi động extender
    extender = VisibilityExtender(
        queue_url=QUEUE_URL,
        receipt_handle=receipt_handle,
        extend_interval=25,  # Gia hạn trước khi 30s timeout
        extend_by=60         # Thêm 60s mỗi lần gia hạn
    )
    extender.start()
    
    try:
        print(f"Processing video: {body['videoId']}")
        time.sleep(300)  # Giả lập xử lý 5 phút
        print("Video processing complete!")
        
        # Xóa message sau khi xử lý xong
        sqs.delete_message(
            QueueUrl=QUEUE_URL,
            ReceiptHandle=receipt_handle
        )
    finally:
        extender.stop()  # Luôn dừng extender dù thành công hay thất bại
```

---

## 🔁 Idempotency (Tính Bất Biến) Trong SQS Consumer

**Idempotency** — một operation được gọi nhiều lần cho ra cùng kết quả như gọi 1 lần. Đây là yêu cầu bắt buộc với Standard Queue vì at-least-once delivery.

### Tại Sao Cần Idempotency?

```
Ngay cả với Visibility Timeout, duplicate vẫn xảy ra:

Tình huống 1: Timeout quá ngắn
  Consumer nhận M1, timeout = 30s
  Consumer xử lý mất 35s
  Ở giây 30: M1 visible lại
  Consumer B nhận M1 → XỬ LÝ TRÙNG

Tình huống 2: Consumer crash sau DeleteMessage thất bại
  Consumer xử lý M1 thành công
  Gọi DeleteMessage → mạng lỗi, timeout
  M1 visible lại → Consumer C nhận và xử lý lại → XỬ LÝ TRÙNG

Tình huống 3: Distributed consumer race condition
  Consumer A và B cùng nhận M1 (trong cùng window ngắn)
  Cả hai cùng bắt đầu xử lý → XỬ LÝ TRÙNG
```

### Chiến Lược Implement Idempotency

#### Chiến Lược 1: Idempotency Key Trong Database

```python
import boto3
import json
from datetime import datetime
import psycopg2  # PostgreSQL

sqs = boto3.client('sqs', region_name='us-east-1')

def process_order_idempotent(message: dict):
    """
    Xử lý đơn hàng với idempotency dùng PostgreSQL.
    INSERT ... ON CONFLICT DO NOTHING đảm bảo chỉ tạo 1 lần.
    """
    message_id = message['MessageId']
    body = json.loads(message['Body'])
    order_id = body['orderId']
    
    conn = psycopg2.connect("postgresql://localhost/mydb")
    cursor = conn.cursor()
    
    try:
        # Thử insert với idempotency key
        cursor.execute("""
            INSERT INTO processed_messages (message_id, order_id, processed_at)
            VALUES (%s, %s, %s)
            ON CONFLICT (message_id) DO NOTHING
            RETURNING message_id
        """, (message_id, order_id, datetime.utcnow()))
        
        result = cursor.fetchone()
        
        if result is None:
            # message_id đã tồn tại → duplicate, bỏ qua
            print(f"Duplicate message {message_id} — skipping")
            conn.rollback()
            return False
        
        # Xử lý đơn hàng thực sự
        cursor.execute("""
            INSERT INTO orders (order_id, status, created_at)
            VALUES (%s, 'PENDING', %s)
        """, (order_id, datetime.utcnow()))
        
        conn.commit()
        print(f"Order {order_id} created successfully")
        return True
        
    except Exception:
        conn.rollback()
        raise
    finally:
        cursor.close()
        conn.close()
```

#### Chiến Lược 2: DynamoDB Conditional Write

```python
import boto3
import json
from datetime import datetime, timedelta
from botocore.exceptions import ClientError

sqs = boto3.client('sqs', region_name='us-east-1')
dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
table = dynamodb.Table('idempotency-keys')

def is_already_processed(message_id: str, ttl_hours: int = 24) -> bool:
    """
    Kiểm tra và đánh dấu message đã xử lý dùng DynamoDB.
    Trả về True nếu đã xử lý (duplicate).
    """
    ttl = int((datetime.utcnow() + timedelta(hours=ttl_hours)).timestamp())
    
    try:
        table.put_item(
            Item={
                "messageId": message_id,
                "processedAt": datetime.utcnow().isoformat(),
                "ttl": ttl
            },
            # Chỉ insert nếu chưa tồn tại
            ConditionExpression="attribute_not_exists(messageId)"
        )
        return False  # Chưa xử lý → tiếp tục
        
    except ClientError as e:
        if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
            return True  # Đã xử lý → duplicate
        raise

def consumer(queue_url: str):
    while True:
        response = sqs.receive_message(
            QueueUrl=queue_url,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20,
            VisibilityTimeout=60
        )
        
        for message in response.get('Messages', []):
            message_id = message['MessageId']
            
            # Kiểm tra duplicate trước
            if is_already_processed(message_id):
                print(f"Skipping duplicate: {message_id}")
                # Vẫn phải delete để remove khỏi queue
                sqs.delete_message(
                    QueueUrl=queue_url,
                    ReceiptHandle=message['ReceiptHandle']
                )
                continue
            
            try:
                # Xử lý message
                body = json.loads(message['Body'])
                handle_business_logic(body)
                
                # Xóa message sau khi xử lý thành công
                sqs.delete_message(
                    QueueUrl=queue_url,
                    ReceiptHandle=message['ReceiptHandle']
                )
                print(f"Processed: {message_id}")
                
            except Exception as e:
                print(f"Error processing {message_id}: {e}")
                # Không xóa → visibility timeout hết → visible lại → retry
```

#### Chiến Lược 3: Redis SETNX (Set If Not Exists)

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379, db=0)

def process_with_redis_idempotency(message: dict):
    message_id = message['MessageId']
    
    # SETNX: chỉ set nếu key chưa tồn tại
    # EX 86400: tự xóa sau 24 giờ
    if not r.set(f"processed:{message_id}", "1", ex=86400, nx=True):
        print(f"Duplicate {message_id} — skipping")
        return
    
    body = json.loads(message['Body'])
    handle_business_logic(body)
    print(f"Processed {message_id}")
```

---

## 📊 Bảng So Sánh Các Chiến Lược Idempotency

| Chiến Lược | Ưu Điểm | Nhược Điểm | Khi Nào Dùng |
|---|---|---|---|
| **PostgreSQL ON CONFLICT** | Atomic với business logic | Cần DB transaction | Khi đã dùng SQL DB |
| **DynamoDB Conditional** | Fully managed, TTL tự động | Thêm chi phí DynamoDB | Serverless architecture |
| **Redis SETNX** | Cực nhanh, <1ms | Mất data khi Redis crash | Cache layer đã có Redis |
| **FIFO Queue** | Built-in deduplication | Throughput giới hạn | Cần exactly-once |

---

## ❓ Câu Hỏi Phỏng Vấn

### Q: Visibility Timeout bao nhiêu là hợp lý?

**A:** Quy tắc: đặt bằng **6 lần** thời gian xử lý trung bình. Ví dụ, nếu xử lý trung bình 10 giây, đặt timeout = 60 giây. Lý do nhân 6: tránh false timeout khi hệ thống bận, đồng thời không để tin nhắn lỗi chờ quá lâu trước khi retry.

### Q: Nếu không delete message thì sao?

**A:** Tin nhắn trở nên **visible lại** sau khi Visibility Timeout hết và sẽ được consumer khác nhận. Sau `maxReceiveCount` lần thất bại, tin nhắn được chuyển sang **DLQ** (Dead Letter Queue — Hàng Đợi Thư Chết).

### Q: Có thể xóa message trước khi xử lý không?

**A:** Kỹ thuật này gọi là **delete-then-process**. Nguy hiểm vì nếu xử lý thất bại, tin nhắn bị mất vĩnh viễn. Chỉ dùng khi chấp nhận được mất tin nhắn và quan trọng nhất là tránh duplicate hơn là tránh mất.

### Q: ReceiptHandle là gì và tại sao thay đổi?

**A:** `ReceiptHandle` là token tạm thời, thay đổi mỗi lần receive. Nó dùng để delete hoặc change visibility của tin nhắn. Cùng `MessageId` nhưng mỗi lần receive lại có `ReceiptHandle` khác nhau — đây là cơ chế bảo mật của SQS.

---

## 🗺️ Điều Hướng

- ◀ Quay lại: [1-standard-vs-fifo.md](./1-standard-vs-fifo.md)
- ▶ Tiếp theo: [3-dead-letter-queue.md](./3-dead-letter-queue.md) — DLQ: Hàng Đợi Thư Chết
