# 🎯 Interview Prep — Chuẩn Bị Phỏng Vấn AWS Database

> Module tổng hợp giúp bạn tự tin đối mặt với mọi câu hỏi phỏng vấn về AWS Database Services — từ câu hỏi khái niệm, câu hỏi tình huống, đến bài toán thiết kế hệ thống.

## 📚 Mục Lục

1. [Tại Sao Module Này Quan Trọng](#tại-sao-module-này-quan-trọng)
2. [Nội Dung Module](#nội-dung-module)
3. [Chiến Lược Chuẩn Bị](#chiến-lược-chuẩn-bị)
4. [Lộ Trình 2 Tuần Trước Phỏng Vấn](#lộ-trình-2-tuần-trước-phỏng-vấn)
5. [Phân Loại Câu Hỏi Phỏng Vấn](#phân-loại-câu-hỏi-phỏng-vấn)
6. [Framework Trả Lời](#framework-trả-lời)

---

## Tại Sao Module Này Quan Trọng

Phỏng vấn kỹ thuật về AWS Database thường gồm ba loại câu hỏi chính:

```
1. Câu hỏi khái niệm   — "Giải thích Multi-AZ là gì?"
2. Câu hỏi tình huống  — "Nếu database chậm, bạn sẽ làm gì?"
3. System design        — "Thiết kế hệ thống e-commerce cho 10 triệu user"
```

Module này cung cấp công cụ và template để xử lý cả ba loại một cách tự tin và có cấu trúc.

---

## Nội Dung Module

| File | Nội Dung | Đối Tượng |
|------|----------|-----------|
| **INTERVIEW_GUIDE.md** | Top 20 câu hỏi + câu trả lời mẫu, tips phỏng vấn | Tất cả cấp độ |
| **system-design-scenarios.md** | 5 kịch bản thiết kế hệ thống hoàn chỉnh | Mid/Senior |
| **trade-off-discussions.md** | So sánh sâu SQL vs NoSQL, RDS vs DynamoDB vs Aurora | Mid/Senior |
| **star-stories.md** | 10 mẫu câu chuyện STAR về database incidents | Mid/Senior |

---

## Chiến Lược Chuẩn Bị

### Nguyên Tắc 80/20

Tập trung 80% thời gian vào 20% kiến thức xuất hiện trong 80% buổi phỏng vấn:

```
Luôn được hỏi (80%+ buổi phỏng vấn):
├── RDS Multi-AZ vs Read Replicas
├── DynamoDB Partition Key design
├── SQL vs NoSQL — khi nào dùng gì
├── RPO/RTO và backup strategy
└── Caching với ElastiCache

Thường được hỏi (40-60% buổi phỏng vấn):
├── Aurora vs RDS trade-offs
├── DynamoDB GSI/LSI design
├── Connection pooling với RDS Proxy
├── KMS encryption và Secrets Manager
└── Database migration với DMS

Ít được hỏi (20-40% buổi phỏng vấn):
├── DynamoDB Global Tables
├── Aurora Global Database
├── Redshift architecture
├── Neptune/DocumentDB
└── Advanced cost optimization
```

### Phương Pháp PREP (Point — Reason — Example — Point)

Dùng cho câu hỏi khái niệm:

```
P — Point (Điểm chính):    "Multi-AZ là cơ chế HA của RDS..."
R — Reason (Lý do):        "...giúp tự động failover khi primary instance lỗi..."
E — Example (Ví dụ):       "...ví dụ nếu us-east-1a bị sự cố, RDS tự chuyển sang us-east-1b..."
P — Point (Kết lại):       "...đảm bảo downtime < 2 phút mà không cần can thiệp thủ công."
```

### Phương Pháp STAR (Situation — Task — Action — Result)

Dùng cho câu hỏi kinh nghiệm:

```
S — Situation (Tình huống): Bối cảnh, vấn đề gặp phải
T — Task (Nhiệm vụ):        Trách nhiệm của bạn là gì
A — Action (Hành động):     Bạn đã làm gì cụ thể
R — Result (Kết quả):       Kết quả đo được, impact ra sao
```

---

## Lộ Trình 2 Tuần Trước Phỏng Vấn

### Tuần 1 — Ôn Tập Kiến Thức

| Ngày | Nội Dung | Thời Gian |
|------|----------|-----------|
| Ngày 1 | RDS fundamentals — Multi-AZ, Read Replicas, RDS Proxy | 2-3 giờ |
| Ngày 2 | Aurora — architecture, serverless, global database | 2-3 giờ |
| Ngày 3 | DynamoDB — data model, capacity modes, GSI/LSI | 3-4 giờ |
| Ngày 4 | ElastiCache — Redis vs Memcached, caching strategies | 2 giờ |
| Ngày 5 | HA & Backup — RPO/RTO, automated backups, PITR, DR | 2-3 giờ |
| Ngày 6 | Security — VPC, IAM, KMS, Secrets Manager | 2 giờ |
| Ngày 7 | Performance Tuning — Performance Insights, slow query | 2 giờ |

### Tuần 2 — Luyện Tập Trả Lời

| Ngày | Nội Dung | Thời Gian |
|------|----------|-----------|
| Ngày 8 | Đọc INTERVIEW_GUIDE.md — luyện 20 câu hỏi | 3-4 giờ |
| Ngày 9 | Đọc trade-off-discussions.md — học cách so sánh | 2-3 giờ |
| Ngày 10 | Đọc system-design-scenarios.md — luyện 2 scenarios | 3-4 giờ |
| Ngày 11 | Đọc star-stories.md — chọn 3 câu chuyện phù hợp | 2-3 giờ |
| Ngày 12 | Mock interview với đồng nghiệp hoặc tự nói to | 2-3 giờ |
| Ngày 13 | Ôn tập điểm yếu, xem lại các điểm chưa chắc | 2 giờ |
| Ngày 14 | Nghỉ ngơi, điểm qua checklist lần cuối | 1 giờ |

---

## Phân Loại Câu Hỏi Phỏng Vấn

### Loại 1 — Câu Hỏi Khái Niệm (Conceptual Questions)

> "Giải thích X là gì và khi nào nên dùng?"

Ví dụ:
- "RDS Proxy (Proxy RDS — Trung Gian Kết Nối) là gì?"
- "DynamoDB DAX (DynamoDB Accelerator — Bộ Nhớ Đệm DynamoDB) hoạt động như thế nào?"
- "Giải thích ACID trong DynamoDB"

**Chiến lược:** Dùng PREP framework, đừng quá dài, tập trung vào use case.

### Loại 2 — Câu Hỏi So Sánh (Comparison Questions)

> "A vs B — khác gì nhau và khi nào dùng cái nào?"

Ví dụ:
- "Multi-AZ vs Read Replicas — khác nhau chỗ nào?"
- "On-Demand vs Provisioned capacity trên DynamoDB?"
- "RDS vs DynamoDB — khi nào chọn cái nào?"

**Chiến lược:** Dùng bảng so sánh trong đầu, sau đó đưa ra decision framework rõ ràng.

### Loại 3 — Câu Hỏi Tình Huống (Scenario Questions)

> "Nếu xảy ra X, bạn sẽ làm gì?"

Ví dụ:
- "Database của bạn đột ngột chậm lúc 3 giờ sáng, bạn xử lý thế nào?"
- "Khách hàng báo lỗi timeout khi checkout, nguyên nhân có thể là gì?"
- "Làm thế nào để migrate 1TB database không có downtime?"

**Chiến lược:** Bắt đầu với data gathering (thu thập dữ liệu), sau đó phân tích, cuối cùng đưa ra action plan.

### Loại 4 — System Design Questions (Câu Hỏi Thiết Kế Hệ Thống)

> "Thiết kế hệ thống X cho Y người dùng"

Ví dụ:
- "Thiết kế hệ thống e-commerce với 10 triệu người dùng"
- "Thiết kế real-time leaderboard cho game"
- "Thiết kế hệ thống chat với lịch sử tin nhắn"

**Chiến lược:** Clarify requirements → Estimate scale → Choose database(s) → Explain trade-offs → Handle edge cases.

### Loại 5 — Câu Hỏi Kinh Nghiệm (Behavioral/Experience Questions)

> "Kể về một lần bạn xử lý X"

Ví dụ:
- "Kể về một lần database của bạn bị down"
- "Bạn đã tối ưu performance database như thế nào?"
- "Kể về một quyết định kiến trúc database bạn đã đưa ra"

**Chiến lược:** Dùng STAR format, có số liệu cụ thể, tập trung vào learning.

---

## Framework Trả Lời

### Cho System Design Questions

```
Bước 1 — Làm rõ yêu cầu (2-3 phút)
├── Quy mô: số users, QPS (Queries Per Second — Số Truy Vấn Mỗi Giây)
├── Tính năng cốt lõi cần hỗ trợ
├── Yêu cầu consistency (tính nhất quán) vs availability
└── Yêu cầu về latency (độ trễ)

Bước 2 — Ước lượng quy mô (1-2 phút)
├── Số lượng reads vs writes
├── Data volume (dung lượng dữ liệu)
└── Tăng trưởng dự kiến

Bước 3 — Lựa chọn database (3-5 phút)
├── SQL hay NoSQL? Lý do?
├── Dịch vụ AWS cụ thể?
└── Cần caching không?

Bước 4 — Chi tiết thiết kế (5-10 phút)
├── Schema design (thiết kế cấu trúc dữ liệu)
├── HA & DR strategy
└── Monitoring & alerting

Bước 5 — Thảo luận trade-offs (2-3 phút)
├── Điểm mạnh của thiết kế
├── Điểm yếu và cách giảm thiểu
└── Các phương án thay thế đã cân nhắc
```

### Cho Câu Hỏi Troubleshooting

```
Bước 1 — Thu thập thông tin
├── Vấn đề bắt đầu khi nào?
├── Có thay đổi gì gần đây không?
└── Ảnh hưởng đến bao nhiêu users?

Bước 2 — Xác định triệu chứng
├── Latency cao? CPU cao? Connection timeout?
└── Error logs nói gì?

Bước 3 — Phân tích nguyên nhân
├── Check Performance Insights
├── Check CloudWatch metrics
└── Check slow query log

Bước 4 — Giải pháp
├── Giải pháp tức thời (quick fix)
└── Giải pháp dài hạn (permanent fix)
```

---

## ✅ Checklist Trước Phỏng Vấn

### Kiến Thức

- [ ] Giải thích được Multi-AZ và Read Replicas không cần ghi chú
- [ ] Biết khi nào chọn RDS, Aurora, hay DynamoDB
- [ ] Hiểu DynamoDB partition key design principles
- [ ] Có thể thiết kế backup strategy cho RPO/RTO đã cho
- [ ] Biết caching patterns — Lazy Loading, Write-Through

### Kỹ Năng Trả Lời

- [ ] Chuẩn bị ít nhất 3 câu chuyện STAR về database
- [ ] Luyện nói to 5 câu hỏi thường gặp
- [ ] Biết cách vẽ sơ đồ kiến trúc đơn giản
- [ ] Chuẩn bị câu hỏi để hỏi ngược lại interviewer

### Logistics (Hậu Cần)

- [ ] Test setup video call nếu phỏng vấn online
- [ ] Có whiteboard/giấy để vẽ diagram
- [ ] Ngủ đủ giấc đêm trước

---

**Bước Tiếp Theo:** Bắt đầu với [INTERVIEW_GUIDE.md](INTERVIEW_GUIDE.md) để xem Top 20 câu hỏi thường gặp nhất.
