# Promises — Promise API, Chaining và Combinators

> Promise (Lời Hứa) là abstraction cho giá trị có thể chưa có ngay — đại diện cho kết quả tương lai của async operation. Là nền tảng của async/await và hầu hết modern Node.js APIs.

## Mục Lục

1. [Promise Là Gì?](#promise-là-gì)
2. [Ba Trạng Thái Promise](#ba-trạng-thái-promise)
3. [Tạo và Consume Promise](#tạo-và-consume-promise)
4. [Promise Chaining](#promise-chaining)
5. [Error Handling với Promise](#error-handling-với-promise)
6. [Promise Combinators](#promise-combinators)
7. [Promise Utilities](#promise-utilities)
8. [Common Pitfalls](#common-pitfalls)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Promise Là Gì?

Promise là object đại diện cho **eventual completion (hoàn thành trong tương lai)** hoặc failure của async operation.

```javascript
const promise = new Promise((resolve, reject) => {
  // Executor function — chạy ngay lập tức (sync)
  setTimeout(() => {
    const success = true;
    if (success) {
      resolve({ id: 1, name: 'Alice' });
    } else {
      reject(new Error('Failed'));
    }
  }, 100);
});

promise
  .then((data) => console.log('Success:', data))
  .catch((err) => console.error('Error:', err))
  .finally(() => console.log('Cleanup'));
```

| Thuật Ngữ | Giải Thích |
| --------- | ---------- |
| **Executor** | Function `(resolve, reject) => {}` chạy khi `new Promise()` |
| **Settlement (Giải Quyết)** | Promise chuyển từ pending sang fulfilled hoặc rejected |
| **Thenable** | Object có method `.then()` — tương thích Promise |
| **Microtask** | `.then/catch/finally` callbacks chạy trong microtask queue |

---

## Ba Trạng Thái Promise

```
                    ┌─────────────┐
                    │   pending   │  (chờ kết quả)
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
     ┌────────────────┐       ┌────────────────┐
     │   fulfilled    │       │    rejected    │
     │  (resolve)     │       │   (reject)     │
     └────────────────┘       └────────────────┘
              │                         │
              └──────── immutable ──────┘
                    (không đổi state)
```

```javascript
const p = new Promise((resolve) => {
  resolve(42);
  resolve(99);  // bị ignore — đã settled
  reject(new Error('nope')); // bị ignore
});

// Kiểm tra state (chủ yếu debug)
Promise.resolve(1).then((v) => {
  // v === 1
});
```

**Immutable (Bất Biến):** Một khi fulfilled/rejected, state không đổi. `resolve`/`reject` chỉ có hiệu lực lần đầu.

---

## Tạo và Consume Promise

### Các Cách Tạo Promise

```javascript
// 1. new Promise
const p1 = new Promise((resolve, reject) => {
  resolve('done');
});

// 2. Promise.resolve — wrap giá trị sync hoặc thenable
const p2 = Promise.resolve(42);
const p3 = Promise.resolve(Promise.resolve(42)); // flatten

// 3. Promise.reject
const p4 = Promise.reject(new Error('fail'));

// 4. Từ async function — luôn trả về Promise
async function fetchUser() {
  return { id: 1 }; // tự động wrap trong Promise.resolve
}

// 5. Từ promisified callback
const readFile = require('fs/promises').readFile;
const p5 = readFile('config.json');
```

### Consume Promise

```javascript
// .then(onFulfilled, onRejected) — onRejected ít dùng, prefer .catch
promise.then(
  (value) => console.log(value),
  (err) => console.error(err)
);

// .catch — shorthand cho .then(null, onRejected)
promise.catch((err) => console.error(err));

// .finally — chạy dù success hay fail (cleanup)
promise.finally(() => {
  console.log('Always runs');
});
```

---

## Promise Chaining

`.then()` trả về **Promise mới** — cho phép chain:

```javascript
const readFile = require('fs/promises').readFile;

readFile('user.json', 'utf8')
  .then((data) => JSON.parse(data))      // return value → next .then nhận
  .then((user) => {
    if (!user.active) throw new Error('User inactive');
    return user;
  })
  .then((user) => readFile(`orders/${user.id}.json`, 'utf8'))
  .then((data) => JSON.parse(data))
  .then((orders) => console.log(orders))
  .catch((err) => console.error('Pipeline failed:', err.message));
```

### Return Rules Trong Chain

```javascript
// Return primitive → Promise.resolve(primitive)
.then((x) => x + 1)

// Return Promise → chain flatten (unwrap)
.then((user) => fetchOrders(user.id)) // fetchOrders trả về Promise

// Throw → Promise.reject → jump to .catch
.then(() => { throw new Error('oops'); })

// Không return → undefined passed to next .then
.then(() => { doSomething(); }) // next nhận undefined
```

### Anti-Pattern: Nested .then (Promise Hell)

```javascript
// SAI — lại thành pyramid
getUser(id)
  .then((user) => {
    getOrders(user.id)
      .then((orders) => {
        getDetails(orders[0].id)
          .then((details) => console.log(details));
      });
  });

// ĐÚNG — flat chain
getUser(id)
  .then((user) => getOrders(user.id))
  .then((orders) => getDetails(orders[0].id))
  .then((details) => console.log(details))
  .catch(handleError);

// TỐT NHẤT — async/await (file 4)
```

---

## Error Handling với Promise

```javascript
// Lỗi "trôi" xuống .catch gần nhất
Promise.resolve()
  .then(() => { throw new Error('step 1 fail'); })
  .then(() => console.log('skipped'))
  .catch((err) => {
    console.error(err.message); // 'step 1 fail'
    return 'recovered'; // recover — chain tiếp tục fulfilled
  })
  .then((val) => console.log(val)); // 'recovered'
```

### Lỗi Sync Trong Executor

```javascript
new Promise((resolve, reject) => {
  throw new Error('sync throw'); // tự động → reject
});

new Promise((resolve, reject) => {
  JSON.parse('invalid'); // SyntaxError → reject
});
```

### Unhandled Rejection

```javascript
// NGUY HIỂM — không có .catch
Promise.reject(new Error('orphan'));

// Node.js emit 'unhandledRejection'
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled:', reason);
});
```

---

## Promise Combinators

### Promise.all — Fail-Fast, Chờ Tất Cả

```javascript
const urls = ['/api/users', '/api/orders', '/api/products'];

const results = await Promise.all(
  urls.map((url) => fetch(url).then((r) => r.json()))
);
// results = [users, orders, products]
// Một reject → cả Promise.all reject ngay
```

### Promise.allSettled — Chờ Tất Cả, Không Fail-Fast

```javascript
const outcomes = await Promise.allSettled([
  fetch('/api/a'),
  fetch('/api/b'),
  Promise.reject(new Error('fail')),
]);

// [
//   { status: 'fulfilled', value: Response },
//   { status: 'fulfilled', value: Response },
//   { status: 'rejected', reason: Error }
// ]

outcomes.forEach((result) => {
  if (result.status === 'fulfilled') {
    console.log('OK:', result.value);
  } else {
    console.error('Fail:', result.reason);
  }
});
```

### Promise.race — Kết Quả Đầu Tiên (Thắng Hoặc Thua)

```javascript
const timeout = new Promise((_, reject) =>
  setTimeout(() => reject(new Error('Timeout')), 5000)
);

const data = await Promise.race([
  fetch('/api/slow-endpoint'),
  timeout,
]);
// Cái nào settle trước thắng
```

### Promise.any — Fulfilled Đầu Tiên (ES2021)

```javascript
// Chỉ reject khi TẤT CẢ reject
const fastest = await Promise.any([
  fetch('https://cdn1.example.com/data'),
  fetch('https://cdn2.example.com/data'),
  fetch('https://cdn3.example.com/data'),
]);
// Dùng cho redundancy — lấy CDN respond nhanh nhất
```

### So Sánh Combinators

| Combinator | Resolve Khi | Reject Khi | Use Case |
| ---------- | ----------- | ---------- | -------- |
| `Promise.all` | Tất cả fulfilled | Một reject | Parallel tasks, cần tất cả |
| `Promise.allSettled` | Luôn (sau tất cả) | Không bao giờ* | Batch với partial failure |
| `Promise.race` | Đầu tiên settle | Đầu tiên reject | Timeout, fastest wins |
| `Promise.any` | Đầu tiên fulfilled | Tất cả reject | Fallback mirrors |

*`allSettled` luôn resolve với array outcomes; không reject.

---

## Promise Utilities

### Promise.withResolvers() — Node.js 22+

```javascript
const { promise, resolve, reject } = Promise.withResolvers();

// Gọi resolve/reject từ bên ngoài executor
setTimeout(() => resolve('done'), 1000);

await promise;
```

### Manual Deferred Pattern (Trước withResolvers)

```javascript
function createDeferred() {
  let resolve, reject;
  const promise = new Promise((res, rej) => {
    resolve = res;
    reject = rej;
  });
  return { promise, resolve, reject };
}
```

### AbortController với Promise

```javascript
const controller = new AbortController();

const fetchPromise = fetch('/api/data', {
  signal: controller.signal,
});

// Hủy sau 3 giây
setTimeout(() => controller.abort(), 3000);

try {
  const res = await fetchPromise;
} catch (err) {
  if (err.name === 'AbortError') {
    console.log('Request cancelled');
  }
}
```

---

## Common Pitfalls

### Pitfall 1: Quên Return Trong .then

```javascript
// SAI — orders là Promise, không phải data
getUser(id).then((user) => {
  getOrders(user.id); // thiếu return!
}).then((orders) => {
  console.log(orders); // undefined
});

// ĐÚNG
getUser(id).then((user) => getOrders(user.id))
  .then((orders) => console.log(orders));
```

### Pitfall 2: Promise trong Loop Không Await

```javascript
// SAI — fire-and-forget, không chờ
ids.forEach((id) => {
  deleteUser(id); // trả về Promise, không await
});
console.log('Done'); // In trước khi xóa xong

// ĐÚNG
await Promise.all(ids.map((id) => deleteUser(id)));
console.log('Done');
```

### Pitfall 3: Wrapping Không Cần Thiết

```javascript
// SAI
async function getUser(id) {
  return new Promise((resolve) => {
    resolve(db.find(id));
  });
}

// ĐÚNG — async function đã return Promise
async function getUser(id) {
  return db.find(id);
}
```

### Pitfall 4: .catch Rồi Im Lặng

```javascript
// NGUY HIỂM — nuốt lỗi
doSomething().catch(() => {});

// ĐÚNG — log hoặc rethrow
doSomething().catch((err) => {
  logger.error(err);
  throw err; // hoặc return default
});
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Ba trạng thái của Promise?

**Gợi ý trả lời:** pending (chờ), fulfilled (thành công — resolve), rejected (thất bại — reject). Một khi settled, state immutable. `.then` nhận fulfilled value, `.catch` nhận rejection reason.

### Câu 2: `Promise.all` vs `Promise.allSettled`?

**Gợi ý trả lời:** `all` fail-fast — một rejection reject cả batch. `allSettled` chờ tất cả, trả array `{status, value/reason}`. Dùng `all` khi cần tất cả thành công; `allSettled` cho independent tasks (ví dụ: gửi notification nhiều channel).

### Câu 3: `Promise.race` vs `Promise.any`?

**Gợi ý trả lời:** `race` — settled đầu tiên (fulfilled hoặc rejected) thắng. `any` — fulfilled đầu tiên thắng; chỉ reject khi tất cả reject. `any` phù hợp redundant sources; `race` phù hợp timeout pattern.

### Câu 4: `.then` trả về gì?

**Gợi ý trả lời:** Trả về Promise mới. Return value → fulfilled với value đó. Return Promise → flatten (chain). Throw → rejected. Cho phép chaining mà không nested.

### Câu 5: Microtask queue và Promise?

**Gợi ý trả lời:** `.then/catch/finally` callbacks đăng ký microtasks. Sau mỗi macrotask (setTimeout, I/O), Event Loop drain hết microtasks trước macrotask tiếp. Đó là lý do Promise.then chạy trước setTimeout(0).

---

**Xem tiếp:** [4-async-await.md](./4-async-await.md) — Cú pháp async/await và execution patterns.
