# TDD — Test-Driven Development — Phát Triển Hướng Kiểm Thử

> Red → Green → Refactor: viết test trước, code sau — phương pháp buộc thiết kế tốt từ đầu.

---

## 🎯 TDD Là Gì?

**TDD — Test-Driven Development — Phát triển hướng kiểm thử** là phương pháp phát triển phần mềm trong đó bạn **viết test TRƯỚC KHI viết code**. Nghe có vẻ ngược, nhưng đây là điểm mạnh của TDD.

### Vòng Lặp Red → Green → Refactor

```
     ┌─────────────────────────────────────────┐
     │                                         │
     ▼                                         │
  🔴 RED                                       │
  Viết một test thất bại                       │
  (test mô tả hành vi muốn có)                 │
     │                                         │
     ▼                                         │
  🟢 GREEN                                     │
  Viết code tối thiểu để test pass             │
  (không cần code đẹp, chỉ cần pass)           │
     │                                         │
     ▼                                         │
  🔵 REFACTOR                                  │
  Dọn dẹp code, cải thiện thiết kế             │
  (test vẫn phải pass sau refactor)            │
     │                                         │
     └─────────────────────────────────────────┘
           (lặp lại cho feature tiếp theo)
```

---

## 💡 Tại Sao TDD?

### Lợi Ích Thực Tế

| Lợi Ích                         | Giải Thích                                                          |
| -------------------------------- | ------------------------------------------------------------------- |
| **Thiết kế API trước**           | Buộc bạn nghĩ về interface trước implementation                     |
| **Mạng lưới an toàn**            | Test suite phủ 100% code bạn viết                                   |
| **Tài liệu sống**                | Test mô tả hành vi mong muốn rõ hơn bất kỳ comment nào             |
| **Tự tin refactor**              | Nếu test vẫn xanh, refactor không phá gì                            |
| **Ít debug hơn**                 | Lỗi được phát hiện ngay, không phải sau 3 ngày                      |
| **Thiết kế loose coupling**      | Code khó test → dấu hiệu tight coupling → buộc cải thiện             |

### Salah Lầm Phổ Biến Về TDD

```
❌ "TDD chậm hơn viết code bình thường"
✅ Thực tế: Nhanh hơn ở tổng thể vì ít thời gian debug và rework

❌ "TDD phải viết test cho mọi thứ"
✅ Thực tế: Chỉ test public interface, không test implementation details

❌ "TDD không thể dùng cho code phức tạp"
✅ Thực tế: TDD đặc biệt hữu ích cho code phức tạp — buộc bạn chia nhỏ

❌ "TDD chỉ dành cho unit test"
✅ Thực tế: Có thể dùng TDD cho integration test (ATDD/BDD)
```

---

## 🚀 Demo TDD Từ Đầu Đến Cuối

### Yêu Cầu: Xây Dựng Shopping Cart (Giỏ Hàng)

```
Tính năng:
- Thêm sản phẩm vào giỏ
- Tính tổng tiền
- Áp dụng mã giảm giá (coupon)
- Giỏ hàng trống không tính được total
```

---

### 🔴 Bước 1: RED — Viết Test Thất Bại

**Bắt đầu với test đơn giản nhất:**

```csharp
// ShoppingCartTests.cs
public class ShoppingCartTests
{
    // Test 1: Giỏ hàng mới phải rỗng
    [Fact]
    public void NewCart_ShouldBeEmpty()
    {
        var cart = new ShoppingCart();
        cart.Items.Should().BeEmpty();
    }
}
```

Chạy test → **🔴 FAIL** (vì `ShoppingCart` chưa tồn tại).

---

### 🟢 Bước 2: GREEN — Code Tối Thiểu Để Pass

```csharp
// ShoppingCart.cs — chỉ đủ để test pass
public class ShoppingCart
{
    public List<CartItem> Items { get; } = new();
}

public class CartItem
{
    public string ProductId { get; set; } = "";
    public decimal Price { get; set; }
    public int Quantity { get; set; }
}
```

Chạy test → **🟢 PASS**.

---

### 🔵 Bước 3: REFACTOR — Không Cần Refactor Gì Ở Bước Này

Tiếp tục vòng lặp tiếp theo.

---

### 🔴 Bước 4: RED — Test Tiếp Theo

```csharp
// Test 2: Thêm sản phẩm vào giỏ
[Fact]
public void AddItem_ValidItem_IncreasesItemCount()
{
    var cart = new ShoppingCart();
    cart.AddItem("PROD-001", price: 10m, quantity: 2);
    cart.Items.Should().HaveCount(1);
}
```

Chạy → **🔴 FAIL** (method `AddItem` chưa có).

---

### 🟢 Bước 5: GREEN

```csharp
public class ShoppingCart
{
    public List<CartItem> Items { get; } = new();

    public void AddItem(string productId, decimal price, int quantity)
    {
        Items.Add(new CartItem
        {
            ProductId = productId,
            Price = price,
            Quantity = quantity
        });
    }
}
```

**🟢 PASS**. Tiếp tục.

---

### 🔴 Bước 6: RED — Test Tính Tổng

```csharp
// Test 3: Tính tổng tiền
[Theory]
[InlineData(10, 2, 20)]   // 1 item: giá 10 × số lượng 2 = 20
[InlineData(5, 3, 15)]    // 1 item: giá 5 × số lượng 3 = 15
public void GetTotal_WithSingleItem_ReturnsCorrectTotal(
    decimal price, int qty, decimal expectedTotal)
{
    var cart = new ShoppingCart();
    cart.AddItem("PROD-001", price, qty);
    cart.GetTotal().Should().Be(expectedTotal);
}

// Test 4: Nhiều items
[Fact]
public void GetTotal_WithMultipleItems_SumsAll()
{
    var cart = new ShoppingCart();
    cart.AddItem("P1", 10m, 2); // 20
    cart.AddItem("P2", 5m, 3);  // 15
    cart.GetTotal().Should().Be(35m);
}
```

**🔴 FAIL** — `GetTotal()` chưa có.

---

### 🟢 Bước 7: GREEN

```csharp
public decimal GetTotal()
{
    return Items.Sum(item => item.Price * item.Quantity);
}
```

**🟢 PASS**.

---

### 🔴 Bước 8: RED — Test Giỏ Rỗng

```csharp
// Test 5: Giỏ rỗng không thể checkout
[Fact]
public void Checkout_EmptyCart_ThrowsInvalidOperationException()
{
    var cart = new ShoppingCart();
    Action act = () => cart.Checkout();
    act.Should().Throw<InvalidOperationException>()
       .WithMessage("*empty*");
}
```

**🔴 FAIL**.

---

### 🟢 Bước 9: GREEN

```csharp
public void Checkout()
{
    if (!Items.Any())
        throw new InvalidOperationException("Cannot checkout an empty cart.");
}
```

**🟢 PASS**.

---

### 🔴 Bước 10: RED — Test Coupon (Mã Giảm Giá)

```csharp
// Test 6: Áp dụng coupon giảm %
[Fact]
public void ApplyCoupon_ValidPercentageCoupon_ReducesTotal()
{
    var cart = new ShoppingCart();
    cart.AddItem("P1", 100m, 1); // Tổng 100

    cart.ApplyCoupon("SAVE10", discountPercent: 10); // Giảm 10%

    cart.GetTotal().Should().Be(90m);
}

// Test 7: Coupon không hợp lệ
[Fact]
public void ApplyCoupon_InvalidCode_ThrowsArgumentException()
{
    var cart = new ShoppingCart();
    Action act = () => cart.ApplyCoupon("INVALID", 10);
    act.Should().Throw<ArgumentException>();
}
```

---

### 🟢 Bước 11: GREEN (Naive Implementation — Cài Đặt Đơn Giản)

```csharp
private string? _couponCode;
private decimal _discountPercent;
private static readonly HashSet<string> _validCoupons = new() { "SAVE10", "SAVE20" };

public void ApplyCoupon(string code, decimal discountPercent)
{
    if (!_validCoupons.Contains(code))
        throw new ArgumentException($"Invalid coupon code: {code}");

    _couponCode = code;
    _discountPercent = discountPercent;
}

public decimal GetTotal()
{
    var subtotal = Items.Sum(item => item.Price * item.Quantity);
    if (_couponCode != null)
        return subtotal * (1 - _discountPercent / 100);
    return subtotal;
}
```

**🟢 PASS**.

---

### 🔵 Bước 12: REFACTOR — Dọn Dẹp Code

Nhìn lại code, nhận thấy coupon validation logic nên tách ra:

```csharp
// Refactor: tách coupon thành separate concern
public interface ICouponValidator
{
    bool IsValid(string code);
    decimal GetDiscountPercent(string code);
}

public class ShoppingCart
{
    private readonly ICouponValidator _couponValidator;
    private decimal _discountPercent = 0;

    public ShoppingCart(ICouponValidator? couponValidator = null)
    {
        _couponValidator = couponValidator ?? new NoCouponValidator();
    }

    public List<CartItem> Items { get; } = new();

    public void AddItem(string productId, decimal price, int quantity)
    {
        if (quantity <= 0) throw new ArgumentException("Quantity must be positive.");
        if (price < 0) throw new ArgumentException("Price cannot be negative.");

        var existing = Items.FirstOrDefault(i => i.ProductId == productId);
        if (existing != null)
            existing.Quantity += quantity;
        else
            Items.Add(new CartItem { ProductId = productId, Price = price, Quantity = quantity });
    }

    public void ApplyCoupon(string code, decimal discountPercent)
    {
        if (!_couponValidator.IsValid(code))
            throw new ArgumentException($"Invalid coupon code: {code}");
        _discountPercent = discountPercent;
    }

    public decimal GetTotal()
    {
        var subtotal = Items.Sum(i => i.Price * i.Quantity);
        return subtotal * (1 - _discountPercent / 100);
    }

    public void Checkout()
    {
        if (!Items.Any())
            throw new InvalidOperationException("Cannot checkout an empty cart.");
    }
}
```

Chạy tất cả test → **🟢 TẤT CẢ PASS**. Refactor thành công.

---

## 🧪 TDD Cho Các Tình Huống Phức Tạp

### Test Behavior Xử Lý Lỗi

```csharp
// Luôn test error paths trước happy path — lỗi thường bị bỏ quên
[Theory]
[InlineData(0)]
[InlineData(-1)]
[InlineData(-100)]
public void AddItem_NonPositiveQuantity_ThrowsArgumentException(int quantity)
{
    var cart = new ShoppingCart();
    Action act = () => cart.AddItem("P1", 10m, quantity);
    act.Should().Throw<ArgumentException>()
       .WithParameterName("quantity");
}
```

### Test Trạng Thái Sau Nhiều Operations

```csharp
[Fact]
public void AddItem_SameProductTwice_MergesQuantity()
{
    var cart = new ShoppingCart();
    cart.AddItem("PROD-001", 10m, 2);
    cart.AddItem("PROD-001", 10m, 3); // Thêm lần 2

    // Phải merge, không phải tạo item mới
    cart.Items.Should().HaveCount(1);
    cart.Items.Single().Quantity.Should().Be(5);
}
```

### Test Với Test Doubles Trong TDD

```csharp
[Fact]
public void ApplyCoupon_ValidCode_AppliesDiscount()
{
    // Arrange: setup mock coupon validator
    var mockValidator = new Mock<ICouponValidator>();
    mockValidator.Setup(v => v.IsValid("PROMO20")).Returns(true);
    mockValidator.Setup(v => v.GetDiscountPercent("PROMO20")).Returns(20m);

    var cart = new ShoppingCart(mockValidator.Object);
    cart.AddItem("P1", 100m, 1);

    // Act
    cart.ApplyCoupon("PROMO20", 20m);

    // Assert
    cart.GetTotal().Should().Be(80m); // 100 - 20% = 80
}
```

---

## 📋 ATDD / BDD — Mở Rộng TDD

### ATDD — Acceptance Test-Driven Development

TDD ở cấp độ **acceptance test** (kiểm thử nghiệm thu) — viết test từ góc nhìn business trước khi viết bất kỳ code nào.

```
Stakeholder → Viết Acceptance Criteria
                → Dev viết failing Acceptance Test
                    → Dev viết code (dùng TDD nội bộ)
                        → Acceptance Test pass
```

### BDD — Behavior-Driven Development (Phát Triển Hướng Hành Vi)

BDD dùng ngôn ngữ tự nhiên (Gherkin) để mô tả behavior:

```gherkin
Feature: Shopping Cart Checkout
  Scenario: Successful checkout with valid items
    Given the cart contains 2 items worth $50 each
    When the customer applies coupon "SAVE10"
    And the customer checks out
    Then the total should be $90
    And an order confirmation email should be sent
```

```csharp
// SpecFlow (BDD framework cho .NET)
[Binding]
public class ShoppingCartSteps
{
    private ShoppingCart _cart;
    private decimal _finalTotal;

    [Given(@"the cart contains (\d+) items worth \$(\d+) each")]
    public void GivenCartContainsItems(int count, decimal price)
    {
        _cart = new ShoppingCart();
        for (int i = 0; i < count; i++)
            _cart.AddItem($"PROD-{i}", price, 1);
    }

    [When(@"the customer applies coupon ""(.*)""")]
    public void WhenApplyCoupon(string code)
    {
        _cart.ApplyCoupon(code, 10);
    }

    [Then(@"the total should be \$(\d+)")]
    public void ThenTotalShouldBe(decimal expected)
    {
        _cart.GetTotal().Should().Be(expected);
    }
}
```

---

## ⚠️ Khi Nào KHÔNG Nên Dùng TDD?

```
❌ Exploratory code (thử nghiệm ý tưởng, prototype)
   → Viết code nhanh để học, xóa đi sau

❌ UI/UX code phụ thuộc visual feedback
   → Khó viết test có ý nghĩa trước khi nhìn thấy

❌ Code trivial (getters/setters đơn giản)
   → Test thêm chi phí mà không thêm giá trị

❌ Deadline cực kỳ gấp cho spike/throwaway code
   → Nhưng production code thì nên dùng TDD
```

---

## 🏆 TDD Best Practices — Thực Hành Tốt Nhất

### 1. Baby Steps — Bước Nhỏ

```
❌ Viết 5 test cùng lúc → quá nhiều để implement
✅ Một test, một implementation, lặp lại
```

### 2. Test Behavior, Not Implementation

```csharp
// ❌ BAD: test implementation detail
[Fact]
public void AddItem_UsesListInternally()
{
    var cart = new ShoppingCart();
    cart.AddItem("P1", 10m, 1);
    // Test dùng reflection để check _items field... quá fragile
}

// ✅ GOOD: test behavior (behavior = hành vi)
[Fact]
public void AddItem_ValidItem_ItemAppearsInCart()
{
    var cart = new ShoppingCart();
    cart.AddItem("P1", 10m, 1);
    cart.Items.Should().ContainSingle(i => i.ProductId == "P1");
}
```

### 3. One Assert Per Test (Một Assert Mỗi Test)

```csharp
// Mỗi test kiểm tra một điều — khi fail biết chính xác cái gì sai
[Fact]
public void AddItem_SetsCorrectProductId() { ... }

[Fact]
public void AddItem_SetsCorrectPrice() { ... }

[Fact]
public void AddItem_SetsCorrectQuantity() { ... }
```

### 4. Test Tên Phải Là Tài Liệu

```csharp
// Tên test đọc như specification (đặc tả)
public class ShoppingCartSpecification
{
    [Fact] public void A_new_cart_should_be_empty() { }
    [Fact] public void Adding_an_item_should_increase_item_count() { }
    [Fact] public void The_total_should_reflect_all_items_and_quantities() { }
    [Fact] public void Checking_out_an_empty_cart_should_not_be_allowed() { }
}
```

---

## 📊 TDD Metrics — Đo Lường Hiệu Quả TDD

| Chỉ Số                         | Mục Tiêu Lý Tưởng               |
| ------------------------------ | -------------------------------- |
| **Test/Code ratio**            | Gần 1:1 (số dòng test ≈ code)   |
| **Time in RED phase**          | < 2 phút mỗi vòng               |
| **Test suite duration**        | < 30 giây cho unit tests         |
| **Defect rate after TDD**      | Giảm 40–80% so với không dùng TDD |
| **Time to implement feature**  | Ban đầu chậm hơn 10–15%, sau quen thì ngang hoặc nhanh hơn |

---

## 🎯 Câu Hỏi Phỏng Vấn TDD

**Q: TDD khác gì với viết test sau?**
> TDD: Test định nghĩa design → code theo test → design tự nhiên tốt hơn.
> Test after: Code có trước → test chỉ xác nhận, không ảnh hưởng design.

**Q: Khi nào dùng TDD, khi nào không?**
> Dùng TDD cho business logic phức tạp, domain rules, edge cases.
> Không bắt buộc cho boilerplate code, configs, migration scripts.

**Q: TDD có nghĩa là 100% code coverage không?**
> Không nhất thiết. TDD đảm bảo coverage cao nhưng mục tiêu là thiết kế tốt, không phải coverage số.

**Q: Làm TDD với legacy code (code cũ) như thế nào?**
> Dùng kỹ thuật "Characterization Tests" — viết test mô tả behavior hiện tại của legacy code trước khi refactor.

---

## 🔗 Liên Quan

- [1-unit-testing.md](./1-unit-testing.md) — Nền tảng viết test trong TDD
- [2-mocking.md](./2-mocking.md) — Mock dependencies trong TDD
- [3-integration-testing.md](./3-integration-testing.md) — ATDD với integration tests

---

*Cập nhật: 2026-06-02 | .NET 8 | xUnit 2.x*
