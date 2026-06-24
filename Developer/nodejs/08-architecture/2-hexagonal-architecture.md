# Hexagonal Architecture — Ports & Adapters Pattern

> Hexagonal Architecture (Kiến Trúc Lục Giác), còn gọi là **Ports and Adapters (Cổng và Bộ Chuyển Đổi)**, đặt application core ở trung tâm, surround bởi các adapters kết nối với thế giới bên ngoài. Mục tiêu: **swap infrastructure dễ dàng** mà không đổi business logic.

## Mục Lục

1. [Hexagonal Architecture Là Gì](#hexagonal-architecture-là-gì)
2. [Ports — Primary vs Secondary](#ports--primary-vs-secondary)
3. [Adapters — Driving vs Driven](#adapters--driving-vs-driven)
4. [Cấu Trúc Thư Mục](#cấu-trúc-thư-mục)
5. [Ví Dụ: Order Service](#ví-dụ-order-service)
6. [Swapping Adapters](#swapping-adapters)
7. [Hexagonal vs Clean Architecture](#hexagonal-vs-clean-architecture)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Hexagonal Architecture Là Gì

Alistair Cockburn đặt tên "hexagonal" để nhấn mạnh: application có **nhiều sides (mặt)** kết nối với outside world — không chỉ "trên" (UI) và "dưới" (DB) như layered architecture.

```
                    ┌─────────────────────────────────┐
                    │         DRIVING ADAPTERS         │
                    │   REST API │ CLI │ Message Consumer│
                    └──────────────┬──────────────────┘
                                   │ calls
                    ┌──────────────▼──────────────────┐
                    │      PRIMARY PORTS (IN)        │
                    │   PlaceOrderPort, GetOrderPort  │
                    └──────────────┬──────────────────┘
                                   │
                    ┌──────────────▼──────────────────┐
                    │      APPLICATION CORE          │
                    │   Domain Logic + Use Cases     │
                    └──────────────┬──────────────────┘
                                   │
                    ┌──────────────▼──────────────────┐
                    │     SECONDARY PORTS (OUT)      │
                    │ OrderRepo │ PaymentGateway │ Email│
                    └──────────────┬──────────────────┘
                                   │ implemented by
                    ┌──────────────▼──────────────────┐
                    │        DRIVEN ADAPTERS          │
                    │  PostgreSQL │ Stripe │ SendGrid │
                    └─────────────────────────────────┘
```

**Application Core (Lõi Ứng Dụng):** Chứa domain logic — không biết HTTP, SQL, hay Stripe API.

**Port (Cổng):** Interface định nghĩa contract — primary (inbound) hoặc secondary (outbound).

**Adapter (Bộ Chuyển Đổi):** Implementation cụ thể kết nối port với technology.

---

## Ports — Primary vs Secondary

| Loại | Hướng | Ai Gọi Ai | Ví Dụ |
| ---- | ----- | --------- | ----- |
| **Primary Port (Driving Port)** | Inbound | Adapter → Core | `PlaceOrderUseCase`, `LoginService` |
| **Secondary Port (Driven Port)** | Outbound | Core → Adapter | `OrderRepository`, `PaymentGateway`, `EmailSender` |

### Primary Port — Application API

```typescript
// application/ports/in/place-order.port.ts
export interface PlaceOrderPort {
  execute(command: PlaceOrderCommand): Promise<PlaceOrderResult>;
}

export type PlaceOrderCommand = {
  customerId: string;
  items: Array<{ productId: string; quantity: number }>;
};

export type PlaceOrderResult =
  | { success: true; orderId: string }
  | { success: false; error: 'INSUFFICIENT_STOCK' | 'INVALID_CUSTOMER' };
```

### Secondary Port — Infrastructure Contract

```typescript
// application/ports/out/order-repository.port.ts
export interface OrderRepository {
  save(order: Order): Promise<void>;
  findById(id: string): Promise<Order | null>;
}

// application/ports/out/payment-gateway.port.ts
export interface PaymentGateway {
  charge(amount: Money, customerId: string): Promise<PaymentResult>;
}

// application/ports/out/notification.port.ts
export interface NotificationPort {
  sendOrderConfirmation(order: Order, email: string): Promise<void>;
}
```

---

## Adapters — Driving vs Driven

### Driving Adapter — REST API

```typescript
// adapters/in/http/order.controller.ts
import type { Request, Response } from 'express';
import type { PlaceOrderPort } from '../../../application/ports/in/place-order.port';

export class OrderHttpAdapter {
  constructor(private readonly placeOrder: PlaceOrderPort) {}

  async handlePlaceOrder(req: Request, res: Response) {
    const result = await this.placeOrder.execute({
      customerId: req.body.customerId,
      items: req.body.items,
    });

    if (!result.success) {
      const status = result.error === 'INSUFFICIENT_STOCK' ? 409 : 400;
      return res.status(status).json({ error: result.error });
    }

    return res.status(201).json({ orderId: result.orderId });
  }
}
```

### Driven Adapter — PostgreSQL Repository

```typescript
// adapters/out/persistence/postgres-order.repository.ts
import type { OrderRepository } from '../../../application/ports/out/order-repository.port';
import type { Pool } from 'pg';

export class PostgresOrderRepository implements OrderRepository {
  constructor(private readonly pool: Pool) {}

  async save(order: Order): Promise<void> {
    await this.pool.query(
      `INSERT INTO orders (id, customer_id, total, status, created_at)
       VALUES ($1, $2, $3, $4, $5)`,
      [order.id, order.customerId, order.total.amount, order.status, order.createdAt],
    );
  }

  async findById(id: string): Promise<Order | null> {
    const { rows } = await this.pool.query('SELECT * FROM orders WHERE id = $1', [id]);
    return rows[0] ? Order.fromRow(rows[0]) : null;
  }
}
```

### Driven Adapter — Stripe Payment

```typescript
// adapters/out/payment/stripe-payment.adapter.ts
import Stripe from 'stripe';
import type { PaymentGateway } from '../../../application/ports/out/payment-gateway.port';

export class StripePaymentAdapter implements PaymentGateway {
  constructor(private readonly stripe: Stripe) {}

  async charge(amount: Money, customerId: string): Promise<PaymentResult> {
    try {
      const intent = await this.stripe.paymentIntents.create({
        amount: amount.cents,
        currency: amount.currency,
        customer: customerId,
      });
      return { success: true, transactionId: intent.id };
    } catch (err) {
      return { success: false, reason: 'PAYMENT_FAILED' };
    }
  }
}
```

---

## Cấu Trúc Thư Mục

```
src/
├── domain/
│   ├── order.entity.ts
│   ├── money.vo.ts
│   └── order-status.enum.ts
│
├── application/
│   ├── ports/
│   │   ├── in/
│   │   │   └── place-order.port.ts
│   │   └── out/
│   │       ├── order-repository.port.ts
│   │       ├── payment-gateway.port.ts
│   │       └── notification.port.ts
│   └── services/
│       └── place-order.service.ts    # Implements PlaceOrderPort
│
├── adapters/
│   ├── in/
│   │   ├── http/
│   │   │   └── order.controller.ts
│   │   └── messaging/
│   │       └── order-created.consumer.ts
│   └── out/
│       ├── persistence/
│       │   └── postgres-order.repository.ts
│       ├── payment/
│       │   └── stripe-payment.adapter.ts
│       └── email/
│           └── sendgrid-notification.adapter.ts
│
└── bootstrap.ts                      # Wire adapters to ports
```

---

## Ví Dụ: Order Service

### Application Service — Core Logic

```typescript
// application/services/place-order.service.ts
export class PlaceOrderService implements PlaceOrderPort {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly inventory: InventoryPort,
    private readonly payment: PaymentGateway,
    private readonly notifications: NotificationPort,
  ) {}

  async execute(command: PlaceOrderCommand): Promise<PlaceOrderResult> {
    const stockCheck = await this.inventory.checkAvailability(command.items);
    if (!stockCheck.available) {
      return { success: false, error: 'INSUFFICIENT_STOCK' };
    }

    const order = Order.create(command.customerId, command.items);
    const paymentResult = await this.payment.charge(order.total, command.customerId);

    if (!paymentResult.success) {
      return { success: false, error: 'INVALID_CUSTOMER' };
    }

    await this.orderRepo.save(order);
    await this.inventory.reserve(command.items);
    await this.notifications.sendOrderConfirmation(order, order.customerEmail);

    return { success: true, orderId: order.id };
  }
}
```

---

## Swapping Adapters

Lợi ích chính của hexagonal: **đổi adapter mà không đổi core**.

| Scenario | Adapter Cũ | Adapter Mới | Core Thay Đổi? |
| -------- | ------------ | ----------- | -------------- |
| Đổi DB | PostgreSQL repo | MongoDB repo | Không |
| Đổi payment | Stripe | PayPal | Không |
| Thêm channel | REST API | + Kafka consumer | Thêm driving adapter |
| Test | Real DB | In-memory repo | Không |

```typescript
// bootstrap.ts — production
const orderRepo = new PostgresOrderRepository(pool);
const payment = new StripePaymentAdapter(stripe);

// bootstrap.test.ts — testing
const orderRepo = new InMemoryOrderRepository();
const payment = new FakePaymentGateway(); // always succeeds
```

### In-Memory Adapter cho Test

```typescript
// adapters/out/persistence/in-memory-order.repository.ts
export class InMemoryOrderRepository implements OrderRepository {
  private orders = new Map<string, Order>();

  async save(order: Order): Promise<void> {
    this.orders.set(order.id, order);
  }

  async findById(id: string): Promise<Order | null> {
    return this.orders.get(id) ?? null;
  }
}
```

---

## Hexagonal vs Clean Architecture

| Khía Cạnh | Clean Architecture | Hexagonal |
| --------- | ------------------ | --------- |
| Metaphor | Concentric circles (vòng tròn đồng tâm) | Hexagon với nhiều sides |
| Focus | Dependency direction | Port/adapter boundaries |
| Terminology | Entities, use cases, gateways | Primary/secondary ports, adapters |
| Thực tế | Gần như tương đương | Gần như tương đương |

**Trong Node.js projects:** Hai pattern thường **implement cùng cấu trúc folder** — chọn terminology team quen thuộc hơn.

---

## Best Practices

### Naming Conventions

- Ports: `*Port`, `*Repository`, `*Gateway` (interface)
- Adapters: `*Adapter`, `*Repository` (implementation), `Postgres*`, `Stripe*`
- Application services implement primary ports

### Multiple Driving Adapters

Một use case có thể được trigger từ nhiều nguồn:

```
REST POST /orders  ──┐
                     ├──► PlaceOrderService ──► OrderRepository
Kafka order.request ─┘
```

Mỗi driving adapter map input format → command DTO → gọi cùng primary port.

### Anti-patterns

| Anti-pattern | Vấn Đề |
| ------------ | ------ |
| Port leak framework types | `findById(): Promise<PrismaUser>` thay vì domain entity |
| Adapter chứa business logic | Validation trong controller thay vì core |
| Quá nhiều ports | Interface cho mọi function — YAGNI |

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Gợi Ý Trả Lời |
| ------- | ------------- |
| Primary vs secondary port? | Primary: inbound (UI/API gọi app). Secondary: outbound (app gọi DB/external) |
| Driving vs driven adapter? | Driving: trigger application. Driven: implement outbound ports |
| Lợi ích chính? | Testability, technology independence, swap infrastructure |
| Khác layered architecture? | Layered: top-down dependency. Hexagonal: core ở giữa, symmetric connections |
| Khi nào dùng? | Nhiều integrations, cần swap tech, hoặc multiple entry points (HTTP + queue) |

---

**Tiếp theo:** [3-design-patterns.md](./3-design-patterns.md) — Design patterns phổ biến trong Node.js
