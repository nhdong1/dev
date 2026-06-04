# Authorization Policies — Phân Quyền Dựa Trên Chính Sách

> ASP.NET Core cung cấp hệ thống phân quyền linh hoạt với ba cấp độ: Role-based — dựa trên vai trò, Claims-based — dựa trên khẳng định, và Policy-based — dựa trên chính sách tùy chỉnh. Policy-based là cách tiếp cận hiện đại và linh hoạt nhất.

---

## 1. Ba Cấp Độ Authorization

```
Đơn giản ──────────────────────────────────────── Linh hoạt

Role-based          Claims-based          Policy-based
──────────────      ─────────────         ──────────────────────
[Authorize          [Authorize            [Authorize
 (Roles="Admin")]    (Claims=...)]          (Policy="CanPublish")]
                                           ↑
                                      Kết hợp nhiều điều kiện:
                                      - Role + Claim
                                      - Resource ownership
                                      - Custom logic
                                      - External service check
```

---

## 2. Role-based Authorization — Phân Quyền Theo Vai Trò

### Cơ Bản

```csharp
// Gán role cho user (Identity)
await _userManager.AddToRoleAsync(user, "Admin");
await _userManager.AddToRoleAsync(user, "Editor");

// Thêm role vào JWT
var claims = new[]
{
    new Claim(ClaimTypes.Role, "Admin"),
    new Claim(ClaimTypes.Role, "Editor")  // Có thể nhiều roles
};

// Bảo vệ endpoint
[Authorize(Roles = "Admin")]           // Chỉ Admin
[Authorize(Roles = "Admin,Editor")]    // Admin HOẶC Editor (OR)
[Authorize(Roles = "Admin")]
[Authorize(Roles = "Editor")]          // Admin VÀ Editor (AND — stacked attributes)
public IActionResult AdminPanel() => Ok();
```

### Nhược Điểm Role-based

```csharp
// ❌ Vấn đề: Role "Manager" và "Admin" đều cần quyền này
// Phải liệt kê mọi role → brittle, khó maintain
[Authorize(Roles = "Admin,Manager,SuperAdmin,GlobalAdmin")]
public IActionResult DeletePost() => Ok();

// ✅ Giải pháp: Policy-based
[Authorize(Policy = "CanDeletePost")]
public IActionResult DeletePost() => Ok();
```

---

## 3. Claims-based Authorization — Phân Quyền Dựa Trên Claims

```csharp
// Thêm claim vào user
await _userManager.AddClaimAsync(user, new Claim("department", "Engineering"));
await _userManager.AddClaimAsync(user, new Claim("clearance", "secret"));
await _userManager.AddClaimAsync(user, new Claim("subscription", "premium"));

// Bảo vệ với claim cụ thể
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("EngineeringOnly", policy =>
        policy.RequireClaim("department", "Engineering"));

    options.AddPolicy("PremiumUser", policy =>
        policy.RequireClaim("subscription", "premium", "enterprise"));  // hoặc enterprise
});

[Authorize(Policy = "PremiumUser")]
[HttpGet("premium-features")]
public IActionResult PremiumFeatures() => Ok();
```

---

## 4. Policy-based Authorization — Phân Quyền Theo Chính Sách

### Định Nghĩa Policy Đơn Giản

```csharp
// Program.cs
builder.Services.AddAuthorization(options =>
{
    // Policy kết hợp nhiều điều kiện
    options.AddPolicy("SeniorEditor", policy =>
        policy
            .RequireRole("Editor")
            .RequireClaim("experience_years", "3", "4", "5", "6", "7", "8", "9", "10")
            .RequireAuthenticatedUser());

    options.AddPolicy("MinimumAge18", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(18)));

    options.AddPolicy("BusinessHoursOnly", policy =>
        policy.Requirements.Add(new BusinessHoursRequirement()));

    // Policy mặc định cho mọi endpoint (thay vì [Authorize] trên từng action)
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});
```

### Custom Requirement — Yêu Cầu Tùy Chỉnh

```csharp
// Yêu Cầu: User phải đủ tuổi tối thiểu
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }

    public MinimumAgeRequirement(int minimumAge)
    {
        MinimumAge = minimumAge;
    }
}

// Handler xử lý yêu cầu
public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var dateOfBirthClaim = context.User.FindFirst(c => c.Type == "date_of_birth");

        if (dateOfBirthClaim == null)
        {
            // context.Fail() — rõ ràng từ chối
            // Không gọi gì — pending (các handler khác vẫn chạy)
            return Task.CompletedTask;
        }

        var dateOfBirth = Convert.ToDateTime(dateOfBirthClaim.Value);
        var age = DateTime.Today.Year - dateOfBirth.Year;

        if (dateOfBirth > DateTime.Today.AddYears(-age)) age--;

        if (age >= requirement.MinimumAge)
        {
            context.Succeed(requirement);  // Cho phép
        }

        return Task.CompletedTask;
    }
}

// Đăng ký handler
builder.Services.AddScoped<IAuthorizationHandler, MinimumAgeHandler>();
```

### Business Hours Requirement — Ví Dụ Phức Tạp Hơn

```csharp
public class BusinessHoursRequirement : IAuthorizationRequirement
{
    public TimeSpan StartTime { get; } = new TimeSpan(9, 0, 0);   // 9:00 AM
    public TimeSpan EndTime { get; } = new TimeSpan(18, 0, 0);    // 6:00 PM
}

public class BusinessHoursHandler : AuthorizationHandler<BusinessHoursRequirement>
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public BusinessHoursHandler(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        BusinessHoursRequirement requirement)
    {
        var now = TimeOnly.FromDateTime(DateTime.Now);
        var start = TimeOnly.FromTimeSpan(requirement.StartTime);
        var end = TimeOnly.FromTimeSpan(requirement.EndTime);

        if (now >= start && now <= end)
        {
            context.Succeed(requirement);
        }
        else
        {
            // Trả lỗi rõ ràng hơn
            var httpContext = _httpContextAccessor.HttpContext;
            httpContext?.Response.Headers.Append("X-Auth-Failure", "outside-business-hours");
        }

        return Task.CompletedTask;
    }
}
```

---

## 5. Resource-based Authorization — Phân Quyền Dựa Trên Tài Nguyên

Dùng khi cần kiểm tra quyền dựa trên đặc điểm cụ thể của tài nguyên (ví dụ: user chỉ có thể xóa bài viết của chính mình).

```csharp
// Requirement
public class SameAuthorRequirement : IAuthorizationRequirement { }

// Handler
public class PostAuthorizationHandler
    : AuthorizationHandler<SameAuthorRequirement, Post>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        SameAuthorRequirement requirement,
        Post post)  // Resource là Post cụ thể
    {
        var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);

        if (post.AuthorId == userId || context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

// Đăng ký
builder.Services.AddScoped<IAuthorizationHandler, PostAuthorizationHandler>();

// Dùng trong Controller
[ApiController]
[Route("api/posts")]
public class PostController : ControllerBase
{
    private readonly IAuthorizationService _authorizationService;
    private readonly IPostRepository _postRepo;

    [Authorize]
    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        var post = await _postRepo.GetByIdAsync(id);
        if (post == null) return NotFound();

        // Kiểm tra quyền với tài nguyên cụ thể
        var authResult = await _authorizationService.AuthorizeAsync(
            User, post, new SameAuthorRequirement());

        if (!authResult.Succeeded)
            return Forbid();  // 403 Forbidden

        await _postRepo.DeleteAsync(post);
        return NoContent();
    }
}
```

---

## 6. IAuthorizationService — Phân Quyền Trong Code

Đôi khi cần kiểm tra quyền trong service layer, không chỉ ở controller:

```csharp
public class PostService
{
    private readonly IAuthorizationService _authorizationService;
    private readonly IHttpContextAccessor _httpContextAccessor;

    public async Task<bool> CanUserEditPost(Post post)
    {
        var user = _httpContextAccessor.HttpContext?.User;
        if (user == null) return false;

        var result = await _authorizationService.AuthorizeAsync(
            user, post, new SameAuthorRequirement());

        return result.Succeeded;
    }

    // Hoặc kiểm tra theo policy name
    public async Task<bool> CanAccessPremiumContent(ClaimsPrincipal user)
    {
        var result = await _authorizationService.AuthorizeAsync(
            user, null, "PremiumUser");

        return result.Succeeded;
    }
}
```

---

## 7. Authorize trên Razor Pages và Minimal APIs

### Minimal API

```csharp
// Bảo vệ nhóm routes
var adminGroup = app.MapGroup("/admin")
    .RequireAuthorization("AdminPolicy");

adminGroup.MapGet("/users", () => "Admin only");
adminGroup.MapGet("/dashboard", () => "Admin dashboard");

// Bảo vệ từng route
app.MapDelete("/posts/{id}", async (int id, IAuthorizationService auth,
    ClaimsPrincipal user, IPostRepository repo) =>
{
    var post = await repo.GetByIdAsync(id);
    if (post == null) return Results.NotFound();

    var authResult = await auth.AuthorizeAsync(user, post, new SameAuthorRequirement());
    if (!authResult.Succeeded) return Results.Forbid();

    await repo.DeleteAsync(post);
    return Results.NoContent();
})
.RequireAuthorization();

// Cho phép anonymous (override FallbackPolicy)
app.MapGet("/public", () => "Public content")
    .AllowAnonymous();
```

---

## 8. Authorization Middleware Pipeline

```csharp
// Thứ tự quan trọng trong middleware pipeline
app.UseRouting();
app.UseAuthentication();   // 1. Xác thực: ai đang gọi?
app.UseAuthorization();    // 2. Phân quyền: họ được làm gì?
app.MapControllers();
```

---

## 9. Global Authorization — Áp Dụng Toàn Cục

```csharp
// Cách 1: FallbackPolicy — yêu cầu xác thực cho mọi endpoint
builder.Services.AddAuthorization(options =>
{
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});
// Sau đó thêm .AllowAnonymous() cho các endpoint public

// Cách 2: Filter toàn cục (Controllers)
builder.Services.AddControllers(options =>
{
    var policy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
    options.Filters.Add(new AuthorizeFilter(policy));
});
```

---

## 10. Authorization Response — Phản Hồi Phân Quyền

```
Kết Quả              HTTP Status      Khi Nào
───────────────       ────────────     ─────────────────────────────
401 Unauthorized      Chưa xác thực   User không có token / token hết hạn
403 Forbidden         Đã xác thực,    User đã login nhưng không có quyền
                      không có quyền
```

```csharp
// Tùy chỉnh response cho 401 và 403
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Events = new JwtBearerEvents
        {
            // 401: Token thiếu hoặc không hợp lệ
            OnChallenge = context =>
            {
                context.HandleResponse();
                context.Response.StatusCode = 401;
                context.Response.ContentType = "application/json";
                return context.Response.WriteAsJsonAsync(new
                {
                    error = "unauthorized",
                    message = "Bạn cần đăng nhập để thực hiện thao tác này"
                });
            },

            // 403: Đã xác thực nhưng không có quyền
            OnForbidden = context =>
            {
                context.Response.StatusCode = 403;
                context.Response.ContentType = "application/json";
                return context.Response.WriteAsJsonAsync(new
                {
                    error = "forbidden",
                    message = "Bạn không có quyền thực hiện thao tác này"
                });
            }
        };
    });
```

---

## 11. Câu Hỏi Phỏng Vấn

**Q: Khi nào dùng Role-based vs Policy-based?**

```
Role-based:
  ✅ Hệ thống đơn giản với ít roles rõ ràng
  ✅ Roles không thay đổi thường xuyên
  ❌ Khi có nhiều roles cùng quyền
  ❌ Khi logic phức tạp hơn chỉ là "có role này"

Policy-based:
  ✅ Logic phân quyền phức tạp
  ✅ Cần kết hợp nhiều điều kiện
  ✅ Quyền dựa trên thuộc tính của resource
  ✅ Quyền thay đổi theo context (giờ làm việc, địa điểm)
  ✅ Dễ unit test (test handler riêng lẻ)
```

**Q: Sự khác nhau giữa 401 và 403?**

```
401 Unauthorized (thực ra là Unauthenticated):
  → Không có token hoặc token không hợp lệ
  → "Tôi không biết bạn là ai"
  → Nên đăng nhập để tiếp tục

403 Forbidden:
  → Đã xác thực, nhưng không có đủ quyền
  → "Tôi biết bạn là ai, nhưng bạn không được vào đây"
  → Đăng nhập với account khác hoặc xin thêm quyền
```

---

## ✅ Checklist Authorization

```
✅ Mọi endpoint nhạy cảm đều có [Authorize]
✅ Dùng Policy thay vì magic strings cho Roles
✅ Resource-based auth cho CRUD operations
✅ 401 vs 403 response đúng ý nghĩa
✅ FallbackPolicy bật (secure by default)
✅ Unit test Authorization handlers
✅ Không hardcode user ID trong authorization logic
✅ Log unauthorized access attempts
```

---

**Xem Tiếp:** [5-data-protection.md](5-data-protection.md) — ASP.NET Core Data Protection API
