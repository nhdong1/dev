# OWASP Top 10 — Phòng Chống Lỗ Hổng Bảo Mật Web

> **OWASP (Open Web Application Security Project)** công bố Top 10 lỗ hổng bảo mật web phổ biến nhất. Mọi Backend Node.js developer cần biết cách prevent từng loại trong production API.

## Mục Lục

1. [OWASP Top 10 (2021) Tổng Quan](#owasp-top-10-2021-tổng-quan)
2. [A01 — Broken Access Control](#a01--broken-access-control)
3. [A02 — Cryptographic Failures](#a02--cryptographic-failures)
4. [A03 — Injection](#a03--injection)
5. [A04 — Insecure Design](#a04--insecure-design)
6. [A05 — Security Misconfiguration](#a05--security-misconfiguration)
7. [A06 — Vulnerable Components](#a06--vulnerable-components)
8. [A07 — Authentication Failures](#a07--authentication-failures)
9. [A08 — Data Integrity Failures](#a08--data-integrity-failures)
10. [A09 — Security Logging Failures](#a09--security-logging-failures)
11. [A10 — SSRF](#a10--ssrf)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## OWASP Top 10 (2021) Tổng Quan

| # | Tên | Mô Tả Ngắn | Node.js Focus |
| - | --- | ---------- | ------------- |
| A01 | Broken Access Control | Truy cập trái phép | RBAC, IDOR checks |
| A02 | Cryptographic Failures | Mã hóa yếu/thiếu | bcrypt, TLS, AES |
| A03 | Injection | SQL, NoSQL, Command | Parameterized queries |
| A04 | Insecure Design | Thiết kế thiếu security | Threat modeling |
| A05 | Security Misconfiguration | Config sai | Helmet, CORS, env |
| A06 | Vulnerable Components | Dependencies lỗi thời | `npm audit` |
| A07 | Authentication Failures | Auth yếu | MFA, rate limit login |
| A08 | Data Integrity Failures | Deserialization, CI/CD | Verify signatures |
| A09 | Security Logging Failures | Log thiếu | Structured security logs |
| A10 | SSRF | Server request forgery | URL whitelist |

---

## A01 — Broken Access Control

**Attack:** Thay đổi URL/parameter truy cập resource không được phép.

```
GET /api/users/123/orders  → User 123 OK
GET /api/users/456/orders  → User 123 cố truy cập user 456
```

**Prevention trong Node.js:**

```typescript
// Luôn authorize resource ownership
app.get('/api/users/:userId/orders', authenticate, async (req: AuthRequest, res) => {
  const { userId } = req.params;

  if (req.user!.sub !== userId && !req.user!.roles.includes('admin')) {
    return res.status(403).json({ error: 'Forbidden' });
  }

  const orders = await db.order.findMany({ where: { userId } });
  res.json(orders);
});
```

- Deny by default
- Server-side authorization — không tin client
- Rate limiting và log access failures

---

## A02 — Cryptographic Failures

**Attack:** Data nhạy cảm exposed do encryption yếu hoặc thiếu.

| Sai | Đúng |
| --- | ---- |
| Plain text password | bcrypt/Argon2 hash |
| MD5/SHA1 cho password | bcrypt cost ≥ 12 |
| HTTP transmission | HTTPS TLS 1.2+ |
| Hardcoded encryption key | KMS/Vault key management |

```typescript
import bcrypt from 'bcryptjs';

// Password hashing
const hash = await bcrypt.hash(password, 12);

// Sensitive data at rest — Node.js crypto
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

function encrypt(plaintext: string, key: Buffer): { iv: string; data: string } {
  const iv = randomBytes(12);
  const cipher = createCipheriv('aes-256-gcm', key, iv);
  const encrypted = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
  const tag = cipher.getAuthTag();
  return {
    iv: iv.toString('hex'),
    data: Buffer.concat([encrypted, tag]).toString('hex'),
  };
}
```

---

## A03 — Injection

### SQL Injection

```typescript
// ❌ VULNERABLE
const query = `SELECT * FROM users WHERE email = '${email}'`;
// Input: ' OR '1'='1' --

// ✅ SAFE — parameterized query
const result = await pool.query(
  'SELECT * FROM users WHERE email = $1',
  [email]
);

// ✅ SAFE — Prisma
const user = await prisma.user.findUnique({ where: { email } });
```

### NoSQL Injection (MongoDB)

```typescript
// ❌ VULNERABLE — user gửi { "email": { "$gt": "" } }
const user = await User.findOne(req.body);

// ✅ SAFE — validate schema trước
const schema = z.object({ email: z.string().email() });
const { email } = schema.parse(req.body);
const user = await User.findOne({ email });

// ✅ SAFE — express-mongo-sanitize
import mongoSanitize from 'express-mongo-sanitize';
app.use(mongoSanitize());
```

### Command Injection

```typescript
// ❌ VULNERABLE
import { exec } from 'child_process';
exec(`convert ${userFilename} output.png`);

// ✅ SAFE — dùng library, validate input, spawn với args array
import { spawn } from 'child_process';
spawn('convert', [sanitizedFilename, 'output.png']);
```

---

## A04 — Insecure Design

**Vấn đề:** Thiếu security requirements từ đầu thiết kế.

| Ví Dụ Thiết Kế Yếu | Thiết Kế An Toàn |
| ------------------ | ---------------- |
| Không rate limit password reset | Rate limit + CAPTCHA + email notification |
| Sequential user IDs | UUID v4 |
| Unlimited pagination `?limit=999999` | Max limit 100, default 20 |
| Public debug endpoint | Disabled trong production |

**Threat Modeling (Mô Hình Hóa Mối Đe Dọa):** STRIDE framework — Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege.

---

## A05 — Security Misconfiguration

**Ví dụ phổ biến trong Node.js:**

```typescript
// ❌ Expose stack trace
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message, stack: err.stack });
});

// ✅ Production error handler
app.use((err, req, res, next) => {
  logger.error({ err, requestId: req.id });
  res.status(500).json({
    error: 'Internal server error',
    requestId: req.id,
  });
});

// ❌ CORS wildcard với credentials
app.use(cors({ origin: '*', credentials: true }));

// ✅ CORS whitelist
app.use(cors({
  origin: ['https://app.example.com'],
  credentials: true,
}));
```

**Checklist:**
- Tắt `X-Powered-By: Express`
- Helmet security headers
- Không expose `.env`, `node_modules` qua static files
- Disable GraphQL introspection trong production

---

## A06 — Vulnerable Components

```bash
# Scan dependencies
npm audit
npm audit fix

# CI pipeline
npx audit-ci --moderate
```

| Tool | Mô Tả |
| ---- | ----- |
| **npm audit** | Built-in vulnerability scan |
| **Dependabot** | GitHub auto PR cho updates |
| **Snyk** | Continuous monitoring |
| **Socket.dev** | Detect malicious packages |

**Best practices:**
- Pin dependencies (`package-lock.json` committed)
- Regular updates (monthly security patch)
- Remove unused dependencies
- Review trước khi `npm install` package mới

---

## A07 — Authentication Failures

```typescript
// Rate limit login
import rateLimit from 'express-rate-limit';

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: { error: 'Too many login attempts' },
  standardHeaders: true,
});

app.post('/auth/login', loginLimiter, loginHandler);

// Strong password policy với Zod
const passwordSchema = z.string()
  .min(12)
  .regex(/[A-Z]/, 'Need uppercase')
  .regex(/[0-9]/, 'Need number')
  .regex(/[^A-Za-z0-9]/, 'Need special char');

// Account lockout sau N failed attempts
// MFA (Multi-Factor Authentication — Xác Thực Đa Yếu Tố) cho sensitive apps
```

**Session security:**
- Regenerate session ID sau login (chống session fixation)
- Secure + HttpOnly + SameSite cookies
- Invalidate all sessions khi đổi password

---

## A08 — Data Integrity Failures

**Deserialization attacks:**

```typescript
// ❌ NEVER — deserialize untrusted data
import { unserialize } from 'node-serialize'; // Known RCE vulnerability pattern
const obj = unserialize(userInput);

// ✅ Use JSON.parse với schema validation
const data = JSON.parse(input);
const validated = schema.parse(data);
```

**CI/CD integrity:**
- Sign container images
- Verify package checksums
- Lock file integrity trong CI

---

## A09 — Security Logging Failures

```typescript
import pino from 'pino';

const logger = pino();

// Log security events
logger.warn({
  event: 'AUTH_FAILURE',
  email: req.body.email,
  ip: req.ip,
  userAgent: req.headers['user-agent'],
  requestId: req.id,
});

logger.warn({
  event: 'ACCESS_DENIED',
  userId: req.user?.sub,
  resource: req.path,
  action: req.method,
});

// KHÔNG log: passwords, tokens, credit cards, PII không cần thiết
```

**Cần log:** Login success/failure, permission denied, validation failures, rate limit hits, suspicious patterns.

**Không log:** Full JWT, passwords, API secrets, full credit card numbers.

---

## A10 — SSRF

**SSRF (Server-Side Request Forgery — Giả Mạo Yêu Cầu Phía Server):** Attacker buộc server gọi internal URLs.

```
POST /api/fetch-url
{ "url": "http://169.254.169.254/latest/meta-data/" }  → AWS metadata leak
{ "url": "http://localhost:6379/" }                     → Redis access
```

**Prevention:**

```typescript
import { URL } from 'url';

const ALLOWED_HOSTS = ['api.trusted.com', 'cdn.example.com'];

function validateUrl(input: string): URL {
  const url = new URL(input);

  if (!['http:', 'https:'].includes(url.protocol)) {
    throw new Error('Invalid protocol');
  }

  // Block private IPs
  const hostname = url.hostname;
  if (
    hostname === 'localhost' ||
    hostname.startsWith('127.') ||
    hostname.startsWith('10.') ||
    hostname.startsWith('192.168.') ||
    hostname === '169.254.169.254'
  ) {
    throw new Error('Blocked host');
  }

  if (!ALLOWED_HOSTS.includes(hostname)) {
    throw new Error('Host not in whitelist');
  }

  return url;
}
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: OWASP Top 3 quan trọng nhất với Node.js API?

**Trả lời:** A01 Broken Access Control (IDOR, missing RBAC), A03 Injection (SQL/NoSQL), A07 Authentication Failures (weak auth, no rate limit). Tùy app có thể thêm A10 SSRF nếu có URL fetch feature.

### Câu 2: SQL Injection prevent trong Prisma?

**Trả lời:** Prisma Client dùng prepared statements — an toàn mặc định. Cẩn thận với `$queryRaw` — phải dùng tagged template `` prisma.$queryRaw`SELECT * FROM users WHERE id = ${id}` `` không concatenate string.

### Câu 3: XSS liên quan Backend Node.js thế nào?

**Trả lời:** XSS chủ yếu frontend, nhưng API có thể amplify — store unsanitized HTML, reflect user input trong error messages. Backend nên validate/sanitize input, set CSP headers qua Helmet, encode output nếu render HTML.

### Câu 4: CSRF là gì? API JWT có cần CSRF protection?

**Trả lời:** CSRF (Cross-Site Request Forgery) — site khác gửi request dùng cookie session của user. JWT trong Authorization header không tự động gửi — ít risk hơn. Nếu dùng cookie-based auth, cần CSRF token hoặc SameSite=Strict.

---

**Xem tiếp:** [5-helmet-rate-limiting.md](./5-helmet-rate-limiting.md) — Security headers và rate limiting.
