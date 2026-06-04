# Error Handling — Xử Lý Lỗi Với Catch & Retry

> Xử lý lỗi là một trong những tính năng mạnh mẽ nhất của Step Functions. **Retry** (Thử Lại) và **Catch** (Bắt Lỗi) được định nghĩa ngay trong state machine definition — không cần code trong Lambda — giúp tách rời logic retry khỏi business logic.

---

## 🎯 Tổng Quan

Step Functions cung cấp hai cơ chế xử lý lỗi:

| Cơ Chế | Mục Đích | Khi Nào Dùng |
|--------|---------|-------------|
| **Retry** (Thử Lại) | Tự động thử lại task khi gặp lỗi | Lỗi tạm thời: network timeout, throttling, service unavailable |
| **Catch** (Bắt Lỗi) | Chuyển sang state xử lý lỗi khi retry hết | Lỗi không thể retry: business error, validation error |

---

## 🔁 Retry — Tự Động Thử Lại

### Cú Pháp Cơ Bản

```json
{
  "XuLyDonHang": {
    "Type": "Task",
    "Resource": "arn:aws:lambda:ap-southeast-1:123:function:ProcessOrder",
    "Retry": [
      {
        "ErrorEquals": ["Lambda.ServiceException", "Lambda.AWSLambdaException"],
        "IntervalSeconds": 2,
        "MaxAttempts": 3,
        "BackoffRate": 2.0,
        "MaxDelaySeconds": 30,
        "JitterStrategy": "FULL"
      },
      {
        "ErrorEquals": ["States.Timeout"],
        "IntervalSeconds": 5,
        "MaxAttempts": 2,
        "BackoffRate": 1.5
      }
    ],
    "Next": "ThanhToan"
  }
}
```

### Các Tham Số Retry

| Tham Số | Kiểu | Bắt Buộc | Mô Tả |
|---------|------|----------|-------|
| `ErrorEquals` | Array | Có | Danh sách error code cần retry |
| `IntervalSeconds` | Number | Không (mặc định: 1) | Thời gian chờ trước lần retry đầu (giây) |
| `MaxAttempts` | Number | Không (mặc định: 3) | Số lần retry tối đa (0 = không retry) |
| `BackoffRate` | Number | Không (mặc định: 2.0) | Hệ số nhân thời gian chờ cho mỗi lần retry |
| `MaxDelaySeconds` | Number | Không | Giới hạn tối đa thời gian chờ giữa các retry |
| `JitterStrategy` | String | Không | `FULL` hoặc `NONE` — thêm ngẫu nhiên vào thời gian chờ |

### Cách Tính Thời Gian Chờ Giữa Các Retry

```
IntervalSeconds = 2, BackoffRate = 2.0, MaxDelaySeconds = 30

Lần retry 1: chờ 2 giây
Lần retry 2: chờ 2 × 2.0 = 4 giây
Lần retry 3: chờ 4 × 2.0 = 8 giây
Lần retry 4: chờ 8 × 2.0 = 16 giây
Lần retry 5: chờ min(16 × 2.0, 30) = 30 giây (bị giới hạn bởi MaxDelaySeconds)

Đây là Exponential Backoff — Tăng Thời Gian Chờ Theo Hàm Mũ
```

### JitterStrategy — Chiến Lược Thêm Ngẫu Nhiên

**Jitter** (Độ Lệch Ngẫu Nhiên) tránh thundering herd (đàn sét) — nhiều client retry cùng lúc làm quá tải service.

```
NONE (không jitter):
  Retry 1: chờ đúng 2 giây
  Retry 2: chờ đúng 4 giây

FULL (jitter đầy đủ):
  Retry 1: chờ random[0, 2] giây (ví dụ: 0.8 giây)
  Retry 2: chờ random[0, 4] giây (ví dụ: 2.3 giây)

→ FULL jitter được khuyến nghị cho distributed systems
```

---

## 🔴 Error Types — Các Loại Lỗi

### Lỗi Built-in Của Step Functions

| Error Code | Ý Nghĩa |
|-----------|---------|
| `States.ALL` | Khớp với **mọi** loại lỗi — dùng làm catch-all |
| `States.Timeout` | Task không hoàn thành trong `TimeoutSeconds` |
| `States.TaskFailed` | Task trả về lỗi (bao gồm Lambda exception) |
| `States.HeartbeatTimeout` | Task không gửi heartbeat trong `HeartbeatSeconds` |
| `States.NoChoiceMatched` | Choice state không khớp rule nào và không có Default |
| `States.PermissionDenied` | Step Functions không có quyền gọi resource |
| `States.ResultPathMatchFailure` | ResultPath không khớp cấu trúc output |
| `States.BranchFailed` | Một nhánh trong Parallel state thất bại |
| `States.ExceedToleratedFailureThreshold` | Map state vượt quá ngưỡng thất bại |
| `States.ItemReaderFailed` | Lỗi đọc item trong Map state Distributed Mode |
| `States.IntrinsicFailure` | Hàm intrinsic (States.Format, States.JsonToString…) gặp lỗi |

### Lỗi Từ Lambda

| Error Code | Ý Nghĩa |
|-----------|---------|
| `Lambda.ServiceException` | Lỗi phía AWS Lambda service |
| `Lambda.AWSLambdaException` | Lambda trả về exception |
| `Lambda.SdkClientException` | Lỗi SDK client khi gọi Lambda |
| `Lambda.TooManyRequestsException` | Lambda bị throttle (giới hạn tốc độ) |

### Custom Error Từ Lambda

Lambda có thể ném exception với tên tùy chỉnh. Step Functions bắt theo tên class exception:

```python
# Python Lambda
class OrderNotFoundError(Exception):
    pass

class InsufficientInventoryError(Exception):
    pass

def lambda_handler(event, context):
    order_id = event['orderId']
    if not find_order(order_id):
        raise OrderNotFoundError(f"Order {order_id} not found")
    
    if not check_inventory(event['items']):
        raise InsufficientInventoryError("Not enough stock")
    
    return process_order(event)
```

```json
{
  "Retry": [
    {
      "ErrorEquals": ["InsufficientInventoryError"],
      "MaxAttempts": 0
    }
  ],
  "Catch": [
    {
      "ErrorEquals": ["OrderNotFoundError"],
      "Next": "TraLoiKhongTimThay",
      "ResultPath": "$.error"
    },
    {
      "ErrorEquals": ["InsufficientInventoryError"],
      "Next": "TraLoiHetHang",
      "ResultPath": "$.error"
    }
  ]
}
```

---

## 🎣 Catch — Bắt Lỗi Và Xử Lý

### Cú Pháp Cơ Bản

```json
{
  "XuLyThanhToan": {
    "Type": "Task",
    "Resource": "arn:aws:lambda:ap-southeast-1:123:function:ProcessPayment",
    "Retry": [
      {
        "ErrorEquals": ["Lambda.ServiceException"],
        "MaxAttempts": 3,
        "IntervalSeconds": 2,
        "BackoffRate": 2.0
      }
    ],
    "Catch": [
      {
        "ErrorEquals": ["PaymentDeclinedError"],
        "Next": "ThongBaoThanhToanThatBai",
        "ResultPath": "$.paymentError"
      },
      {
        "ErrorEquals": ["States.Timeout"],
        "Next": "XuLyTimeout",
        "ResultPath": "$.timeoutError"
      },
      {
        "ErrorEquals": ["States.ALL"],
        "Next": "XuLyLoiChung",
        "ResultPath": "$.generalError"
      }
    ],
    "Next": "XacNhanDonHang"
  }
}
```

### Thứ Tự Ưu Tiên Trong Catch

Step Functions khớp lần lượt từ trên xuống dưới. **Luôn đặt lỗi cụ thể trước, `States.ALL` sau cùng:**

```json
"Catch": [
  {
    "ErrorEquals": ["PaymentDeclinedError"],
    "Next": "XuLyThanhToanTuChoi"
  },
  {
    "ErrorEquals": ["ValidationError"],
    "Next": "XuLyLuThietHop"
  },
  {
    "ErrorEquals": ["States.Timeout"],
    "Next": "XuLyTimeout"
  },
  {
    "ErrorEquals": ["States.ALL"],
    "Next": "XuLyLoiChung"
  }
]
```

### ResultPath Trong Catch — Giữ Lại Thông Tin Lỗi

```json
"Catch": [
  {
    "ErrorEquals": ["States.ALL"],
    "Next": "XuLyLoi",
    "ResultPath": "$.errorInfo"
  }
]
```

```
Input gốc:       { "orderId": "123", "userId": "u456" }

Khi bắt được lỗi:
Lỗi object:      { "Error": "PaymentDeclinedError", "Cause": "Card expired" }

Output sang XuLyLoi:
{
  "orderId": "123",
  "userId": "u456",
  "errorInfo": {
    "Error": "PaymentDeclinedError",
    "Cause": "Card expired"
  }
}

Nếu ResultPath: null → chỉ có error object, mất input gốc
Nếu không có ResultPath → chỉ có error object (mặc định)
```

---

## 🏗️ Ví Dụ Thực Tế: Xử Lý Đơn Hàng Toàn Diện

```json
{
  "Comment": "Workflow xử lý đơn hàng với error handling đầy đủ",
  "StartAt": "XacThucDonHang",
  "States": {
    "XacThucDonHang": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "ValidateOrder",
        "Payload.$": "$"
      },
      "ResultSelector": {
        "validOrder.$": "$.Payload"
      },
      "ResultPath": "$.validation",
      "Retry": [
        {
          "ErrorEquals": ["Lambda.ServiceException", "Lambda.AWSLambdaException"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0,
          "JitterStrategy": "FULL"
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["ValidationError"],
          "Next": "TraLoiLuThietHop",
          "ResultPath": "$.error"
        },
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "LogLoiHeThong",
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
        },
        {
          "ErrorEquals": ["Lambda.ServiceException"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["PaymentDeclinedError"],
          "Next": "ThongBaoThanhToanThatBai",
          "ResultPath": "$.error"
        },
        {
          "ErrorEquals": ["States.Timeout"],
          "Next": "HoiLaiTrangThaiThanhToan",
          "ResultPath": "$.error"
        },
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "HoanTienVaThongBao",
          "ResultPath": "$.error"
        }
      ],
      "TimeoutSeconds": 30,
      "Next": "XacNhanKho"
    },

    "ThongBaoThanhToanThatBai": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:ap-southeast-1:123:PaymentFailedTopic",
        "Message": {
          "orderId.$": "$.orderId",
          "reason.$": "$.error.Cause"
        }
      },
      "Next": "KetThucThatBai"
    },

    "HoanTienVaThongBao": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "RefundPayment",
        "Payload.$": "$"
      },
      "Next": "KetThucLoi"
    },

    "LogLoiHeThong": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "ErrorLogs",
        "Item": {
          "executionId": { "S.$": "$$.Execution.Id" },
          "timestamp": { "S.$": "$$.Execution.StartTime" },
          "error": { "S.$": "States.JsonToString($.error)" }
        }
      },
      "Next": "KetThucLoi"
    },

    "XacNhanKho": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "ConfirmInventory",
        "Payload.$": "$"
      },
      "End": true
    },

    "TraLoiLuThietHop": {
      "Type": "Fail",
      "ErrorPath": "$.error.Error",
      "CausePath": "$.error.Cause"
    },

    "KetThucThatBai": {
      "Type": "Fail",
      "Error": "PaymentFailed",
      "Cause": "Thanh toán bị từ chối sau khi đã thông báo khách hàng"
    },

    "KetThucLoi": {
      "Type": "Fail",
      "Error": "SystemError",
      "Cause": "Lỗi hệ thống nghiêm trọng, đã log và cảnh báo"
    },

    "HoiLaiTrangThaiThanhToan": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "CheckPaymentStatus",
        "Payload.$": "$"
      },
      "End": true
    }
  }
}
```

---

## ⚡ TimeoutSeconds vs HeartbeatSeconds

```json
{
  "XuLyFileLon": {
    "Type": "Task",
    "Resource": "arn:aws:states:::ecs:runTask.sync",
    "Parameters": { "..." : "..." },
    "TimeoutSeconds": 3600,
    "HeartbeatSeconds": 300,
    "Retry": [
      {
        "ErrorEquals": ["States.HeartbeatTimeout"],
        "MaxAttempts": 2,
        "IntervalSeconds": 10
      }
    ]
  }
}
```

| Tham Số | Ý Nghĩa |
|---------|---------|
| `TimeoutSeconds` | Thời gian tối đa cho cả task (bao gồm retry). Nếu vượt quá → `States.Timeout` |
| `HeartbeatSeconds` | Task phải gửi heartbeat trong khoảng thời gian này. Nếu không → `States.HeartbeatTimeout` |

> **Dùng HeartbeatSeconds khi:** Task chạy lâu (ECS, Glue, EMR) và bạn muốn phát hiện khi task bị treo (hung) sớm hơn là đợi hết timeout.

---

## 📋 Best Practices (Thực Hành Tốt Nhất)

### 1. Retry Chỉ Lỗi Có Thể Phục Hồi

```json
// ✅ Tốt: Retry lỗi tạm thời
"Retry": [
  {
    "ErrorEquals": [
      "Lambda.ServiceException",
      "Lambda.AWSLambdaException",
      "Lambda.TooManyRequestsException",
      "States.Timeout"
    ],
    "IntervalSeconds": 2,
    "MaxAttempts": 3,
    "BackoffRate": 2.0
  }
]

// ❌ Không nên: Retry lỗi business logic
"Retry": [
  {
    "ErrorEquals": ["States.ALL"],  // Cũng retry cả ValidationError!
    "MaxAttempts": 3
  }
]
```

### 2. Dùng Jitter Cho Distributed Systems

```json
"Retry": [
  {
    "ErrorEquals": ["Lambda.TooManyRequestsException"],
    "IntervalSeconds": 1,
    "MaxAttempts": 5,
    "BackoffRate": 2.0,
    "MaxDelaySeconds": 30,
    "JitterStrategy": "FULL"
  }
]
```

### 3. Luôn Có Catch States.ALL

```json
"Catch": [
  { "ErrorEquals": ["SpecificError1"], "Next": "HandleSpecific1" },
  { "ErrorEquals": ["SpecificError2"], "Next": "HandleSpecific2" },
  { "ErrorEquals": ["States.ALL"], "Next": "HandleUnexpectedError" }
]
```

### 4. Giữ Context Khi Bắt Lỗi

```json
"Catch": [
  {
    "ErrorEquals": ["States.ALL"],
    "Next": "XuLyLoi",
    "ResultPath": "$.error"
  }
]
// → XuLyLoi nhận được cả input gốc + thông tin lỗi
```

### 5. Set TimeoutSeconds Cho Mọi Task

```json
{
  "Type": "Task",
  "TimeoutSeconds": 30,
  "Retry": [
    {
      "ErrorEquals": ["States.Timeout"],
      "MaxAttempts": 2
    }
  ]
}
```

---

## 🔀 Retry vs Catch — Thứ Tự Xử Lý

```
Task thực thi
      │
      ▼
  Lỗi xảy ra
      │
      ▼
  Kiểm tra Retry
  ├── Khớp và còn lượt? → Thử lại (chờ IntervalSeconds * BackoffRate^attempt)
  └── Không khớp hoặc hết MaxAttempts?
          │
          ▼
      Kiểm tra Catch
      ├── Khớp? → Chuyển sang Next state (với error info)
      └── Không khớp? → Execution thất bại toàn bộ
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Retry và Catch trong Step Functions giải quyết vấn đề gì so với tự implement trong Lambda?**

> Khi retry logic nằm trong Lambda, nó tốn thời gian tính phí và chiếm memory trong khi chờ. Step Functions retry ở cấp orchestration — Lambda không chạy trong thời gian chờ, tiết kiệm chi phí Lambda đáng kể. Ngoài ra, retry logic được khai báo rõ ràng trong state machine definition, dễ đọc và điều chỉnh mà không cần sửa code Lambda.

**Q: Tại sao nên dùng JitterStrategy: FULL?**

> Khi nhiều execution bị lỗi cùng lúc (ví dụ: downstream service down), nếu không có jitter, tất cả sẽ retry cùng lúc sau mỗi interval — tạo ra traffic spike có thể làm quá tải service đang hồi phục. Jitter phân tán retry theo thời gian ngẫu nhiên, giúp service downstream có cơ hội phục hồi dần dần.

**Q: ResultPath: null trong Catch nghĩa là gì?**

> Khi `ResultPath: null`, Step Functions loại bỏ error object và state tiếp theo chỉ nhận được input gốc — không có thông tin về lỗi. Thường dùng khi muốn "bỏ qua lỗi" mà không cần biết chi tiết. Tuy nhiên, thực hành tốt là luôn ghi lại thông tin lỗi qua `ResultPath: "$.error"`.

---

## 🔗 Điều Hướng

- **Quay lại:** [2-standard-vs-express.md](./2-standard-vs-express.md)
- **Tiếp theo:** [4-callback-pattern.md](./4-callback-pattern.md) — Callback với waitForTaskToken
- **Liên quan:** [5-saga-orchestration.md](./5-saga-orchestration.md) — Error handling trong Saga

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
