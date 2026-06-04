# Mocking — Giả Lập Phụ Thuộc Trong Test

> Moq và NSubstitute: tạo mock, stub, spy để cô lập unit test khỏi external dependencies.

---

## 🎯 Tại Sao Cần Mocking?

Khi unit test một class, class đó thường phụ thuộc vào các service khác (repository, email service, HTTP client, ...). Nếu dùng dependency thật:

- Test chạy chậm (network call, DB query)
- Test không ổn định (server down, data thay đổi)
- Không kiểm soát được behavior của dependency

**Mocking** giải quyết vấn đề này bằng cách thay thế dependency thật bằng **đối tượng giả** mà ta kiểm soát hoàn toàn.

---

## 📚 Test Doubles — Các Loại Đối Tượng Thay Thế

| Kiểu        | Tiếng Anh    | Mục Đích                                                           | Ví Dụ                              |
| ----------- | ------------ | ------------------------------------------------------------------ | ---------------------------------- |
| **Dummy**   | Dummy Object | Truyền vào cho đủ tham số, không thực sự dùng                      | `null` hoặc object rỗng            |
| **Stub**    | Stub         | Trả về giá trị cố định để test tiếp tục chạy                       | `GetUser()` luôn trả về user giả   |
| **Mock**    | Mock         | Kiểm tra xem method có được gọi không, với tham số nào             | Verify `SaveAsync()` được gọi 1 lần |
| **Fake**    | Fake         | Cài đặt đơn giản hóa hoạt động được (working implementation)       | In-memory repository               |
| **Spy**     | Spy          | Ghi lại lời gọi để xác minh sau, vẫn gọi implementation thật       | Ít dùng trong .NET                 |

---

## 📦 Cài Đặt

```bash
# Moq — mocking library phổ biến nhất
dotnet add package Moq

# NSubstitute — syntax tự nhiên hơn
dotnet add package NSubstitute

# FluentAssertions — assertion dễ đọc (nên dùng cùng)
dotnet add package FluentAssertions
```

---

## 🔧 Moq — Cơ Bản

### Tạo Mock và Setup Return Value (Cấu Hình Giá Trị Trả Về)

```csharp
// Interface cần mock
public interface IUserRepository
{
    User? GetById(Guid id);
    Task<User?> GetByIdAsync(Guid id);
    Task SaveAsync(User user);
    bool Exists(string email);
}

// Tạo mock
var mockRepo = new Mock<IUserRepository>();

// Setup: khi gọi GetById với bất kỳ Guid nào → trả về user giả
mockRepo.Setup(r => r.GetById(It.IsAny<Guid>()))
        .Returns(new User { Id = Guid.NewGuid(), Name = "John" });

// Setup với tham số cụ thể
var specificId = Guid.Parse("...");
mockRepo.Setup(r => r.GetById(specificId))
        .Returns(new User { Id = specificId, Name = "Jane" });

// Setup trả về null
mockRepo.Setup(r => r.GetById(It.IsAny<Guid>()))
        .Returns((User?)null);

// Setup async
mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<Guid>()))
        .ReturnsAsync(new User { Name = "Async User" });

// Dùng mock.Object để lấy instance
var service = new UserService(mockRepo.Object);
```

### Argument Matchers — Bộ Lọc Tham Số

```csharp
// It.IsAny<T>() — bất kỳ giá trị nào
mockRepo.Setup(r => r.GetById(It.IsAny<Guid>())).Returns(...);

// It.Is<T>(predicate) — giá trị thỏa điều kiện
mockRepo.Setup(r => r.GetById(It.Is<Guid>(id => id != Guid.Empty)))
        .Returns(...);

// Giá trị cụ thể
mockRepo.Setup(r => r.Exists("admin@example.com")).Returns(true);
mockRepo.Setup(r => r.Exists(It.IsAny<string>())).Returns(false);

// It.IsNotNull<T>()
mockRepo.Setup(r => r.SaveAsync(It.IsNotNull<User>()))
        .Returns(Task.CompletedTask);
```

### Setup Exceptions — Cấu Hình Ném Ngoại Lệ

```csharp
// Throw khi gọi method
mockRepo.Setup(r => r.GetById(It.IsAny<Guid>()))
        .Throws<NotFoundException>();

// Throw với message cụ thể
mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<Guid>()))
        .ThrowsAsync(new NotFoundException("User not found"));
```

### Setup Callback — Chạy Code Khi Method Được Gọi

```csharp
var capturedUser = (User?)null;

mockRepo.Setup(r => r.SaveAsync(It.IsAny<User>()))
        .Callback<User>(user => capturedUser = user) // Bắt tham số được truyền vào
        .Returns(Task.CompletedTask);

await service.CreateUserAsync("John", "john@example.com");

// Kiểm tra user được lưu có dữ liệu đúng
capturedUser.Should().NotBeNull();
capturedUser!.Name.Should().Be("John");
```

---

## ✅ Verify — Kiểm Tra Lời Gọi

```csharp
// Verify method được gọi đúng 1 lần
mockRepo.Verify(r => r.SaveAsync(It.IsAny<User>()), Times.Once);

// Verify được gọi đúng N lần
mockRepo.Verify(r => r.GetById(It.IsAny<Guid>()), Times.Exactly(2));

// Verify không được gọi
mockRepo.Verify(r => r.SaveAsync(It.IsAny<User>()), Times.Never);

// Verify được gọi ít nhất/nhiều nhất N lần
mockRepo.Verify(r => r.GetById(It.IsAny<Guid>()), Times.AtLeastOnce);
mockRepo.Verify(r => r.GetById(It.IsAny<Guid>()), Times.AtMost(3));

// Verify với tham số cụ thể
mockRepo.Verify(r => r.SaveAsync(
    It.Is<User>(u => u.Email == "john@example.com")), Times.Once);

// VerifyAll — kiểm tra tất cả setup đều được gọi
mockRepo.VerifyAll();

// VerifyNoOtherCalls — không có gọi nào ngoài đã verify
mockRepo.VerifyNoOtherCalls();
```

### Khi Nào Nên Verify?

```csharp
// ✅ GOOD: Verify khi behavior là side effect quan trọng
// (gửi email, lưu DB, publish event)
[Fact]
public async Task CreateOrder_ValidInput_SavesOrderToDatabase()
{
    var order = new OrderBuilder().WithItem(100m).Build();
    await _service.CreateOrderAsync(order);

    _mockRepo.Verify(r => r.SaveAsync(
        It.Is<Order>(o => o.Items.Count == 1)), Times.Once);
}

// ❌ BAD: Verify quá nhiều internal calls → test fragile
[Fact]
public async Task GetOrder_ShouldWork_BadVerifyExample()
{
    await _service.GetOrderAsync(Guid.NewGuid());

    _mockRepo.Verify(r => r.GetByIdAsync(It.IsAny<Guid>()), Times.Once);
    _mockCache.Verify(c => c.GetAsync(It.IsAny<string>()), Times.Once);
    _mockLogger.Verify(l => l.LogInformation(...), Times.Once);
    // Quá nhiều verify → mọi thay đổi nhỏ làm test fail
}
```

---

## 🔄 MockBehavior — Hành Vi Mặc Định

```csharp
// MockBehavior.Loose (mặc định): Trả về default value nếu không setup
var looseMock = new Mock<IService>(); // Không setup → trả về null/0/false
var result = looseMock.Object.GetData(); // Trả về null, không throw

// MockBehavior.Strict: Throw nếu gọi method không được setup
var strictMock = new Mock<IService>(MockBehavior.Strict);
var result = strictMock.Object.GetData(); // Throw MockException!
// Phải setup tất cả methods sẽ được gọi

// Khuyến nghị: Dùng Loose để test không bị giòn
```

---

## 🔗 NSubstitute — Syntax Tự Nhiên Hơn

### Tạo Substitute (Thay Thế)

```csharp
// Thay vì new Mock<T>()
var repo = Substitute.For<IUserRepository>();

// Dùng trực tiếp — không cần .Object
var service = new UserService(repo);
```

### Setup Returns

```csharp
// NSubstitute — syntax rất tự nhiên
repo.GetById(Arg.Any<Guid>()).Returns(new User { Name = "John" });

// Async
repo.GetByIdAsync(Arg.Any<Guid>()).Returns(new User { Name = "Async" });

// Throw exception
repo.GetById(Arg.Any<Guid>()).Throws<NotFoundException>();

// Multiple returns (lần 1 trả khác, lần 2 trả khác)
repo.GetById(Arg.Any<Guid>()).Returns(
    new User { Name = "First" },
    new User { Name = "Second" });
```

### Argument Matchers Trong NSubstitute

```csharp
// Bất kỳ giá trị
Arg.Any<Guid>()
Arg.Any<string>()

// Thỏa điều kiện
Arg.Is<string>(s => s.Contains("@"))
Arg.Is<Guid>(id => id != Guid.Empty)

// Giá trị cụ thể
repo.GetById(specificId).Returns(user);
```

### Verify Trong NSubstitute

```csharp
// Received() — đã nhận được gọi
repo.Received().SaveAsync(Arg.Any<User>());
repo.Received(1).SaveAsync(Arg.Any<User>()); // Đúng 1 lần

// DidNotReceive() — không được gọi
repo.DidNotReceive().DeleteAsync(Arg.Any<Guid>());

// ReceivedWithAnyArgs() — được gọi với bất kỳ args nào
repo.ReceivedWithAnyArgs().SaveAsync(default!);

// Verify với điều kiện
repo.Received().SaveAsync(Arg.Is<User>(u => u.Email == "john@example.com"));
```

---

## ⚖️ Moq vs NSubstitute — So Sánh

| Tính Năng           | Moq                                    | NSubstitute                              |
| ------------------- | -------------------------------------- | ---------------------------------------- |
| **Setup syntax**    | `mock.Setup(x => x.Method()).Returns()`| `sub.Method().Returns()`                 |
| **Access object**   | `mock.Object`                          | Trực tiếp dùng `sub`                     |
| **Verify calls**    | `mock.Verify(x => x.Method(), Times.Once)` | `sub.Received(1).Method()`           |
| **Verify no calls** | `mock.Verify(x => x.Method(), Times.Never)` | `sub.DidNotReceive().Method()`      |
| **Strict mode**     | `new Mock<T>(MockBehavior.Strict)`     | `Substitute.For<T>()` (loose by default) |
| **Popularity**      | Phổ biến hơn                           | Được ưa thích vì syntax ngắn hơn         |

---

## 🏗️ Ví Dụ Thực Tế Đầy Đủ

### Scenario: Test UserService Với Repository và Email Service

```csharp
// Domain
public class User
{
    public Guid Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
    public bool IsEmailVerified { get; set; }
}

// Interfaces
public interface IUserRepository
{
    Task<User?> GetByEmailAsync(string email);
    Task AddAsync(User user);
}

public interface IEmailService
{
    Task SendVerificationEmailAsync(string email, string token);
}

// Service cần test
public class UserRegistrationService
{
    private readonly IUserRepository _repo;
    private readonly IEmailService _emailService;

    public UserRegistrationService(
        IUserRepository repo, IEmailService emailService)
    {
        _repo = repo;
        _emailService = emailService;
    }

    public async Task<User> RegisterAsync(string name, string email)
    {
        var existing = await _repo.GetByEmailAsync(email);
        if (existing != null)
            throw new DuplicateEmailException(email);

        var user = new User
        {
            Id = Guid.NewGuid(),
            Name = name,
            Email = email
        };

        await _repo.AddAsync(user);
        await _emailService.SendVerificationEmailAsync(email, GenerateToken());
        return user;
    }

    private static string GenerateToken() => Guid.NewGuid().ToString("N");
}
```

```csharp
// Test class
public class UserRegistrationServiceTests
{
    private readonly Mock<IUserRepository> _mockRepo;
    private readonly Mock<IEmailService> _mockEmail;
    private readonly UserRegistrationService _service;

    public UserRegistrationServiceTests()
    {
        _mockRepo = new Mock<IUserRepository>();
        _mockEmail = new Mock<IEmailService>();
        _service = new UserRegistrationService(_mockRepo.Object, _mockEmail.Object);
    }

    [Fact]
    public async Task Register_NewEmail_CreatesUserAndSendsVerificationEmail()
    {
        // Arrange
        _mockRepo.Setup(r => r.GetByEmailAsync(It.IsAny<string>()))
                 .ReturnsAsync((User?)null); // Email chưa tồn tại

        _mockRepo.Setup(r => r.AddAsync(It.IsAny<User>()))
                 .Returns(Task.CompletedTask);

        _mockEmail.Setup(e => e.SendVerificationEmailAsync(
                    It.IsAny<string>(), It.IsAny<string>()))
                  .Returns(Task.CompletedTask);

        // Act
        var user = await _service.RegisterAsync("John Doe", "john@example.com");

        // Assert — kiểm tra kết quả trả về
        user.Should().NotBeNull();
        user.Name.Should().Be("John Doe");
        user.Email.Should().Be("john@example.com");
        user.Id.Should().NotBeEmpty();

        // Assert — kiểm tra side effects (hiệu ứng phụ)
        _mockRepo.Verify(r => r.AddAsync(
            It.Is<User>(u => u.Email == "john@example.com")), Times.Once);

        _mockEmail.Verify(e => e.SendVerificationEmailAsync(
            "john@example.com", It.IsAny<string>()), Times.Once);
    }

    [Fact]
    public async Task Register_DuplicateEmail_ThrowsDuplicateEmailException()
    {
        // Arrange — email đã tồn tại
        _mockRepo.Setup(r => r.GetByEmailAsync("existing@example.com"))
                 .ReturnsAsync(new User { Email = "existing@example.com" });

        // Act
        Func<Task> act = () => _service.RegisterAsync("Jane", "existing@example.com");

        // Assert — exception được throw
        await act.Should().ThrowAsync<DuplicateEmailException>()
                 .WithMessage("*existing@example.com*");

        // Verify — không lưu user, không gửi email
        _mockRepo.Verify(r => r.AddAsync(It.IsAny<User>()), Times.Never);
        _mockEmail.Verify(e => e.SendVerificationEmailAsync(
            It.IsAny<string>(), It.IsAny<string>()), Times.Never);
    }
}
```

---

## 🎭 Mock HttpClient — Giả Lập HTTP Calls

```csharp
// Không mock HttpClient trực tiếp — dùng HttpMessageHandler
public class WeatherApiClientTests
{
    [Fact]
    public async Task GetWeather_ValidCity_ReturnsTemperature()
    {
        // Arrange — tạo fake HTTP handler
        var handlerMock = new Mock<HttpMessageHandler>();
        handlerMock.Protected()
            .Setup<Task<HttpResponseMessage>>(
                "SendAsync",
                ItExpr.IsAny<HttpRequestMessage>(),
                ItExpr.IsAny<CancellationToken>())
            .ReturnsAsync(new HttpResponseMessage
            {
                StatusCode = HttpStatusCode.OK,
                Content = new StringContent("""{"temp": 25.5, "city": "Hanoi"}""")
            });

        var httpClient = new HttpClient(handlerMock.Object)
        {
            BaseAddress = new Uri("https://api.weather.com")
        };

        var client = new WeatherApiClient(httpClient);

        // Act
        var weather = await client.GetWeatherAsync("Hanoi");

        // Assert
        weather.Temperature.Should().Be(25.5);
    }
}
```

---

## 🔄 Auto-mocking với AutoFixture + AutoMoq

```bash
dotnet add package AutoFixture
dotnet add package AutoFixture.AutoMoq
dotnet add package AutoFixture.Xunit2
```

```csharp
// Tự động tạo mock cho tất cả dependencies
public class UserServiceAutoMockTests
{
    [Theory, AutoMoqData]
    public async Task GetUser_ExistingId_ReturnsUser(
        [Frozen] Mock<IUserRepository> mockRepo,
        UserService sut,  // AutoFixture tự tạo với mocks đã inject
        User expectedUser)
    {
        mockRepo.Setup(r => r.GetByIdAsync(expectedUser.Id))
                .ReturnsAsync(expectedUser);

        var result = await sut.GetUserAsync(expectedUser.Id);

        result.Should().BeEquivalentTo(expectedUser);
    }
}

// Custom AutoData attribute
public class AutoMoqDataAttribute : AutoDataAttribute
{
    public AutoMoqDataAttribute()
        : base(() => new Fixture().Customize(new AutoMoqCustomization())) { }
}
```

---

## 📋 Checklist Mocking

```
□ Chỉ mock interfaces/abstractions, không mock concrete classes
□ Không over-mock: chỉ mock dependencies thực sự cần cô lập
□ Tránh verify quá nhiều internal calls (test behavior, not implementation)
□ Dùng Callback để capture tham số khi cần inspect
□ Setup chỉ những gì test cần — để behavior khác return default
□ Mỗi test chỉ verify điều nó quan tâm
□ Dùng Strict mode khi muốn đảm bảo không có unexpected calls
```

---

## 🔗 Liên Quan

- [1-unit-testing.md](./1-unit-testing.md) — Nền tảng viết unit test
- [3-integration-testing.md](./3-integration-testing.md) — Khi nào không cần mock
- [4-tdd-guide.md](./4-tdd-guide.md) — Mocking trong vòng Red/Green/Refactor

---

*Cập nhật: 2026-06-02 | Moq 4.x | NSubstitute 5.x | .NET 8*
