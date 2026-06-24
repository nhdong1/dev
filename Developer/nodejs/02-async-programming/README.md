# Lập Trình Bất Đồng Bộ — Tổng Quan

> Chủ đề cốt lõi nhất của Node.js Backend: hiểu Event Loop (Vòng Lặp Sự Kiện), Callback (Hàm Gọi Lại), Promise, async/await, xử lý lỗi bất đồng bộ, và các pattern concurrency (đồng thời) thực tế.

## Mục Lục

1. [Tại Sao Async Quan Trọng Trong Node.js](#tại-sao-async-quan-trọng-trong-nodejs)
2. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
3. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
4. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
5. [Bài Tập Thực Hành](#bài-tập-thực-hành)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Async Quan Trọng Trong Node.js

Node.js được thiết kế cho **I/O-bound workloads (tải ràng buộc I/O)** — đọc database, gọi API, đọc file. Nếu dùng **synchronous blocking (chặn đồng bộ)**, main thread sẽ đứng yên chờ từng operation, không xử lý được request khác.

| Kỹ Năng | Lý Do Quan Trọng |
| ------- | ---------------- |
| **Event Loop** | Giải thích được tại sao Node.js "single-threaded" vẫn handle nhiều request |
| **Callbacks** | Hiểu nguồn gốc async pattern, đọc legacy code và Node.js APIs cũ |
| **Promises** | Chuẩn hiện đại, composable (kết hợp được), tránh callback hell |
| **async/await** | Cú pháp đọc như sync nhưng vẫn non-blocking — chuẩn trong codebase hiện đại |
| **Error Handling** | Lỗi async dễ bị "nuốt" — production crash nếu không xử lý đúng |
| **Concurrency Patterns** | Throttle, debounce, queue — pattern thực tế trong API và background jobs |

---

## Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────┐
│                    ASYNC EXECUTION MODEL                          │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  Callbacks   │ →  │   Promises   │ →  │   async/await    │  │
│  │  (legacy)    │    │  (ES2015)    │    │  (syntactic      │  │
│  │              │    │              │    │   sugar)         │  │
│  └──────┬───────┘    └──────┬───────┘    └────────┬─────────┘  │
│         └───────────────────┼─────────────────────┘              │
│                             ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Event Loop (libuv)                      │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐ │   │
│  │  │ Call Stack  │  │ Microtask    │  │ Macrotask       │ │   │
│  │  │ (Ngăn xếp)  │  │ Queue        │  │ Queue           │ │   │
│  │  │             │  │ (Promise,    │  │ (setTimeout,    │ │   │
│  │  │             │  │  nextTick)   │  │  I/O callbacks) │ │   │
│  │  └─────────────┘  └──────────────┘  └─────────────────┘ │   │
│  └─────────────────────────────────────────────────────────┘   │
│                             ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │         Thread Pool / OS (file I/O, DNS, crypto)         │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 8–10 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-event-loop.md](./1-event-loop.md) | Event Loop phases, call stack, microtasks vs macrotasks | 2 giờ |
| 2 | [2-callbacks.md](./2-callbacks.md) | Callback pattern, callback hell, error-first callback | 1 giờ |
| 3 | [3-promises.md](./3-promises.md) | Promise API, chaining, combinators | 1.5 giờ |
| 4 | [4-async-await.md](./4-async-await.md) | async/await, parallel vs sequential execution | 1.5 giờ |
| 5 | [5-error-handling.md](./5-error-handling.md) | try/catch, unhandled rejection, process handlers | 1.5 giờ |
| 6 | [6-concurrency-patterns.md](./6-concurrency-patterns.md) | Throttle, debounce, queue, batch processing | 1.5 giờ |

**Thứ tự học bắt buộc:** 1 → 2 → 3 → 4 → 5 → 6. Event Loop phải học trước — mọi pattern async đều dựa trên nó.

---

## Các Tài Liệu Chi Tiết

| File | Nội Dung Chính |
| ---- | -------------- |
| [1-event-loop.md](./1-event-loop.md) | 6 phases của Event Loop, call stack, microtask queue, output order puzzles |
| [2-callbacks.md](./2-callbacks.md) | Error-first callback, callback hell, promisify pattern |
| [3-promises.md](./3-promises.md) | Promise states, chaining, Promise.all/race/allSettled/any |
| [4-async-await.md](./4-async-await.md) | Top-level await, parallel execution, common pitfalls |
| [5-error-handling.md](./5-error-handling.md) | Async error propagation, global handlers, production strategy |
| [6-concurrency-patterns.md](./6-concurrency-patterns.md) | Rate limiting, job queue, p-limit, batch processing |

---

## Bài Tập Thực Hành

### Lab 1: Event Loop Output Order (30 phút)

```javascript
// Dự đoán thứ tự in ra, chạy và giải thích
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve().then(() => console.log('3'));

process.nextTick(() => console.log('4'));

console.log('5');
```

### Lab 2: Callback → Promise → async/await (45 phút)

```javascript
// Refactor hàm đọc file từ callback sang Promise rồi async/await
// fs.readFile → util.promisify → async function
```

### Lab 3: Parallel vs Sequential (30 phút)

```javascript
// Fetch 3 URLs — so sánh thời gian:
// Sequential: await từng cái
// Parallel: Promise.all
```

### Lab 4: Rate-Limited API Client (45 phút)

```javascript
// Implement queue xử lý 100 requests với concurrency = 5
// Dùng pattern từ 6-concurrency-patterns.md
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Event Loop hoạt động như thế nào?

**Gợi ý trả lời:** Event Loop là vòng lặp trong libuv liên tục kiểm tra call stack và các queue. Khi call stack rỗng, nó lấy callback từ các phase (timers, poll, check...) và đưa lên stack thực thi. Microtasks (Promise, process.nextTick) chạy giữa các macrotask. JavaScript chạy single-threaded trên main thread; I/O delegate cho OS/thread pool.

### Câu 2: `process.nextTick()` vs `setImmediate()`?

**Gợi ý trả lời:** `process.nextTick()` chạy ngay sau operation hiện tại, trước Event Loop phase tiếp theo — priority cao nhất. `setImmediate()` chạy trong check phase, sau poll phase. Lạm dụng nextTick có thể **starve (đói)** Event Loop.

### Câu 3: `Promise.all` vs `Promise.allSettled`?

**Gợi ý trả lời:** `Promise.all` fail-fast — một rejection làm cả batch fail. `Promise.allSettled` chờ tất cả hoàn thành, trả về status từng promise. Dùng `all` khi cần tất cả thành công; `allSettled` khi cần kết quả từng item dù có lỗi.

### Câu 4: async/await có block main thread không?

**Gợi ý trả lời:** Không. `await` chỉ **tạm dừng function async** (suspend coroutine), không block Event Loop. Code sau `await` được schedule như microtask/macrotask. Chỉ **CPU-intensive sync code** mới block main thread.

### Câu 5: Xử lý unhandled promise rejection trong production?

**Gợi ý trả lời:** Luôn `.catch()` hoặc try/catch với await. Đăng ký `process.on('unhandledRejection')` để log và graceful shutdown. Không để process chạy trong trạng thái undefined. Node.js 15+ mặc định terminate on unhandled rejection nếu không có handler.

---

**Xem tiếp:** [1-event-loop.md](./1-event-loop.md) — bắt đầu với Event Loop deep dive.
