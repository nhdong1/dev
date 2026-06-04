# 10 — Chủ Đề Nâng Cao (Advanced Topics)

> Khám phá các kỹ thuật nâng cao trong hệ sinh thái Spring Boot — từ Reactive Programming (Lập Trình Phản Ứng) với WebFlux, xử lý batch data (dữ liệu hàng loạt), đến giao tiếp hiệu năng cao với gRPC, API linh hoạt với GraphQL, và biên dịch native siêu nhanh với GraalVM.

---

## 📋 Mục Lục Module

| File | Chủ Đề | Độ Khó |
|------|--------|--------|
| [1-reactive-programming.md](1-reactive-programming.md) | Spring WebFlux — Mono, Flux, Project Reactor | ⭐⭐⭐⭐ |
| [2-spring-batch.md](2-spring-batch.md) | Spring Batch — Job, Step, Partitioning | ⭐⭐⭐ |
| [3-grpc-integration.md](3-grpc-integration.md) | gRPC — Protocol Buffers, Streaming | ⭐⭐⭐⭐ |
| [4-graphql.md](4-graphql.md) | Spring GraphQL — Schema-first, DataLoader | ⭐⭐⭐ |
| [5-native-image.md](5-native-image.md) | GraalVM Native Image — AOT Compilation | ⭐⭐⭐⭐ |

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành module này, bạn có thể:

- [ ] Xây dựng Reactive API (API Phản Ứng) với Spring WebFlux, sử dụng `Mono` và `Flux`
- [ ] Hiểu Backpressure (Áp Lực Ngược) và cách Project Reactor xử lý luồng dữ liệu
- [ ] Thiết kế Spring Batch Job (Công Việc Batch) với Job → Step → ItemReader/Processor/Writer
- [ ] Implement gRPC server/client với Protocol Buffers trong Spring Boot
- [ ] Xây dựng GraphQL API với Spring for GraphQL theo hướng schema-first
- [ ] Giải quyết N+1 problem trong GraphQL bằng DataLoader (Trình Tải Dữ Liệu)
- [ ] Biên dịch Spring Boot application thành GraalVM Native Image (Ảnh Nhị Phân Gốc)
- [ ] Thảo luận trade-off giữa JVM runtime và Native Image trong production

---

## 🗺️ Bản Đồ Chủ Đề Nâng Cao

```
SPRING BOOT ADVANCED TOPICS

┌─────────────────────────────────────────────────────────────────┐
│                  REACTIVE PROGRAMMING (WebFlux)                  │
│   Mono<T>  ─── Flux<T>  ─── Operators  ─── Backpressure        │
│   R2DBC     ─── WebClient ─── SSE ─── Reactive Security        │
└─────────────────────────────────────────────────────────────────┘
          ↓                              ↓
┌─────────────────┐            ┌─────────────────────┐
│  SPRING BATCH   │            │       gRPC           │
│  Job → Step     │            │  Proto → Stub        │
│  Chunk-oriented │            │  Unary / Streaming   │
│  Partitioning   │            │  Spring Boot gRPC    │
└─────────────────┘            └─────────────────────┘
          ↓                              ↓
┌─────────────────┐            ┌─────────────────────┐
│    GRAPHQL      │            │   GRAALVM NATIVE     │
│  Schema-first   │            │  AOT Compilation     │
│  DataFetcher    │            │  Reflection Hints    │
│  DataLoader     │            │  Build Tools         │
└─────────────────┘            └─────────────────────┘
```

---

## ⚡ So Sánh Nhanh Các Công Nghệ

| Công Nghệ | Khi Nào Dùng | Điểm Mạnh | Điểm Yếu |
|-----------|-------------|-----------|----------|
| **Spring MVC** | CRUD API thông thường, JPA | Đơn giản, debug dễ | Blocking I/O |
| **Spring WebFlux** | High-concurrency, streaming, gateway | Non-blocking, throughput cao | Khó debug, learning curve cao |
| **Spring Batch** | ETL, xử lý file lớn, báo cáo | Retry, skip, partitioning built-in | Nặng cho task đơn giản |
| **gRPC** | Internal microservices, low-latency | Binary protocol, streaming, code gen | Khó test bằng browser/curl |
| **GraphQL** | Mobile/BFF (Backend For Frontend) | Flexible query, no over-fetching | N+1 problem, caching phức tạp |
| **Native Image** | Serverless, startup-critical | Startup <100ms, ít RAM | Build chậm, reflection giới hạn |

---

## 🔗 Kiến Thức Cần Có Trước

Trước khi học module này, bạn nên vững:

| Chủ Đề | Tài Liệu |
|--------|---------|
| Spring IoC & DI | [01-fundamentals/2-spring-core.md](../01-fundamentals/2-spring-core.md) |
| REST Controllers | [02-web-layer/1-rest-controllers.md](../02-web-layer/1-rest-controllers.md) |
| Spring Data JPA | [03-data-access/1-spring-data-jpa.md](../03-data-access/1-spring-data-jpa.md) |
| Async & CompletableFuture | [05-async-messaging/1-async-annotations.md](../05-async-messaging/1-async-annotations.md) |
| Docker & Kubernetes | [09-cloud-deployment/1-docker-spring.md](../09-cloud-deployment/1-docker-spring.md) |

---

## 📊 Lộ Trình Học Module Này

```
Tuần 1: Reactive Programming
  ├─ Ngày 1–2: Reactive Streams spec — Publisher, Subscriber, Subscription
  ├─ Ngày 3–4: Project Reactor — Mono, Flux, operators
  ├─ Ngày 5–6: Spring WebFlux — RouterFunction vs @Controller, R2DBC
  └─ Ngày 7:   Reactive Security, WebClient, bài tập thực hành

Tuần 2: Spring Batch & gRPC
  ├─ Ngày 1–3: Spring Batch — Job config, chunk processing, partitioning
  ├─ Ngày 4–5: gRPC — proto files, code generation, server/client
  └─ Ngày 6–7: gRPC streaming — server/client/bidirectional

Tuần 3: GraphQL & Native Image
  ├─ Ngày 1–3: Spring GraphQL — schema, resolvers, DataLoader
  ├─ Ngày 4–5: GraalVM Native Image — AOT, hints, build tools
  └─ Ngày 6–7: Ôn tập, bài tập tổng hợp, câu hỏi phỏng vấn
```

---

## 💡 Ghi Chú Quan Trọng

> **Reactive Programming không phải lúc nào cũng tốt hơn.**
> Nếu ứng dụng của bạn chủ yếu làm CRUD với database quan hệ,
> Spring MVC + JPA thường đơn giản hơn và dễ maintain hơn.
> WebFlux tỏa sáng khi có nhiều concurrent connections và I/O-intensive tasks.

> **gRPC phù hợp cho internal communication.**
> Đừng expose gRPC trực tiếp ra client mobile/web — thay vào đó dùng
> API Gateway để convert gRPC → REST/GraphQL cho external clients.

> **GraalVM Native Image không phải silver bullet.**
> Startup time nhanh và RAM thấp là lợi thế, nhưng build time lâu (5–10 phút),
> reflection có giới hạn, và một số thư viện chưa tương thích hoàn toàn.

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
