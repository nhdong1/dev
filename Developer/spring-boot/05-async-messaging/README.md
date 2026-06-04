# 05 — Bất Đồng Bộ & Nhắn Tin (Async & Messaging)

> Module này bao gồm toàn bộ kiến thức về lập trình bất đồng bộ và hệ thống nhắn tin trong Spring Boot —
> từ `@Async`, `CompletableFuture`, Spring Events, tích hợp Apache Kafka, RabbitMQ,
> đến các tác vụ định kỳ với `@Scheduled` và Quartz Scheduler.

---

## 📋 Mục Tiêu Module

Sau khi hoàn thành module này, bạn có thể:

- [ ] Sử dụng **`@Async`** (Bất Đồng Bộ) và **`CompletableFuture`** (Tương Lai Hoàn Thành) để xử lý tác vụ không chặn luồng chính
- [ ] Cấu hình **`ThreadPoolTaskExecutor`** (Trình Thực Thi Hồ Luồng) với các thông số phù hợp cho production
- [ ] Publish và consume **Spring Events** (Sự Kiện Spring) — `ApplicationEvent`, `@EventListener`, `@TransactionalEventListener`
- [ ] Tích hợp **Apache Kafka** — `@KafkaListener`, `KafkaTemplate`, Consumer Groups (Nhóm Consumer), offset management
- [ ] Tích hợp **RabbitMQ** — `@RabbitListener`, Exchange (Bộ Trao Đổi), Queue (Hàng Đợi), Binding (Liên Kết)
- [ ] Lập lịch tác vụ định kỳ với **`@Scheduled`** và **Quartz Scheduler** (Trình Lập Lịch Quartz)

---

## 🗂️ Danh Sách Bài Học

| File | Chủ Đề | Thời Gian | Độ Khó |
|------|--------|-----------|--------|
| [1-async-annotations.md](1-async-annotations.md) | @Async, CompletableFuture, ThreadPoolTaskExecutor, exception handling | 90 phút | ⭐⭐ |
| [2-spring-events.md](2-spring-events.md) | ApplicationEvent, @EventListener, @TransactionalEventListener, async events | 60 phút | ⭐⭐ |
| [3-kafka-integration.md](3-kafka-integration.md) | @KafkaListener, KafkaTemplate, Consumer Groups, partitions, error handling | 120 phút | ⭐⭐⭐ |
| [4-rabbitmq-integration.md](4-rabbitmq-integration.md) | @RabbitListener, Direct/Fanout/Topic Exchange, Dead Letter Queue, retry | 120 phút | ⭐⭐⭐ |
| [5-scheduled-tasks.md](5-scheduled-tasks.md) | @Scheduled, cron expressions, Quartz Scheduler, distributed locking | 90 phút | ⭐⭐ |

**Tổng thời gian ước tính: 6–8 giờ**

---

## 🔁 Thứ Tự Học Khuyến Nghị

```
1-async-annotations.md       ← Nền tảng — học trước tiên
      ↓
2-spring-events.md           ← Loose coupling trong ứng dụng đơn
      ↓
5-scheduled-tasks.md         ← Tác vụ định kỳ — đơn giản, thực tế
      ↓
4-rabbitmq-integration.md    ← Message broker — RabbitMQ
      ↓
3-kafka-integration.md       ← Event streaming — Kafka phức tạp hơn
```

---

## 🧠 Bức Tranh Tổng Thể

```
┌──────────────────────────────────────────────────────────────────┐
│                    Async & Messaging Patterns                    │
│                                                                  │
│  ┌──────────────┐   ┌──────────────┐   ┌─────────────────────┐  │
│  │   @Async     │   │   Events     │   │   Scheduled Tasks   │  │
│  │ Thread Pool  │   │ ApplicationE │   │   @Scheduled        │  │
│  │ Completable  │   │ @EventList   │   │   Quartz Scheduler  │  │
│  │ Future       │   │ @Transact    │   │   Cron Expressions  │  │
│  └──────┬───────┘   └──────┬───────┘   └─────────────────────┘  │
│         │                  │                                     │
│         └────────┬─────────┘                                     │
│                  │  In-Process (Trong cùng JVM)                  │
│  ────────────────┼──────────────────────────────────────────     │
│                  │  Out-of-Process (Qua Message Broker)          │
│         ┌────────┴─────────┐                                     │
│         │                  │                                     │
│  ┌──────▼───────┐   ┌──────▼───────┐                            │
│  │   RabbitMQ   │   │    Kafka     │                            │
│  │ @RabbitList  │   │ @KafkaList   │                            │
│  │ Exchanges    │   │ KafkaTemplate│                            │
│  │ Queues       │   │ Consumer Grp │                            │
│  └──────────────┘   └──────────────┘                            │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔑 Các Khái Niệm Cốt Lõi

### Synchronous vs Asynchronous (Đồng Bộ vs Bất Đồng Bộ)

| Đặc Điểm | Synchronous (Đồng Bộ) | Asynchronous (Bất Đồng Bộ) |
|-----------|----------------------|---------------------------|
| **Luồng thực thi** | Chặn luồng gọi đến khi xong | Trả về ngay, xử lý ở luồng khác |
| **Throughput** | Thấp hơn | Cao hơn |
| **Độ phức tạp** | Đơn giản | Phức tạp hơn |
| **Use case** | CRUD đơn giản | Email, report, upload file |

### Message Broker (Broker Nhắn Tin) — Kafka vs RabbitMQ

| Tiêu Chí | Apache Kafka | RabbitMQ |
|-----------|-------------|----------|
| **Mô hình** | Event Log (Nhật Ký Sự Kiện) — pull | Message Queue (Hàng Đợi) — push |
| **Thứ tự** | Đảm bảo trong partition | Đảm bảo trong queue |
| **Lưu trữ** | Giữ message lâu dài (configurable) | Xóa sau khi consume |
| **Throughput** | Hàng triệu msg/giây | Hàng chục nghìn msg/giây |
| **Use case** | Event streaming, log, analytics | Task queue, RPC, workflow |
| **Độ phức tạp** | Cao (Zookeeper/KRaft) | Thấp hơn |

### Spring Events vs Message Broker

| | Spring Events | Message Broker |
|---|---|---|
| **Phạm vi** | Trong cùng JVM | Nhiều service/JVM |
| **Reliability** (Độ Tin Cậy) | Mất khi app crash | Bền vững (persistent) |
| **Use case** | Loose coupling trong monolith | Microservices communication |

---

## ⚡ Khi Nào Dùng Gì?

```
Gửi email sau khi đặt hàng?
  └─► @Async — đơn giản, cùng service

Notify nhiều module khi tạo User?
  └─► Spring Events — loose coupling, in-process

Xử lý log/analytics với throughput cao?
  └─► Apache Kafka — event streaming

Phân phối tasks cho worker pools?
  └─► RabbitMQ — work queue pattern

Gửi report hàng ngày lúc 8 giờ sáng?
  └─► @Scheduled hoặc Quartz — cron job
```

---

## ⚠️ Lỗi Thường Gặp

| Lỗi | Nguyên Nhân | Cách Sửa |
|-----|-------------|----------|
| `@Async` không hoạt động | Gọi method trong cùng class (self-invocation) | Inject bean vào chính nó hoặc tách class |
| `@Async` không hoạt động | Thiếu `@EnableAsync` | Thêm vào `@Configuration` class |
| Spring Event không transactional | Dùng `@EventListener` thay vì `@TransactionalEventListener` | Chuyển sang `@TransactionalEventListener(phase = AFTER_COMMIT)` |
| Kafka consumer không nhận message | Sai `group-id` hoặc topic | Kiểm tra `spring.kafka.consumer.group-id` và topic name |
| RabbitMQ message mất | Queue không durable | Set `durable = true` cho queue và exchange |
| `@Scheduled` chạy nhiều lần | Nhiều instance app chạy cùng lúc | Dùng Quartz với database lock hoặc ShedLock |

---

## 📊 Luồng Xử Lý Đặt Hàng (Ví Dụ Thực Tế)

```
Client ──► POST /orders ──► OrderController
                                  │
                          OrderService.createOrder()
                                  │
                    ┌─────────────┼──────────────────┐
                    │             │                  │
              [đồng bộ]     [@Async]          [Kafka Event]
                    │             │                  │
            Lưu DB        Gửi email          order.created
                         xác nhận         ──────────────►
                                          InventoryService
                                          (consumer khác)
```

---

## 📚 Tài Liệu Tham Khảo

- [Spring @Async Reference](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#scheduling-annotation-support-async)
- [Spring for Apache Kafka](https://spring.io/projects/spring-kafka)
- [Spring AMQP (RabbitMQ)](https://spring.io/projects/spring-amqp)
- [Spring Task Scheduling](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#scheduling)
- [Quartz Scheduler](http://www.quartz-scheduler.org/documentation/)

---

## 🎯 Câu Hỏi Phỏng Vấn Quan Trọng

- [ ] `@Async` hoạt động thế nào? Tại sao self-invocation không hoạt động?
- [ ] Sự khác biệt giữa `@EventListener` và `@TransactionalEventListener`?
- [ ] Kafka Consumer Group (Nhóm Consumer) là gì? Cách scale consumer?
- [ ] Sự khác biệt giữa Kafka partition và RabbitMQ queue?
- [ ] Dead Letter Queue (Hàng Đợi Thư Chết) là gì? Khi nào dùng?
- [ ] Cách đảm bảo `@Scheduled` chỉ chạy một lần trong môi trường multi-instance?
- [ ] At-least-once vs exactly-once delivery trong Kafka?
- [ ] Cách handle exception trong Kafka consumer?
