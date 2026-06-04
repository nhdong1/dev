# Fan-out Pattern với SNS + SQS

> Fan-out (Khuếch Tán) là pattern quan trọng nhất trong AWS messaging: một sự kiện kích hoạt nhiều luồng xử lý song song, độc lập, không mất dữ liệu.

## 📌 Tóm Tắt Nhanh

| Thành Phần | Vai Trò |
|---|---|
| **SNS Topic** | Điểm phát sóng trung tâm — nhận một tin, phân phối đến nhiều nơi |
| **SQS Queue** | Buffer (đệm) bền vững cho từng consumer — không mất tin nếu consumer lỗi |
| **Lambda** | Consumer thời gian thực — không cần queue |
| **Filter Policy** | Điều hướng tin nhắn đúng subscriber, không bắt buộc |

---

## 🎯 Fan-out Là Gì?

**Fan-out** nghĩa là một tin nhắn publish một lần, SNS **đồng thời** giao vận đến nhiều subscriber. Mỗi subscriber nhận **bản sao riêng** của tin nhắn và xử lý hoàn toàn độc lập.

```
                         ┌──▶ [SQS Queue A] ──▶ [Consumer A]
                         │
[Publisher] ──1 publish──▶ [SNS Topic] ──▶ [SQS Queue B] ──▶ [Consumer B]
                         │
                         ├──▶ [SQS Queue C] ──▶ [Consumer C]
                         │
                         └──▶ [Lambda D]    ──▶ [Consumer D]
```

**Điểm mấu chốt:**
- Consumer A, B, C, D đều nhận **cùng tin nhắn** nhưng xử lý **độc lập**
- Nếu Consumer B bị lỗi, Consumer A vẫn xử lý bình thường
- Không cần Publisher biết có bao nhiêu consumer đang lắng nghe

---

## 🏗️ Tại Sao Kết Hợp SNS + SQS?

### Chỉ Dùng SNS (Không Có SQS)

```
[Publisher] ──▶ [SNS Topic] ──▶ [Consumer A — Lambda]
                            └──▶ [Consumer B — HTTP Endpoint]
```

**Vấn đề:**
- Nếu Lambda bị throttle (giới hạn) hoặc HTTP endpoint timeout → tin nhắn **mất**
- SNS retry có giới hạn, sau đó bỏ qua
- Consumer phải sẵn sàng **ngay lập tức**

### SNS + SQS (Best Practice)

```
[Publisher] ──▶ [SNS Topic] ──▶ [SQS Queue A] ──▶ [Consumer A]
                            └──▶ [SQS Queue B] ──▶ [Consumer B]
```

**Lợi ích:**
- SQS **lưu tin nhắn** nếu consumer chưa sẵn sàng — không mất dữ liệu
- Consumer xử lý theo **tốc độ riêng** — không bị ép tốc độ của publisher
- SQS quản lý **retry** tự động cho consumer
- DLQ (Dead Letter Queue — Hàng Đợi Thư Chết) catch tin nhắn xử lý thất bại

---

## 🏭 Ví Dụ Thực Tế — E-commerce Order Processing

### Kiến Trúc Đầy Đủ

```
[Order API]
    │ POST /orders
    │ → Tạo order trong DB
    │ → Publish "order.created"
    ▼
[SNS: order-events]
    │
    ├──▶ [SQS: payment-queue]      [DLQ: payment-dlq]
    │         │
    │         ▼
    │    [Payment Service]
    │    - Charge thẻ tín dụng
    │    - Cập nhật trạng thái thanh toán
    │
    ├──▶ [SQS: inventory-queue]    [DLQ: inventory-dlq]
    │         │
    │         ▼
    │    [Inventory Service]
    │    - Reserve stock (giữ hàng)
    │    - Cập nhật số lượng tồn kho
    │
    ├──▶ [SQS: notification-queue] [DLQ: notification-dlq]
    │         │
    │         ▼
    │    [Notification Service]
    │    - Gửi email xác nhận đơn hàng
    │    - Gửi SMS thông báo
    │
    └──▶ [SQS: analytics-queue]    [DLQ: analytics-dlq]
              │
              ▼
         [Analytics Service]
         - Ghi nhận sự kiện đặt hàng
         - Cập nhật dashboard real-time
```

### Ưu Điểm Của Kiến Trúc Này

1. **Isolation (Cách Ly):** Lỗi ở Inventory Service không ảnh hưởng đến Payment Service
2. **Independent Scaling (Mở Rộng Độc Lập):** Tăng số worker Payment mà không ảnh hưởng Notification
3. **Durability (Độ Bền):** Tin nhắn lưu trong SQS, consumer có thể restart thoải mái
4. **Retry Logic:** SQS tự retry khi consumer báo lỗi (không xóa tin nhắn)
5. **Dead Letter Queue:** Tin nhắn thất bại nhiều lần → DLQ để phân tích

---

## 🛠️ Triển Khai Từng Bước

### Bước 1: Tạo SNS Topic và SQS Queues

```bash
# Tạo SNS topic
TOPIC_ARN=$(aws sns create-topic \
  --name order-events \
  --query "TopicArn" --output text)

echo "Topic ARN: $TOPIC_ARN"

# Tạo SQS queues với DLQ
aws sqs create-queue --queue-name payment-dlq
aws sqs create-queue \
  --queue-name payment-queue \
  --attributes '{
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:123456:payment-dlq\",\"maxReceiveCount\":\"3\"}"
  }'

aws sqs create-queue --queue-name inventory-dlq
aws sqs create-queue \
  --queue-name inventory-queue \
  --attributes '{
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:123456:inventory-dlq\",\"maxReceiveCount\":\"3\"}"
  }'
```

### Bước 2: Cấu Hình SQS Queue Policy (Cho Phép SNS Gửi)

```bash
PAYMENT_QUEUE_URL=$(aws sqs get-queue-url \
  --queue-name payment-queue --query "QueueUrl" --output text)

PAYMENT_QUEUE_ARN=$(aws sqs get-queue-attributes \
  --queue-url $PAYMENT_QUEUE_URL \
  --attribute-names QueueArn \
  --query "Attributes.QueueArn" --output text)

# Đặt queue policy cho phép SNS topic gửi tin
aws sqs set-queue-attributes \
  --queue-url $PAYMENT_QUEUE_URL \
  --attributes "{
    \"Policy\": \"{
      \\\"Version\\\": \\\"2012-10-17\\\",
      \\\"Statement\\\": [{
        \\\"Effect\\\": \\\"Allow\\\",
        \\\"Principal\\\": {\\\"Service\\\": \\\"sns.amazonaws.com\\\"},
        \\\"Action\\\": \\\"sqs:SendMessage\\\",
        \\\"Resource\\\": \\\"$PAYMENT_QUEUE_ARN\\\",
        \\\"Condition\\\": {
          \\\"ArnEquals\\\": {\\\"aws:SourceArn\\\": \\\"$TOPIC_ARN\\\"}
        }
      }]
    }\"
  }"
```

### Bước 3: Subscribe Queues Vào Topic

```bash
# Subscribe payment queue (nhận tất cả events)
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol sqs \
  --notification-endpoint $PAYMENT_QUEUE_ARN \
  --attributes '{"RawMessageDelivery":"true"}'

# Subscribe inventory queue (chỉ nhận order.created và order.cancelled)
INVENTORY_QUEUE_ARN="arn:aws:sqs:us-east-1:123456:inventory-queue"
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol sqs \
  --notification-endpoint $INVENTORY_QUEUE_ARN \
  --attributes '{
    "RawMessageDelivery":"true",
    "FilterPolicy":"{\"eventType\":[\"order.created\",\"order.cancelled\"]}"
  }'
```

### Bước 4: Publisher Code

```python
import boto3
import json
from datetime import datetime
from dataclasses import dataclass, asdict
from enum import Enum

class OrderEventType(str, Enum):
    CREATED = "order.created"
    PAID = "order.paid"
    SHIPPED = "order.shipped"
    DELIVERED = "order.delivered"
    CANCELLED = "order.cancelled"
    REFUNDED = "order.refunded"

@dataclass
class OrderEvent:
    event_id: str
    event_type: str
    order_id: str
    customer_id: str
    amount: float
    currency: str
    order_type: str    # "standard" | "express"
    region: str
    timestamp: str

class OrderEventBus:
    """Publisher trung tâm cho tất cả order events."""

    def __init__(self, topic_arn: str, region: str = "us-east-1"):
        self.sns = boto3.client("sns", region_name=region)
        self.topic_arn = topic_arn

    def publish(self, event: OrderEvent) -> str:
        response = self.sns.publish(
            TopicArn=self.topic_arn,
            Message=json.dumps(asdict(event)),
            MessageAttributes={
                "eventType": {
                    "DataType": "String",
                    "StringValue": event.event_type
                },
                "orderType": {
                    "DataType": "String",
                    "StringValue": event.order_type
                },
                "orderAmount": {
                    "DataType": "Number",
                    "StringValue": str(event.amount)
                },
                "region": {
                    "DataType": "String",
                    "StringValue": event.region
                }
            }
        )
        return response["MessageId"]

    def order_created(self, order_id: str, customer_id: str,
                      amount: float, order_type: str, region: str) -> str:
        import uuid
        event = OrderEvent(
            event_id=str(uuid.uuid4()),
            event_type=OrderEventType.CREATED,
            order_id=order_id,
            customer_id=customer_id,
            amount=amount,
            currency="USD",
            order_type=order_type,
            region=region,
            timestamp=datetime.utcnow().isoformat()
        )
        msg_id = self.publish(event)
        print(f"Published {event.event_type} for {order_id}: {msg_id}")
        return msg_id
```

### Bước 5: Consumer Code (mỗi service)

```python
import boto3
import json

sqs = boto3.client("sqs", region_name="us-east-1")
PAYMENT_QUEUE_URL = "https://sqs.us-east-1.amazonaws.com/123456/payment-queue"

def payment_consumer():
    """Consumer của Payment Service — xử lý thanh toán từ queue."""
    print("Payment Service đang lắng nghe...")

    while True:
        response = sqs.receive_message(
            QueueUrl=PAYMENT_QUEUE_URL,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20,   # Long polling (thăm dò dài)
            AttributeNames=["All"]
        )

        for message in response.get("Messages", []):
            receipt_handle = message["ReceiptHandle"]
            message_id = message["MessageId"]

            try:
                # RawMessageDelivery=true → body là JSON trực tiếp
                payload = json.loads(message["Body"])

                if payload.get("event_type") == "order.created":
                    process_payment(payload)
                elif payload.get("event_type") == "order.refunded":
                    process_refund(payload)

                # Xử lý thành công → xóa khỏi queue
                sqs.delete_message(
                    QueueUrl=PAYMENT_QUEUE_URL,
                    ReceiptHandle=receipt_handle
                )
                print(f"Processed {message_id}")

            except Exception as e:
                # KHÔNG xóa → tin nhắn visible lại sau Visibility Timeout
                # Sau maxReceiveCount lần → tự động vào DLQ
                print(f"Error processing {message_id}: {e}")

def process_payment(payload: dict):
    order_id = payload["order_id"]
    amount = payload["amount"]
    print(f"Charging ${amount} for order {order_id}")
    # Gọi payment gateway...

def process_refund(payload: dict):
    order_id = payload["order_id"]
    print(f"Refunding order {order_id}")
    # Gọi refund API...
```

---

## 🔀 Fan-out Với Filtering

### Định Tuyến Theo Loại Event

```
[SNS: order-events]
    │
    ├──▶ [SQS: order-lifecycle-queue]
    │    Filter: {"eventType":["order.created","order.paid","order.shipped","order.delivered","order.cancelled"]}
    │    Consumer: Order State Machine
    │
    ├──▶ [SQS: payment-only-queue]
    │    Filter: {"eventType":["order.paid","order.refunded"]}
    │    Consumer: Finance & Accounting Service
    │
    └──▶ [SQS: analytics-queue]
         No filter — nhận tất cả events
         Consumer: Analytics & Reporting
```

### Định Tuyến Theo Region

```
[SNS: global-order-events]
    │
    ├──▶ [SQS: vn-orders]
    │    Filter: {"region":["VN"]}
    │    Consumer: Vietnam fulfillment center
    │
    ├──▶ [SQS: sg-orders]
    │    Filter: {"region":["SG","MY","TH"]}
    │    Consumer: SEA fulfillment center
    │
    └──▶ [SQS: us-orders]
         Filter: {"region":["US","CA"]}
         Consumer: North America fulfillment center
```

---

## ⚡ Fan-out Với Lambda (Không Qua SQS)

Đôi khi không cần SQS nếu consumer phải xử lý **ngay lập tức** và có thể chấp nhận ít retry hơn:

```
[SNS Topic]
    │
    ├──▶ [SQS: durable-processing]   ← Xử lý bền vững, có thể chậm
    │         └──▶ [Lambda / ECS]
    │
    └──▶ [Lambda: realtime-notify]   ← Xử lý ngay, gửi push notification
              ← SNS retry 2 lần nếu Lambda throttle
              ← Không có DLQ tự động (phải cấu hình async destination)
```

**Cấu hình Lambda Destination cho SNS failure:**

```python
# Nếu Lambda invocation từ SNS thất bại sau retry
# Cấu hình Async Event Destination trong Lambda để gửi đến DLQ

import boto3

lambda_client = boto3.client("lambda", region_name="us-east-1")
lambda_client.put_function_event_invoke_config(
    FunctionName="realtime-notify",
    DestinationConfig={
        "OnFailure": {
            "Destination": "arn:aws:sqs:us-east-1:123456:lambda-failures-dlq"
        }
    }
)
```

---

## 📊 So Sánh Các Lựa Chọn Fan-out

| Pattern | Durability (Độ Bền) | Retry | Throughput | Độ Phức Tạp | Phù Hợp |
|---|---|---|---|---|---|
| **SNS → SQS → Consumer** | ✅ Cao | ✅ SQS tự quản lý | Cao | Trung bình | Production workloads |
| **SNS → Lambda** | ⚠️ Trung bình | ⚠️ 2 lần | Cao | Thấp | Real-time, stateless |
| **SNS → HTTP/S** | ❌ Thấp | ⚠️ Có giới hạn | Trung bình | Thấp | Webhooks |
| **Chỉ SQS (không SNS)** | ✅ Cao | ✅ | Cao | Thấp | Single consumer |

---

## 💡 Best Practices (Thực Hành Tốt Nhất)

### 1. Luôn Dùng DLQ Cho SQS

```bash
# maxReceiveCount = 3: sau 3 lần thất bại → vào DLQ
aws sqs create-queue \
  --queue-name my-dlq \
  --attributes MessageRetentionPeriod=1209600  # 14 ngày

aws sqs create-queue \
  --queue-name my-queue \
  --attributes '{
    "RedrivePolicy":"{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:123456:my-dlq\",\"maxReceiveCount\":\"3\"}"
  }'
```

### 2. Bật RawMessageDelivery

```bash
# Tránh phải parse JSON lồng nhau ở consumer
aws sns set-subscription-attributes \
  --subscription-arn <sub-arn> \
  --attribute-name RawMessageDelivery \
  --attribute-value true
```

### 3. Consumer Phải Idempotent (Bất Biến)

```python
def process_order_with_idempotency(order_id: str, message_id: str):
    """
    SQS Standard: có thể nhận tin trùng lặp (at-least-once).
    Phải kiểm tra đã xử lý chưa trước khi thực hiện.
    """
    dynamodb = boto3.resource("dynamodb")
    table = dynamodb.Table("processed-orders")

    try:
        # Dùng conditional write: chỉ insert nếu chưa tồn tại
        table.put_item(
            Item={"messageId": message_id, "orderId": order_id},
            ConditionExpression="attribute_not_exists(messageId)"
        )
    except Exception as e:
        if "ConditionalCheckFailed" in str(e):
            print(f"Đã xử lý order {order_id} trước đó — bỏ qua")
            return
        raise

    # Xử lý thực sự
    charge_customer(order_id)
```

### 4. Visibility Timeout > Processing Time

Đặt `VisibilityTimeout` lớn hơn thời gian xử lý dự kiến:

```bash
# Nếu consumer mất trung bình 30 giây, đặt timeout 90 giây (3x)
aws sqs set-queue-attributes \
  --queue-url $QUEUE_URL \
  --attributes '{"VisibilityTimeout":"90"}'
```

---

## 🏗️ Pattern Nâng Cao: Conditional Fan-out

Kết hợp fan-out với multiple SNS topics:

```
[Order Service]
    │
    ├──▶ [SNS: order-created-topic]   ← Chỉ publish khi tạo đơn
    │    └──▶ [SQS: payment-queue]
    │    └──▶ [SQS: inventory-queue]
    │
    ├──▶ [SNS: order-status-topic]    ← Publish khi trạng thái thay đổi
    │    └──▶ [SQS: notification-queue]
    │    └──▶ [SQS: analytics-queue]
    │
    └──▶ [SNS: order-cancelled-topic] ← Chỉ publish khi hủy đơn
         └──▶ [SQS: refund-queue]
         └──▶ [SQS: restock-queue]
```

**Khi nào dùng nhiều topic thay vì một topic + filter?**

| Cách | Dùng Khi |
|---|---|
| **Một topic + Filter** | Các event tương tự, subscriber dễ dàng filter theo attributes |
| **Nhiều topic** | Event hoàn toàn khác nhau, hoặc muốn kiểm soát IAM riêng per event type |

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

### Q: Fan-out pattern giải quyết vấn đề gì?

**A:** Giải quyết bài toán "một sự kiện, nhiều hành động": khi đơn hàng được tạo, cần đồng thời xử lý thanh toán, cập nhật kho, gửi email, và ghi analytics. Fan-out cho phép các action này xảy ra song song, độc lập — lỗi một service không ảnh hưởng service khác.

### Q: Tại sao cần SQS trong fan-out, không dùng SNS trực tiếp đến Lambda?

**A:** SQS cung cấp **durability** (độ bền) và **buffering**: nếu consumer bị lỗi hoặc quá tải, tin nhắn không mất — nằm chờ trong SQS. SNS → Lambda không có buffer, nếu Lambda throttle sau 2 lần retry SNS sẽ bỏ qua tin nhắn (hoặc vào DLQ của Lambda).

### Q: Làm sao đảm bảo mỗi SQS queue chỉ nhận event liên quan?

**A:** Dùng **SNS Message Filtering** — đặt Filter Policy trên subscription. Ví dụ: inventory queue chỉ cần nhận `order.created` và `order.cancelled`, không cần `order.paid` hay `order.shipped`.

### Q: Consumer có thể nhận tin trùng không trong fan-out?

**A:** Có, nếu dùng **SQS Standard Queue** (at-least-once delivery — giao vận ít nhất một lần). Giải pháp: consumer phải viết **idempotent logic** — xử lý cùng tin nhiều lần phải cho kết quả như nhau. Nếu cần exactly-once, dùng SQS FIFO + SNS FIFO.

---

## 🗺️ Điều Hướng

- ◀ Quay lại: [2-message-filtering.md](./2-message-filtering.md) — Message Filtering
- ▶ Tiếp theo: [4-fifo-topic.md](./4-fifo-topic.md) — FIFO Topic
