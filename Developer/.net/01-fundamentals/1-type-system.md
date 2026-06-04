# 1 — Type System (Hệ Thống Kiểu) trong C#

> Hiểu hệ thống kiểu là nền tảng để viết code C# đúng và hiệu quả. Đây là chủ đề hỏi thường xuyên trong phỏng vấn.

---

## 📌 Tổng Quan

C# là ngôn ngữ **statically typed** (kiểu tĩnh) — mọi biến đều có kiểu xác định tại thời điểm biên dịch (**compile time**). Hệ thống kiểu chia làm hai nhóm lớn:

```
System.Object (gốc của tất cả)
├── Value Types (Kiểu giá trị)    — lưu trực tiếp giá trị
│   ├── struct (int, double, bool, DateTime, ...)
│   ├── enum
│   └── Nullable<T>  (int?, bool?, ...)
└── Reference Types (Kiểu tham chiếu)  — lưu địa chỉ bộ nhớ
    ├── class
    ├── interface
    ├── delegate
    ├── array
    └── string (đặc biệt: immutable reference type)
```

---

## 1. Value Types — Kiểu Giá Trị

### Đặc điểm cốt lõi

- Biến **chứa trực tiếp giá trị**, không phải địa chỉ
- Phân bổ trên **Stack** (ngăn xếp) khi là biến cục bộ; trên **Heap** (đống) khi là field của class
- Khi gán (`=`) hoặc truyền vào hàm: **copy toàn bộ dữ liệu** (pass by value)
- Mặc định không thể `null` (trừ khi dùng `Nullable<T>`)

### Các kiểu giá trị phổ biến

| Kiểu | Mô tả | Kích thước |
|------|-------|-----------|
| `int` | Số nguyên 32-bit | 4 bytes |
| `long` | Số nguyên 64-bit | 8 bytes |
| `double` | Số thực 64-bit | 8 bytes |
| `decimal` | Số thực chính xác cao (tài chính) | 16 bytes |
| `bool` | Đúng/sai | 1 byte |
| `char` | Ký tự Unicode | 2 bytes |
| `struct` | Kiểu tổng hợp do người dùng định nghĩa | Tùy |
| `enum` | Tập hằng số có tên | Tùy (mặc định int) |
| `DateTime` | Ngày giờ | 8 bytes |

### Ví dụ minh họa copy semantics

```csharp
int a = 10;
int b = a;   // b là bản sao của a
b = 20;

Console.WriteLine(a); // 10  ← a không bị ảnh hưởng
Console.WriteLine(b); // 20
```

```csharp
// struct cũng copy toàn bộ
struct Point
{
    public int X;
    public int Y;
}

Point p1 = new Point { X = 1, Y = 2 };
Point p2 = p1;   // toàn bộ dữ liệu được copy
p2.X = 99;

Console.WriteLine(p1.X); // 1  ← p1 không thay đổi
Console.WriteLine(p2.X); // 99
```

---

## 2. Reference Types — Kiểu Tham Chiếu

### Đặc điểm cốt lõi

- Biến **chứa địa chỉ** (reference) trỏ đến đối tượng trên **Heap**
- Khi gán (`=`) hoặc truyền vào hàm: **copy địa chỉ**, không copy dữ liệu
- Nhiều biến có thể trỏ đến cùng một đối tượng → thay đổi qua một biến ảnh hưởng tất cả
- Mặc định có thể `null`

### Ví dụ minh họa reference semantics

```csharp
class Person
{
    public string Name { get; set; }
}

Person p1 = new Person { Name = "Alice" };
Person p2 = p1;   // p2 trỏ đến CÙNG đối tượng với p1
p2.Name = "Bob";

Console.WriteLine(p1.Name); // "Bob"  ← p1 cũng thay đổi!
Console.WriteLine(p2.Name); // "Bob"
```

```csharp
// Để copy độc lập: phải implement ICloneable hoặc tự viết
Person p3 = new Person { Name = p1.Name }; // shallow copy thủ công
p3.Name = "Charlie";
Console.WriteLine(p1.Name); // "Bob"  ← p1 không ảnh hưởng
```

---

## 3. Sự Khác Biệt Khi Truyền Vào Hàm

### Pass by value (mặc định)

```csharp
void DoubleValue(int x)
{
    x = x * 2; // chỉ thay đổi bản copy trong hàm
}

int number = 5;
DoubleValue(number);
Console.WriteLine(number); // 5  ← không thay đổi
```

### Pass by reference với `ref`

```csharp
void DoubleValue(ref int x)
{
    x = x * 2; // thay đổi biến gốc
}

int number = 5;
DoubleValue(ref number);
Console.WriteLine(number); // 10
```

### `out` — trả về nhiều giá trị

```csharp
bool TryParse(string input, out int result)
{
    return int.TryParse(input, out result);
}

if (TryParse("42", out int value))
{
    Console.WriteLine(value); // 42
}
```

### `in` — pass by reference nhưng readonly (chỉ đọc)

```csharp
// Hiệu quả cho struct lớn: không copy, không thay đổi
void PrintPoint(in Point p)
{
    Console.WriteLine($"({p.X}, {p.Y})");
    // p.X = 0; // Lỗi compile: không thể gán vì là in parameter
}
```

---

## 4. Boxing và Unboxing

**Boxing** — đóng gói: chuyển value type sang reference type (object), phân bổ Heap.  
**Unboxing** — mở gói: trích xuất lại value type từ object.

```csharp
int value = 42;
object boxed = value;   // Boxing — tốn kém: phân bổ Heap, copy dữ liệu

int unboxed = (int)boxed; // Unboxing — tốn kém: kiểm tra kiểu, copy dữ liệu
```

### Vấn đề hiệu năng với Boxing

```csharp
// ❌ Xấu: Boxing xảy ra mỗi lần thêm int vào ArrayList
var list = new System.Collections.ArrayList();
for (int i = 0; i < 1_000_000; i++)
{
    list.Add(i); // Boxing mỗi lần!
}

// ✅ Tốt: List<int> generic — không boxing
var genericList = new List<int>();
for (int i = 0; i < 1_000_000; i++)
{
    genericList.Add(i); // Không boxing
}
```

**Quy tắc:** Dùng **generic collections** (tập hợp generic) thay vì `ArrayList`, `Hashtable` cũ để tránh boxing.

---

## 5. String — Kiểu Đặc Biệt

`string` là **reference type** nhưng có behavior đặc biệt:

### Immutability — Bất biến

```csharp
string s1 = "hello";
string s2 = s1;
s1 = s1 + " world"; // Tạo string MỚI, không sửa string cũ

Console.WriteLine(s1); // "hello world"
Console.WriteLine(s2); // "hello"  ← s2 vẫn trỏ đến string cũ
```

### String Interning — Tái sử dụng chuỗi

```csharp
string a = "hello";
string b = "hello";
Console.WriteLine(ReferenceEquals(a, b)); // True — cùng tham chiếu (interned)

string c = new string("hello".ToCharArray());
Console.WriteLine(ReferenceEquals(a, c)); // False — đối tượng mới
Console.WriteLine(a == c);               // True  — so sánh giá trị
```

### StringBuilder — Khi nối nhiều chuỗi

```csharp
// ❌ Xấu: tạo nhiều string object trung gian
string result = "";
for (int i = 0; i < 10000; i++)
{
    result += i.ToString(); // Tạo string mới mỗi lần!
}

// ✅ Tốt: StringBuilder tái sử dụng buffer
var sb = new StringBuilder();
for (int i = 0; i < 10000; i++)
{
    sb.Append(i);
}
string result = sb.ToString();
```

---

## 6. Nullable Types — Kiểu Có Thể Null

### Nullable Value Types — Kiểu giá trị cho phép null

```csharp
int? age = null;         // Nullable<int>
double? score = 9.5;

if (age.HasValue)
{
    Console.WriteLine(age.Value);
}

// Null coalescing operator — toán tử hợp nhất null
int display = age ?? 0;          // 0 nếu null
int display2 = age ?? default;   // giống trên cho int

// Null coalescing assignment — gán nếu null
age ??= 18;  // age = 18 nếu age == null
```

### Nullable Reference Types (C# 8+) — Kiểu tham chiếu nullable

```csharp
// Bật trong .csproj: <Nullable>enable</Nullable>

string nonNullable = "hello";  // không thể null — compiler cảnh báo nếu gán null
string? nullable = null;       // có thể null — phải kiểm tra trước khi dùng

// Null-conditional operator — toán tử điều kiện null
int? length = nullable?.Length; // null nếu nullable == null

// Null-forgiving operator — cho compiler biết "tôi chắc không null"
string forced = nullable!; // cẩn thận khi dùng
```

---

## 7. Các Kiểu Giá Trị Đặc Biệt Trong C# Hiện Đại

### `record` — Kiểu Bất Biến Với Equality (C# 9+)

```csharp
// record class (reference type) — equality theo giá trị, không phải tham chiếu
record Person(string FirstName, string LastName);

var p1 = new Person("Alice", "Smith");
var p2 = new Person("Alice", "Smith");
Console.WriteLine(p1 == p2); // True — so sánh theo giá trị

// record struct (value type) — C# 10+
record struct Point(int X, int Y);
```

### `readonly struct` — Struct Không Thể Thay Đổi

```csharp
readonly struct Temperature
{
    public double Celsius { get; }
    public Temperature(double celsius) => Celsius = celsius;
    public double Fahrenheit => Celsius * 9 / 5 + 32;
}
```

### `ref struct` — Struct Chỉ Tồn Tại Trên Stack

```csharp
// Span<T> là ref struct — không thể boxing, không thể là field của class
Span<int> span = stackalloc int[10];
```

---

## 8. Cheat Sheet: Khi Nào Dùng Gì

| Tình Huống | Nên Dùng | Lý Do |
|-----------|----------|-------|
| Dữ liệu nhỏ, bất biến (điểm, kích thước) | `struct` | Không Heap, copy rẻ |
| Đối tượng có identity (người dùng, đơn hàng) | `class` | Tham chiếu linh hoạt |
| Dữ liệu bất biến, cần equality theo giá trị | `record` | Value equality tự động |
| Field có thể vắng mặt trong DB/JSON | `T?` (nullable) | Biểu đạt rõ ý định |
| Slice bộ nhớ, zero-allocation | `Span<T>` | Không phân bổ Heap |
| Hằng số có tên | `enum` | Rõ ràng, type-safe |

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Sự khác biệt giữa `struct` và `class`?**  
A: `struct` là value type (lưu giá trị, copy khi gán), `class` là reference type (lưu địa chỉ, dùng chung khi gán). `struct` thường nhanh hơn cho dữ liệu nhỏ vì tránh Heap allocation.

**Q: Tại sao `string` là immutable (bất biến)?**  
A: Để thread safety (an toàn đa luồng) và string interning (tái sử dụng chuỗi). Mỗi "thay đổi" tạo string mới. Dùng `StringBuilder` khi cần nối nhiều chuỗi.

**Q: Boxing là gì và tại sao cần tránh?**  
A: Boxing là chuyển value type sang object trên Heap — tốn bộ nhớ và thời gian. Tránh bằng cách dùng generic collections (`List<int>`) thay vì `ArrayList`.

**Q: `int?` và `Nullable<int>` khác nhau không?**  
A: Không, `int?` là cú pháp rút gọn của `Nullable<int>`. Cả hai hoàn toàn tương đương.

---

## ✅ Checklist Tự Kiểm Tra

- [ ] Giải thích được copy semantics của value type và reference type
- [ ] Biết khi nào boxing xảy ra và cách tránh
- [ ] Dùng được `ref`, `out`, `in` đúng ngữ cảnh
- [ ] Biết tại sao `string` immutable và khi nào dùng `StringBuilder`
- [ ] Hiểu `Nullable<T>` và các toán tử `??`, `?.`, `??=`
- [ ] Biết `record` khác `class` như thế nào

---

**Tiếp theo:** [2-clr-and-memory.md](./2-clr-and-memory.md) — CLR, GC, Stack vs Heap
