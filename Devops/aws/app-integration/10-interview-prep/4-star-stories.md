# Câu Chuyện Phỏng Vấn Theo Phương Pháp STAR

> Template và ví dụ mẫu — điều chỉnh theo kinh nghiệm cá nhân của bạn

---

## Phương Pháp STAR là gì?

**STAR** là khung trả lời câu hỏi phỏng vấn hành vi (behavioral interview):

| Chữ Cái | Ý Nghĩa | Nội Dung |
|---|---|---|
| **S** | Situation (Tình Huống) | Bối cảnh, dự án, thời điểm |
| **T** | Task (Nhiệm Vụ) | Vấn đề bạn cần giải quyết, trách nhiệm của bạn |
| **A** | Action (Hành Động) | Các bước bạn đã thực hiện |
| **R** | Result (Kết Quả) | Kết quả đạt được, bài học rút ra |

**Thời gian trả lời lý tưởng:** 2–3 phút

---

## Câu Hỏi Hành Vi Thường Gặp Trong AWS Integration

Người phỏng vấn thường hỏi dạng:
- "Kể về lần bạn thiết kế một hệ thống messaging/event-driven..."
- "Kể về lần bạn xử lý sự cố trong production liên quan đến queue/messaging..."
- "Kể về lần bạn phải chọn giữa các giải pháp kiến trúc khác nhau..."
- "Kể về lần bạn cải thiện performance/reliability của một hệ thống..."

---

## Template STAR (Điền Vào Theo Kinh Nghiệm Của Bạn)

### Template 1: Thiết Kế Hệ Thống Messaging

```
S (Tình Huống):
"Tại [tên công ty/dự án], chúng tôi có hệ thống [mô tả ngắn]
đang gặp vấn đề [vấn đề cụ thể] vào khoảng [thời điểm]."

T (Nhiệm Vụ):
"Tôi được giao nhiệm vụ [thiết kế / cải thiện / sửa] hệ thống
với yêu cầu [yêu cầu cụ thể: throughput, latency, reliability...].
Thách thức lớn nhất là [khó khăn chính]."

A (Hành Động):
"Tôi đã thực hiện các bước sau:
1. Phân tích [vấn đề cụ thể] bằng cách [cách tiếp cận]
2. Đánh giá các lựa chọn: [option A] vs [option B] — chọn [option] vì [lý do]
3. Thiết kế kiến trúc với [dịch vụ AWS] vì [lý do kỹ thuật]
4. Implement, test, và deploy [mô tả ngắn]
5. Thiết lập monitoring với [metrics cụ thể]"

R (Kết Quả):
"Sau khi triển khai:
- [Metric] cải thiện từ X lên Y (ví dụ: latency giảm từ 5s xuống 200ms)
- [Business impact] (ví dụ: giảm 30% chi phí vận hành)
- Bài học: [điều bạn học được về trade-off, kỹ thuật, hoặc quy trình]"
```

---

## Câu Chuyện Mẫu 1: Xử Lý Sự Cố DLQ Trong Production

*(Đây là ví dụ mẫu — điều chỉnh theo kinh nghiệm thực tế của bạn)*

**Câu hỏi:** "Kể về lần bạn xử lý một sự cố nghiêm trọng trong production."

---

**S — Tình Huống:**

> "Ở công ty trước, chúng tôi có hệ thống xử lý đơn hàng với SQS Standard Queue và Lambda consumer. Vào một buổi sáng, tôi nhận được alert rằng DLQ (Dead Letter Queue — Hàng Đợi Thư Chết) đột ngột có hơn 5,000 message trong 30 phút. Đây là hệ thống xử lý thanh toán, nên mỗi message trong DLQ là một giao dịch chưa được xử lý."

**T — Nhiệm Vụ:**

> "Tôi cần xác định nguyên nhân nhanh nhất có thể và khôi phục hệ thống mà không mất bất kỳ giao dịch nào. Áp lực lớn vì ảnh hưởng trực tiếp đến doanh thu."

**A — Hành Động:**

> "Bước đầu tiên tôi làm là KHÔNG xóa message trong DLQ — đây là bản năng sai của nhiều người. Thay vào đó:
>
> 1. **Investigate trước khi hành động**: Lấy một message từ DLQ, xem payload và lỗi. Phát hiện ra Lambda đang throw `JsonParseException` — có nghĩa là message format bị sai.
>
> 2. **Trace nguyên nhân gốc rễ**: Dùng AWS X-Ray, tôi thấy lỗi bắt đầu từ chính xác 08:47 sáng — đúng lúc team platform deploy một thay đổi vào producer service. Schema của message đã thay đổi nhưng consumer chưa được update.
>
> 3. **Hotfix consumer**: Tôi update Lambda consumer để xử lý cả format cũ và mới (backward-compatible). Deploy trong 10 phút.
>
> 4. **Redrive DLQ**: Sau khi consumer đã handle đúng, tôi dùng DLQ Redrive để đẩy lại 5,000 message về queue gốc. Lambda tự động xử lý.
>
> 5. **Bổ sung safeguard**: Thêm contract test giữa producer và consumer để phát hiện schema mismatch (không khớp lược đồ) trước khi deploy."

**R — Kết Quả:**

> "Tổng thời gian khắc phục: 25 phút. Không mất giao dịch nào nhờ DLQ đã giữ lại toàn bộ message. Sau incident này, chúng tôi implement contract testing với Pact — đã ngăn được 3 incident tương tự trong 6 tháng tiếp theo. Bài học quan trọng: DLQ không phải là nơi vứt message thất bại — đó là safety net quý giá và phải được monitor 24/7."

---

## Câu Chuyện Mẫu 2: Thiết Kế Fan-Out Pattern

**Câu hỏi:** "Kể về lần bạn thiết kế kiến trúc event-driven cho hệ thống mới."

---

**S — Tình Huống:**

> "Chúng tôi đang xây dựng tính năng gửi thông báo khi người dùng hoàn thành một giao dịch. Ban đầu, code rất đơn giản: sau khi ghi database, gọi trực tiếp Email Service API và SMS Service API. Vấn đề xảy ra khi SMS Service API của third-party bị down — toàn bộ giao dịch bị fail theo dù email và SMS là side effects, không phải core transaction."

**T — Nhiệm Vụ:**

> "Tôi được giao refactor kiến trúc này để đảm bảo: (1) lỗi notification không ảnh hưởng giao dịch chính, (2) dễ thêm notification channel mới (ví dụ: push notification), (3) có thể retry notification nếu lỗi tạm thời."

**A — Hành Động:**

> "Tôi đề xuất và implement Fan-out Pattern với SNS + SQS:
>
> 1. **Tách biệt concern**: Transaction service chỉ cần publish event `TransactionCompleted` lên SNS Topic — không cần biết ai nhận.
>
> 2. **Fan-out qua SNS**: SNS Topic có 2 subscriber là 2 SQS Queue: `email-notification-queue` và `sms-notification-queue`.
>
> 3. **Lambda per queue**: Mỗi queue có Lambda consumer riêng, xử lý độc lập. SMS Lambda fail không ảnh hưởng Email Lambda.
>
> 4. **DLQ cho mỗi queue**: Cấu hình maxReceiveCount=3, message lỗi vào DLQ thay vì mất.
>
> 5. **Thuyết phục team về trade-off**: Có ý kiến lo ngại về complexity tăng. Tôi giải thích rằng complexity tăng ở infrastructure (thêm SNS, SQS) nhưng giảm ở code (mỗi service chỉ làm một việc) và tăng reliability đáng kể."

**R — Kết Quả:**

> "Sau khi deploy, lần tiếp theo SMS Service down: Transaction vẫn thành công, message vào DLQ, khi SMS Service recover, message được redrive và gửi thành công. Zero transaction lost, zero user complaint về giao dịch fail.
>
> Thêm nữa, 2 tháng sau khi team muốn thêm push notification: chỉ cần tạo SQS Queue mới và subscribe vào SNS Topic — Transaction Service không cần thay đổi một dòng code nào.
>
> Bài học: Decoupling qua message queue không chỉ giải quyết vấn đề hiện tại mà còn mở ra extensibility (khả năng mở rộng) không tưởng được trước đó."

---

## Câu Chuyện Mẫu 3: Cải Thiện Performance với Kinesis

**Câu hỏi:** "Kể về lần bạn giải quyết vấn đề performance nghiêm trọng."

---

**S — Tình Huống:**

> "Hệ thống analytics của chúng tôi thu thập log từ 50 server, khoảng 10,000 log/giây. Ban đầu, mỗi server gọi trực tiếp Lambda để xử lý mỗi log entry. Khi traffic tăng gấp 3 vào dịp cao điểm, Lambda throttle (bị giới hạn), log bị mất, và chi phí Lambda tăng 400%."

**T — Nhiệm Vụ:**

> "Redesign pipeline để: handle peak 30,000 log/giây, không mất log, giảm chi phí, và nếu có thể, thêm khả năng replay log lịch sử để debug."

**A — Hành Động:**

> "Phân tích constraint:
> - 30,000 records/giây với trung bình 500 bytes/record = 15 MB/giây write
> - Cần 2 consumer: real-time alert và lưu trữ S3
> - Cần replay → Kinesis Data Streams phù hợp hơn SQS
>
> Thiết kế mới:
> 1. **Kinesis Data Streams với 20 shard** (15MB/s ÷ 1MB/s per shard, thêm buffer)
> 2. **Producer**: Batch records trước khi ghi vào Kinesis — giảm số API call
> 3. **Consumer 1**: Lambda Enhanced Fan-Out xử lý real-time alert (Iterator Age < 1s)
> 4. **Consumer 2**: Kinesis Firehose → S3 (không cần viết consumer code)
>
> Thách thức: Hot shard (mảnh nóng) — một số server dùng server hostname làm partition key, tất cả log của server đó vào cùng một shard.
> Giải pháp: Đổi partition key thành `server_id + random_suffix` → phân tán đều."

**R — Kết Quả:**

> "Sau migration:
> - 0 log bị mất kể cả trong peak traffic gấp 5 lần bình thường
> - Chi phí giảm 60% (Kinesis shard-hour rẻ hơn 30,000 Lambda invocations/giây)
> - Lần đầu tiên có khả năng replay log 7 ngày — giúp debug một bug đã tồn tại 2 tháng mà không ai phát hiện
> - Bài học quan trọng: Partition key strategy ảnh hưởng lớn đến performance — phải test với production-like data trước khi deploy."

---

## Câu Chuyện Mẫu 4: Implement Saga Pattern

**Câu hỏi:** "Kể về lần bạn xử lý distributed transaction trong microservices."

---

**S — Tình Huống:**

> "Chúng tôi có 3 microservice: Order Service, Payment Service, và Inventory Service — mỗi service có database riêng. Vấn đề: khi user đặt hàng, đôi khi Inventory Service trừ kho thành công nhưng Payment Service fail — kết quả là hàng bị trừ nhưng tiền không được tính, gây thất thu."

**T — Nhiệm Vụ:**

> "Đảm bảo tính nhất quán (consistency): hoặc cả ba service đều thành công, hoặc tất cả được rollback về trạng thái ban đầu. Không được dùng distributed transaction 2-phase commit vì quá tight coupling."

**A — Hành Động:**

> "Tôi đề xuất Saga Pattern với Step Functions:
>
> 1. **Phân tích compensating transactions** (giao dịch bù trừ): Mỗi bước forward có một bước reverse tương ứng:
>    - ReserveInventory ↔ ReleaseInventory
>    - ChargePayment ↔ RefundPayment
>
> 2. **Build Step Functions State Machine**: Define workflow với Catch block cho mỗi state — nếu ChargePayment fail, tự động chạy ReleaseInventory.
>
> 3. **Idempotency trong mỗi service**: Mỗi Lambda nhận TaskToken từ Step Functions, kiểm tra `idempotency_key = order_id + step_name` trước khi xử lý. Tránh double-charge khi Step Functions retry.
>
> 4. **Testing**: Inject fault vào từng bước để verify compensation hoạt động đúng — quan trọng nhất là test case compensation fail (ví dụ: RefundPayment cũng fail)."

**R — Kết Quả:**

> "Sau deploy, zero inconsistency incidents trong 6 tháng. Step Functions Execution History giúp team product tự tra cứu tại sao một đơn hàng fail — giảm ticket support lên team engineering 40%.
>
> Một bài học khó: Compensation không phải lúc nào cũng hoàn toàn khả thi. Ví dụ nếu hàng đã được giao (Shipping) thì không thể 'ungship' — phải design business process cho trường hợp này (tạo return request thay vì undo shipment).
>
> Distributed transaction là vấn đề không có giải pháp hoàn hảo — Saga giảm thiểu inconsistency nhưng không loại bỏ hoàn toàn, và cần business đồng ý với eventual consistency."

---

## Checklist Chuẩn Bị Câu Chuyện Cá Nhân

### Trước Khi Viết Câu Chuyện

- [ ] Liệt kê 5–10 dự án/tình huống đáng nhớ trong career
- [ ] Với mỗi tình huống: có liên quan đến messaging / event-driven không?
- [ ] Chọn 3–4 câu chuyện tốt nhất — đa dạng: thiết kế, debug, cải thiện, đề xuất

### Khi Viết Câu Chuyện

- [ ] **Situation cụ thể** — tên công ty (nếu được), quy mô hệ thống, thời điểm
- [ ] **Số liệu** — throughput, latency, cost, team size (mọi số liệu đều tăng độ tin cậy)
- [ ] **Quyết định kỹ thuật** — giải thích tại sao chọn X thay vì Y
- [ ] **Khó khăn thực tế** — câu chuyện không có challenge nghe không thật
- [ ] **Bài học** — shows growth mindset (tư duy phát triển)

### Khi Trình Bày

- [ ] Thực hành nói to — không đọc, không thuộc lòng từng từ
- [ ] 2–3 phút — không dài hơn nếu không được hỏi thêm
- [ ] Kết thúc bằng impact (tác động), không kết thúc bằng action

---

## Lưu Ý Quan Trọng

**Điều chỉnh câu chuyện theo vai trò:**
- Junior Engineer: Tập trung vào implementation detail, debugging, learning
- Senior Engineer: Tập trung vào architectural decision, trade-off, mentoring
- Tech Lead / Staff: Tập trung vào cross-team impact, org-level decision, strategy

**Nếu chưa có kinh nghiệm trực tiếp với AWS:**
- Có thể kể về dự án dùng RabbitMQ / Kafka / Redis Pub-Sub — kiến thức tương đương
- Nêu rõ: "Tôi chưa dùng trực tiếp AWS SQS nhưng đã dùng RabbitMQ với pattern tương tự..."
- Thể hiện bạn hiểu concept — không nhất thiết phải dùng đúng dịch vụ AWS

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
