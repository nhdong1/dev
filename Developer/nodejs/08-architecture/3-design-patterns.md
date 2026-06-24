# Design Patterns — Singleton, Factory, Observer, Repository

> Design Patterns (Mẫu Thiết Kế) là các giải pháp tái sử dụng cho problems phổ biến trong software design. Chủ đề này cover các patterns thường gặp nhất trong Node.js Backend — khi nào dùng, khi nào tránh, và implementation TypeScript thực tế.

## Mục Lục

1. [Tổng Quan Design Patterns](#tổng-quan-design-patterns)
2. [Creational Patterns — Mẫu Khởi Tạo](#creational-patterns--mẫu-khởi-tạo)
3. [Structural Patterns — Mẫu Cấu Trúc](#structural-patterns--mẫu-cấu-trúc)
4. [Behavioral Patterns — Mẫu Hành Vi](#behavioral-patterns--mẫu-hành-vi)
5. [Repository Pattern — Chi Tiết](#repository-pattern--chi-tiết)
6. [Dependency Injection — Pattern Node.js](#dependency-injection--pattern-nodejs)
7. [Anti-patterns Phổ Biến](#anti-patterns-phổ-biến)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Design Patterns

Patterns được nhóm theo **GoF (Gang of Four)** classification:

| Nhóm | Mục Đích | Patterns Phổ Biến trong Node.js |
| ---- | -------- | ------------------------------ |
| **Creational** | Cách tạo objects | Singleton, Factory, Builder |
| **Structural** | Cách compose objects | Adapter, Decorator, Facade |
| **Behavioral** | Communication giữa objects | Observer, Strategy, Command |

**Lưu ý Node.js:** JavaScript/TypeScript có first-class functions và module system — nhiều patterns đơn giản hơn so với Java/C#. Không force pattern khi plain functions đủ.

---

## Creational Patterns — Mẫu Khởi Tạo

### Singleton — Một Instance Duy Nhất

Đảm bảo class chỉ có một instance — phổ biến cho connection pools, config, logger.

```typescript
// infrastructure/database/connection.singleton.ts
class DatabaseConnection {
  private static instance: DatabaseConnection;
  private pool: Pool;

  private constructor() {
    this.pool = new Pool({ connectionString: process.env.DATABASE_URL });
  }

  static getInstance(): DatabaseConnection {
    if (!DatabaseConnection.instance) {
      DatabaseConnection.instance = new DatabaseConnection();
    }
    return DatabaseConnection.instance;
  }

  getPool(): Pool {
    return this.pool;
  }
}

export const db = DatabaseConnection.getInstance();
```

**Node.js thực tế:** Module cache (`require`/`import`) đã behave như singleton — export một instance thường đủ:

```typescript
// ✅ Idiomatic Node.js — module = natural singleton
import { Pool } from 'pg';

export const pool = new Pool({ connectionString: process.env.DATABASE_URL });
```

| Dùng Singleton Khi | Tránh Khi |
| ------------------ | --------- |
| DB pool, Redis client | Cần test với mock instances |
| App config sau validate | Hidden global state gây coupling |
| Logger instance | Multi-tenant cần isolated instances |

### Factory — Tạo Object Không Expose Constructor

```typescript
// domain/notifications/notification.factory.ts
type NotificationChannel = 'email' | 'sms' | 'push';

interface NotificationSender {
  send(to: string, message: string): Promise<void>;
}

class EmailSender implements NotificationSender {
  async send(to: string, message: string) { /* SendGrid */ }
}

class SmsSender implements NotificationSender {
  async send(to: string, message: string) { /* Twilio */ }
}

export function createNotificationSender(channel: NotificationChannel): NotificationSender {
  switch (channel) {
    case 'email': return new EmailSender();
    case 'sms': return new SmsSender();
    case 'push': return new PushSender();
    default: throw new Error(`Unknown channel: ${channel}`);
  }
}
```

**Factory Method vs Abstract Factory:**
- **Factory Method:** Một method tạo object based on input
- **Abstract Factory:** Family of related objects (ví dụ: UI components cho platform khác nhau)

### Builder — Construct Complex Objects Step-by-Step

Hữu ích khi object có nhiều optional parameters:

```typescript
class QueryBuilder {
  private filters: string[] = [];
  private limitValue = 20;
  private offsetValue = 0;

  where(field: string, value: unknown): this {
    this.filters.push(`${field} = $${this.filters.length + 1}`);
    return this;
  }

  limit(n: number): this {
    this.limitValue = n;
    return this;
  }

  offset(n: number): this {
    this.offsetValue = n;
    return this;
  }

  build(): { sql: string; params: unknown[] } {
    const where = this.filters.length ? `WHERE ${this.filters.join(' AND ')}` : '';
    return {
      sql: `SELECT * FROM users ${where} LIMIT ${this.limitValue} OFFSET ${this.offsetValue}`,
      params: [],
    };
  }
}

// Usage
const query = new QueryBuilder().where('status', 'active').limit(10).build();
```

---

## Structural Patterns — Mẫu Cấu Trúc

### Adapter — Convert Interface

Đã cover trong [Hexagonal Architecture](./2-hexagonal-architecture.md) — wrap third-party API behind port interface.

```typescript
// Adapter: legacy callback API → Promise
function promisify<T>(fn: (...args: [...unknown[], (err: Error | null, result: T) => void]) => void) {
  return (...args: unknown[]): Promise<T> =>
    new Promise((resolve, reject) => {
      fn(...args, (err, result) => (err ? reject(err) : resolve(result)));
    });
}
```

### Decorator — Add Behavior Without Modifying Class

Phổ biến trong NestJS (`@Injectable`, `@UseGuards`) và Express middleware:

```typescript
// Decorator pattern via composition
function withLogging<T extends (...args: unknown[]) => Promise<unknown>>(fn: T, name: string): T {
  return (async (...args: unknown[]) => {
    const start = Date.now();
    try {
      const result = await fn(...args);
      console.log(`${name} completed in ${Date.now() - start}ms`);
      return result;
    } catch (err) {
      console.error(`${name} failed after ${Date.now() - start}ms`, err);
      throw err;
    }
  }) as T;
}

const fetchUser = withLogging(async (id: string) => userRepo.findById(id), 'fetchUser');
```

### Facade — Simplified Interface cho Subsystem Phức Tạp

```typescript
// Facade che giấu complexity của multiple services
class OrderFacade {
  constructor(
    private readonly orders: OrderService,
    private readonly inventory: InventoryService,
    private readonly payments: PaymentService,
    private readonly notifications: NotificationService,
  ) {}

  async placeOrder(input: PlaceOrderInput): Promise<OrderResult> {
    // Orchestrate — client chỉ gọi một method
    await this.inventory.reserve(input.items);
    const payment = await this.payments.charge(input);
    const order = await this.orders.create(input, payment.id);
    await this.notifications.sendConfirmation(order);
    return { orderId: order.id };
  }
}
```

---

## Behavioral Patterns — Mẫu Hành Vi

### Observer — Publish/Subscribe Events

Node.js `EventEmitter` là implementation native của Observer pattern:

```typescript
import { EventEmitter } from 'node:events';

// Domain events
interface OrderEvents {
  'order:created': { orderId: string; customerId: string };
  'order:shipped': { orderId: string; trackingNumber: string };
}

class OrderEventBus extends EventEmitter {
  emit<K extends keyof OrderEvents>(event: K, payload: OrderEvents[K]): boolean {
    return super.emit(event, payload);
  }

  on<K extends keyof OrderEvents>(event: K, listener: (payload: OrderEvents[K]) => void): this {
    return super.on(event, listener);
  }
}

const eventBus = new OrderEventBus();

// Subscriber — decoupled handlers
eventBus.on('order:created', async ({ orderId, customerId }) => {
  await emailService.sendOrderConfirmation(customerId, orderId);
});

eventBus.on('order:created', async ({ orderId }) => {
  await analytics.track('order_created', { orderId });
});
```

**Production:** Dùng message broker (Redis, RabbitMQ, Kafka) khi cần durability và cross-service events.

### Strategy — Swap Algorithm at Runtime

```typescript
interface PricingStrategy {
  calculate(basePrice: number, context: PricingContext): number;
}

class StandardPricing implements PricingStrategy {
  calculate(basePrice: number) { return basePrice; }
}

class VipPricing implements PricingStrategy {
  calculate(basePrice: number) { return basePrice * 0.85; } // 15% discount
}

class SeasonalPricing implements PricingStrategy {
  calculate(basePrice: number, ctx: PricingContext) {
    return ctx.isHoliday ? basePrice * 0.9 : basePrice;
  }
}

class CheckoutService {
  constructor(private strategy: PricingStrategy) {}

  setStrategy(strategy: PricingStrategy) {
    this.strategy = strategy;
  }

  getTotal(basePrice: number, ctx: PricingContext) {
    return this.strategy.calculate(basePrice, ctx);
  }
}
```

### Command — Encapsulate Request as Object

Hữu ích cho job queues, undo operations, audit logs:

```typescript
interface Command {
  execute(): Promise<void>;
  undo?(): Promise<void>;
}

class CreateUserCommand implements Command {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly data: CreateUserInput,
    private savedUserId?: string,
  ) {}

  async execute() {
    const user = User.create(this.data);
    await this.userRepo.save(user);
    this.savedUserId = user.id;
  }

  async undo() {
    if (this.savedUserId) {
      await this.userRepo.delete(this.savedUserId);
    }
  }
}

// Queue commands via BullMQ
await commandQueue.add('create-user', { email, name, password });
```

---

## Repository Pattern — Chi Tiết

**Repository** abstract data access — domain layer không biết SQL vs MongoDB.

```typescript
// Generic repository interface
export interface Repository<T, ID = string> {
  findById(id: ID): Promise<T | null>;
  findAll(criteria?: Partial<T>): Promise<T[]>;
  save(entity: T): Promise<void>;
  delete(id: ID): Promise<void>;
}

// Domain-specific extension
export interface UserRepository extends Repository<User> {
  findByEmail(email: string): Promise<User | null>;
  findActiveUsers(limit: number): Promise<User[]>;
}
```

### Unit of Work — Transaction Boundary

Khi nhiều repositories cần cùng transaction:

```typescript
export interface UnitOfWork {
  users: UserRepository;
  orders: OrderRepository;
  commit(): Promise<void>;
  rollback(): Promise<void>;
}

// Prisma implementation
export class PrismaUnitOfWork implements UnitOfWork {
  private tx: PrismaTransaction;

  constructor(prisma: PrismaClient) {
    this.tx = prisma.$transaction.bind(prisma);
  }

  async commit() {
    await this.tx(async (tx) => {
      // operations on tx.users, tx.orders
    });
  }
}
```

---

## Dependency Injection — Pattern Node.js

### Manual DI (Composition Root)

```typescript
// bootstrap.ts
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const userRepo = new PostgresUserRepository(pool);
const createUser = new CreateUserUseCase(userRepo, new BcryptHasher());
export const app = createApp({ createUser });
```

### NestJS DI Container

```typescript
@Injectable()
export class CreateUserUseCase {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly hasher: PasswordHasher,
  ) {}
}

@Module({
  providers: [
    CreateUserUseCase,
    { provide: 'UserRepository', useClass: PrismaUserRepository },
    { provide: 'PasswordHasher', useClass: BcryptHasher },
  ],
})
export class UserModule {}
```

### tsyringe — Lightweight DI cho Plain Node.js

```typescript
import { container, injectable, inject } from 'tsyringe';

@injectable()
class CreateUserUseCase {
  constructor(
    @inject('UserRepository') private userRepo: UserRepository,
  ) {}
}

container.register('UserRepository', { useClass: PrismaUserRepository });
```

| Approach | Phù Hợp |
| -------- | ------- |
| Manual DI | Small/medium apps, explicit wiring |
| NestJS DI | Enterprise, decorators, module system |
| tsyringe/inversify | Clean/Hexagonal without full NestJS |

---

## Anti-patterns Phổ Biến

| Anti-pattern | Mô Tả | Thay Thế |
| ------------ | ----- | -------- |
| **God Object** | Một class làm mọi thứ | Split responsibilities, use cases |
| **Singleton abuse** | Global state everywhere | Module exports, DI |
| **Pattern for pattern's sake** | Factory cho 1 implementation | YAGNI — plain constructor |
| **Anemic Repository** | Repo chỉ CRUD, logic ở service rải rác | Rich domain + focused repos |
| **Callback hell** | Nested callbacks | Promises, async/await |

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Gợi Ý Trả Lời |
| ------- | ------------- |
| Singleton trong Node.js? | Module cache = natural singleton. Explicit singleton class thường thừa |
| Repository vs DAO? | Repository: domain-centric, aggregate roots. DAO: table-centric CRUD |
| Observer vs Pub/Sub? | Observer: in-process (EventEmitter). Pub/Sub: distributed (Kafka) |
| Strategy vs Factory? | Strategy: swap behavior. Factory: create objects |
| Khi nào KHÔNG dùng pattern? | Simple CRUD, MVP — patterns add complexity without benefit |
| DI benefits? | Testability, loose coupling, swap implementations |

---

**Tiếp theo:** [4-microservices.md](./4-microservices.md) — Service decomposition và API Gateway
