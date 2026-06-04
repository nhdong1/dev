# Integration Testing — Kiểm Thử Tích Hợp ASP.NET Core

> WebApplicationFactory, TestServer, Testcontainers — test toàn bộ stack API từ HTTP request đến database.

---

## 🎯 Tại Sao Cần Integration Test?

Unit test kiểm tra từng class riêng lẻ. Integration test kiểm tra **nhiều thành phần phối hợp với nhau**:

```
Unit Test:          Service ← Mock(Repository)
Integration Test:   HTTP → Controller → Service → Repository → Database (thật)
```

**Lợi ích:**
- Phát hiện bug ở ranh giới giữa các lớp (serialization, routing, middleware)
- Xác minh middleware pipeline hoạt động đúng (auth, validation, exception handling)
- Test DB queries thực tế (không chỉ dùng in-memory)
- Tự tin deploy hơn

**Chi phí:**
- Chậm hơn unit test (cần khởi động server, DB)
- Cần cấu hình cẩn thận (connection strings, test data)

---

## 🏗️ WebApplicationFactory — Test In-Process

`WebApplicationFactory<TEntryPoint>` khởi động ứng dụng ASP.NET Core **trong cùng process** với test — không cần deploy, không cần port thật.

### Cài Đặt

```bash
dotnet add package Microsoft.AspNetCore.Mvc.Testing
dotnet add package Microsoft.EntityFrameworkCore.InMemory  # Hoặc SQLite cho test
```

```xml
<!-- MyApp.Tests.csproj -->
<ItemGroup>
  <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.*" />
  <PackageReference Include="xunit" Version="2.*" />
  <PackageReference Include="FluentAssertions" Version="6.*" />
</ItemGroup>
```

### Test Cơ Bản

```csharp
// Program.cs cần được public partial hoặc dùng assembly attribute
// Thêm vào cuối Program.cs:
// public partial class Program { }  ← cần thiết cho WebApplicationFactory

public class ProductsApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public ProductsApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetProducts_ReturnsSuccessStatusCode()
    {
        var response = await _client.GetAsync("/api/products");
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }

    [Fact]
    public async Task GetProducts_ReturnsJsonContentType()
    {
        var response = await _client.GetAsync("/api/products");
        response.Content.Headers.ContentType?.MediaType
                .Should().Be("application/json");
    }
}
```

---

## 🔧 Custom WebApplicationFactory — Tùy Chỉnh Môi Trường Test

### Override Services và Configuration

```csharp
public class TestWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureAppConfiguration((context, config) =>
        {
            // Override cấu hình cho test
            config.AddInMemoryCollection(new Dictionary<string, string?>
            {
                ["ConnectionStrings:DefaultConnection"] = "DataSource=:memory:",
                ["Jwt:Secret"] = "test-secret-key-for-testing-only-minimum-32-chars",
                ["FeatureFlags:NewCheckout"] = "true"
            });
        });

        builder.ConfigureTestServices(services =>
        {
            // Thay thế DB thật bằng SQLite in-memory
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor != null) services.Remove(descriptor);

            services.AddDbContext<AppDbContext>(options =>
                options.UseSqlite("DataSource=:memory:"));

            // Thay thế external service bằng fake
            services.AddScoped<IEmailService, FakeEmailService>();
            services.AddScoped<IPaymentGateway, FakePaymentGateway>();
        });
    }
}
```

### Khởi Tạo Database Trước Test

```csharp
public class TestWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            // Dùng SQLite in-memory
            services.AddDbContext<AppDbContext>(options =>
            {
                options.UseSqlite(_connection);
            });
        });
    }

    private SqliteConnection _connection = new("DataSource=:memory:");

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        _connection.Open(); // Giữ connection open để data không bị xóa

        builder.ConfigureTestServices(services =>
        {
            // Remove existing DbContext
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor != null) services.Remove(descriptor);

            services.AddDbContext<AppDbContext>(opts =>
                opts.UseSqlite(_connection));
        });
    }

    public AppDbContext CreateDbContext()
    {
        var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        db.Database.EnsureCreated(); // Tạo schema
        return db;
    }

    protected override void Dispose(bool disposing)
    {
        base.Dispose(disposing);
        _connection.Dispose();
    }
}
```

---

## 🧪 Test Patterns Thường Dùng

### Test CRUD API Đầy Đủ

```csharp
public class ProductsIntegrationTests
    : IClassFixture<TestWebApplicationFactory>, IAsyncLifetime
{
    private readonly TestWebApplicationFactory _factory;
    private readonly HttpClient _client;
    private AppDbContext _db;

    public ProductsIntegrationTests(TestWebApplicationFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    public async Task InitializeAsync()
    {
        _db = _factory.CreateDbContext();
        await _db.Database.EnsureCreatedAsync();
        // Seed test data
        _db.Products.AddRange(
            new Product { Id = 1, Name = "Widget", Price = 9.99m },
            new Product { Id = 2, Name = "Gadget", Price = 19.99m }
        );
        await _db.SaveChangesAsync();
    }

    public async Task DisposeAsync()
    {
        await _db.Database.EnsureDeletedAsync();
        await _db.DisposeAsync();
    }

    [Fact]
    public async Task GetAll_ReturnsAllProducts()
    {
        var response = await _client.GetAsync("/api/products");
        response.EnsureSuccessStatusCode();

        var products = await response.Content
            .ReadFromJsonAsync<List<ProductDto>>();

        products.Should().HaveCount(2);
        products.Should().Contain(p => p.Name == "Widget");
    }

    [Fact]
    public async Task GetById_ExistingProduct_ReturnsProduct()
    {
        var response = await _client.GetAsync("/api/products/1");
        response.StatusCode.Should().Be(HttpStatusCode.OK);

        var product = await response.Content.ReadFromJsonAsync<ProductDto>();
        product!.Name.Should().Be("Widget");
        product.Price.Should().Be(9.99m);
    }

    [Fact]
    public async Task GetById_NonExistingProduct_Returns404()
    {
        var response = await _client.GetAsync("/api/products/999");
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }

    [Fact]
    public async Task Create_ValidProduct_ReturnsCreatedWithLocation()
    {
        var newProduct = new { Name = "New Widget", Price = 5.99m };

        var response = await _client.PostAsJsonAsync("/api/products", newProduct);

        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();

        var created = await response.Content.ReadFromJsonAsync<ProductDto>();
        created!.Name.Should().Be("New Widget");
    }

    [Fact]
    public async Task Create_InvalidProduct_ReturnsBadRequest()
    {
        var invalidProduct = new { Name = "", Price = -1m };

        var response = await _client.PostAsJsonAsync("/api/products", invalidProduct);

        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }
}
```

### Test Authentication — Xác Thực

```csharp
public class AuthenticatedTestFactory : TestWebApplicationFactory
{
    // Tạo client với JWT token hợp lệ
    public HttpClient CreateAuthenticatedClient(string role = "User")
    {
        var client = CreateClient();
        var token = GenerateJwtToken(userId: "test-user", role: role);
        client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", token);
        return client;
    }

    private string GenerateJwtToken(string userId, string role)
    {
        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes("test-secret-key-for-testing-only-minimum-32-chars"));

        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, userId),
            new Claim(ClaimTypes.Role, role)
        };

        var token = new JwtSecurityToken(
            issuer: "test",
            audience: "test",
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: new SigningCredentials(
                key, SecurityAlgorithms.HmacSha256));

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}

// Test với authentication
public class SecureEndpointTests : IClassFixture<AuthenticatedTestFactory>
{
    private readonly AuthenticatedTestFactory _factory;

    public SecureEndpointTests(AuthenticatedTestFactory factory)
    {
        _factory = factory;
    }

    [Fact]
    public async Task AdminEndpoint_WithAdminRole_Returns200()
    {
        var client = _factory.CreateAuthenticatedClient(role: "Admin");
        var response = await client.GetAsync("/api/admin/users");
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }

    [Fact]
    public async Task AdminEndpoint_WithUserRole_Returns403()
    {
        var client = _factory.CreateAuthenticatedClient(role: "User");
        var response = await client.GetAsync("/api/admin/users");
        response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }

    [Fact]
    public async Task AdminEndpoint_Unauthenticated_Returns401()
    {
        var client = _factory.CreateClient(); // Không có token
        var response = await client.GetAsync("/api/admin/users");
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }
}
```

---

## 🐳 Testcontainers — Database Thực Trong Docker

**Testcontainers** khởi động container Docker thực trong quá trình test — dùng được database thật (SQL Server, PostgreSQL, Redis, ...) thay vì in-memory.

```bash
dotnet add package Testcontainers
dotnet add package Testcontainers.MsSql         # SQL Server
dotnet add package Testcontainers.PostgreSql    # PostgreSQL
dotnet add package Testcontainers.Redis         # Redis
```

### SQL Server Testcontainer

```csharp
public class SqlServerIntegrationTests : IAsyncLifetime
{
    private readonly MsSqlContainer _sqlContainer = new MsSqlBuilder()
        .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
        .WithPassword("YourStrong!Password")
        .Build();

    private AppDbContext _db;

    public async Task InitializeAsync()
    {
        // Khởi động container
        await _sqlContainer.StartAsync();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_sqlContainer.GetConnectionString())
            .Options;

        _db = new AppDbContext(options);
        await _db.Database.MigrateAsync(); // Chạy migrations thật
    }

    public async Task DisposeAsync()
    {
        await _db.DisposeAsync();
        await _sqlContainer.DisposeAsync();
    }

    [Fact]
    public async Task SaveOrder_WithRealDatabase_PersistsCorrectly()
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = Guid.NewGuid(),
            TotalAmount = 150m
        };

        _db.Orders.Add(order);
        await _db.SaveChangesAsync();

        // Query để xác nhận thực sự đã lưu
        var saved = await _db.Orders
            .AsNoTracking()
            .FirstOrDefaultAsync(o => o.Id == order.Id);

        saved.Should().NotBeNull();
        saved!.TotalAmount.Should().Be(150m);
    }
}
```

### PostgreSQL Testcontainer

```csharp
public class PostgreSqlFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16")
        .Build();

    public string ConnectionString => _postgres.GetConnectionString();

    public async Task InitializeAsync() => await _postgres.StartAsync();
    public async Task DisposeAsync() => await _postgres.DisposeAsync();
}

[Collection("PostgreSQL")]
public class ProductRepositoryTests : IClassFixture<PostgreSqlFixture>
{
    private readonly AppDbContext _db;

    public ProductRepositoryTests(PostgreSqlFixture fixture)
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(fixture.ConnectionString)
            .Options;
        _db = new AppDbContext(options);
        _db.Database.EnsureCreated();
    }
}
```

### Redis Testcontainer

```csharp
public class CacheServiceTests : IAsyncLifetime
{
    private readonly RedisContainer _redis = new RedisBuilder().Build();
    private IDistributedCache _cache;

    public async Task InitializeAsync()
    {
        await _redis.StartAsync();
        var services = new ServiceCollection();
        services.AddStackExchangeRedisCache(options =>
        {
            options.Configuration = _redis.GetConnectionString();
        });
        _cache = services.BuildServiceProvider()
                         .GetRequiredService<IDistributedCache>();
    }

    public async Task DisposeAsync() => await _redis.DisposeAsync();

    [Fact]
    public async Task SetAndGet_ValidKey_ReturnsCorrectValue()
    {
        await _cache.SetStringAsync("key1", "value1");
        var result = await _cache.GetStringAsync("key1");
        result.Should().Be("value1");
    }
}
```

---

## 📊 Testcontainers vs InMemory Provider

| Tiêu Chí                      | InMemory (EF Core)          | SQLite (In-Memory)              | Testcontainers                  |
| ----------------------------- | --------------------------- | ------------------------------- | ------------------------------- |
| **Tốc độ**                    | Rất nhanh                   | Nhanh                           | Chậm (khởi động Docker)         |
| **SQL chính xác**             | Không — không hỗ trợ SQL    | Hạn chế — không giống production| Giống production 100%           |
| **Stored Procedures**         | Không                       | Hạn chế                         | Hỗ trợ đầy đủ                   |
| **Migrations thực**           | Không                       | Có                              | Có                              |
| **Concurrent tests**          | Tốt                         | Cần careful với shared conn      | Tốt (mỗi test có container riêng)|
| **Phù hợp**                   | Test business logic đơn giản| Test đa số trường hợp           | Test quan trọng, CI/CD           |

---

## 🎯 Stratagem: Khi Nào Dùng Gì?

```
Unit Test (mock repository)
  → Test business logic thuần túy
  → Không cần DB, rất nhanh

WebApplicationFactory + SQLite
  → Test API endpoints, routing, middleware
  → Nhanh, không cần Docker
  → Phủ được 80% integration test scenarios

WebApplicationFactory + Testcontainers
  → Test với database production-identical
  → Cần kiểm tra SQL phức tạp, JSON columns, full-text search
  → Dùng trong CI/CD pipeline

Testcontainers thuần
  → Test Repository layer trực tiếp
  → Test migration scripts
```

---

## 🔄 Respawn — Reset Database Giữa Các Test

```bash
dotnet add package Respawn
```

```csharp
public class DatabaseCleanupFixture : IAsyncLifetime
{
    private readonly MsSqlContainer _db = new MsSqlBuilder().Build();
    private Respawner _respawner;
    private DbConnection _conn;

    public string ConnectionString => _db.GetConnectionString();

    public async Task InitializeAsync()
    {
        await _db.StartAsync();
        _conn = new SqlConnection(ConnectionString);
        await _conn.OpenAsync();

        // Migrate
        using var context = CreateContext();
        await context.Database.MigrateAsync();

        // Cấu hình Respawner để reset data (không drop schema)
        _respawner = await Respawner.CreateAsync(_conn, new RespawnerOptions
        {
            TablesToIgnore = new[] { new Table("__EFMigrationsHistory") }
        });
    }

    public async Task ResetAsync() => await _respawner.ResetAsync(_conn);

    public AppDbContext CreateContext()
    {
        var opts = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(ConnectionString).Options;
        return new AppDbContext(opts);
    }

    public async Task DisposeAsync()
    {
        await _conn.DisposeAsync();
        await _db.DisposeAsync();
    }
}

// Dùng trong test
public class OrderTests : IClassFixture<DatabaseCleanupFixture>, IAsyncLifetime
{
    private readonly DatabaseCleanupFixture _fixture;

    public OrderTests(DatabaseCleanupFixture fixture) { _fixture = fixture; }

    public Task InitializeAsync() => _fixture.ResetAsync(); // Xóa data trước mỗi test
    public Task DisposeAsync() => Task.CompletedTask;
}
```

---

## 📋 Checklist Integration Test

```
□ Dùng WebApplicationFactory thay vì deploy lên server
□ Override connection strings để tránh dùng DB production
□ Seed test data trước mỗi test, cleanup sau
□ Test cả happy path và error paths (400, 404, 401, 403)
□ Test authentication và authorization riêng
□ Dùng Testcontainers khi cần test SQL-specific behavior
□ Không share state giữa các test (dùng Respawn hoặc IAsyncLifetime)
□ Test đủ chậm — chấp nhận vài giây, không cần dưới 1ms
□ Đặt integration tests trong collection riêng để chạy song song an toàn
```

---

## 🔗 Liên Quan

- [1-unit-testing.md](./1-unit-testing.md) — Khi nào không cần integration test
- [2-mocking.md](./2-mocking.md) — Mock external services trong integration test
- [4-tdd-guide.md](./4-tdd-guide.md) — TDD với integration tests
- [05-entity-framework/8-testing-ef-core.md](../05-entity-framework/8-testing-ef-core.md) — Test EF Core cụ thể

---

*Cập nhật: 2026-06-02 | .NET 8 | Testcontainers 3.x | xUnit 2.x*
