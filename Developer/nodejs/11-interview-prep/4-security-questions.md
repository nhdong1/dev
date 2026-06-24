# Câu Hỏi Security — JWT, OWASP & Best Practices

> Bộ câu hỏi phỏng vấn về authentication (xác thực), authorization (phân quyền), OWASP Top 10, rate limiting, và security hardening cho Node.js API.

## Mục Lục

1. [Authentication & JWT](#authentication--jwt)
2. [Authorization & RBAC](#authorization--rbac)
3. [OWASP Top 10](#owasp-top-10)
4. [Security Headers & Rate Limiting](#security-headers--rate-limiting)
5. [Secrets & Production Security](#secrets--production-security)
6. [Live Coding Challenges](#live-coding-challenges)

---

## Authentication & JWT

### Q1: JWT (JSON Web Token) hoạt động như thế nào?

JWT gồm 3 phần Base64-encoded, ngăn cách bởi dấu chấm:

```
header.payload.signature
```

| Phần | Nội Dung |
| ---- | -------- |
| **Header** | Algorithm (HS256, RS256), type (JWT) |
| **Payload** | Claims: `sub` (userId), `iat`, `exp`, custom data |
| **Signature** | `HMACSHA256(header + payload, secret)` — verify integrity |

```javascript
const token = jwt.sign(
  { userId: 123, role: 'admin' },
  process.env.JWT_SECRET,
  { expiresIn: '15m' }
);

const payload = jwt.verify(token, process.env.JWT_SECRET);
```

**Quan trọng:** JWT **không mã hóa** payload — chỉ encode Base64. Không lưu sensitive data (password, credit card).

---

### Q2: Access Token vs Refresh Token — flow đầy đủ?

```
1. Login (email + password)
   → Server verify credentials
   → Trả access token (15 phút) + refresh token (7 ngày)

2. API Request
   → Header: Authorization: Bearer <access_token>
   → Server verify JWT

3. Access token expired
   → POST /auth/refresh { refreshToken }
   → Server verify refresh token (check DB blacklist/whitelist)
   → Trả access token mới (+ refresh token mới nếu rotation)

4. Logout
   → Invalidate refresh token trong DB/Redis
```

**Refresh token rotation:** Mỗi lần refresh, cấp refresh token mới, invalidate token cũ — phát hiện token theft.

---

### Q3: JWT vs Session-based Auth — trade-offs?

| | JWT (Stateless) | Session (Stateful) |
| - | --------------- | ------------------ |
| **Storage** | Client (localStorage/cookie) | Server (Redis/DB) |
| **Scalability** | Dễ scale — không cần shared session store | Cần Redis cho multi-instance |
| **Revocation** | Khó — token valid đến khi expire | Dễ — xóa session |
| **Size** | Lớn hơn (trong mỗi request) | Chỉ session ID |
| **Phù hợp** | Microservices, SPA, mobile | Traditional web app, cần instant revoke |

**Best practice:** Short-lived access token + refresh token với server-side storage.

---

### Q4: Lưu JWT ở đâu — localStorage vs cookie?

| | localStorage | HttpOnly Cookie |
| - | ------------ | --------------- |
| **XSS risk** | Cao — JS có thể đọc | Thấp — JS không access được |
| **CSRF risk** | Không | Có — cần CSRF token |
| **Mobile app** | Phù hợp | Khó implement |
| **Khuyến nghị** | SPA với XSS protection tốt | Web app truyền thống |

```javascript
// HttpOnly cookie
res.cookie('accessToken', token, {
  httpOnly: true,
  secure: true,      // HTTPS only
  sameSite: 'strict', // CSRF protection
  maxAge: 15 * 60 * 1000,
});
```

---

### Q5: Password hashing — bcrypt vs argon2?

```javascript
const bcrypt = require('bcrypt');
const SALT_ROUNDS = 12;

// Hash khi register
const hash = await bcrypt.hash(password, SALT_ROUNDS);

// Verify khi login
const isValid = await bcrypt.compare(password, hash);
```

| | bcrypt | argon2 |
| - | ------ | ------ |
| **Algorithm** | Blowfish-based | Memory-hard |
| **Resistance** | GPU attacks (fair) | GPU/ASIC attacks (better) |
| **Adoption** | Rất phổ biến | OWASP recommended (2024+) |
| **Node.js** | `bcrypt` package | `argon2` package |

**Không bao giờ:** MD5, SHA1, plain text, reversible encryption cho password.

---

## Authorization & RBAC

### Q6: Authentication vs Authorization?

| | Authentication | Authorization |
| - | -------------- | --------------- |
| **Câu hỏi** | "Bạn là ai?" | "Bạn được phép làm gì?" |
| **Ví dụ** | Login, JWT verify | RBAC, permission check |
| **HTTP code** | 401 Unauthorized | 403 Forbidden |
| **Thứ tự** | Trước | Sau authentication |

---

### Q7: RBAC (Role-Based Access Control) — implement thế nào?

```javascript
const permissions = {
  admin: ['users:read', 'users:write', 'users:delete', 'orders:*'],
  editor: ['users:read', 'orders:read', 'orders:write'],
  viewer: ['users:read', 'orders:read'],
};

function authorize(...requiredPermissions) {
  return (req, res, next) => {
    const userPerms = permissions[req.user.role] || [];
    const hasPermission = requiredPermissions.every(p =>
      userPerms.includes(p) || userPerms.includes(p.split(':')[0] + ':*')
    );
    if (!hasPermission) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}

app.delete('/users/:id', authMiddleware, authorize('users:delete'), deleteUser);
```

---

### Q8: OAuth2 flow cho social login?

```
1. User click "Login with Google"
2. Redirect → Google authorization server
3. User consent → Google redirect về callback URL với authorization code
4. Server exchange code → access token (server-to-server)
5. Server dùng access token lấy user profile từ Google
6. Tạo/link local user → issue JWT
```

**Passport.js strategy:**

```javascript
passport.use(new GoogleStrategy({
  clientID: process.env.GOOGLE_CLIENT_ID,
  clientSecret: process.env.GOOGLE_CLIENT_SECRET,
  callbackURL: '/auth/google/callback',
}, async (accessToken, refreshToken, profile, done) => {
  const user = await findOrCreateUser(profile);
  done(null, user);
}));
```

---

## OWASP Top 10

### Q9: OWASP Top 10 — liệt kê và cách prevent trong Node.js?

| # | Vulnerability | Prevention trong Node.js |
| - | ------------- | ------------------------ |
| **A01** | Broken Access Control | RBAC middleware, verify ownership |
| **A02** | Cryptographic Failures | bcrypt/argon2, HTTPS, không log secrets |
| **A03** | Injection | Parameterized queries, input validation (Zod) |
| **A04** | Insecure Design | Threat modeling, principle of least privilege |
| **A05** | Security Misconfiguration | Helmet, disable `X-Powered-By`, env-based config |
| **A06** | Vulnerable Components | `npm audit`, Dependabot, Snyk |
| **A07** | Auth Failures | Rate limit login, MFA, secure session |
| **A08** | Data Integrity Failures | Verify JWT signature, CI/CD integrity |
| **A09** | Logging Failures | Structured logging, không log PII/passwords |
| **A10** | SSRF | Validate URLs, whitelist domains, network segmentation |

---

### Q10: SQL Injection — prevent trong Node.js?

```javascript
// ❌ Vulnerable
const result = await pool.query(`SELECT * FROM users WHERE id = ${req.params.id}`);

// ✅ Parameterized
const result = await pool.query('SELECT * FROM users WHERE id = $1', [req.params.id]);

// ✅ ORM
const user = await prisma.user.findUnique({ where: { id: req.params.id } });
```

**Thêm:** Input validation, least privilege DB user (không dùng superuser), WAF (Web Application Firewall).

---

### Q11: XSS (Cross-Site Scripting) — prevent?

| Loại | Mô Tả | Prevention |
| ---- | ----- | ---------- |
| **Stored XSS** | Script lưu trong DB | Sanitize input, encode output |
| **Reflected XSS** | Script trong URL params | Validate/sanitize query params |
| **DOM XSS** | Client-side injection | CSP header, avoid `innerHTML` |

```javascript
// API: sanitize user input
const sanitizeHtml = require('sanitize-html');
const cleanBio = sanitizeHtml(req.body.bio, { allowedTags: [] });

// Response headers
app.use(helmet.contentSecurityPolicy({
  directives: { defaultSrc: ["'self'"] },
}));
```

---

### Q12: CSRF (Cross-Site Request Forgery) — prevent?

```javascript
const csrf = require('csurf');
app.use(csrf({ cookie: { httpOnly: true, secure: true } }));

// Trả CSRF token cho client
app.get('/csrf-token', (req, res) => {
  res.json({ csrfToken: req.csrfToken() });
});

// Client gửi kèm header
// X-CSRF-Token: <token>
```

**Với JWT trong Authorization header:** CSRF risk thấp hơn (browser không tự gửi custom headers). Với cookie-based auth: bắt buộc CSRF protection.

---

## Security Headers & Rate Limiting

### Q13: Helmet.js — headers quan trọng?

```javascript
const helmet = require('helmet');
app.use(helmet());
// Tự động set:
// X-Content-Type-Options: nosniff
// X-Frame-Options: DENY
// Strict-Transport-Security (HSTS)
// Content-Security-Policy
// X-XSS-Protection (legacy browsers)
```

---

### Q14: Rate Limiting algorithms?

| Algorithm | Mô Tả | Use Case |
| --------- | ----- | -------- |
| **Fixed Window** | N requests per window (e.g., 100/min) | Đơn giản, có burst ở boundary |
| **Sliding Window** | Rolling window | Smooth hơn |
| **Token Bucket** | Tokens refill theo rate | Cho phép burst có kiểm soát |
| **Leaky Bucket** | Queue requests, process steady rate | Smooth output rate |

```javascript
// express-rate-limit dùng sliding window approximation
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5, // 5 login attempts per 15 min
  message: 'Too many login attempts',
});
app.post('/auth/login', loginLimiter, loginHandler);
```

---

### Q15: CORS misconfiguration — rủi ro?

```javascript
// ❌ Nguy hiểm — cho phép mọi origin
app.use(cors({ origin: '*' }));

// ❌ Nguy hiểm — reflect bất kỳ origin
app.use(cors({ origin: true }));

// ✅ Whitelist cụ thể
app.use(cors({
  origin: (origin, callback) => {
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
}));
```

---

## Secrets & Production Security

### Q16: Secrets management — best practices?

| ❌ Không Làm | ✅ Nên Làm |
| ----------- | ---------- |
| Hardcode secrets trong code | Environment variables |
| Commit `.env` vào git | `.env` trong `.gitignore` |
| Log secrets/tokens | Mask sensitive data trong logs |
| Dùng cùng secret dev/prod | Secret khác nhau per environment |
| Share secrets qua chat/email | Vault, AWS Secrets Manager, Doppler |

```javascript
// Validate required env vars at startup
const required = ['JWT_SECRET', 'DATABASE_URL', 'REDIS_URL'];
for (const key of required) {
  if (!process.env[key]) {
    throw new Error(`Missing required env var: ${key}`);
  }
}
```

---

### Q17: `npm audit` — xử lý vulnerabilities?

```bash
npm audit                    # Liệt kê vulnerabilities
npm audit fix                # Auto-fix non-breaking
npm audit fix --force        # Có thể breaking changes — cẩn thận
```

**Production workflow:**
1. CI pipeline chạy `npm audit` — fail nếu critical/high
2. Dependabot/Renovate tự động tạo PR update dependencies
3. Review changelog trước khi merge major version bumps

---

### Q18: Principle of Least Privilege — áp dụng thế nào?

- DB user cho app: chỉ SELECT, INSERT, UPDATE, DELETE trên tables cần thiết
- AWS IAM role: chỉ permissions cần cho service
- JWT payload: chỉ `userId` + `role`, không thêm unnecessary claims
- API endpoints: mỗi route có authorization check riêng
- Container: chạy non-root user trong Docker

---

## Live Coding Challenges

### Challenge 1: Implement password strength validator

```javascript
function validatePassword(password) {
  const errors = [];
  if (password.length < 8) errors.push('Min 8 characters');
  if (!/[A-Z]/.test(password)) errors.push('Need uppercase');
  if (!/[a-z]/.test(password)) errors.push('Need lowercase');
  if (!/[0-9]/.test(password)) errors.push('Need digit');
  if (!/[^A-Za-z0-9]/.test(password)) errors.push('Need special char');
  return errors;
}
```

### Challenge 2: Implement role-based middleware

```javascript
function requireRole(...roles) {
  return (req, res, next) => {
    if (!req.user) return res.status(401).json({ error: 'Unauthorized' });
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}

app.get('/admin/users', authMiddleware, requireRole('admin'), listUsers);
```

### Challenge 3: Secure headers middleware (không dùng Helmet)

```javascript
function securityHeaders(req, res, next) {
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '0'); // Disabled — CSP tốt hơn
  res.removeHeader('X-Powered-By');
  next();
}
```

---

## Tài Liệu Tham Khảo

- [05-security/1-authentication-jwt.md](../05-security/1-authentication-jwt.md)
- [05-security/4-owasp-top10.md](../05-security/4-owasp-top10.md)
- [05-security/5-helmet-rate-limiting.md](../05-security/5-helmet-rate-limiting.md)
- [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) — Câu 31–40
