# Helmet, Rate Limiting và CORS — Security Headers

> **Helmet.js** set HTTP security headers tự động; **rate limiting** bảo vệ API khỏi brute-force và DoS; **CORS (Cross-Origin Resource Sharing — Chia Sẻ Tài Nguyên Đa Nguồn)** kiểm soát cross-origin requests.

## Mục Lục

1. [HTTP Security Headers](#http-security-headers)
2. [Helmet.js Setup](#helmetjs-setup)
3. [CORS Configuration](#cors-configuration)
4. [Rate Limiting Algorithms](#rate-limiting-algorithms)
5. [express-rate-limit](#express-rate-limit)
6. [Distributed Rate Limiting với Redis](#distributed-rate-limiting-với-redis)
7. [HTTPS và HSTS](#https-và-hsts)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## HTTP Security Headers

| Header | Mục Đích |
| ------ | -------- |
| `Content-Security-Policy` | Chống XSS — whitelist script/style sources |
| `Strict-Transport-Security` | Force HTTPS (HSTS) |
| `X-Content-Type-Options: nosniff` | Chống MIME sniffing |
| `X-Frame-Options: DENY` | Chống clickjacking |
| `Referrer-Policy` | Kiểm soát Referer header leak |
| `Permissions-Policy` | Disable browser features (camera, geolocation) |

---

## Helmet.js Setup

```bash
npm install helmet
```

```typescript
import express from 'express';
import helmet from 'helmet';

const app = express();

// Default — bật tất cả headers an toàn
app.use(helmet());

// Custom CSP (Content Security Policy — Chính Sách Bảo Mật Nội Dung)
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", 'data:', 'https:'],
      connectSrc: ["'self'", 'https://api.example.com'],
      frameSrc: ["'none'"],
      objectSrc: ["'none'"],
    },
  },
  hsts: {
    maxAge: 31536000,        // 1 năm
    includeSubDomains: true,
    preload: true,
  },
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
}));

// Tắt X-Powered-By (Helmet làm mặc định)
app.disable('x-powered-by');
```

### API-Only Backend (JSON)

```typescript
// API không serve HTML — CSP đơn giản hơn
app.use(helmet({
  contentSecurityPolicy: false, // Hoặc defaultSrc: ["'none'"]
  crossOriginEmbedderPolicy: false, // Tránh conflict với CORS
}));
```

---

## CORS Configuration

CORS là **browser security mechanism** — server phải explicitly allow cross-origin requests.

```bash
npm install cors
```

```typescript
import cors from 'cors';

// ❌ NGUY HIỂM cho authenticated API
app.use(cors({ origin: '*', credentials: true }));

// ✅ Whitelist origins
const allowedOrigins = [
  'https://app.example.com',
  'https://admin.example.com',
];

app.use(cors({
  origin(origin, callback) {
    // Allow requests không có origin (mobile apps, Postman, server-to-server)
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
  maxAge: 86400, // Preflight cache 24h
}));
```

### Preflight Requests

Browser gửi `OPTIONS` request trước "non-simple" requests (custom headers, PUT/DELETE):

```
OPTIONS /api/users
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization
```

Server phải respond với matching `Access-Control-Allow-*` headers.

---

## Rate Limiting Algorithms

| Algorithm | Mô Tả | Ưu/Nhược |
| --------- | ----- | -------- |
| **Fixed Window** | N requests per time window | Đơn giản; burst ở boundary |
| **Sliding Window** | Rolling window chính xác hơn | Phức tạp hơn, memory cao hơn |
| **Token Bucket** | Tokens refill theo rate | Cho phép burst có kiểm soát |
| **Leaky Bucket** | Queue requests, process steady rate | Smooth traffic |

```
Fixed Window (100 req/min):
|---- minute 1 ----|---- minute 2 ----|
  100 requests OK     100 requests OK
  Request 101 at 0:59 → OK
  Request 101 at 1:00 → OK (new window) ← burst 200 in 2 seconds!

Sliding Window: chính xác hơn, không có boundary burst
```

---

## express-rate-limit

```bash
npm install express-rate-limit
```

### Global Rate Limit

```typescript
import rateLimit from 'express-rate-limit';

const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 phút
  max: 1000,                    // 1000 requests per window per IP
  standardHeaders: true,        // RateLimit-* headers (draft-6)
  legacyHeaders: false,         // Tắt X-RateLimit-* cũ
  message: {
    error: 'Too many requests',
    retryAfter: '15 minutes',
  },
  skip: (req) => req.ip === '127.0.0.1', // Skip health checks
});

app.use(globalLimiter);
```

### Endpoint-Specific Limits

```typescript
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  skipSuccessfulRequests: true, // Chỉ count failed logins
});

const registerLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,
  max: 3,
});

app.post('/auth/login', loginLimiter, loginHandler);
app.post('/auth/register', registerLimiter, registerHandler);

// Expensive endpoints
const exportLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,
  max: 10,
  keyGenerator: (req) => req.user?.sub || req.ip, // Per user, not just IP
});

app.get('/api/reports/export', authenticate, exportLimiter, exportHandler);
```

### Response Headers

```
RateLimit-Limit: 100
RateLimit-Remaining: 42
RateLimit-Reset: 1706000000
```

---

## Distributed Rate Limiting với Redis

Single-instance `express-rate-limit` dùng in-memory — không work với multiple Node.js instances. Dùng Redis store:

```bash
npm install rate-limit-redis ioredis
```

```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

const limiter = rateLimit({
  store: new RedisStore({
    sendCommand: (...args: string[]) => redis.call(...args),
  }),
  windowMs: 60 * 1000,
  max: 100,
});

app.use(limiter);
```

### Manual Redis Counter (Token Bucket đơn giản)

```typescript
async function checkRateLimit(key: string, limit: number, windowSec: number): Promise<boolean> {
  const count = await redis.incr(key);
  if (count === 1) {
    await redis.expire(key, windowSec);
  }
  return count <= limit;
}

// Usage trong middleware
const allowed = await checkRateLimit(`rate:${req.ip}`, 100, 60);
if (!allowed) return res.status(429).json({ error: 'Rate limit exceeded' });
```

---

## HTTPS và HSTS

```typescript
// Redirect HTTP → HTTPS (thường ở reverse proxy — Nginx, ALB)
app.use((req, res, next) => {
  if (req.headers['x-forwarded-proto'] !== 'https' && process.env.NODE_ENV === 'production') {
    return res.redirect(301, `https://${req.hostname}${req.url}`);
  }
  next();
});
```

**HSTS (HTTP Strict Transport Security — Bảo Mật Truyền Tải Nghiêm Ngặt):**

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

Browser sẽ **luôn** dùng HTTPS — kể cả user gõ `http://`. Chỉ enable khi chắc chắn HTTPS hoạt động ổn định.

---

## Best Practices

1. **Helmet trên mọi Express app** — default config đã tốt
2. **CORS whitelist** — không `*` với credentials
3. **Rate limit `/login`, `/register`, `/password-reset`** — chống brute-force
4. **Per-user rate limit** cho authenticated endpoints — `keyGenerator: req => req.user.sub`
5. **429 Too Many Requests** — consistent error format
6. **Redis store** khi chạy multiple instances
7. **HSTS** chỉ sau khi HTTPS stable
8. **Không rate limit health checks** — `/health`, `/ready`

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Helmet làm gì?

**Trả lời:** Middleware set HTTP security headers — CSP, HSTS, X-Frame-Options, X-Content-Type-Options, v.v. Giảm XSS, clickjacking, MIME sniffing risks.

### Câu 2: Fixed window vs sliding window rate limit?

**Trả lời:** Fixed window đơn giản nhưng cho phép 2x burst ở boundary giữa hai windows. Sliding window chính xác hơn, tốn memory/compute hơn. Production thường dùng sliding window hoặc token bucket.

### Câu 3: CORS `origin: *` có an toàn không?

**Trả lời:** OK cho public read-only API không cần credentials. **Không an toàn** với `credentials: true` — browser block combination này. Authenticated API cần explicit origin whitelist.

### Câu 4: Rate limit theo IP có đủ không?

**Trả lời:** IP limit chống basic DoS/brute-force. Shared NAT/proxy có thể false positive. Authenticated endpoints nên limit theo `userId`. Kết hợp IP + user + API key.

---

**Xem tiếp:** [6-input-validation.md](./6-input-validation.md) — Input validation và sanitization.
