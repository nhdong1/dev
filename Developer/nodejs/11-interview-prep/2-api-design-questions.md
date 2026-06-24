# Câu Hỏi API Design & Web Frameworks

> Bộ câu hỏi phỏng vấn về REST API design, Express.js middleware, error handling, validation, và so sánh frameworks — chủ đề cốt lõi cho Backend Node.js interview.

## Mục Lục

1. [REST API Fundamentals](#rest-api-fundamentals)
2. [Express.js & Middleware](#expressjs--middleware)
3. [Error Handling & Validation](#error-handling--validation)
4. [Framework Comparison](#framework-comparison)
5. [Live Coding Challenges](#live-coding-challenges)

---

## REST API Fundamentals

### Q1: REST (Representational State Transfer) là gì? Nguyên tắc cốt lõi?

**Đáp án:**

REST là **architectural style** (phong cách kiến trúc) cho distributed systems, dựa trên:

| Nguyên Tắc | Mô Tả |
| ---------- | ----- |
| **Stateless** (Phi Trạng Thái) | Mỗi request chứa đủ thông tin, server không lưu session state |
| **Resource-based** | Mọi thứ là resource với URI duy nhất (`/users/123`) |
| **HTTP Methods** | GET (read), POST (create), PUT (replace), PATCH (update), DELETE |
| **Representation** | JSON, XML — client và server thỏa thuận format |
| **HATEOAS** (Hypermedia) | Response chứa links đến related resources (optional) |

---

### Q2: Thiết kế REST API cho hệ thống e-commerce — endpoints chính?

```
GET    /products              — Danh sách sản phẩm (pagination, filter)
GET    /products/:id          — Chi tiết sản phẩm
POST   /products              — Tạo sản phẩm (admin)
PATCH  /products/:id          — Cập nhật một phần
DELETE /products/:id          — Xóa sản phẩm

POST   /orders                — Tạo đơn hàng
GET    /orders/:id            — Chi tiết đơn hàng
GET    /users/:id/orders      — Đơn hàng của user (nested resource)

POST   /cart/items            — Thêm vào giỏ
DELETE /cart/items/:id        — Xóa khỏi giỏ
```

**Điểm cộng:** Đề cập pagination (`?page=1&limit=20`), filtering (`?category=electronics`), versioning (`/v1/products`).

---

### Q3: HTTP Status Codes — khi nào dùng gì?

| Code | Ý Nghĩa | Use Case |
| ---- | ------- | -------- |
| **200** OK | Thành công | GET, PATCH thành công |
| **201** Created | Tạo mới thành công | POST tạo resource |
| **204** No Content | Thành công, không body | DELETE thành công |
| **400** Bad Request | Client gửi data sai | Validation fail |
| **401** Unauthorized | Chưa authenticate | Thiếu/invalid token |
| **403** Forbidden | Đã auth nhưng không có quyền | RBAC deny |
| **404** Not Found | Resource không tồn tại | ID không có trong DB |
| **409** Conflict | Xung đột state | Duplicate email |
| **422** Unprocessable Entity | Syntax đúng, semantic sai | Business rule violation |
| **429** Too Many Requests | Rate limit exceeded | Throttling |
| **500** Internal Server Error | Lỗi server | Bug, DB down |

---

### Q4: API Versioning — strategies và trade-offs?

| Strategy | Ví Dụ | Ưu | Nhược |
| -------- | ----- | -- | ----- |
| **URL path** | `/v1/users` | Rõ ràng, dễ route | URL thay đổi |
| **Header** | `Accept: application/vnd.api.v2+json` | URL sạch | Khó test, cache phức tạp |
| **Query param** | `/users?version=2` | Linh hoạt | Không RESTful lắm |

**Best practice:** URL versioning cho public API; header cho internal services.

---

### Q5: Idempotency (Tính Bất Biến) — method nào idempotent?

| Method | Idempotent? | Giải Thích |
| ------ | ----------- | ---------- |
| GET | ✅ | Đọc không thay đổi state |
| PUT | ✅ | Ghi đè cùng data → cùng kết quả |
| DELETE | ✅ | Xóa 2 lần → resource vẫn không tồn tại |
| POST | ❌ | Mỗi lần gọi tạo resource mới |
| PATCH | ❌ (thường) | Phụ thuộc implementation |

**Điểm cộng:** Đề cập **Idempotency Key** header cho POST trong payment systems.

---

## Express.js & Middleware

### Q6: Middleware trong Express hoạt động thế nào?

Middleware là function `(req, res, next) => {}` chạy theo **thứ tự đăng ký**:

```javascript
app.use(express.json());           // 1. Parse JSON body
app.use(requestLogger);            // 2. Log request
app.use('/api', authMiddleware);   // 3. Auth cho /api/*
app.use('/api/users', userRouter); // 4. Route handlers
app.use(errorHandler);             // 5. Error handler (4 params)
```

**Flow:**
1. Request đến → middleware 1 → gọi `next()` → middleware 2 → ...
2. Route handler gửi response → kết thúc
3. Nếu error → skip đến error handler (`err, req, res, next`)

---

### Q7: Thứ tự middleware quan trọng thế nào?

```javascript
// ❌ SAI — error handler trước routes
app.use(errorHandler);
app.use('/api', router);

// ✅ ĐÚNG
app.use(helmet());
app.use(cors());
app.use(express.json({ limit: '10mb' }));
app.use('/api', router);
app.use(notFoundHandler);  // 404
app.use(errorHandler);     // Cuối cùng
```

**Quy tắc:** Security headers → body parser → logging → auth → routes → 404 → error handler.

---

### Q8: Viết auth middleware kiểm tra JWT?

```javascript
const jwt = require('jsonwebtoken');

function authMiddleware(req, res, next) {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Missing token' });
  }

  const token = authHeader.slice(7);
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET);
    req.user = payload; // { userId, role }
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid token' });
  }
}
```

---

### Q9: `app.use` vs `app.get` vs `Router()` — khác gì?

| | `app.use` | `app.get/post` | `Router()` |
| - | --------- | -------------- | ---------- |
| **Match** | Mọi HTTP method | Method cụ thể | Modular routes |
| **Path** | Prefix matching | Exact path | Mountable sub-app |
| **Use case** | Middleware, static | Route handlers | Tổ chức routes theo module |

```javascript
const userRouter = express.Router();
userRouter.get('/', listUsers);
userRouter.post('/', createUser);
app.use('/api/users', authMiddleware, userRouter);
```

---

### Q10: Làm sao handle async errors trong Express?

```javascript
// Wrapper function — pattern phổ biến
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

// Dùng:
app.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await userService.findById(req.params.id);
  if (!user) return res.status(404).json({ error: 'Not found' });
  res.json(user);
}));

// Hoặc: express-async-errors package
```

---

## Error Handling & Validation

### Q11: Error handling strategy cho production API?

```javascript
// Custom error classes
class AppError extends Error {
  constructor(message, statusCode, code) {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
    this.isOperational = true;
  }
}

class NotFoundError extends AppError {
  constructor(resource) {
    super(`${resource} not found`, 404, 'NOT_FOUND');
  }
}

// Global error handler
function errorHandler(err, req, res, next) {
  const statusCode = err.statusCode || 500;
  const response = {
    error: {
      message: err.isOperational ? err.message : 'Internal server error',
      code: err.code || 'INTERNAL_ERROR',
      ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
    },
  };
  logger.error({ err, reqId: req.id }, 'Request failed');
  res.status(statusCode).json(response);
}
```

**Nguyên tắc:**
- Operational errors (400, 404) → trả message rõ ràng
- Programming errors (500) → log chi tiết, trả generic message
- Không leak stack trace trong production

---

### Q12: Request validation với Zod — ví dụ?

```javascript
const { z } = require('zod');

const createUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(100),
  name: z.string().min(1).max(50),
});

function validate(schema) {
  return (req, res, next) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json({
        error: 'Validation failed',
        details: result.error.flatten().fieldErrors,
      });
    }
    req.body = result.data; // Stripped unknown fields
    next();
  };
}

app.post('/users', validate(createUserSchema), createUser);
```

---

### Q13: Pagination — cursor vs offset?

| | Offset (`?page=2&limit=20`) | Cursor (`?cursor=abc&limit=20`) |
| - | --------------------------- | ------------------------------- |
| **Implementation** | `OFFSET 20 LIMIT 20` | `WHERE id > cursor LIMIT 20` |
| **Performance** | Chậm với offset lớn | Ổn định với index |
| **Consistency** | Có thể duplicate/skip khi data thay đổi | Consistent hơn |
| **Use case** | Admin UI, ít data | Feed, infinite scroll, large dataset |

---

### Q14: Rate limiting — implement thế nào?

```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 phút
  max: 100,                  // 100 requests/window
  standardHeaders: true,     // RateLimit-* headers
  keyGenerator: (req) => req.user?.id || req.ip,
  handler: (req, res) => {
    res.status(429).json({ error: 'Too many requests' });
  },
});

app.use('/api/', limiter);
```

**Production:** Dùng Redis store cho distributed rate limiting across multiple instances.

---

### Q15: CORS (Cross-Origin Resource Sharing) — giải thích và config?

```javascript
const cors = require('cors');

app.use(cors({
  origin: ['https://myapp.com', 'https://admin.myapp.com'],
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true, // Cho phép cookies
  maxAge: 86400,     // Preflight cache 24h
}));
```

**Giải thích:** Browser gửi preflight OPTIONS request trước cross-origin request. Server phải trả đúng `Access-Control-*` headers.

---

## Framework Comparison

### Q16: Express vs Fastify vs NestJS?

| | Express | Fastify | NestJS |
| - | ------- | ------- | ------ |
| **Performance** | Baseline | 2–3x nhanh hơn | Tương đương Express |
| **Learning curve** | Thấp | Thấp–trung bình | Cao (Angular-like) |
| **Architecture** | Unopinionated | Plugin-based | Opinionated (modules, DI) |
| **TypeScript** | Cần setup | First-class | Native |
| **Ecosystem** | Lớn nhất | Đang phát triển | Enterprise-focused |
| **Phù hợp** | MVP, startup | High-throughput API | Enterprise, team lớn |

---

### Q17: Khi nào chọn Fastify thay Express?

- **High-throughput** API (>10k RPS)
- Cần **schema validation** tích hợp (JSON Schema)
- **TypeScript-first** project
- Microservices cần **low latency**

Khi nào giữ Express: team quen, ecosystem lớn, MVP nhanh, nhiều middleware có sẵn.

---

### Q18: NestJS Dependency Injection — tại sao hữu ích?

```typescript
@Injectable()
export class UserService {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly cache: CacheService,
  ) {}
}

@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}
}
```

**Lợi ích:** Testability (mock dependencies), loose coupling, lifecycle management, consistent architecture cho team lớn.

---

## Live Coding Challenges

### Challenge 1: CRUD API đơn giản

```javascript
// Yêu cầu: Implement GET /todos và POST /todos
const todos = [];
let nextId = 1;

app.get('/todos', (req, res) => {
  res.json(todos);
});

app.post('/todos', (req, res) => {
  const { title } = req.body;
  if (!title?.trim()) {
    return res.status(400).json({ error: 'Title required' });
  }
  const todo = { id: nextId++, title, completed: false };
  todos.push(todo);
  res.status(201).json(todo);
});
```

### Challenge 2: Middleware đo response time

```javascript
function responseTime(req, res, next) {
  const start = process.hrtime.bigint();
  res.on('finish', () => {
    const duration = Number(process.hrtime.bigint() - start) / 1e6;
    logger.info({ method: req.method, url: req.url, durationMs: duration });
  });
  next();
}
```

### Challenge 3: Implement API key authentication

```javascript
function apiKeyAuth(req, res, next) {
  const apiKey = req.headers['x-api-key'];
  if (!apiKey || !validApiKeys.has(apiKey)) {
    return res.status(401).json({ error: 'Invalid API key' });
  }
  req.clientId = apiKeyToClient[apiKey];
  next();
}
```

---

## Tài Liệu Tham Khảo

- [03-web-frameworks/1-express-basics.md](../03-web-frameworks/1-express-basics.md)
- [03-web-frameworks/5-rest-api-design.md](../03-web-frameworks/5-rest-api-design.md)
- [03-web-frameworks/6-request-validation.md](../03-web-frameworks/6-request-validation.md)
- [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) — Câu 11–20
