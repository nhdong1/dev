# Callbacks — Callback Pattern, Callback Hell và Error-First Convention

> Callback (Hàm Gọi Lại) là pattern async đầu tiên trong JavaScript và Node.js. Dù Promises và async/await là chuẩn hiện đại, hiểu callbacks vẫn cần thiết để đọc legacy code và nhiều Node.js APIs.

## Mục Lục

1. [Callback Là Gì?](#callback-là-gì)
2. [Error-First Callback Convention](#error-first-callback-convention)
3. [Callback Hell (Pyramid of Doom)](#callback-hell-pyramid-of-doom)
4. [Đọc Hiểu Node.js Callback APIs](#đọc-hiểu-nodejs-callback-apis)
5. [Promisify — Chuyển Callback Sang Promise](#promisify--chuyển-callback-sang-promise)
6. [Callback vs Promise vs async/await](#callback-vs-promise-vs-asyncawait)
7. [Anti-Patterns và Best Practices](#anti-patterns-và-best-practices)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Callback Là Gì?

Callback là function được truyền làm argument, gọi sau khi async operation hoàn thành.

```javascript
function fetchData(callback) {
  setTimeout(() => {
    const data = { id: 1, name: 'Alice' };
    callback(null, data); // null = không lỗi
  }, 100);
}

fetchData((err, data) => {
  if (err) {
    console.error('Error:', err);
    return;
  }
  console.log('Data:', data);
});
```

| Thuật Ngữ | Giải Thích |
| --------- | ---------- |
| **Higher-Order Function (Hàm Bậc Cao)** | Function nhận hoặc trả về function khác |
| **Continuation (Tiếp Nối)** | Callback tiếp tục logic sau async operation |
| **Inversion of Control (Đảo Ngược Kiểm Soát)** | Thư viện quyết định KHI gọi callback, không phải bạn |

---

## Error-First Callback Convention

Node.js core APIs tuân theo **error-first callback** — tham số đầu tiên luôn là `err`:

```javascript
const fs = require('fs');

fs.readFile('config.json', 'utf8', (err, data) => {
  // err: Error object hoặc null
  // data: kết quả nếu thành công
  if (err) {
    console.error('Read failed:', err.message);
    return;
  }
  console.log(JSON.parse(data));
});
```

### Quy Ước

```javascript
// ĐÚNG — error-first
function doAsync(callback) {
  if (someError) {
    return callback(new Error('Something went wrong'));
  }
  callback(null, result);
}

// SAI — không theo convention Node.js
function doAsyncWrong(callback) {
  callback(result, null); // thứ tự ngược
}
```

### Xử Lý Lỗi Trong Callback Chain

```javascript
function step1(callback) {
  setTimeout(() => callback(null, 'step1 done'), 100);
}

function step2(input, callback) {
  setTimeout(() => callback(null, `${input} → step2 done`), 100);
}

step1((err, result1) => {
  if (err) return console.error(err);
  step2(result1, (err, result2) => {
    if (err) return console.error(err);
    console.log(result2);
  });
});
```

Mỗi level phải check `err` — dễ quên, dễ duplicate code.

---

## Callback Hell (Pyramid of Doom)

Khi nhiều async operations phụ thuộc tuần tự, code lồng sâu thành "kim tự tháp":

```javascript
getUser(userId, (err, user) => {
  if (err) return handleError(err);
  getOrders(user.id, (err, orders) => {
    if (err) return handleError(err);
    getOrderDetails(orders[0].id, (err, details) => {
      if (err) return handleError(err);
      sendEmail(user.email, details, (err) => {
        if (err) return handleError(err);
        console.log('Email sent');
      });
    });
  });
});
```

### Vấn Đề Của Callback Hell

| Vấn Đề | Hậu Quả |
| ------ | ------- |
| **Khó đọc** | Logic nghiệp vụ bị chôn trong nesting |
| **Khó debug** | Stack trace khó theo dõi |
| **Khó test** | Mock nhiều level callback |
| **Error handling lặp** | `if (err)` ở mọi level |
| **Khó parallelize** | Không có primitive cho chạy song song |

### Giải Pháp Trước Promise

```javascript
// 1. Named functions — giảm nesting
function onUserLoaded(err, user) {
  if (err) return handleError(err);
  getOrders(user.id, onOrdersLoaded);
}

function onOrdersLoaded(err, orders) {
  if (err) return handleError(err);
  getOrderDetails(orders[0].id, onDetailsLoaded);
}

getUser(userId, onUserLoaded);

// 2. Module async — tách logic ra file riêng
// 3. Promises / async-await — giải pháp hiện đại (xem file 3, 4)
```

---

## Đọc Hiểu Node.js Callback APIs

Nhiều built-in modules vẫn có callback API (song song với Promise version):

```javascript
const fs = require('fs');
const http = require('http');

// fs — callback
fs.readFile('file.txt', 'utf8', callback);
fs.writeFile('out.txt', data, callback);

// http — callback per request
const server = http.createServer((req, res) => {
  // req, res là streams — event-driven callbacks
  let body = '';
  req.on('data', (chunk) => { body += chunk; });
  req.on('end', () => {
    res.end('OK');
  });
});

// EventEmitter pattern — cũng là callback
const EventEmitter = require('events');
const emitter = new EventEmitter();
emitter.on('user:created', (user) => console.log(user));
```

### Callback Last Argument Rule

```javascript
// Node.js convention: callback LUÔN là argument cuối
fs.readFile(path, options, callback);
// Không phải: fs.readFile(callback, path) ❌
```

---

## Promisify — Chuyển Callback Sang Promise

`util.promisify()` chuyển error-first callback function sang Promise:

```javascript
const fs = require('fs');
const util = require('util');

const readFile = util.promisify(fs.readFile);

// Dùng như Promise
readFile('config.json', 'utf8')
  .then((data) => console.log(JSON.parse(data)))
  .catch((err) => console.error(err));

// Hoặc async/await
async function loadConfig() {
  const data = await readFile('config.json', 'utf8');
  return JSON.parse(data);
}
```

### fs.promises — Built-in Promise API

```javascript
const fs = require('fs/promises');

// Node.js 10+ — không cần promisify
async function loadConfig() {
  const data = await fs.readFile('config.json', 'utf8');
  return JSON.parse(data);
}
```

### Custom Promisify

```javascript
const util = require('util');

function getUserCallback(id, callback) {
  setTimeout(() => {
    if (id <= 0) return callback(new Error('Invalid id'));
    callback(null, { id, name: 'Alice' });
  }, 100);
}

const getUser = util.promisify(getUserCallback);

// Hoặc thêm .promise property (Node.js pattern)
getUserCallback[util.promisify.custom] = (id) => {
  return new Promise((resolve, reject) => {
    getUserCallback(id, (err, user) => {
      if (err) reject(err);
      else resolve(user);
    });
  });
};
```

---

## Callback vs Promise vs async/await

| Tiêu Chí | Callback | Promise | async/await |
| -------- | -------- | ------- | ----------- |
| **Đọc code** | Nested, khó follow | Chain `.then()` | Giống sync code |
| **Error handling** | `if (err)` mỗi level | `.catch()` một chỗ | `try/catch` |
| **Composition** | Khó | `Promise.all`, etc. | `await Promise.all` |
| **Debugging** | Stack khó đọc | Better stack | Best (với async stack traces) |
| **Cancellation** | Manual | AbortController | AbortController |
| **Node.js support** | Mọi version | ES2015+ | ES2017+ |

```javascript
// Cùng logic — 3 styles

// Callback
fs.readFile('a.txt', 'utf8', (err, a) => {
  if (err) return console.error(err);
  fs.readFile('b.txt', 'utf8', (err, b) => {
    if (err) return console.error(err);
    console.log(a + b);
  });
});

// Promise
const readFile = require('fs/promises').readFile;
readFile('a.txt', 'utf8')
  .then((a) => readFile('b.txt', 'utf8').then((b) => a + b))
  .then(console.log)
  .catch(console.error);

// async/await
async function combine() {
  try {
    const a = await readFile('a.txt', 'utf8');
    const b = await readFile('b.txt', 'utf8');
    console.log(a + b);
  } catch (err) {
    console.error(err);
  }
}
```

---

## Anti-Patterns và Best Practices

### Anti-Pattern 1: Callback Gọi Nhiều Lần

```javascript
// NGUY HIỂM
function badAsync(callback) {
  fs.readFile('a.txt', callback);
  fs.readFile('b.txt', callback); // callback có thể gọi 2 lần!
}

// ĐÚNG — đảm bảo callback chỉ gọi đúng một lần
function goodAsync(callback) {
  let called = false;
  const once = (err, data) => {
    if (called) return;
    called = true;
    callback(err, data);
  };
  fs.readFile('a.txt', once);
}
```

### Anti-Pattern 2: Quên Return Sau Error

```javascript
// SAI — tiếp tục chạy sau lỗi
fs.readFile('missing.txt', (err, data) => {
  if (err) console.error(err); // thiếu return!
  console.log(data); // data undefined, có thể crash
});

// ĐÚNG
fs.readFile('missing.txt', (err, data) => {
  if (err) {
    console.error(err);
    return;
  }
  console.log(data);
});
```

### Best Practices

| Practice | Mô Tả |
| -------- | ----- |
| Luôn check `err` trước dùng `data` | Error-first convention |
| `return` sau khi handle error | Tránh fall-through |
| Callback chỉ gọi **một lần** | Tránh double response trong API |
| Ưu tiên Promise/async API | `fs/promises`, không dùng callback mới |
| Wrap legacy callback với promisify | Migration path dần dần |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Error-first callback là gì?

**Gợi ý trả lời:** Convention trong Node.js: tham số đầu tiên của callback là `err` (Error hoặc null), tham số sau là kết quả. Cho phép xử lý lỗi thống nhất. Ví dụ: `fs.readFile(path, (err, data) => {})`.

### Câu 2: Callback hell là gì và cách giải quyết?

**Gợi ý trả lời:** Nested callbacks sâu khi nhiều async operations phụ thuộc nhau. Giải pháp: named functions giảm nesting, Promises với chaining, async/await cho code đọc như sync, hoặc tách thành smaller functions/modules.

### Câu 3: `util.promisify` làm gì?

**Gợi ý trả lời:** Chuyển function theo error-first callback convention thành function trả về Promise. `promisify(fs.readFile)` → `readFile(path)` trả về Promise. Node.js 10+ có `fs.promises` built-in.

### Câu 4: Tại sao callback có thể gọi nhiều lần là vấn đề?

**Gợi ý trả lời:** Trong HTTP handler, gọi `res.send()` hai lần gây lỗi "headers already sent". Logic nghiệp vụ có thể chạy duplicate (double charge, double email). Phải đảm bảo callback/idempotent response chỉ một lần.

### Câu 5: Khi nào vẫn cần dùng callback?

**Gợi ý trả lời:** Legacy code, một số streams/events API (EventEmitter), thư viện chưa có Promise wrapper, hoặc khi cần performance tối đa trong hot path (hiếm). Hầu hết code mới nên dùng async/await.

---

**Xem tiếp:** [3-promises.md](./3-promises.md) — Promise API và combinators.
