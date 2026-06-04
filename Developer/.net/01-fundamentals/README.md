# 01 — Nền Tảng C# (Fundamentals)

> Đây là điểm khởi đầu bắt buộc. Nếu nền tảng không vững, mọi thứ phía trên sẽ lung lay.

---

## 🎯 Mục Tiêu Của Section Này

Sau khi hoàn thành `01-fundamentals/`, bạn phải:

- Giải thích được sự khác biệt giữa **value type** (kiểu giá trị) và **reference type** (kiểu tham chiếu) và tại sao điều đó quan trọng
- Mô tả được cách **CLR** (Common Language Runtime — Môi Trường Chạy Ngôn Ngữ Chung) quản lý bộ nhớ, **GC** (Garbage Collector — Bộ Thu Gom Rác) hoạt động thế nào
- Chọn đúng **collection** (tập hợp) cho từng tình huống cụ thể
- Viết **LINQ** (Language Integrated Query — Truy Vấn Tích Hợp Ngôn Ngữ) fluent một cách tự tin
- Dùng **delegate** (ủy quyền), **Func**, **Action**, và **event** (sự kiện) đúng ngữ cảnh
- Xử lý ngoại lệ (**exception handling**) đúng chuẩn, không nuốt lỗi

---

## 📁 Các File Trong Section Này

| File | Chủ Đề | Độ Khó | Ưu Tiên |
|------|--------|--------|---------|
| [1-type-system.md](./1-type-system.md) | Value types vs Reference types, Nullable | ⭐ | Bắt buộc |
| [2-clr-and-memory.md](./2-clr-and-memory.md) | CLR, GC, Stack vs Heap, các thế hệ GC | ⭐⭐ | Bắt buộc |
| [3-collections.md](./3-collections.md) | List, Dictionary, IEnumerable, Span | ⭐⭐ | Bắt buộc |
| [4-linq.md](./4-linq.md) | LINQ: deferred vs immediate execution | ⭐⭐ | Bắt buộc |
| [5-delegates-events.md](./5-delegates-events.md) | Delegate, Func, Action, event, EventHandler | ⭐⭐ | Bắt buộc |
| [6-exception-handling.md](./6-exception-handling.md) | try/catch/finally, custom exceptions, AggregateException | ⭐ | Bắt buộc |

---

## 🗺️ Thứ Tự Học Đề Xuất

```
1-type-system  →  2-clr-and-memory  →  3-collections
                                              ↓
               6-exception-handling  ←  4-linq  →  5-delegates-events
```

Học theo thứ tự trên vì mỗi bài xây dựng trên kiến thức của bài trước:
- **Type system** là nền tảng để hiểu **memory model**
- **Memory model** giải thích tại sao **collections** có behavior khác nhau
- **LINQ** dùng **IEnumerable** và **delegate** bên dưới
- **Delegates** được dùng xuyên suốt async/await (sẽ học ở section 03)

---

## ⏱️ Ước Tính Thời Gian

| Hoạt Động | Thời Gian |
|-----------|-----------|
| Đọc lý thuyết (6 file) | 2–3 giờ |
| Viết code thực hành | 2–3 giờ |
| Tự kiểm tra / làm bài tập | 1 giờ |
| **Tổng cộng** | **4–6 giờ** |

---

## 🧱 Kiến Thức Cần Có Trước (Prerequisites)

- Biết lập trình cơ bản (biến, hàm, vòng lặp)
- Đã cài **.NET SDK** — [tải tại dotnet.microsoft.com](https://dotnet.microsoft.com/download)
- Có IDE: **Visual Studio**, **VS Code + C# Dev Kit**, hoặc **Rider**

---

## 🔗 Liên Kết Đến Các Section Tiếp Theo

Sau khi hoàn thành `01-fundamentals/`, học tiếp:

- **[02-oop-patterns/](../02-oop-patterns/)** — SOLID, Design Patterns (dùng type system + delegates)
- **[03-async-concurrency/](../03-async-concurrency/)** — async/await xây dựng trên delegates và tasks
- **[04-aspnet-core/](../04-aspnet-core/)** — DI, Middleware (dùng generics và collections)

---

## 💡 Câu Hỏi Tự Kiểm Tra Sau Khi Học Xong

1. `struct` và `class` khác nhau ở điểm nào khi truyền vào hàm?
2. Tại sao `string` là **immutable** (bất biến) trong C#?
3. GC chạy khi nào? Bạn có thể gọi thủ công không, và có nên không?
4. `IEnumerable<T>` và `IList<T>` — khi nào dùng cái nào?
5. **Deferred execution** (thực thi trì hoãn) trong LINQ là gì? Cho ví dụ gây bug.
6. `Func<int, int, string>` có nghĩa gì?
7. Sự khác nhau giữa `catch (Exception e)` và `catch (Exception e) when (e is ...)`?

---

**Trạng Thái:** ✅ Hoàn thành  
**Cập Nhật:** 2026-06-02
