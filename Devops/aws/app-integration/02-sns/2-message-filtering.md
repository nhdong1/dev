# Message Filtering — Lọc Tin Nhắn SNS

> Message Filtering (Lọc Tin Nhắn) cho phép mỗi subscriber chỉ nhận những tin nhắn phù hợp với điều kiện lọc — giảm tải và tăng tính chính xác mà không cần tạo nhiều topic.

## 📌 Tóm Tắt Nhanh

| Khái Niệm | Giải Thích |
|---|---|
| **Filter Policy** (Chính Sách Lọc) | JSON định nghĩa điều kiện để subscriber nhận tin |
| **Message Attributes** (Thuộc Tính Tin Nhắn) | Metadata của tin nhắn, dùng để match với filter |
| **Filter Policy Scope** | `MessageAttributes` (mặc định) hoặc `MessageBody` |
| **No Filter** (Không Lọc) | Subscription không có filter → nhận **tất cả** tin nhắn |

---

## 🎯 Tại Sao Cần Message Filtering?

### Bài Toán Không Có Filtering

```
Không có filtering:
[SNS Topic: all-orders]
    │
    ├──▶ [SQS: express-queue]     ← nhận CẢ express lẫn standard orders
    │         Consumer phải tự lọc sau khi nhận
    │
    └──▶ [SQS: standard-queue]    ← nhận CẢ express lẫn standard orders
              Consumer phải tự lọc sau khi nhận

Vấn đề:
- Mỗi queue nhận tin nhắn không cần thiết → tốn tài nguyên
- Consumer phải tự lọc → code phức tạp hơn
- Chi phí SQS tăng do nhận tin nhắn không liên quan
```

### Giải Pháp: Message Filtering

```
Có filtering:
[SNS Topic: all-orders]
    │
    ├──▶ [SQS: express-queue]     ← CHỈ nhận tin có orderType="express"
    │         Filter: {"orderType": ["express"]}
    │
    └──▶ [SQS: standard-queue]    ← CHỈ nhận tin có orderType="standard"
              Filter: {"orderType": ["standard"]}

Lợi ích:
✅ Mỗi queue chỉ nhận tin phù hợp
✅ Giảm tải consumer không cần thiết
✅ Giảm chi phí
✅ Code consumer đơn giản hơn
```

---

## 🔧 Cách Cấu Hình Filter Policy

### Cấu Trúc Filter Policy

```json
{
  "attribute-name-1": [condition1, condition2],
  "attribute-name-2": [condition3]
}
```

- Nhiều attribute trong policy: **AND** logic — tất cả phải match
- Nhiều value trong một attribute: **OR** logic — bất kỳ giá trị nào match là được

### Thiết Lập Filter Policy Qua CLI

```bash
# Thêm filter cho subscription
aws sns set-subscription-attributes \
  --subscription-arn arn:aws:sns:us-east-1:123456:order-events:sub-abc \
  --attribute-name FilterPolicy \
  --attribute-value '{"orderType":["express"],"status":["created","paid"]}'

# Xóa filter (nhận tất cả)
aws sns set-subscription-attributes \
  --subscription-arn arn:aws:sns:us-east-1:123456:order-events:sub-abc \
  --attribute-name FilterPolicy \
  --attribute-value '{}'
```

---

## 📋 Các Loại Điều Kiện Filter

### 1. Exact Match — Khớp Chính Xác (String)

```json
{
  "orderType": ["express", "priority"]
}
```

Nhận tin nhắn khi `orderType` là `"express"` **hoặc** `"priority"`.

```python
# Publish tin nhắn sẽ được nhận bởi filter trên
sns.publish(
    TopicArn=topic_arn,
    Message=json.dumps({"orderId": "ORD-001"}),
    MessageAttributes={
        "orderType": {
            "DataType": "String",
            "StringValue": "express"    # ✅ Match
        }
    }
)

# Tin nhắn này KHÔNG được nhận
sns.publish(
    TopicArn=topic_arn,
    Message=json.dumps({"orderId": "ORD-002"}),
    MessageAttributes={
        "orderType": {
            "DataType": "String",
            "StringValue": "standard"   # ❌ Không match
        }
    }
)
```

### 2. Numeric Match — Khớp Số

```json
{
  "orderAmount": [
    {"numeric": [">=", 100]},
    {"numeric": ["<", 1000]}
  ]
}
```

Nhận khi `orderAmount >= 100` **và** `orderAmount < 1000`.

**Các toán tử số học:**

| Toán Tử | Ý Nghĩa | Ví Dụ |
|---|---|---|
| `=` | Bằng | `{"numeric": ["=", 100]}` |
| `<` | Nhỏ hơn | `{"numeric": ["<", 1000]}` |
| `<=` | Nhỏ hơn hoặc bằng | `{"numeric": ["<=", 1000]}` |
| `>` | Lớn hơn | `{"numeric": [">", 0]}` |
| `>=` | Lớn hơn hoặc bằng | `{"numeric": [">=", 100]}` |

**Kết hợp — Range Filter (Lọc Phạm Vi):**

```json
{
  "orderAmount": [
    {"numeric": [">=", 100, "<", 1000]}
  ]
}
```

### 3. Prefix Match — Khớp Tiền Tố

```json
{
  "customerId": [{"prefix": "PREMIUM-"}]
}
```

Nhận khi `customerId` bắt đầu bằng `"PREMIUM-"`.

```python
# ✅ Match: PREMIUM-001, PREMIUM-ABC
# ❌ Không match: STANDARD-001, premium-001 (case-sensitive — phân biệt hoa/thường)
```

### 4. Anything-But — Ngoại Trừ

```json
{
  "status": [{"anything-but": ["cancelled", "refunded"]}]
}
```

Nhận khi `status` là bất kỳ giá trị nào **ngoại trừ** `"cancelled"` và `"refunded"`.

```json
// Anything-but với số
{
  "errorCode": [{"anything-but": [404, 503]}]
}
```

### 5. Exists — Kiểm Tra Sự Tồn Tại

```json
{
  "promotionCode": [{"exists": true}]
}
```

Nhận khi tin nhắn **có** attribute `promotionCode` (không quan tâm giá trị là gì).

```json
// Nhận khi KHÔNG có attribute
{
  "internalFlag": [{"exists": false}]
}
```

### 6. String Array — Mảng Chuỗi

```json
{
  "tags": [{"anything-but": ["internal", "test"]}]
}
```

Dùng với `DataType: "String.Array"` để match khi mảng chứa (hoặc không chứa) giá trị nào đó.

---

## 🔗 Kết Hợp Nhiều Điều Kiện

### AND Logic (Tất Cả Điều Kiện Phải Đúng)

```json
{
  "orderType": ["express"],
  "status": ["created", "processing"],
  "orderAmount": [{"numeric": [">=", 50]}]
}
```

Subscriber nhận khi:
- `orderType` là `"express"` **VÀ**
- `status` là `"created"` hoặc `"processing"` **VÀ**
- `orderAmount >= 50`

### OR Logic (Bất Kỳ Điều Kiện Đúng)

```json
{
  "priority": ["high", "critical", "urgent"]
}
```

Subscriber nhận khi `priority` là `"high"` **HOẶC** `"critical"` **HOẶC** `"urgent"`.

---

## 🏭 Ví Dụ Thực Tế — E-commerce Order System

### Kiến Trúc

```
[Order Service]
    │ Publish với attributes: orderType, status, amount, region
    ▼
[SNS Topic: order-events]
    │
    ├──▶ [SQS: express-processing]
    │    Filter: {"orderType":["express"],"status":["created"]}
    │    Consumer: Xử lý giao hàng nhanh, SLA 2 giờ
    │
    ├──▶ [SQS: payment-alerts]
    │    Filter: {"status":["payment_failed"]}
    │    Consumer: Thông báo thất bại thanh toán đến finance team
    │
    ├──▶ [SQS: high-value-orders]
    │    Filter: {"orderAmount":[{"numeric":[">=",10000]}]}
    │    Consumer: Review thủ công đơn hàng giá trị cao
    │
    ├──▶ [SQS: vn-orders]
    │    Filter: {"region":["VN"],"status":["created","paid"]}
    │    Consumer: Xử lý đơn hàng khu vực Việt Nam
    │
    └──▶ [Lambda: fraud-detection]
         Filter: {"orderAmount":[{"numeric":[">",5000]}],"isNewCustomer":["true"]}
         Consumer: Kiểm tra gian lận đơn lớn từ khách mới
```

### Code Triển Khai

```python
import boto3
import json

sns = boto3.client("sns", region_name="us-east-1")
TOPIC_ARN = "arn:aws:sns:us-east-1:123456:order-events"

def setup_subscriptions():
    """Tạo subscription với filter policies cho từng queue."""

    subscriptions = [
        {
            "queue_arn": "arn:aws:sqs:us-east-1:123456:express-processing",
            "filter": {
                "orderType": ["express"],
                "status": ["created"]
            }
        },
        {
            "queue_arn": "arn:aws:sqs:us-east-1:123456:payment-alerts",
            "filter": {
                "status": ["payment_failed"]
            }
        },
        {
            "queue_arn": "arn:aws:sqs:us-east-1:123456:high-value-orders",
            "filter": {
                "orderAmount": [{"numeric": [">=", 10000]}]
            }
        }
    ]

    for sub in subscriptions:
        response = sns.subscribe(
            TopicArn=TOPIC_ARN,
            Protocol="sqs",
            Endpoint=sub["queue_arn"],
            Attributes={
                "FilterPolicy": json.dumps(sub["filter"]),
                "RawMessageDelivery": "true"
            }
        )
        print(f"Subscribed {sub['queue_arn']}: {response['SubscriptionArn']}")

def publish_order_event(
    order_id: str,
    order_type: str,
    status: str,
    amount: float,
    region: str,
    is_new_customer: bool
):
    """Publish sự kiện đơn hàng với đầy đủ attributes để filter hoạt động."""
    sns.publish(
        TopicArn=TOPIC_ARN,
        Message=json.dumps({
            "orderId": order_id,
            "orderType": order_type,
            "status": status,
            "amount": amount,
            "region": region,
            "isNewCustomer": is_new_customer
        }),
        MessageAttributes={
            "orderType": {
                "DataType": "String",
                "StringValue": order_type
            },
            "status": {
                "DataType": "String",
                "StringValue": status
            },
            "orderAmount": {
                "DataType": "Number",
                "StringValue": str(amount)
            },
            "region": {
                "DataType": "String",
                "StringValue": region
            },
            "isNewCustomer": {
                "DataType": "String",
                "StringValue": str(is_new_customer).lower()
            }
        }
    )

# Ví dụ: Đơn express 15.000 USD từ khách mới tại VN
# Sẽ được nhận bởi: express-processing, high-value-orders, vn-orders, fraud-detection
publish_order_event(
    order_id="ORD-001",
    order_type="express",
    status="created",
    amount=15000,
    region="VN",
    is_new_customer=True
)
```

---

## 🆕 Filter Policy Scope — MessageBody (Mới)

Từ 2022, SNS hỗ trợ filter dựa trên **nội dung tin nhắn** (MessageBody) thay vì chỉ MessageAttributes:

```bash
# Đặt FilterPolicyScope = MessageBody
aws sns set-subscription-attributes \
  --subscription-arn arn:aws:sns:us-east-1:123456:order-events:sub-abc \
  --attribute-name FilterPolicyScope \
  --attribute-value MessageBody

# Filter policy áp dụng lên JSON body, không phải attributes
aws sns set-subscription-attributes \
  --subscription-arn arn:aws:sns:us-east-1:123456:order-events:sub-abc \
  --attribute-name FilterPolicy \
  --attribute-value '{"orderType":["express"]}'
```

**Khi dùng MessageBody scope:**

```json
// Tin nhắn được publish
{
  "orderType": "express",     ← Filter sẽ match vào đây (không phải attributes)
  "orderId": "ORD-001",
  "amount": 299.99
}
```

**So sánh hai scope:**

| Tiêu Chí | MessageAttributes (mặc định) | MessageBody |
|---|---|---|
| **Vị trí filter** | Trên MessageAttributes | Trên body JSON |
| **Cần thêm attributes khi publish?** | Có | Không |
| **Hỗ trợ nested JSON?** | Không | Có (tối đa 5 cấp) |
| **Phù hợp với** | Event với schema có sẵn | Legacy systems, microservices |

---

## ⚠️ Lưu Ý Quan Trọng

### 1. Tin Nhắn Không Có Attribute

Nếu subscriber có filter `{"orderType": ["express"]}` nhưng tin nhắn **không có** attribute `orderType`:

```
Kết quả: Tin nhắn KHÔNG được giao cho subscriber đó
```

Nếu muốn nhận cả tin không có attribute, dùng `exists: false` trong filter.

### 2. Giới Hạn Filter Policy

| Giới Hạn | Giá Trị |
|---|---|
| Số attributes trong một filter policy | Tối đa 5 |
| Số values trong một attribute | Tối đa 150 |
| Kích thước filter policy JSON | Tối đa 256 KB |

### 3. Throughput Khi Có Nhiều Filter

Filter không ảnh hưởng đến throughput (thông lượng) của SNS — SNS vẫn nhận tin nhắn tốc độ cao, chỉ là quá trình phân phối có thêm bước kiểm tra filter.

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

### Q: Message Filtering trong SNS hoạt động ở đâu?

**A:** Filtering xảy ra **phía SNS** trước khi giao vận — subscriber không bao giờ nhận được tin nhắn không phù hợp. Điều này khác với tự lọc ở consumer (nhận hết rồi bỏ qua), giúp giảm chi phí và tải.

### Q: Filter policy AND hay OR?

**A:** Nhiều attributes trong cùng một policy là **AND** (tất cả phải match). Nhiều values trong một attribute là **OR** (một trong các giá trị match là được).

### Q: Subscriber không có filter policy thì sao?

**A:** Nhận **tất cả** tin nhắn được publish vào topic, không phân biệt attributes.

### Q: FilterPolicyScope: MessageBody có gì hay hơn MessageAttributes?

**A:** Không cần bổ sung MessageAttributes khi publish — filter thẳng vào JSON body. Tiện lợi khi body đã có đủ thông tin cần lọc, hoặc khi không kiểm soát được producer (publisher). Hỗ trợ nested JSON (JSON lồng nhau) tối đa 5 cấp.

---

## 🗺️ Điều Hướng

- ◀ Quay lại: [1-topic-subscription.md](./1-topic-subscription.md) — Topic & Subscription
- ▶ Tiếp theo: [3-fanout-pattern.md](./3-fanout-pattern.md) — Fan-out Pattern với SNS + SQS
