# State Types — Các Loại Trạng Thái Trong Step Functions

> Step Functions cung cấp **8 loại state** (trạng thái), mỗi loại phục vụ một mục đích khác nhau trong việc xây dựng workflow (luồng công việc). Hiểu đúng từng loại là chìa khóa để thiết kế state machine hiệu quả.

---

## 📋 Tổng Quan 8 Loại State

| State | Mục Đích | Tần Suất Dùng |
|-------|---------|--------------|
| **Task** | Thực hiện công việc — gọi Lambda, ECS, API… | ⭐⭐⭐ Rất thường |
| **Choice** | Rẽ nhánh theo điều kiện (if/else) | ⭐⭐⭐ Rất thường |
| **Parallel** | Chạy song song nhiều nhánh | ⭐⭐⭐ Thường |
| **Map** | Lặp qua từng phần tử của mảng | ⭐⭐⭐ Thường |
| **Wait** | Tạm dừng một khoảng thời gian | ⭐⭐ Trung bình |
| **Pass** | Chuyển tiếp input sang output (có thể transform) | ⭐⭐ Trung bình |
| **Succeed** | Kết thúc execution thành công | ⭐ Ít dùng |
| **Fail** | Kết thúc execution với lỗi | ⭐ Ít dùng |

---

## 1. Task State — Tác Vụ

**Task** là loại state phổ biến nhất. Nó gọi một **resource** (tài nguyên) bên ngoài và đợi kết quả.

### Cú Pháp Cơ Bản

```json
{
  "XuLyDonHang": {
    "Type": "Task",
    "Resource": "arn:aws:lambda:ap-southeast-1:123456789:function:ProcessOrder",
    "Parameters": {
      "orderId.$": "$.orderId",
      "userId.$": "$.userId"
    },
    "ResultPath": "$.orderResult",
    "TimeoutSeconds": 30,
    "HeartbeatSeconds": 10,
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
        "ErrorEquals": ["States.ALL"],
        "Next": "XuLyLoi"
      }
    ],
    "Next": "ThanhToan"
  }
}
```

### Các Resource Được Hỗ Trợ

```
Lambda Function         → arn:aws:states:::lambda:invoke
ECS Task               → arn:aws:states:::ecs:runTask.sync
DynamoDB PutItem       → arn:aws:states:::dynamodb:putItem
SQS SendMessage        → arn:aws:states:::sqs:sendMessage
SNS Publish            → arn:aws:states:::sns:publish
API Gateway            → arn:aws:states:::apigateway:invoke
Bedrock InvokeModel    → arn:aws:states:::bedrock:invokeModel
Glue StartJobRun       → arn:aws:states:::glue:startJobRun.sync
Athena StartQuery      → arn:aws:states:::athena:startQueryExecution.sync
```

### InputPath, ResultPath, OutputPath — Kiểm Soát Dữ Liệu

```
Input gốc: { "orderId": "123", "userId": "u456", "amount": 100 }

InputPath: "$.orderId"        → Chỉ gửi orderId vào Task
Parameters: { ... }           → Tùy chỉnh input trước khi gửi
ResultPath: "$.taskResult"    → Gắn kết quả vào field này của input gốc
OutputPath: "$.taskResult"    → Chỉ giữ lại phần này làm output
```

```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "Parameters": {
    "FunctionName": "MyFunction",
    "Payload": {
      "orderId.$": "$.orderId"
    }
  },
  "ResultSelector": {
    "result.$": "$.Payload.result"
  },
  "ResultPath": "$.lambdaResult",
  "OutputPath": "$"
}
```

---

## 2. Choice State — Rẽ Nhánh Theo Điều Kiện

**Choice** tương đương `if/else` hoặc `switch` trong lập trình. Không có `Next` ở cấp state, thay vào đó mỗi rule (quy tắc) tự định nghĩa `Next`.

```json
{
  "KiemTraSoTien": {
    "Type": "Choice",
    "Choices": [
      {
        "Variable": "$.amount",
        "NumericLessThan": 1000000,
        "Next": "ThanhToanThuong"
      },
      {
        "Variable": "$.amount",
        "NumericGreaterThanEquals": 1000000,
        "Next": "ThanhToanVIP"
      },
      {
        "And": [
          { "Variable": "$.userType", "StringEquals": "premium" },
          { "Variable": "$.amount", "NumericGreaterThan": 500000 }
        ],
        "Next": "ThanhToanPremiumLon"
      }
    ],
    "Default": "XuLyMacDinh"
  }
}
```

### Các Toán Tử So Sánh Trong Choice

| Toán Tử | Kiểu Dữ Liệu | Ví Dụ |
|---------|-------------|-------|
| `StringEquals` | Chuỗi | `"StringEquals": "PENDING"` |
| `StringMatches` | Chuỗi (wildcard) | `"StringMatches": "order-*"` |
| `NumericEquals` | Số | `"NumericEquals": 100` |
| `NumericLessThan` | Số | `"NumericLessThan": 1000` |
| `NumericGreaterThanEquals` | Số | `"NumericGreaterThanEquals": 0` |
| `BooleanEquals` | Boolean | `"BooleanEquals": true` |
| `TimestampGreaterThan` | Thời gian | ISO 8601 format |
| `IsPresent` | Kiểm tra tồn tại | `"IsPresent": true` |
| `IsNull` | Kiểm tra null | `"IsNull": false` |
| `And` / `Or` / `Not` | Logic kết hợp | Lồng nhiều điều kiện |

> **Lưu ý:** Choice state **không có** `TimeoutSeconds` và **không** trực tiếp xử lý lỗi (không có `Catch`/`Retry`).

---

## 3. Parallel State — Xử Lý Song Song

**Parallel** chạy nhiều nhánh (branch) **cùng lúc** và đợi tất cả hoàn thành mới tiếp tục. Kết quả là một **mảng** chứa output của từng nhánh.

```json
{
  "XuLySongSong": {
    "Type": "Parallel",
    "Branches": [
      {
        "StartAt": "GuiEmail",
        "States": {
          "GuiEmail": {
            "Type": "Task",
            "Resource": "arn:aws:states:::sns:publish",
            "Parameters": {
              "TopicArn": "arn:aws:sns:ap-southeast-1:123:EmailTopic",
              "Message.$": "$.emailContent"
            },
            "End": true
          }
        }
      },
      {
        "StartAt": "CapNhatDatabase",
        "States": {
          "CapNhatDatabase": {
            "Type": "Task",
            "Resource": "arn:aws:states:::dynamodb:putItem",
            "Parameters": {
              "TableName": "Orders",
              "Item": {
                "orderId": { "S.$": "$.orderId" },
                "status": { "S": "PROCESSING" }
              }
            },
            "End": true
          }
        }
      },
      {
        "StartAt": "GuiWebhook",
        "States": {
          "GuiWebhook": {
            "Type": "Task",
            "Resource": "arn:aws:states:::http:invoke",
            "Parameters": {
              "ApiEndpoint": "https://partner.example.com/webhook",
              "Method": "POST"
            },
            "End": true
          }
        }
      }
    ],
    "ResultPath": "$.parallelResults",
    "Next": "TongKetDonHang"
  }
}
```

### Đặc Điểm Quan Trọng Của Parallel

- Nếu **một nhánh thất bại**, toàn bộ Parallel state thất bại ngay lập tức
- Kết quả trả về là **array** (mảng) — mỗi phần tử là output của một nhánh
- Mỗi nhánh là một **state machine độc lập** (có `StartAt` và `States` riêng)
- Có thể dùng `Catch` và `Retry` ở cấp Parallel state

---

## 4. Map State — Lặp Qua Mảng

**Map** xử lý từng phần tử trong một mảng (array) bằng cùng một state machine con. Tương đương `Array.map()` trong JavaScript nhưng chạy **song song** hoặc **tuần tự**.

```json
{
  "XuLyTungSanPham": {
    "Type": "Map",
    "ItemsPath": "$.products",
    "ItemSelector": {
      "productId.$": "$$.Map.Item.Value.productId",
      "quantity.$": "$$.Map.Item.Value.quantity",
      "orderId.$": "$.orderId"
    },
    "MaxConcurrency": 5,
    "Iterator": {
      "StartAt": "KiemTraTonKho",
      "States": {
        "KiemTraTonKho": {
          "Type": "Task",
          "Resource": "arn:aws:lambda:ap-southeast-1:123:function:CheckInventory",
          "End": true
        }
      }
    },
    "ResultPath": "$.inventoryResults",
    "Next": "TongKet"
  }
}
```

### Tham Số Quan Trọng

| Tham Số | Ý Nghĩa | Mặc Định |
|---------|---------|---------|
| `ItemsPath` | JSONPath trỏ đến mảng cần lặp | `$` (toàn bộ input) |
| `MaxConcurrency` | Số phần tử xử lý song song tối đa | `0` (không giới hạn) |
| `ItemSelector` | Transform từng item trước khi đưa vào iterator | — |
| `ToleratedFailureCount` | Cho phép tối đa N item thất bại | `0` |
| `ToleratedFailurePercentage` | Cho phép tối đa X% item thất bại | `0` |

> **Phân biệt Map vs Parallel:**
> - **Map:** Cùng một logic, áp dụng cho N phần tử dữ liệu khác nhau
> - **Parallel:** Logic khác nhau chạy cùng lúc trên cùng một input

---

## 5. Wait State — Tạm Dừng

**Wait** tạm dừng workflow trong một khoảng thời gian nhất định hoặc đến một thời điểm cụ thể.

```json
{
  "ChoiSauKhiGuiEmail": {
    "Type": "Wait",
    "Seconds": 300,
    "Next": "KiemTraPhanHoi"
  }
}
```

```json
{
  "DoiDenNgayGiaoHang": {
    "Type": "Wait",
    "TimestampPath": "$.deliveryDate",
    "Next": "XacNhanGiaoHang"
  }
}
```

### Các Kiểu Wait

| Kiểu | Ví Dụ | Mô Tả |
|------|-------|-------|
| `Seconds` | `"Seconds": 60` | Đợi N giây |
| `SecondsPath` | `"SecondsPath": "$.waitSeconds"` | Đợi N giây (lấy từ input) |
| `Timestamp` | `"Timestamp": "2026-06-01T00:00:00Z"` | Đợi đến thời điểm cố định |
| `TimestampPath` | `"TimestampPath": "$.scheduledAt"` | Đợi đến thời điểm lấy từ input |

> **Lưu ý chi phí:** Trong Standard Workflow, thời gian Wait không tính phí riêng — chỉ tính phí state transitions. Nhưng Max execution duration là 1 năm.

---

## 6. Pass State — Chuyển Tiếp / Biến Đổi Dữ Liệu

**Pass** không làm gì ngoài việc chuyển input thành output, có thể kèm theo dữ liệu tĩnh. Dùng để:
- Inject (chèn thêm) dữ liệu tĩnh vào luồng
- Transform dữ liệu không cần Lambda
- Debug (gỡ lỗi) — chèn vào giữa để kiểm tra dữ liệu
- Placeholder (vị trí giữ chỗ) trong giai đoạn phát triển

```json
{
  "ThemThongTinMacDinh": {
    "Type": "Pass",
    "Parameters": {
      "orderId.$": "$.orderId",
      "status": "PENDING",
      "environment": "production",
      "processedAt.$": "$$.Execution.StartTime"
    },
    "ResultPath": "$.enrichedData",
    "Next": "XuLyDonHang"
  }
}
```

```json
{
  "BuocPlaceholder": {
    "Type": "Pass",
    "Result": {
      "message": "Feature chưa implement, bỏ qua"
    },
    "Next": "BuocTiepTheo"
  }
}
```

---

## 7. Succeed State — Kết Thúc Thành Công

**Succeed** kết thúc execution thành công ngay lập tức. Thường dùng ở cuối nhánh Choice hoặc sau khi hoàn thành luồng xử lý.

```json
{
  "HoanThanh": {
    "Type": "Succeed",
    "Comment": "Đơn hàng đã được xử lý thành công"
  }
}
```

```json
{
  "TruongHopKhongXuLy": {
    "Type": "Choice",
    "Choices": [
      {
        "Variable": "$.status",
        "StringEquals": "CANCELLED",
        "Next": "BequietlySucceed"
      }
    ],
    "Default": "XuLyBinhThuong"
  },
  "BequietlySucceed": {
    "Type": "Succeed"
  }
}
```

---

## 8. Fail State — Kết Thúc Với Lỗi

**Fail** kết thúc execution với trạng thái thất bại và cung cấp thông tin lỗi.

```json
{
  "LuongThatBai": {
    "Type": "Fail",
    "Error": "InvalidOrderError",
    "Cause": "Đơn hàng không hợp lệ: số lượng âm hoặc sản phẩm không tồn tại"
  }
}
```

```json
{
  "LuongThatBaiDong": {
    "Type": "Fail",
    "ErrorPath": "$.errorCode",
    "CausePath": "$.errorMessage"
  }
}
```

> Sau khi Fail state, execution không thể tiếp tục. Dùng `Catch` trong Task state để bắt lỗi và chuyển sang trạng thái xử lý lỗi thay vì dùng Fail ngay.

---

## 🔄 Luồng Dữ Liệu Giữa Các State

```
                    ┌───────────┐
                    │  Input    │
                    │  (JSON)   │
                    └─────┬─────┘
                          │
                    ┌─────▼─────┐
                    │ InputPath │  Lọc phần nào của input gửi vào state
                    └─────┬─────┘
                          │
                    ┌─────▼─────┐
                    │Parameters │  Tùy chỉnh, đổi tên, thêm giá trị tĩnh
                    └─────┬─────┘
                          │
                    ┌─────▼─────┐
                    │   State   │  Xử lý (Task, Choice, Parallel…)
                    │ Execution │
                    └─────┬─────┘
                          │
                    ┌─────▼──────┐
                    │ResultSelect│  Lọc kết quả trả về từ state
                    └─────┬──────┘
                          │
                    ┌─────▼──────┐
                    │ResultPath  │  Gắn kết quả vào đâu trong input gốc
                    └─────┬──────┘
                          │
                    ┌─────▼──────┐
                    │OutputPath  │  Chọn phần nào của tổng hợp làm output
                    └─────┬──────┘
                          │
                    ┌─────▼──────┐
                    │   Output   │
                    │  (JSON)    │
                    └────────────┘
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Sự khác biệt giữa Parallel và Map state?**

> - **Parallel:** N nhánh logic khác nhau chạy song song trên cùng input
> - **Map:** Cùng logic áp dụng cho N phần tử trong một mảng

**Q: Khi nào dùng Pass state thay vì Lambda?**

> Dùng Pass khi chỉ cần inject dữ liệu tĩnh, đổi tên field, hoặc làm placeholder — không cần tính toán. Pass rẻ hơn (0 chi phí compute) và nhanh hơn Lambda cold start.

**Q: ResultPath: `null` có ý nghĩa gì?**

> Khi `ResultPath` là `null`, kết quả của Task bị loại bỏ và output là toàn bộ input gốc không đổi. Dùng khi muốn gọi service nhưng không quan tâm kết quả trả về.

**Q: MaxConcurrency trong Map state nên đặt bao nhiêu?**

> Phụ thuộc vào service downstream (phía dưới). Nếu downstream là Lambda: để cao (10–100). Nếu là DynamoDB: để theo write capacity units. Nếu là hệ thống bên thứ ba có rate limit: đặt thấp (1–5).

---

## 🔗 Điều Hướng

- **Quay lại:** [README.md](./README.md)
- **Tiếp theo:** [2-standard-vs-express.md](./2-standard-vs-express.md) — Standard vs Express Workflow
- **Liên quan:** [3-error-handling.md](./3-error-handling.md) — Xử lý lỗi với Catch & Retry

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
