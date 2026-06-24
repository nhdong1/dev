# Error Handling — Xử Lý Lỗi Bất Đồng Bộ Trong Node.js

> Lỗi trong async code dễ bị "nuốt" (swallowed) hoặc gây **unhandled rejection (từ chối không xử lý)** — dẫn đến crash production hoặc silent data corruption. Chủ đề này cover chiến lược xử lý lỗi từ local đến global.

## Mục Lục

1. [Đặc Thù Lỗi Async](#đặc-thù-lỗi-async)
2. [try/catch với async/await](#trycatch-với-asyncawait)
3. [Promise Error Propagation](#promise-error-propagation)
4. [Unhandled Rejection](#unhandled-rejection)
5. [Process-Level Error Handlers](#process-level-error-handlers)
6. [Express/Fastify Error Handling](#expressfastify-error-handling)
7. [Custom Error Classes](#custom-error-classes)
8. [Production Error Strategy](#production-error-strategy)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Đặc Thù Lỗi Async

### Sync vs Async Errors

```javascript
// Sync — try/catch bắt được ngay
try {
  JSON.parse('invalid');
} catch (err) {
  console.error(err);
}

// Async callback — try/catch KHÔNG bắt được
try {
  setTimeout(() => {
    throw new Error('async throw'); // uncaughtException
  }, 100);
} catch (err) {
  // Không chạy — error xảy ra sau khi try/catch đã xong
}

// Promise — try/catch chỉ bắt được với await
try {
  await Promise.reject(new Error('fail'));
} catch (err) {
  console.error(err); // OK
}
```

| Loại Lỗi | Cách Bắt |
| -------- | -------- |
| Sync throw | try/catch |
| Callback error | Check `err` argument |
| Promise reject | `.catch()` hoặc try/catch + await |
| EventEmitter error | `.on('error', handler)` |
| Unhandled | `unhandledRejection`, `uncaughtException` |

---

## try/catch với async/await

```javascript
async function transferFunds(fromId, toId, amount) {
  try {
    const from = await getAccount(fromId);
    if (from.balance < amount) {
      throw new InsufficientFundsError(fromId, amount);
    }
    await debit(fromId, amount);
    await credit(toId, amount);
    return { success: true };
  } catch (err) {
    if (err instanceof InsufficientFundsError) {
      return { success: false, code: 'INSUFFICIENT_FUNDS' };
    }
    // Unexpected — log và rethrow
    logger.error({ err, fromId, toId }, 'Transfer failed');
    throw err;
  }
}
```

### try/catch Scope

```javascript
// SAI — catch quá rộng, nuốt mọi lỗi
try {
  const user = await getUser(id);
  const orders = await getOrders(user.id);
  await sendEmail(user.email);
} catch (err) {
  return null; // mất context lỗi từ bước nào
}

// TỐT HƠN — handle cụ thể, rethrow unexpected
try {
  const user = await getUser(id);
} catch (err) {
  if (err.code === 'NOT_FOUND') return null;
  throw err;
}
```

---

## Promise Error Propagation

```javascript
// Lỗi "bubble up" qua chain
doStep1()
  .then(() => doStep2())
  .then(() => doStep3())
  .catch((err) => {
    // Bắt lỗi từ bất kỳ step nào
    logger.error(err);
    throw err; // rethrow cho caller
  });

// async equivalent
async function pipeline() {
  await doStep1();
  await doStep2();
  await doStep3();
}
```

### AggregateError — Promise.any

```javascript
try {
  await Promise.any([
    fetch('https://down1.example.com'),
    fetch('https://down2.example.com'),
  ]);
} catch (err) {
  if (err instanceof AggregateError) {
    console.log('All failed:', err.errors); // array of errors
  }
}
```

### finally — Cleanup Luôn Chạy

```javascript
let connection;
try {
  connection = await pool.connect();
  await connection.query('BEGIN');
  await connection.query('INSERT ...');
  await connection.query('COMMIT');
} catch (err) {
  await connection?.query('ROLLBACK');
  throw err;
} finally {
  connection?.release(); // luôn release về pool
}
```

---

## Unhandled Rejection

**Unhandled Promise Rejection** xảy ra khi Promise reject mà không có `.catch()` hoặc `await` trong try/catch.

```javascript
// Gây unhandled rejection
async function leaky() {
  throw new Error('no handler');
}
leaky(); // quên .catch()

// Hoặc
Promise.reject(new Error('orphan'));
```

### Node.js Behavior Theo Version

| Version | Behavior |
| ------- | -------- |
| Node.js < 15 | Warning, process tiếp tục |
| Node.js 15+ | Terminate process (mặc định) nếu không handle |
| Với handler | Log và quyết định có exit hay không |

```javascript
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise);
  console.error('Reason:', reason);

  // Production: log structured, alert, graceful shutdown
  logger.fatal({ reason, promise }, 'Unhandled rejection');

  // Không tiếp tục trong trạng thái undefined
  process.exit(1);
});
```

### Phòng Tránh

```javascript
// 1. Luôn await hoặc return Promise trong async handler
app.get('/users', async (req, res, next) => {
  try {
    const users = await getUsers();
    res.json(users);
  } catch (err) {
    next(err);
  }
});

// 2. .catch() trên mọi floating promise
backgroundJob().catch((err) => logger.error(err));

// 3. void operator — explicit fire-and-forget (cẩn thận)
void backgroundJob().catch(handleError);
```

---

## Process-Level Error Handlers

### uncaughtException

```javascript
process.on('uncaughtException', (err, origin) => {
  logger.fatal({ err, origin }, 'Uncaught exception');

  // Best practice: graceful shutdown
  // KHÔNG tiếp tục chạy — app state có thể corrupt
  shutdown('uncaughtException');
});

function shutdown(signal) {
  server.close(() => {
    logger.info('Server closed');
    process.exit(1);
  });

  // Force exit sau timeout
  setTimeout(() => process.exit(1), 10000).unref();
}
```

### uncaughtException vs unhandledRejection

| Event | Khi Nào | Khuyến Nghị |
| ----- | ------- | ----------- |
| `uncaughtException` | Sync throw không bắt | Log + shutdown |
| `unhandledRejection` | Promise reject không handle | Log + shutdown (production) |
| `warning` | Deprecation, memory leak hints | Monitor |

### rejectionHandled — Late Handler

```javascript
process.on('rejectionHandled', (promise) => {
  // Promise từng unhandled nhưng sau đó có .catch()
  logger.warn('Late rejection handler attached');
});
```

---

## Express/Fastify Error Handling

### Express — Centralized Error Middleware

```javascript
// Error middleware — PHẢI có 4 arguments
app.use((err, req, res, next) => {
  const statusCode = err.statusCode || 500;
  const message = statusCode === 500 ? 'Internal Server Error' : err.message;

  logger.error({
    err,
    requestId: req.id,
    path: req.path,
  });

  res.status(statusCode).json({
    error: {
      code: err.code || 'INTERNAL_ERROR',
      message,
    },
  });
});

// Async route wrapper
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

app.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await getUser(req.params.id);
  if (!user) throw new NotFoundError('User');
  res.json(user);
}));
```

### Fastify — Built-in async Support

```javascript
fastify.get('/users/:id', async (request, reply) => {
  const user = await getUser(request.params.id);
  if (!user) {
    throw fastify.httpErrors.notFound('User not found');
  }
  return user; // Fastify tự catch async errors
});

fastify.setErrorHandler((err, request, reply) => {
  const statusCode = err.statusCode || 500;
  reply.status(statusCode).send({
    error: err.message,
  });
});
```

---

## Custom Error Classes

```javascript
class AppError extends Error {
  constructor(message, { statusCode = 500, code = 'INTERNAL_ERROR', isOperational = true } = {}) {
    super(message);
    this.name = this.constructor.name;
    this.statusCode = statusCode;
    this.code = code;
    this.isOperational = isOperational; // expected vs programmer error
    Error.captureStackTrace(this, this.constructor);
  }
}

class NotFoundError extends AppError {
  constructor(resource) {
    super(`${resource} not found`, { statusCode: 404, code: 'NOT_FOUND' });
  }
}

class ValidationError extends AppError {
  constructor(details) {
    super('Validation failed', { statusCode: 400, code: 'VALIDATION_ERROR' });
    this.details = details;
  }
}

// Phân biệt operational vs programming errors
function isOperationalError(err) {
  if (err instanceof AppError) return err.isOperational;
  return false;
}

process.on('uncaughtException', (err) => {
  if (!isOperationalError(err)) {
    // Programming bug — shutdown
    shutdown('fatal');
  }
});
```

### Error Cause Chain (ES2022)

```javascript
try {
  await db.query('SELECT ...');
} catch (err) {
  throw new AppError('Database query failed', {
    cause: err, // giữ original error
  });
}

console.log(err.cause); // original DB error
```

---

## Production Error Strategy

### Checklist

| Item | Mô Tả |
| ---- | ----- |
| **Operational errors** | Map sang HTTP status, log warn |
| **Programming errors** | Log fatal, alert, shutdown |
| **Never expose stack** | Production response không có stack trace |
| **Structured logging** | `{ err, requestId, userId }` — Pino/Winston |
| **Global handlers** | `unhandledRejection`, `uncaughtException` |
| **Graceful shutdown** | Đóng server, drain connections, exit |
| **Health check** | Liveness probe phát hiện hung process |

### Result Pattern (Alternative)

```javascript
// Thay throw — explicit error handling
async function safeParse(json) {
  try {
    return { ok: true, data: JSON.parse(json) };
  } catch (err) {
    return { ok: false, error: err };
  }
}

const result = await safeParse(input);
if (!result.ok) {
  return res.status(400).json({ error: 'Invalid JSON' });
}
```

### Domain (Deprecated — Không Dùng Mới)

Node.js `domain` module đã deprecated. Dùng async context (AsyncLocalStorage) và proper error handling thay thế.

```javascript
const { AsyncLocalStorage } = require('async_hooks');
const asyncLocalStorage = new AsyncLocalStorage();

// Gắn requestId vào context, accessible trong mọi async call
app.use((req, res, next) => {
  asyncLocalStorage.run({ requestId: req.id }, () => next());
});
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: try/catch có bắt lỗi trong setTimeout không?

**Gợi ý trả lời:** Không. Callback chạy sau khi try/catch đã hoàn thành. Lỗi trong setTimeout → `uncaughtException` nếu không handle trong callback. Cần try/catch bên trong callback hoặc dùng Promise.

### Câu 2: unhandledRejection vs uncaughtException?

**Gợi ý trả lời:** `unhandledRejection` — Promise reject không có handler. `uncaughtException` — sync throw không bắt. Cả hai production nên log và graceful shutdown vì app state có thể inconsistent.

### Câu 3: Xử lý lỗi trong Express async route?

**Gợi ý trả lời:** Express 4 không tự catch async errors. Dùng try/catch + next(err), hoặc async wrapper `fn(req,res,next).catch(next)`, hoặc Express 5 (native support). Error middleware 4 args ở cuối chain.

### Câu 4: Operational error vs programming error?

**Gợi ý trả lời:** Operational — expected runtime failures (404, validation, network timeout). Programming — bugs (null reference, logic error). Operational: handle gracefully. Programming: log, alert, có thể shutdown.

### Câu 5: Có nên dùng `.catch(() => {})` không?

**Gợi ý trả lời:** Không trong production — nuốt lỗi, khó debug, silent failures. Nếu intentionally ignore, log lý do. Prefer explicit handling hoặc rethrow sau log.

---

**Xem tiếp:** [6-concurrency-patterns.md](./6-concurrency-patterns.md) — Throttle, debounce, queue patterns.
