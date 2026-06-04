# Unit Testing — Kiểm Thử Đơn Vị trong .NET

> Viết unit test hiệu quả với xUnit, NUnit, MSTest — từ cơ bản đến best practices nâng cao.

---

## 🎯 Unit Test Là Gì?

**Unit Test** (Kiểm thử đơn vị) kiểm tra **một đơn vị nhỏ nhất** của code (thường là một method hoặc class) trong **isolation** (cô lập) — không phụ thuộc database, file system, network hay external services.

### Đặc Điểm Unit Test Tốt

```
✅ Fast       — Nhanh: chạy toàn bộ suite trong vài giây
✅ Isolated   — Cô lập: không phụ thuộc trạng thái bên ngoài
✅ Repeatable — Lặp lại: cùng kết quả mọi lúc, mọi nơi
✅ Readable   — Dễ đọc: intent rõ ràng như tài liệu
✅ Maintainable — Bảo trì được: không giòn (brittle)
```

---

## 📦 Chọn Framework: xUnit vs NUnit vs MSTest

| Tiêu Chí                 | xUnit                    | NUnit                     | MSTest                   |
| ------------------------ | ------------------------ | ------------------------- | ------------------------ |
| **Constructor setup**    | ✅ Constructor/IDisposable | `[SetUp]` attribute       | `[TestInitialize]`       |
| **Parallel by default**  | ✅ Có                    | Cần cấu hình              | Cần cấu hình             |
| **Data-driven tests**    | `[Theory]` + `[InlineData]` | `[TestCase]`           | `[DataTestMethod]`       |
| **Microsoft dùng**       | ✅ .NET team dùng xUnit  | Ít hơn                    | Dùng trong một số tool   |
| **Extensibility**        | ✅ Rất linh hoạt         | Tốt                       | Hạn chế hơn              |

**Khuyến nghị:** Dùng **xUnit** cho project mới — đây là lựa chọn của .NET team và cộng đồng hiện đại.

---

## 🚀 Bắt Đầu Với xUnit

### Tạo Project Test

```bash
# Tạo test project
dotnet new xunit -n MyApp.Tests

# Thêm reference đến project cần test
dotnet add MyApp.Tests/MyApp.Tests.csproj reference MyApp/MyApp.csproj

# Cài FluentAssertions để assertion dễ đọc hơn
dotnet add MyApp.Tests package FluentAssertions
```

### Cấu Trúc Thư Mục Đề Xuất

```
MyApp.sln
├── MyApp/                          # Project chính
│   ├── Services/
│   │   └── OrderService.cs
│   └── Models/
│       └── Order.cs
└── MyApp.Tests/                    # Project test
    ├── Services/
    │   └── OrderServiceTests.cs    # Mirror structure của source
    ├── Models/
    └── Helpers/
        └── TestDataBuilder.cs      # Builder cho test data
```

---

## ✍️ Viết Unit Test Cơ Bản

### AAA Pattern — Arrange-Act-Assert

```csharp
public class CalculatorTests
{
    [Fact]
    public void Add_TwoPositiveNumbers_ReturnsCorrectSum()
    {
        // Arrange — Chuẩn bị: khởi tạo đối tượng, dữ liệu đầu vào
        var calculator = new Calculator();

        // Act — Thực thi: gọi method cần test
        var result = calculator.Add(3, 7);

        // Assert — Kiểm tra: xác nhận kết quả
        Assert.Equal(10, result);
    }
}
```

### Các Assertion Phổ Biến Trong xUnit

```csharp
// So sánh giá trị
Assert.Equal(expected, actual);
Assert.NotEqual(expected, actual);

// Kiểm tra null
Assert.Null(value);
Assert.NotNull(value);

// Boolean
Assert.True(condition);
Assert.False(condition);

// Collections
Assert.Empty(collection);
Assert.Single(collection);
Assert.Contains(item, collection);
Assert.Equal(3, collection.Count());

// Exception
Assert.Throws<InvalidOperationException>(() => service.DoSomething());
var ex = Assert.Throws<ArgumentException>(() => new Order(-1));
Assert.Equal("quantity", ex.ParamName);

// Async exception
await Assert.ThrowsAsync<NotFoundException>(
    () => service.GetByIdAsync(999));
```

### FluentAssertions — Assertion Dễ Đọc Hơn

```csharp
using FluentAssertions;

// Thay vì Assert.Equal(10, result)
result.Should().Be(10);

// Thay vì Assert.NotNull(result)
result.Should().NotBeNull();

// Collections
items.Should().HaveCount(3);
items.Should().Contain(x => x.IsActive);
items.Should().BeInAscendingOrder(x => x.Name);

// Strings
name.Should().StartWith("John");
name.Should().Contain("Doe");

// Exceptions
Action act = () => service.Process(null);
act.Should().Throw<ArgumentNullException>()
   .WithMessage("*parameter*");

// Async
Func<Task> act = async () => await service.ProcessAsync(null);
await act.Should().ThrowAsync<ArgumentNullException>();
```

---

## 📊 Theory — Data-Driven Tests (Test Hướng Dữ Liệu)

Chạy cùng một test với nhiều bộ dữ liệu đầu vào khác nhau.

### `[InlineData]` — Dữ Liệu Trực Tiếp

```csharp
public class DiscountCalculatorTests
{
    [Theory]
    [InlineData(100, 0.1, 90)]    // giá 100, giảm 10% → còn 90
    [InlineData(200, 0.2, 160)]   // giá 200, giảm 20% → còn 160
    [InlineData(50, 0.5, 25)]     // giá 50, giảm 50% → còn 25
    public void ApplyDiscount_ValidInputs_ReturnsDiscountedPrice(
        decimal price, decimal discount, decimal expected)
    {
        var calculator = new DiscountCalculator();
        var result = calculator.Apply(price, discount);
        result.Should().Be(expected);
    }
}
```

### `[MemberData]` — Dữ Liệu Từ Property/Method

```csharp
public class OrderValidatorTests
{
    public static IEnumerable<object[]> InvalidOrders => new[]
    {
        new object[] { null, "Order không được null" },
        new object[] { new Order { Quantity = 0 }, "Quantity phải lớn hơn 0" },
        new object[] { new Order { Quantity = -1 }, "Quantity không được âm" },
    };

    [Theory]
    [MemberData(nameof(InvalidOrders))]
    public void Validate_InvalidOrder_ThrowsArgumentException(
        Order order, string reason)
    {
        var validator = new OrderValidator();
        Action act = () => validator.Validate(order);
        act.Should().Throw<ArgumentException>(reason);
    }
}
```

### `[ClassData]` — Dữ Liệu Từ Class Riêng

```csharp
public class PriceTestData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return new object[] { 10m, 5m, 15m };
        yield return new object[] { 0m, 100m, 100m };
        yield return new object[] { 99.99m, 0.01m, 100m };
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

[Theory]
[ClassData(typeof(PriceTestData))]
public void Add_PriceAndShipping_ReturnsTotal(
    decimal price, decimal shipping, decimal expected) { }
```

---

## ⚙️ Setup & Teardown — Khởi Tạo và Dọn Dẹp

### Constructor & IDisposable (xUnit)

xUnit tạo instance mới cho mỗi test — dùng constructor thay vì `[SetUp]`:

```csharp
public class OrderServiceTests : IDisposable
{
    private readonly OrderService _service;
    private readonly Mock<IOrderRepository> _repoMock;

    // Constructor chạy trước mỗi test
    public OrderServiceTests()
    {
        _repoMock = new Mock<IOrderRepository>();
        _service = new OrderService(_repoMock.Object);
    }

    // Dispose chạy sau mỗi test
    public void Dispose()
    {
        // Dọn dẹp tài nguyên nếu có
    }

    [Fact]
    public void GetOrder_ExistingId_ReturnsOrder() { }
}
```

### IAsyncLifetime — Async Setup/Teardown

```csharp
public class AsyncSetupTests : IAsyncLifetime
{
    private SqliteConnection _connection;

    public async Task InitializeAsync()
    {
        _connection = new SqliteConnection("Data Source=:memory:");
        await _connection.OpenAsync();
        // Tạo schema, seed data, v.v.
    }

    public async Task DisposeAsync()
    {
        await _connection.DisposeAsync();
    }
}
```

### IClassFixture — Chia Sẻ Setup Giữa Nhiều Test

Dùng khi setup tốn kém (ví dụ: khởi động server), không muốn chạy lại mỗi test:

```csharp
// Fixture được tạo một lần cho toàn bộ class
public class DatabaseFixture : IDisposable
{
    public SqliteConnection Connection { get; }

    public DatabaseFixture()
    {
        Connection = new SqliteConnection("Data Source=:memory:");
        Connection.Open();
        // Migrate schema
    }

    public void Dispose() => Connection.Dispose();
}

public class UserRepositoryTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;

    public UserRepositoryTests(DatabaseFixture fixture)
    {
        _fixture = fixture; // Dùng chung một connection
    }
}
```

### ICollectionFixture — Chia Sẻ Giữa Nhiều Class

```csharp
[CollectionDefinition("Database collection")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }

[Collection("Database collection")]
public class OrderRepositoryTests
{
    public OrderRepositoryTests(DatabaseFixture fixture) { }
}

[Collection("Database collection")]
public class ProductRepositoryTests
{
    public ProductRepositoryTests(DatabaseFixture fixture) { }
}
```

---

## 🏷️ Tổ Chức Test: Traits & Categories

```csharp
// Nhóm test theo trait
[Trait("Category", "Unit")]
[Trait("Feature", "Order")]
public class OrderTests { }

// Chạy chỉ một category
// dotnet test --filter "Category=Unit"
// dotnet test --filter "Feature=Order"
```

### Skip Test Tạm Thời

```csharp
[Fact(Skip = "Đang chờ implement feature X")]
public void Feature_NotYetImplemented() { }
```

---

## 🔬 Test Isolation — Cô Lập Test

### Vấn Đề: Shared State (Trạng Thái Dùng Chung)

```csharp
// ❌ BAD: static state làm test phụ thuộc nhau
public class BadTests
{
    private static List<Order> _orders = new(); // Nguy hiểm!

    [Fact]
    public void Test1_AddsOrder()
    {
        _orders.Add(new Order());
        _orders.Should().HaveCount(1);
    }

    [Fact]
    public void Test2_ChecksEmpty() // Có thể fail nếu Test1 chạy trước
    {
        _orders.Should().BeEmpty();
    }
}

// ✅ GOOD: mỗi test tự tạo data riêng
public class GoodTests
{
    [Fact]
    public void Test1_AddsOrder()
    {
        var orders = new List<Order>();
        orders.Add(new Order());
        orders.Should().HaveCount(1);
    }

    [Fact]
    public void Test2_ChecksEmpty()
    {
        var orders = new List<Order>();
        orders.Should().BeEmpty();
    }
}
```

### Parallel Test Execution — Chạy Song Song

xUnit mặc định chạy test song song. Cẩn thận với shared resources:

```csharp
// Tắt parallel cho một collection khi cần
[assembly: CollectionBehavior(DisableTestParallelization = true)]

// Hoặc chỉ tắt cho một collection
[CollectionDefinition("Sequential", DisableParallelization = true)]
public class SequentialCollection { }
```

---

## 📝 Test Data Builders — Builder Tạo Dữ Liệu Test

### Object Mother Pattern

```csharp
// Factory cho test objects
public static class OrderMother
{
    public static Order Valid() => new Order
    {
        Id = Guid.NewGuid(),
        CustomerId = Guid.NewGuid(),
        Items = new List<OrderItem> { OrderItemMother.Valid() },
        Status = OrderStatus.Pending,
        CreatedAt = DateTime.UtcNow
    };

    public static Order WithStatus(OrderStatus status)
    {
        var order = Valid();
        order.Status = status;
        return order;
    }

    public static Order Empty() => new Order
    {
        Id = Guid.NewGuid(),
        Items = new List<OrderItem>()
    };
}
```

### Builder Pattern Cho Test Data

```csharp
public class OrderBuilder
{
    private Guid _id = Guid.NewGuid();
    private List<OrderItem> _items = new();
    private OrderStatus _status = OrderStatus.Pending;
    private decimal _discount = 0;

    public OrderBuilder WithId(Guid id) { _id = id; return this; }

    public OrderBuilder WithItem(decimal price, int qty = 1)
    {
        _items.Add(new OrderItem { Price = price, Quantity = qty });
        return this;
    }

    public OrderBuilder WithStatus(OrderStatus status)
    {
        _status = status;
        return this;
    }

    public OrderBuilder WithDiscount(decimal discount)
    {
        _discount = discount;
        return this;
    }

    public Order Build() => new Order
    {
        Id = _id,
        Items = _items,
        Status = _status,
        Discount = _discount
    };
}

// Sử dụng
var order = new OrderBuilder()
    .WithItem(100m, 2)
    .WithItem(50m, 1)
    .WithDiscount(0.1m)
    .Build();
```

### Bogus — Tạo Fake Data Tự Động

```csharp
// dotnet add package Bogus
using Bogus;

var userFaker = new Faker<User>()
    .RuleFor(u => u.Id, f => f.Random.Guid())
    .RuleFor(u => u.Name, f => f.Name.FullName())
    .RuleFor(u => u.Email, f => f.Internet.Email())
    .RuleFor(u => u.Age, f => f.Random.Int(18, 80));

var singleUser = userFaker.Generate();
var tenUsers = userFaker.Generate(10);
```

---

## 🚫 Test Anti-Patterns — Các Lỗi Thường Gặp

### 1. Test Logic Không Cần Thiết

```csharp
// ❌ BAD: loop trong test — khó debug khi fail
[Fact]
public void ValidateAll_BadPattern()
{
    var inputs = new[] { "a@b.com", "c@d.com", "e@f.com" };
    foreach (var email in inputs)
    {
        Assert.True(EmailValidator.IsValid(email)); // Khi fail, không biết cái nào
    }
}

// ✅ GOOD: dùng Theory
[Theory]
[InlineData("a@b.com")]
[InlineData("c@d.com")]
[InlineData("e@f.com")]
public void IsValid_ValidEmail_ReturnsTrue(string email)
{
    EmailValidator.IsValid(email).Should().BeTrue();
}
```

### 2. Assert Nhiều Thứ Trong Một Test

```csharp
// ❌ BAD: test nhiều concern → khi fail không rõ cái gì sai
[Fact]
public void CreateUser_ShouldWorkCorrectly()
{
    var user = _service.Create("John", "john@example.com");
    Assert.NotNull(user);
    Assert.Equal("John", user.Name);
    Assert.Equal("john@example.com", user.Email);
    Assert.NotEqual(Guid.Empty, user.Id);
    Assert.True(user.IsActive);
    // ... nhiều assert nữa
}

// ✅ GOOD: tách ra hoặc group logical assertions
[Fact]
public void CreateUser_ValidInput_ReturnsUserWithCorrectData()
{
    var user = _service.Create("John", "john@example.com");

    // Chấp nhận nhiều assert nếu tất cả liên quan đến CÙNG một concern
    user.Should().NotBeNull();
    user.Name.Should().Be("John");
    user.Email.Should().Be("john@example.com");
    user.Id.Should().NotBeEmpty();
    user.IsActive.Should().BeTrue();
}
```

### 3. Test Phụ Thuộc Thứ Tự Chạy

```csharp
// ❌ BAD: Test2 phụ thuộc Test1 chạy trước
[Fact]
public void Test1_CreateUser() { _userId = _service.Create(...); }

[Fact]
public void Test2_UpdateUser() { _service.Update(_userId, ...); } // Fail nếu Test1 chưa chạy
```

### 4. Test Quá Gắn Với Implementation

```csharp
// ❌ BAD: test verify method internal gọi gì — fragile khi refactor
mockRepo.Verify(r => r.FindByIdAsync(id), Times.Once);
mockRepo.Verify(r => r.SaveAsync(It.IsAny<Order>()), Times.Once);
mockCache.Verify(c => c.SetAsync(It.IsAny<string>(), ...), Times.Once);
// Quá nhiều verify → test "kể" implementation thay vì test behavior

// ✅ GOOD: test kết quả (outcome) thay vì cách làm
var result = await _service.GetOrderAsync(id);
result.Should().NotBeNull();
result.Id.Should().Be(id);
```

---

## 📋 Checklist Viết Unit Test

```
□ Tên test mô tả rõ: MethodName_Scenario_Expected
□ Có đủ 3 phần AAA với comment phân cách
□ Mỗi test kiểm tra đúng một điều
□ Không có logic phức tạp trong test
□ Không phụ thuộc external systems (DB, API, file)
□ Test độc lập — có thể chạy theo bất kỳ thứ tự nào
□ Sử dụng Theory cho nhiều bộ dữ liệu tương tự
□ Dùng test builders thay vì tạo object phức tạp inline
□ Assertion message rõ khi fail (dùng FluentAssertions)
□ Test coverage các trường hợp biên (edge cases)
```

---

## 🔗 Liên Quan

- [2-mocking.md](./2-mocking.md) — Moq và NSubstitute cho test isolation
- [3-integration-testing.md](./3-integration-testing.md) — Test toàn bộ stack
- [4-tdd-guide.md](./4-tdd-guide.md) — Viết test trước, code sau
- [5-test-coverage.md](./5-test-coverage.md) — Đo và phân tích coverage

---

*Cập nhật: 2026-06-02 | xUnit 2.x | .NET 8*
