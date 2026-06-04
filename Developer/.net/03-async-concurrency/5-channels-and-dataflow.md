# Channel\<T\> & Dataflow — Luồng Dữ Liệu Bất Đồng Bộ

> `Channel<T>` trong `System.Threading.Channels` là cơ chế giao tiếp bất đồng bộ giữa producer — người tạo dữ liệu — và consumer — người tiêu thụ dữ liệu. Channel là hàng đợi (queue) bất đồng bộ, thread-safe, hiệu năng cao, thay thế hiện đại cho `BlockingCollection<T>`.

---

## 📋 Tổng Quan

| Khái Niệm | Mô Tả |
| --------- | ------ |
| `Channel<T>` | Hàng đợi bất đồng bộ — thread-safe, async-friendly |
| `ChannelWriter<T>` | Producer ghi vào channel |
| `ChannelReader<T>` | Consumer đọc từ channel |
| `UnboundedChannel` | Channel không giới hạn kích thước |
| `BoundedChannel` | Channel có giới hạn — tạo backpressure |
| Backpressure | Cơ chế làm chậm producer khi consumer quá tải |
| Producer-Consumer | Mẫu thiết kế: tách biệt người tạo và người dùng dữ liệu |

---

## 🔷 Vấn Đề Producer-Consumer Pattern Giải Quyết

```
Vấn Đề:
  Producer (đọc DB, nhận message...)  → [???]  → Consumer (xử lý, gửi email...)
  
  Nếu producer nhanh hơn consumer → dữ liệu tích tụ (buffer overflow)
  Nếu consumer nhanh hơn producer → consumer idle, lãng phí tài nguyên
  Nếu gọi trực tiếp → coupling chặt, không thể scale riêng

Channel<T> giải quyết:
  Producer → [Channel<T>] → Consumer
  - Producer: WriteAsync — ghi không block, hoặc chờ nếu channel đầy (bounded)
  - Consumer: ReadAsync — đọc không block, chờ nếu channel rỗng
  - Decoupled — tách biệt hoàn toàn
  - Backpressure — áp ngược: làm chậm producer khi consumer quá tải
```

---

## 🔷 Tạo Channel

### UnboundedChannel — Channel Không Giới Hạn

```csharp
// Không giới hạn số items — producer không bao giờ chờ
var channel = Channel.CreateUnbounded<WorkItem>();

// Tùy chọn:
var channel = Channel.CreateUnbounded<WorkItem>(new UnboundedChannelOptions
{
    SingleWriter = true,  // Chỉ có một producer — tối ưu hóa nội bộ
    SingleReader = true,  // Chỉ có một consumer — tối ưu hóa nội bộ
    AllowSynchronousContinuations = false  // Không chạy continuation đồng bộ
});
```

### BoundedChannel — Channel Có Giới Hạn (Khuyến Nghị)

```csharp
// Giới hạn 100 items — tạo backpressure khi channel đầy
var channel = Channel.CreateBounded<WorkItem>(100);

// Tùy chọn đầy đủ:
var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait,         // Chờ khi đầy (backpressure)
    // FullMode = BoundedChannelFullMode.DropOldest, // Xóa item cũ nhất
    // FullMode = BoundedChannelFullMode.DropNewest, // Xóa item mới nhất
    // FullMode = BoundedChannelFullMode.DropWrite,  // Bỏ qua write khi đầy
    SingleWriter = false,
    SingleReader = false
});
```

---

## 🔷 Viết và Đọc Cơ Bản

```csharp
var channel = Channel.CreateBounded<int>(capacity: 10);
ChannelWriter<int> writer = channel.Writer;
ChannelReader<int> reader = channel.Reader;

// === PRODUCER ===
// WriteAsync: chờ nếu channel đầy (backpressure)
await writer.WriteAsync(42, cancellationToken);

// TryWrite: không chờ — trả false nếu đầy
if (!writer.TryWrite(42))
    _logger.LogWarning("Channel đầy, bỏ qua item");

// Báo hiệu không còn dữ liệu nữa
writer.Complete();                        // Hoàn thành bình thường
writer.Complete(new Exception("lỗi"));    // Hoàn thành với exception

// === CONSUMER ===
// ReadAsync: chờ nếu channel rỗng
var item = await reader.ReadAsync(cancellationToken);

// TryRead: không chờ — trả false nếu rỗng
if (reader.TryRead(out var value))
    Process(value);

// WaitToReadAsync: chờ đến khi có data (không lấy data)
while (await reader.WaitToReadAsync(cancellationToken))
{
    while (reader.TryRead(out var item))
        Process(item);  // Đọc hết trong lúc available
}

// ReadAllAsync: đọc tất cả items đến khi writer Complete
await foreach (var item in reader.ReadAllAsync(cancellationToken))
{
    await ProcessAsync(item, cancellationToken);
}
```

---

## 🔷 Pattern Chuẩn: Single Producer, Single Consumer

```csharp
public class EmailProcessingPipeline
{
    private readonly Channel<EmailMessage> _channel;
    private readonly IEmailService _emailService;

    public EmailProcessingPipeline(IEmailService emailService)
    {
        _emailService = emailService;
        _channel = Channel.CreateBounded<EmailMessage>(
            new BoundedChannelOptions(1000)
            {
                FullMode = BoundedChannelFullMode.Wait,
                SingleWriter = false,  // Nhiều thread có thể enqueue
                SingleReader = true    // Chỉ một consumer
            });
    }

    // Producer: có thể gọi từ nhiều thread
    public async ValueTask EnqueueAsync(EmailMessage email, CancellationToken ct)
    {
        await _channel.Writer.WriteAsync(email, ct);
    }

    // Consumer: chạy như background service
    public async Task ProcessAsync(CancellationToken ct)
    {
        await foreach (var email in _channel.Reader.ReadAllAsync(ct))
        {
            try
            {
                await _emailService.SendAsync(email, ct);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Gửi email thất bại: {Email}", email.To);
                // Quyết định: retry? dead letter queue?
            }
        }
    }

    // Shutdown: báo hiệu không còn email nào nữa
    public void Complete() => _channel.Writer.Complete();
}
```

---

## 🔷 Multiple Producers, Multiple Consumers

```csharp
public class WorkerPool
{
    private readonly Channel<WorkItem> _channel;
    private readonly int _workerCount;

    public WorkerPool(int workerCount = 4)
    {
        _workerCount = workerCount;
        _channel = Channel.CreateBounded<WorkItem>(workerCount * 10);
    }

    public async Task StartAsync(CancellationToken ct)
    {
        // Khởi động N consumer workers song song
        var workers = Enumerable.Range(0, _workerCount)
            .Select(i => RunWorkerAsync(i, ct));

        await Task.WhenAll(workers);
    }

    private async Task RunWorkerAsync(int workerId, CancellationToken ct)
    {
        await foreach (var item in _channel.Reader.ReadAllAsync(ct))
        {
            _logger.LogDebug("Worker {Id} xử lý item {ItemId}", workerId, item.Id);
            await item.ProcessAsync(ct);
        }
    }

    // Nhiều producer có thể gọi cùng lúc
    public async Task EnqueueAsync(WorkItem item, CancellationToken ct)
        => await _channel.Writer.WriteAsync(item, ct);

    public void Complete() => _channel.Writer.Complete();
}
```

---

## 🔷 Pipeline Pattern — Chuỗi Xử Lý

```csharp
// Pipeline: A → B → C, mỗi stage là producer+consumer
// Stage A: đọc files → Stage B: parse → Stage C: lưu DB

public class DataImportPipeline
{
    public async Task RunAsync(string folder, CancellationToken ct)
    {
        // Tạo channels giữa các stage
        var rawChannel   = Channel.CreateBounded<string>(100);
        var parsedChannel = Channel.CreateBounded<Record>(100);

        // Stage 1: Đọc file (producer cho rawChannel)
        var readerTask = Task.Run(async () =>
        {
            try
            {
                foreach (var file in Directory.GetFiles(folder, "*.csv"))
                {
                    var content = await File.ReadAllTextAsync(file, ct);
                    await rawChannel.Writer.WriteAsync(content, ct);
                }
            }
            finally
            {
                rawChannel.Writer.Complete();  // Báo xong đọc
            }
        }, ct);

        // Stage 2: Parse CSV (consumer rawChannel → producer parsedChannel)
        var parserTask = Task.Run(async () =>
        {
            try
            {
                await foreach (var raw in rawChannel.Reader.ReadAllAsync(ct))
                {
                    var records = CsvParser.Parse(raw);
                    foreach (var record in records)
                        await parsedChannel.Writer.WriteAsync(record, ct);
                }
            }
            finally
            {
                parsedChannel.Writer.Complete();  // Báo xong parse
            }
        }, ct);

        // Stage 3: Lưu DB (consumer parsedChannel)
        var writerTask = Task.Run(async () =>
        {
            await foreach (var record in parsedChannel.Reader.ReadAllAsync(ct))
            {
                await _repository.SaveAsync(record, ct);
            }
        }, ct);

        // Chờ tất cả stages hoàn thành
        await Task.WhenAll(readerTask, parserTask, writerTask);
    }
}
```

---

## 🔷 Tích Hợp Với BackgroundService (Hosted Service)

```csharp
// ✅ Pattern chuẩn trong ASP.NET Core: Channel + BackgroundService
public class OrderProcessingService : BackgroundService
{
    private readonly Channel<Order> _channel;
    private readonly IOrderProcessor _processor;

    public OrderProcessingService(IOrderProcessor processor)
    {
        _processor = processor;
        _channel = Channel.CreateBounded<Order>(500);
    }

    // API Controller gọi method này để enqueue order
    public async Task<bool> EnqueueOrderAsync(Order order, CancellationToken ct)
    {
        // WaitToWriteAsync: trả false nếu channel đã Complete
        if (!await _channel.Writer.WaitToWriteAsync(ct))
            return false;

        return _channel.Writer.TryWrite(order);
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var order in _channel.Reader.ReadAllAsync(stoppingToken))
        {
            try
            {
                await _processor.ProcessAsync(order, stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                _logger.LogError(ex, "Failed to process order {OrderId}", order.Id);
            }
        }
    }

    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        _channel.Writer.Complete();    // Không nhận order mới
        await base.StopAsync(cancellationToken);
    }
}
```

---

## 🔷 Channel vs Các Alternatives

### Channel vs BlockingCollection

```csharp
// BlockingCollection — phương thức cũ, không có native async
var blocking = new BlockingCollection<int>(100);

// Producer (synchronous block khi đầy)
blocking.Add(item);  // Block thread — không phải async!

// Consumer (synchronous block khi rỗng)
foreach (var item in blocking.GetConsumingEnumerable())
    Process(item);  // Block thread khi chờ

// ❌ Không có WaitAsync, không async-friendly
// ❌ Block thread thực sự → lãng phí ThreadPool thread
```

```csharp
// Channel<T> — hiện đại, async-friendly
var channel = Channel.CreateBounded<int>(100);

// Producer (async wait khi đầy — không block thread)
await channel.Writer.WriteAsync(item, ct);  // Trả thread về pool

// Consumer (async wait khi rỗng — không block thread)
await foreach (var item in channel.Reader.ReadAllAsync(ct))
    await ProcessAsync(item);

// ✅ Async-native, không block thread, backpressure, cancellation support
```

### Channel vs Queue

| | `Queue<T>` | `ConcurrentQueue<T>` | `Channel<T>` |
| -- | ---------- | -------------------- | ------------ |
| Thread-safe | ❌ | ✅ | ✅ |
| Async-native | ❌ | ❌ | ✅ |
| Backpressure | ❌ | ❌ | ✅ (bounded) |
| Blocking consumer | N/A | Phải poll | ✅ built-in |
| Completion signal | N/A | N/A | ✅ Complete() |

---

## 🔷 Xử Lý Lỗi Và Completion

```csharp
// Complete với exception: consumer nhận Exception khi đọc
channel.Writer.Complete(new Exception("Nguồn dữ liệu lỗi"));

// Consumer xử lý:
try
{
    await foreach (var item in channel.Reader.ReadAllAsync(ct))
    {
        await ProcessAsync(item, ct);
    }
}
catch (ChannelClosedException ex)
{
    // Writer complete với exception
    _logger.LogError(ex.InnerException, "Channel đóng với lỗi");
}

// Kiểm tra channel đã complete chưa:
var completion = channel.Reader.Completion;
if (completion.IsCompleted)
    _logger.LogInformation("Channel đã complete");
```

---

## 📊 Khi Nào Dùng Channel?

```
✅ Dùng Channel<T> khi:
   - Cần producer-consumer async
   - Muốn tách biệt tốc độ producer và consumer (backpressure)
   - Background processing (email, notification, audit log)
   - Pipeline xử lý dữ liệu nhiều bước
   - Worker pool với bounded queue

❌ Không cần Channel khi:
   - Chỉ cần await một task đơn giản
   - Không có producer-consumer tách biệt
   - Số lượng items nhỏ và ít
   - Dùng message broker thực (RabbitMQ, Azure Service Bus) cho distributed system
```

---

## 🎓 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Channel\<T\> khác BlockingCollection\<T\> như thế nào?**
> `BlockingCollection` là phương thức cũ, blocking — block thread thực khi chờ. `Channel<T>` là async-native — dùng `WaitAsync`/`WriteAsync`/`ReadAllAsync`, không block thread, phù hợp với `async`/`await`. `Channel` cũng có backpressure tốt hơn và API rõ ràng hơn.

**Q: Backpressure là gì và tại sao quan trọng?**
> Backpressure — áp ngược — là cơ chế làm chậm producer khi consumer không kịp xử lý. Với `BoundedChannel`, khi channel đầy, `WriteAsync` sẽ await thay vì buffer vô hạn. Nếu không có backpressure, producer có thể tạo hàng triệu items trong memory → OOM (Out of Memory) exception.

**Q: Khi nào dùng UnboundedChannel vs BoundedChannel?**
> `BoundedChannel` an toàn hơn — tránh OOM khi consumer chậm. `UnboundedChannel` chỉ dùng khi chắc chắn producer không thể vượt xa consumer, hoặc khi drop items là không chấp nhận được và bạn tự quản lý memory. Luôn ưu tiên `BoundedChannel` với `FullMode = Wait`.

**Q: Channel có phù hợp cho distributed system không?**
> Không. `Channel<T>` là in-process — trong cùng một process. Cho distributed system — hệ thống phân tán — cần message broker thực: RabbitMQ, Azure Service Bus, Kafka. `Channel` phù hợp cho in-process pipeline, background processing trong một service.

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Thuộc:** Developer/.net/03-async-concurrency/
