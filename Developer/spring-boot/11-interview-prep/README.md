# 🎯 Chuẩn Bị Phỏng Vấn Spring Boot

> Module tổng hợp toàn bộ kiến thức cần thiết để tự tin bước vào phòng phỏng vấn Java/Spring Boot — từ câu hỏi kỹ thuật, thiết kế hệ thống, đến câu chuyện STAR và kế hoạch học 90 ngày.

---

## 📚 Nội Dung Module

| File | Mô Tả | Khi Nào Dùng |
| ---- | ------ | ------------ |
| [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) | Top 50 câu hỏi & câu trả lời mẫu | Ôn tập nhanh 1–2 ngày trước phỏng vấn |
| [1-common-questions.md](./1-common-questions.md) | Câu hỏi thường gặp phân loại theo chủ đề | Học sâu từng chủ đề cụ thể |
| [2-system-design-scenarios.md](./2-system-design-scenarios.md) | Bài toán thiết kế hệ thống có hướng dẫn | Luyện System Design Interview |
| [3-star-stories.md](./3-star-stories.md) | Template & ví dụ câu chuyện STAR | Chuẩn bị câu hỏi hành vi |
| [4-behavioral-questions.md](./4-behavioral-questions.md) | Câu hỏi văn hóa, teamwork, leadership | Phỏng vấn vòng cuối / culture fit |
| [5-90-day-study-plan.md](./5-90-day-study-plan.md) | Kế hoạch học chi tiết 90 ngày | Lên kế hoạch học từ đầu |

---

## 🗺️ Lộ Trình Chuẩn Bị Phỏng Vấn

### 3 Tháng Trước Phỏng Vấn

```
Tuần 1–4:   Nắm vững nền tảng (Spring Core, JPA, Security)
Tuần 5–8:   Thực hành coding + Integration Testing
Tuần 9–12:  System Design + Microservices + Performance
```

→ Xem kế hoạch chi tiết: [5-90-day-study-plan.md](./5-90-day-study-plan.md)

### 1 Tháng Trước Phỏng Vấn

1. Hoàn thành ôn tập tất cả chủ đề core (01–06)
2. Luyện tập trả lời câu hỏi trong [1-common-questions.md](./1-common-questions.md)
3. Luyện 2–3 bài System Design theo [2-system-design-scenarios.md](./2-system-design-scenarios.md)
4. Viết ra 3–5 câu chuyện STAR từ kinh nghiệm thực tế

### 1 Tuần Trước Phỏng Vấn

- Đọc [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) — ôn top 50 câu hỏi
- Mock interview với đồng nghiệp hoặc ghi hình
- Chuẩn bị câu hỏi để hỏi lại nhà tuyển dụng

### Đêm Trước Phỏng Vấn

- Đọc lại INTERVIEW_GUIDE.md một lần
- Xem lại câu chuyện STAR trong [3-star-stories.md](./3-star-stories.md)
- Nghỉ ngơi đủ giấc — đừng học thêm gì mới

---

## 🏆 Cấu Trúc Phỏng Vấn Điển Hình

### Vòng 1: Technical Screening (30–45 phút qua điện thoại / video)

```
- Câu hỏi nền tảng: Spring Core, JPA, Security
- 1–2 câu hỏi về kinh nghiệm dự án thực tế
- Đôi khi có câu code ngắn (FizzBuzz, String manipulation)
```

### Vòng 2: Technical Interview sâu (60–90 phút)

```
- Deep dive vào Spring Boot ecosystem
- Live coding: viết REST API / Unit Test trong 30–45 phút
- Thảo luận về kiến trúc và design decisions
- Câu hỏi về debugging và troubleshooting
```

### Vòng 3: System Design (45–60 phút)

```
- Thiết kế hệ thống từ đầu (ví dụ: URL shortener, e-commerce)
- Thảo luận về scalability, availability, trade-offs
- Vẽ sơ đồ kiến trúc và giải thích từng component
```

### Vòng 4: Behavioral / Culture Fit (30–45 phút)

```
- Câu hỏi STAR về tình huống thực tế
- Đánh giá sự phù hợp với văn hóa công ty
- Câu hỏi về career goals và motivation
```

---

## 📊 Phân Bổ Câu Hỏi Theo Chủ Đề

Dựa trên kinh nghiệm thực tế từ các phỏng vấn Spring Boot:

| Chủ Đề | Tần Suất Xuất Hiện | Mức Độ Quan Trọng |
| ------ | ------------------ | ----------------- |
| Spring Core & DI (Dependency Injection — Tiêm Phụ Thuộc) | 95% | ⭐⭐⭐ Bắt buộc |
| Spring Data JPA & Transaction | 90% | ⭐⭐⭐ Bắt buộc |
| Spring Security & JWT | 85% | ⭐⭐⭐ Bắt buộc |
| Testing (Unit + Integration) | 80% | ⭐⭐⭐ Bắt buộc |
| N+1 Problem | 75% | ⭐⭐⭐ Rất quan trọng |
| Caching (Bộ Nhớ Đệm) & Redis | 70% | ⭐⭐ Quan trọng |
| Microservices & Architecture | 65% | ⭐⭐ Quan trọng |
| Docker & Kubernetes | 60% | ⭐⭐ Quan trọng |
| Async & Messaging (Kafka/RabbitMQ) | 55% | ⭐⭐ Nên biết |
| Performance Tuning (JVM, HikariCP) | 50% | ⭐⭐ Nên biết |
| Reactive Programming (WebFlux) | 35% | ⭐ Bonus |
| gRPC / GraphQL | 20% | ⭐ Bonus |

---

## 💡 Nguyên Tắc Trả Lời Câu Hỏi Kỹ Thuật

### Cấu Trúc WHAT–WHY–HOW

Áp dụng cho mọi câu hỏi kỹ thuật:

```
WHAT: Định nghĩa / giải thích khái niệm
WHY:  Tại sao cần / vấn đề nó giải quyết
HOW:  Cách implement / code ví dụ cụ thể
WHEN: Khi nào dùng / khi nào không nên dùng
```

**Ví dụ áp dụng cho câu hỏi "N+1 Problem là gì?":**

```
WHAT: N+1 là khi load 1 collection thực thi N queries bổ sung để tải liên kết
WHY:  Gây performance degradation nghiêm trọng khi dữ liệu lớn
HOW:  Dùng JOIN FETCH, @EntityGraph, hoặc Batch Size
WHEN: Xảy ra với LAZY loading; giải quyết bằng eager fetching có chọn lọc
```

### Các Lỗi Thường Gặp Khi Trả Lời

- ❌ **Trả lời quá ngắn**: "N+1 là khi có nhiều queries" — thiếu chiều sâu
- ❌ **Trả lời quá dài**: Mất 10 phút cho 1 câu — không biết ưu tiên
- ❌ **Không có ví dụ code**: Nói lý thuyết mà không demo được
- ❌ **Không biết khi nào KHÔNG dùng**: Thiếu kinh nghiệm thực tế
- ✅ **Lý tưởng**: 2–3 phút, có code ví dụ, đề cập trade-offs

---

## 🎤 Checklist Trước Phỏng Vấn

### Kỹ Thuật

- [ ] Giải thích được IoC (Inversion of Control — Đảo Ngược Quyền Kiểm Soát) & DI mà không cần nhìn tài liệu
- [ ] Demo được Spring Security + JWT end-to-end
- [ ] Giải quyết được N+1 Problem với 3 cách khác nhau
- [ ] Viết được Unit Test & Integration Test cho REST API
- [ ] Mô tả Transaction Propagation (Lan Truyền Giao Dịch) với ví dụ
- [ ] Thiết kế được system architecture đơn giản trên whiteboard
- [ ] Giải thích Circuit Breaker (Cầu Dao Mạch) pattern với Resilience4j

### Behavioral (Hành Vi)

- [ ] Chuẩn bị 3 câu chuyện STAR về thách thức kỹ thuật
- [ ] Chuẩn bị câu về conflict resolution (giải quyết mâu thuẫn)
- [ ] Chuẩn bị câu về leadership / mentoring
- [ ] Câu hỏi về failure / lesson learned

### Logistics (Hậu Cần)

- [ ] Nghiên cứu về công ty và tech stack họ dùng
- [ ] Chuẩn bị 5–7 câu hỏi để hỏi lại interviewer
- [ ] Test camera, micro, internet (nếu phỏng vấn online)
- [ ] Chuẩn bị môi trường code (IDE, terminal sẵn sàng)

---

## ❓ Câu Hỏi Để Hỏi Lại Interviewer

Hỏi lại thể hiện sự chủ động và quan tâm thực sự:

### Về Kỹ Thuật

- "Tech stack hiện tại của team là gì? Có kế hoạch migrate công nghệ nào không?"
- "Bạn xử lý Deployment (Triển Khai) và CI/CD (Tích Hợp & Phân Phối Liên Tục) như thế nào?"
- "Code review process của team như thế nào?"
- "Kiến trúc hiện tại là monolith hay microservices? Tại sao?"

### Về Team & Culture

- "Team size là bao nhiêu? Cấu trúc team như thế nào?"
- "Một ngày làm việc điển hình của vị trí này trông như thế nào?"
- "Kỳ vọng 90 ngày đầu với nhân viên mới là gì?"
- "Cơ hội học hỏi và phát triển kỹ năng như thế nào?"

### Về Dự Án

- "Thách thức kỹ thuật lớn nhất hiện tại của team là gì?"
- "Sản phẩm hiện phục vụ bao nhiêu người dùng? Scale như thế nào?"

---

## 🔗 Điều Hướng Nhanh

| Khi Cần | Đọc File |
| ------- | -------- |
| Ôn nhanh top câu hỏi | [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) |
| Học sâu 1 chủ đề cụ thể | [1-common-questions.md](./1-common-questions.md) |
| Luyện System Design | [2-system-design-scenarios.md](./2-system-design-scenarios.md) |
| Chuẩn bị câu chuyện kinh nghiệm | [3-star-stories.md](./3-star-stories.md) |
| Phỏng vấn văn hóa / HR | [4-behavioral-questions.md](./4-behavioral-questions.md) |
| Lên kế hoạch học từ đầu | [5-90-day-study-plan.md](./5-90-day-study-plan.md) |

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
