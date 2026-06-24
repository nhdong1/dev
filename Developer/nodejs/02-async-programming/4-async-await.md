# async/await — Cú Pháp, Parallel vs Sequential Execution

> async/await là **syntactic sugar (cú pháp ngọt)** trên Promises — cho phép viết async code đọc như synchronous code mà vẫn non-blocking. Đây là chuẩn trong hầu hết Node.js codebase hiện đại.

## Mục Lục

1. [async/await Cơ Bản](#asyncawait-cơ-bản)
2. [async Function Luôn Trả Về Promise](#async-function-luôn-trả-về-promise)
3. [await — Tạm Dừng Không Phải Block](#await--tạm-dừng-không-phải-block)
4. [Sequential vs Parallel Execution](#sequential-vs-parallel-execution)
5. [Error Handling với try/catch](#error-handling-với-trycatch)
6. [Loops và async/await](#loops-và-asyncawait)
7. [Top-Level await](#top-level-await)
8. [Common Pitfalls](#common-pitfalls)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## async/await Cơ Bản

```javascript
// Promise style
function getUser(id) {
  return fetch(`/api/users/${id}`)
    .then((res) => res.json());
}

// async/await style
async function getUser(id) {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}

// Sử dụng
async function main() {
  const user = await getUser(1);
  console.log(user);
}

main().catch(console.error);
```

| Keyword | Ý Nghĩa |
| ------- | ------- |
| `async` | Đánh dấu function trả về Promise |
| `await` | Chờ Promise settle, trả về value hoặc throw rejection |

---

## async Function Luôn Trả Về Promise

```javascript
async function foo() {
  return 42;
}

foo() instanceof Promise; // true
foo().then((v) => console.log(v)); // 42

async function bar() {
  throw new Error('fail');
}

bar().catch((err) => console.log(err.message)); // 'fail'

// Tương đương:
function barEquivalent() {
  return Promise.reject(new Error('fail'));
}
```

### Gọi async Function

```javascript
// Cách 1: await (trong async context)
async function main() {
  const result = await doWork();
}

// Cách 2: .then/.catch
doWork().then(handleResult).catch(handleError);

// Cách 3: Top-level await (ESM modules)
// const result = await doWork();
```

---

## await — Tạm Dừng Không Phải Block

```javascript
async function demo() {
  console.log('A');
  await Promise.resolve();
  console.log('B'); // microtask — sau sync code bên ngoài
}

console.log('Start');
demo();
console.log('End');

// Start → A → End → B
```

**Quan trọng:** `await` chỉ suspend **function async hiện tại**, không block Event Loop hay main thread. Code khác (request khác, timers) vẫn chạy.

```javascript
// BLOCK thật sự — sync CPU work
function blocking() {
  for (let i = 0; i < 1e9; i++) {} // Event Loop đứng yên
}

// NON-BLOCK — await I/O
async function nonBlocking() {
  await fs.readFile('large.txt'); // main thread free
}
```

---

## Sequential vs Parallel Execution

### Sequential — Chờ Từng Cái (Chậm Hơn)

```javascript
async function fetchSequential(urls) {
  const results = [];
  for (const url of urls) {
    const res = await fetch(url);  // chờ từng request
    results.push(await res.json());
  }
  return results;
}
// Tổng thời gian ≈ sum(latency của từng request)
```

### Parallel — Chạy Đồng Thời (Nhanh Hơn)

```javascript
async function fetchParallel(urls) {
  const promises = urls.map((url) =>
    fetch(url).then((res) => res.json())
  );
  return Promise.all(promises);
}
// Tổng thời gian ≈ max(latency) — request chậm nhất
```

### So Sánh Trực Quan

```
Sequential (3 requests × 200ms each):
|--req1--|--req2--|--req3--|  = 600ms

Parallel:
|--req1--|
|--req2--|
|--req3--|  = 200ms
```

### Khi Nào Dùng Gì?

| Pattern | Khi Nào |
| ------- | ------- |
| **Sequential** | Bước sau phụ thuộc kết quả bước trước; rate limit API; tiết kiệm resource |
| **Parallel** | Independent requests; read-heavy; latency-sensitive |
| **Batched parallel** | Nhiều items nhưng giới hạn concurrency (xem file 6) |

```javascript
// Phụ thuộc — phải sequential
async function createOrder(userId, items) {
  const user = await getUser(userId);
  const inventory = await checkInventory(items);
  return await placeOrder(user, inventory);
}

// Independent — nên parallel
async function loadDashboard(userId) {
  const [user, orders, notifications] = await Promise.all([
    getUser(userId),
    getOrders(userId),
    getNotifications(userId),
  ]);
  return { user, orders, notifications };
}
```

---

## Error Handling với try/catch

```javascript
async function processOrder(orderId) {
  try {
    const order = await getOrder(orderId);
    const payment = await chargePayment(order);
    await sendConfirmation(order.email);
    return { success: true, paymentId: payment.id };
  } catch (err) {
    if (err.code === 'PAYMENT_FAILED') {
      await rollbackInventory(orderId);
    }
    throw err; // rethrow cho caller
  } finally {
    await logAudit(orderId);
  }
}
```

### try/catch vs .catch()

```javascript
// Tương đương
async function a() {
  try {
    await risky();
  } catch (err) {
    handle(err);
  }
}

function b() {
  return risky().catch(handle);
}
```

### Error Trong Promise.all

```javascript
try {
  const [a, b, c] = await Promise.all([fetchA(), fetchB(), fetchC()]);
} catch (err) {
  // Một trong ba fail → catch err đầu tiên
  // Các request khác vẫn chạy (không cancel tự động)
}
```

---

## Loops và async/await

### for...of + await — Sequential

```javascript
async function processItems(items) {
  const results = [];
  for (const item of items) {
    results.push(await processOne(item)); // tuần tự
  }
  return results;
}
```

### forEach + async — SAI (Không Chờ)

```javascript
// NGUY HIỂM — forEach không await async callback
async function broken(items) {
  items.forEach(async (item) => {
    await processOne(item);
  });
  console.log('Done'); // In NGAY — không chờ processOne
}

// ĐÚNG
async function fixed(items) {
  await Promise.all(items.map((item) => processOne(item)));
}
```

### map + Promise.all — Parallel

```javascript
const results = await Promise.all(
  items.map((item) => processOne(item))
);
```

### Giới Hạn Concurrency Trong Loop

```javascript
// Xem chi tiết ở 6-concurrency-patterns.md
async function processWithLimit(items, limit = 5) {
  const results = [];
  for (let i = 0; i < items.length; i += limit) {
    const batch = items.slice(i, i + limit);
    const batchResults = await Promise.all(batch.map(processOne));
    results.push(...batchResults);
  }
  return results;
}
```

---

## Top-Level await

Cho phép `await` ở module top-level (không trong async function):

```javascript
// config.mjs (ESM only)
import { readFile } from 'fs/promises';

export const config = JSON.parse(
  await readFile(new URL('./config.json', import.meta.url), 'utf8')
);

// app.mjs
import { config } from './config.mjs';
console.log(config.port); // config đã load xong trước khi app chạy
```

### Yêu Cầu

| Yêu Cầu | Chi Tiết |
| ------- | -------- |
| **ESM only** | `"type": "module"` hoặc file `.mjs` |
| **Blocking module graph** | Module import phải chờ top-level await xong |
| **Không dùng trong CommonJS** | `require()` không hỗ trợ |

```json
// package.json
{
  "type": "module"
}
```

---

## Common Pitfalls

### Pitfall 1: Quên await

```javascript
async function getData() {
  const data = fetch('/api/data'); // Promise, không phải data!
  console.log(data.name); // undefined — bug khó phát hiện
}

async function getDataFixed() {
  const res = await fetch('/api/data');
  const data = await res.json();
  console.log(data.name);
}
```

### Pitfall 2: Sequential Khi Có Thể Parallel

```javascript
// CHẬM
async function slow() {
  const a = await fetchA();
  const b = await fetchB();
  return [a, b];
}

// NHANH
async function fast() {
  const [a, b] = await Promise.all([fetchA(), fetchB()]);
  return [a, b];
}
```

### Pitfall 3: async Void Function

```javascript
// Express handler — phải catch errors
app.get('/users', async (req, res) => {
  const users = await getUsers();
  res.json(users);
  // Nếu getUsers() throw → unhandled rejection
  // trừ khi có express async error wrapper
});

// ĐÚNG — dùng wrapper hoặc try/catch
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

app.get('/users', asyncHandler(async (req, res) => {
  const users = await getUsers();
  res.json(users);
}));
```

### Pitfall 4: await Trong Constructor

```javascript
// KHÔNG ĐƯỢC — constructor không thể async
class Service {
  constructor() {
    // await this.init(); // SyntaxError
  }

  static async create() {
    const instance = new Service();
    await instance.init();
    return instance;
  }
}

const service = await Service.create();
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: async/await có block main thread không?

**Gợi ý trả lời:** Không. `await` suspend async function, Event Loop tiếp tục. Chỉ sync CPU-intensive code block. await I/O = delegate cho libuv, callback qua microtask khi xong.

### Câu 2: Làm sao chạy 3 async operations song song?

**Gợi ý trả lời:** `const [a, b, c] = await Promise.all([op1(), op2(), op3()])`. Không dùng 3 await tuần tự nếu independent. Hoặc start promises trước rồi await: `const p1 = op1(); const p2 = op2(); await p1; await p2`.

### Câu 3: `forEach` với async function có hoạt động không?

**Gợi ý trả lời:** Callback async chạy nhưng forEach không await. Loop kết thúc trước khi async work xong. Dùng `for...of` + await (sequential) hoặc `map` + `Promise.all` (parallel).

### Câu 4: async function return non-Promise value?

**Gợi ý trả lời:** Tự động wrap trong `Promise.resolve(value)`. `async () => 42` tương đương `() => Promise.resolve(42)`. Throw trong async = `Promise.reject`.

### Câu 5: Top-level await hoạt động thế nào?

**Gợi ý trả lời:** Chỉ trong ESM modules. Module evaluation pause tại await, resume khi Promise settle. Importers chờ module load xong. Dùng cho config loading, DB connection init.

---

**Xem tiếp:** [5-error-handling.md](./5-error-handling.md) — Xử lý lỗi async trong production.
