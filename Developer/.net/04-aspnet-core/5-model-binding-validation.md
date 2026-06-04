# 5 — Model Binding & Validation (Ràng Buộc và Xác Thực Dữ Liệu)

> Model Binding — Ràng Buộc Mô Hình — là quá trình ASP.NET Core tự động trích xuất dữ liệu từ HTTP request (URL, query string, body, header, form) và ánh xạ vào các tham số của action method. Validation — Xác Thực — đảm bảo dữ liệu hợp lệ trước khi xử lý.

---

## 📋 Tổng Quan Nhanh

| Nguồn Dữ Liệu | Attribute | Ví Dụ |
| ------------- | --------- | ------ |
| Route parameter | `[FromRoute]` | `{id}` trong URL path |
| Query string | `[FromQuery]` | `?page=2&size=20` |
| Request body | `[FromBody]` | JSON, XML body |
| Form field | `[FromForm]` | HTML form, multipart |
| Header | `[FromHeader]` | `X-Api-Key: abc` |
| Service (DI) | `[FromServices]` | Inject service vào action |

---

## 1. Binding Sources — Nguồn Ràng Buộc

### [FromRoute] — Từ URL Path

```csharp
// URL: PUT /api/orders/42/items/7
[HttpPut("{orderId}/items/{itemId}")]
public IActionResult UpdateOrderItem(
    [FromRoute] int orderId,   // ← từ {orderId} trong URL
    [FromRoute] int itemId,    // ← từ {itemId} trong URL
    [FromBody] UpdateItemRequest request)
{
    return NoContent();
}
```

### [FromQuery] — Từ Query String

```csharp
// URL: GET /api/products?category=electronics&minPrice=100&page=2
[HttpGet]
public IActionResult GetProducts(
    [FromQuery] string? category,
    [FromQuery] decimal? minPrice,
    [FromQuery] decimal? maxPrice,
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 20)
{
    return Ok();
}
```

### [FromBody] — Từ Request Body (JSON)

```csharp
[HttpPost]
public IActionResult CreateProduct([FromBody] CreateProductRequest request)
{
    // request được deserialize từ JSON body
    return CreatedAtAction(nameof(GetById), new { id = 1 }, request);
}

// Content-Type: application/json
// Body: { "name": "Laptop", "price": 999.99, "categoryId": 5 }
```

### [FromHeader] — Từ HTTP Header

```csharp
[HttpGet]
public IActionResult GetWithHeader(
    [FromHeader(Name = "X-Correlation-Id")] string? correlationId,
    [FromHeader(Name = "Accept-Language")] string? language)
{
    return Ok(new { correlationId, language });
}
```

### [FromForm] — Từ HTML Form

```csharp
[HttpPost("upload")]
public async Task<IActionResult> Upload(
    [FromForm] string title,
    [FromForm] IFormFile file,           // Upload một file
    [FromForm] List<IFormFile> gallery)  // Upload nhiều files
{
    if (file.Length > 0)
    {
        var path = Path.Combine("uploads", file.FileName);
        using var stream = System.IO.File.Create(path);
        await file.CopyToAsync(stream);
    }
    return Ok(new { title, fileName = file.FileName });
}
```

### [FromServices] — Inject Service Vào Action

```csharp
[HttpPost]
public async Task<IActionResult> Create(
    [FromBody] CreateOrderRequest request,
    [FromServices] IOrderService orderService,    // ← inject service
    [FromServices] IEventPublisher eventPublisher) // ← inject service
{
    var order = await orderService.CreateAsync(request);
    await eventPublisher.PublishAsync(new OrderCreatedEvent(order.Id));
    return CreatedAtAction(nameof(GetById), new { id = order.Id }, order);
}
```

---

## 2. [ApiController] — Behavior Tự Động

`[ApiController]` attribute kích hoạt một số behavior tiện lợi:

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // 1. Tự động infer [FromBody] cho complex types trong POST/PUT/PATCH
    [HttpPost]
    public IActionResult Create(CreateProductRequest request)  // không cần [FromBody]
        => Ok();

    // 2. Tự động infer [FromRoute] khi tên parameter khớp route token
    [HttpGet("{id}")]
    public IActionResult GetById(int id)  // tự biết id từ route
        => Ok();

    // 3. Tự động trả 400 với validation errors (không cần kiểm tra ModelState thủ công)
    [HttpPost("validated")]
    public IActionResult CreateValidated(CreateProductRequest request)
    {
        // Nếu request không hợp lệ → ASP.NET Core tự trả 400 ValidationProblem
        // Không cần: if (!ModelState.IsValid) return BadRequest(ModelState);
        return Ok();
    }
}
```

---

## 3. Data Annotations — Xác Thực Bằng Attribute

### Các Annotation Phổ Biến

```csharp
public class CreateProductRequest
{
    [Required(ErrorMessage = "Tên sản phẩm là bắt buộc")]
    [StringLength(100, MinimumLength = 3, ErrorMessage = "Tên phải từ 3 đến 100 ký tự")]
    public string Name { get; set; } = string.Empty;

    [Required]
    [Range(0.01, 999999.99, ErrorMessage = "Giá phải từ 0.01 đến 999,999.99")]
    public decimal Price { get; set; }

    [Range(0, int.MaxValue, ErrorMessage = "Số lượng không được âm")]
    public int Stock { get; set; }

    [Required]
    [EmailAddress(ErrorMessage = "Email không hợp lệ")]
    public string ContactEmail { get; set; } = string.Empty;

    [Url(ErrorMessage = "URL không hợp lệ")]
    public string? ImageUrl { get; set; }

    [Phone(ErrorMessage = "Số điện thoại không hợp lệ")]
    public string? PhoneNumber { get; set; }

    [RegularExpression(@"^[A-Z]{2}\d{4}$",
        ErrorMessage = "Mã sản phẩm phải có dạng: 2 chữ hoa + 4 chữ số (VD: AB1234)")]
    public string? ProductCode { get; set; }

    [Compare("ConfirmEmail", ErrorMessage = "Email xác nhận không khớp")]
    public string Email { get; set; } = string.Empty;
    public string ConfirmEmail { get; set; } = string.Empty;

    [MaxLength(500, ErrorMessage = "Mô tả tối đa 500 ký tự")]
    public string? Description { get; set; }

    [MinLength(1, ErrorMessage = "Phải có ít nhất 1 danh mục")]
    public List<int> CategoryIds { get; set; } = new();
}
```

### Custom Validation Attribute

```csharp
// Validation attribute tùy chỉnh
public class FutureDateAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(object? value, ValidationContext context)
    {
        if (value is DateTime date && date <= DateTime.UtcNow)
            return new ValidationResult("Ngày phải là trong tương lai");

        return ValidationResult.Success;
    }
}

public class ValidCurrencyCodeAttribute : ValidationAttribute
{
    private static readonly HashSet<string> ValidCodes = new(StringComparer.OrdinalIgnoreCase)
    { "VND", "USD", "EUR", "GBP", "JPY" };

    public override bool IsValid(object? value)
        => value is string code && ValidCodes.Contains(code);

    public override string FormatErrorMessage(string name)
        => $"{name} phải là mã tiền tệ hợp lệ: {string.Join(", ", ValidCodes)}";
}

// Sử dụng
public class CreateInvoiceRequest
{
    [FutureDate]
    public DateTime DueDate { get; set; }

    [ValidCurrencyCode]
    public string Currency { get; set; } = "VND";
}
```

### IValidatableObject — Validate Logic Phức Tạp

```csharp
public class DateRangeRequest : IValidatableObject
{
    [Required]
    public DateTime StartDate { get; set; }

    [Required]
    public DateTime EndDate { get; set; }

    public int MaxDays { get; set; } = 90;

    // Validate cross-field logic
    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        if (EndDate <= StartDate)
            yield return new ValidationResult(
                "Ngày kết thúc phải sau ngày bắt đầu",
                new[] { nameof(EndDate) });

        if ((EndDate - StartDate).TotalDays > MaxDays)
            yield return new ValidationResult(
                $"Khoảng thời gian tối đa là {MaxDays} ngày",
                new[] { nameof(StartDate), nameof(EndDate) });
    }
}
```

---

## 4. FluentValidation — Xác Thực Linh Hoạt

**FluentValidation** là thư viện validate mạnh mẽ hơn Data Annotations, tách logic validation ra khỏi model.

### Cài Đặt

```bash
dotnet add package FluentValidation.AspNetCore
```

### Viết Validator

```csharp
public class CreateUserRequest
{
    public string Username { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
    public DateTime DateOfBirth { get; set; }
    public string Role { get; set; } = string.Empty;
}

public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Username)
            .NotEmpty().WithMessage("Tên đăng nhập là bắt buộc")
            .MinimumLength(3).WithMessage("Tên đăng nhập phải có ít nhất 3 ký tự")
            .MaximumLength(50).WithMessage("Tên đăng nhập tối đa 50 ký tự")
            .Matches(@"^[a-zA-Z0-9_]+$").WithMessage("Chỉ được dùng chữ, số, và dấu _");

        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email là bắt buộc")
            .EmailAddress().WithMessage("Email không đúng định dạng");

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8).WithMessage("Mật khẩu phải có ít nhất 8 ký tự")
            .Matches(@"[A-Z]").WithMessage("Phải có ít nhất 1 chữ hoa")
            .Matches(@"[0-9]").WithMessage("Phải có ít nhất 1 chữ số")
            .Matches(@"[!@#$%^&*]").WithMessage("Phải có ít nhất 1 ký tự đặc biệt");

        RuleFor(x => x.DateOfBirth)
            .LessThan(DateTime.UtcNow.AddYears(-18))
            .WithMessage("Người dùng phải từ 18 tuổi trở lên");

        RuleFor(x => x.Role)
            .Must(r => new[] { "Admin", "User", "Moderator" }.Contains(r))
            .WithMessage("Role không hợp lệ. Hợp lệ: Admin, User, Moderator");
    }
}
```

### Validator Phức Tạp — Async, Inject Dependencies

```csharp
public class CreateOrderValidator : AbstractValidator<CreateOrderRequest>
{
    private readonly IProductRepository _productRepo;
    private readonly IUserRepository _userRepo;

    public CreateOrderValidator(IProductRepository productRepo, IUserRepository userRepo)
    {
        _productRepo = productRepo;
        _userRepo = userRepo;

        RuleFor(x => x.UserId)
            .NotEmpty()
            .MustAsync(UserExistsAsync).WithMessage("User không tồn tại");

        RuleFor(x => x.Items)
            .NotEmpty().WithMessage("Đơn hàng phải có ít nhất 1 sản phẩm")
            .Must(items => items.Count <= 100).WithMessage("Tối đa 100 sản phẩm/đơn hàng");

        RuleForEach(x => x.Items).SetValidator(new OrderItemValidator(_productRepo));

        // Điều kiện validate
        When(x => x.ShippingMethod == "Express", () =>
        {
            RuleFor(x => x.ShippingAddress)
                .NotEmpty().WithMessage("Express shipping yêu cầu địa chỉ giao hàng");
        });
    }

    private async Task<bool> UserExistsAsync(int userId, CancellationToken ct)
        => await _userRepo.ExistsAsync(userId, ct);
}

// Nested validator cho item trong collection
public class OrderItemValidator : AbstractValidator<OrderItem>
{
    private readonly IProductRepository _productRepo;

    public OrderItemValidator(IProductRepository productRepo)
    {
        _productRepo = productRepo;

        RuleFor(x => x.ProductId)
            .NotEmpty()
            .MustAsync((id, ct) => _productRepo.ExistsAsync(id, ct))
            .WithMessage("Sản phẩm không tồn tại");

        RuleFor(x => x.Quantity)
            .GreaterThan(0).WithMessage("Số lượng phải lớn hơn 0")
            .LessThanOrEqualTo(1000).WithMessage("Số lượng tối đa 1000");
    }
}
```

### Đăng Ký FluentValidation

```csharp
// Program.cs — .NET 6+
builder.Services.AddControllers()
    .AddFluentValidation(fv =>
    {
        fv.RegisterValidatorsFromAssemblyContaining<CreateUserRequestValidator>();
        fv.DisableDataAnnotationsValidation = true; // Tắt Data Annotations
    });

// Hoặc đăng ký từng cái
builder.Services.AddScoped<IValidator<CreateUserRequest>, CreateUserRequestValidator>();
builder.Services.AddScoped<IValidator<CreateOrderRequest>, CreateOrderValidator>();
```

### Gọi Validate Thủ Công (Khi Không Dùng Auto-Validation)

```csharp
[HttpPost]
public async Task<IActionResult> Create(
    [FromBody] CreateUserRequest request,
    [FromServices] IValidator<CreateUserRequest> validator)
{
    var validationResult = await validator.ValidateAsync(request);

    if (!validationResult.IsValid)
    {
        var errors = validationResult.Errors
            .GroupBy(e => e.PropertyName)
            .ToDictionary(
                g => g.Key,
                g => g.Select(e => e.ErrorMessage).ToArray());

        return ValidationProblem(new ValidationProblemDetails(errors));
    }

    var user = await _userService.CreateAsync(request);
    return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
}
```

---

## 5. Validation Response Format

### ProblemDetails — Định Dạng Lỗi Chuẩn (RFC 7807)

`[ApiController]` tự động trả `ValidationProblemDetails` khi model invalid:

```json
{
  "type": "https://tools.ietf.org/html/rfc7807",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "traceId": "00-abc123-00",
  "errors": {
    "Name": ["Tên sản phẩm là bắt buộc"],
    "Price": ["Giá phải từ 0.01 đến 999,999.99"],
    "Email": ["Email không đúng định dạng", "Email là bắt buộc"]
  }
}
```

### Tùy Chỉnh Validation Response

```csharp
builder.Services.AddControllers()
    .ConfigureApiBehaviorOptions(options =>
    {
        options.InvalidModelStateResponseFactory = context =>
        {
            var errors = context.ModelState
                .Where(e => e.Value?.Errors.Count > 0)
                .ToDictionary(
                    kvp => kvp.Key,
                    kvp => kvp.Value!.Errors.Select(e => e.ErrorMessage).ToArray());

            var response = new
            {
                success = false,
                message = "Dữ liệu đầu vào không hợp lệ",
                errors,
                timestamp = DateTime.UtcNow
            };

            return new BadRequestObjectResult(response);
        };
    });
```

---

## 6. So Sánh: Data Annotations vs FluentValidation

| | Data Annotations | FluentValidation |
| - | ---------------- | ---------------- |
| **Vị trí validate logic** | Trong model class | Trong Validator class riêng |
| **Separation of Concerns** | Kém (model biết về validation) | Tốt (validator tách biệt) |
| **Async validation** | Không hỗ trợ | Hỗ trợ tốt |
| **Inject dependencies** | Không thể | Có thể (qua constructor) |
| **Test độc lập** | Khó | Dễ (unit test validator) |
| **Conditional rules** | Hạn chế | Linh hoạt (`When`, `Unless`) |
| **Thông báo lỗi** | Attribute parameter | Fluent `.WithMessage()` |
| **Phù hợp** | Validate đơn giản | Validate phức tạp, real-world |

---

## ✅ Checklist

- [ ] Hiểu các binding sources và khi nào dùng mỗi loại
- [ ] Biết `[ApiController]` tự động làm gì (auto-binding, auto-validation)
- [ ] Dùng được Data Annotations cho validate cơ bản
- [ ] Viết custom `ValidationAttribute` khi cần
- [ ] Implement `IValidatableObject` cho cross-field validation
- [ ] Cài đặt và dùng FluentValidation cho validate phức tạp
- [ ] Hiểu ProblemDetails response format (RFC 7807)
- [ ] Tùy chỉnh validation error response khi cần
