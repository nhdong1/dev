# 3 — Routing (Định Tuyến)

> Routing — Định Tuyến — là cơ chế ánh xạ (map) một HTTP request đến đúng handler (controller action hoặc endpoint). ASP.NET Core hỗ trợ hai hệ thống routing: **Conventional Routing** (định tuyến quy ước) và **Attribute Routing** (định tuyến thuộc tính), trong đó Attribute Routing được khuyến nghị cho Web API.

---

## 📋 Tổng Quan Nhanh

| Khái Niệm | Giải Thích |
| --------- | ---------- |
| Route Template | Khuôn mẫu URL như `api/users/{id}` |
| Route Parameter | Tham số trong URL `{id}`, `{name}` |
| Route Constraint | Ràng buộc kiểu dữ liệu: `{id:int}`, `{name:minlength(3)}` |
| Attribute Routing | Định nghĩa route bằng `[Route]`, `[HttpGet]`... trực tiếp trên method |
| Conventional Routing | Route pattern tập trung trong `Program.cs` |
| Endpoint Routing | Hệ thống routing hiện đại từ ASP.NET Core 3.0+ |
| Route Priority | Thứ tự ưu tiên khi nhiều route khớp |

---

## 1. Attribute Routing — Định Tuyến Thuộc Tính

Cách được khuyến nghị cho Web API — khai báo route ngay trên controller/action.

### Khai Báo Route Cơ Bản

```csharp
[ApiController]
[Route("api/[controller]")]          // [controller] = tên class bỏ "Controller"
public class UsersController : ControllerBase
{
    // GET api/users
    [HttpGet]
    public IActionResult GetAll() => Ok(new[] { "user1", "user2" });

    // GET api/users/42
    [HttpGet("{id}")]
    public IActionResult GetById(int id) => Ok($"User {id}");

    // GET api/users/42/orders
    [HttpGet("{id}/orders")]
    public IActionResult GetOrders(int id) => Ok($"Orders of user {id}");

    // POST api/users
    [HttpPost]
    public IActionResult Create([FromBody] CreateUserRequest request)
        => CreatedAtAction(nameof(GetById), new { id = 1 }, request);

    // PUT api/users/42
    [HttpPut("{id}")]
    public IActionResult Update(int id, [FromBody] UpdateUserRequest request)
        => NoContent();

    // DELETE api/users/42
    [HttpDelete("{id}")]
    public IActionResult Delete(int id) => NoContent();
}
```

### Override Route Trên Controller

```csharp
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]  // Route có version
public class ProductsController : ControllerBase
{
    // GET api/v1/products
    [HttpGet]
    public IActionResult GetAll() => Ok();

    // Route hoàn toàn tùy chỉnh (bắt đầu bằng / để override route gốc)
    [HttpGet("/catalog/featured")]  // ← / ở đầu: route tuyệt đối
    public IActionResult GetFeatured() => Ok();
    // URL: GET /catalog/featured (không có api/v1/products prefix)
}
```

---

## 2. Route Parameters — Tham Số Route

### Tham Số Bắt Buộc

```csharp
// {id} — bắt buộc, phải có trong URL
[HttpGet("{id}")]
public IActionResult Get(int id) { ... }
// Match: /api/users/42
// Không match: /api/users  (thiếu id)
```

### Tham Số Tùy Chọn

```csharp
// {id?} — tùy chọn, có thể bỏ qua
[HttpGet("{id?}")]
public IActionResult Get(int? id = null)
{
    return id.HasValue ? Ok($"User {id}") : Ok("All users");
}
// Match: /api/users/42
// Match: /api/users
```

### Tham Số Mặc Định

```csharp
// Dùng default value trong tham số C#
[HttpGet("page/{pageNumber=1}")]
public IActionResult GetPage(int pageNumber)
{
    return Ok($"Page {pageNumber}");
}
// Match: /api/users/page/3  → pageNumber = 3
// Match: /api/users/page    → pageNumber = 1
```

### Catch-All Parameter — Bắt Tất Cả Phần Còn Lại

```csharp
// {**slug} — bắt tất cả phần còn lại của URL kể cả dấu /
[HttpGet("files/{**filePath}")]
public IActionResult GetFile(string filePath)
{
    return Ok($"File: {filePath}");
}
// Match: /api/files/images/2024/photo.jpg → filePath = "images/2024/photo.jpg"
```

---

## 3. Route Constraints — Ràng Buộc Route

Ràng buộc type và format của tham số route. Nếu không khớp, route bị bỏ qua (không throw 400, mà trả 404).

### Constraints Phổ Biến

```csharp
// `:int` — chỉ nhận số nguyên
[HttpGet("{id:int}")]
public IActionResult GetById(int id) { ... }
// Match:     /api/users/42
// Không match: /api/users/abc  → route này bị bỏ, tìm route khác

// `:guid` — chỉ nhận GUID
[HttpGet("{id:guid}")]
public IActionResult GetByGuid(Guid id) { ... }
// Match:     /api/users/3fa85f64-5717-4562-b3fc-2c963f66afa6

// `:minlength(n)` — string tối thiểu n ký tự
[HttpGet("search/{term:minlength(3)}")]
public IActionResult Search(string term) { ... }
// Match:     /api/search/cat   (3 ký tự)
// Không match: /api/search/ab (2 ký tự)

// `:range(min,max)` — số trong khoảng
[HttpGet("page/{page:range(1,100)}")]
public IActionResult GetPage(int page) { ... }

// `:regex(pattern)` — khớp regex
[HttpGet("{username:regex(^[a-zA-Z0-9_]+$)}")]
public IActionResult GetByUsername(string username) { ... }
```

### Bảng Constraints Đầy Đủ

| Constraint | Mô Tả | Ví Dụ |
| ---------- | ------ | ------ |
| `int` | Số nguyên 32-bit | `{id:int}` |
| `long` | Số nguyên 64-bit | `{id:long}` |
| `decimal` | Số thập phân | `{price:decimal}` |
| `double` | Số thực | `{lat:double}` |
| `float` | Số thực 32-bit | `{weight:float}` |
| `bool` | true/false | `{active:bool}` |
| `guid` | GUID | `{id:guid}` |
| `datetime` | Ngày giờ | `{date:datetime}` |
| `alpha` | Chỉ chữ cái | `{name:alpha}` |
| `minlength(n)` | Tối thiểu n ký tự | `{code:minlength(5)}` |
| `maxlength(n)` | Tối đa n ký tự | `{tag:maxlength(20)}` |
| `length(n)` | Đúng n ký tự | `{zip:length(5)}` |
| `length(n,m)` | Từ n đến m ký tự | `{name:length(2,50)}` |
| `min(n)` | Tối thiểu n | `{age:min(18)}` |
| `max(n)` | Tối đa n | `{rating:max(5)}` |
| `range(n,m)` | Trong khoảng n-m | `{page:range(1,100)}` |
| `regex(expr)` | Khớp regex | `{code:regex(^\\d{{5}}$)}` |

### Kết Hợp Nhiều Constraints

```csharp
// Kết hợp: int VÀ trong range 1–1000
[HttpGet("{id:int:range(1,1000)}")]
public IActionResult Get(int id) { ... }

// Kết hợp: string, alpha, minlength 3
[HttpGet("category/{name:alpha:minlength(3)}")]
public IActionResult GetByCategory(string name) { ... }
```

---

## 4. Query String Parameters — Tham Số Query

```csharp
// Query string: /api/users?page=2&pageSize=20&search=john
[HttpGet]
public IActionResult GetUsers(
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 20,
    [FromQuery] string? search = null,
    [FromQuery] bool activeOnly = false)
{
    return Ok(new { page, pageSize, search, activeOnly });
}
```

### Bind Complex Query Object

```csharp
// Thay vì nhiều tham số rời, dùng class
public class UserSearchQuery
{
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 20;
    public string? Search { get; set; }
    public bool ActiveOnly { get; set; }
    public string? SortBy { get; set; }
    public string SortOrder { get; set; } = "asc";
}

[HttpGet]
public IActionResult GetUsers([FromQuery] UserSearchQuery query)
{
    return Ok(query);
}
// URL: /api/users?page=2&search=john&sortBy=name&sortOrder=desc
```

---

## 5. Route Groups — Nhóm Routes (ASP.NET Core 7+)

Dùng cho Minimal APIs để nhóm và chia sẻ prefix/middleware:

```csharp
// Tất cả routes trong group đều có prefix "api/v1/users"
var usersGroup = app.MapGroup("api/v1/users")
    .RequireAuthorization()                  // Áp dụng auth cho cả group
    .WithTags("Users")                       // Swagger tag
    .WithOpenApi();

usersGroup.MapGet("/", GetAllUsers);         // GET api/v1/users
usersGroup.MapGet("/{id:int}", GetUser);     // GET api/v1/users/42
usersGroup.MapPost("/", CreateUser);         // POST api/v1/users
usersGroup.MapPut("/{id:int}", UpdateUser);  // PUT api/v1/users/42
usersGroup.MapDelete("/{id:int}", DeleteUser); // DELETE api/v1/users/42
```

---

## 6. Conventional Routing — Định Tuyến Quy Ước

Thường dùng cho MVC (Razor Pages), ít dùng cho API:

```csharp
// Program.cs — định tuyến tập trung
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

// Ví dụ match:
// /Products/Details/5  → ProductsController.Details(5)
// /Orders              → OrdersController.Index()
// /                    → HomeController.Index()
```

### Tại Sao API Nên Dùng Attribute Routing

| | Conventional Routing | Attribute Routing |
| - | -------------------- | ----------------- |
| Cấu hình | Tập trung, 1 chỗ | Phân tán, trên từng method |
| Linh hoạt | Thấp | Cao |
| RESTful URLs | Khó | Dễ |
| Phù hợp | MVC, Razor Pages | Web API |
| Refactoring | Dễ thay đổi tập trung | Cần sửa nhiều nơi |

---

## 7. Route Naming và Link Generation

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    // Đặt tên route để tham chiếu từ nơi khác
    [HttpGet("{id}", Name = "GetOrder")]
    public IActionResult GetOrder(int id) => Ok(new Order { Id = id });

    [HttpPost]
    public IActionResult CreateOrder([FromBody] CreateOrderRequest request)
    {
        var newOrder = new Order { Id = 42 };

        // CreatedAtAction — generate URL từ action name
        return CreatedAtAction(
            nameof(GetOrder),          // tên action
            new { id = newOrder.Id },  // route values
            newOrder);                 // response body
        // Location header: https://api.example.com/api/orders/42
    }

    [HttpPost("bulk")]
    public IActionResult BulkCreate([FromBody] List<CreateOrderRequest> requests)
    {
        // CreatedAtRoute — generate URL từ route name
        return CreatedAtRoute(
            "GetOrder",
            new { id = 1 },
            new { created = requests.Count });
    }
}
```

---

## 8. Custom Route Constraint

```csharp
// Tạo constraint tùy chỉnh
public class SlugRouteConstraint : IRouteConstraint
{
    private static readonly Regex SlugRegex = new(@"^[a-z0-9]+(?:-[a-z0-9]+)*$",
        RegexOptions.Compiled | RegexOptions.IgnoreCase);

    public bool Match(
        HttpContext? httpContext,
        IRouter? route,
        string routeKey,
        RouteValueDictionary values,
        RouteDirection routeDirection)
    {
        if (!values.TryGetValue(routeKey, out var value) || value is null)
            return false;

        return SlugRegex.IsMatch(value.ToString()!);
    }
}

// Đăng ký
builder.Services.Configure<RouteOptions>(options =>
{
    options.ConstraintMap.Add("slug", typeof(SlugRouteConstraint));
});

// Sử dụng
[HttpGet("posts/{slug:slug}")]
public IActionResult GetPost(string slug) => Ok($"Post: {slug}");
// Match:     /api/posts/my-awesome-post
// Không match: /api/posts/My Awesome Post!
```

---

## ⚠️ Các Lỗi Phổ Biến

| Lỗi | Nguyên Nhân | Cách Sửa |
| --- | ----------- | --------- |
| `AmbiguousMatchException` | Hai route cùng khớp | Thêm constraint hoặc đổi URL pattern |
| Route không khớp | Constraint bị fail | Kiểm tra kiểu dữ liệu, thử bỏ constraint để debug |
| `[Route]` trên class bị override | Dùng route bắt đầu bằng `/` trên method | Đó là cố ý — xem lại có muốn absolute route không |
| Query string không bind | Thiếu `[FromQuery]` | Thêm `[FromQuery]` attribute |
| Order: route cụ thể vs generic | Route generic khớp trước route cụ thể | Đặt route cụ thể hơn trước trong code |

---

## ✅ Checklist

- [ ] Hiểu sự khác biệt Attribute Routing vs Conventional Routing
- [ ] Biết các route constraints phổ biến và cách kết hợp
- [ ] Dùng `[FromQuery]` và `[FromRoute]` đúng cách
- [ ] Bind complex query object thay vì nhiều tham số rời
- [ ] Dùng `CreatedAtAction` để trả `201 Created` với `Location` header đúng
- [ ] Biết khi nào route bắt đầu bằng `/` ghi đè route của controller
