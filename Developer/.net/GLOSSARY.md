# 📖 Từ Điển Thuật Ngữ C# .NET

> Tập hợp các thuật ngữ kỹ thuật quan trọng trong hệ sinh thái C# .NET.  
> Thuật ngữ tiếng Anh được giữ nguyên, kèm giải thích tiếng Việt UTF-8.  
> Sắp xếp theo chủ đề, trong mỗi chủ đề sắp xếp theo bảng chữ cái.

---

## 📋 Tra Cứu Nhanh (A–Z)

| Thuật Ngữ | Ý Nghĩa Ngắn Gọn |
|-----------|-------------------|
| AOT | Ahead-Of-Time Compilation — Biên dịch trước khi chạy |
| API Gateway | Cổng vào duy nhất cho toàn bộ microservices |
| ArrayPool\<T\> | Bộ nhớ mảng tái sử dụng, tránh cấp phát heap |
| async/await | Cú pháp lập trình bất đồng bộ trong C# |
| BCL | Base Class Library — Thư viện lớp cơ sở của .NET |
| BenchmarkDotNet | Thư viện đo lường hiệu năng chính xác |
| Boxing | Chuyển value type thành object (lên heap) |
| CI/CD | Tích hợp và triển khai liên tục |
| CLR | Common Language Runtime — Môi trường chạy .NET |
| ConfigMap | Cấu hình không nhạy cảm trong Kubernetes |
| ConfigureAwait | Kiểm soát SynchronizationContext sau await |
| CORS | Cross-Origin Resource Sharing — Chia sẻ tài nguyên chéo nguồn |
| CQRS | Command Query Responsibility Segregation — Tách lệnh/truy vấn |
| CSRF | Cross-Site Request Forgery — Giả mạo yêu cầu chéo trang |
| DDD | Domain-Driven Design — Thiết kế hướng miền |
| Deadlock | Bế tắc: hai luồng chờ nhau vô hạn |
| DI | Dependency Injection — Tiêm phụ thuộc |
| DIP | Dependency Inversion Principle — Nguyên lý đảo ngược phụ thuộc |
| DbContext | Ngữ cảnh làm việc với database trong EF Core |
| Docker | Nền tảng đóng gói ứng dụng vào container |
| EF Core | Entity Framework Core — ORM chính thức của .NET |
| Event Sourcing | Lưu trạng thái hệ thống qua chuỗi sự kiện bất biến |
| GC | Garbage Collector — Bộ thu gom rác tự động |
| GoF | Gang of Four — Bốn tác giả của sách Design Patterns |
| HPA | Horizontal Pod Autoscaler — Tự động mở rộng pod theo chiều ngang |
| HSTS | HTTP Strict Transport Security — Bắt buộc dùng HTTPS |
| IL | Intermediate Language — Ngôn ngữ trung gian của .NET |
| IoC | Inversion of Control — Đảo ngược quyền điều khiển |
| ISP | Interface Segregation Principle — Nguyên lý phân tách giao diện |
| JIT | Just-In-Time Compiler — Biên dịch tại thời điểm chạy |
| JWT | JSON Web Token — Token xác thực dạng JSON |
| K8s | Kubernetes — Hệ thống điều phối container |
| LINQ | Language Integrated Query — Truy vấn tích hợp ngôn ngữ |
| LOH | Large Object Heap — Heap cho đối tượng kích thước lớn (≥ 85KB) |
| LSP | Liskov Substitution Principle — Nguyên lý thay thế Liskov |
| MediatR | Thư viện Mediator pattern phổ biến trong .NET |
| Middleware | Thành phần xử lý request trong pipeline ASP.NET Core |
| N+1 Problem | Vấn đề hiệu năng: 1 truy vấn gốc + N truy vấn con |
| OCP | Open/Closed Principle — Nguyên lý mở/đóng |
| OIDC | OpenID Connect — Giao thức xác thực danh tính trên OAuth 2.0 |
| ORM | Object-Relational Mapper — Ánh xạ đối tượng — quan hệ |
| Outbox Pattern | Đảm bảo gửi message khi lưu DB trong microservices |
| POH | Pinned Object Heap — Heap cho object được ghim cứng trong bộ nhớ |
| RBAC | Role-Based Access Control — Phân quyền dựa trên vai trò |
| Redis | Remote Dictionary Server — Cơ sở dữ liệu key-value trong bộ nhớ |
| Saga Pattern | Quản lý giao dịch phân tán qua chuỗi sự kiện |
| Scoped | Lifetime DI: một instance mỗi HTTP request |
| Singleton | Lifetime DI: một instance duy nhất suốt vòng đời ứng dụng |
| SOH | Small Object Heap — Heap cho đối tượng nhỏ (< 85KB) |
| SOLID | Năm nguyên lý thiết kế phần mềm hướng đối tượng |
| Span\<T\> | Lát cắt bộ nhớ liên tục, không cấp phát heap |
| SRP | Single Responsibility Principle — Nguyên lý một trách nhiệm |
| TDD | Test-Driven Development — Phát triển hướng kiểm thử |
| TLS | Transport Layer Security — Bảo mật tầng truyền tải |
| TPL | Task Parallel Library — Thư viện tác vụ song song |
| Transient | Lifetime DI: instance mới mỗi lần resolve |
| VPA | Vertical Pod Autoscaler — Tự động điều chỉnh tài nguyên pod |
| XSS | Cross-Site Scripting — Tấn công chèn script chéo trang |

---

## 🔵 01. Nền Tảng C# & CLR

### AOT — Ahead-Of-Time Compilation (Biên Dịch Trước Khi Chạy)
Phương pháp biên dịch IL thành mã máy native ngay lúc build, không cần JIT khi chạy. Giảm thời gian khởi động và bộ nhớ sử dụng. .NET 7+ hỗ trợ Native AOT cho các ứng dụng console và API đơn giản.

### BCL — Base Class Library (Thư Viện Lớp Cơ Sở)
Tập hợp các lớp cơ bản được .NET cung cấp sẵn: `System.String`, `System.Collections`, `System.IO`, `System.Threading`... Mọi ứng dụng .NET đều dùng BCL.

### Boxing / Unboxing (Đóng Gói / Mở Gói)
- **Boxing:** Chuyển value type (struct, int, bool...) thành `object` — cấp phát bộ nhớ trên heap.  
- **Unboxing:** Chuyển ngược từ `object` về value type — cần ép kiểu tường minh.  
- Hiệu năng thấp; tránh boxing trong hot path bằng cách dùng Generics.

### CLR — Common Language Runtime (Môi Trường Chạy Ngôn Ngữ Chung)
Máy ảo của .NET, chịu trách nhiệm: biên dịch JIT, quản lý bộ nhớ qua GC, type safety, exception handling, thread management. Mọi ngôn ngữ .NET (C#, F#, VB.NET) đều chạy trên CLR.

### Delegate (Ủy Quyền)
Kiểu tham chiếu trỏ đến một hoặc nhiều phương thức cùng chữ ký. Nền tảng của event và callback trong C#. Các dạng built-in: `Func<T>`, `Action<T>`, `Predicate<T>`.

### Expression Tree (Cây Biểu Thức)
Biểu diễn code dưới dạng cấu trúc dữ liệu có thể duyệt và phân tích tại runtime. LINQ-to-SQL và EF Core dùng Expression Tree để dịch LINQ sang SQL.

### GC — Garbage Collector (Bộ Thu Gom Rác)
Cơ chế tự động thu hồi bộ nhớ của CLR. Chia thành ba thế hệ:
- **Gen 0:** Đối tượng mới sinh, GC chạy thường xuyên nhất.
- **Gen 1:** Đối tượng sống sót qua một lần GC Gen 0.
- **Gen 2:** Đối tượng tồn tại lâu dài; GC chạy ít thường xuyên nhất.

### Generic (Kiểu Tổng Quát)
Cho phép định nghĩa class, method, interface với tham số kiểu (`<T>`). Tránh boxing và tăng tính tái sử dụng. Ví dụ: `List<T>`, `Dictionary<TKey, TValue>`.

### IL — Intermediate Language / MSIL / CIL (Ngôn Ngữ Trung Gian)
Mã trung gian mà C# biên dịch thành. JIT biên dịch IL thành mã máy native khi chạy. Có thể xem IL bằng `ildasm` hoặc ILSpy.

### IEnumerable\<T\> / IQueryable\<T\>
- **`IEnumerable<T>`:** Duyệt tuần tự trong bộ nhớ; thực thi ngay khi enumerate.  
- **`IQueryable<T>`:** Xây dựng expression tree; thực thi tại data source (SQL). EF Core dùng `IQueryable<T>` để dịch LINQ sang SQL.

### JIT — Just-In-Time Compiler (Biên Dịch Đúng Lúc)
Biên dịch IL thành mã máy native tại thời điểm phương thức được gọi lần đầu. Cho phép tối ưu theo phần cứng thực tế. Kết quả được cache cho các lần gọi sau.

### LINQ — Language Integrated Query (Truy Vấn Tích Hợp Ngôn Ngữ)
Tập hợp extension method cho phép truy vấn dữ liệu trực tiếp trong C#. Hai dạng thực thi:
- **Deferred Execution (Thực thi trì hoãn):** Truy vấn chỉ chạy khi enumerate (`foreach`, `.ToList()`).  
- **Immediate Execution (Thực thi ngay):** `Count()`, `ToList()`, `First()`.

### Nullable Reference Type (Kiểu Tham Chiếu Có Thể Null)
Tính năng C# 8+: phân biệt `string` (không thể null) và `string?` (có thể null) tại compile-time. Giảm `NullReferenceException` ở runtime.

### Span\<T\> / Memory\<T\> / ReadOnlySpan\<T\>
- **`Span<T>`:** Lát cắt bộ nhớ liên tục (stack, heap, native), không cấp phát mới, chỉ dùng trên stack. Zero-allocation.  
- **`Memory<T>`:** Tương tự `Span<T>` nhưng có thể lưu trên heap, dùng được trong async.  
- Thường dùng để xử lý string, buffer mà không tạo bản sao.

### Value Type vs Reference Type (Kiểu Giá Trị vs Kiểu Tham Chiếu)
- **Value Type (`struct`, `int`, `bool`, `enum`):** Lưu trực tiếp giá trị trên stack (hoặc inline trong đối tượng). Copy theo giá trị.  
- **Reference Type (`class`, `interface`, `string`, `array`):** Lưu tham chiếu trên stack, dữ liệu thực trên heap. Copy theo tham chiếu.

---

## 🟢 02. OOP & Design Patterns

### Anti-Pattern (Mẫu Phản Khuôn)
Giải pháp thường dùng nhưng thực ra gây hại về lâu dài. Ví dụ: God Object, Service Locator, Tight Coupling, Spaghetti Code.

### Behavioral Patterns — Mẫu Hành Vi
Nhóm design pattern tập trung vào giao tiếp giữa các đối tượng:
- **Chain of Responsibility:** Truyền request qua chuỗi handler.  
- **Command:** Đóng gói hành động thành đối tượng.  
- **Mediator:** Trung gian điều phối giao tiếp giữa các đối tượng.  
- **Observer:** Đăng ký/thông báo khi trạng thái thay đổi.  
- **Strategy:** Hoán đổi thuật toán tại runtime.

### Creational Patterns — Mẫu Khởi Tạo
Nhóm design pattern tập trung vào cách tạo đối tượng:
- **Abstract Factory:** Tạo họ đối tượng liên quan mà không chỉ định lớp cụ thể.  
- **Builder:** Tách biệt quá trình xây dựng đối tượng phức tạp.  
- **Factory Method:** Để subclass quyết định lớp nào được khởi tạo.  
- **Prototype:** Sao chép đối tượng hiện có.  
- **Singleton:** Đảm bảo chỉ có một instance trong toàn bộ ứng dụng.

### DIP — Dependency Inversion Principle (Nguyên Lý Đảo Ngược Phụ Thuộc)
Module cấp cao không phụ thuộc vào module cấp thấp; cả hai phụ thuộc vào abstraction (interface). Cho phép thay thế implementation mà không sửa module cấp cao.

### GoF — Gang of Four (Nhóm Bốn Tác Giả)
Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides — tác giả cuốn "Design Patterns: Elements of Reusable Object-Oriented Software" (1994), định nghĩa 23 pattern kinh điển.

### ISP — Interface Segregation Principle (Nguyên Lý Phân Tách Giao Diện)
Không ép client phụ thuộc vào các method mà họ không dùng. Chia interface lớn thành nhiều interface nhỏ, chuyên biệt hơn.

### LSP — Liskov Substitution Principle (Nguyên Lý Thay Thế Liskov)
Đối tượng subtype phải có thể thay thế supertype mà không làm vỡ tính đúng đắn của chương trình. Vi phạm LSP thường xảy ra khi override phương thức thay đổi hành vi kỳ vọng.

### OCP — Open/Closed Principle (Nguyên Lý Mở/Đóng)
Module nên mở để mở rộng (thêm hành vi mới) nhưng đóng để sửa đổi (không thay đổi code hiện có). Thường đạt được qua abstraction và polymorphism.

### SOLID
Năm nguyên lý thiết kế phần mềm hướng đối tượng:
- **S** — SRP: Single Responsibility Principle  
- **O** — OCP: Open/Closed Principle  
- **L** — LSP: Liskov Substitution Principle  
- **I** — ISP: Interface Segregation Principle  
- **D** — DIP: Dependency Inversion Principle

### SRP — Single Responsibility Principle (Nguyên Lý Một Trách Nhiệm)
Mỗi class chỉ có một lý do để thay đổi. Không phải "chỉ làm một việc" — mà là "chỉ chịu trách nhiệm với một actor".

### Structural Patterns — Mẫu Cấu Trúc
Nhóm design pattern tập trung vào cách tổ chức class và đối tượng:
- **Adapter:** Chuyển đổi interface không tương thích.  
- **Composite:** Coi group đối tượng như một đối tượng đơn.  
- **Decorator:** Thêm hành vi mà không thay đổi lớp gốc.  
- **Facade:** Cung cấp interface đơn giản cho hệ thống phức tạp.  
- **Proxy:** Đại diện kiểm soát truy cập vào đối tượng khác.

---

## 🟡 03. Async, Task & Concurrency

### async/await (Bất Đồng Bộ)
Cú pháp C# để viết code bất đồng bộ theo phong cách tuần tự. Compiler chuyển thành state machine. `await` không block thread — nó trả thread về thread pool trong khi chờ I/O.

### CancellationToken (Token Hủy)
Cơ chế hủy tác vụ bất đồng bộ theo yêu cầu (cooperative cancellation). Caller tạo `CancellationTokenSource`, truyền `Token` xuống callee. Callee kiểm tra `.IsCancellationRequested` hoặc dùng `.ThrowIfCancellationRequested()`.

### Channel\<T\> (Kênh Truyền Dữ Liệu)
Cấu trúc producer-consumer an toàn thread trong `System.Threading.Channels`. Hỗ trợ bounded/unbounded channel. Thay thế cho `ConcurrentQueue<T>` khi cần back-pressure.

### ConfigureAwait(false) (Cấu Hình Không Bắt Lại Context)
Cho biết code tiếp tục sau `await` không cần chạy lại trên `SynchronizationContext` gốc. Dùng trong library code để tránh deadlock và tăng hiệu năng. Không dùng trong UI code.

### Deadlock (Bế Tắc)
Hai hoặc nhiều luồng chờ nhau giải phóng tài nguyên — dẫn đến chờ vô hạn. Nguyên nhân phổ biến: gọi `.Result` hoặc `.Wait()` trên Task trong môi trường có `SynchronizationContext`.

### Interlocked (Khóa Nguyên Tử)
Lớp `System.Threading.Interlocked` cung cấp các thao tác nguyên tử (atomic) như `Increment`, `Decrement`, `CompareExchange`. An toàn thread mà không cần `lock`.

### Monitor / lock (Màn Hình / Khóa)
`lock` là syntactic sugar cho `Monitor.Enter/Exit`. Đảm bảo chỉ một thread truy cập critical section tại một thời điểm. Dùng đối tượng private readonly làm lock object.

### PLINQ — Parallel LINQ (LINQ Song Song)
Phiên bản song song của LINQ, chia tập dữ liệu và xử lý trên nhiều thread. Dùng `.AsParallel()`. Hiệu quả với CPU-bound operations trên tập dữ liệu lớn.

### Race Condition (Điều Kiện Tranh Chấp)
Lỗi xảy ra khi kết quả phụ thuộc vào thứ tự thực thi của các luồng, mà thứ tự này không được đảm bảo. Phòng tránh bằng lock, `Interlocked`, hoặc dùng cấu trúc dữ liệu immutable.

### SemaphoreSlim (Semaphore Nhẹ)
Giới hạn số lượng thread/task được truy cập tài nguyên đồng thời. Hỗ trợ async/await (`WaitAsync()`). Dùng để throttle HTTP calls, giới hạn concurrency.

### SynchronizationContext (Ngữ Cảnh Đồng Bộ)
Abstraction đại diện cho "nơi" mà code cần được chạy sau `await` (ví dụ: UI thread, ASP.NET request context). ASP.NET Core không có SynchronizationContext — ít rủi ro deadlock hơn.

### Task Parallel Library — TPL (Thư Viện Tác Vụ Song Song)
Framework quản lý tác vụ bất đồng bộ và song song: `Task`, `Task<T>`, `Parallel.For`, `Parallel.ForEach`, `Task.WhenAll`, `Task.WhenAny`. Nền tảng của async/await.

### Thread vs Task (Luồng vs Tác Vụ)
- **Thread:** Luồng OS cụ thể; nặng (1MB stack mặc định); quản lý thủ công.  
- **Task:** Đơn vị công việc logic; được thread pool quản lý; nhẹ hơn; hỗ trợ async/await.

### volatile (Từ Khóa Biến Động)
Đảm bảo đọc/ghi biến không được cache bởi CPU — luôn đọc từ bộ nhớ chính. Không đảm bảo atomicity; dùng `Interlocked` nếu cần.

---

## 🟣 04. ASP.NET Core

### Action Filter / Exception Filter (Bộ Lọc Hành Động / Ngoại Lệ)
Các hook chạy trước/sau action method (`IActionFilter`), hoặc bắt exception toàn cục (`IExceptionFilter`). Dùng cho cross-cutting concerns: logging, validation, error handling.

### Data Annotations (Chú Thích Dữ Liệu)
Attribute trên model property để khai báo validation rules: `[Required]`, `[MaxLength]`, `[EmailAddress]`, `[Range]`. Model binding tự động validate khi controller nhận request.

### Dependency Injection — DI (Tiêm Phụ Thuộc)
Design pattern: thay vì class tự tạo dependency, dependency được cung cấp từ bên ngoài (thường qua constructor). ASP.NET Core có DI container tích hợp sẵn.

### FluentValidation (Validation Linh Hoạt)
Thư viện xây dựng validation logic bằng fluent API (`.RuleFor(x => x.Email).NotEmpty().EmailAddress()`). Tách biệt validation khỏi model, dễ test hơn Data Annotations.

### Health Check (Kiểm Tra Tình Trạng)
Endpoint `/health` cho biết trạng thái ứng dụng:
- **Liveness Probe:** Pod còn sống không? Kubernetes dùng để restart pod hỏng.  
- **Readiness Probe:** Pod sẵn sàng nhận traffic chưa? Kubernetes dùng để điều hướng load balancer.

### IOptions\<T\> / IOptionsMonitor\<T\> (Giao Diện Cấu Hình Có Kiểu)
Pattern quản lý cấu hình strongly-typed trong ASP.NET Core. `IOptions<T>`: giá trị snapshot lúc startup. `IOptionsMonitor<T>`: tự động cập nhật khi file cấu hình thay đổi.

### IoC — Inversion of Control (Đảo Ngược Quyền Điều Khiển)
Nguyên lý: framework (container) điều khiển luồng tạo đối tượng thay vì code ứng dụng. DI là một cách hiện thực IoC.

### Middleware (Phần Mềm Trung Gian)
Thành phần trong pipeline xử lý HTTP request và response. Mỗi middleware nhận request, xử lý, rồi chuyển tiếp (`next()`) hoặc short-circuit (không gọi `next()`). Ví dụ: Authentication, CORS, Routing, Logging.

### Minimal API (API Tối Giản)
Cách xây dựng HTTP endpoint trong ASP.NET Core mà không cần Controller class. Dùng `app.MapGet()`, `app.MapPost()`... Phù hợp microservices và prototype.

### Model Binding (Ràng Buộc Model)
Quá trình ASP.NET Core tự động ánh xạ dữ liệu từ HTTP request (query string, route, body, form, header) vào tham số của action method.

### Routing (Định Tuyến)
Cơ chế ánh xạ URL request đến handler cụ thể. Hai kiểu: Attribute Routing (`[Route("api/[controller]")]`) và Conventional Routing (`app.MapControllerRoute(...)`).

### Scoped / Singleton / Transient (Vòng Đời DI)
- **Singleton:** Một instance duy nhất suốt vòng đời ứng dụng.  
- **Scoped:** Một instance mỗi HTTP request (hoặc mỗi DI scope).  
- **Transient:** Instance mới mỗi lần được resolve từ container.

---

## 🟤 05. Entity Framework Core

### AsNoTracking() (Không Theo Dõi)
Extension method tắt change tracking cho query. Tăng hiệu năng đáng kể cho read-only queries. Không dùng khi cần update entity sau khi query.

### Change Tracking (Theo Dõi Thay Đổi)
EF Core theo dõi trạng thái entity (`Added`, `Modified`, `Deleted`, `Unchanged`). `SaveChanges()` dựa vào change tracking để sinh SQL INSERT/UPDATE/DELETE tương ứng.

### Code-First (Ưu Tiên Code)
Phương pháp: định nghĩa model C#, EF Core sinh SQL schema và migrations. Ngược với Database-First (reverse engineering từ database có sẵn).

### Compiled Query (Truy Vấn Đã Biên Dịch)
`EF.CompileQuery(...)` biên dịch và cache LINQ query thành delegate. Loại bỏ overhead phân tích expression tree mỗi lần gọi. Dùng cho query được gọi nhiều lần.

### DbContext (Ngữ Cảnh Database)
Lớp trung tâm của EF Core: quản lý kết nối, change tracking, transaction, migrations. Nên đăng ký với lifetime `Scoped` trong ASP.NET Core.

### DbSet\<T\> (Tập Thực Thể)
Property trong `DbContext`, đại diện cho một bảng database. Là `IQueryable<T>` — cho phép dùng LINQ để xây dựng query, thực thi tại database.

### Eager Loading (Tải Háo Hức)
Load related entities cùng lúc với entity chính bằng `.Include()` / `.ThenInclude()`. Tránh N+1 problem. Sinh SQL JOIN hoặc split queries.

### Lazy Loading (Tải Lười Biếng)
Related entities được load tự động khi truy cập property lần đầu. Tiện nhưng dễ gây N+1 problem. Cần cấu hình proxy hoặc `ILazyLoader`.

### Migration (Di Chuyển Schema)
Quá trình thay đổi database schema có kiểm soát và versioned. `Add-Migration`, `Update-Database`, `Script-Migration` là các lệnh chính. Mỗi migration là một diff schema.

### N+1 Problem (Vấn Đề N+1 Truy Vấn)
Lỗi hiệu năng: query 1 lấy N records, sau đó N query riêng lẻ lấy related data. Tổng: N+1 round-trips đến database. Giải quyết bằng Eager Loading hoặc projection.

### ORM — Object-Relational Mapper (Ánh Xạ Đối Tượng — Quan Hệ)
Framework chuyển đổi giữa đối tượng C# và bảng quan hệ trong database. EF Core là ORM chính thức của .NET; Dapper là micro-ORM nhẹ hơn.

### Projection (Chiếu Dữ Liệu)
Chỉ lấy đúng column cần thiết thay vì toàn bộ entity. Dùng `.Select(x => new { x.Id, x.Name })` hoặc DTO. Giảm data transfer và tăng hiệu năng.

### Split Query (Truy Vấn Tách)
EF Core 5+: chia một query phức tạp với nhiều `.Include()` thành nhiều query SQL riêng biệt để tránh Cartesian explosion. Dùng `.AsSplitQuery()`.

---

## 🔴 06. Testing

### AAA — Arrange, Act, Assert (Sắp Xếp — Thực Hiện — Kiểm Tra)
Cấu trúc chuẩn của unit test: Arrange (chuẩn bị dữ liệu, mock), Act (gọi code cần test), Assert (kiểm tra kết quả mong đợi).

### BDD — Behavior-Driven Development (Phát Triển Hướng Hành Vi)
Mở rộng TDD, viết test theo ngôn ngữ tự nhiên: Given/When/Then. Thư viện: SpecFlow, Reqnroll.

### BenchmarkDotNet (Đo Lường Hiệu Năng)
Thư viện benchmark chính xác cho .NET: xử lý warmup, multiple runs, statistical analysis. Kết quả tin cậy để so sánh implementations. Attribute `[Benchmark]` đánh dấu method cần đo.

### Code Coverage / Test Coverage (Độ Bao Phủ Code)
Phần trăm code được thực thi bởi test suite. Đo bằng Coverlet. Coverage cao không đảm bảo chất lượng — chất lượng assertion quan trọng hơn số lượng.

### Integration Test (Kiểm Thử Tích Hợp)
Test nhiều component cùng nhau (API endpoint, database, message queue). Dùng `WebApplicationFactory<T>` cho ASP.NET Core. Chậm hơn unit test nhưng tin cậy hơn.

### Mock (Giả Lập)
Đối tượng thay thế dependency thực, có thể lập trình hành vi và verify interaction. Thư viện: Moq (`mock.Setup(...).Returns(...)`), NSubstitute.

### NUnit / xUnit / MSTest (Khung Kiểm Thử)
Các framework unit testing phổ biến cho .NET:
- **xUnit:** Hiện đại nhất, được .NET team dùng; `[Fact]`, `[Theory]`.  
- **NUnit:** Nhiều tính năng, linh hoạt; `[Test]`, `[TestCase]`.  
- **MSTest:** Tích hợp sẵn Visual Studio; `[TestMethod]`.

### Stub (Phiên Bản Giả)
Đối tượng trả về giá trị cố định, không verify interaction. Đơn giản hơn Mock. Dùng khi chỉ cần giá trị trả về, không quan tâm cách gọi.

### TDD — Test-Driven Development (Phát Triển Hướng Kiểm Thử)
Quy trình: viết test thất bại (Red) → viết code tối thiểu để test pass (Green) → refactor (Refactor). Đảm bảo code luôn có test, thiết kế tốt hơn.

### Test Double (Đôi Kiểm Thử)
Thuật ngữ tổng quát cho mọi đối tượng thay thế dependency thực trong test: Mock, Stub, Spy, Fake, Dummy.

### TestContainers (Container Kiểm Thử)
Thư viện khởi động Docker container thực (PostgreSQL, Redis, RabbitMQ...) trong integration test. Cho test sát với production nhất. NuGet: `Testcontainers`.

### Unit Test (Kiểm Thử Đơn Vị)
Test một đơn vị code nhỏ (method, class) trong isolation. Nhanh, deterministic, không phụ thuộc I/O thực. Dùng mock để cô lập dependencies.

### WebApplicationFactory\<T\> (Factory Ứng Dụng Web Kiểm Thử)
Khởi động ASP.NET Core app trong bộ nhớ để integration test. Không cần server thực. Cho phép override cấu hình và services cho test.

---

## 🟠 07. Hiệu Năng & Bộ Nhớ

### ArrayPool\<T\> (Pool Mảng)
`System.Buffers.ArrayPool<T>.Shared` cung cấp mảng tái sử dụng, tránh cấp phát heap liên tục. Dùng `Rent(size)` / `Return(array)`. Quan trọng cho xử lý I/O hiệu năng cao.

### GC Pressure (Áp Lực Thu Gom Rác)
Tần suất GC phải chạy. Cấp phát nhiều đối tượng ngắn hạn gây GC pressure cao, làm chậm ứng dụng. Giảm bằng cách dùng Span, ArrayPool, struct, object pooling.

### LOH — Large Object Heap (Heap Đối Tượng Lớn)
Vùng heap đặc biệt cho đối tượng ≥ 85,000 bytes. LOH không bị compacted thường xuyên, dễ fragmentation. Tránh cấp phát array lớn thường xuyên.

### Memory Leak (Rò Rỉ Bộ Nhớ)
Đối tượng không còn cần nhưng vẫn có reference đến → GC không thu hồi được. Nguyên nhân phổ biến: static event handler, long-lived cache không bounded, Dispose chưa đúng cách.

### MemoryPool\<T\> / ObjectPool\<T\> (Pool Bộ Nhớ / Pool Đối Tượng)
- **`MemoryPool<T>`:** Thuê block `Memory<T>` tái sử dụng.  
- **`ObjectPool<T>`:** Tái sử dụng đối tượng nặng (StringBuilder, HttpClient, DbConnection).

### POH — Pinned Object Heap (Heap Đối Tượng Ghim)
.NET 5+: heap riêng cho object được ghim (pinned) để trao đổi với native code. Tránh fragmentation LOH/SOH. Dùng khi cần interop P/Invoke.

### SOH — Small Object Heap (Heap Đối Tượng Nhỏ)
Heap chính cho object < 85KB. GC compacts SOH sau mỗi collection để tránh fragmentation.

---

## 🔒 08. Bảo Mật

### Authentication (Xác Thực)
Xác minh danh tính: "Bạn là ai?" JWT, Cookie, API Key, OAuth2 là các cơ chế authentication phổ biến.

### Authorization (Phân Quyền)
Kiểm tra quyền hạn: "Bạn được phép làm gì?" Thực hiện sau authentication. Các dạng:
- **RBAC (Role-Based):** Phân quyền theo vai trò (Admin, User, Manager).  
- **Claims-Based:** Phân quyền theo thông tin trong claims.  
- **Policy-Based:** Phân quyền theo rules phức tạp tùy chỉnh.

### Bearer Token (Token Mang Theo)
Cơ chế xác thực: client gửi token trong header `Authorization: Bearer <token>`. Server xác minh token mà không cần session. JWT thường được dùng làm Bearer Token.

### CORS — Cross-Origin Resource Sharing (Chia Sẻ Tài Nguyên Chéo Nguồn Gốc)
Cơ chế browser cho phép/từ chối request từ domain khác. Server cấu hình `Access-Control-Allow-Origin` header. Cấu hình sai gây lỗ hổng bảo mật hoặc block request hợp lệ.

### CSRF — Cross-Site Request Forgery (Giả Mạo Yêu Cầu Chéo Trang)
Tấn công: trang web độc hại gửi request đến site khác sử dụng cookie của nạn nhân. Phòng tránh bằng Anti-Forgery Token (cho form-based) hoặc SameSite cookie.

### Data Protection API (API Bảo Vệ Dữ Liệu)
ASP.NET Core framework mã hóa/giải mã dữ liệu nhạy cảm (cookie, token). Hỗ trợ key rotation tự động. Dùng `IDataProtector` cho custom encryption.

### HSTS — HTTP Strict Transport Security (Bắt Buộc Dùng HTTPS)
HTTP header yêu cầu browser chỉ kết nối qua HTTPS. Ngăn downgrade attack. `max-age` chỉ định thời gian nhớ. Dùng `UseHsts()` trong ASP.NET Core.

### JWT — JSON Web Token (Token Xác Thực JSON)
Chuỗi base64url gồm 3 phần: `Header.Payload.Signature`. Stateless — server không cần lưu session. Payload chứa claims (thông tin người dùng). Ký bằng HMAC hoặc RSA.

### OAuth 2.0 (Khung Ủy Quyền)
Giao thức ủy quyền: cho phép ứng dụng thứ ba truy cập tài nguyên thay mặt người dùng mà không cần mật khẩu. Các flow: Authorization Code, Client Credentials, Implicit (deprecated).

### OIDC — OpenID Connect (Giao Thức Xác Thực Danh Tính)
Layer xác thực danh tính trên OAuth 2.0. Thêm `id_token` (JWT chứa thông tin người dùng). Dùng cho "Login with Google/Microsoft/GitHub".

### SQL Injection (Tiêm Mã SQL)
Tấn công: chèn SQL vào input để thao túng database query. Phòng tránh bằng parameterized queries, ORM, stored procedures. KHÔNG bao giờ nối string vào SQL query.

### TLS — Transport Layer Security (Bảo Mật Tầng Truyền Tải)
Giao thức mã hóa kết nối mạng (thay thế SSL). HTTPS = HTTP + TLS. TLS 1.3 là phiên bản an toàn nhất hiện nay. Certificate xác minh danh tính server.

### XSS — Cross-Site Scripting (Tấn Công Chèn Script Chéo Trang)
Tấn công: chèn script độc hại vào trang web để chạy trên trình duyệt nạn nhân. Phòng tránh bằng output encoding, Content Security Policy (CSP), tránh `innerHTML`.

---

## 🏛️ 09. Kiến Trúc Phần Mềm

### Aggregate / Aggregate Root (Tổng Hợp / Gốc Tổng Hợp)
Trong DDD: nhóm đối tượng liên quan được coi như một đơn vị nhất quán. `Aggregate Root` là điểm vào duy nhất — mọi thao tác phải qua root, không tác động trực tiếp vào child entities.

### Bounded Context (Ngữ Cảnh Giới Hạn)
Trong DDD: ranh giới rõ ràng trong đó một domain model cụ thể có giá trị và ý nghĩa nhất quán. Các Bounded Context khác nhau có thể dùng cùng thuật ngữ với nghĩa khác nhau.

### Clean Architecture (Kiến Trúc Sạch)
Kiến trúc phân tầng đồng tâm của Robert C. Martin:
- **Domain:** Business logic thuần, không phụ thuộc gì.  
- **Application:** Use cases, interfaces.  
- **Infrastructure:** Database, external services.  
- **Presentation:** API, UI.  
Dependency chỉ trỏ vào trong (vào Domain).

### CQRS — Command Query Responsibility Segregation (Phân Tách Trách Nhiệm Lệnh/Truy Vấn)
Tách write model (Command) và read model (Query) thành hai luồng riêng biệt. Command thay đổi state, không trả data. Query đọc data, không thay đổi state. Thường kết hợp với MediatR.

### DDD — Domain-Driven Design (Thiết Kế Hướng Miền)
Phương pháp thiết kế phần mềm của Eric Evans: tập trung vào domain business, xây dựng ubiquitous language chung, mô hình hóa qua Entity, Value Object, Aggregate, Domain Event, Repository.

### Domain Event (Sự Kiện Miền)
Sự kiện xảy ra trong domain business có ý nghĩa với business. Ví dụ: `OrderPlaced`, `PaymentReceived`. Dùng để tách biệt side effects và integrate giữa các Bounded Context.

### Entity (Thực Thể)
Trong DDD: đối tượng có identity duy nhất (ID) tồn tại theo thời gian. Hai entity cùng attributes nhưng khác ID là khác nhau. Ví dụ: `User`, `Order`, `Product`.

### Event Sourcing (Nguồn Sự Kiện)
Lưu trữ trạng thái ứng dụng dưới dạng chuỗi events bất biến thay vì state hiện tại. Có thể replay events để tái tạo state. Cho phép audit trail và time travel.

### MediatR (Thư Viện Mediator)
Thư viện .NET hiện thực Mediator pattern. Request/Response (`IRequest<T>`), Notification (`INotification`), Pipeline Behavior. Dùng để tách biệt logic xử lý CQRS.

### Microservices (Vi Dịch Vụ)
Kiến trúc chia ứng dụng thành nhiều service nhỏ, độc lập, giao tiếp qua API/message. Ưu: scale độc lập, deploy độc lập. Nhược: phức tạp về distributed systems.

### Monolith (Đơn Khối)
Ứng dụng một khối duy nhất, mọi component chạy cùng process. Đơn giản khi nhỏ, khó scale khi lớn. Không phải anti-pattern — đúng cho nhiều use case.

### Outbox Pattern (Mẫu Hộp Thư Gửi)
Đảm bảo message được publish sau khi lưu DB thành công (không mất message). Lưu message vào bảng `Outbox` cùng transaction với DB write. Background job đọc và publish.

### Repository Pattern (Mẫu Kho Lưu Trữ)
Abstraction cho data access: che giấu chi tiết truy vấn database sau interface (`IUserRepository`). Application layer tương tác qua interface, không biết EF Core hay SQL.

### Saga Pattern (Mẫu Saga)
Quản lý giao dịch phân tán qua chuỗi local transactions với compensating transactions nếu thất bại. Hai dạng: Choreography (event-driven) và Orchestration (coordinator).

### Ubiquitous Language (Ngôn Ngữ Phổ Quát)
Trong DDD: ngôn ngữ chung giữa developer và domain expert, dùng nhất quán trong code, tài liệu và giao tiếp. Tránh hiểu nhầm giữa business và kỹ thuật.

### Value Object (Đối Tượng Giá Trị)
Trong DDD: đối tượng không có identity, xác định bởi giá trị attributes. Immutable. Ví dụ: `Money(100, "USD")`, `Address`, `Email`. Hai Value Object cùng giá trị là bằng nhau.

### Vertical Slice Architecture (Kiến Trúc Lát Dọc)
Tổ chức code theo feature (vertical slice) thay vì theo layer (horizontal). Mỗi feature chứa toàn bộ: API, handler, query, model. Giảm coupling giữa features.

---

## ☁️ 10. Cloud & Triển Khai

### AKS — Azure Kubernetes Service (Dịch Vụ Kubernetes Azure)
Dịch vụ Kubernetes được quản lý hoàn toàn trên Azure. Microsoft quản lý control plane; người dùng chỉ quản lý worker nodes.

### API Gateway (Cổng API)
Điểm vào duy nhất cho client truy cập microservices. Xử lý: routing, auth, rate limiting, logging, SSL termination. Ví dụ: Azure API Management, Kong, NGINX.

### CI/CD — Continuous Integration / Continuous Delivery (Tích Hợp / Triển Khai Liên Tục)
- **CI:** Tự động build và test mỗi khi code được push.  
- **CD:** Tự động deploy lên môi trường (staging/production) sau khi CI pass.  
Công cụ: GitHub Actions, Azure DevOps, Jenkins.

### ConfigMap (Bản Đồ Cấu Hình K8s)
Kubernetes resource lưu cặp key-value cho cấu hình không nhạy cảm (URLs, timeouts, feature flags). Inject vào pod qua environment variables hoặc volume.

### Container (Đơn Vị Chứa)
Đơn vị đóng gói ứng dụng cùng dependencies, isolated khỏi host OS. Nhẹ hơn VM vì dùng chung kernel host. Docker là runtime phổ biến nhất.

### Docker (Nền Tảng Container)
Nền tảng tạo, chạy, phân phối container. Dockerfile định nghĩa cách build image. Multi-stage build giảm kích thước image production.

### Dockerfile (File Mô Tả Container)
File text định nghĩa cách build Docker image: base image, copy files, install dependencies, expose port, entrypoint.

### Helm (Quản Lý Gói Kubernetes)
Package manager cho Kubernetes. Helm Chart đóng gói toàn bộ K8s resources của ứng dụng. Dùng để deploy và quản lý version ứng dụng phức tạp trên K8s.

### HPA — Horizontal Pod Autoscaler (Tự Động Mở Rộng Pod Theo Chiều Ngang)
Kubernetes tự động tăng/giảm số replicas dựa trên CPU, memory, hoặc custom metrics. Scale out = thêm pod, scale in = giảm pod.

### Ingress (Bộ Điều Hướng Vào)
Kubernetes resource định nghĩa routing rules từ ngoài vào services trong cluster. Ingress Controller (NGINX, Traefik) thực thi các rules này.

### Kubernetes / K8s (Điều Phối Container)
Hệ thống điều phối container mã nguồn mở. Quản lý: deployment, scaling, health check, service discovery, load balancing cho containerized applications.

### Multi-Stage Build (Build Đa Giai Đoạn)
Dockerfile dùng nhiều `FROM` stage: stage build (SDK) biên dịch code; stage final (runtime) chỉ copy artifact. Image production nhỏ gọn, không có SDK và source code.

### Observability (Khả Năng Quan Sát)
Ba trụ cột:
- **Logging (Ghi Log):** Sự kiện rời rạc — Serilog, structured logging.  
- **Metrics (Chỉ Số):** Số đo tổng hợp theo thời gian — Prometheus, Grafana.  
- **Tracing (Theo Dõi Phân Tán):** Luồng request qua nhiều service — OpenTelemetry, Jaeger.

### OpenTelemetry — OTEL (Quan Sát Mở)
Tiêu chuẩn mã nguồn mở để thu thập telemetry (logs, metrics, traces) từ ứng dụng. Vendor-neutral: dữ liệu có thể gửi đến Jaeger, Zipkin, Datadog, Azure Monitor.

### Pod (Đơn Vị Nhỏ Nhất K8s)
Đơn vị triển khai nhỏ nhất trong Kubernetes, chứa một hoặc nhiều container cùng chia sẻ network và storage. Thường một pod = một container.

### Secret (K8s) (Bí Mật K8s)
Kubernetes resource lưu dữ liệu nhạy cảm (password, API key, certificate) dưới dạng base64-encoded. Inject vào pod qua environment variables hoặc volume mount.

### Serilog (Thư Viện Ghi Log Cấu Trúc)
Thư viện logging phổ biến cho .NET hỗ trợ structured logging: log dưới dạng key-value thay vì plain text. Có thể gửi log đến nhiều sink: Console, File, Seq, Elasticsearch.

### Service (K8s) (Dịch Vụ K8s)
Kubernetes resource cung cấp địa chỉ IP ổn định và DNS name cho một nhóm pods. Load balance traffic giữa các pod. Các loại: ClusterIP, NodePort, LoadBalancer.

### VPA — Vertical Pod Autoscaler (Tự Động Điều Chỉnh Tài Nguyên Pod)
Kubernetes tự động điều chỉnh CPU và memory request/limit cho pod dựa trên usage thực tế. Khác với HPA (thêm pod), VPA tăng tài nguyên cho pod hiện có.

---

## 🔗 Các Thuật Ngữ Thường Bị Nhầm Lẫn

| Cặp Thuật Ngữ | Phân Biệt |
|---------------|-----------|
| **Authentication vs Authorization** | Authentication: "Bạn là ai?" — Authorization: "Bạn được làm gì?" |
| **async vs parallel** | async: không block thread (I/O-bound) — parallel: nhiều thread cùng làm (CPU-bound) |
| **Mock vs Stub** | Mock: verify interaction — Stub: chỉ trả giá trị cố định |
| **Singleton (Pattern) vs Singleton (DI)** | Pattern: đảm bảo một instance — DI lifetime: một instance per DI container |
| **Interface vs Abstract Class** | Interface: chỉ contract — Abstract class: có thể có implementation |
| **Task vs Thread** | Task: đơn vị công việc logic — Thread: luồng OS cụ thể |
| **Eager vs Lazy Loading** | Eager: load cùng lúc với Include() — Lazy: load khi truy cập property |
| **IEnumerable vs IQueryable** | IEnumerable: xử lý trong memory — IQueryable: dịch thành SQL tại database |
| **Value Type vs Reference Type** | Value Type: copy giá trị — Reference Type: copy tham chiếu |
| **Liveness vs Readiness Probe** | Liveness: pod còn sống không? — Readiness: pod sẵn sàng nhận traffic chưa? |
| **HPA vs VPA** | HPA: thêm/bớt pod — VPA: tăng/giảm tài nguyên pod |
| **CQRS vs Event Sourcing** | CQRS: tách read/write model — Event Sourcing: lưu state qua chuỗi events |
| **Monolith vs Microservices** | Không có cái nào tốt hơn tuyệt đối — phụ thuộc quy mô và team |

---

## 📦 Thư Viện & Công Cụ Quan Trọng

| Tên | Mục Đích | NuGet / Lệnh |
|-----|----------|--------------|
| **Moq** | Mocking framework | `Moq` |
| **NSubstitute** | Mocking framework (cú pháp đơn giản hơn) | `NSubstitute` |
| **xUnit** | Unit testing framework | `xunit` |
| **FluentAssertions** | Assertion library dễ đọc | `FluentAssertions` |
| **BenchmarkDotNet** | Benchmarking chính xác | `BenchmarkDotNet` |
| **MediatR** | Mediator pattern (CQRS) | `MediatR` |
| **FluentValidation** | Validation linh hoạt | `FluentValidation.AspNetCore` |
| **Serilog** | Structured logging | `Serilog.AspNetCore` |
| **Polly** | Resilience: retry, circuit breaker, timeout | `Polly` |
| **MassTransit** | Message bus (RabbitMQ, Azure Service Bus) | `MassTransit` |
| **Dapper** | Micro-ORM — raw SQL an toàn | `Dapper` |
| **Coverlet** | Code coverage | `coverlet.collector` |
| **Testcontainers** | Docker container trong integration test | `Testcontainers` |
| **dotnet-trace** | Profiling CPU/memory | `dotnet tool install dotnet-trace` |
| **dotnet-dump** | Memory dump analysis | `dotnet tool install dotnet-dump` |

---

## 🔢 Số Liệu & Ngưỡng Tham Khảo

| Chỉ Số | Ngưỡng / Giá Trị |
|--------|-----------------|
| LOH threshold | Object ≥ 85,000 bytes vào LOH |
| Gen 0 GC | Kích hoạt thường xuyên nhất (~256KB default) |
| JWT expiry khuyến nghị | Access token: 15 phút — Refresh token: 7–30 ngày |
| Health check timeout | Khuyến nghị < 3 giây |
| HTTP timeout khuyến nghị | 30 giây (tùy use case) |
| Kubernetes liveness period | 10–30 giây |
| Thread pool min threads | Bằng số CPU cores |

---

**Cập Nhật Lần Cuối:** 2026-06-02  
**Phiên Bản:** 1.0  
**Liên Quan:** [README.md](README.md) | [INDEX.md](INDEX.md)
