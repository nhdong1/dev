# Request Validation — Zod, Joi và class-validator

> Validation (xác thực dữ liệu đầu vào) là defense layer đầu tiên — ngăn invalid, malicious, hoặc malformed data vào business logic và database. Không tin tưởng client input.

## Mục Lục

1. [Tại Sao Cần Validation](#tại-sao-cần-validation)
2. [Validation Layers](#validation-layers)
3. [Zod — TypeScript-First Schema](#zod--typescript-first-schema)
4. [Joi — Mature Validation Library](#joi--mature-validation-library)
5. [class-validator — NestJS Integration](#class-validator--nestjs-integration)
6. [So Sánh Thư Viện](#so-sánh-thư-viện)
7. [Validation Middleware Patterns](#validation-middleware-patterns)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Validation

| Rủi Ro Không Validate | Hậu Quả |
| --------------------- | ------- |
| Missing required fields | Runtime errors, corrupt data |
| Wrong data types | Type coercion bugs, SQL errors |
| Oversized input | DoS, memory exhaustion |
| Invalid formats (email, UUID) | Business logic fail |
| Unknown/extra fields | Mass assignment vulnerability |

**Nguyên tắc:** Validate at the boundary (biên) — ngay khi data vào server.

---

## Validation Layers

```
Client (UI validation)     → UX only, KHÔNG tin cậy
        ↓
API Gateway / WAF          → Rate limit, basic rules
        ↓
Request Validation (Zod)   → Schema validation ← FOCUS
        ↓
Business Logic Validation  → Domain rules (unique email, stock check)
        ↓
Database Constraints       → Last line of defense (NOT NULL, UNIQUE)
```

---

## Zod — TypeScript-First Schema

[Zod](https://zod.dev/) — schema validation với TypeScript type inference:

```bash
npm install zod
```

### Basic Schema

```typescript
import { z } from 'zod';

const createUserSchema = z.object({
  email: z.string().email('Invalid email format'),
  name: z.string().min(2).max(100),
  age: z.number().int().positive().optional(),
  role: z.enum(['user', 'admin']).default('user'),
  tags: z.array(z.string()).max(10).optional(),
});

// Infer TypeScript type từ schema
type CreateUserInput = z.infer<typeof createUserSchema>;
// { email: string; name: string; age?: number; role: 'user' | 'admin'; tags?: string[] }
```

### Parsing và Error Handling

```typescript
// .parse() — throw ZodError nếu invalid
const user = createUserSchema.parse(req.body);

// .safeParse() — return result object, không throw
const result = createUserSchema.safeParse(req.body);
if (!result.success) {
  return res.status(400).json({
    error: {
      code: 'VALIDATION_ERROR',
      details: result.error.flatten().fieldErrors,
    },
  });
}
const user = result.data; // Typed và sanitized
```

### Advanced Zod Features

```typescript
// Transform — normalize data
const emailSchema = z.string().email().transform(s => s.toLowerCase());

// Refine — custom validation
const passwordSchema = z.string()
  .min(8)
  .refine(val => /[A-Z]/.test(val), 'Must contain uppercase')
  .refine(val => /[0-9]/.test(val), 'Must contain number');

// Union và Discriminated Union
const eventSchema = z.discriminatedUnion('type', [
  z.object({ type: z.literal('click'), x: z.number(), y: z.number() }),
  z.object({ type: z.literal('scroll'), direction: z.enum(['up', 'down']) }),
]);

// Coerce — convert types từ string query params
const querySchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().max(100).default(20),
});

// Partial, Pick, Omit
const updateUserSchema = createUserSchema.partial(); // Tất cả fields optional
```

### Express Middleware với Zod

```typescript
import { z, ZodSchema } from 'zod';

function validateBody<T extends ZodSchema>(schema: T) {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json({
        error: {
          code: 'VALIDATION_ERROR',
          details: result.error.flatten().fieldErrors,
        },
      });
    }
    req.body = result.data;
    next();
  };
}

function validateQuery<T extends ZodSchema>(schema: T) {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.query);
    if (!result.success) {
      return res.status(400).json({
        error: { code: 'VALIDATION_ERROR', details: result.error.flatten().fieldErrors },
      });
    }
    req.query = result.data as any;
    next();
  };
}

app.post('/users', validateBody(createUserSchema), createUserHandler);
app.get('/users', validateQuery(querySchema), listUsersHandler);
```

---

## Joi — Mature Validation Library

[Joi](https://joi.dev/) — validation library phổ biến từ hapi ecosystem:

```bash
npm install joi
```

```javascript
const Joi = require('joi');

const createUserSchema = Joi.object({
  email: Joi.string().email().required(),
  name: Joi.string().min(2).max(100).required(),
  age: Joi.number().integer().positive(),
  role: Joi.string().valid('user', 'admin').default('user'),
}).options({ stripUnknown: true }); // Loại bỏ unknown fields

// Validate
const { error, value } = createUserSchema.validate(req.body, {
  abortEarly: false, // Trả tất cả lỗi, không dừng ở lỗi đầu
});

if (error) {
  return res.status(400).json({
    error: {
      code: 'VALIDATION_ERROR',
      details: error.details.map(d => ({
        field: d.path.join('.'),
        message: d.message,
      })),
    },
  });
}
req.body = value;
```

**Joi middleware:**

```javascript
const validate = (schema, property = 'body') => (req, res, next) => {
  const { error, value } = schema.validate(req[property], { abortEarly: false });
  if (error) {
    return res.status(400).json({
      error: {
        code: 'VALIDATION_ERROR',
        details: error.details.map(d => ({
          field: d.path.join('.'),
          message: d.message,
        })),
      },
    });
  }
  req[property] = value;
  next();
};

app.post('/users', validate(createUserSchema), handler);
```

---

## class-validator — NestJS Integration

[class-validator](https://github.com/typestack/class-validator) dùng decorators trên DTO classes:

```bash
npm install class-validator class-transformer
```

```typescript
import {
  IsEmail, IsString, MinLength, IsOptional, IsInt, IsEnum, IsUUID,
} from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(2)
  name: string;

  @IsOptional()
  @IsInt()
  age?: number;

  @IsEnum(['user', 'admin'])
  role: 'user' | 'admin' = 'user';
}

export class UpdateUserDto {
  @IsOptional()
  @IsEmail()
  email?: string;

  @IsOptional()
  @IsString()
  @MinLength(2)
  name?: string;
}

export class UserParamsDto {
  @IsUUID()
  id: string;
}
```

**NestJS ValidationPipe** tự động validate (xem [4-nestjs.md](./4-nestjs.md)):

```typescript
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,
  forbidNonWhitelisted: true,
  transform: true,
}));
```

**Custom validator:**

```typescript
import { registerDecorator, ValidationOptions, ValidationArguments } from 'class-validator';

export function IsStrongPassword(validationOptions?: ValidationOptions) {
  return function (object: object, propertyName: string) {
    registerDecorator({
      name: 'isStrongPassword',
      target: object.constructor,
      propertyName,
      options: validationOptions,
      validator: {
        validate(value: string) {
          return /^(?=.*[A-Z])(?=.*[0-9])(?=.*[!@#$%]).{8,}$/.test(value);
        },
        defaultMessage(args: ValidationArguments) {
          return `${args.property} must be a strong password`;
        },
      },
    });
  };
}
```

---

## So Sánh Thư Viện

| Tiêu Chí | Zod | Joi | class-validator |
| -------- | --- | --- | --------------- |
| **TypeScript inference** | Excellent | Qua `@types` | Native với DTO |
| **Bundle size** | ~12KB | ~150KB | ~45KB + reflect-metadata |
| **Runtime** | Node + Browser | Node + Browser | Node (decorators) |
| **Schema style** | Code-first | Chain API | Decorators |
| **NestJS integration** | Manual / plugin | Manual | Built-in |
| **Ecosystem** | Growing fast | Mature | NestJS ecosystem |
| **Learning curve** | Thấp | Trung bình | Cần hiểu decorators |

**Khuyến nghị:**
- **Express + TypeScript:** Zod
- **Express + JavaScript:** Joi
- **NestJS:** class-validator + DTO
- **Shared schemas frontend/backend:** Zod (có thể dùng cả client)

---

## Validation Middleware Patterns

### Validate Multiple Sources

```typescript
function validate(schemas: {
  body?: ZodSchema;
  query?: ZodSchema;
  params?: ZodSchema;
}) {
  return (req: Request, res: Response, next: NextFunction) => {
    const errors: Record<string, unknown> = {};

    for (const [key, schema] of Object.entries(schemas)) {
      if (!schema) continue;
      const result = schema.safeParse(req[key as keyof Request]);
      if (!result.success) {
        errors[key] = result.error.flatten().fieldErrors;
      } else {
        (req as any)[key] = result.data;
      }
    }

    if (Object.keys(errors).length > 0) {
      return res.status(400).json({ error: { code: 'VALIDATION_ERROR', details: errors } });
    }
    next();
  };
}

app.put('/users/:id',
  validate({
    params: z.object({ id: z.string().uuid() }),
    body: updateUserSchema,
  }),
  updateUserHandler
);
```

### Sanitization (Làm Sạch Dữ Liệu)

```typescript
import sanitizeHtml from 'sanitize-html';

const safeString = z.string().transform(val =>
  sanitizeHtml(val, { allowedTags: [], allowedAttributes: {} })
);

// Hoặc dùng validator.js
import validator from 'validator';
const emailSchema = z.string().transform(val => validator.normalizeEmail(val));
```

---

## Best Practices

| Practice | Chi Tiết |
| -------- | -------- |
| **Validate sớm** | Middleware trước controller/handler |
| **Whitelist fields** | `stripUnknown` / `whitelist: true` — chống mass assignment |
| **abortEarly: false** | Trả tất cả lỗi cùng lúc — UX tốt hơn |
| **Consistent error format** | Field-level errors với path và message |
| **Reuse schemas** | `.partial()`, `.pick()`, `.omit()` cho update DTOs |
| **Coerce query params** | Query luôn là string — cần `z.coerce.number()` |
| **Không validate business rules** | Unique email check ở service layer, không schema |
| **Version schemas** | Khi API version thay đổi, schema cũng version |

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Validation vs Sanitization?

**Trả lời:** Validation kiểm tra data có đúng format/rules không — reject nếu invalid. Sanitization transform data để an toàn (escape HTML, trim whitespace, normalize email). Cả hai nên dùng — validate structure, sanitize content.

### Câu 2: Zod vs Joi — chọn gì?

**Trả lời:** Zod khi dùng TypeScript — type inference tự động, bundle nhỏ, API hiện đại. Joi khi JavaScript thuần, team đã quen hapi ecosystem, cần validation rules phức tạp mature hơn.

### Câu 3: Validate ở đâu — client hay server?

**Trả lời:** Cả hai nhưng chỉ tin server validation. Client validation cho UX (instant feedback). Server validation bắt buộc — client có thể bypass (curl, Postman, modified requests).

### Câu 4: Làm sao validate nested objects?

**Trả lời:** Zod: `z.object({ address: z.object({ city: z.string() }) })`. Joi: `Joi.object({ address: Joi.object({ city: Joi.string() }) })`. class-validator: `@ValidateNested()` + `@Type(() => AddressDto)`.

---

**Xem tiếp:** [7-api-documentation.md](./7-api-documentation.md) — Swagger/OpenAPI documentation.
