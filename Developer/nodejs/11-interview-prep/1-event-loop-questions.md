# Câu Hỏi Event Loop & Async — Deep Dive

> Bộ câu hỏi phỏng vấn chuyên sâu về Event Loop (Vòng Lặp Sự Kiện), Promises, async/await, và concurrency patterns — chủ đề được hỏi nhiều nhất trong phỏng vấn Node.js.

## Mục Lục

1. [Câu Hỏi Cơ Bản](#câu-hỏi-cơ-bản)
2. [Câu Hỏi Trung Bình](#câu-hỏi-trung-bình)
3. [Câu Hỏi Nâng Cao](#câu-hỏi-nâng-cao)
4. [Bài Tập Thứ Tự Output](#bài-tập-thứ-tự-output)
5. [Live Coding Challenges](#live-coding-challenges)

---

## Câu Hỏi Cơ Bản

### Q1: Event Loop hoạt động như thế nào?

**Đáp án mẫu:**

Node.js chạy JavaScript trên **single main thread** (luồng chính đơn) với **Event Loop** điều phối async operations:

1. Code sync chạy trên **Call Stack** (Ngăn Xếp Gọi Hàm)
2. Async I/O được đăng ký với **libuv** (thư viện C xử lý I/O)
3. Khi I/O hoàn thành, callback vào **Callback Queue** (Hàng Đợi Callback)
4. Event Loop lấy callback từ queue đưa lên Call Stack khi stack rỗng
5. **Microtasks** (Promise callbacks, `process.nextTick`) được ưu tiên trước macrotasks

**Điểm cộng:** Đề cập thread pool (mặc định 4 threads) cho `fs`, `crypto`, `dns.lookup`.

---

### Q2: Sự khác biệt giữa `process.nextTick()` và `setImmediate()`?

| | `process.nextTick()` | `setImmediate()` |
| - | -------------------- | ---------------- |
| **Queue** | nextTick queue (microtask) | check phase của Event Loop |
| **Ưu tiên** | Cao nhất — chạy trước mọi thứ | Sau I/O callbacks |
| **Use case** | Propagate error trước khi tiếp tục | Defer execution sau I/O |
| **Rủi ro** | nextTick starvation nếu lặp vô hạn | An toàn hơn cho defer |

```javascript
setImmediate(() => console.log('immediate'));
process.nextTick(() => console.log('nextTick'));
console.log('sync');
// Output: sync → nextTick → immediate
```

---

### Q3: `setTimeout(fn, 0)` vs `setImmediate(fn)` — khác gì?

Trong **main module** (top-level), thứ tự không đảm bảo — phụ thuộc performance của process.

Trong **I/O callback**, `setImmediate` luôn chạy trước `setTimeout(0)` vì nằm trong check phase ngay sau poll phase.

**Điểm phỏng vấn:** Không dùng `setTimeout(0)` để defer — dùng `setImmediate` hoặc `queueMicrotask`.

---

### Q4: Microtask vs Macrotask — ví dụ?

| Loại | Ví Dụ |
| ---- | ----- |
| **Microtask** | `Promise.then/catch/finally`, `queueMicrotask()`, `process.nextTick()` |
| **Macrotask** | `setTimeout`, `setInterval`, `setImmediate`, I/O callbacks |

**Thứ tự:** Sync code → drain all microtasks → 1 macrotask → drain all microtasks → ...

---

### Q5: Node.js có thực sự single-threaded không?

**Đáp án:** JavaScript code chạy trên **single main thread**, nhưng Node.js runtime **không hoàn toàn single-threaded**:

- **libuv thread pool** (4 threads mặc định): `fs`, `crypto`, `dns.lookup`, compression
- **Worker Threads**: CPU-intensive tasks trên thread riêng
- **Child Processes**: process riêng biệt

---

## Câu Hỏi Trung Bình

### Q6: `Promise.all` vs `Promise.allSettled` vs `Promise.race`?

| API | Hành Vi | Khi Nào Dùng |
| --- | ------- | ------------ |
| `Promise.all` | Fail fast — 1 reject → toàn bộ reject | Cần tất cả kết quả, 1 fail = abort |
| `Promise.allSettled` | Chờ tất cả, trả về status từng promise | Không quan tâm 1 vài fail (batch operations) |
| `Promise.race` | Kết quả của promise đầu tiên settle | Timeout pattern, fastest response wins |

```javascript
// Timeout pattern với Promise.race
const timeout = (ms) => new Promise((_, reject) =>
  setTimeout(() => reject(new Error('Timeout')), ms)
);

const result = await Promise.race([fetchData(), timeout(5000)]);
```

---

### Q7: Làm sao chạy async tasks song song nhưng giới hạn concurrency?

**Đáp án:** Dùng **semaphore pattern** hoặc thư viện `p-limit`:

```javascript
async function mapWithConcurrency(items, fn, limit) {
  const results = [];
  const executing = new Set();

  for (const item of items) {
    const p = fn(item).then(r => {
      executing.delete(p);
      return r;
    });
    results.push(p);
    executing.add(p);

    if (executing.size >= limit) {
      await Promise.race(executing);
    }
  }
  return Promise.all(results);
}

// Dùng: mapWithConcurrency(urls, fetchUrl, 5) — tối đa 5 concurrent
```

**Điểm cộng:** Giải thích tại sao không dùng `Promise.all` cho 10,000 requests (memory + connection exhaustion).

---

### Q8: Unhandled Promise Rejection — xử lý thế nào?

```javascript
// Global handler — log và graceful shutdown
process.on('unhandledRejection', (reason, promise) => {
  logger.error({ reason, promise }, 'Unhandled rejection');
  // Production: alert + graceful shutdown
});

process.on('uncaughtException', (error) => {
  logger.fatal(error, 'Uncaught exception');
  process.exit(1); // Không tiếp tục trong unknown state
});
```

**Best practice:**
- Luôn `await` hoặc `.catch()` mọi Promise
- ESLint rule `no-floating-promises`
- Trong Express: wrap async handler với `express-async-errors` hoặc wrapper function

---

### Q9: Callback Hell là gì? Giải quyết thế nào?

**Callback Hell:** Nested callbacks sâu → code khó đọc, khó debug, error handling phức tạp.

**Giải pháp theo thứ tự evolution:**
1. **Named functions** — tách callback thành hàm riêng
2. **Promises** — chaining với `.then()`
3. **async/await** — syntax đồng bộ cho async code
4. **Async libraries** — `async.js` waterfall/parallel (legacy)

---

### Q10: `async/await` có thực sự chạy đồng bộ không?

**Không.** `async/await` là **syntactic sugar** (cú pháp tiện lợi) trên Promises:

```javascript
async function example() {
  console.log('1');
  await Promise.resolve(); // yield control — microtask
  console.log('2');
}
example();
console.log('3');
// Output: 1 → 3 → 2
```

Code sau `await` chạy như `.then()` callback — không block main thread.

---

## Câu Hỏi Nâng Cao

### Q11: Event Loop Starvation — nguyên nhân và cách tránh?

**Nguyên nhân:** CPU-intensive sync code hoặc vòng `process.nextTick` vô hạn block Event Loop → không xử lý I/O callbacks → API timeout.

**Giải pháp:**
- Offload CPU work sang **Worker Threads** hoặc **Child Process**
- Chia nhỏ task với `setImmediate` (yield to Event Loop)
- Dùng `cluster` module cho multi-process scaling

```javascript
// Anti-pattern: block Event Loop
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2); // Sync recursion
}
fibonacci(45); // Block vài giây!

// Fix: Worker Thread
const { Worker } = require('worker_threads');
const worker = new Worker('./fib-worker.js', { workerData: { n: 45 } });
```

---

### Q12: Giải thích output của đoạn code phức tạp

```javascript
console.log('start');

setTimeout(() => console.log('timeout 1'), 0);
setImmediate(() => console.log('immediate 1'));

Promise.resolve().then(() => {
  console.log('promise 1');
  process.nextTick(() => console.log('nextTick inside promise'));
});

process.nextTick(() => console.log('nextTick 1'));

setTimeout(() => console.log('timeout 2'), 0);
setImmediate(() => console.log('immediate 2'));

console.log('end');
```

**Output:**
```
start
end
nextTick 1
promise 1
nextTick inside promise
timeout 1    (hoặc immediate 1 — không đảm bảo thứ tự ở top-level)
immediate 1  (hoặc timeout 1)
timeout 2
immediate 2
```

---

### Q13: Blocking vs Non-blocking I/O — ảnh hưởng throughput?

| | Blocking | Non-blocking |
| - | -------- | ------------ |
| **Thread model** | 1 thread/request (Apache) | 1 thread/xử lý nhiều request |
| **Memory** | Cao (mỗi thread ~1MB stack) | Thấp hơn nhiều |
| **Phù hợp** | CPU-bound, ít concurrent connections | I/O-bound, nhiều concurrent connections |
| **Node.js** | `fs.readFileSync` — tránh dùng | `fs.readFile` — default pattern |

**Caveat:** Node.js không phù hợp CPU-intensive workload trên main thread — cần Worker Threads hoặc offload sang service khác.

---

### Q14: `util.promisify` và tại sao cần nó?

Chuyển **error-first callback** sang Promise:

```javascript
const { promisify } = require('util');
const fs = require('fs');
const readFile = promisify(fs.readFile);

// Thay vì:
// fs.readFile('file.txt', 'utf8', (err, data) => { ... });

const data = await readFile('file.txt', 'utf8');
```

Node.js v18+ có `fs.promises` built-in — `promisify` ít cần hơn nhưng vẫn hữu ích cho legacy callback APIs.

---

### Q15: AbortController với async operations?

```javascript
const controller = new AbortController();
const { signal } = controller;

// Fetch với timeout/abort
const response = await fetch(url, { signal });

// Cancel sau 5 giây
setTimeout(() => controller.abort(), 5000);

try {
  const data = await response.json();
} catch (err) {
  if (err.name === 'AbortError') {
    console.log('Request aborted');
  }
}
```

**Điểm cộng:** Đề cập graceful shutdown — abort pending requests khi server nhận SIGTERM.

---

## Bài Tập Thứ Tự Output

### Bài 1

```javascript
async function async1() {
  console.log('async1 start');
  await async2();
  console.log('async1 end');
}

async function async2() {
  console.log('async2');
}

console.log('script start');
setTimeout(() => console.log('setTimeout'), 0);
async1();
new Promise(resolve => {
  console.log('promise1');
  resolve();
}).then(() => console.log('promise2'));
console.log('script end');
```

<details>
<summary>Đáp án</summary>

```
script start
async1 start
async2
promise1
script end
async1 end
promise2
setTimeout
```

</details>

### Bài 2

```javascript
const promise = new Promise((resolve, reject) => {
  console.log('promise executor');
  resolve('resolved');
});

promise.then(val => console.log(val));
promise.then(val => console.log(val + ' again'));

console.log('sync end');
```

<details>
<summary>Đáp án</summary>

```
promise executor
sync end
resolved
resolved again
```

**Giải thích:** Promise executor chạy sync. `.then` callbacks là microtasks, chạy sau sync code.

</details>

---

## Live Coding Challenges

### Challenge 1: Implement `sleep(ms)`

```javascript
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// Dùng:
await sleep(1000);
console.log('1 second later');
```

### Challenge 2: Implement `retry(fn, maxAttempts, delay)`

```javascript
async function retry(fn, maxAttempts = 3, delay = 1000) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxAttempts) throw error;
      await sleep(delay * attempt); // exponential backoff optional
    }
  }
}
```

### Challenge 3: Implement async `forEach` (sequential)

```javascript
async function asyncForEach(array, callback) {
  for (const item of array) {
    await callback(item);
  }
}
// Lưu ý: Array.forEach không await — đây là bug phổ biến!
```

### Challenge 4: Debounce async function

```javascript
function debounceAsync(fn, wait) {
  let timeoutId;
  let pendingReject;

  return function (...args) {
    if (pendingReject) pendingReject(new Error('Debounced'));
    clearTimeout(timeoutId);

    return new Promise((resolve, reject) => {
      pendingReject = reject;
      timeoutId = setTimeout(async () => {
        pendingReject = null;
        try {
          resolve(await fn.apply(this, args));
        } catch (err) {
          reject(err);
        }
      }, wait);
    });
  };
}
```

---

## Tài Liệu Tham Khảo

- [02-async-programming/1-event-loop.md](../02-async-programming/1-event-loop.md)
- [02-async-programming/3-promises.md](../02-async-programming/3-promises.md)
- [02-async-programming/4-async-await.md](../02-async-programming/4-async-await.md)
- [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) — Câu 1–10
