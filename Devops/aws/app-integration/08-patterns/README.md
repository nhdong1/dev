# 08 — Messaging Patterns — Mẫu Kiến Trúc Tích Hợp

> Tổng quan các mẫu (pattern) thiết kế hệ thống tích hợp phân tán: khi nào dùng gì, đánh đổi gì, và cách triển khai trên AWS.

---

## 📚 Mục Lục

1. [Tại Sao Cần Pattern?](#tại-sao-cần-pattern)
2. [Danh Sách Pattern Trong Module Này](#danh-sách-pattern-trong-module-này)
3. [Ma Trận Lựa Chọn Nhanh](#ma-trận-lựa-chọn-nhanh)
4. [Lộ Trình Học Đề Xuất](#lộ-trình-học-đề-xuất)
5. [Mối Quan Hệ Giữa Các Pattern](#mối-quan-hệ-giữa-các-pattern)

---

## Tại Sao Cần Pattern?

Trong kiến trúc microservices (vi dịch vụ) và hệ thống phân tán, các vấn đề lặp đi lặp lại:

| Vấn Đề | Nếu Không Có Pattern | Pattern Giải Quyết |
|---|---|---|
| Giao dịch trải qua nhiều service | Dữ liệu không nhất quán | Saga Pattern |
| Tin nhắn gửi trùng do retry | Xử lý hai lần, số liệu sai | Idempotency |
| DB và message bus tách biệt | Mất tin nhắn khi crash | Outbox Pattern |
| Lịch sử thay đổi trạng thái | Không audit được | Event Sourcing |
| Read/Write có nhu cầu khác nhau | Database bottleneck | CQRS |
| Downstream service bị lỗi | Lỗi lan dây chuyền | Circuit Breaker |
| Xử lý hàng đợi chậm | Queue tích tụ | Competing Consumers |
| Cần chọn đúng dịch vụ AWS | Over-engineer hoặc under-engineer | Service Comparison |

**Pattern không phải là silver bullet (giải pháp toàn năng)** — mỗi pattern giải quyết một vấn đề cụ thể và đi kèm với độ phức tạp riêng.

---

## Danh Sách Pattern Trong Module Này

### 1. [Service Comparison — So Sánh Dịch Vụ](./1-service-comparison.md)

**Câu hỏi:** SQS, SNS, EventBridge, Kinesis — dùng cái nào?

- Bảng so sánh toàn diện 4 dịch vụ AWS chính
- Decision tree (cây quyết định) theo use case
- Ví dụ kiến trúc thực tế

**Dùng khi:** Bắt đầu thiết kế hệ thống mới, cần chọn đúng dịch vụ.

---

### 2. [Saga Pattern](./2-saga-pattern.md)

**Câu hỏi:** Làm sao quản lý giao dịch phân tán qua nhiều service mà không dùng 2PC (Two-Phase Commit — Cam Kết Hai Giai Đoạn)?

- Choreography (Vũ Đạo — Event-driven) vs Orchestration (Điều Phối — Centralized)
- Compensating transactions (Giao Dịch Bù Trừ) — rollback phân tán
- Triển khai với Step Functions và EventBridge

**Dùng khi:** Order processing (xử lý đơn hàng), payment flow (luồng thanh toán), booking system (hệ thống đặt chỗ).

---

### 3. [Idempotency — Tính Bất Biến](./3-idempotency.md)

**Câu hỏi:** Điều gì xảy ra khi tin nhắn bị gửi hai lần? Làm sao xử lý an toàn?

- At-least-once (Ít Nhất Một Lần) vs Exactly-once (Đúng Một Lần) delivery
- Idempotency key (Khóa Bất Biến) — thiết kế và lưu trữ
- Deduplication (Loại Trùng) trong SQS FIFO và SNS FIFO
- Idempotency trong Lambda, SQS consumer

**Dùng khi:** Mọi consumer trong distributed system cần implement idempotency.

---

### 4. [Outbox Pattern — Hộp Thư Đi](./4-outbox-pattern.md)

**Câu hỏi:** Làm sao đảm bảo dữ liệu lưu vào DB và sự kiện gửi lên message bus là nhất quán?

- Vấn đề dual-write (ghi đôi) — tại sao nguy hiểm
- Transactional Outbox (Hộp Thư Đi Giao Dịch) với polling hoặc CDC (Change Data Capture — Thu Nạp Thay Đổi Dữ Liệu)
- Triển khai với DynamoDB Streams và EventBridge

**Dùng khi:** Cần đảm bảo "đã lưu DB thì chắc chắn có event, đã có event thì chắc chắn đã lưu DB".

---

### 5. [Event Sourcing — Nguồn Sự Kiện](./5-event-sourcing.md)

**Câu hỏi:** Thay vì lưu trạng thái hiện tại, có thể lưu toàn bộ lịch sử thay đổi không?

- Event Store (Kho Sự Kiện) vs Traditional Database (Cơ Sở Dữ Liệu Truyền Thống)
- Event Replay (Phát Lại Sự Kiện) — tái tạo trạng thái tại bất kỳ thời điểm
- Snapshot (Ảnh Chụp) — tối ưu hiệu năng
- Triển khai với Kinesis, DynamoDB, EventBridge

**Dùng khi:** Cần audit log (nhật ký kiểm tra) đầy đủ, tái tạo lịch sử, hoặc temporal queries (truy vấn theo thời gian).

---

### 6. [CQRS — Phân Tách Lệnh và Truy Vấn](./6-cqrs.md)

**Câu hỏi:** Read và Write có nhu cầu rất khác nhau — sao phải dùng cùng một model?

- CQRS — Command Query Responsibility Segregation (Phân Tách Trách Nhiệm Lệnh và Truy Vấn)
- Command Model (Mô Hình Lệnh) vs Query Model (Mô Hình Truy Vấn)
- Eventual Consistency (Nhất Quán Cuối Cùng) trong CQRS
- Kết hợp CQRS + Event Sourcing
- Triển khai với DynamoDB + OpenSearch hoặc RDS

**Dùng khi:** Read-heavy workload (tải trọng đọc nặng), cần tối ưu riêng cho read và write.

---

### 7. [Circuit Breaker — Cầu Dao Ngắt Lỗi](./7-circuit-breaker.md)

**Câu hỏi:** Nếu downstream service bị chậm hoặc lỗi, làm sao ngăn hệ thống sập theo?

- Circuit Breaker States (Trạng Thái Cầu Dao): Closed, Open, Half-Open
- Cascade Failure (Lỗi Dây Chuyền) — tại sao nguy hiểm
- Bulkhead (Vách Ngăn) pattern — tách biệt failure domain
- Triển khai với AWS Lambda, API Gateway, Step Functions

**Dùng khi:** Gọi external service (dịch vụ ngoài), third-party API, hoặc service trong cùng cluster có thể bị quá tải.

---

### 8. [Competing Consumers — Người Tiêu Dùng Cạnh Tranh](./8-competing-consumers.md)

**Câu hỏi:** Làm sao scale (mở rộng) xử lý hàng đợi song song mà không xử lý trùng?

- Competing Consumers Pattern — nhiều worker cùng xử lý một queue
- Visibility Timeout (Thời Gian Ẩn) và Message Lock
- Autoscaling (Tự Động Mở Rộng) consumer dựa trên queue depth
- Partition-based consumption (Tiêu Thụ Dựa Trên Phân Vùng) với Kinesis
- So sánh SQS Competing Consumers vs Kinesis per-shard consumer

**Dùng khi:** Cần tăng throughput (thông lượng) xử lý hàng đợi, background job processing (xử lý công việc nền).

---

## Ma Trận Lựa Chọn Nhanh

### Khi Gặp Vấn Đề → Dùng Pattern Nào?

```
Vấn đề: Giao dịch qua nhiều service có thể thất bại
  └─→ Saga Pattern (2-saga-pattern.md)

Vấn đề: Tin nhắn có thể bị gửi lại, cần xử lý an toàn
  └─→ Idempotency (3-idempotency.md)

Vấn đề: Cần đảm bảo DB write và message publish là atomic
  └─→ Outbox Pattern (4-outbox-pattern.md)

Vấn đề: Cần audit log đầy đủ và khả năng replay lịch sử
  └─→ Event Sourcing (5-event-sourcing.md)

Vấn đề: Read query phức tạp, write đơn giản hoặc ngược lại
  └─→ CQRS (6-cqrs.md)

Vấn đề: Downstream service hay bị lỗi, gây cascade failure
  └─→ Circuit Breaker (7-circuit-breaker.md)

Vấn đề: Queue tích tụ, cần xử lý nhanh hơn
  └─→ Competing Consumers (8-competing-consumers.md)

Vấn đề: Không biết chọn SQS, SNS, EventBridge, hay Kinesis
  └─→ Service Comparison (1-service-comparison.md)
```

### Khi Xây Dựng Use Case Cụ Thể

| Use Case | Pattern Cần Thiết |
|---|---|
| Order Processing (Xử lý đơn hàng) | Saga + Idempotency + Outbox |
| E-commerce notification | SNS Fan-out + Idempotency |
| Real-time analytics | Kinesis + CQRS |
| Audit system (Hệ thống kiểm tra) | Event Sourcing + CQRS |
| Payment processing (Xử lý thanh toán) | Saga + Idempotency + Circuit Breaker |
| Background job processing | Competing Consumers + DLQ |
| Microservices integration | EventBridge + Circuit Breaker |

---

## Mối Quan Hệ Giữa Các Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                    DISTRIBUTED SYSTEM LAYER                      │
│                   (Tầng Hệ Thống Phân Tán)                      │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              TRANSACTION PATTERNS (Mẫu Giao Dịch)        │   │
│  │                                                            │   │
│  │    Saga Pattern ◄──────────► Outbox Pattern               │   │
│  │         │                         │                        │   │
│  │         ▼                         ▼                        │   │
│  │    Idempotency ◄──────────► Event Sourcing                 │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │             SCALABILITY PATTERNS (Mẫu Mở Rộng)           │   │
│  │                                                            │   │
│  │   Competing Consumers ◄──────────► CQRS                   │   │
│  │          │                           │                     │   │
│  │          ▼                           ▼                     │   │
│  │    Circuit Breaker ◄────────► Service Comparison           │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**Nhóm hay đi cùng nhau:**
- **Saga + Idempotency** — bất kỳ distributed transaction nào đều cần cả hai
- **Event Sourcing + CQRS** — thường được triển khai cùng nhau
- **Outbox + Idempotency** — đảm bảo exactly-once semantics (ngữ nghĩa đúng một lần)
- **Competing Consumers + Circuit Breaker** — scale an toàn khi có dependency không ổn định

---

## Lộ Trình Học Đề Xuất

### Tuần 1 — Nền Tảng

1. **[1-service-comparison.md](./1-service-comparison.md)** — Chọn đúng dịch vụ (đọc đầu tiên)
2. **[3-idempotency.md](./3-idempotency.md)** — Áp dụng cho mọi consumer

### Tuần 2 — Giao Dịch Phân Tán

3. **[2-saga-pattern.md](./2-saga-pattern.md)** — Quản lý distributed transactions
4. **[4-outbox-pattern.md](./4-outbox-pattern.md)** — Nhất quán DB + messaging

### Tuần 3 — Kiến Trúc Nâng Cao

5. **[5-event-sourcing.md](./5-event-sourcing.md)** — Lưu trữ theo sự kiện
6. **[6-cqrs.md](./6-cqrs.md)** — Tách read/write model

### Tuần 4 — Resilience & Scale

7. **[7-circuit-breaker.md](./7-circuit-breaker.md)** — Chống cascade failure
8. **[8-competing-consumers.md](./8-competing-consumers.md)** — Scale song song

---

## Câu Hỏi Phỏng Vấn Thường Gặp

| Câu Hỏi | File Tham Khảo |
|---|---|
| SQS vs SNS vs EventBridge — khi nào dùng cái nào? | 1-service-comparison.md |
| Giải thích Saga Pattern, khác gì 2PC? | 2-saga-pattern.md |
| Tại sao cần idempotency trong SQS consumer? | 3-idempotency.md |
| Outbox Pattern giải quyết vấn đề gì? | 4-outbox-pattern.md |
| Event Sourcing là gì, ưu nhược điểm? | 5-event-sourcing.md |
| CQRS là gì, khi nào dùng? | 6-cqrs.md |
| Circuit Breaker hoạt động như thế nào? | 7-circuit-breaker.md |
| Làm sao scale SQS consumer? | 8-competing-consumers.md |

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
