# JavaScript ES6+ — Cú Pháp Hiện Đại Cho Backend

> ES6 (ECMAScript 2015 — Chuẩn JavaScript 2015) và các phiên bản sau định nghĩa cú pháp JavaScript hiện đại. Mọi codebase Node.js Backend đều dùng các tính năng này.

## Mục Lục

1. [let, const và Block Scope](#let-const-và-block-scope)
2. [Arrow Functions](#arrow-functions)
3. [Template Literals](#template-literals)
4. [Destructuring](#destructuring)
5. [Spread và Rest](#spread-và-rest)
6. [Default Parameters](#default-parameters)
7. [Object Shorthand](#object-shorthand)
8. [Array Methods Quan Trọng](#array-methods-quan-trọng)
9. [Optional Chaining và Nullish Coalescing](#optional-chaining-và-nullish-coalescing)
10. [Classes](#classes)
11. [Modules (ESM Preview)](#modules-esm-preview)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## let, const và Block Scope

### var — Tránh Dùng

```javascript
// var có function scope, không có block scope
function example() {
  if (true) {
    var x = 10;
  }
  console.log(x); // 10 — x "rò rỉ" ra ngoài block
}
```

### let và const — Block Scope

```javascript
// Block scope — chỉ tồn tại trong { }
if (true) {
  let a = 1;
  const b = 2;
}
// console.log(a); // ReferenceError

// const: không reassign, nhưng object vẫn mutable
const user = { name: 'Alice' };
user.name = 'Bob';   // OK — thay đổi property
// user = {};        // TypeError — không reassign được
```

### Quy Tắc Thực Hành

| Quy Tắc | Lý Do |
| ------- | ----- |
| Dùng `const` mặc định | Tránh reassignment không chủ ý |
| Dùng `let` khi cần reassign | Vòng lặp, accumulator |
| Không dùng `var` | Tránh hoisting bugs |

---

## Arrow Functions

Arrow function (`=>`) là cú pháp viết gọn cho function expression, với điểm khác biệt quan trọng về `this`.

```javascript
// Function declaration
function add(a, b) {
  return a + b;
}

// Arrow function
const add = (a, b) => a + b;

// Multi-line body
const processUser = (user) => {
  const { id, name } = user;
  return { id, displayName: name.toUpperCase() };
};

// Single parameter — không cần ngoặc
const double = x => x * 2;
```

### this Binding — Khác Biệt Quan Trọng

```javascript
// Regular function: this phụ thuộc caller
const obj = {
  name: 'Server',
  start: function() {
    setTimeout(function() {
      console.log(this.name); // undefined — this là global/undefined
    }, 100);
  },
};

// Arrow function: this kế thừa từ enclosing scope (lexical this)
const obj2 = {
  name: 'Server',
  start() {
    setTimeout(() => {
      console.log(this.name); // 'Server'
    }, 100);
  },
};
```

**Trong Node.js Backend:** Arrow function phù hợp cho callbacks, middleware wrappers. Dùng regular function cho class methods hoặc khi cần `arguments` object.

---

## Template Literals

```javascript
const name = 'Alice';
const port = 3000;

// String concatenation cũ
const msg1 = 'Server ' + name + ' running on port ' + port;

// Template literal
const msg2 = `Server ${name} running on port ${port}`;

// Multi-line string
const sql = `
  SELECT id, name, email
  FROM users
  WHERE status = 'active'
`;

// Tagged template (ít dùng trong Backend thông thường)
function sql(strings, ...values) {
  // Sanitize values trước khi interpolate
  return strings.reduce((acc, str, i) => acc + str + (values[i] ?? ''), '');
}
```

---

## Destructuring

### Array Destructuring

```javascript
const [first, second, ...rest] = [1, 2, 3, 4, 5];
// first=1, second=2, rest=[3,4,5]

// Swap variables
let a = 1, b = 2;
[a, b] = [b, a];

// Default values
const [x = 0, y = 0] = [42];
```

### Object Destructuring

```javascript
const user = { id: 1, name: 'Alice', email: 'alice@example.com', role: 'admin' };

// Basic
const { name, email } = user;

// Rename
const { name: userName } = user;

// Default value
const { phone = 'N/A' } = user;

// Nested
const config = { db: { host: 'localhost', port: 5432 } };
const { db: { host, port } } = config;
```

### Destructuring Trong Function Parameters

```javascript
// Pattern phổ biến trong Express handlers và service layer
function createUser({ name, email, role = 'user' }) {
  return { id: Date.now(), name, email, role };
}

// Express route handler
app.get('/users/:id', (req, res) => {
  const { id } = req.params;
  const { page = 1, limit = 20 } = req.query;
  // ...
});
```

---

## Spread và Rest

### Spread Operator (`...`) — Mở Rộng

```javascript
// Clone array
const original = [1, 2, 3];
const copy = [...original];

// Merge arrays
const merged = [...arr1, ...arr2];

// Clone object (shallow copy — sao chép nông)
const defaults = { timeout: 5000, retries: 3 };
const config = { ...defaults, timeout: 10000, host: 'localhost' };
// { timeout: 10000, retries: 3, host: 'localhost' }

// Spread trong function call
Math.max(...numbers);
```

### Rest Operator (`...`) — Thu Gom

```javascript
// Rest parameters
function sum(...numbers) {
  return numbers.reduce((acc, n) => acc + n, 0);
}
sum(1, 2, 3, 4); // 10

// Rest trong destructuring
const { id, ...userWithoutId } = user;
```

---

## Default Parameters

```javascript
function connect(host = 'localhost', port = 5432, options = {}) {
  const { ssl = false, timeout = 5000 } = options;
  return { host, port, ssl, timeout };
}

connect();                          // defaults
connect('db.example.com');          // custom host
connect('db.example.com', 3306, { ssl: true });
```

---

## Object Shorthand

```javascript
const name = 'Alice';
const role = 'admin';

// ES5
const user1 = { name: name, role: role };

// ES6 shorthand
const user2 = { name, role };

// Method shorthand
const service = {
  async getUser(id) {
    return await db.findById(id);
  },
};
```

---

## Array Methods Quan Trọng

Các method này xuất hiện liên tục trong Backend code — xử lý dữ liệu từ database, transform API response.

```javascript
const users = [
  { id: 1, name: 'Alice', active: true },
  { id: 2, name: 'Bob', active: false },
  { id: 3, name: 'Carol', active: true },
];

// map — transform mỗi phần tử
const names = users.map(u => u.name);
// ['Alice', 'Bob', 'Carol']

// filter — lọc theo điều kiện
const activeUsers = users.filter(u => u.active);

// find — tìm phần tử đầu tiên
const bob = users.find(u => u.name === 'Bob');

// findIndex — tìm index
const bobIndex = users.findIndex(u => u.name === 'Bob');

// some / every — kiểm tra điều kiện
const hasInactive = users.some(u => !u.active);  // true
const allActive = users.every(u => u.active);     // false

// reduce — aggregate
const userMap = users.reduce((acc, u) => {
  acc[u.id] = u;
  return acc;
}, {});

// flatMap — map + flatten một cấp
const tags = users.flatMap(u => u.tags ?? []);

// includes — kiểm tra tồn tại (array of primitives)
[1, 2, 3].includes(2); // true
```

### Chaining Pattern

```javascript
const result = users
  .filter(u => u.active)
  .map(u => ({ id: u.id, name: u.name.toUpperCase() }))
  .sort((a, b) => a.name.localeCompare(b.name));
```

---

## Optional Chaining và Nullish Coalescing

### Optional Chaining (`?.`)

```javascript
const user = { profile: { address: { city: 'Hanoi' } } };

// Tránh TypeError khi property không tồn tại
const city = user?.profile?.address?.city;     // 'Hanoi'
const zip = user?.profile?.address?.zip;       // undefined (không throw)

// Optional call
const result = maybeFn?.();

// Optional array access
const first = arr?.[0];
```

### Nullish Coalescing (`??`)

```javascript
// || trả về right operand khi left là falsy (0, '', false)
const port1 = config.port || 3000;  // Bug nếu port = 0

// ?? chỉ khi left là null hoặc undefined
const port2 = config.port ?? 3000;  // 0 được giữ nguyên

const name = user.nickname ?? user.name ?? 'Anonymous';
```

---

## Classes

```javascript
class ApiError extends Error {
  constructor(message, statusCode, code) {
    super(message);
    this.name = 'ApiError';
    this.statusCode = statusCode;
    this.code = code;
  }

  toJSON() {
    return {
      error: { message: this.message, code: this.code },
    };
  }
}

class UserService {
  #db; // Private field (ES2022)

  constructor(db) {
    this.#db = db;
  }

  async findById(id) {
    const user = await this.#db.query('SELECT * FROM users WHERE id = $1', [id]);
    if (!user) throw new ApiError('User not found', 404, 'USER_NOT_FOUND');
    return user;
  }

  static fromConfig(config) {
    return new UserService(createPool(config));
  }
}
```

**Lưu ý:** Classes trong JavaScript là syntactic sugar trên prototype. Trong nhiều dự án Node.js, functional style (factory functions) vẫn phổ biến.

---

## Modules (ESM Preview)

```javascript
// export named
export const VERSION = '1.0.0';
export function helper() { /* ... */ }

// export default
export default class App { /* ... */ }

// import
import App, { VERSION, helper } from './app.js';
```

Chi tiết CommonJS vs ESM xem tại [3-module-systems.md](./3-module-systems.md).

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Closure là gì? Cho ví dụ trong Node.js

**Gợi ý trả lời:** Closure là khả năng function "nhớ" lexical scope nơi nó được tạo, kể cả khi scope đó đã kết thúc. Ví dụ: factory function tạo middleware với config đóng gói, hoặc module pattern dùng IIFE trước ES modules.

```javascript
function createRateLimiter(maxRequests) {
  let count = 0;
  return function(req, res, next) {
    if (++count > maxRequests) return res.status(429).send('Too Many Requests');
    next();
  };
}
```

### Câu 2: Shallow copy vs Deep copy?

**Gợi ý trả lời:** Spread operator (`{...obj}`) và `Object.assign()` tạo shallow copy — nested object vẫn share reference. Deep copy cần `structuredClone()` (Node.js 17+) hoặc thư viện như lodash `cloneDeep`.

### Câu 3: `map` vs `forEach` — khi nào dùng gì?

**Gợi ý trả lời:** `map` trả về array mới (dùng khi transform data). `forEach` không return value, chỉ side effects (logging, mutate external state). Prefer `map`/`filter`/`reduce` cho functional pipeline; `for...of` khi cần `break`/`continue` hoặc `await` trong loop.

### Câu 4: `==` vs `===`?

**Gợi ý trả lời:** `===` (strict equality) so sánh không ép kiểu. `==` ép kiểu trước khi so sánh (`0 == ''` → true). Trong Backend code, **luôn dùng `===`** để tránh bugs.

---

**Xem tiếp:** [2-nodejs-runtime.md](./2-nodejs-runtime.md) — hiểu cách Node.js thực thi JavaScript.
