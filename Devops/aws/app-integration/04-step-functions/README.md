# AWS Step Functions — Tổng Quan & Lộ Trình Học

> **Step Functions** là dịch vụ điều phối workflow (luồng công việc) serverless (không máy chủ) của AWS, cho phép bạn xây dựng và chạy **state machine** (máy trạng thái) để điều phối các bước xử lý một cách trực quan, có thể kiểm soát lỗi, và hoàn toàn được quản lý.

---

## 📚 Mục Lục Module Này

| File | Chủ Đề | Độ Ưu Tiên |
|------|--------|-----------|
| [README.md](./README.md) | Tổng quan, khi nào dùng | Đọc đầu tiên |
| [1-state-types.md](./1-state-types.md) | Các loại State: Task, Choice, Parallel… | ⭐⭐⭐ |
| [2-standard-vs-express.md](./2-standard-vs-express.md) | Standard vs Express Workflow | ⭐⭐⭐ |
| [3-error-handling.md](./3-error-handling.md) | Catch & Retry — Xử Lý Lỗi | ⭐⭐⭐ |
| [4-callback-pattern.md](./4-callback-pattern.md) | Callback Pattern với waitForTaskToken | ⭐⭐ |
| [5-saga-orchestration.md](./5-saga-orchestration.md) | Saga Pattern với Step Functions | ⭐⭐⭐ |

---

## 🎯 Step Functions Là Gì?

**AWS Step Functions** cho phép bạn mô hình hóa quy trình kinh doanh (business process) thành một **state machine** — một tập hợp các trạng thái (state) và chuyển tiếp (transition) giữa chúng. Mỗi bước trong quy trình là một **state**, và Step Functions quản lý việc:

- Thực thi tuần tự hoặc song song các bước
- Xử lý lỗi và retry (thử lại) tự động
- Lưu trữ toàn bộ lịch sử thực thi (execution history)
- Tích hợp với hơn 200 dịch vụ AWS

```
Bạn định nghĩa luồng công việc bằng ngôn ngữ JSON (Amazon States Language — ASL)
AWS Step Functions thực thi, giám sát, và ghi log toàn bộ quá trình
```

---

## 🏗️ Kiến Trúc Cốt Lõi

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS Step Functions                        │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   State 1    │───▶│   State 2    │───▶│   State 3    │  │
│  │  (Task)      │    │  (Choice)    │    │  (Parallel)  │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│         │                  │                    │           │
│         ▼                  ▼                    ▼           │
│     Lambda /          Nhánh A/B           Task A || Task B  │
│   ECS / API           (điều kiện)         (song song)       │
└─────────────────────────────────────────────────────────────┘
```

### Các Khái Niệm Cốt Lõi

| Khái Niệm | Giải Thích |
|-----------|-----------|
| **State Machine** (Máy Trạng Thái) | Định nghĩa toàn bộ luồng công việc dưới dạng JSON (ASL — Amazon States Language) |
| **State** (Trạng Thái) | Một bước trong workflow — Task, Choice, Wait, Parallel, Map, Pass, Succeed, Fail |
| **Execution** (Thực Thi) | Một lần chạy cụ thể của state machine với input nhất định |
| **Task** (Tác Vụ) | State gọi một service bên ngoài: Lambda, ECS, DynamoDB, SQS… |
| **Transition** (Chuyển Tiếp) | Di chuyển từ state này sang state tiếp theo |
| **Input / Output** | Mỗi state nhận input JSON và trả ra output JSON |

---

## 💡 Khi Nào Dùng Step Functions?

### Nên Dùng Khi

| Tình Huống | Ví Dụ |
|-----------|-------|
| **Orchestration** (Điều Phối) nhiều service | Đặt hàng → Thanh toán → Giao hàng → Thông báo |
| **Long-running workflow** (Luồng Chạy Dài) | Phê duyệt đơn vay — chờ người dùng phản hồi hàng ngày |
| **Parallel processing** (Xử Lý Song Song) | Xử lý hình ảnh: resize + watermark + upload cùng lúc |
| **Distributed transactions** (Giao Dịch Phân Tán) | Saga Pattern — rollback khi một bước thất bại |
| **Complex retry logic** (Logic Thử Lại Phức Tạp) | Retry với exponential backoff, fallback phức tạp |
| **Audit trail** (Dấu Vết Kiểm Toán) | Cần lưu lịch sử từng bước để tuân thủ compliance |

### Không Nên Dùng Khi

| Tình Huống | Thay Bằng |
|-----------|---------|
| Simple event routing (Định Tuyến Sự Kiện Đơn Giản) | EventBridge |
| Pub/Sub notification (Thông Báo) | SNS |
| Task queue (Hàng Đợi Tác Vụ) | SQS |
| Real-time streaming (Truyền Phát Thời Gian Thực) | Kinesis |
| Latency < 100ms (Độ Trễ Rất Thấp) | Lambda trực tiếp |

---

## ⚙️ Tích Hợp Dịch Vụ

Step Functions hỗ trợ hai kiểu tích hợp:

### 1. Optimized Integrations (Tích Hợp Tối Ưu)

Gọi trực tiếp API của service AWS mà không cần Lambda làm trung gian:

```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::dynamodb:putItem",
  "Parameters": {
    "TableName": "Orders",
    "Item": {
      "orderId": { "S.$": "$.orderId" },
      "status": { "S": "PENDING" }
    }
  }
}
```

Hỗ trợ: **DynamoDB**, **SQS**, **SNS**, **ECS**, **Lambda**, **Glue**, **Athena**, **Bedrock**…

### 2. Request-Response vs .sync vs .waitForTaskToken

| Kiểu Gọi | Ý Nghĩa | Dùng Khi |
|----------|---------|---------|
| **Request-Response** | Gửi yêu cầu và tiếp tục ngay | Không cần đợi kết quả |
| **`.sync`** | Đợi tác vụ hoàn thành | Lambda, ECS task chạy xong |
| **`.waitForTaskToken`** | Đợi callback từ bên ngoài | Con người phê duyệt, hệ thống bên thứ ba |

---

## 🔄 Hai Loại Workflow

```
Standard Workflow          Express Workflow
──────────────────         ─────────────────
Thời gian: 1 năm           Thời gian: 5 phút
Tính phí: theo state       Tính phí: theo số lần + thời gian
Đúng một lần (exactly-once) Ít nhất một lần (at-least-once)
Lịch sử: lưu đầy đủ        Lịch sử: CloudWatch Logs
Phù hợp: long-running      Phù hợp: high-volume, event-driven
```

Chi tiết: [2-standard-vs-express.md](./2-standard-vs-express.md)

---

## 🛡️ Error Handling (Xử Lý Lỗi)

Step Functions cung cấp cơ chế xử lý lỗi mạnh mẽ:

- **Retry** (Thử Lại) — tự động thử lại với exponential backoff (tăng thời gian chờ theo hàm mũ)
- **Catch** (Bắt Lỗi) — chuyển sang state xử lý lỗi khi vượt quá số lần retry
- **Fallback State** (Trạng Thái Dự Phòng) — luồng thay thế khi lỗi
- **Compensating Transaction** (Giao Dịch Bù Trừ) — undo các bước đã thực hiện trong Saga

Chi tiết: [3-error-handling.md](./3-error-handling.md)

---

## 📋 Amazon States Language (ASL) — Ngôn Ngữ Định Nghĩa State

Step Functions dùng JSON với cú pháp đặc biệt:

```json
{
  "Comment": "Ví dụ state machine đơn giản",
  "StartAt": "XuLyDonHang",
  "States": {
    "XuLyDonHang": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:ap-southeast-1:123:function:ProcessOrder",
      "Next": "ThanhToan"
    },
    "ThanhToan": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:ap-southeast-1:123:function:ProcessPayment",
      "End": true
    }
  }
}
```

### Toán Tử Tham Chiếu JSON

| Ký Hiệu | Ý Nghĩa |
|---------|---------|
| `$.field` | Tham chiếu field trong input hiện tại |
| `$$.Execution.Id` | Metadata của execution (ID thực thi) |
| `States.Format(…)` | Định dạng chuỗi |
| `States.JsonToString(…)` | Chuyển JSON thành chuỗi |

---

## 💰 Chi Phí (Pricing)

### Standard Workflow

- **$0.025** cho mỗi 1,000 state transitions (chuyển đổi trạng thái)
- Free tier: 4,000 state transitions/tháng

### Express Workflow

- **$1.00** cho mỗi 1,000,000 lần thực thi (execution)
- **$0.00001** cho mỗi GB-giây thời gian thực thi
- Free tier: 1,000 lần thực thi/tháng

---

## 🧰 Công Cụ Thực Hành

| Công Cụ | Mục Đích |
|---------|---------|
| **AWS Console — Workflow Studio** | Visual editor kéo thả để tạo state machine |
| **AWS CDK / CloudFormation** | Infrastructure as Code cho state machine |
| **AWS SAM** | Deploy serverless application với Step Functions |
| **Step Functions Local** | Chạy thử trên máy local trước khi deploy |
| **AWS X-Ray** | Distributed tracing (Theo Dõi Phân Tán) toàn luồng |

---

## 🎓 Checklist Học Tập

### Mới Bắt Đầu

- [ ] Tạo state machine đầu tiên trên AWS Console (Workflow Studio)
- [ ] Hiểu 8 loại state và khi nào dùng từng loại
- [ ] Phân biệt Standard và Express Workflow
- [ ] Chạy thử một execution và xem execution history

### Trung Cấp

- [ ] Implement Catch và Retry trong state machine
- [ ] Dùng optimized integration với DynamoDB và SQS (không qua Lambda)
- [ ] Implement Callback Pattern với waitForTaskToken
- [ ] Xây dựng Parallel state và Map state

### Nâng Cao

- [ ] Thiết kế Saga Pattern với compensating transactions
- [ ] Cross-account Step Functions execution
- [ ] Cost optimization: chọn đúng workflow type
- [ ] Monitoring với CloudWatch và X-Ray

---

## 🔗 Điều Hướng Tiếp Theo

- **Tiếp theo:** [1-state-types.md](./1-state-types.md) — Tìm hiểu 8 loại State
- **Sau đó:** [2-standard-vs-express.md](./2-standard-vs-express.md) — Chọn đúng workflow type
- **Thực hành ngay:** Tạo state machine trên AWS Console > Step Functions > Workflow Studio

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
