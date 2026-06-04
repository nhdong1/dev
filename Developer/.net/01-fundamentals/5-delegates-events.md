# 5 — Delegates, Func, Action và Events

> Delegate là nền tảng của callback, LINQ, async/await, và event-driven programming. Hiểu delegate giúp bạn hiểu sâu hơn toàn bộ hệ sinh thái .NET.

---

## 📌 Delegate Là Gì?

**Delegate** (ủy quyền) là kiểu tham chiếu đại diện cho một **phương thức** (method) có signature (chữ ký — kiểu tham số và kiểu trả về) nhất định. Nói đơn giản: delegate là **con trỏ hàm type-safe** (type-safe function pointer) trong C#.

```csharp
// Khai báo delegate type
delegate int MathOperation(int a, int b);

// Phương thức có signature phù hợp
int Add(int a, int b) => a + b;
int Multiply(int a, int b) => a * b;

// Sử dụng
MathOperation op = Add;
Console.WriteLine(op(3, 4)); // 7

op = Multiply; // Đổi hàm được ủy quyền
Console.WriteLine(op(3, 4)); // 12
```

---

## 1. Multicast Delegate — Delegate Đa Điểm

Một delegate có thể trỏ đến **nhiều phương thức** cùng lúc.

```csharp
delegate void Logger(string message);

void LogToConsole(string msg) => Console.WriteLine($"[Console] {msg}");
void LogToFile(string msg) => File.AppendAllText("log.txt", msg + "\n");

Logger logger = LogToConsole;
logger += LogToFile;  // Thêm handler thứ hai

logger("Hello!"); // Gọi CẢ HAI phương thức theo thứ tự

logger -= LogToConsole; // Xóa một handler
logger("World");        // Chỉ còn LogToFile
```

---

## 2. Built-in Delegate Types — Các Kiểu Delegate Dựng Sẵn

Thay vì tự khai báo delegate, C# cung cấp sẵn:

### `Action` — Đại diện cho phương thức không trả về giá trị (void)

```csharp
Action greet = () => Console.WriteLine("Hello!");
greet();

Action<string> greetPerson = name => Console.WriteLine($"Hello, {name}!");
greetPerson("Alice");

Action<int, int> printSum = (a, b) => Console.WriteLine(a + b);
printSum(3, 4); // 7

// Action có từ 0 đến 16 tham số
Action<T1, T2, ..., T16> // Tối đa 16 tham số
```

### `Func` — Đại diện cho phương thức có trả về giá trị

```csharp
Func<int> getAnswer = () => 42;
Console.WriteLine(getAnswer()); // 42

Func<int, int, int> add = (a, b) => a + b;
Console.WriteLine(add(3, 4)); // 7

Func<string, int> getLength = s => s.Length;
Console.WriteLine(getLength("hello")); // 5

// Quy tắc: Tham số cuối cùng luôn là kiểu trả về
// Func<TInput1, TInput2, ..., TReturn>
Func<string, bool, int, string> transform; // (string, bool, int) → string
```

### `Predicate<T>` — Đặc trường hợp của `Func<T, bool>`

```csharp
Predicate<int> isEven = n => n % 2 == 0;
Console.WriteLine(isEven(4));  // True
Console.WriteLine(isEven(5));  // False

// List.Find, List.FindAll, List.RemoveAll dùng Predicate<T>
var evens = numbers.FindAll(isEven);
```

---

## 3. Lambda Expressions — Biểu Thức Lambda

**Lambda expression** là cú pháp gọn để tạo anonymous functions (hàm ẩn danh).

```csharp
// Expression lambda — biểu thức đơn
Func<int, int> square = x => x * x;

// Statement lambda — khối lệnh
Func<int, int, int> maxOf = (a, b) =>
{
    if (a > b) return a;
    return b;
};

// Không có tham số
Action sayHi = () => Console.WriteLine("Hi!");

// Nhiều tham số
Func<int, int, bool> greaterThan = (a, b) => a > b;
```

### Closure — Đóng Gói Biến Ngoài

Lambda có thể **capture** (đóng gói, nắm bắt) biến từ scope bên ngoài.

```csharp
int multiplier = 3;
Func<int, int> triple = x => x * multiplier; // Capture biến ngoài

Console.WriteLine(triple(5)); // 15
multiplier = 10;
Console.WriteLine(triple(5)); // 50 ← dùng giá trị MỚI của multiplier!
```

**Cảnh báo:** Lambda capture biến **by reference** (theo tham chiếu), không phải by value!

```csharp
// Bug kinh điển với closure trong loop
var actions = new List<Action>();
for (int i = 0; i < 5; i++)
{
    actions.Add(() => Console.WriteLine(i)); // Capture 'i' by reference
}
actions.ForEach(a => a()); // In: 5 5 5 5 5  ← tất cả dùng i=5!

// ✅ Fix: capture giá trị tại thời điểm đó
for (int i = 0; i < 5; i++)
{
    int captured = i;
    actions.Add(() => Console.WriteLine(captured));
}
actions.ForEach(a => a()); // In: 0 1 2 3 4
```

---

## 4. Events — Sự Kiện

**Event** (sự kiện) là cơ chế publish-subscribe (xuất bản-đăng ký) được xây dựng trên delegate. Event cung cấp encapsulation (đóng gói) — bên ngoài class chỉ được `+=` (subscribe) và `-=` (unsubscribe), không thể invoke (gọi) trực tiếp.

### Khai báo và sử dụng Event

```csharp
public class Button
{
    // Khai báo event với EventHandler delegate
    public event EventHandler? Clicked;

    // Phương thức nội bộ trigger event (kích hoạt sự kiện)
    protected virtual void OnClicked()
    {
        // Thread-safe invoke (gọi an toàn): copy trước khi kiểm tra null
        Clicked?.Invoke(this, EventArgs.Empty);
    }

    public void SimulateClick() => OnClicked();
}

// Sử dụng
var button = new Button();

// Subscribe — đăng ký lắng nghe sự kiện
button.Clicked += (sender, args) => Console.WriteLine("Đã nhấn!");
button.Clicked += HandleClick; // Có thể có nhiều handler

void HandleClick(object? sender, EventArgs e)
{
    Console.WriteLine($"Handler 2: {sender?.GetType().Name}");
}

button.SimulateClick();
// Output:
// Đã nhấn!
// Handler 2: Button

// Unsubscribe — hủy đăng ký
button.Clicked -= HandleClick;
```

### EventHandler Generic — Truyền Dữ Liệu Theo Event

```csharp
// Tạo EventArgs tùy chỉnh
public class OrderEventArgs : EventArgs
{
    public int OrderId { get; init; }
    public decimal Amount { get; init; }
}

public class OrderService
{
    // EventHandler<T> — sự kiện mang dữ liệu tùy chỉnh
    public event EventHandler<OrderEventArgs>? OrderPlaced;

    public void PlaceOrder(int id, decimal amount)
    {
        // ... logic đặt hàng ...
        OrderPlaced?.Invoke(this, new OrderEventArgs { OrderId = id, Amount = amount });
    }
}

// Sử dụng
var service = new OrderService();
service.OrderPlaced += (sender, e) =>
{
    Console.WriteLine($"Đơn hàng #{e.OrderId}: {e.Amount:C}");
};

service.PlaceOrder(123, 99.99m);
```

### Tự định nghĩa Custom Delegate cho Event

```csharp
// Khai báo delegate tùy chỉnh
public delegate void PriceChangedEventHandler(decimal oldPrice, decimal newPrice);

public class Stock
{
    private decimal _price;
    public event PriceChangedEventHandler? PriceChanged;

    public decimal Price
    {
        get => _price;
        set
        {
            if (_price != value)
            {
                decimal old = _price;
                _price = value;
                PriceChanged?.Invoke(old, value);
            }
        }
    }
}
```

---

## 5. So Sánh: Event vs Direct Delegate

```csharp
public class Publisher
{
    // ✅ Event — encapsulated (có đóng gói)
    public event Action<string>? MessageReceived;

    // ❌ Public delegate field — không đóng gói
    public Action<string>? OnMessage;
}

Publisher pub = new Publisher();

// Với event: chỉ có thể += và -=
pub.MessageReceived += msg => Console.WriteLine(msg);
// pub.MessageReceived = null;    // ❌ Lỗi compile từ ngoài class
// pub.MessageReceived?.Invoke(); // ❌ Lỗi compile từ ngoài class

// Với public field: ai cũng có thể thay thế hoặc invoke
pub.OnMessage = msg => Console.WriteLine(msg); // Ghi đè!
pub.OnMessage?.Invoke("Hi");                   // Gọi từ ngoài!
```

**Quy tắc:** Dùng `event` khi muốn publish-subscribe pattern. Dùng `Action`/`Func` trực tiếp khi là callback một-một.

---

## 6. Functional Patterns với Delegate

### Higher-Order Functions — Hàm Bậc Cao

```csharp
// Hàm nhận delegate làm tham số
T[] Filter<T>(T[] items, Predicate<T> predicate)
{
    return items.Where(x => predicate(x)).ToArray();
}

var evens = Filter(new[] { 1, 2, 3, 4, 5 }, n => n % 2 == 0);

// Hàm trả về delegate
Func<int, int> Multiplier(int factor) => x => x * factor;

var triple = Multiplier(3);
var quintuple = Multiplier(5);

Console.WriteLine(triple(4));     // 12
Console.WriteLine(quintuple(4));  // 20
```

### Composition — Kết Hợp Hàm

```csharp
// Kết hợp hai hàm: compose(f, g)(x) = g(f(x))
Func<TInput, TOutput> Compose<TInput, TMiddle, TOutput>(
    Func<TInput, TMiddle> first,
    Func<TMiddle, TOutput> second) => x => second(first(x));

Func<string, int> parseLength = s => s.Length;
Func<int, bool> isLong = n => n > 10;

Func<string, bool> isLongString = Compose(parseLength, isLong);

Console.WriteLine(isLongString("short"));           // False
Console.WriteLine(isLongString("this is a long text")); // True
```

---

## 7. Expression Trees — Cây Biểu Thức

`Expression<TDelegate>` lưu trữ lambda dưới dạng **cấu trúc dữ liệu** (không phải mã thực thi), cho phép phân tích và biên dịch tại runtime. EF Core dùng Expression Trees để chuyển LINQ thành SQL.

```csharp
// Func<int, bool> — compiled, thực thi ngay
Func<int, bool> isEvenFunc = x => x % 2 == 0;

// Expression<Func<int, bool>> — lưu cây biểu thức, chưa thực thi
Expression<Func<int, bool>> isEvenExpr = x => x % 2 == 0;

// Compile expression tree thành delegate khi cần
Func<int, bool> compiled = isEvenExpr.Compile();
Console.WriteLine(compiled(4)); // True

// Phân tích cấu trúc (EF Core làm điều này để tạo SQL)
Console.WriteLine(isEvenExpr.Body);         // (x % 2) == 0
Console.WriteLine(isEvenExpr.Parameters[0].Name); // x
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Sự khác biệt giữa `Action`, `Func`, và `delegate`?**  
A: `Action` là built-in delegate không trả về giá trị. `Func` là built-in delegate có trả về giá trị (tham số cuối là kiểu trả về). `delegate` là từ khóa để khai báo custom delegate types. `Action` và `Func` là generic delegates dựng sẵn giúp tránh khai báo thừa.

**Q: Event khác delegate field ở điểm nào?**  
A: `event` cung cấp encapsulation — ngoài class chỉ được `+=` và `-=`, không được gán (`=`) hay gọi trực tiếp. Delegate field public thì ai cũng thể ghi đè và invoke.

**Q: Closure trong C# là gì?**  
A: Lambda có thể capture biến từ scope bên ngoài. Biến được capture by reference, không phải by value — nghĩa là thay đổi biến gốc sau khi định nghĩa lambda sẽ ảnh hưởng kết quả khi lambda được gọi.

**Q: Tại sao dùng `EventHandler<T>` thay vì delegate tùy chỉnh?**  
A: Convention trong .NET — tất cả events dùng pattern `(sender, eventArgs)`. Nhất quán, dễ đọc, tương thích với infrastructure như WinForms, WPF, ASP.NET.

---

## ✅ Checklist Tự Kiểm Tra

- [ ] Giải thích delegate là gì và so sánh với interface
- [ ] Viết event với EventHandler và EventArgs tùy chỉnh
- [ ] Biết sự khác biệt giữa `event` và public delegate field
- [ ] Giải thích closure và bug capture trong loop
- [ ] Dùng được `Action`, `Func`, `Predicate` đúng chỗ
- [ ] Biết Expression Trees được EF Core dùng như thế nào

---

**Tiếp theo:** [6-exception-handling.md](./6-exception-handling.md) — Xử Lý Ngoại Lệ
