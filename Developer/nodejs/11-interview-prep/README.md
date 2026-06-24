# Chuẩn Bị Phỏng Vấn Node.js — Tổng Quan

> Bộ tài liệu ôn thi phỏng vấn Backend Node.js & JavaScript — câu hỏi lý thuyết, coding challenges, system design scenarios (tình huống thiết kế hệ thống), STAR stories (câu chuyện theo mô hình Situation-Task-Action-Result), và kế hoạch học 90 ngày.

## Mục Lục

1. [Tại Sao Cần Chuẩn Bị Riêng](#tại-sao-cần-chuẩn-bị-riêng)
2. [Cấu Trúc Phỏng Vấn Backend Node.js](#cấu-trúc-phỏng-vấn-backend-nodejs)
3. [Lộ Trình Ôn Thi](#lộ-trình-ôn-thi)
4. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
5. [Chiến Lược Trả Lời Hiệu Quả](#chiến-lược-trả-lời-hiệu-quả)
6. [Checklist Trước Phỏng Vấn](#checklist-trước-phỏng-vấn)
7. [Liên Kết Knowledge Base](#liên-kết-knowledge-base)

---

## Tại Sao Cần Chuẩn Bị Riêng

Kiến thức trong `01-fundamentals` đến `10-advanced` là **nền tảng**. Phỏng vấn yêu cầu thêm:

| Kỹ Năng Phỏng Vấn | Mô Tả |
| ----------------- | ----- |
| **Giải thích ngắn gọn** | Trả lời trong 2–3 phút, không lan man |
| **Trade-off thinking** | So sánh Express vs Fastify, SQL vs NoSQL, sync vs async |
| **Live coding** | Viết async patterns, API endpoint, error handling dưới áp lực thời gian |
| **Behavioral (hành vi)** | Kể câu chuyện STAR từ kinh nghiệm thực tế |
| **System design** | Thiết kế hệ thống với Node.js constraints (single-threaded, event-driven) |

**Nguyên tắc cốt lõi:** Interviewer (người phỏng vấn) không chỉ kiểm tra "biết hay không" — họ đánh giá **cách bạn suy nghĩ**, **cách debug**, và **cách làm việc trong team**.

---

## Cấu Trúc Phỏng Vấn Backend Node.js

```
┌─────────────────────────────────────────────────────────────┐
│              TYPICAL NODE.JS INTERVIEW ROUNDS               │
├─────────────────────────────────────────────────────────────┤
│  Round 1: Screening (30–45 phút)                            │
│    → JavaScript fundamentals, Event Loop cơ bản             │
│                                                             │
│  Round 2: Technical Deep Dive (60–90 phút)                  │
│    → Async patterns, Express/NestJS, Database, Security     │
│                                                             │
│  Round 3: System Design (45–60 phút)                        │
│    → Thiết kế URL shortener, chat app, notification system  │
│                                                             │
│  Round 4: Behavioral + Culture (30–45 phút)                 │
│    → STAR stories, conflict resolution, mentoring           │
│                                                             │
│  Round 5: Live Coding (45–60 phút)                          │
│    → Implement API, fix bug, optimize query                │
└─────────────────────────────────────────────────────────────┘
```

| Vòng | Trọng Tâm | Tài Liệu Ôn |
| ---- | --------- | ----------- |
| Screening | JS basics, Event Loop | [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md), [1-event-loop-questions.md](./1-event-loop-questions.md) |
| Technical | API, DB, Security | [2-api-design-questions.md](./2-api-design-questions.md), [3-database-questions.md](./3-database-questions.md), [4-security-questions.md](./4-security-questions.md) |
| System Design | Scalability, trade-offs | [5-system-design-scenarios.md](./5-system-design-scenarios.md) |
| Behavioral | Kinh nghiệm thực tế | [6-star-stories.md](./6-star-stories.md) |

---

## Lộ Trình Ôn Thi

### 2 Tuần Trước Phỏng Vấn (Crash Course)

```
Tuần 1:
├── Ngày 1–2: Event Loop + Promises + async/await (bắt buộc)
├── Ngày 3–4: Express middleware, REST API design, error handling
├── Ngày 5–6: Database (N+1, transactions, Redis caching)
└── Ngày 7:   Security (JWT, OWASP, rate limiting)

Tuần 2:
├── Ngày 1–2: INTERVIEW_GUIDE.md — Top 50 câu hỏi
├── Ngày 3–4: System design scenarios (2–3 bài)
├── Ngày 5:   Chuẩn bị 3 STAR stories
├── Ngày 6:   Mock interview với đồng nghiệp
└── Ngày 7:   Review nhẹ, nghỉ ngơi
```

### 90 Ngày (Học Toàn Diện)

Xem chi tiết tại [7-90-day-study-plan.md](./7-90-day-study-plan.md).

---

## Các Tài Liệu Chi Tiết

| File | Nội Dung | Thời Gian Ôn |
| ---- | -------- | ------------ |
| [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) | Top 50 câu hỏi + đáp án chi tiết | 4–6 giờ |
| [1-event-loop-questions.md](./1-event-loop-questions.md) | Event Loop & async deep dive | 2–3 giờ |
| [2-api-design-questions.md](./2-api-design-questions.md) | REST API, middleware, error handling | 2–3 giờ |
| [3-database-questions.md](./3-database-questions.md) | ORM, N+1, transactions, caching | 2–3 giờ |
| [4-security-questions.md](./4-security-questions.md) | JWT, OWASP, rate limiting | 1.5–2 giờ |
| [5-system-design-scenarios.md](./5-system-design-scenarios.md) | Bài toán thiết kế hệ thống | 3–4 giờ |
| [6-star-stories.md](./6-star-stories.md) | Template câu chuyện STAR | 2–3 giờ |
| [7-90-day-study-plan.md](./7-90-day-study-plan.md) | Kế hoạch học 90 ngày | Tham khảo |

---

## Chiến Lược Trả Lời Hiệu Quả

### Framework STAR cho Câu Hỏi Kỹ Thuật

```
1. CLARIFY    → Hỏi lại requirements, constraints
2. HIGH-LEVEL → Mô tả approach tổng quan (30 giây)
3. DEEP DIVE  → Giải thích chi tiết, trade-offs
4. EXAMPLE    → Ví dụ code hoặc kinh nghiệm thực tế
5. EDGE CASES → Đề cập edge cases và error handling
```

### Framework cho System Design

```
1. REQUIREMENTS   → Functional + non-functional (scale, latency)
2. ESTIMATION     → Back-of-envelope (QPS, storage, bandwidth)
3. HIGH-LEVEL     → Components diagram (API, DB, cache, queue)
4. DEEP DIVE      → Database schema, API design, caching
5. BOTTLENECKS    → Node.js constraints, scaling strategy
6. TRADE-OFFS     → So sánh alternatives đã cân nhắc
```

### Mẹo Live Coding

- **Nói to suy nghĩ** — interviewer muốn thấy thought process
- **Bắt đầu đơn giản** — working solution trước, optimize sau
- **Handle errors** — luôn có try/catch và validation cơ bản
- **Đặt câu hỏi** — "Input có thể null không?", "Cần pagination không?"

---

## Checklist Trước Phỏng Vấn

### Kiến Thức Kỹ Thuật

- [ ] Giải thích Event Loop mà không cần nhìn tài liệu
- [ ] Viết REST API endpoint với validation và error handling
- [ ] Giải thích JWT flow (access token + refresh token)
- [ ] Mô tả cách fix N+1 query problem
- [ ] So sánh Express vs Fastify vs NestJS
- [ ] Giải thích connection pool sizing
- [ ] Liệt kê OWASP Top 10 và cách prevent trong Node.js

### Kỹ Năng Mềm

- [ ] Chuẩn bị 3 STAR stories (incident, conflict, achievement)
- [ ] Biết mô tả project portfolio trong 2 phút
- [ ] Có câu hỏi ngược cho interviewer (team, tech stack, culture)
- [ ] Practice mock interview ít nhất 1 lần

### Chuẩn Bị Thực Tế

- [ ] Test microphone/camera (nếu remote interview)
- [ ] Chuẩn bị IDE hoặc online editor quen thuộc
- [ ] In hoặc mở sẵn notes (không đọc nguyên văn khi trả lời)
- [ ] Ngủ đủ giấc đêm trước phỏng vấn

---

## Liên Kết Knowledge Base

Khi cần ôn sâu lý thuyết, quay lại các chủ đề gốc:

| Chủ Đề Phỏng Vấn | Tài Liệu Gốc |
| ---------------- | ------------ |
| Event Loop | [02-async-programming/1-event-loop.md](../02-async-programming/1-event-loop.md) |
| Express & REST API | [03-web-frameworks/](../03-web-frameworks/) |
| Database & ORM | [04-data-access/](../04-data-access/) |
| JWT & OWASP | [05-security/](../05-security/) |
| Testing | [06-testing/](../06-testing/) |
| Performance | [07-performance/](../07-performance/) |
| Architecture | [08-architecture/](../08-architecture/) |
| Deployment | [09-cloud-deployment/](../09-cloud-deployment/) |

---

**Cập Nhật:** 2026-06-24 | **Trạng Thái:** ✅ Hoàn thành
