# Express.js Basics — Middleware, Routing và Request/Response

> Express.js là web framework minimal và unopinionated (không áp đặt kiến trúc) cho Node.js. Nắm vững middleware chain và routing là nền tảng cho mọi Node.js Backend interview.

## Mục Lục

1. [Express Là Gì](#express-là-gì)
2. [Khởi Tạo Application](#khởi-tạo-application)
3. [Middleware Chain (Chuỗi Phần Mềm Trung Gian)](#middleware-chain-chuỗi-phần-mềm-trung-gian)
4. [Routing (Định Tuyến)](#routing-định-tuyến)
5. [Request và Response Objects](#request-và-response-objects)
6. [Router Module](#router-module)
7. [Static Files và Body Parsing](#static-files-và-body-parsing)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Express Là Gì

Express.js xây dựng trên module `http` của Node.js, cung cấp:

| Tính Năng | Mô Tả |
| --------- | ----- |
| **Routing** | Map HTTP method + path → handler function |
| **Middleware** | Pipeline xử lý request trước khi đến route handler |
| **Template engines** | Pug, EJS (ít dùng trong pure API) |
| **Static file serving** | Serve files từ thư mục `public/` |

```bash
npm install express
```

---

## Khởi Tạo Application

```javascript
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

// Middleware và routes đăng ký ở đây

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

**ESM (ECMAScript Modules — Module Chuẩn JavaScript):**

```javascript
import express from 'express';

const app = express();
export default app;
```

**TypeScript:**

```typescript
import express, { Request, Response, NextFunction } from 'express';

const app = express();
```

---

## Middleware Chain (Chuỗi Phần Mềm Trung Gian)

Middleware là function có access đến `req`, `res`, và `next`. Chạy theo thứ tự đăng ký.

```javascript
// Application-level middleware
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next(); // BẮT BUỘC gọi next() để chuyển tiếp
});

// Route-level middleware — chỉ áp dụng cho route cụ thể
app.get('/protected', authMiddleware, (req, res) => {
  res.json({ message: 'Authenticated' });
});
```

### Luồng Middleware

```
Request ──► Middleware 1 ──► Middleware 2 ──► Route Handler ──► Response
                │                │
                │ next()         │ next()
                ▼                ▼
           (nếu không gọi next() → request bị "treo")
```

### Built-in Middleware

```javascript
const express = require('express');
const app = express();

// Parse JSON body — Content-Type: application/json
app.use(express.json({ limit: '10mb' }));

// Parse URL-encoded form — Content-Type: application/x-www-form-urlencoded
app.use(express.urlencoded({ extended: true }));

// Serve static files từ thư mục 'public'
app.use(express.static('public'));
```

### Custom Middleware Ví Dụ

```javascript
// Request ID cho distributed tracing
const { randomUUID } = require('crypto');

function requestIdMiddleware(req, res, next) {
  req.id = req.headers['x-request-id'] || randomUUID();
  res.setHeader('X-Request-Id', req.id);
  next();
}

// Timing middleware
function timingMiddleware(req, res, next) {
  const start = Date.now();
  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`${req.method} ${req.url} ${res.statusCode} ${duration}ms`);
  });
  next();
}

app.use(requestIdMiddleware);
app.use(timingMiddleware);
```

---

## Routing (Định Tuyến)

### HTTP Methods (Phương Thức HTTP)

| Method | Mục Đích | Idempotent (Lặp Lại An Toàn) |
| ------ | -------- | --------------------------- |
| **GET** | Đọc resource | Có |
| **POST** | Tạo resource mới | Không |
| **PUT** | Thay thế toàn bộ resource | Có |
| **PATCH** | Cập nhật một phần | Không |
| **DELETE** | Xóa resource | Có |

```javascript
// GET /users
app.get('/users', (req, res) => {
  res.json({ users: [] });
});

// GET /users/:id — route parameter
app.get('/users/:id', (req, res) => {
  const { id } = req.params;
  res.json({ id });
});

// POST /users
app.post('/users', (req, res) => {
  const user = req.body;
  res.status(201).json({ user });
});

// PUT /users/:id
app.put('/users/:id', (req, res) => {
  res.json({ id: req.params.id, ...req.body });
});

// DELETE /users/:id
app.delete('/users/:id', (req, res) => {
  res.status(204).send();
});
```

### Route Parameters và Query String

```javascript
// Route params: /users/123
app.get('/users/:id', (req, res) => {
  console.log(req.params.id); // '123'
});

// Query string: /users?page=1&limit=10
app.get('/users', (req, res) => {
  const { page = 1, limit = 10 } = req.query;
  res.json({ page, limit });
});

// Optional params với regex
app.get('/files/:filename(*)', (req, res) => {
  res.json({ filename: req.params.filename });
});
```

### Route Matching Order

Express match routes **theo thứ tự đăng ký**. Route cụ thể phải đăng ký trước route generic:

```javascript
// ĐÚNG — /users/me trước /users/:id
app.get('/users/me', (req, res) => res.json({ current: true }));
app.get('/users/:id', (req, res) => res.json({ id: req.params.id }));

// SAI — /users/:id sẽ match 'me' như một id
```

---

## Request và Response Objects

### Request (`req`) — Thuộc Tính Quan Trọng

| Property | Mô Tả |
| -------- | ----- |
| `req.params` | Route parameters (`:id`) |
| `req.query` | Query string parsed |
| `req.body` | Request body (sau `express.json()`) |
| `req.headers` | HTTP headers |
| `req.method` | GET, POST, PUT... |
| `req.path` | URL path không có query string |
| `req.ip` | Client IP address |
| `req.cookies` | Cookies (cần `cookie-parser`) |

### Response (`res`) — Methods Quan Trọng

```javascript
// JSON response — set Content-Type: application/json
res.json({ data: 'value' });

// Status code
res.status(404).json({ error: 'Not found' });
res.status(201).json({ created: true });

// Send plain text hoặc buffer
res.send('Hello');
res.send(Buffer.from('binary'));

// Redirect
res.redirect(301, '/new-url');

// Set headers
res.set('X-Custom-Header', 'value');
res.set({ 'Cache-Control': 'no-cache', 'ETag': 'abc123' });

// Download file
res.download('/path/to/file.pdf');

// End response không có body
res.status(204).end();
```

### Async Route Handlers

```javascript
// Express 5+ tự động catch rejected promises
// Express 4 — cần wrapper hoặc .catch()

app.get('/users/:id', async (req, res, next) => {
  try {
    const user = await userService.findById(req.params.id);
    if (!user) return res.status(404).json({ error: 'User not found' });
    res.json(user);
  } catch (err) {
    next(err); // Chuyển sang error handler
  }
});

// Wrapper pattern (Express 4)
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

app.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await userService.findById(req.params.id);
  res.json(user);
}));
```

---

## Router Module

`express.Router()` tạo mini-application cho modular routes:

```javascript
// routes/users.js
const express = require('express');
const router = express.Router();

router.get('/', (req, res) => res.json({ users: [] }));
router.get('/:id', (req, res) => res.json({ id: req.params.id }));
router.post('/', (req, res) => res.status(201).json(req.body));

module.exports = router;

// app.js
const userRoutes = require('./routes/users');
app.use('/api/users', userRoutes);
// → GET /api/users, GET /api/users/:id, POST /api/users
```

**Router-level middleware:**

```javascript
const router = express.Router();

// Áp dụng cho mọi route trong router này
router.use(authMiddleware);

router.get('/profile', (req, res) => {
  res.json({ user: req.user });
});
```

---

## Static Files và Body Parsing

```javascript
// Multiple static directories
app.use('/static', express.static('public'));
app.use('/uploads', express.static('uploads'));

// Conditional body parser
app.use('/webhooks/stripe', express.raw({ type: 'application/json' })); // Stripe cần raw body
app.use(express.json()); // Các route khác
```

**Giới hạn body size** — bảo vệ khỏi DoS (Denial of Service — Từ Chối Dịch Vụ):

```javascript
app.use(express.json({ limit: '100kb' }));
```

---

## Best Practices

| Practice | Lý Do |
| -------- | ----- |
| Tách routes ra files riêng | Maintainability (khả năng bảo trì) |
| Dùng `asyncHandler` wrapper | Tránh unhandled promise rejection |
| Đặt error handler cuối cùng | Catch mọi lỗi từ routes/middleware |
| Không gửi response 2 lần | Gây `ERR_HTTP_HEADERS_SENT` |
| Validate input sớm | Trước khi vào business logic |
| Dùng `helmet` middleware | Security headers (xem `05-security/`) |

**Cấu trúc project khuyến nghị:**

```
src/
├── app.js              # Express app setup
├── server.js           # app.listen()
├── routes/
│   ├── index.js
│   └── users.routes.js
├── controllers/
│   └── users.controller.js
├── services/
│   └── users.service.js
├── middleware/
│   ├── auth.middleware.js
│   └── error.middleware.js
└── utils/
    └── asyncHandler.js
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: `app.use()` vs `app.METHOD()` khác nhau thế nào?

**Trả lời:** `app.use()` mount middleware cho mọi HTTP method trên path (hoặc mọi path nếu không chỉ định). `app.get()`, `app.post()`... chỉ match method và path cụ thể. `app.use('/api', router)` mount router tại prefix `/api`.

### Câu 2: Điều gì xảy ra nếu không gọi `next()`?

**Trả lời:** Request bị "treo" — client chờ timeout. Middleware phải gọi `next()`, gửi response (`res.send/json`), hoặc gọi `next(err)` để báo lỗi. Không được làm cả ba.

### Câu 3: Express 4 vs Express 5 — khác biệt chính?

**Trả lời:** Express 5 tự động forward rejected promises từ async handlers sang error middleware. Hỗ trợ `req.query` với prototype null (an toàn hơn). Một số deprecated methods bị remove (`app.del` → `app.delete`). Path route matching cải thiện.

### Câu 4: `res.send()` vs `res.json()`?

**Trả lời:** `res.json()` serialize object thành JSON và set `Content-Type: application/json`. `res.send()` tự detect type (string, buffer, object) — object cũng được JSON.stringify nhưng `res.json()` rõ ràng hơn cho API.

---

**Xem tiếp:** [2-express-advanced.md](./2-express-advanced.md) — Error handling, validation, file upload.
