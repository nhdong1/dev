# Express.js Advanced — Error Handling, Validation và File Upload

> Production-ready Express API cần centralized error handling (xử lý lỗi tập trung), input validation, file upload an toàn, và graceful shutdown (tắt máy chủ an toàn).

## Mục Lục

1. [Centralized Error Handling](#centralized-error-handling)
2. [Custom Error Classes](#custom-error-classes)
3. [Async Error Wrapper](#async-error-wrapper)
4. [404 Not Found Handler](#404-not-found-handler)
5. [Request Validation với Middleware](#request-validation-với-middleware)
6. [File Upload với Multer](#file-upload-với-multer)
7. [CORS (Cross-Origin Resource Sharing — Chia Sẻ Tài Nguyên Khác Nguồn)](#cors-cross-origin-resource-sharing--chia-sẻ-tài-nguyên-khác-nguồn)
8. [Graceful Shutdown](#graceful-shutdown)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Centralized Error Handling

Error-handling middleware có **4 tham số** — Express nhận diện qua signature `(err, req, res, next)`:

```javascript
// ĐẶT CUỐI CÙNG — sau tất cả routes
app.use((err, req, res, next) => {
  const statusCode = err.statusCode || 500;
  const message = err.isOperational ? err.message : 'Internal Server Error';

  // Log đầy đủ cho server, không gửi ra client
  console.error({
    requestId: req.id,
    error: err.message,
    stack: err.stack,
    path: req.path,
    method: req.method,
  });

  res.status(statusCode).json({
    error: {
      code: err.code || 'INTERNAL_ERROR',
      message,
      ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
    },
  });
});
```

### Operational vs Programming Errors

| Loại | Mô Tả | Xử Lý |
| ---- | ----- | ----- |
| **Operational** | Lỗi dự kiến: validation fail, not found, unauthorized | Trả message cho client |
| **Programming** | Bug: null reference, type error | Log, alert, có thể restart process |

---

## Custom Error Classes

```javascript
class AppError extends Error {
  constructor(message, statusCode, code = 'APP_ERROR') {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
    this.isOperational = true;
    Error.captureStackTrace(this, this.constructor);
  }
}

class NotFoundError extends AppError {
  constructor(resource = 'Resource') {
    super(`${resource} not found`, 404, 'NOT_FOUND');
  }
}

class ValidationError extends AppError {
  constructor(details) {
    super('Validation failed', 400, 'VALIDATION_ERROR');
    this.details = details;
  }
}

class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(message, 401, 'UNAUTHORIZED');
  }
}

// Sử dụng trong service
async function getUser(id) {
  const user = await db.users.findById(id);
  if (!user) throw new NotFoundError('User');
  return user;
}
```

---

## Async Error Wrapper

```javascript
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Hoặc dùng express-async-errors package
require('express-async-errors'); // Patch Express 4 tự động

app.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await userService.findById(req.params.id);
  res.json(user);
}));
```

---

## 404 Not Found Handler

```javascript
// Sau tất cả routes, TRƯỚC error handler
app.use((req, res, next) => {
  next(new NotFoundError(`Route ${req.method} ${req.path}`));
});

// Error handler (4-arg) đặt sau 404 handler
app.use(errorHandler);
```

---

## Request Validation với Middleware

```javascript
const { z } = require('zod');

const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(2).max(100),
  age: z.number().int().positive().optional(),
});

function validate(schema) {
  return (req, res, next) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return next(new ValidationError(
        result.error.flatten().fieldErrors
      ));
    }
    req.validated = result.data; // Dùng data đã sanitize
    next();
  };
}

app.post('/users', validate(createUserSchema), asyncHandler(async (req, res) => {
  const user = await userService.create(req.validated);
  res.status(201).json(user);
}));
```

**Validate params và query:**

```javascript
const idParamSchema = z.object({
  id: z.string().uuid(),
});

function validateParams(schema) {
  return (req, res, next) => {
    const result = schema.safeParse(req.params);
    if (!result.success) {
      return next(new ValidationError(result.error.flatten().fieldErrors));
    }
    req.params = result.data;
    next();
  };
}
```

---

## File Upload với Multer

[Multer](https://github.com/expressjs/multer) — middleware xử lý `multipart/form-data`:

```bash
npm install multer
```

```javascript
const multer = require('multer');
const path = require('path');

// Disk storage
const storage = multer.diskStorage({
  destination: (req, file, cb) => cb(null, 'uploads/'),
  filename: (req, file, cb) => {
    const unique = `${Date.now()}-${Math.random().toString(36).slice(2)}`;
    cb(null, `${unique}${path.extname(file.originalname)}`);
  },
});

// File filter — whitelist MIME types
const fileFilter = (req, file, cb) => {
  const allowed = ['image/jpeg', 'image/png', 'image/webp'];
  if (allowed.includes(file.mimetype)) {
    cb(null, true);
  } else {
    cb(new AppError('Invalid file type', 400, 'INVALID_FILE'), false);
  }
};

const upload = multer({
  storage,
  fileFilter,
  limits: {
    fileSize: 5 * 1024 * 1024, // 5MB
    files: 1,
  },
});

// Single file
app.post('/avatar', upload.single('avatar'), (req, res) => {
  res.json({ filename: req.file.filename });
});

// Multiple files
app.post('/gallery', upload.array('photos', 5), (req, res) => {
  res.json({ files: req.files.map(f => f.filename) });
});
```

**Security checklist cho upload:**

| Rule | Lý Do |
| ---- | ----- |
| Validate MIME type + extension | Tránh upload executable |
| Giới hạn file size | Tránh DoS |
| Không dùng `originalname` trực tiếp | Path traversal attack |
| Store ngoài web root hoặc dùng S3 | Tránh execute uploaded file |
| Scan virus trong production | Malware protection |

---

## CORS (Cross-Origin Resource Sharing — Chia Sẻ Tài Nguyên Khác Nguồn)

Browser chặn request từ domain khác trừ khi server cho phép:

```bash
npm install cors
```

```javascript
const cors = require('cors');

// Development — cho phép mọi origin
app.use(cors());

// Production — whitelist origins
app.use(cors({
  origin: ['https://app.example.com', 'https://admin.example.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true, // Cho phép cookies
  maxAge: 86400, // Preflight cache 24h
}));
```

**Preflight request:** Browser gửi `OPTIONS` trước POST/PUT với custom headers. Server phải respond đúng CORS headers.

---

## Graceful Shutdown

Đóng server an toàn khi nhận SIGTERM (từ K8s, PM2):

```javascript
const server = app.listen(PORT);

function gracefulShutdown(signal) {
  console.log(`${signal} received, shutting down gracefully`);

  server.close(() => {
    console.log('HTTP server closed');
    // Đóng DB connections, Redis, etc.
    db.disconnect().then(() => {
      process.exit(0);
    });
  });

  // Force shutdown sau 30s
  setTimeout(() => {
    console.error('Forced shutdown after timeout');
    process.exit(1);
  }, 30000);
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Tại sao error middleware phải có 4 tham số?

**Trả lời:** Express phân biệt error handler vs regular middleware qua arity (số tham số). 3 tham số `(req, res, next)` là regular middleware; 4 tham số `(err, req, res, next)` là error handler. Nếu chỉ có 3 tham số, Express coi là regular middleware và không catch errors.

### Câu 2: Làm sao tránh leak stack trace trong production?

**Trả lời:** Dùng `isOperational` flag trên custom errors. Error handler chỉ trả `err.message` cho operational errors. Programming errors trả generic message. Chỉ include `stack` khi `NODE_ENV === 'development'`. Log full stack server-side.

### Câu 3: Multer memory vs disk storage?

**Trả lời:** Memory storage (`multer.memoryStorage()`) giữ file trong RAM — phù hợp upload nhỏ, stream lên S3. Disk storage ghi file local — phù hợp file lớn, cần persist tạm. Memory có risk OOM (Out of Memory) với file lớn.

---

**Xem tiếp:** [5-rest-api-design.md](./5-rest-api-design.md) hoặc [6-request-validation.md](./6-request-validation.md)
