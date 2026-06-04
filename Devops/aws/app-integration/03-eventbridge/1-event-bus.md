# Event Bus — Default, Custom, và Partner Event Bus

> **Event Bus** (Xe Buýt Sự Kiện) là kênh trung tâm nhận, lưu trữ tạm thời và phân phối sự kiện đến các targets. Hiểu rõ 3 loại event bus là nền tảng để thiết kế kiến trúc EventBridge đúng.

---

## 📚 Mục Lục

1. [Default Event Bus](#1-default-event-bus)
2. [Custom Event Bus](#2-custom-event-bus)
3. [Partner Event Bus](#3-partner-event-bus)
4. [So Sánh Ba Loại](#4-so-sánh-ba-loại)
5. [Resource Policy](#5-resource-policy--chính-sách-tài-nguyên)
6. [Cross-Account Routing](#6-cross-account-routing--định-tuyến-liên-tài-khoản)
7. [Thiết Kế Multi-Bus Architecture](#7-thiết-kế-multi-bus-architecture)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Default Event Bus

### Đặc Điểm

- **Tồn tại sẵn** trong mọi tài khoản AWS, không cần tạo
- **Tự động nhận** sự kiện từ hơn 100 dịch vụ AWS (EC2, S3, RDS, IAM...)
- **Tên:** `default` (không thể đổi tên)
- **Không tính phí** cho sự kiện từ AWS services trên default bus

### Ví Dụ Sự Kiện Tự Động Từ AWS

```json
// EC2 instance state change (thay đổi trạng thái EC2 instance)
{
  "source": "aws.ec2",
  "detail-type": "EC2 Instance State-change Notification",
  "detail": {
    "instance-id": "i-1234567890abcdef0",
    "state": "running"
  }
}

// S3 object created (đối tượng S3 được tạo)
{
  "source": "aws.s3",
  "detail-type": "Object Created",
  "detail": {
    "bucket": { "name": "my-bucket" },
    "object": { "key": "uploads/photo.jpg", "size": 1024 }
  }
}

// RDS DB instance event (sự kiện RDS)
{
  "source": "aws.rds",
  "detail-type": "RDS DB Instance Event",
  "detail": {
    "EventCategories": ["backup"],
    "SourceIdentifier": "my-db"
  }
}
```

### Use Cases Phổ Biến

```
Auto remediation (tự động khắc phục):
EC2 terminated ──▶ Default Bus ──▶ Rule ──▶ Lambda (restart instance)

Compliance monitoring (giám sát tuân thủ):
IAM policy changed ──▶ Default Bus ──▶ Rule ──▶ SNS (alert security team)

Cost management (quản lý chi phí):
Trusted Advisor ──▶ Default Bus ──▶ Rule ──▶ Lambda (optimize resources)

CI/CD pipeline:
CodePipeline stage ──▶ Default Bus ──▶ Rule ──▶ Slack notification
```

---

## 2. Custom Event Bus

### Đặc Điểm

- **Tự tạo** — tách biệt sự kiện của ứng dụng với sự kiện AWS
- **Isolation** (Cô Lập) — mỗi domain/service có bus riêng
- **Tên** tùy chỉnh theo domain: `orders-bus`, `payments-bus`
- **Tính phí:** $1.00 / triệu events

### Tại Sao Cần Custom Bus?

**Vấn đề:** Nếu mọi thứ đều trên default bus, các rules sẽ nhiễu lẫn nhau và khó quản lý.

**Giải pháp:** Mỗi domain có bus riêng → tách biệt concerns (mối quan tâm):

```
Default Bus:      AWS service events (EC2, S3, RDS...)
orders-bus:       Order domain events (OrderPlaced, OrderShipped, OrderCancelled)
payments-bus:     Payment domain events (PaymentProcessed, PaymentFailed, Refunded)
inventory-bus:    Inventory domain events (StockUpdated, LowStockAlert)
```

### Tạo Custom Event Bus

```bash
# Tạo custom event bus
aws events create-event-bus \
  --name "orders-bus" \
  --tags Key=Environment,Value=Production

# List all event buses
aws events list-event-buses

# Xóa event bus
aws events delete-event-bus --name "orders-bus"
```

### Gửi Sự Kiện Đến Custom Bus

```python
import boto3
import json
from datetime import datetime

client = boto3.client('events', region_name='ap-southeast-1')

def publish_order_event(order_id: str, customer_id: str, amount: float):
    """Gửi sự kiện đặt hàng lên custom event bus."""
    response = client.put_events(
        Entries=[
            {
                'Time': datetime.utcnow(),
                'Source': 'com.mycompany.orders',
                'DetailType': 'Order Placed',
                'Detail': json.dumps({
                    'orderId': order_id,
                    'customerId': customer_id,
                    'amount': amount,
                    'currency': 'VND',
                    'status': 'PENDING'
                }),
                'EventBusName': 'orders-bus',  # Chỉ định custom bus
                'Resources': [
                    f'arn:aws:orders:ap-southeast-1:123456789:order/{order_id}'
                ]
            }
        ]
    )

    # Kiểm tra lỗi từng entry
    if response['FailedEntryCount'] > 0:
        for entry in response['Entries']:
            if 'ErrorCode' in entry:
                raise Exception(f"Failed to send event: {entry['ErrorMessage']}")

    return response

# Gửi batch events (nhiều sự kiện cùng lúc) — tối đa 10 entries/lần
def publish_batch_events(events: list):
    """Gửi nhiều sự kiện cùng lúc, tối đa 10."""
    entries = []
    for event in events[:10]:  # EventBridge giới hạn 10 entries/call
        entries.append({
            'Source': event['source'],
            'DetailType': event['detail_type'],
            'Detail': json.dumps(event['detail']),
            'EventBusName': 'orders-bus'
        })

    return client.put_events(Entries=entries)
```

### Custom Bus với CloudFormation (Hạ Tầng Dưới Dạng Mã)

```yaml
# cloudformation-eventbridge.yaml
Resources:
  OrdersEventBus:
    Type: AWS::Events::EventBus
    Properties:
      Name: orders-bus

  OrderPlacedRule:
    Type: AWS::Events::Rule
    Properties:
      EventBusName: !Ref OrdersEventBus
      Name: process-order-placed
      EventPattern:
        source:
          - com.mycompany.orders
        detail-type:
          - Order Placed
      State: ENABLED
      Targets:
        - Arn: !GetAtt OrderProcessorFunction.Arn
          Id: OrderProcessorTarget

  # Cho phép EventBridge gọi Lambda
  LambdaPermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref OrderProcessorFunction
      Action: lambda:InvokeFunction
      Principal: events.amazonaws.com
      SourceArn: !GetAtt OrderPlacedRule.Arn
```

---

## 3. Partner Event Bus

### Đặc Điểm

- **Nhận sự kiện từ SaaS partners** — không cần polling API
- **Event source** được AWS và partner cùng tạo
- Hỗ trợ **hơn 200+ partners** (Shopify, Stripe, Zendesk, GitHub, Datadog, PagerDuty...)
- **Không tính phí** cho việc nhận sự kiện từ partner (partner tự chịu phí)

### Danh Sách Partners Phổ Biến

| Partner | Sự Kiện Phổ Biến | Use Case |
|---|---|---|
| **Shopify** | Order created, Product updated | Sync inventory, fulfillment |
| **Stripe** | Payment succeeded, Invoice paid | Accounting, subscription |
| **Zendesk** | Ticket created, Ticket resolved | CRM, support analytics |
| **GitHub** | Pull request, Push, Issue | CI/CD, notifications |
| **Datadog** | Alert triggered, Recovery | Incident management |
| **PagerDuty** | Incident created, Resolved | On-call automation |
| **Salesforce** | Lead created, Opportunity won | Sales automation |

### Cách Thiết Lập Partner Event Bus

```
Bước 1: Vào EventBridge Console → Partner event sources
Bước 2: Chọn partner (ví dụ Shopify)
Bước 3: Partner tạo event source và cấp permission
Bước 4: Associate event source với event bus mới
Bước 5: Tạo Rules trên partner event bus
```

```bash
# Associate partner event source với event bus
aws events create-event-bus \
  --name "shopify-events" \
  --event-source-name "aws.partner/shopify.com/SHOP_ID/events"
```

### Ví Dụ Sự Kiện Từ Shopify

```json
{
  "version": "0",
  "source": "aws.partner/shopify.com/12345/orders",
  "detail-type": "shopify.orders/create",
  "detail": {
    "id": 820982911946154500,
    "email": "customer@example.com",
    "total_price": "199.90",
    "currency": "USD",
    "line_items": [
      {
        "variant_id": 808950810,
        "title": "Premium Widget",
        "quantity": 2,
        "price": "99.95"
      }
    ]
  }
}
```

---

## 4. So Sánh Ba Loại

| Tiêu Chí | Default Bus | Custom Bus | Partner Bus |
|---|---|---|---|
| **Ai tạo** | AWS (tự động) | Bạn tự tạo | AWS + Partner |
| **Nguồn sự kiện** | AWS services | Ứng dụng của bạn | SaaS partners |
| **Chi phí nhận** | Miễn phí | $1/triệu | Miễn phí |
| **Số lượng** | 1/account/region | Tối đa 100 | Theo số partner |
| **Xóa được** | Không | Có | Có |
| **Isolation** | Thấp | Cao | Cao |
| **Cross-account** | Có | Có | Không |

---

## 5. Resource Policy — Chính Sách Tài Nguyên

**Resource Policy** kiểm soát ai được phép `PutEvents` vào event bus:

### Cho Phép Account Khác Gửi Sự Kiện

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCrossAccountPutEvents",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::PRODUCER_ACCOUNT_ID:root"
      },
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:ap-southeast-1:CONSUMER_ACCOUNT_ID:event-bus/orders-bus"
    }
  ]
}
```

### Cho Phép Toàn Bộ Organization (Tổ Chức AWS)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOrganizationPutEvents",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:ap-southeast-1:CONSUMER_ACCOUNT_ID:event-bus/central-bus",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-ORGANIZATION_ID"
        }
      }
    }
  ]
}
```

### Áp Dụng Resource Policy

```bash
aws events put-permission \
  --event-bus-name "orders-bus" \
  --action "events:PutEvents" \
  --principal "123456789012" \  # Account ID của producer
  --statement-id "AllowProducerAccount"
```

---

## 6. Cross-Account Routing — Định Tuyến Liên Tài Khoản

### Kiến Trúc Hub-and-Spoke (Trung Tâm và Nan Hoa)

```
Account A (Dev)          Central Account           Account B (Prod)
┌─────────────┐          ┌───────────────┐          ┌─────────────┐
│ Custom Bus  │─────────▶│ Central Bus   │─────────▶│ Custom Bus  │
│ (dev-events)│          │ (all-events)  │          │(prod-events)│
└─────────────┘          │               │          └─────────────┘
                         │  ┌─────────┐  │
Account C (Staging)      │  │  Rules  │  │          Account D (Analytics)
┌─────────────┐          │  │ (route) │  │          ┌─────────────┐
│ Custom Bus  │─────────▶│  └─────────┘  │─────────▶│ Custom Bus  │
│(stg-events) │          └───────────────┘          │(analytics)  │
└─────────────┘                                     └─────────────┘
```

### Implementation (Triển Khai)

```python
# Producer Account — gửi sự kiện đến central bus
client.put_events(
    Entries=[
        {
            'Source': 'com.mycompany.orders',
            'DetailType': 'Order Placed',
            'Detail': json.dumps(order_data),
            # ARN của central bus ở account khác
            'EventBusName': 'arn:aws:events:ap-southeast-1:CENTRAL_ACCOUNT:event-bus/central-bus'
        }
    ]
)
```

---

## 7. Thiết Kế Multi-Bus Architecture

### Kiến Trúc Cho Microservices (Dịch Vụ Nhỏ)

```
                        ┌──────────────────────────────────┐
                        │         Event Bus Per Domain     │
                        │                                  │
┌─────────────┐         │  ┌─────────────┐                 │
│ Order       │────────▶│  │ orders-bus  │──▶ Lambda, SQS  │
│ Service     │         │  └─────────────┘                 │
└─────────────┘         │                                  │
┌─────────────┐         │  ┌─────────────┐                 │
│ Payment     │────────▶│  │payments-bus │──▶ Lambda, SQS  │
│ Service     │         │  └─────────────┘                 │
└─────────────┘         │                                  │
┌─────────────┐         │  ┌─────────────┐                 │
│ Inventory   │────────▶│  │inventory-bus│──▶ Lambda, SQS  │
│ Service     │         │  └─────────────┘                 │
└─────────────┘         └──────────────────────────────────┘
```

### Best Practices (Thực Hành Tốt Nhất)

```
1. Một bus per domain — tách biệt concerns
2. Dùng default bus chỉ cho AWS service events
3. Đặt tên bus theo domain: <domain>-<env>-bus (vd: orders-prod-bus)
4. Tag buses để quản lý cost
5. Dùng resource policy để kiểm soát access
6. Giới hạn số rules per bus — tối đa 300 rules/bus
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: Tại sao không đặt tất cả sự kiện vào default bus?**

> Default bus nhận sự kiện từ tất cả AWS services — nếu thêm sự kiện ứng dụng vào đây, rules sẽ bị lẫn lộn và khó debug. Custom bus cho phép isolation (cô lập) rõ ràng, dễ quản lý IAM permissions, và giảm nguy cơ rules vô tình match nhầm.

**Q: Khi nào dùng partner event bus thay vì polling SaaS API?**

> Partner event bus phù hợp khi: (1) SaaS partner hỗ trợ EventBridge integration, (2) cần realtime events (không muốn delay từ polling), (3) muốn giảm code complexity (không cần viết webhook server). Trade-off: phụ thuộc vào partner hỗ trợ, không phải tất cả SaaS đều có.

**Q: Cross-account event routing giải quyết vấn đề gì?**

> Trong tổ chức nhiều tài khoản AWS (multi-account), mỗi team có account riêng. Cross-account routing cho phép: (1) centralized event logging ở account monitoring, (2) event sharing giữa teams mà không cần coupling trực tiếp, (3) security — producer không cần quyền vào account consumer.

---

**Liên Kết:** [README.md](README.md) | [2-event-rules-patterns.md](2-event-rules-patterns.md)
