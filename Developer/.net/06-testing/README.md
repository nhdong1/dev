# 06 — Testing — Kiểm Thử Phần Mềm .NET

> Chiến lược kiểm thử toàn diện: từ unit test đến integration test, TDD và performance testing trong hệ sinh thái .NET.

---

## 🎯 Tại Sao Testing Quan Trọng?

Kiểm thử không phải là công việc tốn thời gian — đó là **đầu tư vào chất lượng dài hạn**:

- **Bắt lỗi sớm** — chi phí sửa lỗi trong dev thấp hơn 100× so với production
- **Tự tin refactor** — test suite là mạng lưới an toàn khi thay đổi code
- **Tài liệu sống** — test tốt mô tả hành vi mong đợi rõ hơn comment
- **Thiết kế tốt hơn** — code khó test thường là dấu hiệu thiết kế kém (tight coupling)
- **CI/CD tin cậy** — pipeline tự động ngăn merge code lỗi

---

## 📐 Test Pyramid — Kim Tự Tháp Kiểm Thử

```
           /\
          /E2E\          ← Ít nhất, chạy chậm, tốn kém nhất
         /------\
        /  Integ  \      ← Trung bình, test tích hợp thực sự
       /------------\
      /  Unit Tests  \   ← Nhiều nhất, nhanh nhất, rẻ nhất
     /________________\
```

| Tầng                                   | Số Lượng | Tốc Độ   | Chi Phí  | Mục Đích                              |
| -------------------------------------- | -------- | -------- | -------- | ------------------------------------- |
| **Unit Test** (Kiểm thử đơn vị)        | 70–80%   | Rất nhanh| Thấp     | Logic nghiệp vụ, function, class      |
| **Integration Test** (Kiểm thử tích hợp) | 15–20% | Trung bình | Trung bình | API, DB, external services          |
| **E2E Test** (Kiểm thử đầu cuối)       | 5–10%    | Chậm     | Cao      | User journey, toàn bộ hệ thống        |

### Nguyên Tắc Test Pyramid

- Đừng dựa hoàn toàn vào E2E — chúng chậm và giòn (brittle)
- Unit test phải **cực kỳ nhanh** (dưới 1ms mỗi test)
- Integration test đủ để xác nhận các thành phần hoạt động cùng nhau
- **Ice Cream Anti-Pattern** — nhiều E2E, ít unit → vòng feedback chậm, không bền vững

---

## 📁 Nội Dung Section Này

| File                         | Chủ Đề                                                      | Độ Khó |
| ---------------------------- | ----------------------------------------------------------- | ------ |
| `1-unit-testing.md`          | xUnit, NUnit, MSTest — viết unit test đúng cách             | ⭐⭐   |
| `2-mocking.md`               | Moq, NSubstitute — mock, stub, spy, fake                    | ⭐⭐   |
| `3-integration-testing.md`   | WebApplicationFactory, TestServer, Testcontainers           | ⭐⭐⭐ |
| `4-tdd-guide.md`             | TDD — Test-Driven Development — Red/Green/Refactor          | ⭐⭐⭐ |
| `5-test-coverage.md`         | Coverlet, phân tích độ bao phủ, what to test                | ⭐⭐   |
| `6-performance-testing.md`   | BenchmarkDotNet, k6, NBomber — kiểm thử hiệu năng           | ⭐⭐⭐ |

---

## 🏗️ Các Loại Test Trong .NET

### Unit Test — Kiểm Thử Đơn Vị

Test một **đơn vị nhỏ** (thường là một method hoặc class) trong **isolation** (cô lập) — không phụ thuộc database, network, file system.

**Framework phổ biến:**
- **xUnit** — framework hiện đại nhất, được Microsoft ưa chuộng
- **NUnit** — phổ biến lâu đời, nhiều tính năng
- **MSTest** — framework mặc định của Microsoft

```csharp
// Ví dụ unit test với xUnit
public class OrderServiceTests
{
    [Fact]
    public void CalculateTotal_WithValidItems_ReturnsCorrectSum()
    {
        // Arrange — chuẩn bị dữ liệu
        var service = new OrderService();
        var items = new[] { new OrderItem(10m), new OrderItem(20m) };

        // Act — thực thi hành động cần test
        var total = service.CalculateTotal(items);

        // Assert — kiểm tra kết quả
        Assert.Equal(30m, total);
    }
}
```

### Integration Test — Kiểm Thử Tích Hợp

Test **nhiều thành phần** hoạt động cùng nhau: controller + service + database.

```csharp
// Ví dụ integration test với WebApplicationFactory
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
        response.EnsureSuccessStatusCode();
    }
}
```

### E2E Test — Kiểm Thử Đầu Cuối

Test toàn bộ luồng từ **giao diện người dùng đến database**. Thường dùng Playwright hoặc Selenium cho web.

---

## 🧪 Nguyên Tắc Viết Test Tốt

### AAA Pattern — Mẫu Arrange-Act-Assert

```
Arrange → Act → Assert
(Chuẩn bị) → (Thực thi) → (Kiểm tra)
```

Mỗi test chỉ làm **một việc**, có **ba phần rõ ràng**.

### FIRST Principles — Nguyên Lý FIRST

| Chữ Cái | Tiếng Anh   | Ý Nghĩa                                                  |
| ------- | ----------- | -------------------------------------------------------- |
| **F**   | Fast        | Nhanh — unit test chạy dưới 1ms                          |
| **I**   | Independent | Độc lập — không phụ thuộc thứ tự chạy                    |
| **R**   | Repeatable  | Lặp lại được — cùng kết quả mọi môi trường               |
| **S**   | Self-validating | Tự xác nhận — pass/fail rõ ràng, không cần kiểm tra thủ công |
| **T**   | Timely      | Kịp thời — viết trước hoặc cùng lúc với code             |

### Đặt Tên Test Đúng Chuẩn

```csharp
// Pattern: MethodName_StateUnderTest_ExpectedBehavior
// (TênPhương Thức_TrạngTháiKiểmTra_HànhViMongĐợi)

[Fact]
public void Withdraw_WithInsufficientBalance_ThrowsInvalidOperationException() { }

[Fact]
public void Add_WithTwoPositiveNumbers_ReturnsSumCorrectly() { }

[Fact]
public void CreateUser_WithDuplicateEmail_ReturnsBadRequest() { }
```

---

## 🔄 Test Doubles — Các Kiểu Thay Thế Trong Test

| Kiểu          | Tiếng Anh | Mô Tả                                                          |
| ------------- | --------- | -------------------------------------------------------------- |
| **Dummy**     | Dummy     | Đối tượng truyền vào nhưng không dùng                          |
| **Stub**      | Stub      | Trả về giá trị cố định, không kiểm tra tương tác               |
| **Mock**      | Mock      | Kiểm tra xem method có được gọi không, với tham số nào         |
| **Fake**      | Fake      | Cài đặt thực sự nhưng đơn giản hóa (như InMemory database)     |
| **Spy**       | Spy       | Ghi lại lời gọi để xác minh sau                                |

---

## 📊 Độ Bao Phủ Test — Test Coverage

**Coverage** (Độ bao phủ) đo lường phần trăm code được test chạy qua.

| Loại Coverage             | Mô Tả                                          | Mục Tiêu     |
| ------------------------- | ---------------------------------------------- | ------------ |
| **Line Coverage**         | % dòng code được thực thi                      | > 80%        |
| **Branch Coverage**       | % nhánh if/else được kiểm tra                  | > 70%        |
| **Method Coverage**       | % method được gọi                              | > 90%        |

> ⚠️ **Cảnh báo:** 100% coverage không có nghĩa là không có bug. Coverage là chỉ số hỗ trợ, không phải mục tiêu cuối cùng.

---

## 🛠️ Công Cụ Testing .NET

| Công Cụ                          | Mục Đích                                         |
| -------------------------------- | ------------------------------------------------ |
| **xUnit**                        | Unit testing framework chính                     |
| **Moq**                          | Mocking library phổ biến nhất                    |
| **NSubstitute**                  | Mocking library với syntax tự nhiên hơn           |
| **FluentAssertions**             | Assertion library dễ đọc, thông báo lỗi rõ hơn   |
| **Coverlet**                     | Đo test coverage cho .NET                        |
| **WebApplicationFactory**        | Test ASP.NET Core apps in-process                |
| **Testcontainers**               | Chạy Docker containers trong test (DB thực)      |
| **BenchmarkDotNet**              | Đo lường hiệu năng micro-benchmark               |
| **NBomber**                      | Load testing và stress testing                   |
| **Bogus**                        | Tạo dữ liệu giả (fake data) cho test             |

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

1. **Sự khác nhau giữa mock và stub là gì?**
   - Stub trả về dữ liệu cố định; Mock kiểm tra hành vi (có được gọi không, với tham số gì)

2. **Khi nào dùng unit test, khi nào dùng integration test?**
   - Unit: logic nghiệp vụ thuần túy; Integration: khi cần xác minh luồng đầy đủ qua nhiều lớp

3. **TDD là gì và lợi ích của nó?**
   - Test-Driven Development: viết test trước → code pass test → refactor; buộc thiết kế API trước

4. **Test coverage 100% có đảm bảo không có bug không?**
   - Không — coverage chỉ đo code được thực thi, không đảm bảo mọi trường hợp biên được kiểm tra

5. **Làm thế nào để test code có phụ thuộc external (DB, API)?**
   - Dùng mock/stub để cô lập; hoặc dùng Testcontainers để chạy DB thực trong Docker

---

## 🔗 Điều Hướng

| Trước                                       | Section Này    | Tiếp Theo                              |
| ------------------------------------------- | -------------- | -------------------------------------- |
| [05-entity-framework/](../05-entity-framework/) | **06-testing** | [07-performance/](../07-performance/)  |

---

*Cập nhật: 2026-06-02 | .NET 8 / .NET 9*
