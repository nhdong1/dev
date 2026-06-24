# TypeScript Basics — Static Typing Cho Node.js Backend

> TypeScript (TS) là superset (siêu tập) của JavaScript thêm **static typing (kiểu tĩnh)** — bắt lỗi tại compile time, cải thiện IDE support, và là chuẩn trong dự án enterprise Node.js (NestJS, Prisma, tRPC).

## Mục Lục

1. [Tại Sao Dùng TypeScript](#tại-sao-dùng-typescript)
2. [Thiết Lập TypeScript](#thiết-lập-typescript)
3. [Basic Types](#basic-types)
4. [Interfaces và Type Aliases](#interfaces-và-type-aliases)
5. [Functions và Generics](#functions-và-generics)
6. [Classes và Access Modifiers](#classes-và-access-modifiers)
7. [Enums](#enums)
8. [Utility Types](#utility-types)
9. [Type Inference và Assertions](#type-inference-và-assertions)
10. [TypeScript với Node.js](#typescript-với-nodejs)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Dùng TypeScript

| Lợi Ích | Mô Tả |
| ------- | ----- |
| **Catch errors early** | Lỗi type phát hiện khi compile, không phải runtime |
| **Better IDE** | Autocomplete, go-to-definition, rename refactoring |
| **Self-documenting** | Types là documentation sống |
| **Safer refactoring** | Compiler báo mọi chỗ cần sửa khi đổi interface |
| **Ecosystem** | NestJS, Prisma, tRPC, Zod integration mạnh |

| Trade-off | Mô Tả |
| --------- | ----- |
| Build step | Cần compile TS → JS (hoặc ts-node/tsx) |
| Learning curve | Generics, conditional types phức tạp |
| Verbosity | Thêm type annotations |

---

## Thiết Lập TypeScript

```bash
npm init -y
npm install typescript @types/node --save-dev
npx tsc --init
```

### tsconfig.json Cơ Bản

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

| Option | Ý Nghĩa |
| ------ | ------- |
| `strict: true` | Bật tất cả strict checks — **luôn bật** |
| `target` | JS version output |
| `module: NodeNext` | ESM/CJS tương thích Node.js hiện đại |
| `outDir` / `rootDir` | Thư mục output / source |
| `declaration` | Tạo `.d.ts` files cho libraries |

```bash
# Compile
npx tsc

# Watch mode
npx tsc --watch

# Chạy trực tiếp (development)
npx tsx src/index.ts
npx ts-node src/index.ts
```

---

## Basic Types

```typescript
// Primitives
let name: string = 'Alice';
let age: number = 30;
let active: boolean = true;
let nothing: null = null;
let notDefined: undefined = undefined;

// Arrays
let ids: number[] = [1, 2, 3];
let tags: Array<string> = ['api', 'rest'];

// Tuple — fixed length, typed positions
let pair: [string, number] = ['port', 3000];

// any — tắt type checking (tránh dùng)
let data: any = fetchSomething();

// unknown — an toàn hơn any, phải narrow trước khi dùng
let input: unknown = getUserInput();
if (typeof input === 'string') {
  console.log(input.toUpperCase());
}

// void — function không return
function log(msg: string): void {
  console.log(msg);
}

// never — function không bao giờ return (throw hoặc infinite loop)
function fail(msg: string): never {
  throw new Error(msg);
}

// object
let config: object = { host: 'localhost' }; // quá generic
let dbConfig: { host: string; port: number } = { host: 'localhost', port: 5432 };
```

---

## Interfaces và Type Aliases

### Interface

```typescript
interface User {
  readonly id: number;       // không thể gán lại
  name: string;
  email: string;
  role?: 'admin' | 'user';   // optional property
  createdAt: Date;
}

interface Admin extends User {
  permissions: string[];
}

// Implement interface
class UserEntity implements User {
  readonly id: number;
  name: string;
  email: string;
  createdAt: Date;

  constructor(id: number, name: string, email: string) {
    this.id = id;
    this.name = name;
    this.email = email;
    this.createdAt = new Date();
  }
}
```

### Type Alias

```typescript
type UserId = number;
type Role = 'admin' | 'user' | 'guest';  // Union type

type User = {
  id: UserId;
  name: string;
  role: Role;
};

// Intersection type
type AdminUser = User & { permissions: string[] };
```

### Interface vs Type

| Khía Cạnh | Interface | Type Alias |
| --------- | --------- | ---------- |
| Extend | `extends` | `&` intersection |
| Declaration merging | Có (merge cùng tên) | Không |
| Union/Primitive | Không | Có |
| **Khuyến nghị** | Object shapes, classes | Unions, primitives, complex types |

---

## Functions và Generics

### Function Types

```typescript
// Parameter và return types
function greet(name: string): string {
  return `Hello, ${name}`;
}

// Arrow function
const multiply = (a: number, b: number): number => a * b;

// Optional và default parameters
function connect(host: string, port: number = 5432): void { /* ... */ }

// Function type
type RequestHandler = (req: Request, res: Response) => void;
type AsyncHandler = (id: number) => Promise<User | null>;
```

### Generics — Type Parameters

```typescript
// Generic function
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

const num = first([1, 2, 3]);      // number | undefined
const str = first(['a', 'b']);     // string | undefined

// Generic interface — pattern phổ biến cho API response
interface ApiResponse<T> {
  data: T;
  meta: {
    page: number;
    total: number;
  };
}

interface PaginatedUsers extends ApiResponse<User[]> {}

// Generic với constraint
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

getProperty(user, 'name');   // OK
// getProperty(user, 'foo'); // Error
```

### Generic Class

```typescript
class Repository<T extends { id: number }> {
  constructor(private items: T[] = []) {}

  findById(id: number): T | undefined {
    return this.items.find(item => item.id === id);
  }

  create(item: Omit<T, 'id'>): T {
    const newItem = { ...item, id: Date.now() } as T;
    this.items.push(newItem);
    return newItem;
  }
}

const userRepo = new Repository<User>();
```

---

## Classes và Access Modifiers

```typescript
class UserService {
  private readonly db: Database;      // chỉ access trong class
  protected logger: Logger;            // class + subclasses
  public readonly version = '1.0';    // public (default)

  constructor(db: Database, logger: Logger) {
    this.db = db;
    this.logger = logger;
  }

  async findById(id: number): Promise<User | null> {
    return this.db.query<User>('SELECT * FROM users WHERE id = $1', [id]);
  }
}

// Parameter properties shorthand
class ProductService {
  constructor(
    private readonly db: Database,
    private readonly cache: Cache,
  ) {}
}
```

---

## Enums

```typescript
// Numeric enum (mặc định)
enum HttpStatus {
  OK = 200,
  Created = 201,
  BadRequest = 400,
  NotFound = 404,
  InternalError = 500,
}

// String enum — khuyến nghị hơn
enum UserRole {
  Admin = 'ADMIN',
  User = 'USER',
  Guest = 'GUEST',
}

function authorize(role: UserRole): boolean {
  return role === UserRole.Admin;
}

// Const enum — inline tại compile, không emit JS object
const enum Direction {
  Up = 'UP',
  Down = 'DOWN',
}
```

**Alternative hiện đại:** Union types thay enum — tree-shake friendly hơn:

```typescript
type UserRole = 'ADMIN' | 'USER' | 'GUEST';
const ROLES = ['ADMIN', 'USER', 'GUEST'] as const;
type UserRoleFromConst = typeof ROLES[number];
```

---

## Utility Types

TypeScript cung cấp built-in utility types — cực kỳ hữu ích cho API development.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

// Partial<T> — tất cả properties optional (update DTO)
type UpdateUserDto = Partial<Pick<User, 'name' | 'email'>>;

// Required<T> — tất cả properties required
type RequiredUser = Required<User>;

// Pick<T, K> — chọn subset properties
type UserPublic = Pick<User, 'id' | 'name' | 'email'>;

// Omit<T, K> — loại bỏ properties
type CreateUserDto = Omit<User, 'id' | 'createdAt'>;
type SafeUser = Omit<User, 'password'>;

// Record<K, V> — object với keys K và values V
type UserMap = Record<number, User>;
type ErrorCodes = Record<string, string>;

// Readonly<T>
type ImmutableUser = Readonly<User>;

// ReturnType<F> — return type của function
type FindResult = ReturnType<typeof userService.findById>; // Promise<User | null>

// Parameters<F> — parameter types của function
type FindParams = Parameters<typeof userService.findById>; // [number]

// Extract / Exclude — filter union types
type AdminOrUser = Extract<UserRole, 'ADMIN' | 'USER'>;
type NonAdmin = Exclude<UserRole, 'ADMIN'>;

// NonNullable<T> — loại null và undefined
type DefiniteUser = NonNullable<User | null | undefined>;
```

### Pattern: DTO (Data Transfer Object) Types

```typescript
// Entity (database)
interface UserEntity {
  id: number;
  name: string;
  email: string;
  password_hash: string;
  created_at: Date;
}

// DTO cho API
type CreateUserRequest = Pick<UserEntity, 'name' | 'email'> & { password: string };
type UserResponse = Omit<UserEntity, 'password_hash' | 'created_at'> & { createdAt: string };
```

---

## Type Inference và Assertions

### Type Inference

```typescript
// TypeScript tự suy luận type
const port = 3000;              // number
const users = [{ id: 1 }];      // { id: number }[]

// Return type inference
function createUser(name: string) {
  return { id: 1, name };       // inferred: { id: number; name: string }
}
```

### Type Assertions

```typescript
// as syntax
const input = document.getElementById('email') as HTMLInputElement;

// Node.js: unknown từ JSON parse
const data = JSON.parse(jsonString) as ApiResponse<User>;

// Non-null assertion (!) — dùng cẩn thận
const user = findUser(id)!;  // Assert không null
```

### Type Guards

```typescript
function isApiError(err: unknown): err is ApiError {
  return err instanceof ApiError;
}

function processError(err: unknown) {
  if (isApiError(err)) {
    console.log(err.statusCode);  // TypeScript biết err là ApiError
  }
}

// typeof / instanceof guards
if (typeof value === 'string') { /* ... */ }
if (value instanceof Date) { /* ... */ }
```

---

## TypeScript với Node.js

### @types Packages

```bash
npm install -D @types/express @types/node @types/pg @types/jsonwebtoken
```

DefinitelyTyped cung cấp type definitions cho JavaScript libraries.

### Express Handler Typing

```typescript
import { Request, Response, NextFunction } from 'express';

interface AuthRequest extends Request {
  user?: { id: number; role: string };
}

const getProfile = async (
  req: AuthRequest,
  res: Response,
  next: NextFunction,
): Promise<void> => {
  try {
    const user = await userService.findById(req.user!.id);
    res.json({ data: user });
  } catch (err) {
    next(err);
  }
};
```

### Environment Variables Typing

```typescript
// src/env.ts
function requireEnv(key: string): string {
  const value = process.env[key];
  if (!value) throw new Error(`Missing env: ${key}`);
  return value;
}

export const env = {
  NODE_ENV: process.env.NODE_ENV ?? 'development',
  PORT: parseInt(process.env.PORT ?? '3000', 10),
  DATABASE_URL: requireEnv('DATABASE_URL'),
  JWT_SECRET: requireEnv('JWT_SECRET'),
} as const;
```

### Zod — Runtime Validation + Types

```typescript
import { z } from 'zod';

const CreateUserSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
  role: z.enum(['admin', 'user']).default('user'),
});

type CreateUserInput = z.infer<typeof CreateUserSchema>;
// Tự động suy ra type từ schema — single source of truth
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: `any` vs `unknown`?

**Gợi ý trả lời:** `any` tắt type checking — có thể gọi bất kỳ method nào. `unknown` là type-safe top type — phải narrow (typeof, type guard) trước khi sử dụng. Luôn prefer `unknown` cho external data (API response, user input).

### Câu 2: Generics dùng để làm gì?

**Gợi ý trả lời:** Generics tạo reusable components giữ type safety. Ví dụ: `ApiResponse<T>` wrap mọi API response, `Repository<T>` cho CRUD operations, `Promise<T>` cho async return. Cho phép code generic mà không mất type information.

### Câu 3: `interface` vs `type` — khi nào dùng gì?

**Gợi ý trả lời:** Interface cho object shapes và class contracts — hỗ trợ declaration merging (mở rộng từ nhiều file). Type alias cho unions, intersections, mapped types, primitives. Team convention thường: interface cho public API objects, type cho mọi thứ khác.

### Câu 4: `strict: true` trong tsconfig bao gồm gì?

**Gợi ý trả lời:** Bật: `strictNullChecks` (null/undefined checking), `noImplicitAny`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, `alwaysStrict`. Quan trọng nhất: `strictNullChecks` — bắt `null`/`undefined` bugs.

### Câu 5: TypeScript compile ra gì? Node.js chạy file nào?

**Gợi ý trả lời:** `tsc` compile `.ts` → `.js` trong `outDir` (thường `dist/`). Production chạy `node dist/index.js`. Development có thể dùng `tsx`/`ts-node` chạy trực tiếp `.ts` không cần build step.

---

**Xem tiếp:** [6-builtin-modules.md](./6-builtin-modules.md) — Node.js built-in modules.
