# Event Loop — Vòng Lặp Sự Kiện, Call Stack và Microtasks

> Event Loop (Vòng Lặp Sự Kiện) là cơ chế cho phép Node.js thực hiện non-blocking I/O (I/O không chặn) trên single main thread. Đây là câu hỏi phỏng vấn Node.js số 1.

## Mục Lục

1. [Sync vs Async — Tại Sao Cần Event Loop](#sync-vs-async--tại-sao-cần-event-loop)
2. [Call Stack (Ngăn Xếp Gọi Hàm)](#call-stack-ngăn-xếp-gọi-hàm)
3. [Web APIs / Node.js APIs và Thread Pool](#web-apis--nodejs-apis-và-thread-pool)
4. [Callback Queue và Microtask Queue](#callback-queue-và-microtask-queue)
5. [Event Loop Phases Chi Tiết](#event-loop-phases-chi-tiết)
6. [process.nextTick vs setImmediate vs setTimeout](#processnexttick-vs-setimmediate-vs-settimeout)
7. [Bài Tập Thứ Tự Output](#bài-tập-thứ-tự-output)
8. [Starvation và Best Practices](#starvation-và-best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Sync vs Async — Tại Sao Cần Event Loop

### Blocking (Chặn) — Vấn Đề

```javascript
// Giả lập blocking I/O — KHÔNG làm thế này trong production
const fs = require('fs');

// fs.readFileSync — block main thread cho đến khi đọc xong
const data = fs.readFileSync('large-file.json', 'utf8');
console.log('Done reading');
// Trong lúc chờ, KHÔNG request nào khác được xử lý
```

### Non-Blocking (Không Chặn) — Giải Pháp

```javascript
const fs = require('fs');

fs.readFile('large-file.json', 'utf8', (err, data) => {
  if (err) throw err;
  console.log('Done reading');
});
console.log('Reading started...'); // In TRƯỚC khi đọc xong
// Main thread tiếp tục xử lý request khác
```

| Khái Niệm | Mô Tả |
| --------- | ----- |
| **Synchronous (Đồng Bộ)** | Code chạy tuần tự, operation sau chờ operation trước |
| **Asynchronous (Bất Đồng Bộ)** | Gửi operation, tiếp tục code khác, callback khi xong |
| **Event Loop** | Cơ chế dispatch callback khi I/O hoàn thành |
| **Concurrency (Đồng Thời)** | Nhiều task "cùng lúc" trên single thread (interleaving — xen kẽ) |
| **Parallelism (Song Song)** | Nhiều thread/process thực sự chạy đồng thời |

---

## Call Stack (Ngăn Xếp Gọi Hàm)

Call Stack là cấu trúc LIFO (Last In, First Out — Vào Sau Ra Trước) theo dõi function đang thực thi.

```javascript
function third() {
  console.log('third');
}

function second() {
  third();
}

function first() {
  second();
}

first();
```

```
Call Stack evolution:

[first]           → push first()
[first, second]   → push second()
[first, second, third] → push third()
[first, second]   → third() return, pop
[first]           → second() return, pop
[]                → first() return, pop
```

**Stack Overflow:** Khi recursion (đệ quy) quá sâu hoặc vòng lặp vô hạn gọi hàm — `Maximum call stack size exceeded`.

---

## Web APIs / Node.js APIs và Thread Pool

Khi gặp async operation (setTimeout, fs.readFile, http.request), Node.js **không chờ** trên call stack:

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐
│ Call Stack  │     │  Node.js / libuv │     │  Thread Pool    │
│             │     │  (register I/O)  │     │  (4 threads)    │
│ main()      │ ──► │  fs.readFile()   │ ──► │  đọc file       │
│             │     │  return ngay     │     │  (background)   │
└─────────────┘     └──────────────────┘     └────────┬────────┘
                                                      │
                                                      ▼
                                             Callback vào Queue
                                             khi I/O xong
```

| Operation | Xử Lý Bởi |
| --------- | ---------- |
| `setTimeout`, `setInterval` | Timers phase của Event Loop |
| `fs.readFile` (mặc định) | Thread pool → poll phase callback |
| `fs.readFile` với flag sync | Block main thread |
| `http.get`, network I/O | OS async I/O (epoll/kqueue) |
| `crypto.pbkdf2` | Thread pool |
| `dns.lookup` | Thread pool (hoặc c-ares async) |
| Pure JavaScript computation | Call stack — **blocks** main thread |

---

## Callback Queue và Microtask Queue

Node.js có nhiều queue với **priority (ưu tiên)** khác nhau:

```
Priority (cao → thấp):

1. process.nextTick queue     — chạy sau mỗi operation, trước Event Loop
2. Microtask queue            — Promise.then/catch/finally, queueMicrotask()
3. Macrotask queues           — setTimeout, setImmediate, I/O callbacks
```

### Luồng Thực Thi Một "Tick"

```
1. Chạy code sync trên call stack cho đến khi rỗng
2. Drain toàn bộ nextTick queue (có thể recursive)
3. Drain toàn bộ microtask queue (Promise callbacks)
4. Chạy MỘT phase của Event Loop (hoặc tiếp tục microtasks nếu có)
5. Lặp lại
```

```javascript
console.log('A');

setTimeout(() => console.log('B'), 0);        // macrotask — timers

Promise.resolve().then(() => console.log('C')); // microtask

process.nextTick(() => console.log('D'));      // nextTick — cao nhất

console.log('E');

// Output: A → E → D → C → B
```

---

## Event Loop Phases Chi Tiết

Event Loop trong libuv có **6 phases** (theo thứ tự):

```
   ┌───────────────────────────┐
┌─>│         timers            │  setTimeout, setInterval callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │     pending callbacks     │  I/O callbacks deferred từ lần trước
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │       idle, prepare       │  internal — libuv dùng nội bộ
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │          poll             │  Lấy I/O events mới; block ở đây nếu không có gì
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │          check            │  setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │      close callbacks      │  socket.on('close'), etc.
│  └─────────────┬─────────────┘
│                │
└────────────────┘ (lặp lại nếu còn callbacks)
```

### Giải Thích Từng Phase

| Phase | Chức Năng | Ví Dụ |
| ----- | --------- | ----- |
| **timers** | Thực thi timer callbacks đã đến hạn | `setTimeout(fn, 1000)` |
| **pending callbacks** | Callbacks hệ thống bị defer | một số TCP errors |
| **idle, prepare** | Internal libuv | không dùng trực tiếp |
| **poll** | Lấy I/O events; có thể block chờ I/O | `fs.readFile` callback, incoming HTTP |
| **check** | `setImmediate` callbacks | defer work sau poll |
| **close callbacks** | Cleanup khi handle đóng | `ws.on('close')` |

### Poll Phase — Quan Trọng Nhất

Poll phase:
1. Tính thời gian block cho timers
2. Thực thi pending I/O callbacks trong queue
3. Nếu queue rỗng: chờ I/O mới hoặc chuyển phase (nếu có setImmediate)

Đây là lý do Node.js hiệu quả với nhiều concurrent connections — poll chờ I/O từ OS, không busy-wait.

---

## process.nextTick vs setImmediate vs setTimeout

```javascript
// nextTick — chạy TRƯỚC Event Loop phase tiếp theo
process.nextTick(() => {
  console.log('nextTick');
});

// setImmediate — check phase, SAU poll
setImmediate(() => {
  console.log('setImmediate');
});

// setTimeout(0) — timers phase
setTimeout(() => {
  console.log('setTimeout');
}, 0);
```

### Trong I/O Callback

```javascript
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
});

// Trong I/O callback: setImmediate thường chạy TRƯỚC setTimeout(0)
// Vì check phase ngay sau poll phase hiện tại
```

### So Sánh

| API | Queue | Khi Nào Dùng |
| --- | ----- | ------------ |
| `process.nextTick(fn)` | nextTick queue | Defer sau sync code, trước I/O; dùng tiết kiệm |
| `queueMicrotask(fn)` | microtask queue | Tương tự Promise.then — chuẩn cross-platform |
| `Promise.resolve().then(fn)` | microtask queue | Async continuation sau Promise resolve |
| `setImmediate(fn)` | check phase | Defer work sau I/O poll |
| `setTimeout(fn, 0)` | timers phase | Delay tối thiểu (thực tế ≥ 1ms do timer resolution) |

---

## Bài Tập Thứ Tự Output

### Puzzle 1

```javascript
console.log('start');

setTimeout(() => console.log('timeout 1'), 0);
setTimeout(() => console.log('timeout 2'), 0);

Promise.resolve()
  .then(() => console.log('promise 1'))
  .then(() => console.log('promise 2'));

process.nextTick(() => {
  console.log('nextTick 1');
  process.nextTick(() => console.log('nextTick 2'));
});

console.log('end');

// start → end → nextTick 1 → nextTick 2 → promise 1 → promise 2 → timeout 1 → timeout 2
```

### Puzzle 2 — async/await

```javascript
async function foo() {
  console.log('foo start');
  await bar();
  console.log('foo end');
}

async function bar() {
  console.log('bar');
}

console.log('script start');
foo();
console.log('script end');

// script start → foo start → bar → script end → foo end
// await bar() = microtask cho phần sau await
```

### Puzzle 3 — Nested nextTick

```javascript
process.nextTick(() => {
  console.log('tick 1');
  process.nextTick(() => console.log('tick 2'));
});
process.nextTick(() => console.log('tick 3'));

// tick 1 → tick 2 → tick 3
// nextTick queue drain hết trước khi sang microtask/macrotask
```

---

## Starvation và Best Practices

### Event Loop Starvation (Đói Vòng Lặp)

```javascript
// NGUY HIỂM — vòng nextTick vô hạn block Event Loop
function spin() {
  process.nextTick(spin);
}
// spin(); // Không bao giờ đến I/O phase — server không respond

// NGUY HIỂM — CPU-intensive trên main thread
function heavyCompute() {
  let sum = 0;
  for (let i = 0; i < 1e9; i++) sum += i;
  return sum;
}
// Block toàn bộ Event Loop trong vài giây
```

### Best Practices

| Practice | Lý Do |
| -------- | ----- |
| Tránh sync I/O (`readFileSync`) trên request path | Block mọi concurrent request |
| CPU-heavy work → Worker Threads hoặc child process | Giữ Event Loop responsive |
| Hạn chế recursive `process.nextTick` | Có thể starve I/O |
| Monitor **event loop lag** | `perf_hooks`, clinic.js, Prometheus |
| `setImmediate` thay nextTick cho defer lớn | Nhường I/O giữa các batch |

```javascript
const { monitorEventLoopDelay } = require('perf_hooks');

const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();

setInterval(() => {
  console.log(`Event loop p99 delay: ${h.percentile(99) / 1e6}ms`);
  h.reset();
}, 5000);
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Vẽ và giải thích Event Loop

**Gợi ý trả lời:** JavaScript chạy trên single call stack. Async operations đăng ký với libuv/OS. Khi hoàn thành, callback vào queue. Event Loop lấy callback khi stack rỗng. Microtasks (Promise) chạy sau mỗi macrotask. 6 phases: timers → pending → idle/prepare → poll → check → close.

### Câu 2: Microtask vs Macrotask?

**Gợi ý trả lời:** Microtasks: Promise callbacks, queueMicrotask, (và nextTick — technically riêng). Macrotasks: setTimeout, setInterval, setImmediate, I/O. Sau mỗi macrotask, drain hết microtasks trước macrotask tiếp theo.

### Câu 3: Tại sao `setTimeout(fn, 0)` không chạy ngay?

**Gợi ý trả lời:** Callback vào timers queue, chờ ít nhất một Event Loop iteration. Timer resolution tối thiểu ~1ms (browser) hoặc phụ thuộc hệ thống. Phải chờ sync code và microtasks hiện tại xong.

### Câu 4: Node.js handle 10,000 concurrent connections thế nào?

**Gợi ý trả lời:** Non-blocking I/O + Event Loop. Mỗi connection không chiếm thread riêng. Khi chờ data, callback pending; Event Loop xử lý connection khác. Giới hạn thực tế: memory per connection, file descriptors, và không block main thread.

### Câu 5: Làm sao phát hiện Event Loop bị block?

**Gợi ý trả lời:** Monitor event loop delay/lag. Symptoms: API latency spike đồng loạt, health check timeout, heartbeat miss. Tools: clinic.js doctor, `perf_hooks.monitorEventLoopDelay`, APM metrics. Root cause thường là sync crypto, JSON.parse lớn, hoặc tight loop.

---

**Xem tiếp:** [2-callbacks.md](./2-callbacks.md) — Callback pattern và error-first convention.
