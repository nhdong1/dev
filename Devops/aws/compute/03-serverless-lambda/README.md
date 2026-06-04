# ⚡ AWS Lambda & Serverless — Tổng Quan Kiến Trúc Không Máy Chủ

> Lambda (Hàm Không Máy Chủ) là dịch vụ compute của AWS cho phép chạy code mà không cần quản lý server. AWS tự động cấp phát tài nguyên, co giãn từ 0 đến hàng ngàn lần chạy song song, và chỉ tính phí theo thời gian thực thi thực tế.

## 📚 Mục Lục

1. [Serverless Là Gì?](#serverless-là-gì)
2. [Lambda Hoạt Động Thế Nào?](#lambda-hoạt-động-thế-nào)
3. [Khi Nào Dùng Lambda?](#khi-nào-dùng-lambda)
4. [Kiến Trúc Event-Driven](#kiến-trúc-event-driven)
5. [Các Khái Niệm Cốt Lõi](#các-khái-niệm-cốt-lõi)
6. [Tổng Quan Các File Trong Topic](#tổng-quan-các-file-trong-topic)
7. [Lambda vs Các Dịch Vụ Khác](#lambda-vs-các-dịch-vụ-khác)
8. [Câu Hỏi Phỏng Vấn Nhanh](#câu-hỏi-phỏng-vấn-nhanh)

---

## ☁️ Serverless Là Gì?

**Serverless** (Không Máy Chủ) không có nghĩa là không có server — server vẫn tồn tại, nhưng bạn không cần quan tâm đến việc quản lý chúng. AWS chịu trách nhiệm:

- Cấp phát và thu hồi server (provisioning/deprovisioning)
- Vá lỗi hệ điều hành và runtime (OS/runtime patching)
- Đảm bảo tính sẵn sàng cao (high availability)
- Co giãn tự động (auto-scaling)

**Bạn chỉ cần:** Viết code và cấu hình trigger (sự kiện kích hoạt).

### Mô Hình Serverless vs Truyền Thống

```
Truyền thống (EC2):
┌─────────────────────────────────────────┐
│  Bạn phải lo: OS → Runtime → App → Scale │
│  Chi phí: 24/7 dù không có traffic       │
└─────────────────────────────────────────┘

Serverless (Lambda):
┌─────────────────────────────────────────┐
│  Bạn chỉ lo: Code                        │
│  AWS lo: Mọi thứ còn lại                 │
│  Chi phí: Chỉ khi code đang chạy         │
└─────────────────────────────────────────┘
```

---

## ⚙️ Lambda Hoạt Động Thế Nào?

### Vòng Đời Của Một Lần Gọi Lambda (Invocation Lifecycle)

```
Event Source            Lambda Service          Your Code
(Nguồn Sự Kiện)        (Dịch Vụ Lambda)        (Code Của Bạn)
      │                        │                      │
      │── Event ──────────────>│                      │
      │                        │── Init (Khởi tạo) ──>│
      │                        │   (Cold Start nếu mới)│
      │                        │                      │── handler()
      │                        │<── Response ─────────│
      │<── Response ───────────│                      │
      │                        │                      │
      │── Next Event ─────────>│                      │
      │                        │<── Response ─────────│ (Warm Start)
      │<── Response ───────────│                      │
```

### Execution Environment (Môi Trường Thực Thi)

Mỗi Lambda function chạy trong một **Execution Environment** bao gồm:

- **MicroVM** — máy ảo siêu nhỏ dựa trên Firecracker
- **Runtime** — Node.js, Python, Java, Go, .NET, Ruby, hoặc custom
- **Handler** — hàm entry point trong code của bạn
- **/tmp storage** — 512 MB - 10 GB lưu trữ tạm thời
- **Environment Variables** — biến môi trường

### Cold Start vs Warm Start (Khởi Động Lạnh vs Khởi Động Nóng)

```
Cold Start (Khởi Động Lạnh):
[Download Code] → [Init Runtime] → [Init Handler] → [Execute]
    ~50-500ms           ~100ms          ~100ms         actual time

Warm Start (Khởi Động Nóng):
[Reuse Environment] → [Execute]
    ~1ms                actual time
```

---

## 🎯 Khi Nào Dùng Lambda?

### ✅ Phù Hợp Với Lambda

| Use Case                              | Lý Do                                          |
| ------------------------------------- | ---------------------------------------------- |
| **API backend** cho mobile/web        | Tự động scale, không cần quản lý server         |
| **Event processing** (xử lý sự kiện) | S3 upload, SQS message, DynamoDB change        |
| **Scheduled tasks** (tác vụ định kỳ) | Cron jobs, data cleanup, report generation     |
| **Data transformation** (biến đổi dữ liệu) | ETL pipeline, image resize, format conversion |
| **Webhooks** (điểm nhận sự kiện web) | GitHub webhooks, Stripe payments, Slack bots   |
| **Real-time file processing**         | Video thumbnail, PDF generation                |

### ❌ Không Phù Hợp Với Lambda

| Use Case                              | Lý Do Không Phù Hợp                            |
| ------------------------------------- | ----------------------------------------------- |
| Long-running jobs > 15 phút           | Lambda timeout tối đa 15 phút                   |
| Workload cần GPU                      | Lambda không hỗ trợ GPU                         |
| Stateful applications                 | Lambda là stateless (không lưu trạng thái)      |
| WebSocket connections dài hạn        | Kết nối bị đóng sau mỗi invocation              |
| Workload cần > 10 GB RAM              | Lambda tối đa 10 GB RAM                         |
| Persistent background processes       | Lambda không chạy liên tục                      |

---

## 🏗️ Kiến Trúc Event-Driven (Hướng Sự Kiện)

Lambda là trung tâm của kiến trúc **Event-Driven Architecture** (Kiến Trúc Hướng Sự Kiện) trên AWS:

```
                    ┌──────────────────────────────────────────────┐
                    │           AWS Event Sources                   │
                    │  (Nguồn Sự Kiện AWS)                         │
                    └──┬───────┬───────┬───────┬───────┬───────────┘
                       │       │       │       │       │
                     S3    API GW    SQS    SNS   DynamoDB
                     │       │       │       │   Streams
                     └───────┴───────┴───────┴───┘
                                     │
                                     ▼
                    ┌────────────────────────────────┐
                    │       AWS Lambda Function       │
                    │    (Hàm AWS Lambda)             │
                    │  ┌──────────────────────────┐  │
                    │  │  handler(event, context) │  │
                    │  │    # Your business logic │  │
                    │  │    return response       │  │
                    │  └──────────────────────────┘  │
                    └────────────────────────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
                    ▼                ▼                ▼
                DynamoDB          SQS/SNS          Other AWS
                RDS               EventBridge      Services
```

### Patterns Phổ Biến (Mẫu Kiến Trúc Thường Dùng)

#### 1. Synchronous Pattern — Đồng Bộ (API Backend)

```
Client → API Gateway → Lambda → DynamoDB/RDS → Response → Client
Đặc điểm: Client chờ response, phù hợp cho CRUD APIs
```

#### 2. Asynchronous Pattern — Bất Đồng Bộ (Event Processing)

```
Producer → SQS Queue → Lambda (polling) → Process & Store
Đặc điểm: Không chờ response, xử lý hàng loạt, fault-tolerant
```

#### 3. Fan-out Pattern — Phân Tán

```
S3 Event → Lambda → SNS Topic → [Lambda A, Lambda B, Lambda C]
Đặc điểm: Một event kích hoạt nhiều xử lý song song
```

#### 4. Choreography Pattern — Phối Hợp Sự Kiện

```
Lambda A → EventBridge → Lambda B → SQS → Lambda C
Đặc điểm: Microservices giao tiếp qua sự kiện, loose coupling
```

---

## 📖 Các Khái Niệm Cốt Lõi

### Function (Hàm)

Đơn vị cơ bản của Lambda — một đoạn code với cấu hình cụ thể:

- **Runtime** — môi trường chạy code (Python 3.12, Node.js 20, Java 21...)
- **Handler** — điểm vào của function (ví dụ: `index.handler`)
- **Memory** — RAM từ 128 MB đến 10,240 MB (CPU tăng theo memory)
- **Timeout** — thời gian tối đa từ 1 giây đến 900 giây (15 phút)
- **Execution Role** — IAM Role cấp quyền cho Lambda

### Event (Sự Kiện)

JSON object được truyền vào handler khi Lambda được gọi. Cấu trúc thay đổi tùy event source.

### Context (Ngữ Cảnh)

Object chứa thông tin về invocation: request ID, thời gian còn lại, log group name...

### Concurrency (Đồng Thời)

Số lượng function instances đang chạy song song tại một thời điểm:

```
Concurrency = (Requests/giây) × (Thời gian xử lý trung bình/giây)
Ví dụ: 1000 req/s × 0.1s = 100 concurrent executions
```

### Invocation Types (Loại Gọi Hàm)

| Type               | Mô Tả                                  | Ví Dụ                    |
| ------------------ | --------------------------------------- | ------------------------ |
| **Synchronous**    | Gọi và chờ response                     | API Gateway, ALB         |
| **Asynchronous**   | Gọi và không chờ (Lambda retry tự động) | S3, SNS, EventBridge     |
| **Poll-based**     | Lambda tự polling từ stream/queue        | SQS, Kinesis, DynamoDB   |

---

## 📁 Tổng Quan Các File Trong Topic

| File                              | Nội Dung                                              | Độ Khó |
| --------------------------------- | ----------------------------------------------------- | ------ |
| `README.md` (file này)            | Tổng quan Lambda & kiến trúc Serverless               | ⭐     |
| `1-lambda-fundamentals.md`        | Function anatomy, runtime, handler, execution model   | ⭐⭐   |
| `2-event-sources.md`              | API Gateway, SQS, SNS, S3, DynamoDB Streams           | ⭐⭐   |
| `3-layers-extensions.md`          | Lambda Layers, Extensions, Container images           | ⭐⭐⭐ |
| `4-concurrency-throttling.md`     | Reserved/Provisioned Concurrency, throttling          | ⭐⭐⭐ |
| `5-performance-best-practices.md` | Cold start, SnapStart, memory tuning, VPC Lambda      | ⭐⭐⭐ |

---

## ⚖️ Lambda vs Các Dịch Vụ Khác

### Lambda vs EC2

| Tiêu Chí              | Lambda                          | EC2                              |
| --------------------- | ------------------------------- | -------------------------------- |
| Quản lý server        | AWS hoàn toàn                   | Bạn quản lý OS, runtime, patches |
| Scaling               | Tự động, từ 0 đến ∞             | Cần cấu hình Auto Scaling        |
| Chi phí               | Pay-per-invocation              | Pay-per-hour (dù không dùng)     |
| Thời gian chạy tối đa | 15 phút                         | Không giới hạn                   |
| State                 | Stateless (không lưu trạng thái)| Stateful (có thể lưu local state)|
| Kiểm soát             | Hạn chế (chỉ config function)   | Toàn quyền                       |

### Lambda vs ECS Fargate

| Tiêu Chí              | Lambda                          | ECS Fargate                      |
| --------------------- | ------------------------------- | -------------------------------- |
| Packaging             | ZIP hoặc Container (≤10 GB)     | Docker Container (bất kỳ size)   |
| Startup time          | Milliseconds (warm)             | Seconds (container startup)      |
| Thời gian chạy        | Tối đa 15 phút                  | Không giới hạn                   |
| Networking            | Mặc định không trong VPC        | Luôn trong VPC                   |
| Chi phí               | Pay per 1ms                     | Pay per task running time        |
| Phù hợp cho           | Short, event-driven tasks       | Long-running, containerized apps |

### Khi Nào Chọn Cái Nào?

```
Câu hỏi quyết định:
1. Chạy < 15 phút? → Lambda là ứng viên
2. Event-driven hoặc API? → Lambda tốt hơn
3. Cần Kubernetes ecosystem? → EKS
4. Container nhưng không muốn manage server? → ECS Fargate
5. Cần full control, persistent process? → EC2
```

---

## 🔢 Pricing (Định Giá) Lambda

Lambda tính phí theo 2 chiều:

### 1. Number of Requests (Số Lượng Yêu Cầu)

- **Free tier:** 1 triệu requests/tháng
- **Sau free tier:** $0.20 per 1 triệu requests

### 2. Duration (Thời Gian Thực Thi)

Tính bằng **GB-seconds** (RAM × thời gian):

- **Free tier:** 400,000 GB-seconds/tháng
- **Sau free tier:** $0.0000166667 per GB-second

**Ví dụ tính chi phí:**

```
Function: 512 MB RAM, chạy 200ms, 10 triệu lần/tháng

Requests cost:
  10M requests × $0.20/1M = $2.00

Duration cost:
  10M × 0.2s × 0.5 GB = 1,000,000 GB-seconds
  1,000,000 × $0.0000166667 = $16.67

Total: ~$18.67/tháng
(So sánh: EC2 t3.micro 24/7 = ~$8.50/tháng nhưng không scale)
```

---

## 🏗️ Kiến Trúc Serverless Phổ Biến

### API Backend Serverless (Tiêu Chuẩn)

```
Mobile/Web App
      │
      ▼
API Gateway (REST/HTTP API)
      │
      ▼
Lambda Function(s)
      │
   ┌──┴──────────────┐
   │                 │
   ▼                 ▼
DynamoDB          S3 Bucket
(Dữ liệu)         (Files)
```

### Event Processing Pipeline (Luồng Xử Lý Sự Kiện)

```
Data Source → S3 → Lambda (transform) → DynamoDB
                         │
                         └→ SQS → Lambda (notify) → SNS → Email/SMS
```

### Microservices Serverless (Vi Dịch Vụ Không Máy Chủ)

```
API GW → Lambda (Auth)    → JWT Token
API GW → Lambda (User)    → DynamoDB (users)
API GW → Lambda (Order)   → DynamoDB (orders) → SQS
                                                  │
                                            Lambda (Fulfill) → External API
```

---

## 🎓 Câu Hỏi Phỏng Vấn Nhanh

**Q: Lambda là stateless hay stateful? Tại sao?**
> Lambda là stateless. Mỗi invocation có thể chạy trên một execution environment mới. Để lưu state, phải dùng external storage như DynamoDB, ElastiCache, hoặc S3.

**Q: Cold start là gì và ảnh hưởng thế nào?**
> Cold start xảy ra khi Lambda cần khởi tạo execution environment mới — download code, init runtime, init handler. Thêm 100ms-1s latency. Dùng Provisioned Concurrency hoặc SnapStart để giảm thiểu.

**Q: Lambda timeout tối đa là bao nhiêu?**
> 900 giây (15 phút). Workload dài hơn cần dùng ECS, EC2, hoặc AWS Batch.

**Q: Lambda scale thế nào?**
> Lambda tự động scale bằng cách tăng số concurrent executions. Default account limit là 1,000 concurrent (có thể tăng lên). Mỗi AZ có burst limit riêng.

**Q: Khi nào dùng Reserved Concurrency vs Provisioned Concurrency?**
> Reserved Concurrency: Giới hạn tối đa concurrent để bảo vệ downstream services. Provisioned Concurrency: Pre-warm execution environments để tránh cold start — trả phí ngay cả khi không có request.

---

## 🗺️ Điều Hướng Topic

```
Bắt đầu đây → README.md (file này)
     │
     ├─ 1-lambda-fundamentals.md   ← Học trước: cấu trúc function
     ├─ 2-event-sources.md         ← Tiếp theo: trigger & integration
     ├─ 3-layers-extensions.md     ← Sau đó: packaging & extensions
     ├─ 4-concurrency-throttling.md ← Quan trọng: concurrency model
     └─ 5-performance-best-practices.md ← Cuối: tối ưu hiệu năng
```

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
