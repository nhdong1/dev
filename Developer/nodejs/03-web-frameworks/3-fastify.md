# Fastify — Schema Validation, Plugins và Performance

> Fastify là web framework tập trung hiệu năng cao, hỗ trợ JSON Schema validation tích hợp, plugin encapsulation (đóng gói plugin), và TypeScript-first design.

## Mục Lục

1. [Fastify Là Gì](#fastify-là-gì)
2. [Khởi Tạo và Routing](#khởi-tạo-và-routing)
3. [JSON Schema Validation](#json-schema-validation)
4. [Hooks Lifecycle](#hooks-lifecycle)
5. [Plugin System](#plugin-system)
6. [Serialization và Performance](#serialization-và-performance)
7. [So Sánh với Express](#so-sánh-với-express)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Fastify Là Gì

| Đặc Điểm | Chi Tiết |
| -------- | -------- |
| **Performance** | ~2–3x throughput so với Express (benchmark) |
| **Schema-first** | Request/response validation qua JSON Schema |
| **Plugin encapsulation** | Mỗi plugin có context riêng, tránh pollution |
| **Logging** | Pino logger tích hợp — structured logging nhanh |
| **TypeScript** | Type inference từ schema |

```bash
npm install fastify
```

```javascript
const fastify = require('fastify')({ logger: true });

fastify.get('/health', async () => ({ status: 'ok' }));

const start = async () => {
  await fastify.listen({ port: 3000, host: '0.0.0.0' });
};
start();
```

---

## Khởi Tạo và Routing

```javascript
const fastify = require('fastify')({
  logger: {
    level: 'info',
    transport: process.env.NODE_ENV === 'development'
      ? { target: 'pino-pretty' }
      : undefined,
  },
  trustProxy: true, // Khi đứng sau reverse proxy
  bodyLimit: 1048576, // 1MB
});

// Route với schema
fastify.get('/users/:id', {
  schema: {
    params: {
      type: 'object',
      properties: { id: { type: 'string', format: 'uuid' } },
      required: ['id'],
    },
  },
}, async (request, reply) => {
  const { id } = request.params;
  return { id, name: 'John' };
});

// POST với body schema
fastify.post('/users', {
  schema: {
    body: {
      type: 'object',
      required: ['email', 'name'],
      properties: {
        email: { type: 'string', format: 'email' },
        name: { type: 'string', minLength: 2 },
      },
    },
    response: {
      201: {
        type: 'object',
        properties: {
          id: { type: 'string' },
          email: { type: 'string' },
          name: { type: 'string' },
        },
      },
    },
  },
}, async (request, reply) => {
  const user = await createUser(request.body);
  reply.code(201);
  return user;
});
```

**Reply object:**

```javascript
fastify.get('/redirect', async (request, reply) => {
  return reply.redirect('/new-url');
});

fastify.get('/custom', async (request, reply) => {
  reply.header('X-Custom', 'value');
  reply.code(200);
  return { data: 'value' };
});
```

---

## JSON Schema Validation

Fastify dùng [Ajv](https://ajv.js.org/) validate request/response tự động:

```javascript
const userSchema = {
  type: 'object',
  required: ['email'],
  properties: {
    email: { type: 'string', format: 'email' },
    age: { type: 'integer', minimum: 0, maximum: 150 },
    role: { type: 'string', enum: ['user', 'admin'] },
  },
  additionalProperties: false, // Reject unknown fields
};

fastify.post('/users', { schema: { body: userSchema } }, handler);
```

**Validation error response mặc định:**

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": "body/email must match format \"email\""
}
```

**Custom error handler:**

```javascript
fastify.setErrorHandler((error, request, reply) => {
  if (error.validation) {
    return reply.status(400).send({
      error: 'VALIDATION_ERROR',
      details: error.validation,
    });
  }
  reply.status(error.statusCode || 500).send({
    error: error.message,
  });
});
```

**Dùng Zod với Fastify** — `@fastify/type-provider-typebox` hoặc `fastify-type-provider-zod`:

```typescript
import { serializerCompiler, validatorCompiler } from 'fastify-type-provider-zod';
import { z } from 'zod';

const app = fastify();
app.setValidatorCompiler(validatorCompiler);
app.setSerializerCompiler(serializerCompiler);

const userSchema = z.object({
  email: z.string().email(),
  name: z.string().min(2),
});

app.post('/users', {
  schema: { body: userSchema },
}, async (req) => req.body);
```

---

## Hooks Lifecycle

Fastify có lifecycle hooks chi tiết hơn Express middleware:

```
onRequest → preParsing → preValidation → preHandler → Handler → preSerialization → onSend → onResponse
                ↓ (on error)
              onError
```

```javascript
// Authentication hook
fastify.addHook('preHandler', async (request, reply) => {
  const token = request.headers.authorization?.replace('Bearer ', '');
  if (!token) {
    return reply.status(401).send({ error: 'Unauthorized' });
  }
  request.user = await verifyToken(token);
});

// Request timing
fastify.addHook('onRequest', async (request) => {
  request.startTime = Date.now();
});

fastify.addHook('onResponse', async (request, reply) => {
  const duration = Date.now() - request.startTime;
  request.log.info({ duration, statusCode: reply.statusCode }, 'request completed');
});
```

| Hook | Mục Đích |
| ---- | -------- |
| `onRequest` | Đầu tiên — logging, request ID |
| `preValidation` | Trước schema validation — transform body |
| `preHandler` | Sau validation — auth, authorization |
| `onSend` | Trước gửi response — modify payload |
| `onError` | Khi có lỗi |

---

## Plugin System

Plugins được đăng ký với `fastify.register()` — mỗi plugin có **encapsulated context**:

```javascript
// plugins/users.js
async function usersPlugin(fastify, options) {
  fastify.get('/users', async () => {
    return [{ id: 1, name: 'Alice' }];
  });

  fastify.get('/users/:id', async (request) => {
    return { id: request.params.id };
  });
}

module.exports = usersPlugin;

// app.js
fastify.register(usersPlugin, { prefix: '/api/v1' });
// → GET /api/v1/users
```

**Plugin với dependencies:**

```javascript
const fp = require('fastify-plugin'); // Bỏ encapsulation khi cần share decorators

async function dbPlugin(fastify) {
  const db = await connectDatabase();
  fastify.decorate('db', db); // Thêm fastify.db

  fastify.addHook('onClose', async () => {
    await db.disconnect();
  });
}

module.exports = fp(dbPlugin); // fp() làm decorator available globally
```

**Autoload routes:**

```bash
npm install @fastify/autoload
```

```javascript
const AutoLoad = require('@fastify/autoload');
const path = require('path');

fastify.register(AutoLoad, {
  dir: path.join(__dirname, 'routes'),
  options: { prefix: '/api' },
});
```

---

## Serialization và Performance

Fastify pre-compile JSON Schema thành serialization function — nhanh hơn `JSON.stringify` thông thường:

```javascript
// Response schema → fast-json-stringify compile
fastify.get('/users', {
  schema: {
    response: {
      200: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            id: { type: 'string' },
            name: { type: 'string' },
          },
        },
      },
    },
  },
}, async () => users);
```

**Benchmark tips:**

| Optimization | Hiệu Quả |
| ------------ | --------- |
| Response schema | Serialization nhanh hơn 2–3x |
| `logger: false` trong benchmark | Loại bỏ I/O overhead |
| Keep-alive connections | Giảm TCP handshake |
| `@fastify/compress` | Trade CPU cho bandwidth |

```bash
npx autocannon -c 100 -d 10 http://localhost:3000/health
```

---

## So Sánh với Express

| Aspect | Express | Fastify |
| ------ | ------- | ------- |
| Validation | Thư viện ngoài | Built-in JSON Schema |
| Async errors | Cần wrapper (v4) | Native support |
| Plugin model | `app.use()` flat | Encapsulated plugins |
| Logging | Thư viện ngoài | Pino built-in |
| Learning curve | Thấp | Trung bình |
| Ecosystem middleware | Rất lớn | Cần `@fastify/*` packages |

**Migration từ Express:** Nhiều `@fastify/*` packages tương đương Express middleware (`@fastify/cors`, `@fastify/helmet`, `@fastify/multipart`).

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Tại sao Fastify nhanh hơn Express?

**Trả lời:** Schema-based serialization với `fast-json-stringify` (pre-compiled). Ít overhead trong routing (find-my-way router). Pino logger async và minimal. Plugin encapsulation tránh middleware chain dài không cần thiết.

### Câu 2: Plugin encapsulation là gì?

**Trả lời:** Mỗi plugin có Fastify instance riêng. Decorators và hooks trong plugin không leak ra ngoài trừ khi dùng `fastify-plugin` wrapper. Giúp modular và tránh global state pollution.

### Câu 3: Khi nào KHÔNG nên dùng Fastify?

**Trả lời:** Team chỉ quen Express và không cần performance boost. Cần middleware Express-specific không có `@fastify` equivalent. Legacy codebase lớn — migration cost cao.

---

**Xem tiếp:** [4-nestjs.md](./4-nestjs.md) — Enterprise framework với DI và decorators.
