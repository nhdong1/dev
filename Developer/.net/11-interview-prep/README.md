# 11. Chuẩn Bị Phỏng Vấn — Interview Preparation

> Bộ tài liệu hoàn chỉnh để chuẩn bị phỏng vấn .NET Developer từ Junior đến Senior, bao gồm câu hỏi kỹ thuật, câu chuyện STAR, kịch bản system design và kế hoạch học 90 ngày.

---

## 📋 Nội Dung Chương Này

| File | Mô Tả | Đối Tượng |
| ---- | ------ | --------- |
| [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) | Top 30 câu hỏi phỏng vấn C# .NET với đáp án chi tiết | Mọi cấp độ |
| [1-system-design-scenarios.md](./1-system-design-scenarios.md) | 5 kịch bản thiết kế hệ thống thực tế với hướng dẫn trả lời | Senior / Mid |
| [2-star-stories.md](./2-star-stories.md) | Mẫu câu chuyện STAR theo từng chủ đề kỹ thuật | Mọi cấp độ |
| [3-code-challenges.md](./3-code-challenges.md) | Bài tập lập trình thường gặp với giải thích và code mẫu | Junior / Mid |
| [4-behavioral-questions.md](./4-behavioral-questions.md) | Câu hỏi hành vi: teamwork, conflict, growth mindset | Mọi cấp độ |
| [5-90-day-study-plan.md](./5-90-day-study-plan.md) | Kế hoạch học 90 ngày có cấu trúc theo tuần | Mọi cấp độ |

---

## 🎯 Chiến Lược Phỏng Vấn Tổng Quan

### Phân Loại Câu Hỏi Phỏng Vấn .NET

```
┌─────────────────────────────────────────────────────────┐
│                  CÁC LOẠI CÂU HỎI                       │
├─────────────────────┬───────────────────────────────────┤
│  Kỹ Thuật (60%)     │  Hành Vi / Soft Skill (40%)        │
├─────────────────────┼───────────────────────────────────┤
│ • C# Language       │ • Teamwork & Collaboration          │
│ • ASP.NET Core      │ • Problem Solving (STAR)            │
│ • EF Core & DB      │ • Leadership & Ownership            │
│ • Architecture      │ • Communication & Growth            │
│ • Performance       │ • Conflict Resolution               │
│ • Testing           │ • Adaptability                      │
│ • Security          │                                     │
│ • Live Coding       │                                     │
└─────────────────────┴───────────────────────────────────┘
```

### Thứ Tự Ưu Tiên Ôn Tập

```
[Ưu tiên 1 — Bắt buộc nắm chắc]
  ✅ SOLID Principles + Design Patterns → luôn được hỏi
  ✅ async/await & Task — deadlock, ConfigureAwait → phổ biến nhất
  ✅ Dependency Injection: Singleton/Scoped/Transient → câu cơ bản
  ✅ JWT Authentication flow → 90% interview hỏi
  ✅ EF Core N+1 Problem → rất hay gặp

[Ưu tiên 2 — Nên biết cho Mid/Senior]
  ✅ Clean Architecture layers & dependency rule
  ✅ CQRS — Command Query Responsibility Segregation pattern
  ✅ Span<T> và zero-allocation programming
  ✅ System Design: cache, message queue, scaling
  ✅ Câu chuyện STAR thực chiến (ít nhất 3 câu)

[Ưu tiên 3 — Nâng cao / Senior+]
  ✅ Event Sourcing & Saga pattern
  ✅ Distributed tracing với OpenTelemetry
  ✅ Kubernetes deployment strategy
  ✅ Database sharding & partitioning
```

---

## 📅 Lịch Trình Ôn Tập Theo Thời Gian Có

### Có 1 Tuần (Phỏng Vấn Gấp)

```
Ngày 1: Đọc INTERVIEW_GUIDE.md — 30 câu hỏi
Ngày 2: SOLID + Design Patterns (Observer, Strategy, Factory)
Ngày 3: async/await + DI lifetimes + Middleware pipeline
Ngày 4: JWT Auth + EF Core N+1 + Clean Architecture tóm tắt
Ngày 5: 3 câu STAR stories + System Design cơ bản
Ngày 6: Luyện giải thích không nhìn tài liệu
Ngày 7: Mock interview + Behavioral questions
```

### Có 1 Tháng

```
Tuần 1: Nền tảng C# + OOP + Design Patterns
Tuần 2: ASP.NET Core + EF Core + Testing
Tuần 3: Architecture + Security + Performance
Tuần 4: System Design + STAR stories + Mock interviews
```

### Có 3 Tháng

Xem [5-90-day-study-plan.md](./5-90-day-study-plan.md) để có kế hoạch chi tiết theo tuần.

---

## 🧠 Framework Trả Lời Câu Hỏi Kỹ Thuật

### Cấu Trúc PREP — Chuẩn Bị — Raise — Explain — Prove

```
P — Point       Đưa ra định nghĩa / câu trả lời ngắn gọn (1-2 câu)
R — Reason      Giải thích tại sao / cách hoạt động
E — Example     Ví dụ code hoặc tình huống thực tế
P — Pitfall     Lỗi thường gặp / edge case / trade-off
```

**Ví dụ áp dụng PREP cho câu "Singleton vs Scoped vs Transient là gì?":**

```
P: Đây là 3 vòng đời (lifetime) của service trong DI container ASP.NET Core.

R: Singleton — tạo một lần, tái sử dụng suốt vòng đời app.
   Scoped — tạo mới mỗi HTTP request, dùng chung trong request đó.
   Transient — tạo mới mỗi lần resolve, nhẹ và stateless.

E: Singleton phù hợp cho HttpClient, IMemoryCache, cấu hình ứng dụng.
   Scoped phù hợp cho DbContext, unit of work.
   Transient phù hợp cho service không có state.

P: Lỗi hay gặp: inject Scoped service vào Singleton → "captive dependency"
   → DbContext bị dùng chung giữa nhiều request → race condition, stale data.
```

---

## 🎤 Framework Trả Lời Câu Hỏi Behavioral — STAR Method

```
S — Situation   Bối cảnh, tình huống cụ thể (2-3 câu)
T — Task        Nhiệm vụ / trách nhiệm của bạn trong tình huống đó
A — Action      Hành động cụ thể bạn đã làm (phần quan trọng nhất, 50%)
R — Result      Kết quả đo lường được, bài học rút ra
```

**Lưu ý quan trọng:**
- Luôn có **số liệu cụ thể** trong phần Result (giảm 40% latency, tăng coverage lên 80%...)
- Phần Action phải là "tôi đã làm" không phải "chúng tôi đã làm"
- Chuẩn bị ít nhất **5 câu chuyện** để cover nhiều loại câu hỏi khác nhau

---

## 💻 Chuẩn Bị Live Coding

### Quy Trình 45 Phút

```
[0-5 phút]   Đọc kỹ đề, hỏi clarifying questions
[5-10 phút]  Phác thảo approach, nêu ra assumptions
[10-35 phút] Viết code, chạy từng bước nhỏ
[35-40 phút] Test với edge cases (null, empty, boundary)
[40-45 phút] Nói về time/space complexity và cách tối ưu
```

### Clarifying Questions Hay Hỏi

```
- Input size? (ảnh hưởng algorithm choice)
- Input có thể null/empty không?
- Cần handle concurrent access không?
- Có memory constraint không?
- Expected output format?
```

### Chủ Đề Code Challenge Thường Gặp Với .NET

```
- String manipulation: palindrome, anagram, reverse
- Collections: group by, sort, distinct, flatten
- LINQ: complex queries, deferred execution
- Async: làm đúng async chain, không deadlock
- Generics: tạo generic repository hoặc utility
- Design Pattern: implement Observer hoặc Strategy
```

Xem chi tiết tại [3-code-challenges.md](./3-code-challenges.md).

---

## 🏗️ System Design Interview — Cách Tiếp Cận

### Framework 4 Bước

```
Bước 1 — Clarify Requirements (5 phút)
  "Hệ thống phục vụ bao nhiêu users?"
  "Read-heavy hay write-heavy?"
  "Consistency hay availability ưu tiên hơn?"
  "Latency requirement?"

Bước 2 — High-Level Design (10 phút)
  Vẽ các thành phần chính: Client → API → Service → DB
  Xác định data flow
  Nêu technology choices sơ bộ

Bước 3 — Deep Dive Components (20 phút)
  Database schema
  API design
  Caching strategy
  Queue & async processing
  Authentication & authorization

Bước 4 — Scale & Trade-offs (10 phút)
  Bottleneck ở đâu?
  Horizontal scaling như thế nào?
  CAP theorem — bạn chọn gì?
  Failure scenarios & resilience
```

Xem 5 kịch bản chi tiết tại [1-system-design-scenarios.md](./1-system-design-scenarios.md).

---

## 📊 Đánh Giá Mức Độ Sẵn Sàng

### Kiểm Tra Nhanh — Junior Level

- [ ] Giải thích được `value type` vs `reference type` với ví dụ
- [ ] Mô tả middleware pipeline trong ASP.NET Core
- [ ] Tạo được API CRUD với EF Core trong 30 phút
- [ ] Viết được unit test với xUnit và Moq
- [ ] Giải thích JWT — JSON Web Token — flow

### Kiểm Tra Nhanh — Mid Level

- [ ] Giải thích SOLID với ví dụ code thực tế cho mỗi principle
- [ ] Implement async/await đúng cách, tránh deadlock
- [ ] Thiết kế Clean Architecture cho một feature mới
- [ ] Debug và fix N+1 query trong EF Core
- [ ] Kể 3 câu chuyện STAR về technical challenges

### Kiểm Tra Nhanh — Senior Level

- [ ] Thiết kế hệ thống distributed với microservices
- [ ] Giải thích trade-off giữa CQRS và CRUD truyền thống
- [ ] Tối ưu memory với `Span<T>` và `ArrayPool<T>`
- [ ] Thiết kế authentication system từ đầu (OAuth2 + JWT)
- [ ] Mentor junior developer — kể chuyện thực tế

---

## 🔗 Điều Hướng Nhanh

| Câu Hỏi Tôi Cần | Đọc File |
| ---------------- | -------- |
| Top câu hỏi C# .NET phổ biến nhất | [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) |
| Kịch bản thiết kế hệ thống | [1-system-design-scenarios.md](./1-system-design-scenarios.md) |
| Câu chuyện STAR mẫu | [2-star-stories.md](./2-star-stories.md) |
| Luyện code challenge | [3-code-challenges.md](./3-code-challenges.md) |
| Câu hỏi behavioral / soft skill | [4-behavioral-questions.md](./4-behavioral-questions.md) |
| Kế hoạch học dài hạn | [5-90-day-study-plan.md](./5-90-day-study-plan.md) |

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
