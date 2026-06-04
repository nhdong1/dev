# Kiến Trúc Phần Mềm — Software Architecture

> Tổng quan về các mô hình kiến trúc phần mềm hiện đại trong hệ sinh thái .NET: khi nào dùng gì, đánh đổi ra sao, và cách triển khai thực tế.

---

## 📚 Mục Lục

| File | Chủ Đề | Tóm Tắt |
|------|---------|---------|
| [1-clean-architecture.md](1-clean-architecture.md) | Clean Architecture | Domain / Application / Infrastructure / Presentation layers |
| [2-ddd-fundamentals.md](2-ddd-fundamentals.md) | DDD — Domain-Driven Design | Aggregate, Entity, Value Object, Domain Events |
| [3-cqrs-pattern.md](3-cqrs-pattern.md) | CQRS | Command/Query split, MediatR integration |
| [4-event-sourcing.md](4-event-sourcing.md) | Event Sourcing | Event store, replay, snapshots |
| [5-microservices-basics.md](5-microservices-basics.md) | Microservices | Decomposition, communication, API Gateway |
| [6-messaging-patterns.md](6-messaging-patterns.md) | Messaging Patterns | MassTransit, RabbitMQ, Outbox Pattern |
| [7-vertical-slice.md](7-vertical-slice.md) | Vertical Slice Architecture | Slice theo tính năng, thay thế layered |

---

## 🗺️ Bức Tranh Tổng Thể

```
┌─────────────────────────────────────────────────────────┐
│                   KIẾN TRÚC PHẦN MỀM                    │
│                                                         │
│  Monolith (Đơn khối)          Microservices (Vi dịch vụ)│
│  ┌──────────────────┐         ┌─────────────────────┐   │
│  │  Clean Arch      │         │  Service A          │   │
│  │  ┌────────────┐  │         │  Service B          │   │
│  │  │ Domain     │  │   ──►   │  Service C          │   │
│  │  │ App Layer  │  │         │  API Gateway        │   │
│  │  │ Infra      │  │         │  Message Bus        │   │
│  │  └────────────┘  │         └─────────────────────┘   │
│  └──────────────────┘                                   │
│                                                         │
│  Patterns Bên Trong:                                    │
│  DDD + CQRS + Event Sourcing + Vertical Slice           │
└─────────────────────────────────────────────────────────┘
```

---

## 🎯 Khi Nào Dùng Gì

### Clean Architecture — Kiến Trúc Sạch

**Dùng khi:**
- Ứng dụng có business logic — logic nghiệp vụ — phức tạp
- Cần test được domain logic mà không phụ thuộc database
- Team muốn rõ ràng về ranh giới trách nhiệm

**Tránh khi:**
- CRUD đơn giản — Create, Read, Update, Delete — ít logic
- Prototype — bản thử nghiệm nhanh

---

### DDD — Domain-Driven Design — Thiết Kế Hướng Miền

**Dùng khi:**
- Domain — miền nghiệp vụ — phức tạp (e-commerce, banking, logistics)
- Cần cộng tác chặt với domain expert — chuyên gia nghiệp vụ
- Nhiều bounded context — ngữ cảnh giới hạn cần rõ ràng

**Tránh khi:**
- CRUD thuần túy, không có logic nghiệp vụ phức tạp
- Team nhỏ, timeline ngắn

---

### CQRS — Command Query Responsibility Segregation — Phân Tách Trách Nhiệm Đọc/Ghi

**Dùng khi:**
- Read/Write workload — khối lượng đọc/ghi mất cân đối (nhiều đọc hơn ghi)
- Cần tối ưu query riêng biệt với write model
- Kết hợp với Event Sourcing

**Tránh khi:**
- Đơn giản — thêm CQRS chỉ tạo thêm boilerplate
- Team chưa quen với pattern

---

### Event Sourcing — Nguồn Sự Kiện

**Dùng khi:**
- Cần audit trail — vết kiểm toán — đầy đủ
- Business yêu cầu "time travel" — quay lại trạng thái quá khứ
- Tích hợp hệ thống phức tạp

**Tránh khi:**
- Không cần lịch sử thay đổi
- Team chưa kinh nghiệm — learning curve — đường cong học cao

---

### Microservices — Vi Dịch Vụ

**Dùng khi:**
- Team lớn, nhiều nhóm phát triển độc lập
- Scale — mở rộng từng service riêng biệt theo nhu cầu
- Deployment độc lập — triển khai không ảnh hưởng nhau

**Tránh khi:**
- Team nhỏ (< 10 người)
- Distributed systems complexity — phức tạp hệ thống phân tán chưa cần thiết
- Chưa có monolith ổn định để tách

---

## 📊 Ma Trận So Sánh

| Tiêu Chí | Layered | Clean Arch | Vertical Slice | Microservices |
|----------|---------|------------|----------------|---------------|
| Độ phức tạp | Thấp | Trung bình | Trung bình | Cao |
| Testability | Trung bình | Cao | Cao | Cao |
| Team size | Nhỏ | Vừa | Vừa-Lớn | Lớn |
| Scale độc lập | Không | Không | Không | Có |
| Thời gian setup | Nhanh | Vừa | Vừa | Chậm |

---

## 🔗 Kết Hợp Patterns

Các pattern này **không loại trừ nhau** — có thể kết hợp:

```
Microservices (macro)
  └── Clean Architecture (trong từng service)
        └── DDD (domain modeling)
              └── CQRS (command/query)
                    └── Event Sourcing (write side)
```

**Ví dụ thực tế phổ biến:**
- Monolith + Clean Architecture + DDD
- Monolith + Vertical Slice + CQRS/MediatR
- Microservices + Clean Architecture + CQRS + Message Bus

---

## 🏗️ Lộ Trình Học Trong Section Này

```
1. Clean Architecture (nền tảng, đọc trước)
   → Hiểu layering và dependency rule

2. DDD Fundamentals (kết hợp với Clean Architecture)
   → Hiểu modeling domain entities

3. CQRS Pattern (thực tế nhất, dùng MediatR)
   → Tách command và query

4. Event Sourcing (nâng cao)
   → Lưu state qua events

5. Microservices Basics (sau khi nắm monolith tốt)
   → Khi nào và cách tách service

6. Messaging Patterns (đi kèm Microservices)
   → Giao tiếp bất đồng bộ

7. Vertical Slice (phương án thay thế thực dụng)
   → Feature-first organization
```

---

## 💬 Câu Hỏi Phỏng Vấn Thường Gặp

1. **"Bạn giải thích Clean Architecture là gì và tại sao dùng nó?"**
   → Nói về dependency rule, testability, separation of concerns

2. **"DDD khác gì với Layered Architecture thông thường?"**
   → DDD focus vào domain model, ubiquitous language, bounded context

3. **"CQRS giải quyết vấn đề gì? Nhược điểm là gì?"**
   → Tách read/write để optimize, nhưng tăng complexity

4. **"Bạn sẽ migrate monolith sang microservices như thế nào?"**
   → Strangler Fig Pattern, domain boundaries, event-driven

5. **"Outbox Pattern là gì và tại sao cần nó?"**
   → Đảm bảo at-least-once delivery khi vừa save DB vừa publish message

---

**Cập Nhật Lần Cuối:** 2026-06-02
