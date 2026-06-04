# Chuẩn Bị Phỏng Vấn AWS Application Integration

> Bộ tài liệu tổng hợp giúp bạn tự tin trả lời mọi câu hỏi về dịch vụ tích hợp ứng dụng AWS trong phỏng vấn kỹ thuật.

## 📚 Nội Dung Thư Mục Này

| File | Nội Dung | Ưu Tiên |
|---|---|---|
| [1-INTERVIEW_GUIDE.md](./1-INTERVIEW_GUIDE.md) | Top 25 câu hỏi + gợi ý trả lời chi tiết | ⭐⭐⭐ Bắt buộc |
| [2-service-comparison.md](./2-service-comparison.md) | Bảng so sánh đầy đủ SQS / SNS / EventBridge / Kinesis / MQ | ⭐⭐⭐ Bắt buộc |
| [3-system-design-scenarios.md](./3-system-design-scenarios.md) | 5 bài toán thiết kế hệ thống có lời giải | ⭐⭐⭐ Quan trọng |
| [4-star-stories.md](./4-star-stories.md) | Mẫu câu chuyện tình huống theo phương pháp STAR | ⭐⭐ Nên có |
| [5-90-day-study-plan.md](./5-90-day-study-plan.md) | Kế hoạch học 90 ngày có lộ trình rõ ràng | ⭐⭐ Tham khảo |

---

## 🎯 Chiến Lược Phỏng Vấn

### Hiểu Người Phỏng Vấn Muốn Gì

Khi hỏi về AWS Integration, người phỏng vấn thường kiểm tra ba điều:

1. **Kiến Thức Dịch Vụ** — Bạn biết SQS, SNS, EventBridge, Kinesis làm gì không?
2. **Tư Duy Kiến Trúc** — Bạn có thể chọn đúng công cụ cho đúng bài toán không?
3. **Kinh Nghiệm Thực Tế** — Bạn đã gặp vấn đề gì trong production và giải quyết thế nào?

### Khung Trả Lời Hiệu Quả

Với câu hỏi kỹ thuật, dùng cấu trúc **PREP**:

```
P — Point (Điểm chính): Nêu câu trả lời trực tiếp trong 1–2 câu
R — Reason (Lý do): Giải thích tại sao, cơ chế hoạt động
E — Example (Ví dụ): Đưa ra ví dụ cụ thể hoặc use case thực tế
P — Pitfall (Cạm bẫy): Đề cập điểm cần lưu ý, trade-off
```

**Ví dụ áp dụng PREP cho câu hỏi "SQS vs SNS khi nào dùng cái nào?":**

```
P: SQS dùng khi cần task queue (hàng đợi tác vụ) — một producer, một consumer.
   SNS dùng khi cần fan-out (khuếch tán) — một publisher, nhiều subscriber.

R: SQS là pull-based (kéo) — consumer chủ động lấy message.
   SNS là push-based (đẩy) — tự đẩy đến subscriber ngay khi có message.

E: E-commerce: đặt hàng → SQS → Lambda xử lý từng đơn.
   E-commerce: đặt hàng thành công → SNS → gửi email + SMS + cập nhật inventory cùng lúc.

P: SNS không lưu message nếu subscriber fail. Kết hợp SNS + SQS (Fan-out Pattern)
   để vừa broadcast vừa đảm bảo message không mất.
```

---

## 📋 Checklist Trước Ngày Phỏng Vấn

### Ngày T-7 (1 Tuần Trước)

- [ ] Đọc toàn bộ `1-INTERVIEW_GUIDE.md` — nắm 25 câu hỏi
- [ ] Học thuộc bảng so sánh trong `2-service-comparison.md`
- [ ] Làm quen với 3 bài toán thiết kế trong `3-system-design-scenarios.md`

### Ngày T-3 (3 Ngày Trước)

- [ ] Chuẩn bị 2–3 câu chuyện STAR cá nhân từ `4-star-stories.md`
- [ ] Ôn lại các trade-off quan trọng: Standard vs FIFO, SQS vs SNS vs EventBridge
- [ ] Thực hành nói to câu trả lời — nghe lại và chỉnh sửa

### Ngày T-1 (Ngày Hôm Trước)

- [ ] Xem lại nhanh bảng so sánh dịch vụ
- [ ] Ôn 5 câu hỏi bạn cảm thấy chưa chắc nhất
- [ ] Ngủ đủ giấc — không thức khuya ôn

### Ngày Phỏng Vấn

- [ ] Đọc lại PREP framework trong 5 phút
- [ ] Nhớ: Vẽ sơ đồ khi được hỏi thiết kế hệ thống
- [ ] Nhớ: Hỏi lại yêu cầu trước khi thiết kế ("Có cần đảm bảo ordering không?")

---

## 🔥 Những Điểm Dễ Bị Hỏi Nhất

Dựa trên phân tích câu hỏi phỏng vấn thực tế:

### Tier 1 — Gần Như Chắc Chắn Bị Hỏi

| Chủ Đề | Câu Hỏi Điển Hình |
|---|---|
| SQS vs SNS | "Khi nào dùng SQS, khi nào dùng SNS?" |
| DLQ | "Dead Letter Queue là gì, cấu hình thế nào?" |
| Idempotency | "Tại sao SQS at-least-once delivery gây vấn đề gì?" |
| Fan-out | "Thiết kế hệ thống thông báo cho 1 triệu user khi có sự kiện" |
| FIFO vs Standard | "Khi nào cần FIFO Queue?" |

### Tier 2 — Thường Gặp Trong Vòng 2 Trở Lên

| Chủ Đề | Câu Hỏi Điển Hình |
|---|---|
| EventBridge | "Tại sao dùng EventBridge thay vì SNS?" |
| Step Functions | "Step Functions vs Lambda chaining — chọn cái nào?" |
| Kinesis | "Kinesis Shard là gì, tính thế nào?" |
| Saga Pattern | "Xử lý distributed transaction (giao dịch phân tán) thế nào?" |
| Visibility Timeout | "Visibility Timeout cấu hình sai gây ra vấn đề gì?" |

### Tier 3 — Vòng Senior / Principal

| Chủ Đề | Câu Hỏi Điển Hình |
|---|---|
| Event Sourcing | "Mô tả kiến trúc Event Sourcing + CQRS" |
| Outbox Pattern | "Đảm bảo tính nhất quán giữa DB và message queue thế nào?" |
| Cost Optimization | "Tối ưu chi phí cho 1 tỷ message/ngày" |
| Cross-account | "Thiết kế messaging architecture cho multi-account AWS" |

---

## 💡 Mẹo Ghi Điểm Cao

### Chủ Động Nêu Trade-off

Người phỏng vấn đánh giá cao ứng viên tự nêu trade-off (đánh đổi) mà không cần được hỏi:

> "Tôi sẽ dùng SQS FIFO Queue vì cần đảm bảo thứ tự. Nhưng cần lưu ý FIFO có giới hạn 300 TPS (transactions per second — giao dịch mỗi giây) thay vì unlimited như Standard Queue, nên nếu throughput cao hơn thì cần tính toán lại."

### Kết Nối Với Kinh Nghiệm Thực Tế

Nếu đã dùng AWS trong dự án, hãy đề cập:

> "Trong dự án X, chúng tôi gặp vấn đề consumer bị crash giữa chừng khiến message bị mất. Tôi đã giải quyết bằng cách cấu hình Visibility Timeout phù hợp và thêm DLQ để không mất message."

### Hỏi Clarifying Questions (Câu Hỏi Làm Rõ)

Trước khi thiết kế hệ thống, luôn hỏi:

- "Message có cần đảm bảo thứ tự (ordering) không?"
- "Hệ thống cần exactly-once hay at-least-once delivery?"
- "Throughput (thông lượng) dự kiến là bao nhiêu message/giây?"
- "Consumer có thể xử lý duplicate (trùng lặp) không?"

---

## 🗺️ Lộ Trình Sử Dụng Tài Liệu Này

```
Bước 1: Đọc 2-service-comparison.md (30 phút)
        → Nắm bảng so sánh, học thuộc khi nào dùng gì

Bước 2: Đọc 1-INTERVIEW_GUIDE.md (2 giờ)
        → Đọc câu hỏi, che lời giải, tự trả lời, rồi so sánh

Bước 3: Thực hành 3-system-design-scenarios.md (3 giờ)
        → Mỗi bài: tự vẽ sơ đồ trước, rồi so sánh với lời giải

Bước 4: Chuẩn bị câu chuyện STAR từ 4-star-stories.md (1 giờ)
        → Điều chỉnh template phù hợp với kinh nghiệm cá nhân

Bước 5: Lên kế hoạch dài hạn từ 5-90-day-study-plan.md
        → Theo dõi tiến độ hàng tuần
```

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
