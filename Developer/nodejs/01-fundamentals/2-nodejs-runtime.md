# Node.js Runtime — V8, libuv và Process Model

> Node.js là JavaScript Runtime (Môi Trường Thực Thi JavaScript) được xây dựng trên V8 Engine của Google Chrome và thư viện libuv, cho phép chạy JavaScript phía server với I/O bất đồng bộ hiệu quả.

## Mục Lục

1. [Node.js Là Gì?](#nodejs-là-gì)
2. [Kiến Trúc Runtime](#kiến-trúc-runtime)
3. [V8 Engine](#v8-engine)
4. [libuv và Event Loop](#libuv-và-event-loop)
5. [Single-Threaded Model](#single-threaded-model)
6. [Process Object](#process-object)
7. [Global Objects](#global-objects)
8. [Node.js vs Browser JavaScript](#nodejs-vs-browser-javascript)
9. [Phiên Bản và LTS](#phiên-bản-và-lts)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Node.js Là Gì?

Node.js **không phải** ngôn ngữ hay framework — nó là runtime cho phép thực thi JavaScript bên ngoài trình duyệt.

| Đặc Điểm | Mô Tả |
| -------- | ----- |
| **Ngôn ngữ** | JavaScript (ECMAScript) |
| **Engine** | V8 (Google) — biên dịch JS sang machine code |
| **I/O Model** | Non-blocking, event-driven (hướng sự kiện, không chặn) |
| **Use Case** | REST API, microservices, CLI tools, real-time apps |
| **Không phù hợp** | CPU-intensive tasks trên main thread (ML inference, video encoding) |

```bash
# Kiểm tra phiên bản
node --version    # v20.x.x (LTS)
npm --version     # 10.x.x

# Chạy file JavaScript
node app.js

# REPL (Read-Eval-Print Loop — Vòng Lặp Đọc-Đánh Giá-In)
node
> 1 + 1
2
> process.version
'v20.11.0'
```

---

## Kiến Trúc Runtime

```
┌──────────────────────────────────────────────────────────────┐
│                     NODE.JS PROCESS                           │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                    JavaScript Code                        │ │
│  │         (Your app + npm packages + Node.js APIs)         │ │
│  └─────────────────────────┬──────────────────────────────┘ │
│                            │                                 │
│  ┌─────────────────────────▼──────────────────────────────┐ │
│  │                    V8 JavaScript Engine                   │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │ │
│  │  │   Parser     │→ │  Ignition    │→ │   TurboFan    │  │ │
│  │  │  (Phân tích) │  │ (Interpreter)│  │  (Compiler)   │  │ │
│  │  └──────────────┘  └──────────────┘  └───────────────┘  │ │
│  │  ┌──────────────────────────────────────────────────┐  │ │
│  │  │  Memory: Heap (bộ nhớ động) + Stack (ngăn xếp)    │  │ │
│  │  │  Garbage Collector (GC — Thu Gom Rác)             │  │ │
│  │  └──────────────────────────────────────────────────┘  │ │
│  └─────────────────────────┬──────────────────────────────┘ │
│                            │                                 │
│  ┌─────────────────────────▼──────────────────────────────┐ │
│  │              Node.js Bindings (C++ layer)                │ │
│  │  fs, http, crypto, net, os, child_process, buffer...    │ │
│  └─────────────────────────┬──────────────────────────────┘ │
│                            │                                 │
│  ┌─────────────────────────▼──────────────────────────────┐ │
│  │                        libuv                              │ │
│  │  ┌─────────────────┐  ┌─────────────────────────────┐ │ │
│  │  │   Event Loop      │  │   Thread Pool (mặc định 4)  │ │ │
│  │  │   (Vòng lặp sự    │  │   file I/O, DNS, crypto,    │ │ │
│  │  │    kiện)          │  │   compression               │ │ │
│  │  └─────────────────┘  └─────────────────────────────┘ │ │
│  └─────────────────────────┬──────────────────────────────┘ │
│                            │                                 │
│  ┌─────────────────────────▼──────────────────────────────┐ │
│  │              Operating System (OS — Hệ Điều Hành)         │ │
│  │         epoll (Linux) / kqueue (macOS) / IOCP (Windows)  │ │
│  └──────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

---

## V8 Engine

V8 biên dịch JavaScript thành machine code tối ưu cho CPU.

### Pipeline Thực Thi

```
Source Code → Parser → AST (Abstract Syntax Tree)
                ↓
           Ignition (Interpreter — Trình Thông Dịch)
                ↓
           Bytecode
                ↓
     TurboFan (Optimizing Compiler — Trình Biên Dịch Tối Ưu)
                ↓
          Machine Code
```

### Memory Management

| Vùng Nhớ | Chứa Gì | Đặc Điểm |
| -------- | ------- | -------- |
| **Stack** | Primitives, function call frames | Fixed size, fast, LIFO |
| **Heap** | Objects, arrays, closures, strings | Dynamic, GC quản lý |

```javascript
// Stack: a, b là primitives
let a = 10;
let b = 20;

// Heap: object được allocate trên heap, reference trên stack
const user = { name: 'Alice', scores: [90, 85] };
```

**Garbage Collection (GC — Thu Gom Rác):** V8 tự động giải phóng memory không còn reference. Generational GC chia heap thành Young Generation (thu gom thường xuyên) và Old Generation (thu gom ít hơn). Memory leak xảy ra khi object vẫn được reference dù không cần dùng.

---

## libuv và Event Loop

libuv là thư viện C cung cấp Event Loop và thread pool — trái tim của I/O bất đồng bộ trong Node.js.

### Event Loop Phases (Tóm Tắt)

```
   ┌───────────────────────────┐
┌─>│         timers            │  setTimeout, setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │     pending callbacks     │  I/O callbacks deferred
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │       idle, prepare       │  internal use
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │           poll            │  retrieve new I/O events
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │           check             │  setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │      close callbacks      │  socket.on('close', ...)
│  └───────────────────────────┘
```

**Microtasks** (Promise callbacks, `process.nextTick`) chạy **giữa** các phase — chi tiết tại [02-async-programming/1-event-loop.md](../02-async-programming/1-event-loop.md).

### Thread Pool

Một số operations dùng thread pool thay vì OS async APIs:

- File system (`fs.readFile` trên một số hệ thống)
- DNS lookup (`dns.lookup`)
- Crypto (`crypto.pbkdf2`, `bcrypt`)
- Compression (`zlib`)

```javascript
// Tăng thread pool size (mặc định 4)
process.env.UV_THREADPOOL_SIZE = '8';
```

---

## Single-Threaded Model

JavaScript code trong Node.js chạy trên **một main thread** duy nhất.

```
Main Thread:  [JS Code] → [Event Loop] → [JS Code] → ...
Thread Pool:  [fs read] [crypto] [dns]  (parallel, background)
```

### Hệ Quả Thực Tế

```javascript
// BAD — block main thread
app.get('/compute', (req, res) => {
  let result = 0;
  for (let i = 0; i < 1e10; i++) result += i; // Block ~vài giây
  res.json({ result });
});

// GOOD — offload CPU work
const { Worker } = require('worker_threads');
app.get('/compute', (req, res) => {
  const worker = new Worker('./compute-worker.js');
  worker.on('message', result => res.json({ result }));
});
```

| Mô Hình | Khi Nào Dùng |
| ------- | ------------ |
| **Single thread + async I/O** | REST API, database queries, file I/O |
| **Cluster module** | Scale HTTP server across CPU cores |
| **Worker Threads** | CPU-intensive computation |
| **Child Process** | Chạy external program, isolate crash |

---

## Process Object

`process` là global object cung cấp thông tin và điều khiển tiến trình Node.js hiện tại.

### Thông Tin Process

```javascript
process.pid;           // Process ID
process.version;       // Node.js version
process.versions;        // { node, v8, openssl, ... }
process.platform;      // 'linux', 'darwin', 'win32'
process.arch;          // 'x64', 'arm64'
process.cwd();         // Current working directory
process.uptime();      // Seconds since process started
process.memoryUsage(); // { rss, heapTotal, heapUsed, external }
```

### Environment Variables

```javascript
// Đọc biến môi trường
const port = process.env.PORT || 3000;
const dbUrl = process.env.DATABASE_URL;

// Thường dùng dotenv để load .env file
// require('dotenv').config();
// process.env.NODE_ENV === 'development' | 'production' | 'test'
```

### Process Events

```javascript
// Graceful shutdown (tắt an toàn)
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, shutting down...');
  await server.close();
  await db.disconnect();
  process.exit(0);
});

process.on('SIGINT', () => {
  console.log('Ctrl+C pressed');
  process.exit(0);
});

// Unhandled errors — production phải handle
process.on('uncaughtException', (err) => {
  console.error('Uncaught Exception:', err);
  process.exit(1); // Không nên tiếp tục chạy
});

process.on('unhandledRejection', (reason) => {
  console.error('Unhandled Rejection:', reason);
  process.exit(1);
});
```

### Exit Codes

```javascript
process.exit(0);  // Success
process.exit(1);  // General failure
// Không gọi process.exit() trong request handler — dùng graceful shutdown
```

---

## Global Objects

| Global | Mô Tả | Browser Tương Đương |
| ------ | ----- | ------------------- |
| `global` | Global object trong Node.js | `window` |
| `process` | Process info và control | Không có |
| `Buffer` | Binary data handling | Không có (dùng ArrayBuffer) |
| `__dirname` | Đường dẫn thư mục file hiện tại (CommonJS) | Không có |
| `__filename` | Đường dẫn file hiện tại (CommonJS) | Không có |
| `console` | Logging | `console` |
| `setTimeout` / `setInterval` | Timers | Có |
| `setImmediate` | Chạy sau I/O phase | Không có |

### ESM Equivalents

```javascript
// CommonJS
console.log(__dirname, __filename);

// ESM — không có __dirname mặc định
import { fileURLToPath } from 'url';
import { dirname } from 'path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```

### globalThis

```javascript
// Cross-platform global reference (Node.js, browser, worker)
globalThis.setTimeout(() => {}, 1000);
```

---

## Node.js vs Browser JavaScript

| Khía Cạnh | Node.js | Browser |
| --------- | ------- | ------- |
| **Global object** | `global` / `globalThis` | `window` |
| **DOM** | Không có | Có |
| **Module system** | CommonJS + ESM | ESM (script type="module") |
| **File system** | `fs` module | Không có (sandbox) |
| **HTTP server** | `http` module | Không có |
| **Environment** | Server, CLI | Client UI |

---

## Phiên Bản và LTS

Node.js theo **release schedule**:

| Loại | Mô Tả | Khuyến Nghị |
| ---- | ----- | ----------- |
| **Current** | Features mới nhất, support ~8 tháng | Development, thử nghiệm |
| **LTS (Long Term Support — Hỗ Trợ Dài Hạn)** | Ổn định, security patches ~30 tháng | **Production** |
| **EOL (End of Life — Hết Hạn)** | Không còn security updates | Nâng cấp ngay |

```bash
# Dùng nvm để quản lý phiên bản
nvm install --lts
nvm use --lts
nvm list

# Kiểm tra deprecation warnings
node --trace-deprecation app.js
```

**Khuyến nghị 2026:** Dùng Node.js 20 LTS hoặc 22 LTS cho production.

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Giải thích kiến trúc Node.js

**Gợi ý trả lời:** Node.js gồm V8 (thực thi JavaScript), Node.js bindings (C++ APIs như fs, http), và libuv (Event Loop + thread pool). JavaScript chạy single-threaded trên main thread; I/O operations delegate cho OS hoặc thread pool, callback được đưa vào Event Loop khi hoàn thành.

### Câu 2: Node.js xử lý concurrent requests như thế nào nếu single-threaded?

**Gợi ý trả lời:** Nhờ non-blocking I/O và Event Loop. Khi request cần đọc database, Node.js gửi query và **không chờ** — tiếp tục xử lý request khác. Khi DB trả kết quả, callback được đưa vào Event Loop. Với I/O-bound workloads (đa số REST API), một process có thể handle hàng nghìn concurrent connections.

### Câu 3: `process.nextTick()` vs `setImmediate()` vs `setTimeout(fn, 0)`?

**Gợi ý trả lời:** `process.nextTick()` chạy trước mọi thứ — ngay sau current operation, trước Event Loop phase tiếp theo. `setImmediate()` chạy trong check phase, sau poll. `setTimeout(fn, 0)` chạy trong timers phase. Thứ tự: nextTick → microtasks (Promise) → timers → ... → check (setImmediate).

### Câu 4: `NODE_ENV` dùng để làm gì?

**Gợi ý trả lời:** Convention đánh dấu môi trường chạy: `development`, `production`, `test`. Framework và thư viện dùng để bật/tắt features: Express error stack trace, React production build, logging level. Không phải built-in Node.js variable — là community convention.

---

**Xem tiếp:** [3-module-systems.md](./3-module-systems.md) — CommonJS vs ESM.
