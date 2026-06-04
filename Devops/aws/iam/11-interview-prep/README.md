# 🎯 Chuẩn Bị Phỏng Vấn AWS Security

> Module tổng hợp giúp bạn tự tin bước vào mọi vòng phỏng vấn AWS Security — từ câu hỏi kỹ thuật, debug IAM, thiết kế hệ thống đến kể chuyện sự cố theo phương pháp STAR.

---

## 📚 Nội Dung Module Này

| File | Nội Dung | Thời Gian Ôn |
|---|---|---|
| [1-top-questions.md](1-top-questions.md) | Top 25 câu hỏi AWS Security + đáp án chi tiết | 3–4 giờ |
| [2-iam-troubleshooting.md](2-iam-troubleshooting.md) | Debug Access Denied, xung đột policy | 2–3 giờ |
| [3-system-design-security.md](3-system-design-security.md) | Bài toán thiết kế kiến trúc bảo mật | 3–4 giờ |
| [4-star-stories.md](4-star-stories.md) | Mẫu câu chuyện sự cố theo phương pháp STAR | 2–3 giờ |
| [5-90-day-study-plan.md](5-90-day-study-plan.md) | Kế hoạch học tập 90 ngày có cấu trúc | 30 phút |

**Tổng thời gian ôn tập: 10–15 giờ**

---

## 🗺️ Lộ Trình Sử Dụng Module

### Nếu Bạn Còn 1 Tuần Trước Phỏng Vấn

```
Ngày 1–2: Đọc 1-top-questions.md — nắm 25 câu hỏi phổ biến
Ngày 3:   Luyện 2-iam-troubleshooting.md — debug IAM hands-on
Ngày 4:   Ôn 3-system-design-security.md — luyện thiết kế hệ thống
Ngày 5:   Viết và luyện tập 4-star-stories.md
Ngày 6–7: Mock interview tổng hợp + review điểm yếu
```

### Nếu Bạn Còn 1 Tháng Trước Phỏng Vấn

```
Tuần 1: Ôn lại toàn bộ 01-iam-fundamentals/ + 04-encryption-kms/
Tuần 2: Đọc 1-top-questions.md, bắt đầu 2-iam-troubleshooting.md
Tuần 3: Luyện 3-system-design-security.md, viết STAR stories
Tuần 4: Mock interviews, refinement, review tất cả điểm yếu
```

### Nếu Bạn Có 90 Ngày

Xem [5-90-day-study-plan.md](5-90-day-study-plan.md) để có lộ trình đầy đủ từ nền tảng đến phỏng vấn thực chiến.

---

## 🎯 Các Vòng Phỏng Vấn AWS Security Thường Gặp

### Vòng 1 — Phone Screen / HR Screen (30–45 phút)

**Trọng tâm:** Kiến thức nền tảng + kinh nghiệm tổng quan

Câu hỏi phổ biến:
- "Kể về dự án bảo mật AWS gần nhất bạn làm"
- "Bạn hiểu gì về Shared Responsibility Model?"
- "Bạn từng xử lý security incident nào chưa?"

**Tip:** Chuẩn bị 2–3 câu chuyện ngắn có số liệu cụ thể.

### Vòng 2 — Technical Interview (60–90 phút)

**Trọng tâm:** IAM, encryption, troubleshooting

Câu hỏi phổ biến:
- Viết IAM policy trực tiếp
- Debug scenario "Access Denied"
- Giải thích envelope encryption
- Cross-account role hoạt động như thế nào

**Tip:** Đọc kỹ [2-iam-troubleshooting.md](2-iam-troubleshooting.md) và luyện viết policy JSON không cần nhìn tài liệu.

### Vòng 3 — System Design (60–90 phút)

**Trọng tâm:** Thiết kế kiến trúc bảo mật cho hệ thống thực tế

Bài toán phổ biến:
- "Thiết kế kiến trúc multi-account an toàn cho 500 kỹ sư"
- "Bảo mật ứng dụng fintech xử lý PCI-DSS"
- "Xây dựng security monitoring cho 50 AWS accounts"

**Tip:** Đọc kỹ [3-system-design-security.md](3-system-design-security.md), luyện vẽ sơ đồ kiến trúc.

### Vòng 4 — Behavioral Interview (45–60 phút)

**Trọng tâm:** Cách bạn xử lý tình huống khó, làm việc nhóm, leadership

Câu hỏi phổ biến:
- "Kể về lần bạn phát hiện và xử lý security incident"
- "Làm thế nào bạn thuyết phục team thay đổi thực hành bảo mật?"
- "Khi nào bạn phải chọn giữa security và business deadline?"

**Tip:** Luyện [4-star-stories.md](4-star-stories.md) cho đến khi kể trơn tru dưới 3 phút mỗi câu chuyện.

---

## 📋 Checklist Sẵn Sàng Phỏng Vấn

### Kiến Thức Kỹ Thuật

- [ ] Viết được IAM policy với conditions từ đầu (không tài liệu)
- [ ] Giải thích được IAM evaluation logic đầy đủ (explicit deny → SCP → permission boundary → identity policy → resource policy)
- [ ] Debug được "Access Denied" một cách có hệ thống
- [ ] Giải thích được envelope encryption với DEK (Data Encryption Key — Khóa Mã Hóa Dữ Liệu) và CMK (Customer Master Key — Khóa Chính Do Khách Hàng Quản Lý)
- [ ] Thiết kế được multi-account structure với Organizations + SCPs + Control Tower
- [ ] Biết khi nào dùng Secrets Manager vs Parameter Store vs environment variables
- [ ] Giải thích được GuardDuty finding types và cách xử lý từng loại
- [ ] Thiết kế được SIEM (Security Information and Event Management — Hệ Thống Quản Lý Thông Tin và Sự Kiện Bảo Mật) đơn giản trên AWS

### Kỹ Năng Trình Bày

- [ ] Chuẩn bị ≥ 3 câu chuyện STAR về bảo mật
- [ ] Có thể giải thích kiến trúc bảo mật bằng cả sơ đồ và ngôn ngữ tự nhiên
- [ ] Biết đặt câu hỏi thông minh cuối phỏng vấn
- [ ] Đã luyện mock interview ít nhất 2 lần

### Chuẩn Bị Thực Hành

- [ ] Có AWS lab account với CloudTrail, GuardDuty đang chạy
- [ ] Đã tự tay tạo cross-account role, viết SCP, cấu hình KMS
- [ ] Đã xem xét ít nhất 1 breach case study liên quan đến AWS (Capital One, Tesla)
- [ ] Theo dõi AWS Security Blog ít nhất 1 tháng gần đây

---

## 💡 Nguyên Tắc Phỏng Vấn AWS Security

### 1. Nói Về Trade-offs, Không Chỉ Giải Pháp

❌ Sai: "Dùng Secrets Manager để lưu credentials."

✅ Đúng: "Tôi sẽ dùng Secrets Manager thay vì Parameter Store vì cần auto-rotation cho database credentials. Chi phí cao hơn (~$0.40/secret/tháng vs miễn phí với Standard Parameter Store), nhưng với môi trường production xử lý dữ liệu khách hàng, khả năng rotate không cần downtime quan trọng hơn tiết kiệm chi phí."

### 2. Bắt Đầu Bằng Yêu Cầu, Không Phải Dịch Vụ

❌ Sai: "Tôi sẽ dùng GuardDuty và Security Hub."

✅ Đúng: "Với yêu cầu phát hiện mối đe dọa realtime và tổng hợp alerts đa tài khoản, tôi cần threat detection layer (GuardDuty) và aggregation layer (Security Hub). Cụ thể..."

### 3. Luôn Nhắc Đến Least Privilege và Defense-in-Depth

Mọi câu trả lời thiết kế bảo mật đều nên có:
- **Least Privilege** (Nguyên Tắc Đặc Quyền Tối Thiểu): chỉ cấp quyền tối thiểu cần thiết
- **Defense-in-Depth** (Bảo Vệ Theo Chiều Sâu): nhiều lớp bảo vệ, không phụ thuộc một điểm duy nhất

### 4. Thừa Nhận Khi Không Biết

Phỏng vấn viên đánh giá cao: "Tôi chưa dùng dịch vụ đó trong thực tế, nhưng theo kiến trúc tổng thể, tôi hiểu nó giải quyết vấn đề X bằng cách Y. Tôi sẽ xác nhận chi tiết trước khi triển khai."

---

## 🔗 Điều Hướng

| Chủ Đề | Link |
|---|---|
| Câu hỏi phỏng vấn hàng đầu | [1-top-questions.md](1-top-questions.md) |
| Debug IAM | [2-iam-troubleshooting.md](2-iam-troubleshooting.md) |
| System design bảo mật | [3-system-design-security.md](3-system-design-security.md) |
| Câu chuyện STAR | [4-star-stories.md](4-star-stories.md) |
| Kế hoạch học 90 ngày | [5-90-day-study-plan.md](5-90-day-study-plan.md) |
| Quay lại INDEX | [../INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-16
