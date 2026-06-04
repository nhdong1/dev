# Top 30 Câu Hỏi Phỏng Vấn C# .NET — Đáp Án Chi Tiết

> Bộ câu hỏi tổng hợp từ các cuộc phỏng vấn thực tế cho vị trí Junior → Senior .NET Developer. Mỗi câu có đáp án ngắn gọn, giải thích sâu và ví dụ code.

---

## 📊 Phân Loại Theo Chủ Đề

| Nhóm | Câu Hỏi | File Section |
| ---- | -------- | ------------ |
| C# Language & CLR | Q1–Q8 | [→ xem bên dưới](#nhóm-1--c-language--clr) |
| ASP.NET Core | Q9–Q14 | [→ xem bên dưới](#nhóm-2--aspnet-core) |
| Entity Framework Core | Q15–Q18 | [→ xem bên dưới](#nhóm-3--entity-framework-core) |
| Design Patterns & Architecture | Q19–Q24 | [→ xem bên dưới](#nhóm-4--design-patterns--architecture) |
| Testing & Performance | Q25–Q27 | [→ xem bên dưới](#nhóm-5--testing--performance) |
| Security | Q28–Q30 | [→ xem bên dưới](#nhóm-6--security) |

---

## Nhóm 1 — C# Language & CLR

### Q1. Sự khác nhau giữa `value type` (kiểu giá trị) và `reference type` (kiểu tham chiếu) là gì?

**Đáp án ngắn:** Value type lưu dữ liệu trực tiếp trên stack, reference type lưu địa chỉ tham chiếu trên stack trỏ đến dữ liệu trên heap.

**Giải thích chi tiết:**

```csharp
// Value type — struct, int, bool, double, enum, DateTime
int a = 10;
int b = a;  // Copy giá trị
b = 20;
Console.WriteLine(a); // 10 — a không bị ảnh hưởng

// Reference type — class, string, array, delegate
var list1 = new List<int> { 1, 2, 3 };
var list2 = list1;  // Copy tham chiếu (địa chỉ), KHÔNG copy dữ liệu
list2.Add(4);
Console.WriteLine(list1.Count); // 4 — list1 bị ảnh hưởng
```

**Bảng so sánh:**

| Tiêu Chí | Value Type | Reference Type |
| -------- | ---------- | -------------- |
| Lưu ở đâu | Stack (thường) | Heap |
| Gán biến | Copy dữ liệu | Copy địa chỉ |
| Default value | 0 / false / null (struct) | null |
| Nullable | Cần `int?` | Mặc định null |
| GC quản lý | Không (stack frame) | Có |
| Ví dụ | int, bool, struct, enum | class, string, array |

**Trade-off cần biết:** `struct` phù hợp cho dữ liệu nhỏ, bất biến (Vector2D, Money). Class phù hợp khi cần kế thừa, polymorphism hoặc dữ liệu lớn.

---

### Q2. GC — Garbage Collector — hoạt động như thế nào trong .NET?

**Đáp án ngắn:** GC — Garbage Collector — Bộ Thu Gom Rác — tự động giải phóng bộ nhớ heap không còn được tham chiếu. Dùng thuật toán mark-and-sweep với 3 thế hệ.

**Các thế hệ của GC:**

```
Gen 0 — Thế Hệ 0 (Young objects — đối tượng mới tạo)
  • Thu gom thường xuyên nhất (~100ms/lần)
  • Hầu hết đối tượng chết ở đây (infant mortality)

Gen 1 — Thế Hệ 1 (Buffer giữa Gen0 và Gen2)
  • Đối tượng sống sót qua Gen0
  • Thu gom ít thường xuyên hơn

Gen 2 — Thế Hệ 2 (Long-lived objects — đối tượng sống lâu)
  • Static fields, cache, long-running objects
  • Full GC: tốn kém, pause ứng dụng (stop-the-world)

LOH — Large Object Heap — Heap Đối Tượng Lớn
  • Đối tượng >= 85,000 bytes
  • Không compact, chỉ thu gom khi Gen2 GC
```

**Cách giúp GC hoạt động tốt:**

```csharp
// 1. Dùng using để dispose sớm
using var conn = new SqlConnection(connStr);
// conn.Dispose() được gọi tự động khi ra khỏi block

// 2. Tránh allocation không cần thiết
// Xấu: tạo string mới trong loop
for (int i = 0; i < 1000; i++)
    var s = "prefix" + i.ToString(); // boxing + concatenation

// Tốt: dùng StringBuilder hoặc Span<T>
var sb = new StringBuilder();
for (int i = 0; i < 1000; i++)
    sb.Append("prefix").Append(i);

// 3. ArrayPool để tái dụng array
var pool = ArrayPool<byte>.Shared;
var buffer = pool.Rent(1024);
try { /* dùng buffer */ }
finally { pool.Return(buffer); }
```

---

### Q3. `async`/`await` hoạt động như thế nào? State machine là gì?

**Đáp án ngắn:** `async`/`await` là syntactic sugar — cú pháp tiện — cho state machine. Compiler biến method `async` thành một class implement `IAsyncStateMachine`, cho phép "tạm dừng" method mà không block thread.

**Flow thực thi:**

```csharp
public async Task<string> GetDataAsync()
{
    // Thread pool thread T1 chạy đến đây
    var data = await httpClient.GetStringAsync(url);
    // T1 được giải phóng, chờ I/O
    // Khi I/O xong, thread pool thread T2 (có thể khác T1) tiếp tục
    return data.ToUpper();
}
```

**Lỗi phổ biến — deadlock với `.Result` hoặc `.Wait()`:**

```csharp
// ❌ XẤU — Gây deadlock trong ASP.NET Framework (không phải Core)
public string GetData()
{
    return GetDataAsync().Result; // Block thread đang chờ SynchronizationContext
}

// ✅ TỐT — Await toàn bộ chain
public async Task<string> GetData()
{
    return await GetDataAsync();
}

// ✅ TỐT — ConfigureAwait(false) trong library code
public async Task<string> GetLibraryData()
{
    var data = await httpClient.GetStringAsync(url).ConfigureAwait(false);
    return data;
}
```

**`async void` — cạm bẫy cần tránh:**

```csharp
// ❌ XẤU — Exception không được catch, app crash
public async void SendEmail()
{
    await emailService.SendAsync(); // Nếu throw → crash app
}

// ✅ TỐT — async Task
public async Task SendEmailAsync()
{
    await emailService.SendAsync();
}
```

---

### Q4. `IDisposable` — Giao Diện Giải Phóng — và `using` statement dùng khi nào?

**Đáp án ngắn:** Dùng `IDisposable` khi class quản lý tài nguyên **không được quản lý** (unmanaged) như file handle, database connection, network socket, COM object.

```csharp
public class FileProcessor : IDisposable
{
    private FileStream _stream;
    private bool _disposed = false;

    public FileProcessor(string path)
    {
        _stream = new FileStream(path, FileMode.Open);
    }

    public void Process()
    {
        if (_disposed) throw new ObjectDisposedException(nameof(FileProcessor));
        // xử lý file
    }

    // Implement Dispose Pattern đúng chuẩn
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this); // Báo GC không cần gọi finalizer
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        if (disposing)
        {
            _stream?.Dispose(); // Giải phóng managed resources
        }
        // Giải phóng unmanaged resources (nếu có)
        _disposed = true;
    }

    ~FileProcessor() => Dispose(false); // Finalizer — safety net
}

// Sử dụng
using var processor = new FileProcessor("data.txt");
processor.Process();
// Dispose() tự động gọi khi ra khỏi scope
```

---

### Q5. `ref`, `out`, `in` — các modifier truyền tham số — khác nhau thế nào?

```csharp
// ref — Tham chiếu 2 chiều: phải khởi tạo TRƯỚC khi truyền, có thể đọc + ghi
void Double(ref int value) => value *= 2;
int x = 5;
Double(ref x);
Console.WriteLine(x); // 10

// out — Đầu ra: KHÔNG cần khởi tạo trước, PHẢI gán giá trị trong method
bool TryParse(string s, out int result)
{
    if (int.TryParse(s, out result)) return true;
    result = 0; // Bắt buộc gán
    return false;
}

// in — Chỉ đọc: truyền tham chiếu để tránh copy, KHÔNG được sửa
void PrintArea(in Rectangle rect)
{
    // rect.Width = 10; // Compile error
    Console.WriteLine(rect.Width * rect.Height);
}
// Dùng cho struct lớn để tránh copy tốn kém
```

---

### Q6. Generics trong C# — kiểu tổng quát — là gì? Constraint dùng để làm gì?

```csharp
// Generic method — làm việc với bất kỳ kiểu nào
public T Max<T>(T a, T b) where T : IComparable<T>
{
    return a.CompareTo(b) > 0 ? a : b;
}

// Generic class
public class Repository<T> where T : class, IEntity, new()
{
    private readonly DbContext _db;
    public Repository(DbContext db) => _db = db;

    public async Task<T?> GetByIdAsync(int id)
        => await _db.Set<T>().FindAsync(id);

    public async Task AddAsync(T entity)
    {
        _db.Set<T>().Add(entity);
        await _db.SaveChangesAsync();
    }
}

// Các loại constraint — ràng buộc kiểu
// where T : class         → T phải là reference type
// where T : struct        → T phải là value type
// where T : new()         → T phải có constructor không tham số
// where T : SomeClass     → T phải kế thừa SomeClass
// where T : IInterface    → T phải implement interface
// where T : notnull       → T không được là null
```

---

### Q7. LINQ — Language Integrated Query — deferred execution là gì?

**Đáp án ngắn:** Deferred execution — thực thi trễ — có nghĩa là LINQ query KHÔNG thực thi ngay khi khai báo. Nó chỉ thực thi khi bạn **enumerate** (duyệt) kết quả: gọi `ToList()`, `foreach`, `Count()`, `First()`.

```csharp
var numbers = new List<int> { 1, 2, 3, 4, 5 };

// Deferred — chỉ là "công thức", chưa chạy
var query = numbers.Where(x => x > 2).Select(x => x * 2);

numbers.Add(6); // Thêm sau khi khai báo query

// Thực thi tại đây (enumerate)
var result = query.ToList(); // [6, 8, 10, 12] — bao gồm 6!

// Immediate execution — thực thi ngay
var count = numbers.Where(x => x > 2).Count(); // Thực thi ngay
```

**Nguy hiểm của deferred execution:**

```csharp
// ❌ NGUY HIỂM — Multiple enumeration
IEnumerable<User> expensiveQuery = db.Users.Where(u => u.IsActive);
var count = expensiveQuery.Count();     // Query 1 hit DB
var first = expensiveQuery.First();     // Query 2 hit DB lại!

// ✅ TỐT — Materialize một lần
var users = expensiveQuery.ToList();    // Query 1 lần
var count = users.Count;                // In-memory, nhanh
var first = users.First();              // In-memory, nhanh
```

---

### Q8. `string` trong C# — immutable nghĩa là gì? Khi nào dùng `StringBuilder`?

```csharp
// string là immutable — bất biến
string s = "hello";
s.ToUpper(); // Tạo string mới, s vẫn là "hello"
s = s.ToUpper(); // Gán lại mới thay đổi s

// string interning — tái dụng string literals
string a = "hello";
string b = "hello";
Console.WriteLine(ReferenceEquals(a, b)); // True — cùng object!

// StringBuilder — khi nối nhiều string
// ❌ Chậm — tạo N string objects
string result = "";
for (int i = 0; i < 1000; i++)
    result += i.ToString(); // 1000 allocations!

// ✅ Nhanh — O(n) thay vì O(n²)
var sb = new StringBuilder(capacity: 4096);
for (int i = 0; i < 1000; i++)
    sb.Append(i);
string result2 = sb.ToString(); // 1 allocation cuối
```

---

## Nhóm 2 — ASP.NET Core

### Q9. Middleware Pipeline — Chuỗi Xử Lý Yêu Cầu — hoạt động như thế nào?

**Đáp án ngắn:** Middleware pipeline là chuỗi các component xử lý HTTP request theo thứ tự. Mỗi middleware có thể xử lý request, gọi middleware tiếp theo bằng `next()`, và xử lý response trên đường về.

```
Request  →  [MW1] → [MW2] → [MW3] → Handler
              ↑        ↑        ↑
Response ←  [MW1] ← [MW2] ← [MW3] ← Handler
```

```csharp
// Program.cs — thứ tự QUAN TRỌNG
var app = builder.Build();

app.UseExceptionHandler("/error");   // 1. Xử lý exception toàn cục
app.UseHttpsRedirection();           // 2. Redirect HTTP → HTTPS
app.UseStaticFiles();                // 3. Serve file tĩnh
app.UseRouting();                    // 4. Định tuyến
app.UseAuthentication();             // 5. Xác thực (ai là bạn?)
app.UseAuthorization();              // 6. Phân quyền (bạn làm được gì?)
app.MapControllers();                // 7. Endpoint

// Custom middleware
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;

    public RequestTimingMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();
        
        await _next(context); // Gọi middleware tiếp theo
        
        sw.Stop();
        context.Response.Headers["X-Response-Time"] = $"{sw.ElapsedMilliseconds}ms";
    }
}
```

**Short-circuit — cắt ngắn pipeline:**

```csharp
// Middleware có thể không gọi next() để "cắt ngắn"
app.Use(async (context, next) =>
{
    if (context.Request.Headers["X-Api-Key"] != "secret")
    {
        context.Response.StatusCode = 401;
        return; // Không gọi next() → response trả về ngay
    }
    await next(context);
});
```

---

### Q10. Dependency Injection — DI — Singleton vs Scoped vs Transient là gì?

**Đáp án ngắn:** Ba vòng đời service trong DI container ASP.NET Core, quyết định khi nào object được tạo mới và khi nào tái dụng.

```csharp
// Program.cs
builder.Services.AddSingleton<ICache, RedisCache>();    // 1 instance suốt app
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();  // 1/request
builder.Services.AddTransient<IEmailSender, SmtpEmailSender>(); // Mới mỗi lần inject
```

**Khi nào dùng gì:**

| Lifetime | Tạo Mới Khi | Dùng Cho |
| -------- | ----------- | -------- |
| Singleton | App start, 1 lần duy nhất | IMemoryCache, HttpClient, config |
| Scoped | Mỗi HTTP request | DbContext, Unit of Work, current user |
| Transient | Mỗi lần resolve | Stateless service, validator, mapper |

**Lỗi "Captive Dependency" — phụ thuộc bị giam cầm:**

```csharp
// ❌ SAI — Singleton inject Scoped → DbContext bị share giữa requests!
public class CachedUserService  // Singleton
{
    private readonly AppDbContext _db; // Scoped — ĐÃ CHẾT sau request đầu tiên!
    
    public CachedUserService(AppDbContext db) => _db = db;
}

// ✅ ĐÚNG — Dùng IServiceScopeFactory để tạo scope mới khi cần
public class CachedUserService  // Singleton
{
    private readonly IServiceScopeFactory _scopeFactory;
    
    public CachedUserService(IServiceScopeFactory scopeFactory) 
        => _scopeFactory = scopeFactory;
    
    public async Task<User?> GetUserAsync(int id)
    {
        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        return await db.Users.FindAsync(id);
    }
}
```

---

### Q11. JWT — JSON Web Token — authentication flow hoạt động như thế nào?

**Flow đầy đủ:**

```
1. Client gửi POST /auth/login {username, password}
2. Server xác thực credentials
3. Server tạo JWT (Header.Payload.Signature) và trả về
4. Client lưu JWT (localStorage / httpOnly cookie)
5. Client gửi request kèm: Authorization: Bearer <token>
6. Server validate JWT signature + expiry
7. Server đọc claims từ payload, authorize
```

**Cấu trúc JWT:**

```
eyJhbGc.eyJzdWI.SflKxw...
  Header   Payload  Signature
  (Base64) (Base64) (HMAC/RSA)

Header:  { "alg": "HS256", "typ": "JWT" }
Payload: { "sub": "123", "name": "John", "role": "Admin", "exp": 1735689600 }
Signature: HMACSHA256(base64(header) + "." + base64(payload), secret)
```

**Triển khai trong ASP.NET Core:**

```csharp
// Tạo token
var token = new JwtSecurityToken(
    issuer: "https://myapp.com",
    audience: "https://myapp.com",
    claims: new[]
    {
        new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
        new Claim(ClaimTypes.Email, user.Email),
        new Claim(ClaimTypes.Role, user.Role)
    },
    expires: DateTime.UtcNow.AddHours(1),
    signingCredentials: new SigningCredentials(
        new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secret)),
        SecurityAlgorithms.HmacSha256)
);

// Validate token (Program.cs)
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = "https://myapp.com",
            ValidAudience = "https://myapp.com",
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(secret))
        };
    });
```

**Refresh Token Flow — Luồng Làm Mới Token:**

```
Access Token: thời hạn ngắn (15 phút - 1 giờ)
Refresh Token: thời hạn dài (7-30 ngày), lưu DB

Khi access token hết hạn:
1. Client gửi refresh token
2. Server verify refresh token (DB lookup)
3. Server tạo access token mới + refresh token mới
4. Trả về client
```

---

### Q12. Minimal APIs vs Controller-based APIs — khi nào chọn gì?

```csharp
// ─────── Minimal API (ASP.NET Core 6+) ───────
var app = builder.Build();

app.MapGet("/users/{id}", async (int id, IUserService svc) =>
{
    var user = await svc.GetByIdAsync(id);
    return user is null ? Results.NotFound() : Results.Ok(user);
});

app.MapPost("/users", async (CreateUserRequest req, IUserService svc) =>
{
    var user = await svc.CreateAsync(req);
    return Results.Created($"/users/{user.Id}", user);
});

// ─────── Controller-based API ───────
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _svc;
    public UsersController(IUserService svc) => _svc = svc;

    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(int id)
    {
        var user = await _svc.GetByIdAsync(id);
        return user is null ? NotFound() : Ok(user);
    }
}
```

**Khi nào chọn gì:**

| Tiêu Chí | Minimal API | Controller |
| -------- | ----------- | ---------- |
| Microservice nhỏ | ✅ Lý tưởng | Overhead |
| App lớn, nhiều endpoint | Khó maintain | ✅ Tổ chức tốt |
| Filters phức tạp | Giới hạn | ✅ Đầy đủ |
| Versioning | Cần extra config | ✅ Tích hợp sẵn |
| OpenAPI/Swagger | ✅ Tích hợp | ✅ Tích hợp |
| Unit testing | ✅ Dễ | ✅ Dễ |

---

### Q13. Filters trong ASP.NET Core — khi nào dùng thay vì Middleware?

**Đáp án ngắn:** Filters chạy trong pipeline của MVC/Minimal API, có access vào ActionContext, ModelState. Middleware chạy trước MVC, không biết về controller/action.

```
Request → [Middleware] → [Routing] → [Filters] → [Action] → [Filters] → [Middleware] → Response
```

| Loại Filter | Dùng Khi |
| ----------- | -------- |
| Authorization Filter | Kiểm tra quyền trước action |
| Resource Filter | Cache response, anti-forgery |
| Action Filter | Log, validate input/output |
| Exception Filter | Xử lý exception trong controller scope |
| Result Filter | Transform response |

```csharp
// Custom Action Filter
public class RequestLogFilter : IActionFilter
{
    private readonly ILogger _logger;
    public RequestLogFilter(ILogger<RequestLogFilter> logger) => _logger = logger;

    public void OnActionExecuting(ActionExecutingContext context)
    {
        _logger.LogInformation("Executing {Action} with args: {Args}",
            context.ActionDescriptor.DisplayName,
            JsonSerializer.Serialize(context.ActionArguments));
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        if (context.Exception != null)
            _logger.LogError(context.Exception, "Action threw exception");
    }
}

// Đăng ký global
builder.Services.AddControllers(options =>
    options.Filters.Add<RequestLogFilter>());
```

---

### Q14. Configuration & Options Pattern — `IOptions<T>` là gì?

```csharp
// appsettings.json
{
  "EmailSettings": {
    "SmtpHost": "smtp.gmail.com",
    "SmtpPort": 587,
    "FromEmail": "noreply@myapp.com"
  }
}

// Options class
public class EmailSettings
{
    public string SmtpHost { get; set; } = "";
    public int SmtpPort { get; set; }
    public string FromEmail { get; set; } = "";
}

// Đăng ký (Program.cs)
builder.Services.Configure<EmailSettings>(
    builder.Configuration.GetSection("EmailSettings"));

// Sử dụng
public class EmailService
{
    private readonly EmailSettings _settings;
    
    // IOptions<T> — giá trị được cache khi app start, không thay đổi runtime
    public EmailService(IOptions<EmailSettings> options)
        => _settings = options.Value;
    
    // IOptionsMonitor<T> — reload khi file thay đổi (hot reload)
    // IOptionsSnapshot<T> — scoped, mới mỗi request (Scoped lifetime)
}
```

---

## Nhóm 3 — Entity Framework Core

### Q15. N+1 Query Problem — vấn đề N+1 truy vấn — là gì?

**Đáp án ngắn:** N+1 xảy ra khi bạn load N entities rồi lại query N lần nữa để load related data, thay vì dùng JOIN một lần.

```csharp
// ❌ N+1 Problem — 1 query lấy orders + N query lấy customer mỗi order
var orders = await db.Orders.ToListAsync(); // Query 1: SELECT * FROM Orders (N=100)

foreach (var order in orders)
{
    // Query N lần: SELECT * FROM Customers WHERE Id = @orderId
    Console.WriteLine(order.Customer.Name); // Lazy loading! 100 queries!
}
// Tổng: 101 queries cho 100 orders!

// ✅ Eager Loading — Tải Sớm với Include()
var orders = await db.Orders
    .Include(o => o.Customer)
    .Include(o => o.OrderItems)
        .ThenInclude(i => i.Product)
    .ToListAsync(); // 1 query với JOIN

// ✅ Projection — chỉ lấy dữ liệu cần thiết
var summary = await db.Orders
    .Select(o => new OrderSummaryDto
    {
        OrderId = o.Id,
        CustomerName = o.Customer.Name, // EF tự JOIN
        TotalItems = o.OrderItems.Count
    })
    .ToListAsync();

// ✅ Split Query — cho quan hệ collection phức tạp
var orders = await db.Orders
    .Include(o => o.OrderItems)
    .AsSplitQuery() // Tách thành 2 queries riêng, tránh cartesian explosion
    .ToListAsync();
```

---

### Q16. `AsNoTracking()` — khi nào nên dùng?

**Đáp án ngắn:** Dùng `AsNoTracking()` khi chỉ **đọc** dữ liệu và không cần update. Bỏ qua change tracking → nhanh hơn 20–30%, ít memory hơn.

```csharp
// ✅ Read-only — dùng AsNoTracking
var products = await db.Products
    .AsNoTracking()  // Không theo dõi thay đổi
    .Where(p => p.IsActive)
    .ToListAsync();

// ❌ Không dùng AsNoTracking khi cần update
var user = await db.Users.FindAsync(id); // Có tracking
user.LastLoginAt = DateTime.UtcNow;
await db.SaveChangesAsync(); // EF biết user đã thay đổi → UPDATE

// ✅ Global configuration cho read-only DbContext
services.AddDbContext<ReadOnlyDbContext>(options =>
{
    options.UseSqlServer(connStr)
           .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
});
```

---

### Q17. Code-First Migration — khi xung đột migration trong team xử lý thế nào?

**Conflict xảy ra khi:** Hai dev cùng tạo migration từ cùng base migration.

```bash
# Dev A tạo migration
dotnet ef migrations add AddProductTable

# Dev B (cùng lúc) tạo migration khác  
dotnet ef migrations add AddCategoryTable

# Khi merge: hai file migration có cùng "MigrationId" parent → conflict

# Giải pháp:
# 1. Xóa migration của một người
dotnet ef migrations remove  # Xóa migration cuối

# 2. Pull code của người kia
# 3. Tạo lại migration của mình
dotnet ef migrations add AddCategoryTable

# 4. Migration mới sẽ kế thừa migration của đồng đội
```

**Best practice:**

```
• Tạo migration ngay trước khi merge, sau khi merge xong
• Review migration files trong code review
• Không để nhiều người làm DB schema cùng lúc
• Dùng feature branch + merge nhanh
```

---

### Q18. Compiled Queries — Truy Vấn Biên Dịch — là gì và khi nào dùng?

```csharp
// Mỗi lần gọi LINQ query, EF phải:
// 1. Parse expression tree
// 2. Translate sang SQL
// 3. Compile query plan

// Compiled Query — biên dịch 1 lần, cache lại
private static readonly Func<AppDbContext, int, Task<User?>> GetUserById =
    EF.CompileAsyncQuery((AppDbContext db, int id) =>
        db.Users.SingleOrDefault(u => u.Id == id));

// Sử dụng — không cần re-compile mỗi lần
public async Task<User?> FindUser(int id)
    => await GetUserById(_db, id); // Nhanh hơn ~30% lần 2 trở đi

// Phù hợp cho: hot path, query được gọi rất thường xuyên
// Không cần cho: query ít dùng, query phức tạp thay đổi runtime
```

---

## Nhóm 4 — Design Patterns & Architecture

### Q19. SOLID — 5 nguyên lý thiết kế — giải thích với ví dụ thực tế?

**S — Single Responsibility Principle — Nguyên Lý Một Trách Nhiệm:**

```csharp
// ❌ Vi phạm SRP — class làm 3 việc
public class UserService
{
    public User GetUser(int id) { /* query DB */ }
    public void SendWelcomeEmail(User user) { /* gửi email */ }
    public string GenerateReport(List<User> users) { /* tạo report */ }
}

// ✅ Đúng SRP — mỗi class một trách nhiệm
public class UserRepository { public User GetUser(int id) { } }
public class EmailService { public void SendWelcomeEmail(User user) { } }
public class UserReportService { public string GenerateReport(List<User> users) { } }
```

**O — Open/Closed Principle — Mở Để Mở Rộng, Đóng Để Sửa Đổi:**

```csharp
// ✅ Thêm payment method mới không sửa code cũ
public interface IPaymentProcessor
{
    Task ProcessAsync(PaymentRequest request);
}

public class StripePaymentProcessor : IPaymentProcessor { ... }
public class PayPalPaymentProcessor : IPaymentProcessor { ... }
public class VNPayPaymentProcessor : IPaymentProcessor { ... } // Thêm mới, không sửa cũ

public class PaymentService
{
    private readonly IPaymentProcessor _processor; // Depend on abstraction
    public PaymentService(IPaymentProcessor processor) => _processor = processor;
}
```

**L — Liskov Substitution Principle — Nguyên Lý Thay Thế Liskov:**

```csharp
// ❌ Vi phạm LSP — Square không thể thay thế Rectangle
public class Rectangle { public virtual int Width { get; set; } public virtual int Height { get; set; } }
public class Square : Rectangle
{
    // Khi set Width → cũng set Height → phá vỡ behavior của Rectangle!
    public override int Width { set { base.Width = base.Height = value; } }
}

// ✅ Đúng LSP — dùng composition thay inheritance
public interface IShape { int Area(); }
public class Rectangle : IShape { ... }
public class Square : IShape { ... } // Không kế thừa Rectangle
```

**I — Interface Segregation — Tách Biệt Giao Diện:**

```csharp
// ❌ Fat interface — nhiều method không liên quan
public interface IWorker { void Work(); void Eat(); void Sleep(); }
// Robot implement IWorker phải implement Eat() và Sleep() vô nghĩa!

// ✅ Segregated interfaces
public interface IWorkable { void Work(); }
public interface IFeedable { void Eat(); }
public class Human : IWorkable, IFeedable { ... }
public class Robot : IWorkable { ... } // Không cần implement Eat()
```

**D — Dependency Inversion — Đảo Ngược Phụ Thuộc:**

```csharp
// ❌ High-level depend on low-level
public class OrderService
{
    private readonly SqlOrderRepository _repo = new SqlOrderRepository(); // Tight coupling!
}

// ✅ Cả hai depend on abstraction
public interface IOrderRepository { Task<Order?> GetByIdAsync(int id); }
public class SqlOrderRepository : IOrderRepository { ... }
public class OrderService
{
    private readonly IOrderRepository _repo; // Depend on abstraction
    public OrderService(IOrderRepository repo) => _repo = repo;
}
```

---

### Q20. Singleton Pattern — khi nào dùng và thread safety?

```csharp
// Thread-safe Singleton với Lazy<T>
public sealed class ConnectionPool
{
    private static readonly Lazy<ConnectionPool> _instance =
        new(() => new ConnectionPool(), LazyThreadSafetyMode.ExecutionAndPublication);

    private ConnectionPool() { /* private constructor */ }

    public static ConnectionPool Instance => _instance.Value;

    public SqlConnection GetConnection() { /* ... */ }
}

// Trong ASP.NET Core — dùng DI thay Singleton pattern!
builder.Services.AddSingleton<IConnectionPool, ConnectionPool>();
// DI container lo quản lý lifetime thay bạn
```

---

### Q21. Strategy Pattern — mẫu chiến lược — ví dụ thực tế?

```csharp
// Bài toán: Sort theo nhiều tiêu chí khác nhau
public interface ISortStrategy<T>
{
    IEnumerable<T> Sort(IEnumerable<T> items);
}

public class PriceSortStrategy : ISortStrategy<Product>
{
    public IEnumerable<Product> Sort(IEnumerable<Product> items)
        => items.OrderBy(p => p.Price);
}

public class RatingSortStrategy : ISortStrategy<Product>
{
    public IEnumerable<Product> Sort(IEnumerable<Product> items)
        => items.OrderByDescending(p => p.Rating);
}

public class ProductCatalog
{
    private ISortStrategy<Product> _sortStrategy = new PriceSortStrategy();

    public void SetSortStrategy(ISortStrategy<Product> strategy)
        => _sortStrategy = strategy;

    public IEnumerable<Product> GetProducts()
        => _sortStrategy.Sort(/* load products */);
}

// Sử dụng
var catalog = new ProductCatalog();
catalog.SetSortStrategy(new RatingSortStrategy()); // Đổi strategy runtime
```

---

### Q22. Clean Architecture — 4 tầng là gì? Dependency rule?

```
┌─────────────────────────────────────┐
│  Presentation (Controllers, gRPC)   │ ← Phụ thuộc vào Application
├─────────────────────────────────────┤
│  Infrastructure (EF Core, Email)    │ ← Phụ thuộc vào Application/Domain
├─────────────────────────────────────┤
│  Application (Use Cases, CQRS)      │ ← Phụ thuộc vào Domain
├─────────────────────────────────────┤
│  Domain (Entities, Value Objects)   │ ← KHÔNG phụ thuộc ai
└─────────────────────────────────────┘

Dependency Rule: Mũi tên chỉ hướng vào trong (vào Domain)
Domain không được import bất cứ gì từ outer layers
```

```csharp
// Domain — không reference gì ngoài .NET BCL
public class Order  // Pure domain object
{
    public int Id { get; private set; }
    public Money TotalAmount { get; private set; }
    private readonly List<OrderItem> _items = new();
    
    public void AddItem(Product product, int quantity)
    {
        // Business rule: không thể thêm item cho order đã shipped
        if (Status == OrderStatus.Shipped)
            throw new DomainException("Cannot add items to shipped order");
        
        _items.Add(new OrderItem(product, quantity));
        RecalculateTotal();
    }
}

// Application — orchestrates use cases
public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, int>
{
    private readonly IOrderRepository _repo; // Interface, không phải EF
    private readonly IEventBus _eventBus;
    
    public async Task<int> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd.CustomerId);
        await _repo.AddAsync(order, ct);
        await _eventBus.PublishAsync(new OrderCreatedEvent(order.Id));
        return order.Id;
    }
}
```

---

### Q23. CQRS — Command Query Responsibility Segregation — là gì?

**Đáp án ngắn:** Tách model đọc (Query) và model ghi (Command) thành hai phần riêng biệt. Cho phép tối ưu từng phần độc lập.

```csharp
// Command — Lệnh ghi, có side effect
public record CreateProductCommand(string Name, decimal Price) : IRequest<int>;

public class CreateProductCommandHandler : IRequestHandler<CreateProductCommand, int>
{
    public async Task<int> Handle(CreateProductCommand cmd, CancellationToken ct)
    {
        var product = new Product(cmd.Name, cmd.Price);
        _db.Products.Add(product);
        await _db.SaveChangesAsync(ct);
        return product.Id;
    }
}

// Query — Truy vấn đọc, không có side effect
public record GetProductsQuery(string? SearchTerm) : IRequest<List<ProductDto>>;

public class GetProductsQueryHandler : IRequestHandler<GetProductsQuery, List<ProductDto>>
{
    public async Task<List<ProductDto>> Handle(GetProductsQuery query, CancellationToken ct)
    {
        return await _db.Products
            .AsNoTracking() // Read model — không cần tracking
            .Where(p => query.SearchTerm == null || p.Name.Contains(query.SearchTerm))
            .Select(p => new ProductDto(p.Id, p.Name, p.Price))
            .ToListAsync(ct);
    }
}

// Controller sử dụng MediatR
[HttpPost]
public async Task<IActionResult> Create(CreateProductCommand cmd)
    => Ok(await _mediator.Send(cmd));

[HttpGet]
public async Task<IActionResult> GetAll([FromQuery] string? search)
    => Ok(await _mediator.Send(new GetProductsQuery(search)));
```

---

### Q24. Microservices vs Monolith — khi nào nên chuyển?

**Bắt đầu với Monolith khi:**
- Team nhỏ (< 5 developers)
- Product chưa validated (early stage)
- Domain chưa rõ ràng

**Chuyển sang Microservices khi:**
- Team lớn, nhiều squad độc lập
- Cần scale từng phần khác nhau (payment scale x10, search scale x5)
- Deploy frequency cao, cần CI/CD độc lập
- Domain boundaries rõ ràng

**Trade-off:**

| Tiêu Chí | Monolith | Microservices |
| -------- | -------- | ------------- |
| Độ phức tạp | Thấp | Cao |
| Network calls | Không | Có (latency) |
| Transaction | ACID | Eventual consistency |
| Deploy | 1 unit | Phức tạp (K8s, service mesh) |
| Debug | Dễ | Khó (distributed tracing) |
| Team size | Nhỏ | Lớn, nhiều team |

---

## Nhóm 5 — Testing & Performance

### Q25. Unit Test vs Integration Test — khác nhau thế nào?

```
Unit Test:
• Kiểm tra một "đơn vị" (class, method) độc lập
• Mock tất cả dependencies
• Chạy rất nhanh (milliseconds)
• Không cần DB, file system, network

Integration Test:
• Kiểm tra nhiều component phối hợp
• Dùng real DB (hoặc test DB)
• Chạy chậm hơn (seconds)
• Kiểm tra end-to-end flow
```

```csharp
// Unit Test — mock dependency
public class OrderServiceTests
{
    [Fact]
    public async Task CreateOrder_ValidData_ReturnsOrderId()
    {
        // Arrange
        var mockRepo = new Mock<IOrderRepository>();
        mockRepo.Setup(r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
                .Returns(Task.CompletedTask);
        
        var service = new OrderService(mockRepo.Object);
        
        // Act
        var orderId = await service.CreateAsync(new CreateOrderDto { CustomerId = 1 });
        
        // Assert
        mockRepo.Verify(r => r.AddAsync(It.IsAny<Order>(), default), Times.Once);
        Assert.True(orderId > 0);
    }
}

// Integration Test — real HTTP + real DB
public class OrdersApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrdersApiTests(WebApplicationFactory<Program> factory)
        => _client = factory.CreateClient();

    [Fact]
    public async Task POST_CreateOrder_Returns201()
    {
        var response = await _client.PostAsJsonAsync("/api/orders",
            new { CustomerId = 1, Items = new[] { new { ProductId = 1, Qty = 2 } } });
        
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    }
}
```

---

### Q26. `Span<T>` — khi nào dùng và tại sao nhanh hơn?

**Đáp án ngắn:** `Span<T>` là "cửa sổ" vào vùng nhớ liên tiếp, không cấp phát heap. Dùng để xử lý string, array, memory mà không tạo copy.

```csharp
// Bài toán: Parse "2026-06-02" thành Date
// ❌ Cách thường — substring tạo 3 string mới
string dateStr = "2026-06-02";
int year = int.Parse(dateStr.Substring(0, 4));    // Allocation!
int month = int.Parse(dateStr.Substring(5, 2));   // Allocation!
int day = int.Parse(dateStr.Substring(8, 2));     // Allocation!

// ✅ Dùng Span<T> — zero allocation
ReadOnlySpan<char> span = dateStr.AsSpan();
int year = int.Parse(span[..4]);    // Không allocation
int month = int.Parse(span[5..7]); // Không allocation
int day = int.Parse(span[8..]);    // Không allocation

// BenchmarkDotNet kết quả điển hình:
// Substring: 200ns, 96 bytes allocated
// Span:       80ns,  0 bytes allocated
```

---

### Q27. Caching — chiến lược nào phù hợp khi nào?

| Chiến Lược | Công Cụ | Khi Dùng |
| ---------- | ------- | -------- |
| In-Memory Cache | `IMemoryCache` | Single server, data ít thay đổi |
| Distributed Cache | Redis, SQL | Multi-server, session, rate limiting |
| Response Cache | `[ResponseCache]` | Public GET endpoints |
| Output Cache | Output Caching MW | .NET 7+, granular invalidation |

```csharp
// Cache-Aside Pattern — tải từ cache trước, miss thì query DB
public async Task<Product?> GetProductAsync(int id)
{
    var cacheKey = $"product:{id}";
    
    if (_cache.TryGetValue(cacheKey, out Product? product))
        return product; // Cache hit
    
    // Cache miss — query DB
    product = await _db.Products.FindAsync(id);
    
    if (product != null)
    {
        _cache.Set(cacheKey, product, new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
            SlidingExpiration = TimeSpan.FromMinutes(2)
        });
    }
    
    return product;
}
```

---

## Nhóm 6 — Security

### Q28. OAuth 2.0 và OpenID Connect — khác nhau thế nào?

```
OAuth 2.0 — Authorization — Phân Quyền
  • Cho phép app A truy cập tài nguyên của user ở app B
  • Trả về: Access Token (để gọi API)
  • Câu hỏi trả lời: "App này được làm gì?"

OpenID Connect (OIDC) — Authentication — Xác Thực
  • Xây trên OAuth 2.0
  • Trả về thêm: ID Token (JWT chứa thông tin user)
  • Câu hỏi trả lời: "Người dùng là ai?"

Flow: Authorization Code + PKCE (Proof Key for Code Exchange)
1. App redirect user đến Identity Provider (Google, Azure AD)
2. User đăng nhập, authorize
3. Identity Provider redirect về app kèm "code"
4. App đổi "code" lấy tokens (server-to-server)
5. Dùng ID Token để biết user là ai
6. Dùng Access Token để gọi protected API
```

---

### Q29. CORS — Cross-Origin Resource Sharing — cấu hình đúng cách?

```csharp
// ❌ XẤU — cho phép tất cả origin trong production
builder.Services.AddCors(options =>
    options.AddPolicy("AllowAll", p => p.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader()));

// ✅ TỐT — chỉ cho phép origin cụ thể
builder.Services.AddCors(options =>
{
    options.AddPolicy("ProductionPolicy", policy =>
    {
        policy.WithOrigins("https://myapp.com", "https://admin.myapp.com")
              .WithMethods("GET", "POST", "PUT", "DELETE")
              .WithHeaders("Content-Type", "Authorization")
              .AllowCredentials(); // Chỉ dùng khi cần cookie/auth
    });
    
    options.AddPolicy("DevelopmentPolicy", policy =>
        policy.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader());
});

// Middleware (thứ tự quan trọng!)
app.UseCors("ProductionPolicy"); // Trước Authentication
app.UseAuthentication();
app.UseAuthorization();
```

---

### Q30. SQL Injection — tấn công tiêm SQL — và cách phòng tránh?

```csharp
// ❌ XẤU — SQL injection vulnerability
string query = $"SELECT * FROM Users WHERE Name = '{userName}'";
// userName = "'; DROP TABLE Users; --" → XÓA TABLE!

// ✅ Parameterized Query — truy vấn có tham số
using var cmd = new SqlCommand(
    "SELECT * FROM Users WHERE Name = @name", connection);
cmd.Parameters.AddWithValue("@name", userName); // Input được sanitize

// ✅ EF Core — tự động parameterize
var user = await db.Users
    .Where(u => u.Name == userName) // LINQ → parameterized SQL
    .FirstOrDefaultAsync();

// ✅ Raw SQL an toàn trong EF Core
var users = await db.Users
    .FromSqlInterpolated($"SELECT * FROM Users WHERE Name = {userName}")
    .ToListAsync();
// KHÔNG dùng FromSqlRaw với string interpolation thủ công!
```

**Các lỗ hổng bảo mật khác cần biết:**

```
XSS — Cross-Site Scripting — Kịch Bản Chéo Trang:
  → Luôn encode output HTML: HtmlEncoder.Default.Encode(userInput)
  → Dùng Content-Security-Policy header

CSRF — Cross-Site Request Forgery — Giả Mạo Yêu Cầu Chéo Trang:
  → ASP.NET Core tự bảo vệ với [ValidateAntiForgeryToken]
  → Dùng SameSite=Strict cho cookie

Mass Assignment — Gán Hàng Loạt:
  → Dùng DTO, không bind trực tiếp domain model từ request
  → Dùng [Bind(Include = "...")] hoặc FluentValidation
```

---

## 🎯 Checklist Trước Phỏng Vấn

### 1 Ngày Trước

- [ ] Đọc lại 10 câu hỏi quan trọng nhất (Q1, Q3, Q9, Q10, Q11, Q15, Q19, Q22, Q25, Q28)
- [ ] Nói to ít nhất 3 câu hỏi mà không nhìn tài liệu
- [ ] Chuẩn bị 3 câu chuyện STAR
- [ ] Review CV — chuẩn bị hỏi về mọi tech stack đã ghi

### Trong Phỏng Vấn

- [ ] Hỏi về công nghệ stack của team trước khi trả lời (adapt context)
- [ ] Nói "tôi chưa biết cụ thể nhưng approach của tôi là..." thay vì im lặng
- [ ] Đề cập trade-off thay vì chỉ một câu trả lời
- [ ] Hỏi câu hỏi thông minh cuối buổi

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
