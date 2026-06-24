# Chủ Đề Nâng Cao — Tổng Quan

> Hướng dẫn các kỹ thuật Node.js nâng cao — Streams (luồng dữ liệu), Child Processes (tiến trình con), GraphQL, WebSockets (kết nối thời gian thực), gRPC, Serverless (không máy chủ), và Job Queues (hàng đợi công việc) — dành cho backend developer muốn xử lý use case phức tạp ngoài REST API thông thường.

## Mục Lục

1. [Tại Sao Học Chủ Đề Nâng Cao](#tại-sao-học-chủ-đề-nâng-cao)
2. [Phổ Công Nghệ Nâng Cao](#phổ-công-nghệ-nâng-cao)
3. [Workflow Chọn Công Nghệ](#workflow-chọn-công-nghệ)
4. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
5. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
6. [Bài Tập Thực Hành](#bài-tập-thực-hành)
7. [Advanced Checklist](#advanced-checklist)
8. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Học Chủ Đề Nâng Cao

REST API + CRUD đủ cho 80% use case, nhưng production systems thường cần thêm:

| Nhu Cầu | Công Nghệ Phù Hợp |
| ------- | ----------------- |
| Xử lý file lớn (GB) mà không OOM (Out Of Memory — Hết Bộ Nhớ) | Streams API |
| Chạy CLI tools, shell scripts từ Node.js | Child Processes |
| Client cần query linh hoạt, giảm over-fetching | GraphQL |
| Chat, notifications, live updates | WebSockets |
| Microservices giao tiếp nội bộ hiệu năng cao | gRPC |
| Event-driven, pay-per-use, auto-scale | Serverless (AWS Lambda) |
| Background jobs, scheduled tasks, retry | BullMQ / Agenda |

**Nguyên tắc cốt lõi:** Chỉ dùng công nghệ nâng cao khi có **lý do rõ ràng** — complexity (độ phức tạp) phải được justify bởi business requirement (yêu cầu nghiệp vụ).

---

## Phổ Công Nghệ Nâng Cao

```
┌─────────────────────────────────────────────────────────────────┐
│                    NODE.JS ADVANCED SPECTRUM                     │
│                                                                 │
│  I/O Intensive ◄──────────────────────────────────► Real-time   │
│                                                                 │
│  Streams          Child Processes    Job Queues    WebSockets   │
│  (file/network)   (CPU offload)      (async work)  (live push)  │
│                                                                 │
│  API Style:                                                     │
│  REST ──────► GraphQL (flexible query) ──────► gRPC (internal)  │
│                                                                 │
│  Deployment:                                                    │
│  Traditional Server ──────► Serverless (Lambda, Functions)      │
└─────────────────────────────────────────────────────────────────┘
```

| Công Nghệ | Độ Phức Tạp | Khi Nào Dùng |
| --------- | ----------- | ------------ |
| **Streams** | ⭐⭐ | File upload/download lớn, ETL pipeline, transform data |
| **Child Processes** | ⭐⭐ | Shell commands, image processing, legacy CLI tools |
| **GraphQL** | ⭐⭐⭐ | Mobile apps, nhiều client với query khác nhau |
| **WebSockets** | ⭐⭐⭐ | Chat, collaborative editing, live dashboards |
| **gRPC** | ⭐⭐⭐ | Service-to-service, streaming RPC, polyglot microservices |
| **Serverless** | ⭐⭐⭐ | Spiky traffic, event triggers, cost optimization |
| **Job Queues** | ⭐⭐ | Email sending, report generation, cron jobs |

---

## Workflow Chọn Công Nghệ

```
1. XÁC ĐỊNH USE CASE     → Real-time? Background? Large file? Internal RPC?
        ↓
2. ĐÁNH GIÁ ALTERNATIVES  → REST + polling? SSE? Message queue?
        ↓
3. TÍNH COMPLEXITY COST   → Ops overhead, learning curve, debugging difficulty
        ↓
4. PROTOTYPE NHỎ          → POC (Proof of Concept — Chứng Minh Khái Niệm) trước khi commit
        ↓
5. PRODUCTION HARDENING   → Error handling, monitoring, scaling strategy
```

**Quy tắc vàng:** REST + Redis + BullMQ giải quyết được hầu hết backend problems. Chỉ thêm GraphQL/WebSockets/gRPC khi REST không đủ.

---

## Lộ Trình Học Trong Chủ Đề

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-streams-api.md](./1-streams-api.md) | Readable, Writable, Transform, pipeline, backpressure | 2 giờ |
| 2 | [2-child-processes.md](./2-child-processes.md) | spawn, exec, fork, IPC | 1 giờ |
| 3 | [3-graphql.md](./3-graphql.md) | Apollo Server, resolvers, DataLoader, N+1 | 2 giờ |
| 4 | [4-websockets.md](./4-websockets.md) | Socket.io, ws, rooms, scaling | 1.5 giờ |
| 5 | [5-grpc.md](./5-grpc.md) | Protocol Buffers, streaming RPC, @grpc/grpc-js | 1.5 giờ |
| 6 | [6-serverless.md](./6-serverless.md) | AWS Lambda, cold start, event sources | 1.5 giờ |
| 7 | [7-job-queues.md](./7-job-queues.md) | BullMQ, Agenda, retry, scheduled jobs | 2 giờ |

**Tổng thời gian ước tính:** 12–15 giờ

---

## Các Tài Liệu Chi Tiết

### [1. Streams API](./1-streams-api.md)

Readable, Writable, Transform, Duplex streams — xử lý dữ liệu theo chunk (khối nhỏ), backpressure (áp lực ngược), `pipeline()` và `stream/promises`.

### [2. Child Processes](./2-child-processes.md)

`spawn`, `exec`, `execFile`, `fork` — chạy tiến trình con, IPC (Inter-Process Communication — Giao Tiếp Giữa Các Tiến Trình), shell commands.

### [3. GraphQL](./3-graphql.md)

Apollo Server, schema design, resolvers, DataLoader pattern — giải quyết N+1 trong GraphQL, subscriptions.

### [4. WebSockets](./4-websockets.md)

Socket.io vs `ws` library — real-time broadcasting, rooms, authentication, scaling với Redis adapter.

### [5. gRPC](./5-grpc.md)

Protocol Buffers (protobuf — định dạng nhị phân), unary/streaming RPC, `@grpc/grpc-js` trong Node.js microservices.

### [6. Serverless](./6-serverless.md)

AWS Lambda, API Gateway, cold start optimization, event-driven triggers, serverless framework.

### [7. Job Queues](./7-job-queues.md)

BullMQ với Redis, Agenda với MongoDB — background jobs, cron scheduling, retry policies, dead letter queue.

---

## Bài Tập Thực Hành

### Bài 1: File Processing với Streams

```
1. Tạo Transform stream gzip file CSV lớn (>100MB)
2. Dùng pipeline() kết nối read → transform → write
3. Monitor memory usage — verify không OOM
4. Handle backpressure khi consumer chậm
```

### Bài 2: GraphQL API

```
1. Setup Apollo Server với Express
2. Định nghĩa schema: User, Post, Comment
3. Implement resolvers với Prisma
4. Thêm DataLoader để fix N+1 query problem
```

### Bài 3: Real-time Chat

```
1. Socket.io server với room-based chat
2. JWT authentication cho WebSocket connection
3. Broadcast message tới room members
4. Redis adapter cho multi-instance scaling
```

### Bài 4: Background Job System

```
1. Setup BullMQ với Redis
2. Queue: email sending, report generation
3. Retry với exponential backoff
4. Dashboard monitoring với Bull Board
```

---

## Advanced Checklist

- [ ] Hiểu backpressure và cách Streams xử lý memory-efficient
- [ ] Biết khi nào dùng `spawn` vs `fork` vs Worker Threads
- [ ] Thiết kế GraphQL schema tránh over-fetching và N+1
- [ ] WebSocket scaling strategy với sticky sessions hoặc Redis pub/sub
- [ ] gRPC service definition với `.proto` files
- [ ] Lambda cold start mitigation strategies
- [ ] Job queue với idempotency (tính bất biến) và dead letter handling

---

## Câu Hỏi Phỏng Vấn Thường Gặp

| Câu Hỏi | Gợi Ý Trả Lời |
| ------- | ------------- |
| Streams khác gì đọc file sync toàn bộ? | Streams xử lý theo chunk — memory constant, phù hợp file lớn |
| Backpressure là gì? | Consumer chậm → producer pause — tránh buffer overflow |
| GraphQL vs REST — trade-offs? | GraphQL: flexible query, single endpoint; REST: caching đơn giản, tooling mature |
| WebSockets vs Server-Sent Events (SSE)? | WebSockets: bidirectional; SSE: server→client only, đơn giản hơn, HTTP-based |
| gRPC vs REST cho microservices? | gRPC: binary, fast, streaming; REST: human-readable, browser-friendly |
| Lambda cold start — cách giảm? | Provisioned concurrency, smaller bundle, avoid heavy init, ARM Graviton |
| BullMQ retry strategy? | Exponential backoff, max attempts, dead letter queue cho failed jobs |
| Khi nào dùng child process vs worker thread? | Child process: isolate process, run CLI; Worker thread: share memory, CPU tasks |

---

**Tiếp theo:** Bắt đầu với [1-streams-api.md](./1-streams-api.md) — nền tảng I/O hiệu quả trong Node.js.
