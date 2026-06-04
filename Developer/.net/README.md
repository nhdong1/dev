# 🔷 C# .NET Developer — Lộ Trình Kiến Thức

> Hướng dẫn toàn diện về lập trình C# và hệ sinh thái .NET, từ nền tảng cơ bản đến kiến trúc nâng cao và phỏng vấn thực chiến.

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Các Chủ Đề](#tổng-quan-các-chủ-đề)
4. [Ma Trận Kỹ Năng](#ma-trận-kỹ-năng)
5. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🎯 Lộ Trình Học

### **Giai Đoạn 1: Nền Tảng C# (Tuần 1–2)**

- [ ] C# Language Basics — Cú pháp cơ bản C#, kiểu dữ liệu, collections
- [ ] OOP — Object-Oriented Programming — Lập trình hướng đối tượng
- [ ] CLR — Common Language Runtime — Môi trường chạy ngôn ngữ chung
- [ ] Memory Model — Mô hình bộ nhớ: Stack vs Heap, Value vs Reference types
- [ ] LINQ — Language Integrated Query — Truy vấn tích hợp ngôn ngữ

### **Giai Đoạn 2: Kỹ Năng Lập Trình Trung Cấp (Tuần 3–6)**

- [ ] SOLID Principles — Năm nguyên lý thiết kế phần mềm
- [ ] Design Patterns — Mẫu thiết kế (GoF: Gang of Four)
- [ ] Async/Await & Task — Lập trình bất đồng bộ
- [ ] Dependency Injection — DI — Tiêm phụ thuộc
- [ ] ASP.NET Core — Framework web hiện đại của Microsoft
- [ ] Entity Framework Core — ORM — Object-Relational Mapper

### **Giai Đoạn 3: Kỹ Năng Nâng Cao (Tuần 7–10)**

- [ ] Performance & Memory Optimization — Tối ưu hiệu năng và bộ nhớ
- [ ] Testing Strategy — Chiến lược kiểm thử (Unit, Integration, E2E)
- [ ] Security Best Practices — Bảo mật: Authentication, Authorization
- [ ] Clean Architecture — Kiến trúc sạch, phân tầng rõ ràng

### **Giai Đoạn 4: Chuyên Sâu & Thực Chiến (Tuần 11+)**

- [ ] Microservices Architecture — Kiến trúc vi dịch vụ
- [ ] DDD — Domain-Driven Design — Thiết kế hướng miền
- [ ] CQRS — Command Query Responsibility Segregation — Phân tách trách nhiệm đọc/ghi
- [ ] Cloud & DevOps — Azure, Docker, Kubernetes, CI/CD

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực                            | Ưu Tiên | Thời Gian | Trạng Thái |
| ----------------------------------- | ------- | --------- | ---------- |
| **C# Language & OOP**               | ⭐⭐⭐  | 2 tuần    | -          |
| **ASP.NET Core & Middleware**        | ⭐⭐⭐  | 2 tuần    | -          |
| **Async/Await & Concurrency**        | ⭐⭐⭐  | 1 tuần    | -          |
| **Entity Framework Core**            | ⭐⭐⭐  | 2 tuần    | -          |
| **Design Patterns & SOLID**          | ⭐⭐⭐  | 2 tuần    | -          |
| **Testing (Unit & Integration)**     | ⭐⭐⭐  | 1 tuần    | -          |
| **Performance & GC Tuning**          | ⭐⭐    | 1 tuần    | -          |
| **Security & Authentication**        | ⭐⭐⭐  | 1 tuần    | -          |
| **Clean Architecture & DDD**         | ⭐⭐    | 2 tuần    | -          |
| **Cloud & Deployment**               | ⭐⭐    | 2 tuần    | -          |

---

## 🗂️ Tổng Quan Các Chủ Đề

### 📁 **1. Nền Tảng C#** (`01-fundamentals/`)

- C# Type System — Hệ thống kiểu: Value types, Reference types, Nullable
- CLR — Common Language Runtime — Môi trường chạy và quản lý mã
- GC — Garbage Collector — Bộ thu gom rác: thế hệ 0, 1, 2
- Collections — Tập hợp: `List<T>`, `Dictionary<K,V>`, `IEnumerable<T>`
- LINQ — Truy vấn dữ liệu tích hợp ngôn ngữ
- Generics — Kiểu tổng quát
- Delegates & Events — Ủy quyền và sự kiện
- Exception Handling — Xử lý ngoại lệ

### 📁 **2. OOP & Design Patterns** (`02-oop-patterns/`)

- **SOLID Principles** — Năm nguyên lý thiết kế:
  - SRP — Single Responsibility — Một trách nhiệm duy nhất
  - OCP — Open/Closed — Mở rộng nhưng không sửa đổi
  - LSP — Liskov Substitution — Thay thế Liskov
  - ISP — Interface Segregation — Tách biệt giao diện
  - DIP — Dependency Inversion — Đảo ngược phụ thuộc
- Creational Patterns — Mẫu khởi tạo: Singleton, Factory, Builder
- Structural Patterns — Mẫu cấu trúc: Adapter, Decorator, Proxy
- Behavioral Patterns — Mẫu hành vi: Observer, Strategy, Mediator

### 📁 **3. Async/Await & Concurrency** (`03-async-concurrency/`)

- `async`/`await` — Cú pháp lập trình bất đồng bộ
- Task Parallel Library — TPL — Thư viện tác vụ song song
- `CancellationToken` — Token hủy tác vụ
- Thread vs Task — Luồng vs Tác vụ
- Deadlock Prevention — Phòng tránh bế tắc
- `Channel<T>` & Producer-Consumer Pattern
- `SemaphoreSlim` & Throttling — Giới hạn đồng thời

### 📁 **4. ASP.NET Core** (`04-aspnet-core/`)

- Middleware Pipeline — Chuỗi xử lý yêu cầu
- Dependency Injection — DI Container tích hợp sẵn
- Routing — Định tuyến URL
- Minimal APIs vs Controllers — Hai phong cách xây dựng API
- Model Binding & Validation — Ràng buộc và xác thực dữ liệu
- Filters — Action Filter, Exception Filter, Authorization Filter
- Configuration & Options Pattern — Quản lý cấu hình ứng dụng
- Health Checks — Kiểm tra tình trạng dịch vụ

### 📁 **5. Entity Framework Core** (`05-entity-framework/`)

- DbContext & DbSet — Ngữ cảnh và tập thực thể
- Code-First Migrations — Tạo schema từ code
- LINQ Queries — Truy vấn dữ liệu với LINQ
- Relationships — Quan hệ: 1-1, 1-N, N-N
- Change Tracking — Theo dõi thay đổi
- N+1 Query Problem — Vấn đề N+1 truy vấn và cách tránh
- Compiled Queries — Truy vấn đã biên dịch để tối ưu
- Raw SQL & Stored Procedures — SQL thô và thủ tục lưu trữ

### 📁 **6. Testing** (`06-testing/`)

- Unit Testing — Kiểm thử đơn vị với xUnit/NUnit/MSTest
- Mocking — Giả lập với Moq/NSubstitute
- Integration Testing — Kiểm thử tích hợp với `WebApplicationFactory`
- TDD — Test-Driven Development — Phát triển hướng kiểm thử
- Test Coverage — Độ bao phủ kiểm thử
- Snapshot Testing — Kiểm thử ảnh chụp nhanh
- Performance Testing — Kiểm thử hiệu năng với BenchmarkDotNet

### 📁 **7. Hiệu Năng & Bộ Nhớ** (`07-performance/`)

- Memory Profiling — Phân tích bộ nhớ: heap dump, memory leak
- `Span<T>` & `Memory<T>` — Slice bộ nhớ không cấp phát
- `ArrayPool<T>` & `MemoryPool<T>` — Tái sử dụng bộ nhớ
- `record` & `struct` — Lựa chọn kiểu phù hợp
- BenchmarkDotNet — Đo lường hiệu năng chính xác
- GC Pressure Reduction — Giảm áp lực thu gom rác
- Caching Strategies — Chiến lược cache: In-Memory, Distributed, Redis

### 📁 **8. Bảo Mật** (`08-security/`)

- Authentication — Xác thực: JWT, Cookie, OAuth2, OpenID Connect
- Authorization — Phân quyền: Policy, Role-based, Claims-based
- ASP.NET Core Identity — Hệ thống quản lý người dùng
- Data Protection API — Bảo vệ dữ liệu nhạy cảm
- HTTPS & TLS — Truyền tải an toàn
- CORS — Cross-Origin Resource Sharing — Chia sẻ tài nguyên chéo nguồn gốc
- Input Validation & SQL Injection Prevention — Phòng chống tấn công
- Secrets Management — Quản lý bí mật với Azure Key Vault / AWS Secrets Manager

### 📁 **9. Kiến Trúc Phần Mềm** (`09-architecture/`)

- Clean Architecture — Kiến trúc sạch: Domain, Application, Infrastructure, Presentation
- DDD — Domain-Driven Design — Aggregate, Entity, Value Object, Repository
- CQRS — Command Query Responsibility Segregation — Tách lệnh và truy vấn
- Event Sourcing — Nguồn sự kiện: lưu trạng thái qua chuỗi sự kiện
- Microservices — Vi dịch vụ: giao tiếp, service discovery, API Gateway
- Messaging — Nhắn tin bất đồng bộ: MassTransit, RabbitMQ, Azure Service Bus
- Outbox Pattern — Đảm bảo gửi message khi lưu DB

### 📁 **10. Cloud & Triển Khai** (`10-cloud-deployment/`)

- Docker — Đóng gói ứng dụng vào container
- Kubernetes — K8s — Điều phối container
- Azure App Service / AKS — Triển khai trên Azure
- CI/CD — GitHub Actions, Azure DevOps — Tích hợp và triển khai liên tục
- Configuration Management — Quản lý cấu hình theo môi trường
- Health Probes — Liveness & Readiness trong Kubernetes
- Observability — Logging, Metrics, Tracing với OpenTelemetry

### 📁 **11. Chuẩn Bị Phỏng Vấn** (`11-interview-prep/`)

- Top 30 câu hỏi phỏng vấn C# .NET
- System Design Scenarios — Kịch bản thiết kế hệ thống
- Câu chuyện thực chiến theo phương pháp STAR
- Code Challenge — Bài tập lập trình thực tế
- Behavioral Questions — Câu hỏi về hành vi và kinh nghiệm

---

## 🏆 Theo Vị Trí Công Việc

### **Junior .NET Developer**

```
Tập trung: C# Basics, OOP, ASP.NET Core CRUD, EF Core căn bản
Thời gian học: 2-3 tháng
Phủ: 01-fundamentals, 04-aspnet-core, 05-entity-framework
```

### **Mid-Level .NET Developer**

```
Tập trung: Design Patterns, Async, Testing, Clean Architecture, Security
Thời gian học: 2-3 tháng để nâng cấp
Phủ: 02-oop-patterns, 03-async-concurrency, 06-testing, 08-security, 09-architecture
```

### **Senior .NET Developer / Tech Lead**

```
Tập trung: Microservices, DDD, CQRS, Performance, Cloud, System Design
Thời gian học: Học liên tục
Phủ: Tất cả sections, đặc biệt 07-performance, 09-architecture, 10-cloud-deployment
```

---

## 📊 Ma Trận Kỹ Năng

### Beginner — Người Mới Bắt Đầu (0–1 năm)

- [ ] Cú pháp C# cơ bản: biến, vòng lặp, hàm
- [ ] OOP: class, inheritance, interface, polymorphism
- [ ] LINQ cơ bản: `Where`, `Select`, `OrderBy`, `FirstOrDefault`
- [ ] ASP.NET Core: tạo API CRUD đơn giản
- [ ] EF Core: code-first, migration cơ bản
- [ ] Xử lý ngoại lệ: try/catch/finally

### Intermediate — Trung Cấp (1–3 năm)

- [ ] SOLID Principles và Design Patterns thông dụng
- [ ] Dependency Injection: đăng ký và resolve service
- [ ] Async/Await: tránh deadlock, dùng `ConfigureAwait`
- [ ] Unit Testing với Moq và xUnit
- [ ] JWT Authentication và Role-based Authorization
- [ ] EF Core: query optimization, N+1 problem
- [ ] Configuration, Middleware pipeline, Filters

### Advanced — Nâng Cao (3–5+ năm)

- [ ] Clean Architecture toàn phần
- [ ] DDD: Aggregate, Domain Events, Value Objects
- [ ] CQRS + MediatR + Event Sourcing
- [ ] `Span<T>`, `Memory<T>`, `ArrayPool<T>` để tối ưu bộ nhớ
- [ ] GC Tuning — điều chỉnh thu gom rác
- [ ] Microservices với message broker
- [ ] Distributed Tracing — Theo dõi phân tán với OpenTelemetry
- [ ] Kubernetes deployment và health probes

---

## 🚀 Bắt Đầu Nhanh

### Bước 1: Xác Định Mục Tiêu

```
Chọn hướng đi của bạn:
- Backend Developer (API/Web)    → 04-aspnet-core + 05-entity-framework
- Architect / Tech Lead          → 09-architecture + 07-performance
- Cloud/DevOps focused           → 10-cloud-deployment + 09-architecture
- Phỏng vấn gấp                  → 11-interview-prep + 02-oop-patterns
```

### Bước 2: Cài Đặt Môi Trường

```bash
# Cài .NET SDK (https://dotnet.microsoft.com)
dotnet --version

# Tạo project mới để thực hành
dotnet new webapi -n PracticeApi
dotnet new xunit -n PracticeApi.Tests

# Cài extensions VS Code: C# Dev Kit, REST Client
```

### Bước 3: Học Theo Chu Trình

```
1. Đọc tài liệu lý thuyết (30 phút)
2. Xem ví dụ code trong file .md (15 phút)
3. Tự viết code theo ví dụ (30-60 phút)
4. Thêm unit test để xác minh hiểu đúng (15 phút)
5. Review checklist cuối bài (10 phút)
```

### Bước 4: Chuẩn Bị Câu Chuyện Phỏng Vấn

```
Với mỗi chủ đề, chuẩn bị câu chuyện theo STAR:
- Situation  — Tình huống: bối cảnh, vấn đề gặp phải
- Task       — Nhiệm vụ: bạn chịu trách nhiệm điều gì
- Action     — Hành động: bạn đã làm gì cụ thể
- Result     — Kết quả: outcome đo lường được
```

---

## 🔗 Liên Kết Nhanh

| Chủ Đề                         | Thư Mục                                                                             | Ưu Tiên         |
| ------------------------------ | ----------------------------------------------------------------------------------- | --------------- |
| Lộ trình học đầy đủ            | [INDEX.md](./INDEX.md)                                                              | Bắt đầu ở đây  |
| Câu hỏi phỏng vấn              | [11-interview-prep/](./11-interview-prep/)                                          | Trước phỏng vấn |
| SOLID & Design Patterns        | [02-oop-patterns/](./02-oop-patterns/)                                              | Thiết yếu       |
| Async/Await thực chiến         | [03-async-concurrency/](./03-async-concurrency/)                                    | Thiết yếu       |
| Clean Architecture             | [09-architecture/](./09-architecture/)                                              | Nâng cao        |
| Tối ưu hiệu năng               | [07-performance/](./07-performance/)                                                | Senior level    |

---

## 📖 Tài Liệu Tham Khảo

### Sách Thiết Yếu

- **"C# in Depth"** — Jon Skeet — Kiến thức C# chuyên sâu nhất
- **"Pro ASP.NET Core"** — Adam Freeman — ASP.NET Core toàn diện
- **"Designing Data-Intensive Applications"** — Martin Kleppmann — Hệ thống phân tán
- **"Domain-Driven Design"** — Eric Evans — DDD kinh điển
- **"Clean Architecture"** — Robert C. Martin — Kiến trúc sạch

### Tài Liệu Chính Thức

- [Microsoft Docs — C# Documentation](https://learn.microsoft.com/en-us/dotnet/csharp/)
- [ASP.NET Core Documentation](https://learn.microsoft.com/en-us/aspnet/core/)
- [Entity Framework Core Docs](https://learn.microsoft.com/en-us/ef/core/)
- [.NET Performance Tips](https://learn.microsoft.com/en-us/dotnet/core/performance/)

### Blog & Kênh Học

- [Andrew Lock — .NET Escapades](https://andrewlock.net/)
- [Nick Chapsas — YouTube](https://www.youtube.com/@nickchapsas)
- [Milan Jovanović — Blog](https://www.milanjovanovic.tech/)
- [Khalid Abuhakmeh — Blog](https://khalidabuhakmeh.com/)

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Thường Gặp Theo Nhóm

#### C# & CLR

- [ ] Sự khác nhau giữa `value type` và `reference type`
- [ ] GC — Garbage Collector — hoạt động như thế nào, các thế hệ (Gen 0, 1, 2)
- [ ] `async`/`await` và `Task` — luồng thực thi ra sao
- [ ] `IDisposable` và `using` — quản lý tài nguyên không được quản lý
- [ ] `ref`, `out`, `in` — các từ khóa truyền tham chiếu

#### ASP.NET Core

- [ ] Middleware pipeline hoạt động như thế nào
- [ ] Dependency Injection: Singleton vs Scoped vs Transient
- [ ] JWT Authentication — xác thực JSON Web Token
- [ ] Minimal APIs vs Controller-based APIs — khi nào dùng cái nào

#### Entity Framework Core

- [ ] Code-First vs Database-First
- [ ] Vấn đề N+1 Query là gì và cách phòng tránh
- [ ] `AsNoTracking()` — khi nào nên dùng
- [ ] Migration conflict — xung đột migration trong team

#### Kiến Trúc

- [ ] Giải thích SOLID với ví dụ cụ thể
- [ ] Clean Architecture vs Layered Architecture — khi nào chọn cái nào
- [ ] CQRS — lợi ích và đánh đổi
- [ ] Microservices vs Monolith — khi nào nên chuyển đổi

#### Thực Chiến

- [ ] Kể về lần bạn fix performance issue (STAR)
- [ ] Cách bạn thiết kế authentication trong ứng dụng thực
- [ ] Bạn đã dùng Design Pattern nào, trong tình huống nào

Xem `11-interview-prep/` để có bộ Q&A đầy đủ.

---

## ✅ Checklist Tự Đánh Giá

Trước khi đi phỏng vấn hoặc nhận dự án mới, hãy xác nhận:

- [ ] Giải thích được `async`/`await` mà không cần ghi chú
- [ ] Thiết kế được API RESTful với ASP.NET Core
- [ ] Cài đặt được JWT Authentication từ đầu
- [ ] Viết được unit test với Moq/xUnit
- [ ] Xác định và fix được vấn đề N+1 trong EF Core
- [ ] Áp dụng được SOLID vào code thực tế
- [ ] Giải thích được sự khác biệt Scoped/Singleton/Transient DI
- [ ] Tối ưu được memory usage với `Span<T>` hoặc `ArrayPool<T>`
- [ ] Thiết kế được hệ thống theo Clean Architecture
- [ ] Trả lời được câu hỏi về trade-off giữa các pattern

---

## 🗺️ Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Xem INDEX.md để nắm cấu trúc toàn bộ tài liệu
├─ 3️⃣  Chọn learning path phù hợp vị trí mục tiêu
├─ 4️⃣  Bắt đầu từ 01-fundamentals/ nếu là người mới
├─ 5️⃣  Hoàn thành bài tập thực hành từng section
├─ 6️⃣  Xây dựng một project portfolio áp dụng kiến thức
└─ 7️⃣  Luyện phỏng vấn với 11-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
**Người Duy Trì:** Backend Interview Prep
