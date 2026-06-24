# Input Validation và Sanitization — Whitelist Approach

> **Input validation (xác thực đầu vào)** và **sanitization (làm sạch dữ liệu)** là defense layer quan trọng chống Injection, mass assignment, và data corruption. Nguyên tắc: **whitelist (danh sách cho phép)**, không blacklist.

## Mục Lục

1. [Tại Sao Validation Là Security](#tại-sao-validation-là-security)
2. [Whitelist vs Blacklist](#whitelist-vs-blacklist)
3. [Zod Schema Validation](#zod-schema-validation)
4. [Express Validation Middleware](#express-validation-middleware)
5. [SQL Parameterization](#sql-parameterization)
6. [NoSQL Injection Prevention](#nosql-injection-prevention)
7. [XSS Prevention ở Backend](#xss-prevention-ở-backend)
8. [Mass Assignment Protection](#mass-assignment-protection)
9. [File Upload Validation](#file-upload-validation)
10. [Best Practices](#best-practices)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Validation Là Security

| Không Validate | Hậu Quả Bảo Mật |
| -------------- | --------------- |
| Unvalidated SQL input | SQL Injection |
| Unvalidated MongoDB query | NoSQL Injection |
| Oversized JSON body | DoS — memory exhaustion |
| Extra fields (`isAdmin: true`) | Mass assignment |
| Unvalidated file upload | Malware, path traversal |
| Unvalidated URL | SSRF |

Validation ở **03-web-frameworks/6-request-validation.md** tập trung DX và types; file này tập trung **security implications**.

---

## Whitelist vs Blacklist

```
Blacklist:  Chặn <script>, SELECT, DROP, ../  → Dễ bypass, không đủ
Whitelist:  Chỉ cho phép email format, enum values, known fields  → An toàn
```

**Nguyên tắc OWASP:** Validate input theo **expected format**, reject everything else.

```typescript
// ❌ Blacklist — dễ bypass
if (input.includes('<script>')) throw new Error('Invalid');

// ✅ Whitelist — chỉ cho phép known schema
const schema = z.object({
  email: z.string().email(),
  role: z.enum(['user', 'editor']), // Không có 'admin'
});
```

---

## Zod Schema Validation

```bash
npm install zod
```

### Strict Schema — Chặn Extra Fields

```typescript
import { z } from 'zod';

const createUserSchema = z.object({
  email: z.string().email().max(255),
  name: z.string().min(1).max(100),
  password: z.string().min(12).max(128),
}).strict(); // Reject unknown keys — chống mass assignment

// Hoặc .strip() để silently remove unknown keys
const updateUserSchema = z.object({
  name: z.string().min(1).max(100).optional(),
  bio: z.string().max(500).optional(),
}).strict();
```

### Sanitize Transforms

```typescript
const loginSchema = z.object({
  email: z.string().email().transform(s => s.toLowerCase().trim()),
  password: z.string().min(1).max(128),
});

const paginationSchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20), // Max 100 — chống DoS
});
```

### UUID Validation — Chống Injection qua Params

```typescript
const idParamSchema = z.object({
  id: z.string().uuid(),
});

app.get('/api/users/:id', (req, res, next) => {
  const result = idParamSchema.safeParse(req.params);
  if (!result.success) {
    return res.status(400).json({ error: 'Invalid ID format' });
  }
  next();
});
```

---

## Express Validation Middleware

```typescript
import { z, ZodSchema } from 'zod';
import { Request, Response, NextFunction } from 'express';

export function validate(schema: {
  body?: ZodSchema;
  query?: ZodSchema;
  params?: ZodSchema;
}) {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      if (schema.body) req.body = schema.body.parse(req.body);
      if (schema.query) req.query = schema.query.parse(req.query);
      if (schema.params) req.params = schema.params.parse(req.params);
      next();
    } catch (err) {
      if (err instanceof z.ZodError) {
        return res.status(400).json({
          error: 'Validation failed',
          details: err.flatten().fieldErrors,
        });
      }
      next(err);
    }
  };
}

// Usage
app.post('/api/users',
  validate({ body: createUserSchema }),
  createUserHandler
);
```

### Limit Request Body Size

```typescript
app.use(express.json({ limit: '10kb' })); // Chống large payload DoS
app.use(express.urlencoded({ extended: true, limit: '10kb' }));
```

---

## SQL Parameterization

```typescript
import { Pool } from 'pg';

const pool = new Pool();

// ✅ Parameterized — pg tự escape
async function getUserByEmail(email: string) {
  const { rows } = await pool.query(
    'SELECT id, email, name FROM users WHERE email = $1',
    [email]
  );
  return rows[0];
}

// ✅ Prisma — safe by default
const user = await prisma.user.findUnique({ where: { email } });

// ⚠️ Prisma $queryRaw — dùng tagged template
const users = await prisma.$queryRaw`
  SELECT * FROM users WHERE created_at > ${date}
`;

// ❌ NEVER — string interpolation
const users = await prisma.$queryRawUnsafe(
  `SELECT * FROM users WHERE email = '${email}'`
);
```

---

## NoSQL Injection Prevention

```typescript
import mongoSanitize from 'express-mongo-sanitize';

// Remove $ và . từ req.body, req.query, req.params
app.use(mongoSanitize({
  replaceWith: '_',
  onSanitize: ({ req, key }) => {
    console.warn(`Sanitized ${key} in request`);
  },
}));
```

```typescript
// ❌ VULNERABLE
// POST { "email": { "$gt": "" }, "password": { "$gt": "" } }
const user = await User.findOne(req.body);

// ✅ SAFE
const { email, password } = loginSchema.parse(req.body);
const user = await User.findOne({ email });
```

### Mongoose Type Casting

```typescript
const userSchema = new Schema({
  email: { type: String, required: true },
  age: { type: Number, min: 0, max: 150 },
  role: { type: String, enum: ['user', 'editor', 'admin'], default: 'user' },
});
```

---

## XSS Prevention ở Backend

XSS chủ yếu là client-side, nhưng API có thể contribute:

```typescript
import { escape } from 'he'; // HTML entity encode

// Nếu API trả HTML hoặc render templates
function sanitizeHtml(input: string): string {
  return escape(input);
}

// Stored XSS — validate trước khi lưu DB
const commentSchema = z.object({
  content: z.string().max(2000),
  // Không cho phép raw HTML — hoặc dùng DOMPurify server-side
});
```

**Backend responsibilities:**
- Validate/sanitize input trước khi store
- Set `Content-Type: application/json` — không `text/html` cho API
- CSP headers qua Helmet
- Không reflect user input trong error messages

---

## Mass Assignment Protection

```typescript
// ❌ VULNERABLE — user gửi { "name": "Alice", "isAdmin": true }
const user = await prisma.user.update({
  where: { id },
  data: req.body,
});

// ✅ SAFE — explicit allowed fields
const updateSchema = z.object({
  name: z.string().min(1).max(100).optional(),
  bio: z.string().max(500).optional(),
}).strict();

app.patch('/api/users/:id', validate({ body: updateSchema }), async (req, res) => {
  const user = await prisma.user.update({
    where: { id: req.params.id },
    data: req.body, // Đã validated — chỉ name, bio
  });
  res.json(user);
});
```

---

## File Upload Validation

```typescript
import multer from 'multer';
import path from 'path';

const ALLOWED_MIME_TYPES = ['image/jpeg', 'image/png', 'image/webp'];
const MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB

const upload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: MAX_FILE_SIZE, files: 1 },
  fileFilter(req, file, cb) {
    if (!ALLOWED_MIME_TYPES.includes(file.mimetype)) {
      return cb(new Error('Invalid file type'));
    }
    // Validate extension — nhưng trust mimetype hơn extension
    const ext = path.extname(file.originalname).toLowerCase();
    if (!['.jpg', '.jpeg', '.png', '.webp'].includes(ext)) {
      return cb(new Error('Invalid file extension'));
    }
    cb(null, true);
  },
});

app.post('/api/avatar', upload.single('avatar'), async (req, res) => {
  if (!req.file) return res.status(400).json({ error: 'No file' });

  // Generate random filename — chống path traversal
  const filename = `${crypto.randomUUID()}.webp`;
  // Process và store...
});
```

**Không bao giờ:** dùng `file.originalname` trực tiếp làm path, execute uploaded files, trust client MIME type alone (verify magic bytes).

---

## Best Practices

1. **Validate at boundary** — ngay khi data vào server
2. **Whitelist schema** — Zod `.strict()` chống extra fields
3. **Parameterized queries** — mọi SQL interaction
4. **Limit body size** — `express.json({ limit: '10kb' })`
5. **Validate params** — UUID format, numeric IDs
6. **Pagination max limit** — default 20, max 100
7. **`express-mongo-sanitize`** cho MongoDB apps
8. **Không log raw passwords** trong validation errors

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Validation vs Sanitization — khác gì?

**Trả lời:** Validation kiểm tra data đúng format/rules — reject nếu invalid. Sanitization transform data (trim, lowercase, escape HTML). Prefer validation + reject; sanitize khi cần normalize.

### Câu 2: Tại sao blacklist không đủ?

**Trả lời:** Attackers tìm bypass — encoding, unicode, nested objects (`$gt`). Whitelist chỉ accept known good patterns — attack surface nhỏ hơn nhiều.

### Câu 3: ORM có tự chống SQL Injection không?

**Trả lời:** ORM query builder an toàn mặc định. Raw queries (`$queryRawUnsafe`, string concat) vẫn vulnerable. Luôn parameterized.

### Câu 4: Mass assignment là gì?

**Trả lời:** Attacker gửi extra fields (`isAdmin: true`, `role: admin`) trong request body. Fix: strict schema, explicit allowed fields trong update operations.

---

**Xem tiếp:** [7-secrets-management.md](./7-secrets-management.md) — Quản lý secrets và environment variables.
