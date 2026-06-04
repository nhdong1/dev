# Topic, Subscription, Protocol — Nền Tảng SNS

> Hiểu rõ cách SNS Topic, Subscription và các Protocol hoạt động là nền tảng để xây dựng kiến trúc thông báo đúng đắn.

## 📌 Tóm Tắt Nhanh

| Thành Phần | Vai Trò | Ví Dụ |
|---|---|---|
| **Topic** (Chủ Đề) | Kênh phân phối trung tâm | `arn:aws:sns:us-east-1:123:order-events` |
| **Subscription** (Đăng Ký) | Liên kết topic với endpoint | SQS Queue đăng ký nhận từ topic |
| **Publisher** (Nhà Phát) | Gửi tin nhắn vào topic | Order Service publish "order.created" |
| **Subscriber** (Người Đăng Ký) | Nhận tin nhắn từ topic | Lambda, SQS, Email... |
| **Protocol** (Giao Thức) | Cách giao vận | `sqs`, `lambda`, `https`, `email`, `sms` |

---

## 📂 Topic — Kênh Phân Phối Trung Tâm

### Topic Là Gì?

**Topic** là điểm trung tâm trong SNS. Publisher phát tin vào topic, SNS tự động chuyển tiếp đến mọi subscriber đã đăng ký.

```
Topic ARN (Amazon Resource Name — Tên Tài Nguyên Amazon):
arn:aws:sns:{region}:{account-id}:{topic-name}
arn:aws:sns:us-east-1:123456789012:order-events
```

### Tạo Topic

```bash
# Standard Topic
aws sns create-topic \
  --name order-events \
  --attributes DisplayName="Order Events",KmsMasterKeyId=alias/aws/sns

# FIFO Topic (kết thúc bằng .fifo)
aws sns create-topic \
  --name order-events.fifo \
  --attributes FifoTopic=true,ContentBasedDeduplication=false

# Lấy thông tin topic
aws sns get-topic-attributes \
  --topic-arn arn:aws:sns:us-east-1:123456789:order-events
```

### Topic Attributes (Thuộc Tính Topic)

| Thuộc Tính | Mô Tả | Ví Dụ |
|---|---|---|
| `TopicArn` | ARN định danh duy nhất | `arn:aws:sns:...` |
| `DisplayName` | Tên hiển thị (dùng trong email subject) | "Order Notifications" |
| `SubscriptionsConfirmed` | Số subscription đã xác nhận | `5` |
| `SubscriptionsPending` | Số subscription chờ xác nhận | `1` |
| `KmsMasterKeyId` | Key KMS để mã hóa | `alias/aws/sns` |
| `FifoTopic` | Có phải FIFO topic không | `true` / `false` |
| `ContentBasedDeduplication` | Tự động khử trùng theo nội dung | `true` / `false` |

---

## 🔗 Subscription — Đăng Ký Nhận Tin

### Subscription Là Gì?

**Subscription** là liên kết giữa một topic và một endpoint. Khi topic nhận tin nhắn mới, SNS giao vận đến tất cả subscription đang `Confirmed` (Đã Xác Nhận).

### Trạng Thái Subscription

```
Tạo Subscription
      │
      ▼
PendingConfirmation (Chờ Xác Nhận)
      │
      │── [Endpoint tự xác nhận — HTTP/Email]
      │        hoặc
      │── [Tự động xác nhận — SQS, Lambda]
      ▼
Confirmed (Đã Xác Nhận) ──▶ Nhận tin nhắn bình thường
      │
      │── [Hết hạn / Unsubscribe]
      ▼
Deleted (Đã Xóa)
```

**Lưu ý quan trọng:**
- **Email/HTTP/HTTPS** — subscriber phải click link xác nhận trong email/request
- **SQS/Lambda** — SNS tự động xác nhận ngay

### Tạo Subscription

```bash
# Subscribe SQS Queue
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456:order-events \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:123456:order-processing-queue

# Subscribe Lambda Function
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456:order-events \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:us-east-1:123456:function:process-order

# Subscribe HTTPS endpoint
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456:order-events \
  --protocol https \
  --notification-endpoint https://api.example.com/webhooks/orders

# Subscribe Email
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456:order-events \
  --protocol email \
  --notification-endpoint ops-team@example.com

# Subscribe với filter policy (chính sách lọc)
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456:order-events \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:123456:express-queue \
  --attributes '{"FilterPolicy":"{\"orderType\":[\"express\"]}"}'
```

---

## 📡 Protocol — Giao Thức Giao Vận

SNS hỗ trợ nhiều protocol (giao thức) khác nhau, mỗi loại phù hợp cho một use case riêng:

### 1. SQS — Amazon Simple Queue Service

```
SNS Topic ──▶ SQS Queue ──▶ Consumer

Đặc điểm:
- Durable (bền) — tin nhắn lưu trong SQS nếu consumer chưa sẵn sàng
- Retry handled by SQS — SQS tự quản lý retry
- Phù hợp: Fan-out pattern, background processing
```

**Cấu hình SQS Queue Policy** (bắt buộc để SNS có quyền gửi):

```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "sns.amazonaws.com"
  },
  "Action": "sqs:SendMessage",
  "Resource": "arn:aws:sqs:us-east-1:123456789:order-processing-queue",
  "Condition": {
    "ArnEquals": {
      "aws:SourceArn": "arn:aws:sns:us-east-1:123456789:order-events"
    }
  }
}
```

### 2. Lambda — AWS Lambda Function

```
SNS Topic ──▶ Lambda Function (xử lý ngay, không qua queue)

Đặc điểm:
- Synchronous invocation (gọi đồng bộ) — SNS chờ Lambda trả kết quả
- Retry 2 lần nếu Lambda throttle hoặc lỗi
- Phù hợp: Real-time processing, transformation, routing
```

**Cấu hình Lambda Permission** (bắt buộc):

```bash
aws lambda add-permission \
  --function-name process-order \
  --statement-id sns-invoke \
  --action lambda:InvokeFunction \
  --principal sns.amazonaws.com \
  --source-arn arn:aws:sns:us-east-1:123456:order-events
```

### 3. HTTPS — HTTP/HTTPS Endpoint

```
SNS Topic ──POST──▶ https://api.example.com/webhook

Đặc điểm:
- SNS gửi HTTP POST với JSON payload
- Endpoint phải trả HTTP 200 trong thời gian quy định
- Phải xác nhận subscription qua SubscribeURL
- Phù hợp: Third-party webhooks, custom endpoints
```

**Payload HTTP Nhận Từ SNS:**

```json
{
  "Type": "Notification",
  "MessageId": "abc-123-def",
  "TopicArn": "arn:aws:sns:us-east-1:123456:order-events",
  "Subject": "New Order",
  "Message": "{\"orderId\":\"ORD-001\",\"amount\":100}",
  "Timestamp": "2026-05-18T10:00:00.000Z",
  "SignatureVersion": "1",
  "Signature": "...",
  "SigningCertURL": "https://sns.amazonaws.com/...",
  "UnsubscribeURL": "https://sns.amazonaws.com/...",
  "MessageAttributes": {
    "orderType": {
      "Type": "String",
      "Value": "express"
    }
  }
}
```

**Xác Minh Signature (Chữ Ký) — Quan Trọng Cho Bảo Mật:**

```python
import boto3
import json
import base64
import urllib.request
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import padding
from cryptography.x509 import load_pem_x509_certificate

def verify_sns_signature(notification: dict) -> bool:
    """Xác minh chữ ký SNS để tránh request giả mạo."""
    # Tải certificate từ AWS
    cert_url = notification["SigningCertURL"]
    assert cert_url.startswith("https://sns.") and ".amazonaws.com/" in cert_url

    with urllib.request.urlopen(cert_url) as response:
        cert_data = response.read()

    cert = load_pem_x509_certificate(cert_data)
    public_key = cert.public_key()

    # Xây dựng chuỗi để xác minh
    message_to_verify = (
        f"Message\n{notification['Message']}\n"
        f"MessageId\n{notification['MessageId']}\n"
        f"Subject\n{notification.get('Subject', '')}\n"
        f"Timestamp\n{notification['Timestamp']}\n"
        f"TopicArn\n{notification['TopicArn']}\n"
        f"Type\n{notification['Type']}\n"
    )

    signature = base64.b64decode(notification["Signature"])

    try:
        public_key.verify(signature, message_to_verify.encode(), padding.PKCS1v15(), hashes.SHA1())
        return True
    except Exception:
        return False
```

### 4. Email / Email-JSON

```
SNS Topic ──▶ Email Inbox (Hộp Thư Đến)

Đặc điểm:
- email: nội dung tin nhắn raw (thô)
- email-json: toàn bộ JSON payload (bao gồm metadata)
- Subscription phải được xác nhận qua click link email
- Phù hợp: Alert cho operations team, thông báo đơn giản

Giới hạn: Không dùng cho automation (tự động hóa) vì phải xác nhận bằng tay
```

### 5. SMS — Short Message Service (Tin Nhắn Ngắn)

```
SNS Topic ──▶ Điện Thoại Di Động (Tin Nhắn SMS)

Đặc điểm:
- Hỗ trợ hơn 200 quốc gia
- Transactional SMS (SMS Giao Dịch): OTP, xác nhận đơn hàng — ưu tiên cao
- Promotional SMS (SMS Quảng Cáo): marketing — ưu tiên thấp hơn
- Chi phí theo từng tin nhắn và quốc gia

Lưu ý: SNS SMS không có DLQ — tin nhắn thất bại sẽ mất.
```

**Gửi SMS Trực Tiếp (Không Qua Topic):**

```python
import boto3

sns = boto3.client("sns", region_name="us-east-1")

def send_otp_sms(phone_number: str, otp: str):
    """Gửi OTP qua SMS, không cần topic."""
    response = sns.publish(
        PhoneNumber=phone_number,  # E.164 format: +84912345678
        Message=f"Mã OTP của bạn là: {otp}. Có hiệu lực trong 5 phút.",
        MessageAttributes={
            "AWS.SNS.SMS.SMSType": {
                "DataType": "String",
                "StringValue": "Transactional"  # Ưu tiên cao cho OTP
            },
            "AWS.SNS.SMS.SenderID": {
                "DataType": "String",
                "StringValue": "MyApp"  # Tên người gửi (không hỗ trợ mọi quốc gia)
            }
        }
    )
    return response["MessageId"]
```

### 6. Mobile Push (Thông Báo Đẩy Di Động)

```
SNS Topic ──▶ [Amazon SNS Mobile Push]
                    │
                    ├──▶ APNs (Apple Push Notification service — Dịch Vụ Thông Báo Đẩy Apple)
                    ├──▶ FCM (Firebase Cloud Messaging — Nhắn Tin Đám Mây Firebase/Google)
                    ├──▶ ADM (Amazon Device Messaging — Nhắn Tin Thiết Bị Amazon)
                    └──▶ Baidu Cloud Push (cho thị trường Trung Quốc)
```

**Luồng Mobile Push:**

```
1. App đăng ký với APNs/FCM → nhận device token
2. App gửi device token lên server
3. Server đăng ký device token với SNS → nhận SNS Endpoint ARN
4. SNS Endpoint ARN subscribe vào topic
5. Khi publish vào topic → SNS gửi push notification đến device
```

---

## 🔄 Message Structure (Cấu Trúc Tin Nhắn)

### Publish Simple Message (Tin Nhắn Đơn Giản)

```python
import boto3
import json

sns = boto3.client("sns", region_name="us-east-1")

sns.publish(
    TopicArn="arn:aws:sns:us-east-1:123456:order-events",
    Message=json.dumps({
        "orderId": "ORD-001",
        "customerId": "CUST-123",
        "amount": 299.99,
        "status": "created"
    }),
    Subject="New Order Created"
)
```

### Message Structure Per Protocol (Cấu Trúc Khác Nhau Theo Protocol)

SNS cho phép gửi nội dung **khác nhau** cho từng protocol trong **một lần publish**:

```python
sns.publish(
    TopicArn="arn:aws:sns:us-east-1:123456:order-events",
    MessageStructure="json",  # Bắt buộc khi dùng per-protocol message
    Message=json.dumps({
        "default": "Đơn hàng ORD-001 đã được tạo",         # Fallback
        "email": "Kính gửi khách hàng, đơn hàng #ORD-001 của bạn đã được xác nhận...",
        "sms": "DH ORD-001 da xac nhan. Giao trong 2-3 ngay.",
        "sqs": json.dumps({                                 # JSON cho SQS consumer
            "orderId": "ORD-001",
            "customerId": "CUST-123",
            "amount": 299.99,
            "status": "created",
            "timestamp": "2026-05-18T10:00:00Z"
        }),
        "lambda": json.dumps({                              # JSON cho Lambda
            "eventType": "ORDER_CREATED",
            "orderId": "ORD-001",
            "priority": "high"
        })
    })
)
```

### Message Attributes (Thuộc Tính Tin Nhắn)

Message Attributes dùng để **filter** (lọc) subscription — subscriber chỉ nhận tin phù hợp:

```python
sns.publish(
    TopicArn="arn:aws:sns:us-east-1:123456:order-events",
    Message=json.dumps({"orderId": "ORD-001"}),
    MessageAttributes={
        "orderType": {
            "DataType": "String",
            "StringValue": "express"       # Giá trị String
        },
        "orderAmount": {
            "DataType": "Number",
            "StringValue": "299.99"        # Số phải là StringValue nhưng DataType là Number
        },
        "isPremiumCustomer": {
            "DataType": "String",
            "StringValue": "true"
        },
        "tags": {
            "DataType": "String.Array",
            "StringValue": '["electronics", "express", "gift"]'
        }
    }
)
```

---

## 🛠️ Ví Dụ Code Thực Tế — Publisher/Subscriber

### Publisher Service (Dịch Vụ Phát Sóng)

```python
import boto3
import json
from dataclasses import dataclass, asdict
from datetime import datetime

@dataclass
class OrderEvent:
    order_id: str
    customer_id: str
    amount: float
    order_type: str   # "standard" | "express"
    status: str       # "created" | "paid" | "shipped" | "cancelled"

class OrderEventPublisher:
    def __init__(self, topic_arn: str, region: str = "us-east-1"):
        self.sns = boto3.client("sns", region_name=region)
        self.topic_arn = topic_arn

    def publish_order_created(self, event: OrderEvent) -> str:
        response = self.sns.publish(
            TopicArn=self.topic_arn,
            Message=json.dumps({
                **asdict(event),
                "eventType": "ORDER_CREATED",
                "timestamp": datetime.utcnow().isoformat()
            }),
            Subject=f"Order Created: {event.order_id}",
            MessageAttributes={
                "eventType": {
                    "DataType": "String",
                    "StringValue": "ORDER_CREATED"
                },
                "orderType": {
                    "DataType": "String",
                    "StringValue": event.order_type
                },
                "orderAmount": {
                    "DataType": "Number",
                    "StringValue": str(event.amount)
                }
            }
        )
        print(f"Published: {response['MessageId']} for order {event.order_id}")
        return response["MessageId"]

# Sử dụng
publisher = OrderEventPublisher("arn:aws:sns:us-east-1:123456:order-events")
publisher.publish_order_created(OrderEvent(
    order_id="ORD-001",
    customer_id="CUST-123",
    amount=299.99,
    order_type="express",
    status="created"
))
```

### Subscriber qua SQS + Lambda Consumer

```python
import boto3
import json

def lambda_handler(event, context):
    """Lambda nhận sự kiện từ SNS qua SQS trigger."""
    for record in event.get("Records", []):
        # SQS message body chứa SNS notification
        sqs_body = json.loads(record["body"])

        # Nếu đến từ SNS, body của SNS nằm trong "Message"
        if sqs_body.get("Type") == "Notification":
            sns_message = json.loads(sqs_body["Message"])
            sns_attributes = sqs_body.get("MessageAttributes", {})

            event_type = sns_attributes.get("eventType", {}).get("Value")
            order_id = sns_message.get("order_id")

            print(f"Processing {event_type} for order {order_id}")

            if event_type == "ORDER_CREATED":
                handle_order_created(sns_message)
            elif event_type == "ORDER_PAID":
                handle_order_paid(sns_message)

def handle_order_created(payload: dict):
    print(f"Initializing shipping for order {payload['order_id']}")

def handle_order_paid(payload: dict):
    print(f"Confirming payment for order {payload['order_id']}")
```

---

## 🔧 Subscription Management (Quản Lý Đăng Ký)

```bash
# Liệt kê tất cả subscriptions của topic
aws sns list-subscriptions-by-topic \
  --topic-arn arn:aws:sns:us-east-1:123456:order-events

# Lấy thông tin subscription
aws sns get-subscription-attributes \
  --subscription-arn arn:aws:sns:us-east-1:123456:order-events:abc-123

# Cập nhật subscription attributes (thêm filter policy)
aws sns set-subscription-attributes \
  --subscription-arn arn:aws:sns:us-east-1:123456:order-events:abc-123 \
  --attribute-name FilterPolicy \
  --attribute-value '{"orderType":["express"]}'

# Bật Raw Message Delivery (giao vận tin nhắn thô — không bọc JSON của SNS)
aws sns set-subscription-attributes \
  --subscription-arn arn:aws:sns:us-east-1:123456:order-events:abc-123 \
  --attribute-name RawMessageDelivery \
  --attribute-value true

# Unsubscribe
aws sns unsubscribe \
  --subscription-arn arn:aws:sns:us-east-1:123456:order-events:abc-123
```

### Raw Message Delivery (Giao Vận Tin Nhắn Thô)

Mặc định, khi SNS giao vận vào SQS, tin nhắn được **bọc** thêm metadata:

```json
// Mặc định (có bọc SNS wrapper):
{
  "Type": "Notification",
  "MessageId": "abc-123",
  "TopicArn": "arn:aws:sns:...",
  "Subject": "New Order",
  "Message": "{\"orderId\":\"ORD-001\"}",   ← nội dung thực bị serialize lồng nhau
  "Timestamp": "...",
  "Signature": "..."
}

// Khi bật RawMessageDelivery = true:
{
  "orderId": "ORD-001",                     ← nội dung gốc, không bọc thêm
  "customerId": "CUST-123",
  "amount": 299.99
}
```

**Khuyến nghị:** Bật `RawMessageDelivery=true` cho SQS/Lambda subscriber để đơn giản hóa code consumer.

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

### Q: SNS hỗ trợ những protocol nào?

**A:** SQS, Lambda, HTTP/HTTPS, Email, Email-JSON, SMS, và Mobile Push (APNs cho iOS, FCM cho Android, ADM cho Kindle). Đối với FIFO Topic, chỉ hỗ trợ SQS FIFO Queue.

### Q: Subscription cần xác nhận không?

**A:** Tùy protocol:
- **Tự động xác nhận:** SQS, Lambda, SQS FIFO
- **Cần xác nhận thủ công:** Email, HTTP/HTTPS — SNS gửi request xác nhận, endpoint phải reply `SubscriptionConfirmation`

### Q: Raw Message Delivery dùng khi nào?

**A:** Dùng khi consumer (SQS hoặc Lambda) chỉ cần payload thuần túy, không cần metadata của SNS (MessageId, Signature...). Giúp đơn giản hóa code parsing ở phía consumer.

### Q: Tối đa bao nhiêu subscription cho một topic?

**A:** Mặc định 12.500.000 subscription per topic (hầu như không có giới hạn thực tế). Có thể tăng qua AWS Support nếu cần.

---

## 🗺️ Điều Hướng

- ◀ Quay lại: [README.md](./README.md) — Tổng quan SNS
- ▶ Tiếp theo: [2-message-filtering.md](./2-message-filtering.md) — Message Filtering
