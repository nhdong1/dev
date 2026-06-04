# 8 — Testing EF Core: InMemory, SQLite & Mocking DbContext

> Test code sử dụng EF Core đúng cách là kỹ năng quan trọng. Có nhiều chiến lược khác nhau, mỗi cái có trade-off riêng.

---

## 📌 Các Chiến Lược Test EF Core

| Chiến Lược | Tốc Độ | Độ Tin Cậy | Khi Nào Dùng |
|-----------|--------|-----------|--------------|
| **Mocking DbContext** | ⭐⭐⭐⭐⭐ | ⭐⭐ | Unit test logic thuần túy |
| **InMemory Provider** | ⭐⭐⭐⭐ | ⭐⭐⭐ | Unit test đơn giản, không cần SQL |
| **SQLite In-Memory** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Test với SQL thật, nhanh |
| **SQL Server (TestContainers)** | ⭐⭐ | ⭐⭐⭐⭐⭐ | Integration test thật sự |

---

## 1. InMemory Provider — Nhà Cung Cấp Trong Bộ Nhớ

```bash
dotnet add package Microsoft.EntityFrameworkCore.InMemory
```

```csharp
public class ProductServiceTests
{
    private AppDbContext CreateContext(string dbName = "TestDb")
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(databaseName: dbName) // Mỗi test 1 DB riêng
            .Options;

        var context = new AppDbContext(options);

        // Seed data cho test
        context.Categories.AddRange(
            new Category { Id = 1, Name = "Electronics" },
            new Category { Id = 2, Name = "Books" });
        context.Products.AddRange(
            new Product { Id = 1, Name = "Laptop", Price = 1000, CategoryId = 1, IsActive = true },
            new Product { Id = 2, Name = "Phone", Price = 500, CategoryId = 1, IsActive = true },
            new Product { Id = 3, Name = "Novel", Price = 20, CategoryId = 2, IsActive = false });
        context.SaveChanges();

        return context;
    }

    [Fact]
    public async Task GetActiveProducts_ReturnsOnlyActiveProducts()
    {
        // Arrange
        await using var context = CreateContext("Test_GetActive");
        var service = new ProductService(context);

        // Act
        var result = await service.GetActiveProductsAsync();

        // Assert
        Assert.Equal(2, result.Count);
        Assert.All(result, p => Assert.True(p.IsActive));
    }

    [Fact]
    public async Task AddProduct_SavesCorrectly()
    {
        await using var context = CreateContext("Test_Add");
        var service = new ProductService(context);

        var newProduct = new Product
        {
            Name = "Tablet",
            Price = 300,
            CategoryId = 1,
            IsActive = true
        };

        // Act
        await service.AddProductAsync(newProduct);

        // Assert — Kiểm tra với context mới (tránh cache)
        await using var verifyContext = CreateContext("Test_Add"); // Cùng DB name!
        var saved = await verifyContext.Products
            .FirstOrDefaultAsync(p => p.Name == "Tablet");

        Assert.NotNull(saved);
        Assert.Equal(300, saved.Price);
    }
}
```

### Hạn Chế Của InMemory Provider

```csharp
// ❌ InMemory không kiểm tra SQL constraints
// Các tính năng này không hoạt động:
// - Required/MaxLength không được enforce
// - Unique constraints bị bỏ qua
// - Cascade delete không giống SQL
// - Raw SQL queries không chạy được
// - FromSqlRaw → throw exception!

// Ví dụ: Điều này PASS với InMemory nhưng FAIL với real DB!
context.Products.Add(new Product { Name = null! }); // Violation IsRequired
await context.SaveChangesAsync(); // InMemory không báo lỗi!
```

---

## 2. SQLite In-Memory — Tốt Nhất Cho Unit Tests

SQLite thực sự execute SQL, nhưng dữ liệu chỉ trong bộ nhớ:

```bash
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
```

```csharp
public class SqliteTestBase : IDisposable
{
    protected readonly AppDbContext Context;
    private readonly SqliteConnection _connection;

    public SqliteTestBase()
    {
        // Tạo connection và giữ mở (InMemory SQLite xóa DB khi connection đóng)
        _connection = new SqliteConnection("DataSource=:memory:");
        _connection.Open();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlite(_connection)
            .Options;

        Context = new AppDbContext(options);

        // Tạo schema từ migrations (hoặc EnsureCreated)
        Context.Database.EnsureCreated();
    }

    public void Dispose()
    {
        Context.Dispose();
        _connection.Dispose();
    }
}

public class ProductRepositoryTests : SqliteTestBase
{
    [Fact]
    public async Task GetByCategory_ReturnsCorrectProducts()
    {
        // Arrange
        Context.Categories.Add(new Category { Id = 1, Name = "Electronics" });
        Context.Products.AddRange(
            new Product { Name = "Laptop", Price = 1000, CategoryId = 1, IsActive = true },
            new Product { Name = "Book", Price = 20, CategoryId = 2, IsActive = true });
        await Context.SaveChangesAsync();

        // Act
        var electronics = await Context.Products
            .Where(p => p.CategoryId == 1)
            .ToListAsync();

        // Assert
        Assert.Single(electronics);
        Assert.Equal("Laptop", electronics[0].Name);
    }

    [Fact]
    public async Task SaveProduct_EnforcesNameRequired()
    {
        // Arrange — SQLite thực sự kiểm tra constraint
        Context.Products.Add(new Product { Name = null!, Price = 100, CategoryId = 1 });

        // Act & Assert — SQLite có thể throw DbUpdateException
        await Assert.ThrowsAsync<DbUpdateException>(
            () => Context.SaveChangesAsync());
    }
}
```

---

## 3. TestContainers — Real Database Trong Docker

Cho integration tests thật sự với SQL Server, PostgreSQL:

```bash
dotnet add package Testcontainers.MsSql
# Yêu cầu Docker đang chạy
```

```csharp
public class IntegrationTestBase : IAsyncLifetime
{
    private readonly MsSqlContainer _sqlContainer;
    protected AppDbContext Context { get; private set; } = null!;

    public IntegrationTestBase()
    {
        _sqlContainer = new MsSqlBuilder()
            .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
            .WithPassword("YourStrong@Password")
            .Build();
    }

    public async Task InitializeAsync()
    {
        await _sqlContainer.StartAsync(); // Khởi động container Docker

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_sqlContainer.GetConnectionString())
            .Options;

        Context = new AppDbContext(options);
        await Context.Database.MigrateAsync(); // Chạy migrations thật
    }

    public async Task DisposeAsync()
    {
        await Context.DisposeAsync();
        await _sqlContainer.DisposeAsync(); // Dừng container
    }
}

[Collection("Integration Tests")]
public class OrderIntegrationTests : IntegrationTestBase
{
    [Fact]
    public async Task CreateOrder_WithTransaction_IsAtomic()
    {
        // Arrange
        var customer = new Customer { Name = "Test Customer", Email = "test@test.com" };
        Context.Customers.Add(customer);
        await Context.SaveChangesAsync();

        // Act
        var order = new Order
        {
            CustomerId = customer.Id,
            Status = OrderStatus.Pending,
            CreatedAt = DateTime.UtcNow
        };
        Context.Orders.Add(order);
        await Context.SaveChangesAsync();

        // Assert — SQL Server thật, migration thật, constraint thật
        var saved = await Context.Orders
            .Include(o => o.Customer)
            .FirstAsync(o => o.Id == order.Id);

        Assert.Equal("Test Customer", saved.Customer.Name);
    }
}
```

---

## 4. Mocking DbContext — Unit Test Thuần Túy

Khi muốn test logic không liên quan đến database queries:

```bash
dotnet add package Moq
```

### Cách 1: Mock Trực Tiếp DbContext (Hạn Chế)

```csharp
// ❌ Không thể mock DbContext trực tiếp dễ dàng
// DbSet<T>.Add(), SaveChangesAsync() cần setup phức tạp

// ✅ Tốt hơn: Repository Pattern + Interface
public interface IProductRepository
{
    Task<List<Product>> GetActiveAsync();
    Task<Product?> GetByIdAsync(int id);
    Task AddAsync(Product product);
    Task SaveChangesAsync();
}

// Mock repository trong test
[Fact]
public async Task OrderService_CreatesOrder_WhenProductAvailable()
{
    // Arrange
    var mockRepo = new Mock<IProductRepository>();
    mockRepo.Setup(r => r.GetByIdAsync(1))
            .ReturnsAsync(new Product { Id = 1, Name = "Laptop", Stock = 5 });

    var service = new OrderService(mockRepo.Object);

    // Act
    var result = await service.CreateOrderAsync(customerId: 1, productId: 1, quantity: 2);

    // Assert
    Assert.True(result.Success);
    mockRepo.Verify(r => r.SaveChangesAsync(), Times.Once);
}
```

### Cách 2: Dùng MockQueryable Cho LINQ

```bash
dotnet add package MockQueryable.Moq
```

```csharp
[Fact]
public async Task ProductService_Filter_WorksCorrectly()
{
    // Arrange
    var products = new List<Product>
    {
        new() { Id = 1, Name = "Laptop", Price = 1000, IsActive = true },
        new() { Id = 2, Name = "Book", Price = 20, IsActive = false },
        new() { Id = 3, Name = "Phone", Price = 500, IsActive = true }
    };

    // MockQueryable chuyển List thành IQueryable mock
    var mockSet = products.AsQueryable().BuildMockDbSet();

    var mockContext = new Mock<AppDbContext>();
    mockContext.Setup(c => c.Products).Returns(mockSet.Object);

    var service = new ProductService(mockContext.Object);

    // Act
    var activeProducts = await service.GetActiveProductsAsync();

    // Assert
    Assert.Equal(2, activeProducts.Count);
}
```

---

## 5. WebApplicationFactory — Integration Test Cho API

Test toàn bộ API pipeline với database thật (SQLite hoặc real DB):

```csharp
// CustomWebApplicationFactory.cs
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Xóa DbContext registration thật
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor != null)
                services.Remove(descriptor);

            // Thay bằng SQLite InMemory cho test
            services.AddDbContext<AppDbContext>(options =>
                options.UseSqlite("DataSource=:memory:"));
        });
    }
}

// Test sử dụng factory
public class ProductApiTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;
    private readonly CustomWebApplicationFactory _factory;

    public ProductApiTests(CustomWebApplicationFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetProducts_ReturnsSuccess()
    {
        // Seed data trước khi test
        using var scope = _factory.Services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        context.Database.EnsureCreated();
        context.Products.Add(new Product { Name = "Test", Price = 100, IsActive = true });
        await context.SaveChangesAsync();

        // Act
        var response = await _client.GetAsync("/api/products");

        // Assert
        response.EnsureSuccessStatusCode();
        var products = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        Assert.NotEmpty(products!);
    }
}
```

---

## 6. Builder Pattern Cho Test Data — Dữ Liệu Test Có Cấu Trúc

```csharp
// Product builder — tạo test data dễ đọc và tái sử dụng
public class ProductBuilder
{
    private Product _product = new()
    {
        Name = "Default Product",
        Price = 100,
        IsActive = true,
        CategoryId = 1
    };

    public ProductBuilder WithName(string name)
    {
        _product.Name = name;
        return this;
    }

    public ProductBuilder WithPrice(decimal price)
    {
        _product.Price = price;
        return this;
    }

    public ProductBuilder AsInactive()
    {
        _product.IsActive = false;
        return this;
    }

    public ProductBuilder InCategory(int categoryId)
    {
        _product.CategoryId = categoryId;
        return this;
    }

    public Product Build() => _product;
}

// Dùng trong test
[Fact]
public async Task FilterByPriceRange_ReturnsCorrectProducts()
{
    await using var context = CreateContext("Test_Filter");

    context.Products.AddRange(
        new ProductBuilder().WithName("Cheap").WithPrice(50).Build(),
        new ProductBuilder().WithName("Medium").WithPrice(500).Build(),
        new ProductBuilder().WithName("Expensive").WithPrice(5000).Build()
    );
    await context.SaveChangesAsync();

    var result = await context.Products
        .Where(p => p.Price >= 100 && p.Price <= 1000)
        .ToListAsync();

    Assert.Single(result);
    Assert.Equal("Medium", result[0].Name);
}
```

---

## 7. Respawn — Reset Database Giữa Các Tests

Khi dùng real database, cần reset data giữa tests:

```bash
dotnet add package Respawn
```

```csharp
public class DatabaseFixture : IAsyncLifetime
{
    public AppDbContext Context { get; private set; } = null!;
    private Respawner _respawner = null!;
    private SqlConnection _connection = null!;

    public async Task InitializeAsync()
    {
        _connection = new SqlConnection(TestConnectionString);
        await _connection.OpenAsync();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_connection).Options;
        Context = new AppDbContext(options);

        await Context.Database.MigrateAsync();

        // Cấu hình Respawn — biết bảng nào cần reset
        _respawner = await Respawner.CreateAsync(_connection, new RespawnerOptions
        {
            TablesToIgnore = new Table[] { "__EFMigrationsHistory" }
        });
    }

    // Gọi sau mỗi test để reset dữ liệu
    public async Task ResetAsync() => await _respawner.ResetAsync(_connection);

    public async Task DisposeAsync()
    {
        await Context.DisposeAsync();
        await _connection.DisposeAsync();
    }
}
```

---

## 📋 Testing Checklist

- [ ] Unit tests: Dùng SQLite InMemory (tốt hơn InMemory provider)
- [ ] Integration tests: Dùng TestContainers với real DB
- [ ] API tests: Dùng WebApplicationFactory + SQLite
- [ ] Mỗi test class dùng DB riêng (unique database name)
- [ ] Seed data rõ ràng, không phụ thuộc vào test khác
- [ ] Kiểm tra cả happy path lẫn error cases (constraint violations)
- [ ] Test migrations: Đảm bảo `MigrateAsync()` chạy thành công

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: InMemory Provider vs SQLite InMemory — cái nào tốt hơn cho tests?**
> SQLite InMemory tốt hơn vì nó thực sự thực thi SQL — kiểm tra được constraints, unique indexes, và behavior gần với production hơn. InMemory Provider không enforce constraints và không chạy được `FromSqlRaw`. Nhược điểm: cần mở connection suốt test lifetime.

**Q: Có nên mock DbContext không?**
> Tránh mock trực tiếp DbContext vì phức tạp và brittle. Thay vào đó: (1) Dùng Repository pattern với interface để mock, (2) Dùng SQLite InMemory cho real query behavior, (3) Chỉ mock khi test logic hoàn toàn không liên quan đến database.

**Q: TestContainers dùng khi nào?**
> Khi integration tests cần behavior chính xác của production database (SQL Server/PostgreSQL specific features: json columns, full-text search, specific indexes, stored procedures). Chi phí: chậm hơn (start/stop Docker container), cần Docker trên CI/CD. Thường dùng trong pipeline chạy đêm, không phải mỗi lần commit.
