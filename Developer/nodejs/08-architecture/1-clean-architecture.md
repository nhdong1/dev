# Clean Architecture — Layers, Dependency Rule và Use Cases

> Clean Architecture (Kiến Trúc Sạch) của Uncle Bob tổ chức code thành các vòng tròn đồng tâm — business logic ở trung tâm, framework và database ở ngoài cùng. Mục tiêu: **testability (khả năng kiểm thử)**, **independence from frameworks**, và **maintainability (dễ bảo trì)**.

## Mục Lục

1. [Clean Architecture Là Gì](#clean-architecture-là-gì)
2. [Các Layer và Trách Nhiệm](#các-layer-và-trách-nhiệm)
3. [Dependency Rule — Quy Tắc Phụ Thuộc](#dependency-rule--quy-tắc-phụ-thuộc)
4. [Cấu Trúc Thư Mục Node.js](#cấu-trúc-thư-mục-nodejs)
5. [Use Case — Application Layer](#use-case--application-layer)
6. [Ví Dụ Hoàn Chỉnh: Create User](#ví-dụ-hoàn-chỉnh-create-user)
7. [So Sánh với MVC Truyền Thống](#so-sánh-với-mvc-truyền-thống)
8. [Best Practices và Pitfalls](#best-practices-và-pitfalls)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Clean Architecture Là Gì

```
                    ┌─────────────────────────────────────┐
                    │         FRAMEWORKS & DRIVERS         │
                    │   Express, Fastify, PostgreSQL, Redis │
                    └──────────────────┬──────────────────┘
                                       │ depends on ▼
                    ┌──────────────────▼──────────────────┐
                    │      INTERFACE ADAPTERS              │
                    │  Controllers, Presenters, Gateways   │
                    └──────────────────┬──────────────────┘
                                       │ depends on ▼
                    ┌──────────────────▼──────────────────┐
                    │       APPLICATION (USE CASES)        │
                    │   CreateUser, PlaceOrder, Login      │
                    └──────────────────┬──────────────────┘
                                       │ depends on ▼
                    ┌──────────────────▼──────────────────┐
                    │            ENTITIES                  │
                    │   User, Order, Business Rules        │
                    └─────────────────────────────────────┘

    ◄─── Dependencies chỉ point INWARD (vào trong) ───►
```

**Entities (Thực Thể):** Objects chứa enterprise business rules — logic cốt lõi không phụ thuộc application cụ thể.

**Use Cases (Trường Hợp Sử Dụng):** Application-specific business rules — orchestrate flow giữa entities và external systems.

**Interface Adapters (Bộ Chuyển Đổi Giao Diện):** Convert data giữa use cases và external world (HTTP request → DTO → entity).

**Frameworks & Drivers:** Express routes, Prisma client, Redis — details implementation.

---

## Các Layer và Trách Nhiệm

| Layer | Trách Nhiệm | Không Được Chứa |
| ----- | ----------- | --------------- |
| **Domain / Entities** | Business rules, validation cốt lõi | HTTP, DB queries, framework imports |
| **Application / Use Cases** | Orchestrate workflow, transaction boundaries | Express `req`/`res`, SQL strings |
| **Infrastructure** | DB, cache, email, third-party APIs | Business logic |
| **Presentation** | HTTP handlers, CLI, GraphQL resolvers | Direct DB access |

---

## Dependency Rule — Quy Tắc Phụ Thuộc

> **Source code dependencies chỉ được point inward.** Inner circles không biết gì về outer circles.

### Dependency Inversion Principle (DIP — Nguyên Tắc Đảo Ngược Phụ Thuộc)

Thay vì use case import trực tiếp `PrismaClient`, define **interface (port)** ở inner layer, implement ở outer layer:

```typescript
// application/ports/user-repository.port.ts — INNER (interface)
export interface UserRepository {
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<void>;
}

// infrastructure/persistence/prisma-user.repository.ts — OUTER (implementation)
import { PrismaClient } from '@prisma/client';
import type { UserRepository } from '../../application/ports/user-repository.port';

export class PrismaUserRepository implements UserRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findByEmail(email: string) {
    const row = await this.prisma.user.findUnique({ where: { email } });
    return row ? User.fromPersistence(row) : null;
  }

  async save(user: User) {
    await this.prisma.user.upsert({
      where: { id: user.id },
      create: user.toPersistence(),
      update: user.toPersistence(),
    });
  }
}
```

Use case chỉ biết interface:

```typescript
// application/use-cases/create-user.use-case.ts
export class CreateUserUseCase {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly hasher: PasswordHasher,
  ) {}

  async execute(input: CreateUserInput): Promise<Result<User, CreateUserError>> {
    const existing = await this.userRepo.findByEmail(input.email);
    if (existing) return err('EMAIL_ALREADY_EXISTS');

    const user = User.create({ email: input.email, name: input.name });
    user.setPasswordHash(await this.hasher.hash(input.password));

    await this.userRepo.save(user);
    return ok(user);
  }
}
```

---

## Cấu Trúc Thư Mục Node.js

```
src/
├── domain/
│   ├── entities/
│   │   └── user.entity.ts
│   ├── value-objects/
│   │   └── email.vo.ts
│   └── errors/
│       └── domain.errors.ts
│
├── application/
│   ├── ports/
│   │   ├── user-repository.port.ts
│   │   └── password-hasher.port.ts
│   ├── use-cases/
│   │   ├── create-user.use-case.ts
│   │   └── login.use-case.ts
│   └── dto/
│       └── create-user.dto.ts
│
├── infrastructure/
│   ├── persistence/
│   │   └── prisma-user.repository.ts
│   ├── security/
│   │   └── bcrypt-password-hasher.ts
│   └── di/
│       └── container.ts          # Manual DI hoặc tsyringe/inversify
│
├── presentation/
│   ├── http/
│   │   ├── routes/
│   │   │   └── user.routes.ts
│   │   ├── controllers/
│   │   │   └── user.controller.ts
│   │   └── middleware/
│   │       └── error-handler.middleware.ts
│   └── app.ts
│
└── main.ts                       # Bootstrap, wire dependencies
```

---

## Use Case — Application Layer

**Use case** = một hành động business cụ thể mà user/system thực hiện. Mỗi use case:

1. Nhận input (DTO — Data Transfer Object)
2. Validate qua domain rules
3. Gọi ports (repository, services)
4. Trả output hoặc error

```typescript
// domain/entities/user.entity.ts
export class User {
  private constructor(
    readonly id: string,
    readonly email: string,
    readonly name: string,
    private passwordHash: string,
  ) {}

  static create(props: { email: string; name: string }): User {
    if (!Email.isValid(props.email)) {
      throw new DomainError('INVALID_EMAIL');
    }
    return new User(crypto.randomUUID(), props.email, props.name, '');
  }

  setPasswordHash(hash: string): void {
    this.passwordHash = hash;
  }

  toPersistence() {
    return { id: this.id, email: this.email, name: this.name, passwordHash: this.passwordHash };
  }

  static fromPersistence(row: { id: string; email: string; name: string; passwordHash: string }) {
    return new User(row.id, row.email, row.name, row.passwordHash);
  }
}
```

### Single Responsibility

Mỗi use case = một class/function — dễ test, dễ đọc:

| Use Case | Input | Output |
| -------- | ----- | ------ |
| `CreateUserUseCase` | email, password, name | User hoặc error |
| `LoginUseCase` | email, password | tokens hoặc error |
| `GetUserProfileUseCase` | userId | UserProfile DTO |

---

## Ví Dụ Hoàn Chỉnh: Create User

### Presentation Layer — Controller mỏng

```typescript
// presentation/http/controllers/user.controller.ts
import type { Request, Response, NextFunction } from 'express';
import type { CreateUserUseCase } from '../../../application/use-cases/create-user.use-case';

export class UserController {
  constructor(private readonly createUser: CreateUserUseCase) {}

  async register(req: Request, res: Response, next: NextFunction) {
    const result = await this.createUser.execute({
      email: req.body.email,
      password: req.body.password,
      name: req.body.name,
    });

    if (result.isErr()) {
      return res.status(409).json({ error: result.error });
    }

    return res.status(201).json({
      id: result.value.id,
      email: result.value.email,
      name: result.value.name,
    });
  }
}
```

### Wiring — Composition Root

```typescript
// main.ts — nơi DUY NHẤT biết concrete implementations
import express from 'express';
import { PrismaClient } from '@prisma/client';
import { PrismaUserRepository } from './infrastructure/persistence/prisma-user.repository';
import { BcryptPasswordHasher } from './infrastructure/security/bcrypt-password-hasher';
import { CreateUserUseCase } from './application/use-cases/create-user.use-case';
import { UserController } from './presentation/http/controllers/user.controller';

const prisma = new PrismaClient();
const userRepo = new PrismaUserRepository(prisma);
const hasher = new BcryptPasswordHasher();
const createUser = new CreateUserUseCase(userRepo, hasher);
const userController = new UserController(createUser);

const app = express();
app.use(express.json());
app.post('/users', (req, res, next) => userController.register(req, res, next));
```

### Unit Test — Không Cần Database

```typescript
// create-user.use-case.test.ts
describe('CreateUserUseCase', () => {
  it('returns error when email exists', async () => {
    const userRepo: UserRepository = {
      findByEmail: async () => User.create({ email: 'a@b.com', name: 'A' }),
      save: async () => {},
    };
    const hasher: PasswordHasher = { hash: async (p) => `hash:${p}` };

    const useCase = new CreateUserUseCase(userRepo, hasher);
    const result = await useCase.execute({
      email: 'a@b.com',
      password: 'secret',
      name: 'B',
    });

    expect(result.isErr()).toBe(true);
    expect(result.error).toBe('EMAIL_ALREADY_EXISTS');
  });
});
```

---

## So Sánh với MVC Truyền Thống

| Khía Cạnh | MVC "Fat Controller" | Clean Architecture |
| --------- | -------------------- | ------------------ |
| Business logic | Trong route handler | Trong use case + entity |
| DB access | Trực tiếp Prisma trong controller | Qua repository port |
| Testability | Cần HTTP mock | Test use case với fake repo |
| Framework coupling | Cao — khó đổi Express → Fastify | Thấp — chỉ presentation layer đổi |
| Boilerplate | Ít ban đầu | Nhiều hơn — trade-off cho maintainability |

```
MVC Fat Controller:
  Request → [Controller + Business Logic + DB Query] → Response

Clean Architecture:
  Request → Controller → UseCase → Entity
                              ↓
                         Repository (port)
                              ↓
                         Prisma (adapter) → DB
```

---

## Best Practices và Pitfalls

### Nên Làm

- **Composition root** tập trung ở `main.ts` hoặc DI container
- **DTOs** tách biệt HTTP schema và domain entities
- **Domain errors** typed — không throw generic `Error`
- Bắt đầu với 3-4 layers — không over-engineer từ ngày 1

### Tránh

| Pitfall | Vấn Đề | Giải Pháp |
| ------- | ------ | --------- |
| Anemic domain model | Entity chỉ getters/setters, logic ở service | Đưa validation vào entity |
| Leaky abstractions | Repository trả Prisma types | Map sang domain entities |
| God use case | Một use case làm quá nhiều việc | Split theo single action |
| Over-abstraction | Interface cho mọi thứ dù 1 implementation | YAGNI — abstract khi cần swap/test |

### Khi Nào KHÔNG Cần Full Clean Architecture

- MVP/prototype cần ship trong vài ngày
- CRUD đơn giản không có business rules phức tạp
- Team 1-2 người, codebase nhỏ

**Pragmatic approach:** Áp dụng dependency rule và use cases — không bắt buộc mọi entity phải rich domain model.

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Gợi Ý Trả Lời |
| ------- | ------------- |
| Dependency rule là gì? | Dependencies chỉ point inward — inner không import outer |
| Use case vs service? | Use case = một action cụ thể. Service có thể chứa nhiều operations liên quan |
| DIP trong Clean Architecture? | High-level define interfaces, low-level implement — invert dependency direction |
| Làm sao test use case? | Mock ports (fake repository), không cần DB hay HTTP |
| Clean Architecture vs DDD? | Clean Architecture là structure. DDD là approach model domain — thường kết hợp |
| NestJS có Clean Architecture không? | NestJS cung cấp modules/DI — có thể map layers vào modules nếu discipline |

---

**Tiếp theo:** [2-hexagonal-architecture.md](./2-hexagonal-architecture.md) — Ports & Adapters pattern
```
