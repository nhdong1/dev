# Standard vs Express Workflow — So Sánh Hai Loại Workflow

> AWS Step Functions cung cấp hai loại workflow (luồng công việc): **Standard** và **Express**. Chọn đúng loại ảnh hưởng trực tiếp đến chi phí, hiệu năng và độ tin cậy của hệ thống.

---

## 📊 So Sánh Tổng Quan

| Tiêu Chí | Standard Workflow | Express Workflow |
|---------|-------------------|-----------------|
| **Thời Gian Tối Đa** | 1 năm | 5 phút |
| **Mô Hình Tính Phí** | Theo số state transitions (chuyển đổi trạng thái) | Theo số lần thực thi + thời gian chạy |
| **Đảm Bảo Thực Thi** | Exactly-once (đúng một lần) | At-least-once (ít nhất một lần) |
| **Tốc Độ Thực Thi** | Chậm hơn (audit overhead) | Nhanh hơn (low-latency) |
| **Execution Rate** | 2,000/giây (mặc định) | 100,000/giây |
| **Lịch Sử Thực Thi** | Lưu đầy đủ 90 ngày trong console | CloudWatch Logs (cấu hình thêm) |
| **Idempotency** (Tính Bất Biến) | Có (execution ID duy nhất) | Không đảm bảo |
| **Phù Hợp Nhất** | Long-running, business-critical | High-volume, event processing |

---

## 🔷 Standard Workflow — Luồng Chuẩn

### Đặc Điểm

**Standard Workflow** được thiết kế cho các quy trình kinh doanh quan trọng, chạy dài, cần lịch sử kiểm toán đầy đủ.

```
Ưu điểm:
✅ Exactly-once execution — không bao giờ chạy một state hai lần
✅ Lịch sử đầy đủ trên AWS Console — debug dễ dàng
✅ Thời gian chạy đến 1 năm — phù hợp quy trình chờ phê duyệt dài
✅ Hỗ trợ waitForTaskToken — callback từ hệ thống bên ngoài
✅ Phù hợp với Saga Pattern

Hạn chế:
❌ Giá cao hơn Express ($0.025/1,000 state transitions)
❌ Không phù hợp high-volume (rate limit 2,000 execution/giây)
❌ Không phù hợp latency-sensitive use case (có overhead)
```

### Mô Hình Tính Phí Standard

```
$0.025 / 1,000 state transitions

Ví dụ workflow 5 bước:
- 1 execution = ~5 state transitions
- 1,000 executions/tháng = 5,000 transitions = $0.125
- 100,000 executions/tháng = 500,000 transitions = $12.50

Free tier: 4,000 state transitions/tháng
```

### Use Cases (Trường Hợp Sử Dụng) Điển Hình

- **Order Processing** (Xử Lý Đơn Hàng) — thanh toán, kho, giao hàng, thông báo
- **Human Approval Workflow** (Luồng Phê Duyệt Người Dùng) — duyệt khoản vay, duyệt nội dung
- **Document Processing** (Xử Lý Tài Liệu) — OCR, phân loại, lưu trữ
- **Saga Pattern** — distributed transactions cần rollback chính xác
- **Compliance Workflow** (Luồng Tuân Thủ) — cần audit trail đầy đủ

---

## 🔶 Express Workflow — Luồng Biểu Hiện

### Hai Chế Độ Con

**Express Workflow** có thêm hai chế độ:

```
Express Workflow
├── Asynchronous (Bất Đồng Bộ) — mặc định
│   └── Bắt đầu và không đợi kết quả
│       Caller nhận execution ARN ngay lập tức
│       Dùng cho: event processing, fire-and-forget
│
└── Synchronous (Đồng Bộ)
    └── Chờ đến khi execution hoàn thành rồi trả kết quả
        Timeout tối đa: 5 phút
        Dùng cho: real-time API với logic phức tạp
```

### Đặc Điểm

```
Ưu điểm:
✅ Giá thấp hơn nhiều cho high-volume use case
✅ Rate cao hơn: 100,000+ executions/giây
✅ Đồng bộ (synchronous mode): trả kết quả trực tiếp cho caller
✅ Phù hợp event-driven, streaming data processing

Hạn chế:
❌ At-least-once — có thể chạy state nhiều hơn một lần khi lỗi
❌ Thời gian tối đa 5 phút
❌ Lịch sử phải cấu hình qua CloudWatch Logs (không có trong console)
❌ Không hỗ trợ waitForTaskToken
❌ Không phù hợp Saga Pattern (không exactly-once)
```

### Mô Hình Tính Phí Express

```
$1.00 / 1,000,000 lần thực thi (execution requests)
$0.00001 / GB-giây thời gian chạy

Ví dụ:
- 1,000,000 executions/tháng, mỗi cái chạy 1 giây, dùng 64MB:
  - Execution cost: 1,000,000 / 1,000,000 * $1.00 = $1.00
  - Duration cost: 1,000,000 * 1s * (64/1024)GB * $0.00001 = $0.625
  - Tổng: ~$1.625

So với Standard:
- 1,000,000 executions * 5 transitions = 5,000,000 transitions
- $0.025 / 1,000 * 5,000,000 = $125.00

→ Express rẻ hơn ~75 lần cho high-volume!

Free tier: 1,000 lần thực thi/tháng
```

---

## 🎯 Chọn Loại Nào?

### Flowchart Quyết Định

```
                    ┌─────────────────────────────┐
                    │ Workflow cần chạy > 5 phút? │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │           Có                │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │        STANDARD             │
                    │  (bắt buộc cho long-running)│
                    └─────────────────────────────┘

                    ┌─────────────────────────────┐
                    │ Chạy < 5 phút. Cần          │
                    │ exactly-once execution?     │
                    └──────────────┬──────────────┘
                         │                   │
                        Có                 Không
                         │                   │
                    ┌────▼────┐         ┌────▼─────┐
                    │STANDARD │         │ EXPRESS  │
                    └─────────┘         └──────────┘
```

### Bảng Quyết Định Nhanh

| Câu Hỏi | Trả Lời → Chọn |
|---------|---------------|
| Cần lịch sử audit đầy đủ? | Có → Standard |
| Cần exactly-once semantics? | Có → Standard |
| Saga Pattern với compensating transactions? | Có → Standard |
| Cần waitForTaskToken (chờ callback)? | Có → Standard |
| Xử lý > 2,000 events/giây? | Có → Express |
| Workflow < 5 phút và high-volume? | Có → Express |
| Streaming data, IoT events? | Có → Express |
| Muốn sync API call với complex logic? | Có → Express Sync |
| Tích hợp trực tiếp với API Gateway response? | Có → Express Sync |

---

## 🔄 At-Least-Once vs Exactly-Once

### Standard — Exactly-Once (Đúng Một Lần)

```
Execution bắt đầu
    │
    ▼
State A thực thi ────────→ Ghi nhận vào execution history
    │                              │
    ▼                              ▼
State B thực thi ────────→ Ghi nhận vào execution history
    │
    ▼
Kết thúc

Nếu State B thất bại → Retry từ State B (không chạy lại State A)
Nếu system crash → Tiếp tục đúng từ state cuối ghi nhận
```

### Express — At-Least-Once (Ít Nhất Một Lần)

```
Execution bắt đầu
    │
    ▼
State A thực thi
    │
    ▼ ← System crash hoặc timeout ở đây
    ?

→ Step Functions có thể retry TOÀN BỘ execution từ đầu
→ State A có thể chạy nhiều hơn một lần
→ Consumer PHẢI implement idempotency (tính bất biến)
```

**Hàm ý khi dùng Express:** Mọi Task trong Express Workflow phải có thiết kế idempotent — gọi hai lần phải cho kết quả giống gọi một lần.

---

## 📝 Cấu Hình Logging Cho Express Workflow

Vì Express Workflow không lưu lịch sử trong console, phải cấu hình CloudWatch Logs:

```json
{
  "Type": "AWS::StepFunctions::StateMachine",
  "Properties": {
    "StateMachineType": "EXPRESS",
    "LoggingConfiguration": {
      "Level": "ALL",
      "IncludeExecutionData": true,
      "Destinations": [
        {
          "CloudWatchLogsLogGroup": {
            "LogGroupArn": "arn:aws:logs:ap-southeast-1:123:log-group:/aws/states/my-express-workflow:*"
          }
        }
      ]
    }
  }
}
```

### Các Cấp Độ Log

| Level | Ghi Log Khi |
|-------|-----------|
| `OFF` | Không ghi gì |
| `ERROR` | Chỉ khi execution thất bại |
| `FATAL` | Chỉ khi execution kết thúc với Fail state |
| `ALL` | Mọi transition (tốn CloudWatch Logs cost) |

---

## 🏗️ Ví Dụ Kiến Trúc

### Standard: Hệ Thống Xử Lý Đơn Hàng

```
API Gateway ──POST /orders──▶ Lambda ──StartExecution──▶ Standard Workflow
                                                                │
                                         ┌──────────────────────┤
                                         │                      │
                                    ┌────▼────┐           ┌─────▼─────┐
                                    │ Thanh   │           │  Đặt Hàng │
                                    │  Toán   │ ──────▶   │   Kho     │
                                    └────┬────┘           └─────┬─────┘
                                         │                      │
                                    ┌────▼────────────────────────▼────┐
                                    │          Gửi Thông Báo           │
                                    └──────────────────────────────────┘

Mỗi bước ghi nhận vào execution history → Dễ debug, audit đầy đủ
```

### Express Sync: API Với Logic Phức Tạp

```
Client ──POST /api──▶ API Gateway ──▶ Express Sync Workflow
                                              │
                              ┌───────────────┤
                              │               │
                         ┌────▼────┐    ┌─────▼─────┐
                         │ Validate│    │ Enrich    │
                         │  Input  │    │  Data     │
                         └────┬────┘    └─────┬─────┘
                              └───────┬───────┘
                                      │
                              ┌───────▼───────┐
                              │  Return Result │
                              └───────────────┘
                                      │
Client ◀──────response────── API Gateway ◀──result── Workflow kết thúc

Toàn bộ trong 1 request/response cycle
```

---

## 💡 Anti-Patterns (Cách Dùng Sai)

### Dùng Standard cho high-volume event processing

```
❌ Sai: Trigger Standard Workflow cho mỗi Kinesis record
→ Tốn kém + rate limit

✅ Đúng: Dùng Express Workflow hoặc Lambda trực tiếp
```

### Dùng Express cho Saga Pattern

```
❌ Sai: Dùng Express Workflow để orchestrate distributed transactions
→ At-least-once có thể chạy compensating transaction nhiều lần
→ Dữ liệu không nhất quán

✅ Đúng: Dùng Standard Workflow cho Saga Pattern
```

### Dùng Standard thay Lambda cho đơn giản

```
❌ Sai: Tạo Standard Workflow chỉ có 1-2 bước đơn giản
→ Over-engineering, tốn tiền không cần thiết

✅ Đúng: Lambda trực tiếp hoặc Express Workflow đủ rồi
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Khi nào bạn chọn Express thay vì Standard Workflow?**

> Chọn Express khi: (1) workflow chạy dưới 5 phút, (2) volume cao (hàng nghìn executions/giây), (3) có thể đảm bảo idempotency ở service downstream, và (4) không cần lịch sử audit trong console. Ví dụ: xử lý từng record từ Kinesis stream, hay serve API call với logic orchestration phức tạp (Express Sync).

**Q: At-least-once trong Express có nghĩa là gì cho developer?**

> Bất kỳ Task nào trong Express Workflow đều có thể được gọi nhiều hơn một lần — đặc biệt khi có lỗi hoặc retry. Developer phải đảm bảo **idempotency**: gọi Lambda hai lần với cùng input phải cho kết quả giống gọi một lần. Ví dụ: dùng `PutItem` với điều kiện thay vì `UpdateItem`, hoặc check-then-insert với transaction key.

**Q: Làm thế nào để debug Express Workflow?**

> Express Workflow không có execution history trong console. Phải cấu hình CloudWatch Logs với `Level: ALL` và `IncludeExecutionData: true`. Sau đó dùng CloudWatch Logs Insights để query log. X-Ray tracing cũng hỗ trợ tốt cho Express Workflow.

---

## 🔗 Điều Hướng

- **Quay lại:** [1-state-types.md](./1-state-types.md)
- **Tiếp theo:** [3-error-handling.md](./3-error-handling.md) — Catch & Retry
- **Liên quan:** [5-saga-orchestration.md](./5-saga-orchestration.md) — Saga cần Standard Workflow

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
