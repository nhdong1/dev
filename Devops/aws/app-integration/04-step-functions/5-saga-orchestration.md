# Saga Pattern Với Step Functions — Orchestration Giao Dịch Phân Tán

> **Saga Pattern** (Mẫu Saga) là giải pháp cho bài toán **distributed transaction** (giao dịch phân tán) trong microservices: làm thế nào để đảm bảo tính nhất quán dữ liệu khi một quy trình nghiệp vụ liên quan đến nhiều service độc lập, và một bước trong chuỗi thất bại?

---

## 🎯 Vấn Đề: Distributed Transaction Trong Microservices

### Bài Toán Đặt Hàng

```
Hệ thống e-commerce có 4 service riêng biệt:
┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐
│  Order     │  │  Payment   │  │ Inventory  │  │  Shipping  │
│  Service   │  │  Service   │  │  Service   │  │  Service   │
│            │  │            │  │            │  │            │
│ Database A │  │ Database B │  │ Database C │  │ Database D │
└────────────┘  └────────────┘  └────────────┘  └────────────┘

Quy trình đặt hàng:
1. Tạo đơn hàng (Order DB)
2. Trừ tiền (Payment DB)
3. Giảm tồn kho (Inventory DB)
4. Tạo lệnh giao hàng (Shipping DB)

Vấn đề: Nếu bước 3 (Inventory) thất bại sau khi bước 2 (Payment) đã hoàn thành,
làm thế nào để hoàn tiền cho khách hàng?
→ Không thể dùng ACID transaction vì 4 database độc lập!
```

### Tại Sao Không Dùng 2PC (Two-Phase Commit)?

**2PC** (Two-Phase Commit — Cam Kết Hai Giai Đoạn) là giải pháp truyền thống cho distributed transaction, nhưng:

- **Blocking protocol** (giao thức chặn): Tất cả service phải lock tài nguyên đồng thời → giảm throughput nghiêm trọng
- **Single point of failure** (điểm lỗi duy nhất): Coordinator chết → toàn bộ hệ thống đình trệ
- **Không phù hợp microservices**: Microservices cần độc lập, 2PC tạo coupling (kết nối chặt)
- **Không scalable**: Khó mở rộng khi số service tăng

---

## 🧩 Saga Pattern — Giải Pháp

**Saga** chia giao dịch lớn thành chuỗi các **local transaction** (giao dịch cục bộ) nhỏ hơn. Mỗi local transaction:
- Cập nhật database của **một service**
- Kích hoạt bước tiếp theo hoặc **compensating transaction** (giao dịch bù trừ) khi thất bại

### Hai Kiểu Saga

```
Saga
├── Choreography (Vũ Đạo / Phân Tán)
│   └── Các service tự phát event và phản ứng với event của nhau
│       Không có coordinator trung tâm
│       Phù hợp: đơn giản, ít service
│
└── Orchestration (Điều Phối / Tập Trung) ← Step Functions
    └── Một orchestrator trung tâm điều khiển toàn bộ luồng
        Orchestrator quyết định bước tiếp theo
        Phù hợp: phức tạp, nhiều service, cần audit trail
```

---

## 🏗️ Saga Orchestration Với Step Functions

### Compensating Transactions (Giao Dịch Bù Trừ)

Mỗi bước "forward" (tiến) phải có một bước "backward" (lùi) tương ứng:

| Bước Forward | Bước Backward (Compensating) |
|-------------|------------------------------|
| Tạo đơn hàng | Hủy đơn hàng |
| Trừ tiền | Hoàn tiền |
| Giảm tồn kho | Bổ sung lại tồn kho |
| Tạo lệnh giao hàng | Hủy lệnh giao hàng |

### Luồng Thực Hiện

```
Bình thường (Happy Path — Đường Đi Thành Công):
TạoĐơn → TrừTiền → GiảmKho → TạoGiaoHàng → Thành công ✅

Khi GiảmKho thất bại (Compensating Path — Đường Bù Trừ):
TạoĐơn → TrừTiền → GiảmKho ❌
                         │
                         ▼
                  HoànTiền ← BổSungKho (nếu cần)
                         │
                         ▼
                  HủyĐơnHàng
                         │
                         ▼
                  Thông báo khách hàng
```

---

## 📝 Ví Dụ State Machine: Order Processing Saga

```json
{
  "Comment": "Saga Pattern cho xử lý đơn hàng",
  "StartAt": "TaoVaXacThucDonHang",
  "States": {

    "TaoVaXacThucDonHang": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "CreateOrder",
        "Payload.$": "$"
      },
      "ResultPath": "$.orderResult",
      "Retry": [
        {
          "ErrorEquals": ["Lambda.ServiceException"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["ValidationError", "DuplicateOrderError"],
          "Next": "TraLoiLoiDauVao",
          "ResultPath": "$.error"
        },
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "TraLoiLoiHeThong",
          "ResultPath": "$.error"
        }
      ],
      "Next": "XuLyThanhToan"
    },

    "XuLyThanhToan": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "ProcessPayment",
        "Payload": {
          "orderId.$": "$.orderId",
          "amount.$": "$.amount",
          "paymentMethod.$": "$.paymentMethod"
        }
      },
      "ResultPath": "$.paymentResult",
      "Retry": [
        {
          "ErrorEquals": ["Lambda.TooManyRequestsException"],
          "IntervalSeconds": 5,
          "MaxAttempts": 5,
          "BackoffRate": 2.0,
          "MaxDelaySeconds": 60
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["PaymentDeclinedError", "InsufficientFundsError"],
          "Next": "BuTruHuyDonHang",
          "ResultPath": "$.error"
        },
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "BuTruHuyDonHang",
          "ResultPath": "$.error"
        }
      ],
      "Next": "GiamTonKho"
    },

    "GiamTonKho": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "DeductInventory",
        "Payload": {
          "orderId.$": "$.orderId",
          "items.$": "$.items"
        }
      },
      "ResultPath": "$.inventoryResult",
      "Retry": [
        {
          "ErrorEquals": ["Lambda.ServiceException"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["OutOfStockError"],
          "Next": "BuTruHoanTienVaHuyDon",
          "ResultPath": "$.error"
        },
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "BuTruHoanTienVaHuyDon",
          "ResultPath": "$.error"
        }
      ],
      "Next": "TaoLeenhGiaoHang"
    },

    "TaoLeenhGiaoHang": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "CreateShipment",
        "Payload": {
          "orderId.$": "$.orderId",
          "address.$": "$.shippingAddress",
          "items.$": "$.items"
        }
      },
      "ResultPath": "$.shipmentResult",
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "BuTruHoanTienBoSungKhoHuyDon",
          "ResultPath": "$.error"
        }
      ],
      "Next": "ThongBaoThanhCong"
    },

    "ThongBaoThanhCong": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:ap-southeast-1:123:OrderSuccessTopic",
        "Message": {
          "orderId.$": "$.orderId",
          "trackingNumber.$": "$.shipmentResult.Payload.trackingNumber"
        }
      },
      "End": true
    },

    "BuTruHuyDonHang": {
      "Type": "Task",
      "Comment": "Compensating: Hủy đơn hàng (chưa có thanh toán)",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "CancelOrder",
        "Payload": {
          "orderId.$": "$.orderId",
          "reason.$": "$.error"
        }
      },
      "ResultPath": "$.cancelResult",
      "Retry": [
        {
          "ErrorEquals": ["States.ALL"],
          "IntervalSeconds": 5,
          "MaxAttempts": 5,
          "BackoffRate": 2.0
        }
      ],
      "Next": "ThongBaoThatBai"
    },

    "BuTruHoanTienVaHuyDon": {
      "Type": "Parallel",
      "Comment": "Compensating: Hoàn tiền VÀ hủy đơn hàng đồng thời",
      "Branches": [
        {
          "StartAt": "HoanTien",
          "States": {
            "HoanTien": {
              "Type": "Task",
              "Resource": "arn:aws:states:::lambda:invoke",
              "Parameters": {
                "FunctionName": "RefundPayment",
                "Payload": {
                  "orderId.$": "$.orderId",
                  "paymentResult.$": "$.paymentResult"
                }
              },
              "Retry": [
                {
                  "ErrorEquals": ["States.ALL"],
                  "IntervalSeconds": 10,
                  "MaxAttempts": 10,
                  "BackoffRate": 2.0,
                  "MaxDelaySeconds": 300
                }
              ],
              "End": true
            }
          }
        },
        {
          "StartAt": "HuyDonHang",
          "States": {
            "HuyDonHang": {
              "Type": "Task",
              "Resource": "arn:aws:states:::lambda:invoke",
              "Parameters": {
                "FunctionName": "CancelOrder",
                "Payload.$": "$"
              },
              "Retry": [
                {
                  "ErrorEquals": ["States.ALL"],
                  "IntervalSeconds": 5,
                  "MaxAttempts": 5,
                  "BackoffRate": 2.0
                }
              ],
              "End": true
            }
          }
        }
      ],
      "ResultPath": "$.compensationResults",
      "Next": "ThongBaoThatBai"
    },

    "BuTruHoanTienBoSungKhoHuyDon": {
      "Type": "Parallel",
      "Comment": "Compensating: Hoàn tiền + Bổ sung tồn kho + Hủy đơn",
      "Branches": [
        {
          "StartAt": "HoanTienSauGiaoHang",
          "States": {
            "HoanTienSauGiaoHang": {
              "Type": "Task",
              "Resource": "arn:aws:states:::lambda:invoke",
              "Parameters": {
                "FunctionName": "RefundPayment",
                "Payload.$": "$"
              },
              "Retry": [
                { "ErrorEquals": ["States.ALL"], "IntervalSeconds": 10, "MaxAttempts": 10, "BackoffRate": 2.0 }
              ],
              "End": true
            }
          }
        },
        {
          "StartAt": "BoSungKho",
          "States": {
            "BoSungKho": {
              "Type": "Task",
              "Resource": "arn:aws:states:::lambda:invoke",
              "Parameters": {
                "FunctionName": "RestoreInventory",
                "Payload.$": "$"
              },
              "Retry": [
                { "ErrorEquals": ["States.ALL"], "IntervalSeconds": 5, "MaxAttempts": 5, "BackoffRate": 2.0 }
              ],
              "End": true
            }
          }
        },
        {
          "StartAt": "HuyDonHangSauGiaoHang",
          "States": {
            "HuyDonHangSauGiaoHang": {
              "Type": "Task",
              "Resource": "arn:aws:states:::lambda:invoke",
              "Parameters": {
                "FunctionName": "CancelOrder",
                "Payload.$": "$"
              },
              "Retry": [
                { "ErrorEquals": ["States.ALL"], "IntervalSeconds": 5, "MaxAttempts": 5, "BackoffRate": 2.0 }
              ],
              "End": true
            }
          }
        }
      ],
      "ResultPath": "$.compensationResults",
      "Next": "ThongBaoThatBai"
    },

    "ThongBaoThatBai": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:ap-southeast-1:123:OrderFailedTopic",
        "Message": {
          "orderId.$": "$.orderId",
          "error.$": "$.error"
        }
      },
      "Next": "KetThucThatBai"
    },

    "TraLoiLoiDauVao": {
      "Type": "Fail",
      "ErrorPath": "$.error.Error",
      "CausePath": "$.error.Cause"
    },

    "TraLoiLoiHeThong": {
      "Type": "Fail",
      "Error": "SystemError",
      "Cause": "Lỗi hệ thống khi tạo đơn hàng"
    },

    "KetThucThatBai": {
      "Type": "Fail",
      "Error": "OrderFailed",
      "Cause": "Đơn hàng thất bại sau khi đã bù trừ thành công"
    }
  }
}
```

---

## 🔀 Choreography vs Orchestration — So Sánh

### Choreography (Vũ Đạo — Event-Driven)

```
Client ──▶ Order Service ──event──▶ EventBridge/SNS
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    ▼                      ▼                      ▼
             Payment Service        Inventory Service       Notification
             (lắng nghe event        (lắng nghe event       Service
              OrderCreated)           PaymentCompleted)

Mỗi service tự phát event và tự quyết định làm gì tiếp theo
```

```
Ưu điểm Choreography:
✅ Loose coupling (kết nối lỏng) — service không biết nhau
✅ Dễ thêm service mới (chỉ cần subscribe event)
✅ Không có single point of failure

Hạn chế Choreography:
❌ Khó theo dõi luồng toàn bộ (flow không rõ ràng)
❌ Khó debug khi lỗi xảy ra giữa nhiều service
❌ Compensating logic phức tạp và phân tán
❌ Khó enforce sequence (thứ tự) nghiêm ngặt
```

### Orchestration (Điều Phối — Step Functions)

```
Client ──▶ Step Functions ──▶ Order Service
               │
               ├──▶ Payment Service
               │
               ├──▶ Inventory Service
               │
               └──▶ Shipping Service

Step Functions kiểm soát toàn bộ luồng
```

```
Ưu điểm Orchestration:
✅ Luồng công việc rõ ràng, có thể nhìn thấy trực quan (Workflow Studio)
✅ Dễ debug với execution history đầy đủ
✅ Compensating transactions rõ ràng và đáng tin cậy
✅ Dễ enforce sequence và retry logic
✅ Audit trail đầy đủ cho compliance

Hạn chế Orchestration:
❌ Coupling hơn — orchestrator biết tất cả service
❌ Single point of failure nếu orchestrator lỗi (AWS quản lý → ít lo)
❌ Chi phí cao hơn (Step Functions state transitions)
❌ Thêm một component cần quản lý
```

### Khi Nào Chọn Gì?

| Tình Huống | Choreography | Orchestration |
|-----------|-------------|--------------|
| Ít service (2–3) | ✅ | — |
| Nhiều service (4+) | — | ✅ |
| Cần audit trail | — | ✅ |
| Cần compensating transaction phức tạp | — | ✅ |
| Thêm service thường xuyên | ✅ | — |
| Debugging quan trọng | — | ✅ |
| Event-driven, loosely coupled | ✅ | — |
| Compliance, tài chính, y tế | — | ✅ |

---

## 🛡️ Idempotency Trong Saga

Vì Step Functions có thể retry, mọi bước trong Saga **phải idempotent** (bất biến):

### Ví Dụ Idempotent Lambda

```python
import boto3
import json
from botocore.exceptions import ClientError

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Payments')

def lambda_handler(event, context):
    order_id = event['orderId']
    amount = event['amount']
    
    try:
        # Dùng conditional write — chỉ tạo nếu chưa tồn tại
        table.put_item(
            Item={
                'orderId': order_id,
                'amount': amount,
                'status': 'COMPLETED',
                'processedAt': datetime.utcnow().isoformat()
            },
            ConditionExpression='attribute_not_exists(orderId)'
        )
        return {'status': 'CHARGED', 'orderId': order_id}
    
    except ClientError as e:
        if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
            # Đã xử lý rồi — trả về thành công (idempotent)
            existing = table.get_item(Key={'orderId': order_id})
            return {'status': 'ALREADY_CHARGED', 'orderId': order_id}
        raise

def refund_handler(event, context):
    order_id = event['orderId']
    
    # Dùng UpdateItem với condition — chỉ hoàn tiền nếu chưa hoàn
    try:
        table.update_item(
            Key={'orderId': order_id},
            UpdateExpression='SET #status = :refunded, refundedAt = :now',
            ConditionExpression='#status = :charged',
            ExpressionAttributeNames={'#status': 'status'},
            ExpressionAttributeValues={
                ':refunded': 'REFUNDED',
                ':charged': 'COMPLETED',
                ':now': datetime.utcnow().isoformat()
            }
        )
        return {'status': 'REFUNDED', 'orderId': order_id}
    
    except ClientError as e:
        if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
            # Đã hoàn tiền rồi — idempotent
            return {'status': 'ALREADY_REFUNDED', 'orderId': order_id}
        raise
```

---

## 📊 Trạng Thái Saga Và Tracking

Cần theo dõi trạng thái saga để debug và audit:

```python
# DynamoDB schema theo dõi saga
{
  "sagaId": "saga-uuid",           # ID của execution
  "orderId": "order-123",          # Business ID
  "status": "COMPENSATING",        # STARTED, COMPLETED, COMPENSATING, FAILED
  "steps": {
    "createOrder": "COMPLETED",
    "processPayment": "COMPLETED",
    "deductInventory": "FAILED",
    "refundPayment": "COMPLETED",
    "cancelOrder": "COMPLETED"
  },
  "startTime": "2026-05-18T10:00:00Z",
  "endTime": "2026-05-18T10:00:45Z",
  "errorInfo": {
    "step": "deductInventory",
    "error": "OutOfStockError",
    "cause": "Item A hết hàng"
  }
}
```

---

## ⚠️ Các Thách Thức Của Saga

### 1. Partial Failure Trong Compensating Transactions

```
Vấn đề: Compensating transaction cũng có thể thất bại!

TrừTiền ✅ → GiảmKho ❌ → HoànTiền ❌ (hệ thống thanh toán down)

Giải pháp:
- Retry aggressively (thử lại nhiều lần) với MaxAttempts cao hơn cho compensating steps
- Alert (cảnh báo) ngay và có SOP (Standard Operating Procedure) manual
- Dead Letter Queue cho compensation failures
- Idempotency key để retry an toàn
```

### 2. Dirty Reads — Đọc Dữ Liệu Chưa Nhất Quán

```
Vấn đề: Trong khi saga đang chạy, service khác có thể đọc trạng thái trung gian

Ví dụ: Order status = "PAID" nhưng kho chưa xác nhận
→ User thấy đơn hàng thành công nhưng thực ra chưa hoàn thành

Giải pháp:
- Semantic lock (khóa ngữ nghĩa): đánh dấu record là "IN_PROGRESS"
- Chỉ expose trạng thái cuối cùng (CONFIRMED, CANCELLED) ra API
- Polling hoặc webhook để thông báo trạng thái cuối
```

### 3. Out-of-Order Events

```
Vấn đề: Trong Choreography, events có thể đến không theo thứ tự

Giải pháp khi dùng Step Functions Orchestration:
- Không có vấn đề này vì orchestrator kiểm soát thứ tự tuyệt đối
→ Đây là một ưu điểm lớn của Orchestration Saga
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Saga Pattern giải quyết vấn đề gì mà ACID transaction không làm được trong microservices?**

> ACID transaction đòi hỏi tất cả resource phải nằm trong cùng một transaction boundary — không thể có với nhiều database độc lập trong microservices. Saga thay thế bằng chuỗi local transactions — mỗi service tự commit database riêng. Khi lỗi, Saga chạy compensating transactions để undo các bước đã thực hiện, đạt được "eventual consistency" (nhất quán theo thời gian) thay vì ACID consistency.

**Q: Tại sao phải dùng Standard Workflow cho Saga, không dùng Express?**

> Saga cần **exactly-once execution** — đảm bảo mỗi bước (đặc biệt là thanh toán, hoàn tiền) chỉ chạy đúng một lần. Express Workflow dùng at-least-once, có thể chạy bước nhiều hơn một lần → nếu `ProcessPayment` chạy hai lần, khách hàng bị trừ tiền hai lần. Ngoài ra, Standard Workflow lưu execution history đầy đủ — quan trọng cho audit trong hệ thống tài chính.

**Q: Nếu compensating transaction cũng thất bại, làm thế nào?**

> Đây là "saga failure" nghiêm trọng. Cần: (1) Retry aggressively với MaxAttempts cao và exponential backoff dài hơn — hầu hết lỗi tạm thời sẽ phục hồi; (2) Nếu vẫn thất bại, gửi alert ngay lập tức cho ops team; (3) Có runbook (tài liệu xử lý thủ công) để team có thể manual compensate; (4) Ghi vào DLQ (Dead Letter Queue) để audit và retry sau. Một số hệ thống tài chính chấp nhận trạng thái này và xử lý offline sau — gọi là "saga pivot transaction".

**Q: Choreography hay Orchestration — bạn chọn gì và khi nào?**

> Với hệ thống nhỏ ít service: Choreography vì loose coupling, dễ thêm service. Với quy trình nghiệp vụ phức tạp nhiều bước: Orchestration với Step Functions — luồng rõ ràng, dễ debug, compensating transaction đáng tin cậy. Thực tế nhiều hệ thống dùng cả hai: Choreography cho event routing đơn giản, Orchestration cho business process phức tạp có nhiều bước phụ thuộc.

---

## 🔗 Điều Hướng

- **Quay lại:** [4-callback-pattern.md](./4-callback-pattern.md)
- **Liên quan:** [3-error-handling.md](./3-error-handling.md) — Retry/Catch cần cho Saga
- **Liên quan:** [../08-patterns/2-saga-pattern.md](../08-patterns/2-saga-pattern.md) — Saga Pattern nói chung
- **Tổng quan module:** [README.md](./README.md)

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
