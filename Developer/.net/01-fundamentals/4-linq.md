# 4 — LINQ (Language Integrated Query — Truy Vấn Tích Hợp Ngôn Ngữ)

> LINQ là một trong những tính năng mạnh mẽ nhất của C#. Hiểu đúng cách nó hoạt động — đặc biệt **deferred execution** — giúp tránh nhiều bug tinh vi.

---

## 📌 LINQ Là Gì?

**LINQ** cho phép truy vấn dữ liệu trực tiếp trong C# với cú pháp nhất quán, bất kể nguồn dữ liệu:

- `LINQ to Objects` — Truy vấn collections trong bộ nhớ
- `LINQ to Entities` / `EF Core` — Truy vấn cơ sở dữ liệu (chuyển thành SQL)
- `LINQ to XML` — Truy vấn tài liệu XML
- `LINQ to JSON` — Truy vấn JSON (với Newtonsoft.Json hoặc System.Text.Json)

---

## 1. Hai Cú Pháp LINQ

### Query Syntax — Cú pháp truy vấn (giống SQL)

```csharp
var result = from student in students
             where student.Grade >= 8.0
             orderby student.Name
             select new { student.Name, student.Grade };
```

### Method Syntax — Cú pháp phương thức (fluent, phổ biến hơn)

```csharp
var result = students
    .Where(s => s.Grade >= 8.0)
    .OrderBy(s => s.Name)
    .Select(s => new { s.Name, s.Grade });
```

Cả hai tương đương. **Method syntax** được dùng nhiều hơn trong thực tế vì dễ chain (nối chuỗi) và IDE autocomplete tốt hơn.

---

## 2. Deferred Execution vs Immediate Execution

Đây là khái niệm **quan trọng nhất** và dễ gây bug nhất trong LINQ.

### Deferred Execution — Thực Thi Trì Hoãn

Query **không chạy ngay** khi được định nghĩa. Chỉ chạy khi kết quả được **enumerate** (duyệt qua) lần đầu tiên.

```csharp
var numbers = new List<int> { 1, 2, 3, 4, 5 };

// ❗ Query được ĐỊNH NGHĨA nhưng CHƯA chạy
IEnumerable<int> query = numbers.Where(n => n > 2);

// ✅ Query chạy LÚC NÀY (khi foreach bắt đầu duyệt)
foreach (int n in query)
{
    Console.WriteLine(n); // 3, 4, 5
}
```

### Bug Kinh Điển: Thay Đổi Nguồn Dữ Liệu Sau Khi Định Nghĩa Query

```csharp
var numbers = new List<int> { 1, 2, 3, 4, 5 };
IEnumerable<int> query = numbers.Where(n => n > 2);

numbers.Add(10);  // Thêm vào nguồn SAU khi định nghĩa query

// Nhưng TRƯỚC khi thực thi query!
foreach (int n in query)
{
    Console.WriteLine(n); // 3, 4, 5, 10  ← 10 xuất hiện!
}
```

### Multiple Enumeration — Duyệt Nhiều Lần (Bug Hiệu Năng)

```csharp
IEnumerable<int> query = numbers.Where(n => n > 2).Select(n => n * 2);

// ❌ Query chạy 2 lần!
int count = query.Count();       // Chạy lần 1
int sum = query.Sum();           // Chạy lần 2

// ✅ Materialize (hiện thực hóa) ngay để chỉ chạy 1 lần
List<int> materialized = query.ToList();
int count2 = materialized.Count;
int sum2 = materialized.Sum();
```

### Immediate Execution — Thực Thi Ngay

Các phương thức sau **ngay lập tức** thực thi query và trả về kết quả cụ thể:

| Phương thức | Trả về | Ghi chú |
|-------------|--------|---------|
| `ToList()` | `List<T>` | Phổ biến nhất |
| `ToArray()` | `T[]` | Dùng khi cần array |
| `ToDictionary()` | `Dictionary<K,V>` | |
| `ToHashSet()` | `HashSet<T>` | |
| `Count()` | `int` | |
| `Sum()`, `Min()`, `Max()`, `Average()` | scalar | |
| `First()`, `FirstOrDefault()` | `T` | |
| `Single()`, `SingleOrDefault()` | `T` | |
| `Any()`, `All()` | `bool` | |

---

## 3. Các Operator LINQ Phổ Biến

### Filtering — Lọc

```csharp
var adults = people.Where(p => p.Age >= 18);
var first = people.First(p => p.Name == "Alice");  // Exception nếu không có
var firstOrNull = people.FirstOrDefault(p => p.Name == "Alice"); // null nếu không có
var single = people.Single(p => p.Id == 1);  // Exception nếu có 0 hoặc >1 kết quả
```

### Projection — Chiếu (Chuyển đổi dạng dữ liệu)

```csharp
// Select — chiếu thành kiểu khác
var names = people.Select(p => p.Name);
var dtos = people.Select(p => new PersonDto { Id = p.Id, FullName = p.Name });

// SelectMany — "flatten" (làm phẳng) danh sách lồng nhau
var allTags = posts.SelectMany(p => p.Tags);
// posts: [ {Tags: ["c#", ".net"]}, {Tags: ["linq"]} ]
// allTags: ["c#", ".net", "linq"]
```

### Ordering — Sắp Xếp

```csharp
var sorted = people.OrderBy(p => p.LastName).ThenBy(p => p.FirstName);
var descending = people.OrderByDescending(p => p.Age);
var reversed = list.Reverse(); // Đảo ngược thứ tự hiện tại
```

### Grouping — Nhóm

```csharp
var byDept = employees.GroupBy(e => e.Department);

foreach (var group in byDept)
{
    Console.WriteLine($"Phòng ban: {group.Key}");
    foreach (var emp in group)
    {
        Console.WriteLine($"  {emp.Name}");
    }
}

// GroupBy với projection
var summary = employees.GroupBy(e => e.Department)
    .Select(g => new
    {
        Department = g.Key,
        Count = g.Count(),
        AverageSalary = g.Average(e => e.Salary)
    });
```

### Joining — Kết Hợp

```csharp
// Inner join — kết hợp trong (chỉ lấy matching records)
var joined = orders.Join(
    customers,
    order => order.CustomerId,
    customer => customer.Id,
    (order, customer) => new { order.Id, customer.Name, order.Amount }
);

// Group join — tương tự LEFT JOIN trong SQL
var grouped = customers.GroupJoin(
    orders,
    customer => customer.Id,
    order => order.CustomerId,
    (customer, customerOrders) => new
    {
        customer.Name,
        Orders = customerOrders.ToList()
    }
);
```

### Aggregation — Tổng Hợp

```csharp
int total = numbers.Sum();
double avg = numbers.Average();
int max = numbers.Max();
int min = numbers.Min();
int count = numbers.Count(n => n > 0);  // Đếm theo điều kiện

// Aggregate — tổng hợp tùy chỉnh (như fold/reduce)
int product = numbers.Aggregate(1, (acc, n) => acc * n);
string sentence = words.Aggregate((a, b) => $"{a} {b}");
```

### Set Operations — Phép Toán Tập Hợp

```csharp
var distinct = list.Distinct();                      // Loại bỏ trùng lặp
var union = list1.Union(list2);                      // Hợp (không trùng)
var intersection = list1.Intersect(list2);           // Giao
var difference = list1.Except(list2);                // Hiệu
```

### Partitioning — Phân Chia

```csharp
var firstFive = list.Take(5);                        // Lấy 5 phần tử đầu
var skipFive = list.Skip(5);                         // Bỏ qua 5 đầu, lấy phần còn lại
var page = list.Skip(pageIndex * pageSize).Take(pageSize); // Phân trang

// TakeWhile / SkipWhile — lấy/bỏ khi điều kiện còn đúng
var whileSmall = list.TakeWhile(n => n < 10);
```

---

## 4. Custom LINQ với `yield return`

**`yield return`** cho phép tạo lazy sequences (chuỗi lười) — không tính toán tất cả phần tử cùng lúc.

```csharp
// Generator method — phương thức tạo sinh
IEnumerable<int> GetEvenNumbers(int max)
{
    for (int i = 0; i <= max; i += 2)
    {
        yield return i; // Trả về từng phần tử, dừng lại cho đến khi được yêu cầu tiếp
    }
}

// Chỉ tính các số chẵn khi cần
foreach (int n in GetEvenNumbers(100))
{
    if (n > 20) break; // Dừng sớm — không lãng phí tính toán
}
```

### Ứng dụng: Xử lý file lớn không load vào RAM

```csharp
IEnumerable<string> ReadLines(string filePath)
{
    using var reader = new StreamReader(filePath);
    string? line;
    while ((line = reader.ReadLine()) != null)
    {
        yield return line; // Đọc từng dòng khi được yêu cầu
    }
} // StreamReader được Dispose khi enumeration kết thúc

// Chỉ giữ 1 dòng trong RAM tại một thời điểm
var longLines = ReadLines("huge.log")
    .Where(line => line.Length > 100)
    .Take(50)
    .ToList();
```

---

## 5. LINQ Performance Tips — Mẹo Hiệu Năng

### Dùng `Any()` thay vì `Count() > 0`

```csharp
// ❌ Đếm tất cả phần tử chỉ để kiểm tra có phần tử không
if (list.Count() > 0) { }

// ✅ Dừng ngay khi tìm thấy phần tử đầu tiên
if (list.Any()) { }
if (list.Any(x => x.IsActive)) { }
```

### `FirstOrDefault()` vs `SingleOrDefault()`

```csharp
// SingleOrDefault — xác nhận chỉ có 1 kết quả, duyệt hết để kiểm tra
// FirstOrDefault — chỉ cần tìm thấy 1, dừng ngay

// ✅ Khi biết chỉ có 1 (ví dụ: lookup by unique ID)
var user = users.SingleOrDefault(u => u.Id == id);

// ✅ Khi chỉ cần phần tử đầu tiên thoả điều kiện
var first = users.FirstOrDefault(u => u.IsAdmin);
```

### Tránh Closure Capture Không Cần Thiết

```csharp
// ❌ Mỗi lần query chạy, đóng gói biến ngoài
int threshold = GetThreshold(); // Gọi 1 lần

var query = list.Where(x => x > threshold); // OK nếu threshold không đổi

// ❌ Bug: biến thay đổi trong loop
for (int i = 0; i < 5; i++)
{
    // i được capture by reference!
    var q = list.Where(x => x > i); // i sẽ là 5 khi query chạy
    queries.Add(q);
}

// ✅ Capture giá trị tại thời điểm đó
for (int i = 0; i < 5; i++)
{
    int captured = i; // Tạo biến mới mỗi iteration
    var q = list.Where(x => x > captured);
    queries.Add(q);
}
```

### PLINQ — Parallel LINQ (LINQ Song Song)

```csharp
// AsParallel() — chia công việc cho nhiều thread
var result = largeList
    .AsParallel()
    .WithDegreeOfParallelism(4) // Tối đa 4 thread
    .Where(x => ExpensiveFilter(x))
    .Select(x => Transform(x))
    .ToList();

// Cẩn thận: PLINQ có overhead — chỉ có lợi với tập dữ liệu lớn và tác vụ CPU-intensive
```

---

## 6. Common Pitfalls — Những Lỗi Thường Gặp

### Truy vấn trên null collection

```csharp
List<int>? maybeNull = null;

// ❌ NullReferenceException
var result = maybeNull.Where(x => x > 0);

// ✅ Null-safe với ?. (null-conditional) và ?? (null-coalescing)
var result = (maybeNull ?? Enumerable.Empty<int>()).Where(x => x > 0);
// Hoặc
var result = maybeNull?.Where(x => x > 0) ?? Enumerable.Empty<int>();
```

### N+1 trong LINQ to Entities (EF Core)

```csharp
// ❌ N+1: 1 query lấy orders + N queries lấy customer từng cái
var orders = dbContext.Orders.ToList(); // 1 query
foreach (var order in orders)
{
    Console.WriteLine(order.Customer.Name); // N queries lazy load!
}

// ✅ Eager loading (tải háo hức) với Include
var orders = dbContext.Orders
    .Include(o => o.Customer)  // 1 JOIN query
    .ToList();
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Deferred execution là gì? Cho ví dụ gây bug.**  
A: Query không chạy khi định nghĩa mà chạy khi enumerate. Bug: định nghĩa query, sửa nguồn dữ liệu, rồi mới chạy → kết quả không như mong đợi. Fix: gọi `.ToList()` để materialize ngay.

**Q: Sự khác biệt giữa `Where` và `FirstOrDefault`?**  
A: `Where` là deferred — trả về `IEnumerable`, lazy. `FirstOrDefault` là immediate — chạy ngay, trả về phần tử đầu tiên hoặc null.

**Q: Khi nào nên dùng `Select` vs `SelectMany`?**  
A: `Select` 1-to-1 transform. `SelectMany` flatten collection lồng nhau — mỗi phần tử input tạo ra nhiều phần tử output, gộp lại thành một chuỗi phẳng.

**Q: `yield return` có tác dụng gì?**  
A: Tạo lazy generator — trả về từng phần tử khi được yêu cầu mà không load hết vào bộ nhớ. Hữu ích khi xử lý stream dữ liệu lớn.

---

## ✅ Checklist Tự Kiểm Tra

- [ ] Giải thích deferred vs immediate execution với ví dụ cụ thể
- [ ] Biết khi nào cần gọi `.ToList()` để avoid multiple enumeration
- [ ] Dùng được `GroupBy`, `Join`, `SelectMany` đúng ngữ cảnh
- [ ] Giải thích tại sao `Any()` tốt hơn `Count() > 0`
- [ ] Viết được custom iterator với `yield return`
- [ ] Nhận biết N+1 problem trong LINQ to Entities

---

**Tiếp theo:** [5-delegates-events.md](./5-delegates-events.md) — Delegate, Func, Action, Event
