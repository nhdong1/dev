# Error Handling Architecture — Result Pattern, Either Monad và Global Error Boundary

> Xử lý lỗi (error handling) không chỉ là try/catch — đó là **kiến trúc** quyết định cách errors flow qua layers, được log, và trả về client. Chủ đề cover **Result pattern**, **error taxonomy (phân loại lỗi)**, và **global error boundary** cho Node.js API.

## Mục Lục

1. [Tại Sao Error Handling Cần Kiến Trúc](#tại-sao-error-handling-cần-kiến-trúc)
2. [Error Taxonomy — Phân Loại Lỗi](#error-taxonomy--phân-loại-lỗi)
3. [Throw vs Return — Trade-offs](#throw-vs-return--trade-offs)
4. [Result Pattern](#result-pattern)
5. [Either Monad — Functional Approach](#either-monad--functional-approach)
6. [Custom Error Classes](#custom-error-classes)
7. [Global Error Boundary — Express/Fastify/NestJS](#global-error-boundary--expressfastifynestjs)
8. [Error Handling Across Layers](#error-handling-across-layers)
9. [Logging và Observability](#logging-và-observability)
10. [Best Practices](#best-practices)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Error Handling Cần Kiến Trúc

```
❌ Không có kiến trúc:
   - throw Error everywhere với message khác nhau
   - Controller catch và res.status(500) generic
   - Client nhận "Internal Server Error" — không actionable
   - Logs thiếu context — debug impossible

✅ Có kiến trúc:
   - Typed errors với error codes
   - Layer boundaries map errors appropriately
   - Client nhận structured error response
   - Logs có correlationId, stack, business context
```

**Mục tiêu:**
1. **Predictable** — Developer biết error sẽ flow thế nào
2. **Type-safe** — Compiler catch unhandled cases
3. **User-friendly** — Client messages actionable, không leak internals
4. **Observable** — Logs và metrics classify errors

---

## Error Taxonomy — Phân Loại Lỗi

| Loại | HTTP Status | Ví Dụ | Client Action |
| ---- | ----------- | ----- | ------------- |
| **Validation Error** | 400 | Invalid email format | Fix input |
| **Authentication Error** | 401 | Token expired | Re-login |
| **Authorization Error** | 403 | Insufficient permissions | Contact admin |
| **Not Found** | 404 | User ID không tồn tại | Check resource ID |
| **Conflict** | 409 | Email đã registered | Use different email |
| **Business Rule Violation** | 422 | Insufficient stock | Reduce quantity |
| **Rate Limit** | 429 | Too many requests | Retry after delay |
| **Internal Error** | 500 | DB connection failed | Retry, contact support |
| **Service Unavailable** | 503 | Dependency down | Retry with backoff |

### Operational vs Programmer Errors

Theo Node.js best practices (Joyent):

| | Operational Error | Programmer Error |
| --- | ----------------- | ---------------- |
| **Nguyên nhân** | Network timeout, invalid input | Bug, null reference |
| **Xử lý** | Catch, respond gracefully | Log, fix code, restart if needed |
| **Ví dụ** | `ECONNREFUSED`, validation fail | `undefined is not a function` |

---

## Throw vs Return — Trade-offs

### Throw (Exception-based)

```typescript
async function getUser(id: string): Promise<User> {
  const user = await repo.findById(id);
  if (!user) throw new NotFoundError('USER_NOT_FOUND', `User ${id} not found`);
  return user;
}

// Caller
try {
  const user = await getUser(id);
} catch (err) {
  if (err instanceof NotFoundError) { /* handle */ }
  throw err;
}
```

**Pros:** Familiar, works với async/await, stack traces
**Cons:** Implicit control flow, dễ miss catch, không type-enforced

### Return (Result-based)

```typescript
async function getUser(id: string): Promise<Result<User, UserError>> {
  const user = await repo.findById(id);
  if (!user) return err({ code: 'USER_NOT_FOUND', message: `User ${id} not found` });
  return ok(user);
}

// Caller — MUST handle both cases
const result = await getUser(id);
if (result.isErr()) {
  return mapErrorToHttp(result.error);
}
const user = result.value;
```

**Pros:** Explicit, type-safe, compiler forces handling
**Cons:** Verbose, boilerplate mapping

**Khuyến nghị Node.js:**
- **Domain/Application layer:** Result pattern cho expected business errors
- **Infrastructure:** Throw operational errors (DB down)
- **Presentation:** Global handler map tất cả → HTTP response

---

## Result Pattern

### Implementation

```typescript
// shared/result.ts
export type Ok<T> = { ok: true; value: T };
export type Err<E> = { ok: false; error: E };
export type Result<T, E = Error> = Ok<T> | Err<E>;

export const ok = <T>(value: T): Ok<T> => ({ ok: true, value });
export const err = <E>(error: E): Err<E> => ({ ok: false, error });

export function isOk<T, E>(result: Result<T, E>): result is Ok<T> {
  return result.ok === true;
}

export function isErr<T, E>(result: Result<T, E>): result is Err<E> {
  return result.ok === false;
}
```

### Use Case với Result

```typescript
type CreateUserError =
  | { code: 'EMAIL_ALREADY_EXISTS'; email: string }
  | { code: 'INVALID_EMAIL'; reason: string }
  | { code: 'WEAK_PASSWORD'; requirements: string[] };

class CreateUserUseCase {
  async execute(input: CreateUserInput): Promise<Result<User, CreateUserError>> {
    if (!Email.isValid(input.email)) {
      return err({ code: 'INVALID_EMAIL', reason: 'Format invalid' });
    }

    const existing = await this.userRepo.findByEmail(input.email);
    if (existing) {
      return err({ code: 'EMAIL_ALREADY_EXISTS', email: input.email });
    }

    const passwordCheck = Password.validate(input.password);
    if (!passwordCheck.valid) {
      return err({ code: 'WEAK_PASSWORD', requirements: passwordCheck.requirements });
    }

    const user = User.create(input);
    await this.userRepo.save(user);
    return ok(user);
  }
}
```

### Map Result → HTTP

```typescript
function mapCreateUserError(error: CreateUserError): { status: number; body: unknown } {
  switch (error.code) {
    case 'EMAIL_ALREADY_EXISTS':
      return { status: 409, body: { error: error.code, email: error.email } };
    case 'INVALID_EMAIL':
      return { status: 400, body: { error: error.code, reason: error.reason } };
    case 'WEAK_PASSWORD':
      return { status: 422, body: { error: error.code, requirements: error.requirements } };
  }
}
```

---

## Either Monad — Functional Approach

**Either<L, R>** tương tự Result — Left = error, Right = success. Thư viện phổ biến: `fp-ts`, `neverthrow`.

### neverthrow — Lightweight cho TypeScript

```typescript
import { ok, err, ResultAsync } from 'neverthrow';

function parseUserId(raw: string): Result<string, 'INVALID_UUID'> {
  const uuidRegex = /^[0-9a-f-]{36}$/i;
  return uuidRegex.test(raw) ? ok(raw) : err('INVALID_UUID');
}

function fetchUser(id: string): ResultAsync<User, 'NOT_FOUND' | 'DB_ERROR'> {
  return ResultAsync.fromPromise(
    userRepo.findById(id),
    () => 'DB_ERROR' as const,
  ).andThen((user) =>
    user ? ok(user) : err('NOT_FOUND' as const),
  );
}

// Chain operations — errors short-circuit
const result = parseUserId(req.params.id)
  .asyncAndThen(fetchUser)
  .map((user) => ({ id: user.id, name: user.name }));
```

### Railway Oriented Programming

```
Success track:  ──ok──► ──ok──► ──ok──► Result
                     │       │
Error track:         err     err     err ──► early return
```

---

## Custom Error Classes

```typescript
// domain/errors/app-error.ts
export abstract class AppError extends Error {
  abstract readonly code: string;
  abstract readonly httpStatus: number;
  readonly isOperational = true;

  constructor(
    message: string,
    public readonly context?: Record<string, unknown>,
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }

  toJSON() {
    return {
      error: this.code,
      message: this.message,
      ...(process.env.NODE_ENV !== 'production' && { context: this.context }),
    };
  }
}

export class NotFoundError extends AppError {
  readonly code = 'NOT_FOUND';
  readonly httpStatus = 404;
}

export class ValidationError extends AppError {
  readonly code = 'VALIDATION_ERROR';
  readonly httpStatus = 400;

  constructor(
    message: string,
    public readonly fields: Record<string, string[]>,
  ) {
    super(message, { fields });
  }
}

export class UnauthorizedError extends AppError {
  readonly code = 'UNAUTHORIZED';
  readonly httpStatus = 401;
}
```

---

## Global Error Boundary — Express/Fastify/NestJS

### Express — Error Middleware (Phải Đặt Cuối)

```typescript
import type { Request, Response, NextFunction } from 'express';
import { AppError } from '../domain/errors/app-error';

export function globalErrorHandler(
  err: unknown,
  req: Request,
  res: Response,
  _next: NextFunction,
) {
  const requestId = req.headers['x-request-id'] ?? 'unknown';

  if (err instanceof AppError) {
    logger.warn({ err, requestId, code: err.code }, err.message);
    return res.status(err.httpStatus).json(err.toJSON());
  }

  // Programmer error — log full stack
  logger.error({ err, requestId, stack: err instanceof Error ? err.stack : undefined }, 'Unhandled error');

  return res.status(500).json({
    error: 'INTERNAL_SERVER_ERROR',
    message: 'An unexpected error occurred',
    requestId,
  });
}

// Async route wrapper — forward errors to handler
export const asyncHandler = (fn: RequestHandler): RequestHandler =>
  (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);

// Usage
app.get('/users/:id', asyncHandler(async (req, res) => {
  const result = await getUserUseCase.execute(req.params.id);
  if (result.isErr()) throw mapToAppError(result.error);
  res.json(result.value);
}));

app.use(globalErrorHandler); // LAST middleware
```

### Fastify — setErrorHandler

```typescript
app.setErrorHandler((err, request, reply) => {
  if (err instanceof AppError) {
    return reply.status(err.httpStatus).send(err.toJSON());
  }
  request.log.error(err);
  return reply.status(500).send({ error: 'INTERNAL_SERVER_ERROR' });
});
```

### NestJS — Exception Filters

```typescript
@Catch(AppError)
export class AppErrorFilter implements ExceptionFilter {
  catch(exception: AppError, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();

    response.status(exception.httpStatus).json(exception.toJSON());
  }
}

// Global
app.useGlobalFilters(new AppErrorFilter(), new AllExceptionsFilter());
```

---

## Error Handling Across Layers

```
┌─────────────────────────────────────────────────────────────┐
│ Presentation    │ Map AppError/Result → HTTP response        │
├─────────────────┼───────────────────────────────────────────┤
│ Application     │ Return Result<T, BusinessError>            │
│                 │ Throw AppError for unexpected              │
├─────────────────┼───────────────────────────────────────────┤
│ Domain          │ Return Result or throw DomainError         │
├─────────────────┼───────────────────────────────────────────┤
│ Infrastructure  │ Catch DB/Network errors → wrap AppError    │
│                 │ Không leak Prisma/axios errors ra ngoài    │
└─────────────────┴───────────────────────────────────────────┘
```

### Infrastructure Error Wrapping

```typescript
class PrismaUserRepository implements UserRepository {
  async findById(id: string): Promise<User | null> {
    try {
      const row = await this.prisma.user.findUnique({ where: { id } });
      return row ? User.fromPersistence(row) : null;
    } catch (err) {
      if (err instanceof Prisma.PrismaClientKnownRequestError) {
        throw new DatabaseError('DB_QUERY_FAILED', { cause: err.code });
      }
      throw err;
    }
  }
}
```

---

## Logging và Observability

### Structured Error Logging

```typescript
logger.error({
  err: {
    message: err.message,
    code: err instanceof AppError ? err.code : 'UNKNOWN',
    stack: err.stack,
  },
  requestId,
  userId: req.user?.id,
  method: req.method,
  path: req.path,
  duration: Date.now() - startTime,
}, 'Request failed');
```

### Error Metrics

```typescript
const errorCounter = new Counter({
  name: 'http_errors_total',
  help: 'Total HTTP errors by status and code',
  labelNames: ['status', 'error_code', 'route'],
});

// In error handler
errorCounter.inc({
  status: String(status),
  error_code: err.code ?? 'UNKNOWN',
  route: req.route?.path ?? req.path,
});
```

### Standard Error Response Format

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "fields": {
    "email": ["Invalid email format"],
    "password": ["Must be at least 8 characters"]
  },
  "requestId": "req-abc-123",
  "timestamp": "2026-06-24T10:00:00Z"
}
```

---

## Best Practices

### Do

- **Fail fast** — Validate input sớm ở presentation layer
- **Single error handler** — Một nơi map errors → HTTP
- **Include requestId** — Trace errors across logs
- **Sanitize production responses** — Không leak stack traces, SQL, internal paths
- **Handle unhandledRejection** — Process-level safety net

```typescript
process.on('unhandledRejection', (reason) => {
  logger.fatal({ reason }, 'Unhandled promise rejection');
  // Graceful shutdown in production
});
```

### Don't

- **Empty catch blocks** — `catch (e) {}` nuốt errors
- **Generic messages everywhere** — "Something went wrong" không giúp debug
- **Mix error styles** — Chọn Result OR throw, document convention
- **Return 200 with error body** — Luôn dùng appropriate HTTP status

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Gợi Ý Trả Lời |
| ------- | ------------- |
| Result vs throw? | Result: explicit, type-safe expected errors. Throw: unexpected, simpler syntax |
| Operational vs programmer error? | Operational: handle gracefully. Programmer: fix bug, log, possibly restart |
| Global error handler Express? | 4-arg middleware `(err, req, res, next)` — must be last |
| Error response structure? | code, message, optional fields/context, requestId — no internals in prod |
| Error handling across layers? | Domain returns Result, infra wraps DB errors, presentation maps to HTTP |
| unhandledRejection handler? | Log fatal, graceful shutdown — prevent silent failures |
| Either/neverthrow? | Functional Result type — railway-oriented chaining |

---

**Quay lại:** [README.md](./README.md) — Tổng quan chủ đề Kiến Trúc
