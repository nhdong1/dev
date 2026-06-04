# 3 — Collections (Tập Hợp) trong C#

> Chọn đúng collection cho đúng tình huống là kỹ năng cơ bản. Chọn sai ảnh hưởng nghiêm trọng đến hiệu năng.

---

## 📌 Tổng Quan Hệ Thống Collections

```
System.Collections.Generic (Generic Collections — Tập hợp generic, không boxing)
├── List<T>              — Danh sách động (dynamic array)
├── Dictionary<TKey, TValue> — Bảng băm (hash map)
├── HashSet<T>           — Tập hợp không trùng lặp
├── Queue<T>             — Hàng đợi FIFO
├── Stack<T>             — Ngăn xếp LIFO
├── LinkedList<T>        — Danh sách liên kết đôi
└── SortedDictionary<TKey, TValue> — Dictionary sắp xếp theo key

System.Collections.Concurrent (Thread-safe Collections — Tập hợp an toàn đa luồng)
├── ConcurrentDictionary<TKey, TValue>
├── ConcurrentQueue<T>
├── ConcurrentBag<T>
└── BlockingCollection<T>

System.Collections.Immutable (Immutable Collections — Tập hợp bất biến)
├── ImmutableList<T>
├── ImmutableDictionary<TKey, TValue>
└── ImmutableArray<T>
```

---

## 1. Các Interface Quan Trọng

Hiểu interface trước khi học concrete types (kiểu cụ thể):

```
IEnumerable<T>         — Có thể duyệt (foreach), read-only, lazy
    ↑ kế thừa
ICollection<T>         — + Count, Add, Remove, Contains
    ↑ kế thừa
IList<T>               — + truy cập theo index, IndexOf, Insert
    ↑ kế thừa
List<T>                — Cài đặt đầy đủ
```

### `IEnumerable<T>` — Giao diện Cơ Bản Nhất

```csharp
// Chỉ cần IEnumerable<T> khi chỉ cần duyệt qua
IEnumerable<int> GetNumbers()
{
    yield return 1;
    yield return 2;
    yield return 3;
    // Lazy evaluation — chỉ tính khi được yêu cầu
}

// Tham số nhận IEnumerable thay vì List để linh hoạt hơn
void PrintAll(IEnumerable<string> items)
{
    foreach (var item in items)
        Console.WriteLine(item);
}

// Có thể truyền bất kỳ collection nào
PrintAll(new List<string> { "a", "b" });
PrintAll(new string[] { "x", "y" });
PrintAll(GetNumbers().Select(n => n.ToString()));
```

---

## 2. `List<T>` — Danh Sách Động

**Cấu trúc bên trong:** mảng (array) có khả năng tự mở rộng.

```csharp
var list = new List<int>();          // Khởi tạo rỗng, capacity mặc định = 4
var list2 = new List<int>(100);      // Khởi tạo với capacity = 100 (tránh resize)
var list3 = new List<int> { 1, 2, 3 }; // Khởi tạo với giá trị ban đầu
```

### Thao tác cơ bản

```csharp
var fruits = new List<string>();

// Thêm
fruits.Add("apple");
fruits.AddRange(new[] { "banana", "cherry" });
fruits.Insert(1, "avocado"); // Chèn tại vị trí index 1

// Xóa
fruits.Remove("banana");     // Xóa phần tử đầu tiên match
fruits.RemoveAt(0);          // Xóa theo index
fruits.RemoveAll(f => f.StartsWith("a")); // Xóa theo điều kiện

// Tìm kiếm
bool has = fruits.Contains("cherry");
int idx = fruits.IndexOf("cherry");
string found = fruits.Find(f => f.Length > 5); // FirstOrDefault tương đương

// Sắp xếp
fruits.Sort();
fruits.Sort((a, b) => b.CompareTo(a)); // Đảo ngược
```

### Độ phức tạp thời gian (Time Complexity)

| Thao tác | Độ Phức Tạp | Ghi chú |
|----------|-------------|---------|
| `Add` (thêm cuối) | O(1) amortized | Đôi khi O(n) khi resize |
| `Insert` (chèn giữa) | O(n) | Phải dịch các phần tử |
| `Remove` / `RemoveAt` | O(n) | Phải dịch các phần tử |
| `Contains` | O(n) | Linear search (tìm kiếm tuyến tính) |
| Truy cập index | O(1) | Direct array access |

---

## 3. `Dictionary<TKey, TValue>` — Bảng Băm

**Cấu trúc bên trong:** hash table (bảng băm) — ánh xạ key → value.

```csharp
var scores = new Dictionary<string, int>();
var scores2 = new Dictionary<string, int>
{
    ["Alice"] = 95,
    ["Bob"] = 87
};
```

### Thao tác cơ bản

```csharp
var dict = new Dictionary<string, int>();

// Thêm / Cập nhật
dict["key"] = 42;           // Gán — thêm mới hoặc ghi đè
dict.Add("key2", 99);       // Thêm — ném exception nếu key đã tồn tại
dict.TryAdd("key2", 100);   // An toàn — trả về false nếu đã tồn tại

// Đọc
int value = dict["key"];    // Ném KeyNotFoundException nếu không có

// ✅ Cách an toàn để đọc
if (dict.TryGetValue("key", out int val))
{
    Console.WriteLine(val);
}

// GetValueOrDefault (C# 8+)
int v = dict.GetValueOrDefault("missing", 0); // 0 nếu không có

// Kiểm tra
bool has = dict.ContainsKey("key");
bool hasVal = dict.ContainsValue(42);

// Xóa
dict.Remove("key");

// Duyệt
foreach (var (key, value2) in dict)
{
    Console.WriteLine($"{key} = {value2}");
}

// Chỉ keys hoặc values
IEnumerable<string> keys = dict.Keys;
IEnumerable<int> values = dict.Values;
```

### Độ phức tạp thời gian

| Thao tác | Độ Phức Tạp | Ghi chú |
|----------|-------------|---------|
| `Add` / `[]` set | O(1) avg | O(n) worst khi hash collision |
| `TryGetValue` / `[]` get | O(1) avg | |
| `ContainsKey` | O(1) avg | |
| `Remove` | O(1) avg | |

### GetOrAdd Pattern — Lấy hoặc Thêm Mới

```csharp
// Cần đếm số lần xuất hiện
var wordCount = new Dictionary<string, int>();

foreach (var word in words)
{
    // ❌ Cách dài dòng
    if (!wordCount.ContainsKey(word))
        wordCount[word] = 0;
    wordCount[word]++;

    // ✅ Cách gọn hơn
    wordCount.TryGetValue(word, out int count);
    wordCount[word] = count + 1;

    // ✅ CollectionsMarshal cho hiệu năng cao (tránh double lookup)
    ref int countRef = ref CollectionsMarshal.GetValueRefOrAddDefault(wordCount, word, out _);
    countRef++;
}
```

---

## 4. `HashSet<T>` — Tập Hợp Không Trùng Lặp

```csharp
var set = new HashSet<int> { 1, 2, 3, 4, 5 };
set.Add(3);          // Không thêm vì đã có — trả về false
bool has = set.Contains(3); // O(1)

// Phép toán tập hợp
var set2 = new HashSet<int> { 3, 4, 5, 6, 7 };

set.IntersectWith(set2);   // Giao — { 3, 4, 5 }
set.UnionWith(set2);       // Hợp — { 1, 2, 3, 4, 5, 6, 7 }
set.ExceptWith(set2);      // Hiệu — { 1, 2 }
set.IsSubsetOf(set2);      // Kiểm tra tập con

// Loại bỏ trùng lặp nhanh từ list
var unique = new HashSet<string>(listWithDuplicates);
```

---

## 5. `Queue<T>` và `Stack<T>`

### Queue — Hàng Đợi FIFO (First In First Out — Vào Trước Ra Trước)

```csharp
var queue = new Queue<string>();
queue.Enqueue("first");   // Thêm vào cuối
queue.Enqueue("second");
queue.Enqueue("third");

string next = queue.Peek();     // Xem phần tử đầu không xóa: "first"
string item = queue.Dequeue();  // Lấy và xóa phần tử đầu: "first"
int count = queue.Count;        // 2
```

### Stack — Ngăn Xếp LIFO (Last In First Out — Vào Sau Ra Trước)

```csharp
var stack = new Stack<int>();
stack.Push(1);
stack.Push(2);
stack.Push(3);

int top = stack.Peek();  // Xem đỉnh không xóa: 3
int val = stack.Pop();   // Lấy và xóa đỉnh: 3
```

---

## 6. `Span<T>` và `Memory<T>` — Zero-Allocation Slices

**Span\<T\>** — Lát Cắt Bộ Nhớ Không Phân Bổ: tham chiếu đến một vùng liên tiếp trong bộ nhớ mà không copy.

```csharp
// Làm việc với mảng không cần tạo mảng con mới
int[] array = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

Span<int> span = array;              // Toàn bộ mảng
Span<int> slice = array.AsSpan(2, 5); // Phần tử index 2 đến 6 — không copy!

// Duyệt và sửa
foreach (ref int item in slice)
{
    item *= 2; // Sửa trực tiếp mảng gốc
}

// So sánh hiệu năng: parsing không cần allocation
ReadOnlySpan<char> text = "2026-06-02".AsSpan();
ReadOnlySpan<char> year = text.Slice(0, 4);   // "2026" — không tạo string mới
```

### Giới hạn của `Span<T>`

```csharp
// Span là ref struct — chỉ tồn tại trên Stack
// ❌ Không thể lưu làm field của class
public class MyClass
{
    // private Span<int> _span; // Lỗi compile
    private Memory<int> _memory; // ✅ Memory<T> có thể là field
}

// ❌ Không thể dùng trong async method
async Task BadExample()
{
    Span<int> span = new int[10]; // Lỗi compile trong async
}

// ✅ Dùng Memory<T> trong async
async Task GoodExample()
{
    Memory<int> memory = new int[10];
    await SomeAsync();
    Span<int> span = memory.Span; // OK — Span trong synchronous code
}
```

---

## 7. Concurrent Collections — Tập Hợp An Toàn Đa Luồng

```csharp
// ConcurrentDictionary — thread-safe dictionary
var concurrent = new ConcurrentDictionary<string, int>();

concurrent.TryAdd("key", 1);
concurrent.AddOrUpdate("key",
    addValue: 1,
    updateValueFactory: (key, existing) => existing + 1);

int value = concurrent.GetOrAdd("key", key => ComputeValue(key));

// ConcurrentQueue — thread-safe queue
var cq = new ConcurrentQueue<string>();
cq.Enqueue("item");
if (cq.TryDequeue(out string result)) { }
```

---

## 8. Immutable Collections — Tập Hợp Bất Biến

```csharp
using System.Collections.Immutable;

// Tạo immutable list
ImmutableList<int> list = ImmutableList.Create(1, 2, 3);

// "Thay đổi" tạo ra instance mới, list gốc không đổi
ImmutableList<int> list2 = list.Add(4);      // list vẫn là {1,2,3}
ImmutableList<int> list3 = list.Remove(2);   // {1,3}

// Builder pattern cho hiệu năng khi cần nhiều thay đổi
var builder = ImmutableList.CreateBuilder<int>();
for (int i = 0; i < 1000; i++) builder.Add(i);
ImmutableList<int> result = builder.ToImmutable();
```

---

## 9. Cheat Sheet: Khi Nào Dùng Gì

| Tình Huống | Collection | Lý Do |
|-----------|-----------|-------|
| Danh sách thứ tự, truy cập index | `List<T>` | Phổ biến nhất, đủ dùng |
| Tra cứu nhanh theo key | `Dictionary<TKey, TValue>` | O(1) lookup |
| Kiểm tra membership, loại trùng | `HashSet<T>` | O(1) Contains |
| FIFO queue (message queue) | `Queue<T>` | Đúng ngữ nghĩa |
| LIFO stack (undo history) | `Stack<T>` | Đúng ngữ nghĩa |
| Bộ nhớ liên tiếp, zero-alloc | `Span<T>` / `ArrayPool<T>` | High performance |
| Đa luồng shared state | `Concurrent*` | Thread-safe |
| Immutable shared data | `Immutable*` | Không cần lock |
| Chỉ cần duyệt (read-only) | `IEnumerable<T>` | Lazy, linh hoạt |

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: `IEnumerable<T>` vs `IList<T>` — khi nào dùng cái nào?**  
A: Dùng `IEnumerable<T>` khi chỉ cần duyệt — linh hoạt nhất, lazy evaluation. Dùng `IList<T>` khi cần truy cập index hoặc biết số lượng phần tử. Tham số hàm nên nhận interface rộng nhất đáp ứng nhu cầu.

**Q: Tại sao không nên dùng `ArrayList` trong code mới?**  
A: `ArrayList` không generic → boxing/unboxing với value types, không type-safe tại compile time. Luôn dùng `List<T>` thay thế.

**Q: `Dictionary<K,V>` có thread-safe không?**  
A: Không. Đọc đồng thời an toàn, nhưng đọc+ghi đồng thời gây race condition. Dùng `ConcurrentDictionary<K,V>` cho multi-threaded scenarios.

**Q: Khi nào dùng `Span<T>`?**  
A: Khi cần làm việc với slice của mảng/string mà không muốn allocate thêm bộ nhớ. Phổ biến trong parsing, serialization, và hot paths (đường dẫn nóng — code chạy rất thường xuyên).

---

## ✅ Checklist Tự Kiểm Tra

- [ ] Giải thích được khi nào dùng List vs Dictionary vs HashSet
- [ ] Biết time complexity của các thao tác phổ biến
- [ ] Dùng `TryGetValue` thay vì double lookup trong Dictionary
- [ ] Hiểu `Span<T>` là gì và tại sao không thể làm field của class
- [ ] Biết dùng Concurrent collections trong multi-threaded code
- [ ] Viết được code loại bỏ trùng lặp với HashSet

---

**Tiếp theo:** [4-linq.md](./4-linq.md) — LINQ: Truy Vấn Tích Hợp Ngôn Ngữ
