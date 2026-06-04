# OOP & Design Patterns — Lập Trình Hướng Đối Tượng & Mẫu Thiết Kế

> Nền tảng thiết kế phần mềm chất lượng cao: OOP — Object-Oriented Programming — Lập Trình Hướng Đối Tượng, các nguyên lý SOLID, và 23 mẫu thiết kế kinh điển của GoF — Gang of Four.

---

## 📚 Nội Dung Section Này

| File | Nội Dung | Trạng Thái |
| ---- | -------- | ---------- |
| [1-solid-principles.md](./1-solid-principles.md) | SRP, OCP, LSP, ISP, DIP với ví dụ C# | ✅ |
| [2-creational-patterns.md](./2-creational-patterns.md) | Singleton, Factory, Abstract Factory, Builder, Prototype | ✅ |
| [3-structural-patterns.md](./3-structural-patterns.md) | Adapter, Decorator, Facade, Proxy, Composite | ✅ |
| [4-behavioral-patterns.md](./4-behavioral-patterns.md) | Strategy, Observer, Mediator, Command, Chain of Responsibility | ✅ |
| [5-anti-patterns.md](./5-anti-patterns.md) | God Object, Tight Coupling, Service Locator — cái cần tránh | ✅ |

---

## 🎯 Tại Sao Section Này Quan Trọng

Trong **mọi cuộc phỏng vấn .NET**, ít nhất 3–5 câu hỏi sẽ liên quan đến:

- Giải thích nguyên lý SOLID với ví dụ cụ thể
- Bạn đã dùng Design Pattern nào, trong tình huống nào
- Nhận biết và sửa anti-pattern trong đoạn code cho sẵn

OOP và Design Patterns không chỉ là lý thuyết — chúng là ngôn ngữ chung để lập trình viên giao tiếp về thiết kế phần mềm.

---

## 🔑 Bốn Trụ Cột OOP — Pillars of OOP

### 1. Encapsulation — Đóng Gói

Che giấu chi tiết cài đặt bên trong, chỉ lộ ra những gì cần thiết qua interface — giao diện công khai.

```csharp
public class BankAccount
{
    private decimal _balance; // ẩn, chỉ class này truy cập

    public decimal Balance => _balance; // chỉ đọc từ bên ngoài

    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("Số tiền phải dương");
        _balance += amount;
    }
}
```

**Lợi ích:** Thay đổi nội bộ không phá vỡ code bên ngoài.

### 2. Inheritance — Kế Thừa

Lớp con (derived class) kế thừa thuộc tính và phương thức của lớp cha (base class), có thể override — ghi đè để thay đổi hành vi.

```csharp
public abstract class Shape // lớp trừu tượng
{
    public abstract double Area(); // buộc subclass phải triển khai
    public string Describe() => $"Hình có diện tích {Area():F2}";
}

public class Circle : Shape
{
    private double _radius;
    public Circle(double radius) => _radius = radius;
    public override double Area() => Math.PI * _radius * _radius;
}
```

**Cảnh báo:** Ưu tiên composition — kết hợp đối tượng hơn inheritance — kế thừa sâu (> 2 cấp thường là dấu hiệu xấu).

### 3. Polymorphism — Đa Hình

Cùng một interface, nhiều cách thực thi khác nhau. Có hai loại:
- **Compile-time polymorphism** — Đa hình lúc biên dịch: method overloading — nạp chồng phương thức
- **Runtime polymorphism** — Đa hình lúc chạy: method overriding — ghi đè phương thức qua virtual/override

```csharp
List<Shape> shapes = [new Circle(5), new Rectangle(4, 6), new Triangle(3, 4, 5)];

foreach (var shape in shapes)
    Console.WriteLine(shape.Describe()); // gọi Area() đúng của từng loại
```

### 4. Abstraction — Trừu Tượng Hóa

Mô hình hóa thực thể phức tạp bằng cách chỉ giữ lại những đặc điểm cốt lõi, bỏ qua chi tiết không cần thiết. Thể hiện qua `abstract class` và `interface`.

```csharp
public interface IPaymentGateway // chỉ định "cái gì", không định nghĩa "như thế nào"
{
    Task<PaymentResult> ChargeAsync(decimal amount, string currency);
    Task<RefundResult> RefundAsync(string transactionId);
}

// Stripe, PayPal, VNPay đều implement IPaymentGateway theo cách riêng
```

---

## 📐 SOLID — Năm Nguyên Lý Thiết Kế

| Chữ | Nguyên Lý | Nội Dung Tóm Tắt |
| --- | --------- | ---------------- |
| **S** | Single Responsibility — Trách Nhiệm Đơn | Một class chỉ có một lý do để thay đổi |
| **O** | Open/Closed — Mở/Đóng | Mở rộng mà không sửa code hiện có |
| **L** | Liskov Substitution — Thay Thế Liskov | Subclass phải thay thế được base class |
| **I** | Interface Segregation — Tách Biệt Giao Diện | Không ép implement những gì không dùng |
| **D** | Dependency Inversion — Đảo Ngược Phụ Thuộc | Phụ thuộc vào abstraction, không vào implementation |

Chi tiết với ví dụ C# đầy đủ → [1-solid-principles.md](./1-solid-principles.md)

---

## 🏗️ Design Patterns — Mẫu Thiết Kế

GoF — Gang of Four — nhóm 4 tác giả của cuốn sách "Design Patterns: Elements of Reusable Object-Oriented Software" (1994) phân loại 23 mẫu thiết kế thành 3 nhóm:

### Creational Patterns — Mẫu Khởi Tạo

> Giải quyết vấn đề **tạo đối tượng** linh hoạt, tránh phụ thuộc vào class cụ thể.

| Pattern | Mục Đích | Ví Dụ Thực Tế |
| ------- | -------- | ------------- |
| **Singleton** | Đảm bảo chỉ có một instance | Configuration, Logger, Connection Pool |
| **Factory Method** | Để subclass quyết định tạo class nào | ILoggerFactory, DbProviderFactory |
| **Abstract Factory** | Tạo nhóm đối tượng liên quan | UI themes (Light/Dark), cross-platform widgets |
| **Builder** | Xây dựng object phức tạp từng bước | SqlConnectionStringBuilder, IHostBuilder |
| **Prototype** | Tạo object bằng cách clone | Deep copy, object templates |

### Structural Patterns — Mẫu Cấu Trúc

> Giải quyết vấn đề **kết hợp các class** thành cấu trúc lớn hơn.

| Pattern | Mục Đích | Ví Dụ Thực Tế |
| ------- | -------- | ------------- |
| **Adapter** | Chuyển đổi interface không tương thích | Tích hợp API bên thứ ba |
| **Decorator** | Thêm chức năng mà không sửa class gốc | ASP.NET Middleware, logging wrapper |
| **Facade** | Cung cấp interface đơn giản cho hệ thống phức tạp | Service layer che giấu EF Core |
| **Proxy** | Kiểm soát truy cập vào object | Lazy loading, caching proxy, auth proxy |
| **Composite** | Xử lý cây object đồng nhất | File system, UI component tree |

### Behavioral Patterns — Mẫu Hành Vi

> Giải quyết vấn đề **giao tiếp và phân chia trách nhiệm** giữa các đối tượng.

| Pattern | Mục Đích | Ví Dụ Thực Tế |
| ------- | -------- | ------------- |
| **Strategy** | Hoán đổi thuật toán khi chạy | Sorting, payment processing, validation |
| **Observer** | Thông báo nhiều đối tượng khi state thay đổi | Event system, SignalR, domain events |
| **Mediator** | Trung gian điều phối giao tiếp | MediatR, chat room |
| **Command** | Đóng gói yêu cầu thành object | Undo/Redo, queue commands, CQRS |
| **Chain of Responsibility** | Truyền request qua chuỗi handler | ASP.NET Middleware, validation pipeline |

---

## 🚫 Anti-Patterns — Mẫu Thiết Kế Sai

Anti-pattern — Mẫu thiết kế sai — là những giải pháp trông có vẻ hợp lý nhưng thực tế gây ra nhiều vấn đề hơn là giải quyết.

| Anti-Pattern | Triệu Chứng | Cách Sửa |
| ------------ | ----------- | -------- |
| **God Object** | Class làm quá nhiều thứ | Tách thành các class nhỏ theo SRP |
| **Tight Coupling** | Thay đổi một class phá vỡ nhiều class khác | Dùng DI và interface |
| **Service Locator** | "Hộp ma thuật" tự tìm dependency | Dùng Dependency Injection đúng cách |
| **Anemic Domain Model** | Domain object chỉ có data, không có behavior | Đưa business logic vào domain |
| **Premature Optimization** | Tối ưu trước khi đo lường | Đo, đo, mới tối ưu |

---

## 🎓 Câu Hỏi Phỏng Vấn Thường Gặp

### Cơ Bản

1. **"Giải thích 4 trụ cột OOP"** — Encapsulation, Inheritance, Polymorphism, Abstraction với ví dụ cụ thể
2. **"SOLID là gì? Cho ví dụ nguyên lý bạn áp dụng nhiều nhất"**
3. **"Sự khác nhau giữa abstract class và interface?"**
4. **"Khi nào dùng interface, khi nào dùng abstract class?"**

### Nâng Cao

5. **"Design Pattern nào bạn hay dùng nhất trong thực tế?"** — Chuẩn bị câu chuyện STAR
6. **"Giải thích Decorator vs Inheritance — khi nào dùng cái nào?"**
7. **"MediatR dùng Pattern nào? Lợi ích là gì?"**
8. **"Service Locator bị coi là anti-pattern — tại sao?"**

### Thực Chiến

9. **"Cho đoạn code này, vi phạm nguyên lý SOLID nào? Sửa như thế nào?"**
10. **"Hệ thống payment của bạn cần hỗ trợ nhiều provider — thiết kế thế nào?"**

---

## 🗺️ Thứ Tự Học Khuyến Nghị

```
1. SOLID Principles (quan trọng nhất, học trước)
   → Đây là nền tảng để hiểu mọi pattern
   
2. Creational Patterns (Singleton, Factory, Builder hay gặp nhất)
   → Liên hệ với DI Container trong ASP.NET Core
   
3. Structural Patterns (Decorator, Adapter, Proxy)
   → Liên hệ với Middleware Pipeline
   
4. Behavioral Patterns (Strategy, Observer, Mediator)
   → Liên hệ với MediatR, SignalR, Event handling
   
5. Anti-Patterns (học để nhận biết và tránh)
   → Đọc code review với con mắt phê bình
```

---

## 🔗 Liên Hệ Với Các Section Khác

| Kiến Thức Ở Đây | Ứng Dụng Ở |
| ---------------- | ---------- |
| SOLID → DIP | [04-aspnet-core/2-dependency-injection.md](../04-aspnet-core/2-dependency-injection.md) |
| Strategy Pattern | [09-architecture/3-cqrs-pattern.md](../09-architecture/3-cqrs-pattern.md) |
| Observer Pattern | Domain Events trong [09-architecture/2-ddd-fundamentals.md](../09-architecture/2-ddd-fundamentals.md) |
| Decorator Pattern | [04-aspnet-core/1-middleware-pipeline.md](../04-aspnet-core/1-middleware-pipeline.md) |
| Mediator Pattern | MediatR trong [09-architecture/3-cqrs-pattern.md](../09-architecture/3-cqrs-pattern.md) |

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Trạng Thái:** ✅ Hoàn thành
