# 08 — Kiến Trúc Ứng Dụng Spring Boot

> Tổng quan về các mô hình kiến trúc phổ biến trong hệ sinh thái Spring Boot — từ Layered Architecture (Kiến Trúc Phân Tầng) truyền thống đến Microservices, Event-Driven Architecture (Kiến Trúc Hướng Sự Kiện) và các pattern chịu lỗi hiện đại.

---

## 📋 Mục Lục Module

| File | Chủ Đề | Độ Khó |
|------|--------|--------|
| [1-clean-architecture.md](1-clean-architecture.md) | Clean Architecture — Hexagonal, Ports & Adapters | ⭐⭐⭐ |
| [2-layered-architecture.md](2-layered-architecture.md) | Layered Architecture — Controller-Service-Repository | ⭐⭐ |
| [3-microservices.md](3-microservices.md) | Microservices — DDD, Service Decomposition | ⭐⭐⭐ |
| [4-api-gateway.md](4-api-gateway.md) | API Gateway — Spring Cloud Gateway | ⭐⭐⭐ |
| [5-service-discovery.md](5-service-discovery.md) | Service Discovery — Eureka, Consul | ⭐⭐⭐ |
| [6-event-driven.md](6-event-driven.md) | Event-Driven — CQRS, Event Sourcing, Saga | ⭐⭐⭐ |
| [7-circuit-breaker.md](7-circuit-breaker.md) | Circuit Breaker — Resilience4j | ⭐⭐⭐ |

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành module này, bạn có thể:

- [ ] Giải thích sự khác biệt giữa Monolith (Nguyên Khối) và Microservices
- [ ] Implement Clean Architecture (Kiến Trúc Sạch) với Ports & Adapters trong Spring Boot
- [ ] Thiết kế service boundaries (ranh giới dịch vụ) dựa trên DDD (Domain-Driven Design — Thiết Kế Hướng Miền)
- [ ] Cấu hình Spring Cloud Gateway với routing và rate limiting
- [ ] Triển khai Service Discovery (Khám Phá Dịch Vụ) với Eureka
- [ ] Implement CQRS (Command Query Responsibility Segregation) và Event Sourcing (Nguồn Sự Kiện)
- [ ] Xử lý lỗi hệ thống phân tán với Resilience4j Circuit Breaker (Cầu Dao Mạch)
- [ ] Thiết kế Saga Pattern (Mẫu Saga) cho distributed transactions (giao dịch phân tán)

---

## 🗺️ Bản Đồ Kiến Trúc

```
KIẾN TRÚC ĐƠN GIẢN → KIẾN TRÚC PHỨC TẠP

Layered Architecture          Clean Architecture
(Phân Tầng Truyền Thống)      (Hexagonal / Ports & Adapters)
    ↓                              ↓
    └──────────────────────────────┘
                 ↓
        Monolith (Nguyên Khối)
                 ↓
        Microservices (Vi Dịch Vụ)
         ├── API Gateway
         ├── Service Discovery
         ├── Event-Driven Architecture
         │    ├── CQRS
         │    ├── Event Sourcing
         │    └── Saga Pattern
         └── Resilience Patterns
              ├── Circuit Breaker
              ├── Retry
              └── Bulkhead
```

---

## 🏗️ Khi Nào Dùng Kiến Trúc Nào?

### Layered Architecture — Khi Nào Phù Hợp?

```
✅ Dùng khi:
- Team nhỏ (2–5 người)
- Deadline ngắn, cần ra thị trường nhanh
- Domain logic chưa phức tạp
- Budget hạn chế (infra đơn giản)
- MVP (Minimum Viable Product — Sản Phẩm Khả Dụng Tối Thiểu)

❌ Tránh khi:
- Scale > 20+ người phát triển
- Các domain có tốc độ thay đổi khác nhau
- Cần scale từng phần độc lập
```

### Clean Architecture — Khi Nào Phù Hợp?

```
✅ Dùng khi:
- Domain logic phức tạp, cần isolate khỏi framework
- Cần testability (khả năng kiểm thử) cao
- Long-lived codebase (codebase tồn tại lâu dài)
- Team muốn áp dụng DDD

❌ Tránh khi:
- Simple CRUD app không có logic nghiệp vụ
- Team chưa quen với pattern
```

### Microservices — Khi Nào Phù Hợp?

```
✅ Dùng khi:
- Scale lớn (50+ engineers)
- Các domain cần deploy độc lập
- Cần scale horizontally (mở rộng chiều ngang) từng service
- Technology stack diversity (đa dạng công nghệ)

❌ Tránh khi:
- Team nhỏ — operational overhead quá cao
- Domain chưa được hiểu rõ — khó phân chia đúng boundaries
- Không có DevOps capability (khả năng vận hành)
```

---

## 📐 Spring Cloud Ecosystem (Hệ Sinh Thái Spring Cloud)

```
Spring Cloud Components (Các Thành Phần Spring Cloud):

┌─────────────────────────────────────────────────┐
│              CLIENT SIDE                         │
│    Browser / Mobile / External API               │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│           API GATEWAY (Cổng API)                 │
│         Spring Cloud Gateway                     │
│   • Routing (Định Tuyến)                        │
│   • Rate Limiting (Giới Hạn Tốc Độ)            │
│   • Authentication (Xác Thực)                   │
│   • Load Balancing (Cân Bằng Tải)              │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│        SERVICE DISCOVERY (Khám Phá Dịch Vụ)     │
│          Eureka Server / Consul                  │
│   • Registry (Đăng Ký)                         │
│   • Health Checks (Kiểm Tra Sức Khỏe)         │
└──────┬────────────────────────────┬─────────────┘
       │                            │
┌──────▼──────┐              ┌──────▼──────┐
│  Service A  │              │  Service B  │
│  (Dịch Vụ) │◄────────────►│  (Dịch Vụ) │
│             │   REST/gRPC  │             │
│  Resilience │              │  Resilience │
│  4j CB      │              │  4j CB      │
└─────────────┘              └─────────────┘
       │                            │
       └────────────┬───────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│          MESSAGE BROKER (Trung Gian Tin Nhắn)    │
│             Kafka / RabbitMQ                     │
│   Event-Driven Communication (Giao Tiếp Hướng   │
│   Sự Kiện)                                       │
└─────────────────────────────────────────────────┘
```

---

## 🔑 Khái Niệm Cốt Lõi

### Domain-Driven Design (DDD — Thiết Kế Hướng Miền)

| Khái Niệm | Tiếng Anh | Giải Thích |
|-----------|-----------|------------|
| Bounded Context | Bounded Context | Ranh giới ngữ nghĩa của một domain |
| Aggregate | Aggregate | Nhóm entities được xử lý như một đơn vị |
| Domain Event | Domain Event | Sự kiện phản ánh thay đổi trong domain |
| Repository | Repository | Abstraction (Trừu Tượng Hóa) cho việc lưu trữ |
| Value Object | Value Object | Object không có identity, bất biến |
| Entity | Entity | Object có identity duy nhất |

### CAP Theorem (Định Lý CAP)

```
Trong hệ thống phân tán, chỉ có thể đảm bảo 2 trong 3:

C — Consistency (Nhất Quán): Mọi node đọc dữ liệu mới nhất
A — Availability (Sẵn Sàng): Mọi request đều nhận được response
P — Partition Tolerance (Chịu Phân Mảnh): Hệ thống hoạt động khi network bị chia

→ Microservices thường chọn AP (Availability + Partition Tolerance)
  → Đánh đổi Consistency → dùng Eventual Consistency (Nhất Quán Cuối Cùng)
```

---

## 📚 Thứ Tự Học Đề Xuất

```
Người Mới (Junior):
1. 2-layered-architecture.md     ← Bắt đầu tại đây
2. 1-clean-architecture.md
3. 7-circuit-breaker.md

Trung Cấp (Mid-level):
1. 3-microservices.md
2. 4-api-gateway.md
3. 5-service-discovery.md
4. 7-circuit-breaker.md

Nâng Cao (Senior):
1. 6-event-driven.md             ← CQRS & Event Sourcing
2. Toàn bộ module theo thứ tự
```

---

## 🔗 Liên Kết Nội Bộ

- **Trước khi học module này:** [07-performance/](../07-performance/README.md)
- **Sau khi học module này:** [09-cloud-deployment/](../09-cloud-deployment/README.md)
- **Liên quan:** [05-async-messaging/3-kafka-integration.md](../05-async-messaging/3-kafka-integration.md) — Kafka cho Event-Driven

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
