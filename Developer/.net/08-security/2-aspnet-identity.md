# ASP.NET Core Identity — Hệ Thống Quản Lý Người Dùng

> ASP.NET Core Identity là framework built-in của Microsoft để quản lý người dùng, mật khẩu, roles và claims. Nó xử lý mọi thứ từ đăng ký, đăng nhập, quản lý role cho đến hai bước xác thực (2FA — Two-Factor Authentication).

---

## 1. Tổng Quan ASP.NET Core Identity

```
ASP.NET Core Identity gồm các thành phần chính:

┌─────────────────────────────────────────────────────────┐
│                  ASP.NET Core Identity                   │
│                                                          │
│  ┌─────────────────┐   ┌──────────────────┐             │
│  │  UserManager<T> │   │  RoleManager<T>  │             │
│  │  Quản lý user   │   │  Quản lý roles   │             │
│  └─────────────────┘   └──────────────────┘             │
│                                                          │
│  ┌─────────────────┐   ┌──────────────────┐             │
│  │  SignInManager  │   │  IPasswordHasher │             │
│  │  Xác thực user  │   │  Hash password   │             │
│  └─────────────────┘   └──────────────────┘             │
│                                                          │
│  ┌──────────────────────────────────────────┐           │
│  │  IUserStore / IRoleStore (EF Core backed)│           │
│  └──────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
                    SQL Database
               (AspNetUsers, AspNetRoles,
                AspNetUserRoles, AspNetUserClaims...)
```

---

## 2. Cài Đặt ASP.NET Core Identity

### Package

```bash
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
```

### Mở Rộng IdentityUser — Custom User Entity

```csharp
// Models/ApplicationUser.cs
public class ApplicationUser : IdentityUser
{
    // Mở rộng thêm các trường tùy chỉnh
    public string FullName { get; set; } = string.Empty;
    public string? AvatarUrl { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public bool IsActive { get; set; } = true;
    public string? Department { get; set; }
}
```

**Các thuộc tính có sẵn trong IdentityUser:**

```
Id                  — Unique identifier (string GUID)
UserName            — Tên đăng nhập
NormalizedUserName  — Uppercase để search
Email               — Email
NormalizedEmail     — Uppercase để search
EmailConfirmed      — Email đã xác nhận chưa
PasswordHash        — PBKDF2 hashed password
PhoneNumber         — Số điện thoại
PhoneNumberConfirmed
TwoFactorEnabled    — Bật 2FA chưa
LockoutEnabled      — Cho phép khóa tài khoản
LockoutEnd          — Thời điểm hết khóa
AccessFailedCount   — Số lần đăng nhập sai
SecurityStamp       — Thay đổi khi thông tin bảo mật thay đổi
ConcurrencyStamp    — Optimistic concurrency
```

### DbContext

```csharp
// Data/ApplicationDbContext.cs
public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options) { }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);  // Quan trọng: phải gọi base

        // Tùy chỉnh tên bảng (tùy chọn)
        builder.Entity<ApplicationUser>().ToTable("Users");
        builder.Entity<IdentityRole>().ToTable("Roles");
        builder.Entity<IdentityUserRole<string>>().ToTable("UserRoles");
        builder.Entity<IdentityUserClaim<string>>().ToTable("UserClaims");
        builder.Entity<IdentityUserLogin<string>>().ToTable("UserLogins");
        builder.Entity<IdentityRoleClaim<string>>().ToTable("RoleClaims");
        builder.Entity<IdentityUserToken<string>>().ToTable("UserTokens");
    }
}
```

### Đăng Ký Services trong Program.cs

```csharp
// Program.cs
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    // Password policy — chính sách mật khẩu
    options.Password.RequireDigit = true;
    options.Password.RequiredLength = 8;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireLowercase = true;
    options.Password.RequiredUniqueChars = 4;

    // Lockout policy — chính sách khóa tài khoản
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(5);
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.AllowedForNewUsers = true;

    // User policy
    options.User.RequireUniqueEmail = true;
    options.User.AllowedUserNameCharacters =
        "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789-._@+";

    // SignIn policy
    options.SignIn.RequireConfirmedEmail = false;  // true ở production
    options.SignIn.RequireConfirmedAccount = false;
})
.AddEntityFrameworkStores<ApplicationDbContext>()
.AddDefaultTokenProviders();  // Token để reset password, confirm email
```

---

## 3. UserManager — Quản Lý Người Dùng

```csharp
[ApiController]
[Route("api/users")]
public class UserController : ControllerBase
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly ITokenService _tokenService;

    public UserController(UserManager<ApplicationUser> userManager,
                          ITokenService tokenService)
    {
        _userManager = userManager;
        _tokenService = tokenService;
    }

    // Đăng Ký Người Dùng
    [HttpPost("register")]
    public async Task<IActionResult> Register([FromBody] RegisterRequest request)
    {
        // Kiểm tra email đã tồn tại chưa
        var existingUser = await _userManager.FindByEmailAsync(request.Email);
        if (existingUser != null)
        {
            return Conflict(new { message = "Email đã được sử dụng" });
        }

        var user = new ApplicationUser
        {
            UserName = request.Email,
            Email = request.Email,
            FullName = request.FullName
        };

        // CreateAsync — tự hash password với PBKDF2
        var result = await _userManager.CreateAsync(user, request.Password);
        if (!result.Succeeded)
        {
            // IdentityResult.Errors chứa danh sách lỗi cụ thể
            return BadRequest(new
            {
                errors = result.Errors.Select(e => e.Description)
            });
        }

        // Gán role mặc định
        await _userManager.AddToRoleAsync(user, "User");

        return Created($"/api/users/{user.Id}", new { userId = user.Id });
    }

    // Lấy Thông Tin User Hiện Tại
    [Authorize]
    [HttpGet("me")]
    public async Task<IActionResult> GetCurrentUser()
    {
        var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
        var user = await _userManager.FindByIdAsync(userId!);
        if (user == null) return NotFound();

        var roles = await _userManager.GetRolesAsync(user);
        var claims = await _userManager.GetClaimsAsync(user);

        return Ok(new
        {
            user.Id,
            user.Email,
            user.FullName,
            Roles = roles,
            Claims = claims.Select(c => new { c.Type, c.Value })
        });
    }

    // Thay Đổi Mật Khẩu
    [Authorize]
    [HttpPost("change-password")]
    public async Task<IActionResult> ChangePassword([FromBody] ChangePasswordRequest request)
    {
        var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
        var user = await _userManager.FindByIdAsync(userId!);
        if (user == null) return NotFound();

        var result = await _userManager.ChangePasswordAsync(
            user, request.CurrentPassword, request.NewPassword);

        if (!result.Succeeded)
            return BadRequest(new { errors = result.Errors.Select(e => e.Description) });

        // Cập nhật SecurityStamp để invalidate các sessions khác
        await _userManager.UpdateSecurityStampAsync(user);
        return NoContent();
    }
}
```

---

## 4. RoleManager — Quản Lý Roles

```csharp
[Authorize(Roles = "Admin")]
[ApiController]
[Route("api/roles")]
public class RoleController : ControllerBase
{
    private readonly RoleManager<IdentityRole> _roleManager;
    private readonly UserManager<ApplicationUser> _userManager;

    // Tạo Role Mới
    [HttpPost]
    public async Task<IActionResult> CreateRole([FromBody] string roleName)
    {
        if (await _roleManager.RoleExistsAsync(roleName))
            return Conflict($"Role '{roleName}' đã tồn tại");

        var result = await _roleManager.CreateAsync(new IdentityRole(roleName));
        if (!result.Succeeded)
            return BadRequest(result.Errors);

        return Created("", new { roleName });
    }

    // Gán Role Cho User
    [HttpPost("assign")]
    public async Task<IActionResult> AssignRole([FromBody] AssignRoleRequest request)
    {
        var user = await _userManager.FindByIdAsync(request.UserId);
        if (user == null) return NotFound("User không tồn tại");

        if (!await _roleManager.RoleExistsAsync(request.RoleName))
            return NotFound($"Role '{request.RoleName}' không tồn tại");

        if (await _userManager.IsInRoleAsync(user, request.RoleName))
            return Conflict("User đã có role này");

        var result = await _userManager.AddToRoleAsync(user, request.RoleName);
        if (!result.Succeeded)
            return BadRequest(result.Errors);

        return Ok(new { message = $"Đã gán role '{request.RoleName}' cho user" });
    }

    // Xóa Role Khỏi User
    [HttpDelete("revoke")]
    public async Task<IActionResult> RevokeRole([FromBody] AssignRoleRequest request)
    {
        var user = await _userManager.FindByIdAsync(request.UserId);
        if (user == null) return NotFound();

        await _userManager.RemoveFromRoleAsync(user, request.RoleName);
        return NoContent();
    }
}
```

---

## 5. Claims — Khẳng Định Về Người Dùng

Claims là các key-value pair chứa thông tin về người dùng, gắn vào token hoặc session.

```csharp
// Thêm Claims Cho User
public async Task AddCustomClaims(string userId)
{
    var user = await _userManager.FindByIdAsync(userId);
    if (user == null) return;

    var claims = new List<Claim>
    {
        new Claim("department", "Engineering"),
        new Claim("permission", "read:reports"),
        new Claim("permission", "write:reports"),
        new Claim("subscription", "premium")
    };

    await _userManager.AddClaimsAsync(user, claims);
}

// Xóa Claim
await _userManager.RemoveClaimAsync(user, new Claim("subscription", "basic"));

// Thêm Claims Vào JWT Khi Login
public async Task<string> GenerateTokenWithClaims(ApplicationUser user)
{
    var userClaims = await _userManager.GetClaimsAsync(user);
    var userRoles = await _userManager.GetRolesAsync(user);

    var claims = new List<Claim>
    {
        new(JwtRegisteredClaimNames.Sub, user.Id),
        new(JwtRegisteredClaimNames.Email, user.Email!)
    };

    // Thêm role claims
    claims.AddRange(userRoles.Select(r => new Claim(ClaimTypes.Role, r)));

    // Thêm custom claims từ database
    claims.AddRange(userClaims);

    // ... tạo JWT với claims này
    return GenerateJwt(claims);
}
```

---

## 6. Password Hashing — Băm Mật Khẩu

ASP.NET Core Identity dùng **PBKDF2** — Password-Based Key Derivation Function 2 — với HMAC-SHA256 mặc định.

```csharp
// Identity tự động hash khi dùng CreateAsync
await _userManager.CreateAsync(user, "MyPassword123!");

// Verify password thủ công (nếu cần)
var passwordHasher = new PasswordHasher<ApplicationUser>();
var result = passwordHasher.VerifyHashedPassword(user, user.PasswordHash!, "input");

switch (result)
{
    case PasswordVerificationResult.Success:
        // Mật khẩu đúng
        break;
    case PasswordVerificationResult.SuccessRehashNeeded:
        // Đúng nhưng cần rehash với algorithm mới hơn
        var newHash = passwordHasher.HashPassword(user, "input");
        user.PasswordHash = newHash;
        await _userManager.UpdateAsync(user);
        break;
    case PasswordVerificationResult.Failed:
        // Mật khẩu sai
        break;
}
```

---

## 7. Email Confirmation — Xác Nhận Email

```csharp
// Gửi Email Xác Nhận
[HttpPost("register")]
public async Task<IActionResult> Register([FromBody] RegisterRequest request)
{
    var user = new ApplicationUser { Email = request.Email, UserName = request.Email };
    var result = await _userManager.CreateAsync(user, request.Password);

    if (result.Succeeded)
    {
        // Tạo token xác nhận email
        var token = await _userManager.GenerateEmailConfirmationTokenAsync(user);
        var encodedToken = WebEncoders.Base64UrlEncode(Encoding.UTF8.GetBytes(token));

        // Gửi email với link xác nhận
        var confirmationLink = $"https://myapp.com/confirm-email?userId={user.Id}&token={encodedToken}";
        await _emailService.SendConfirmationEmail(user.Email!, confirmationLink);
    }

    return Ok("Đăng ký thành công, vui lòng kiểm tra email để xác nhận");
}

// Xác Nhận Email
[HttpGet("confirm-email")]
public async Task<IActionResult> ConfirmEmail([FromQuery] string userId, [FromQuery] string token)
{
    var user = await _userManager.FindByIdAsync(userId);
    if (user == null) return NotFound();

    var decodedToken = Encoding.UTF8.GetString(WebEncoders.Base64UrlDecode(token));
    var result = await _userManager.ConfirmEmailAsync(user, decodedToken);

    return result.Succeeded ? Ok("Email đã được xác nhận") : BadRequest("Token không hợp lệ");
}
```

---

## 8. Password Reset — Đặt Lại Mật Khẩu

```csharp
// Yêu Cầu Đặt Lại Mật Khẩu
[HttpPost("forgot-password")]
public async Task<IActionResult> ForgotPassword([FromBody] ForgotPasswordRequest request)
{
    var user = await _userManager.FindByEmailAsync(request.Email);

    // Không tiết lộ user có tồn tại không (security through obscurity)
    if (user == null || !await _userManager.IsEmailConfirmedAsync(user))
        return Ok("Nếu email tồn tại, bạn sẽ nhận được hướng dẫn đặt lại mật khẩu");

    var token = await _userManager.GeneratePasswordResetTokenAsync(user);
    var encodedToken = WebEncoders.Base64UrlEncode(Encoding.UTF8.GetBytes(token));

    var resetLink = $"https://myapp.com/reset-password?email={user.Email}&token={encodedToken}";
    await _emailService.SendPasswordResetEmail(user.Email!, resetLink);

    return Ok("Nếu email tồn tại, bạn sẽ nhận được hướng dẫn đặt lại mật khẩu");
}

// Đặt Lại Mật Khẩu
[HttpPost("reset-password")]
public async Task<IActionResult> ResetPassword([FromBody] ResetPasswordRequest request)
{
    var user = await _userManager.FindByEmailAsync(request.Email);
    if (user == null) return BadRequest("Yêu cầu không hợp lệ");

    var decodedToken = Encoding.UTF8.GetString(WebEncoders.Base64UrlDecode(request.Token));
    var result = await _userManager.ResetPasswordAsync(user, decodedToken, request.NewPassword);

    if (!result.Succeeded)
        return BadRequest(new { errors = result.Errors.Select(e => e.Description) });

    // Cập nhật SecurityStamp để invalidate tất cả sessions đang tồn tại
    await _userManager.UpdateSecurityStampAsync(user);

    return Ok("Mật khẩu đã được đặt lại thành công");
}
```

---

## 9. Two-Factor Authentication — 2FA — Xác Thực Hai Bước

```csharp
// Bật 2FA
[Authorize]
[HttpPost("2fa/enable")]
public async Task<IActionResult> Enable2FA()
{
    var user = await _userManager.GetUserAsync(User);
    if (user == null) return NotFound();

    // Lấy authenticator key (để dùng với Google Authenticator / Authy)
    var unformattedKey = await _userManager.GetAuthenticatorKeyAsync(user);
    if (string.IsNullOrEmpty(unformattedKey))
    {
        await _userManager.ResetAuthenticatorKeyAsync(user);
        unformattedKey = await _userManager.GetAuthenticatorKeyAsync(user);
    }

    // Format key dễ đọc: XXXX XXXX XXXX XXXX
    var formattedKey = FormatKey(unformattedKey!);

    // URI cho QR code
    var email = await _userManager.GetEmailAsync(user);
    var authenticatorUri = GenerateQrCodeUri(email!, unformattedKey!);

    return Ok(new { sharedKey = formattedKey, authenticatorUri });
}

// Xác Nhận Bật 2FA
[Authorize]
[HttpPost("2fa/verify")]
public async Task<IActionResult> Verify2FA([FromBody] string verificationCode)
{
    var user = await _userManager.GetUserAsync(User);
    if (user == null) return NotFound();

    var is2FATokenValid = await _userManager.VerifyTwoFactorTokenAsync(
        user,
        _userManager.Options.Tokens.AuthenticatorTokenProvider,
        verificationCode.Replace(" ", "").Replace("-", ""));

    if (!is2FATokenValid) return BadRequest("Mã xác thực không đúng");

    await _userManager.SetTwoFactorEnabledAsync(user, true);

    // Tạo recovery codes — mã khôi phục khi mất thiết bị
    var recoveryCodes = await _userManager.GenerateNewTwoFactorRecoveryCodesAsync(user, 10);

    return Ok(new { message = "2FA đã được bật", recoveryCodes });
}
```

---

## 10. Database Tables — Bảng Dữ Liệu

ASP.NET Core Identity tự động tạo các bảng:

```sql
AspNetUsers          -- Thông tin người dùng
AspNetRoles          -- Danh sách roles
AspNetUserRoles      -- Mapping user ↔ role (many-to-many)
AspNetUserClaims     -- Claims của từng user
AspNetRoleClaims     -- Claims của từng role
AspNetUserLogins     -- Đăng nhập bên ngoài (Google, Facebook)
AspNetUserTokens     -- Token (email confirmation, reset password, 2FA)
```

---

## 11. Câu Hỏi Phỏng Vấn

**Q: ASP.NET Core Identity lưu mật khẩu như thế nào?**

```
Dùng PBKDF2 với HMAC-SHA256:
- Salt: 128-bit random salt cho mỗi password
- Iterations: 100,000 lần (mặc định trong Identity v3)
- Output: 256-bit hash

Format: base64(version_marker + salt + hash)
→ Không thể reverse engineer
→ Rainbow table attacks thất bại do salt ngẫu nhiên
```

**Q: SecurityStamp dùng để làm gì?**

```
SecurityStamp thay đổi khi:
- Đổi mật khẩu
- Đổi email
- Bật/tắt 2FA
- Thêm/xóa external login

Cookie Authentication có thể validate SecurityStamp định kỳ
→ Nếu thay đổi → logout tất cả sessions
→ Hữu ích khi account bị compromise và cần revoke ngay
```

---

## ✅ Checklist ASP.NET Core Identity

```
✅ RequireUniqueEmail = true
✅ RequireConfirmedEmail = true (production)
✅ Lockout bật với MaxFailedAccessAttempts ≤ 10
✅ Password policy: ≥ 8 ký tự, chữ hoa, số, ký tự đặc biệt
✅ Không lưu plain text password
✅ Reset password token expire sau 24 giờ
✅ Log failed login attempts
✅ 2FA available cho user quan trọng
```

---

**Xem Tiếp:** [3-oauth2-openidconnect.md](3-oauth2-openidconnect.md) — OAuth 2.0 & OpenID Connect
