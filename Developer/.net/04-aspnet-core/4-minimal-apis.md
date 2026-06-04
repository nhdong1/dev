# 4 — Minimal APIs vs Controllers

> Minimal APIs — API Tối Giản — được giới thiệu từ ASP.NET Core 6, cho phép xây dựng API với ít boilerplate hơn, không cần `[ApiController]`, không cần class riêng. Kể từ .NET 7+, Minimal APIs đã đầy đủ tính năng để thay thế Controllers trong nhiều trường hợp.

---

## 📋 Tổng Quan Nhanh

| | Minimal API | Controller-Based API |
| - | ----------- | -------------------- |
| **Boilerplate** | Rất ít | Nhiều hơn |
| **Tổ chức code** | Cần tự tổ chức | Tự nhiên theo class |
| **Tính năng** | Đầy đủ (.NET 7+) | Đầy đủ |
| **Filters** | Endpoint Filters | Action/Exception Filters |
| **Model Binding** | Tự động | Tự động + `[ApiController]` |
| **Phù hợp** | Microservice, prototype, API nhỏ | API lớn, cần cấu trúc rõ ràng |

---

## 1. So Sánh Trực Tiếp

### Controller-Based API

```csharp
// Cần: using, [ApiController], class, ControllerBase
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;

    public UsersController(IUserService userService)
    {
        _userService = userService;
    }

    [HttpGet]
    public async Task<IActionResult> GetAll()
    {
        var users = await _userService.GetAllAsync();
        return Ok(users);
    }

    [HttpGet("{id:int}")]
    public async Task<IActionResult> GetById(int id)
    {
        var user = await _userService.GetByIdAsync(id);
        return user is null ? NotFound() : Ok(user);
    }

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateUserRequest request)
    {
        var user = await _userService.CreateAsync(request);
        return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    }
}
```

### Minimal API — Tương Đương

```csharp
// Trong Program.cs hoặc file riêng
var group = app.MapGroup("api/users");

group.MapGet("/", async (IUserService userService) =>
    await userService.GetAllAsync());

group.MapGet("/{id:int}", async (int id, IUserService userService) =>
{
    var user = await userService.GetByIdAsync(id);
    return user is null ? Results.NotFound() : Results.Ok(user);
});

group.MapPost("/", async (CreateUserRequest request, IUserService userService) =>
{
    var user = await userService.CreateAsync(request);
    return Results.CreatedAtRoute("GetUser", new { id = user.Id }, user);
});
```

---

## 2. Minimal API — Cú Pháp Cơ Bản

### HTTP Methods

```csharp
app.MapGet("/api/products", () => "GET all products");
app.MapPost("/api/products", (Product product) => Results.Created($"/api/products/1", product));
app.MapPut("/api/products/{id}", (int id, Product product) => Results.NoContent());
app.MapDelete("/api/products/{id}", (int id) => Results.NoContent());
app.MapPatch("/api/products/{id}/price", (int id, decimal price) => Results.Ok());
```

### Return Types — Kiểu Trả Về

```csharp
// IResult — kiểu trả về chung
app.MapGet("/typed", (): IResult => Results.Ok(new { message = "hello" }));

// TypedResults — type-safe hơn (khuyến nghị)
app.MapGet("/typed-safe", (): Results<Ok<User>, NotFound> =>
{
    var user = FindUser();
    return user is null ? TypedResults.NotFound() : TypedResults.Ok(user);
});

// Trả trực tiếp object — tự động serialize thành JSON
app.MapGet("/implicit-json", () => new { name = "John", age = 30 });

// Async
app.MapGet("/async", async (IUserService svc) =>
    await svc.GetAllAsync());
```

### Results Helpers

```csharp
Results.Ok(value)                    // 200 OK với JSON body
Results.Created(uri, value)          // 201 Created với Location header
Results.CreatedAtRoute(name, values, value) // 201 Created với named route
Results.NoContent()                  // 204 No Content
Results.BadRequest(errors)           // 400 Bad Request
Results.Unauthorized()               // 401 Unauthorized
Results.Forbid()                     // 403 Forbidden
Results.NotFound()                   // 404 Not Found
Results.Conflict(value)              // 409 Conflict
Results.UnprocessableEntity(errors)  // 422 Unprocessable Entity
Results.Problem(detail, title, status) // ProblemDetails response
Results.ValidationProblem(errors)    // 400 với ValidationProblemDetails
Results.File(bytes, contentType)     // File download
Results.Redirect(url)                // 302 Redirect
Results.Json(value, options)         // Custom JSON serialization
Results.Text(content, contentType)   // Plain text
```

---

## 3. Dependency Injection Trong Minimal APIs

### Inject Service Qua Parameter

```csharp
// DI tự động inject khi parameter type được đăng ký trong container
app.MapGet("/api/orders", async (
    IOrderService orderService,           // ← DI inject
    ILogger<Program> logger,              // ← DI inject
    CancellationToken cancellationToken)  // ← DI inject
=>
{
    logger.LogInformation("Getting all orders");
    return await orderService.GetAllAsync(cancellationToken);
});
```

### Inject HTTP-Specific Services

```csharp
app.MapGet("/api/me", (
    HttpContext context,           // HttpContext trực tiếp
    HttpRequest request,           // Request
    HttpResponse response,         // Response
    ClaimsPrincipal user)          // User claims
=>
{
    var userId = user.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    return Results.Ok(new { userId, ip = context.Connection.RemoteIpAddress });
});
```

### [AsParameters] — Bind Complex Object

```csharp
// Thay vì nhiều tham số rời
public record ProductSearchParams(
    [FromQuery] string? Name,
    [FromQuery] decimal? MinPrice,
    [FromQuery] decimal? MaxPrice,
    [FromQuery] int Page = 1,
    [FromQuery] int PageSize = 20);

app.MapGet("/api/products", ([AsParameters] ProductSearchParams query,
    IProductService svc) =>
    svc.SearchAsync(query));
// URL: /api/products?name=laptop&minPrice=500&page=2
```

---

## 4. Endpoint Filters — Bộ Lọc Endpoint

Endpoint Filters là tương đương của Action Filters trong Minimal APIs:

### Viết Endpoint Filter

```csharp
// Filter để validate request trước khi đến handler
public class ValidationFilter<T> : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        // Lấy argument kiểu T từ request
        var model = context.Arguments.OfType<T>().FirstOrDefault();

        if (model is null)
            return Results.BadRequest("Invalid request body");

        // Validate (ví dụ: DataAnnotations)
        var validationContext = new ValidationContext(model);
        var validationResults = new List<ValidationResult>();

        if (!Validator.TryValidateObject(model, validationContext, validationResults, true))
        {
            var errors = validationResults
                .ToDictionary(r => r.MemberNames.First(), r => r.ErrorMessage!);
            return Results.ValidationProblem(errors);
        }

        return await next(context);  // Tiếp tục đến handler
    }
}

// Sử dụng
app.MapPost("/api/users", async (CreateUserRequest request, IUserService svc) =>
{
    var user = await svc.CreateAsync(request);
    return Results.Created($"/api/users/{user.Id}", user);
})
.AddEndpointFilter<ValidationFilter<CreateUserRequest>>();
```

### Filter Inline — Nhanh Hơn

```csharp
app.MapGet("/api/admin/stats", async (IStatsService svc) =>
    await svc.GetStatsAsync())
.AddEndpointFilter(async (context, next) =>
{
    // Kiểm tra API key inline
    if (!context.HttpContext.Request.Headers.TryGetValue("X-Admin-Key", out var key)
        || key != "admin-secret")
    {
        return Results.Unauthorized();
    }
    return await next(context);
});
```

### Filter Pipeline — Nhiều Filters

```csharp
// Thứ tự filter: F1 wrap F2 wrap F3 wrap Handler
app.MapPost("/api/orders", CreateOrder)
    .AddEndpointFilter<LoggingFilter>()       // Outer — chạy đầu tiên
    .AddEndpointFilter<ValidationFilter<CreateOrderRequest>>()  // Middle
    .AddEndpointFilter<RateLimitFilter>();    // Inner — chạy cuối trước handler
```

---

## 5. Tổ Chức Minimal API Code

Minimal API dễ trở nên lộn xộn nếu để tất cả trong `Program.cs`. Có hai cách tổ chức phổ biến:

### Cách 1: Extension Methods

```csharp
// UserEndpoints.cs
public static class UserEndpoints
{
    public static IEndpointRouteBuilder MapUserEndpoints(
        this IEndpointRouteBuilder routes)
    {
        var group = routes.MapGroup("/api/users")
            .WithTags("Users")
            .RequireAuthorization();

        group.MapGet("/", GetAllUsers);
        group.MapGet("/{id:int}", GetUserById).WithName("GetUser");
        group.MapPost("/", CreateUser);
        group.MapPut("/{id:int}", UpdateUser);
        group.MapDelete("/{id:int}", DeleteUser);

        return routes;
    }

    private static async Task<IResult> GetAllUsers(
        IUserService svc, [AsParameters] PaginationQuery query)
    {
        var users = await svc.GetAllAsync(query.Page, query.PageSize);
        return TypedResults.Ok(users);
    }

    private static async Task<Results<Ok<User>, NotFound>> GetUserById(
        int id, IUserService svc)
    {
        var user = await svc.GetByIdAsync(id);
        return user is null ? TypedResults.NotFound() : TypedResults.Ok(user);
    }

    private static async Task<Results<Created<User>, ValidationProblem>> CreateUser(
        CreateUserRequest request, IUserService svc)
    {
        var user = await svc.CreateAsync(request);
        return TypedResults.Created($"/api/users/{user.Id}", user);
    }

    private static async Task<IResult> UpdateUser(
        int id, UpdateUserRequest request, IUserService svc)
    {
        await svc.UpdateAsync(id, request);
        return TypedResults.NoContent();
    }

    private static async Task<IResult> DeleteUser(int id, IUserService svc)
    {
        await svc.DeleteAsync(id);
        return TypedResults.NoContent();
    }
}

// Program.cs gọn gàng
app.MapUserEndpoints();
app.MapOrderEndpoints();
app.MapProductEndpoints();
```

### Cách 2: IEndpointDefinition Interface (Custom Pattern)

```csharp
// Interface định nghĩa contract
public interface IEndpointDefinition
{
    void DefineEndpoints(WebApplication app);
    void DefineServices(IServiceCollection services);
}

// Implement cho từng feature
public class UserEndpointDefinition : IEndpointDefinition
{
    public void DefineServices(IServiceCollection services)
    {
        services.AddScoped<IUserService, UserService>();
        services.AddScoped<IUserRepository, UserRepository>();
    }

    public void DefineEndpoints(WebApplication app)
    {
        var group = app.MapGroup("/api/users").WithTags("Users");
        group.MapGet("/", GetAll);
        group.MapGet("/{id:int}", GetById);
        // ...
    }

    private static async Task<IResult> GetAll(IUserService svc)
        => Results.Ok(await svc.GetAllAsync());

    private static async Task<IResult> GetById(int id, IUserService svc)
    {
        var user = await svc.GetByIdAsync(id);
        return user is null ? Results.NotFound() : Results.Ok(user);
    }
}

// Extension method để đăng ký tự động qua reflection
public static class EndpointDefinitionExtensions
{
    public static void AddEndpointDefinitions(
        this IServiceCollection services, params Type[] scanMarkers)
    {
        var definitions = scanMarkers
            .SelectMany(m => m.Assembly.GetTypes())
            .Where(t => typeof(IEndpointDefinition).IsAssignableFrom(t) && !t.IsAbstract)
            .Select(Activator.CreateInstance)
            .Cast<IEndpointDefinition>();

        foreach (var def in definitions)
            def.DefineServices(services);

        services.AddSingleton(definitions.ToList());
    }

    public static void UseEndpointDefinitions(this WebApplication app)
    {
        var definitions = app.Services.GetRequiredService<List<IEndpointDefinition>>();
        foreach (var def in definitions)
            def.DefineEndpoints(app);
    }
}
```

---

## 6. OpenAPI / Swagger với Minimal APIs

```csharp
app.MapGet("/api/users/{id:int}", async (int id, IUserService svc) =>
{
    var user = await svc.GetByIdAsync(id);
    return user is null ? Results.NotFound() : Results.Ok(user);
})
.WithName("GetUserById")                    // Tên operation
.WithSummary("Lấy thông tin user theo ID") // Summary cho Swagger
.WithDescription("Trả về user nếu tìm thấy, 404 nếu không có")
.WithTags("Users")                         // Nhóm trong Swagger UI
.Produces<User>(200)                        // Response type 200
.Produces(404)                              // Response type 404
.RequireAuthorization()                    // Đánh dấu cần auth trong Swagger
.WithOpenApi();                            // Kích hoạt OpenAPI metadata
```

---

## 7. Khi Nào Chọn Minimal API vs Controllers

### Dùng Minimal APIs Khi:
- Xây dựng **microservice** nhỏ, ít endpoint
- **Prototype** nhanh, demo
- API đơn giản không cần nhiều cross-cutting logic
- Muốn **performance** tốt nhất (ít overhead hơn controller)
- .NET 7+ và team quen với functional style

### Dùng Controllers Khi:
- API **lớn và phức tạp**, nhiều endpoint liên quan
- Cần **tổ chức code** rõ ràng theo class
- Team quen với MVC pattern
- Cần **Action Filters** phức tạp (Result Filters, View Rendering)
- Tích hợp với **Razor Views**
- Code base cũ đã dùng Controllers

---

## ✅ Checklist

- [ ] Viết được CRUD endpoint với Minimal API
- [ ] Dùng `TypedResults` thay vì `Results` để type-safe hơn
- [ ] Tổ chức Minimal API code theo extension methods
- [ ] Viết và áp dụng Endpoint Filter
- [ ] Hiểu sự khác biệt với Action Filters trong Controllers
- [ ] Cấu hình OpenAPI metadata cho Swagger
- [ ] Biết khi nào nên chọn Minimal API vs Controllers
