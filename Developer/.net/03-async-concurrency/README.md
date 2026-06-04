# Async/Await & Concurrency — Lập Trình Bất Đồng Bộ và Đồng Thời

> Async/Await — Lập trình bất đồng bộ — và Concurrency — Lập trình đồng thời — là hai trong số các chủ đề quan trọng nhất trong .NET. Hiểu sâu những khái niệm này giúp bạn viết ứng dụng phản hồi nhanh, không bị block, xử lý nhiều tác vụ hiệu quả.

---

## 📋 Tổng Quan Nhanh

| Khái Niệm | Ý Nghĩa | Khi Nào Dùng |
| --------- | ------- | ------------ |
| `async`/`await` | Bất đồng bộ — không block thread chờ I/O | Mọi I/O: HTTP, DB, file |
| `Task` | Đơn vị công việc bất đồng bộ | Đại diện cho một tác vụ |
| `Thread` — Luồng | Đơn vị thực thi của OS | Khi cần CPU-bound thực sự |
| `Task.WhenAll` | Chạy nhiều Task song song | Fan-out: nhiều request cùng lúc |
| `CancellationToken` | Token hủy tác vụ | Timeout, user cancel |
| `lock` / `Monitor` | Khóa độc quyền — mutual exclusion | Bảo vệ shared state |
| `Channel<T>` | Hàng đợi bất đồng bộ | Producer-Consumer pattern |
| `SemaphoreSlim` | Giới hạn số lượng đồng thời — throttling | Rate limiting, connection pool |

---

## 📁 Cấu Trúc Files

| File | Chủ Đề | Độ Khó |
| ---- | ------- | ------ |
| [1-async-await-deep-dive.md](1-async-await-deep-dive.md) | State machine, SynchronizationContext, ConfigureAwait | ⭐⭐⭐ |
| [2-task-parallel-library.md](2-task-parallel-library.md) | Task, Parallel.For, PLINQ, WhenAll/WhenAny | ⭐⭐ |
| [3-cancellation.md](3-cancellation.md) | CancellationToken, timeout, cooperative cancellation | ⭐⭐ |
| [4-thread-safety.md](4-thread-safety.md) | lock, Monitor, Interlocked, volatile, immutability | ⭐⭐⭐ |
| [5-channels-and-dataflow.md](5-channels-and-dataflow.md) | Channel\<T\>, producer-consumer pattern | ⭐⭐ |
| [6-deadlock-prevention.md](6-deadlock-prevention.md) | Deadlock — bế tắc: nguyên nhân, phát hiện, phòng tránh | ⭐⭐⭐ |

---

## 🎯 Khái Niệm Cốt Lõi Cần Nắm

### Sự Khác Biệt: Concurrency vs Parallelism vs Asynchrony

```
Concurrency — Đồng thời:
  Nhiều tác vụ được xử lý trong cùng khoảng thời gian,
  không nhất thiết chạy cùng lúc về mặt vật lý.
  Ví dụ: Thread pool xoay vòng giữa các task.

Parallelism — Song Song:
  Nhiều tác vụ chạy thực sự cùng lúc trên nhiều CPU core.
  Ví dụ: Parallel.For trên mảng triệu phần tử.

Asynchrony — Bất Đồng Bộ:
  Tác vụ được bắt đầu và bạn không cần chờ nó xong —
  có thể làm việc khác trong lúc chờ.
  Ví dụ: await HttpClient.GetAsync(...) — thread không bị block.
```

### Tại Sao async/await Quan Trọng?

```
Synchronous (Đồng Bộ):
  Thread A → [==== chờ DB 200ms ====] → xử lý kết quả

  Thread A bị block 200ms. Với 1000 request đồng thời,
  cần 1000 threads → tốn RAM, context switch cao.

Asynchronous (Bất Đồng Bộ):
  Thread A → [gửi yêu cầu DB] → [làm việc khác]
  [DB xong] → Thread B (từ pool) → xử lý kết quả

  Thread không bị block. Với 1000 request,
  chỉ cần vài chục threads từ pool.
```

---

## ⚡ Quy Tắc Vàng Async/Await

### 1. Async All the Way — Bất Đồng Bộ Từ Đầu Đến Cuối

```csharp
// ❌ SAI: Blocking trong async context — gây deadlock
public async Task<string> GetDataAsync()
{
    var result = SomeAsyncMethod().Result;  // BLOCK! Nguy hiểm!
    return result;
}

// ✅ ĐÚNG: await xuyên suốt
public async Task<string> GetDataAsync()
{
    var result = await SomeAsyncMethod();  // Không block
    return result;
}
```

### 2. ConfigureAwait(false) Trong Library Code

```csharp
// Trong thư viện (không phải UI/ASP.NET Controller):
public async Task<string> FetchDataAsync()
{
    var data = await httpClient.GetStringAsync(url)
        .ConfigureAwait(false);  // Không cần quay về SynchronizationContext gốc
    return data;
}
```

### 3. Không Dùng async void (Trừ Event Handler)

```csharp
// ❌ SAI: Exception sẽ crash process, không thể await
public async void ProcessData() { ... }

// ✅ ĐÚNG: Trả về Task để có thể await và catch exception
public async Task ProcessDataAsync() { ... }

// ✅ OK: Event handler — exception nơi nào handle?
button.Click += async (s, e) => { await DoSomethingAsync(); };
```

### 4. CancellationToken Luôn Được Truyền Qua

```csharp
// ✅ Pattern chuẩn: nhận CancellationToken và truyền xuống
public async Task ProcessOrderAsync(int orderId, CancellationToken ct = default)
{
    var order = await _repo.GetByIdAsync(orderId, ct);
    await _service.ValidateAsync(order, ct);
    await _repo.SaveAsync(order, ct);
}
```

---

## 🧵 Thread vs Task — Luồng vs Tác Vụ

| | `Thread` — Luồng | `Task` — Tác Vụ |
| -- | ---------------- | --------------- |
| **Tạo ra** | Tốn kém (~1MB stack) | Nhẹ, từ ThreadPool |
| **Quản lý** | Thủ công | .NET runtime quản lý |
| **Hủy** | Không thể hủy trực tiếp | `CancellationToken` |
| **Kết quả** | Không có (dùng shared state) | `Task<T>` trả về giá trị |
| **Exception** | Crash process nếu unhandled | Được gói trong `AggregateException` |
| **Dùng khi** | Cần kiểm soát hoàn toàn | Hầu hết mọi trường hợp |

---

## 📊 Khi Nào Dùng Gì

```
I/O-bound (Gọi API, đọc file, truy vấn DB):
  → Dùng async/await với Task
  → KHÔNG dùng Thread hay Parallel

CPU-bound (Tính toán nặng, xử lý ảnh):
  → Dùng Task.Run() để đẩy sang background thread
  → Cân nhắc Parallel.For / PLINQ cho data parallelism

Fan-out (Gọi nhiều API cùng lúc):
  → Task.WhenAll(task1, task2, task3)

Streaming dữ liệu liên tục:
  → Channel<T> với producer-consumer pattern
  → IAsyncEnumerable<T> cho async streams

Bảo vệ shared state:
  → lock / Monitor cho đoạn code ngắn
  → SemaphoreSlim cho async context
  → Interlocked cho counter đơn giản
  → Immutable objects — giải pháp không cần lock
```

---

## 🔗 Liên Kết Với Các Chủ Đề Khác

- **01-fundamentals/2-clr-and-memory.md** — ThreadPool, GC và thread interaction
- **04-aspnet-core/2-dependency-injection.md** — Scoped services trong async context (Scoped Captive Dependency)
- **07-performance/5-caching.md** — Cache với async patterns
- **09-architecture/6-messaging-patterns.md** — Message queue và async processing

---

## 🚀 Learning Path Cho Topic Này

```
Bước 1: Đọc 1-async-await-deep-dive.md
         → Hiểu sâu cách async/await hoạt động bên dưới

Bước 2: Đọc 2-task-parallel-library.md
         → Nắm các API của TPL: Task, Parallel, PLINQ

Bước 3: Đọc 3-cancellation.md
         → Thực hành CancellationToken trong mọi async method

Bước 4: Đọc 4-thread-safety.md
         → Hiểu các primitive để bảo vệ shared state

Bước 5: Đọc 5-channels-and-dataflow.md
         → Xây dựng pipeline producer-consumer

Bước 6: Đọc 6-deadlock-prevention.md
         → Nhận biết và phòng tránh deadlock
```

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Thuộc:** Developer/.net/03-async-concurrency/
