# Event-Driven Architecture — Event Sourcing, CQRS và Saga Pattern

> Event-Driven Architecture (EDA — Kiến Trúc Hướng Sự Kiện) dùng **events (sự kiện)** làm cơ chế giao tiếp chính giữa components. Chủ đề cover message brokers, **Event Sourcing (Lưu Trữ Sự Kiện)**, **CQRS (Command Query Responsibility Segregation — Phân Tách Trách Nhiệm Đọc/Ghi)**, và **Saga Pattern** cho distributed transactions.

## Mục Lục

1. [Event-Driven Architecture Là Gì](#event-driven-architecture-là-gì)
2. [Message Broker — Kafka, RabbitMQ, Redis](#message-broker--kafka-rabbitmq-redis)
3. [Event Design — Schema và Naming](#event-design--schema-và-naming)
4. [Event Sourcing](#event-sourcing)
5. [CQRS — Command Query Responsibility Segregation](#cqrs--command-query-responsibility-segregation)
6. [Saga Pattern — Distributed Transactions](#saga-pattern--distributed-transactions)
7. [Idempotency và Exactly-Once Semantics](#idempotency-và-exactly-once-semantics)
8. [Node.js Implementation](#nodejs-implementation)
9. [Best Practices](#best-practices)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Event-Driven Architecture Là Gì

```
┌──────────────┐    OrderCreated     ┌─────────────────┐
│ Order Service│ ──────────────────► │  Message Broker │
└──────────────┘                     │  (Kafka/RabbitMQ)│
                                     └────────┬────────┘
                          ┌──────────────────┼──────────────────┐
                          ▼                  ▼                  ▼
                 ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
                 │  Inventory   │  │ Notification │  │  Analytics   │
                 │   Service    │  │   Service    │  │   Service    │
                 └──────────────┘  └──────────────┘  └──────────────┘
```

**Event:** Một fact đã xảy ra trong hệ thống — immutable (bất biến), past tense naming.

| Loại | Mô Tả | Ví Dụ |
| ---- | ----- | ----- |
| **Domain Event** | Business fact | `OrderPlaced`, `PaymentCompleted` |
| **Integration Event** | Cross-service communication | `order.placed.v1` |
| **Command** | Request action (imperative) | `PlaceOrder`, `CancelSubscription` |

**EDA vs Request-Response:**

| | Request-Response | Event-Driven |
| --- | ---------------- | ------------ |
| Coupling | Tight — caller biết callee | Loose — publisher không biết subscribers |
| Timing | Synchronous | Async — eventual consistency |
| Failure | Caller handles | Retry, DLQ (Dead Letter Queue — Hàng Đợi Thư Chết) |
| Use case | CRUD, real-time queries | Notifications, analytics, decoupling |

---

## Message Broker — Kafka, RabbitMQ, Redis

| Broker | Strengths | Node.js Client |
| ------ | --------- | -------------- |
| **Redis Pub/Sub** | Simple, low latency, đã có Redis | `ioredis` |
| **RabbitMQ** | Routing flexibility, AMQP | `amqplib` |
| **Apache Kafka** | High throughput, event log, replay | `kafkajs` |
| **BullMQ** | Job queues trên Redis, Node-native | `bullmq` |

### Redis Pub/Sub — Simple Events

```typescript
import Redis from 'ioredis';

const publisher = new Redis(process.env.REDIS_URL);
const subscriber = new Redis(process.env.REDIS_URL);

// Publisher — Order Service
await publisher.publish('events:order:created', JSON.stringify({
  eventId: crypto.randomUUID(),
  orderId: order.id,
  customerId: order.customerId,
  total: order.total,
  occurredAt: new Date().toISOString(),
}));

// Subscriber — Notification Service
subscriber.subscribe('events:order:created');
subscriber.on('message', async (channel, message) => {
  const event = JSON.parse(message);
  await sendOrderConfirmationEmail(event.customerId, event.orderId);
});
```

### BullMQ — Reliable Job Processing

```typescript
import { Queue, Worker } from 'bullmq';

const orderQueue = new Queue('orders', { connection: { host: 'localhost', port: 6379 } });

// Producer
await orderQueue.add('order-created', {
  orderId: order.id,
  customerId: order.customerId,
}, {
  attempts: 3,
  backoff: { type: 'exponential', delay: 1000 },
});

// Consumer
const worker = new Worker('orders', async (job) => {
  if (job.name === 'order-created') {
    await processOrderCreated(job.data);
  }
}, { connection: { host: 'localhost', port: 6379 } });
```

### Kafka — Event Log với Replay

```typescript
import { Kafka } from 'kafkajs';

const kafka = new Kafka({ brokers: ['localhost:9092'] });
const producer = kafka.producer();

await producer.connect();
await producer.send({
  topic: 'order-events',
  messages: [{
    key: order.id,
    value: JSON.stringify({
      type: 'OrderCreated',
      data: { orderId: order.id, customerId: order.customerId },
      metadata: { correlationId, timestamp: Date.now() },
    }),
  }],
});
```

---

## Event Design — Schema và Naming

### Naming Conventions

```
✅ Past tense:     OrderCreated, PaymentFailed, UserRegistered
❌ Imperative:     CreateOrder, ProcessPayment

✅ Versioned:      order.created.v1
✅ Namespaced:     com.myapp.order.created
```

### Event Envelope — Standard Structure

```typescript
interface DomainEvent<T = unknown> {
  eventId: string;           // UUID — idempotency key
  eventType: string;         // 'OrderCreated'
  eventVersion: number;      // Schema version
  aggregateId: string;       // Entity ID
  aggregateType: string;     // 'Order'
  occurredAt: string;        // ISO 8601
  correlationId: string;     // Trace across services
  causationId?: string;      // Event that caused this event
  payload: T;
}

// Example
const event: DomainEvent<OrderCreatedPayload> = {
  eventId: '550e8400-e29b-41d4-a716-446655440000',
  eventType: 'OrderCreated',
  eventVersion: 1,
  aggregateId: 'order-123',
  aggregateType: 'Order',
  occurredAt: '2026-06-24T10:00:00Z',
  correlationId: 'req-abc-789',
  payload: {
    customerId: 'cust-456',
    items: [{ productId: 'prod-1', quantity: 2 }],
    total: 99.99,
  },
};
```

### Schema Evolution

- **Backward compatible:** Thêm optional fields — consumers cũ vẫn parse được
- **Version trong event:** `eventVersion: 2` — consumer handle multiple versions
- **Schema Registry:** Confluent Schema Registry cho Avro/Protobuf

---

## Event Sourcing

**Event Sourcing:** Lưu state changes dưới dạng sequence of events — không overwrite current state.

```
Traditional:  UPDATE orders SET status = 'SHIPPED' WHERE id = '123'

Event Sourcing:
  INSERT events: OrderCreated { orderId: 123, ... }
  INSERT events: OrderPaid { orderId: 123, amount: 99 }
  INSERT events: OrderShipped { orderId: 123, tracking: 'XYZ' }

Current state = replay all events for aggregate
```

```typescript
interface EventStore {
  append(streamId: string, events: DomainEvent[]): Promise<void>;
  getEvents(streamId: string, fromVersion?: number): Promise<DomainEvent[]>;
}

class OrderAggregate {
  private events: DomainEvent[] = [];
  status: OrderStatus = 'PENDING';

  static fromHistory(events: DomainEvent[]): OrderAggregate {
    const order = new OrderAggregate();
    for (const event of events) {
      order.apply(event);
    }
    return order;
  }

  ship(trackingNumber: string) {
    if (this.status !== 'PAID') throw new Error('Cannot ship unpaid order');
    this.raise({ eventType: 'OrderShipped', payload: { trackingNumber } });
  }

  private apply(event: DomainEvent) {
    switch (event.eventType) {
      case 'OrderCreated': this.status = 'PENDING'; break;
      case 'OrderPaid': this.status = 'PAID'; break;
      case 'OrderShipped': this.status = 'SHIPPED'; break;
    }
  }

  private raise(partial: Partial<DomainEvent>) {
    const event = { eventId: crypto.randomUUID(), ...partial } as DomainEvent;
    this.apply(event);
    this.events.push(event);
  }

  getUncommittedEvents() { return this.events; }
}
```

| Pros | Cons |
| ---- | ---- |
| Complete audit trail | Learning curve |
| Temporal queries ("state at time T") | Eventual consistency complexity |
| Replay/rebuild projections | Storage growth — cần snapshots |
| Debug — xem exact sequence | Not fit mọi use case |

---

## CQRS — Command Query Responsibility Segregation

Tách **write model (command side)** và **read model (query side)**:

```
                    ┌─────────────────┐
  POST /orders ────►│  Command Handler │──► Write DB (normalized)
                    └────────┬────────┘
                             │ publish OrderCreated
                             ▼
                    ┌─────────────────┐
                    │  Event Handler   │──► Read DB (denormalized)
                    └─────────────────┘
                             │
  GET /orders ───────────────┘ Query Read DB (optimized for reads)
```

```typescript
// Command side — write
class PlaceOrderHandler {
  async execute(command: PlaceOrderCommand) {
    const order = Order.create(command);
    await this.orderRepo.save(order);
    await this.eventBus.publish(new OrderCreatedEvent(order));
  }
}

// Query side — read (denormalized view)
class OrderListProjection {
  async onOrderCreated(event: OrderCreatedEvent) {
    await this.readDb.query(`
      INSERT INTO order_list_view (order_id, customer_name, total, status, created_at)
      VALUES ($1, $2, $3, $4, $5)
    `, [event.orderId, event.customerName, event.total, 'PENDING', event.occurredAt]);
  }
}

class GetOrdersQuery {
  async execute(customerId: string) {
    // Read from optimized view — no joins
    return this.readDb.query(
      'SELECT * FROM order_list_view WHERE customer_id = $1 ORDER BY created_at DESC',
      [customerId],
    );
  }
}
```

**Khi dùng CQRS:**
- Read/write patterns khác nhau significantly
- Complex queries cần denormalized views
- Kết hợp Event Sourcing

**Khi KHÔNG cần:** Simple CRUD — overhead không justify.

---

## Saga Pattern — Distributed Transactions

Không có ACID transaction across services — **Saga** coordinate qua local transactions + compensating actions.

### Choreography Saga — Event-Based

Mỗi service listen events và react — không orchestrator:

```
Order Service:  OrderCreated ──►
Inventory:      ◄── reserve stock ── InventoryReserved ──►
Payment:        ◄── charge ── PaymentCompleted ──►
Order Service:  ◄── mark paid

Failure path:
Payment:        PaymentFailed ──►
Inventory:      ◄── release stock (compensate)
Order Service:  ◄── mark cancelled
```

### Orchestration Saga — Central Coordinator

```typescript
class PlaceOrderSaga {
  async execute(orderId: string, items: OrderItem[]) {
    const sagaId = crypto.randomUUID();

    try {
      await this.inventory.reserve(items, sagaId);
      await this.payment.charge(orderId, sagaId);
      await this.order.confirm(orderId);
    } catch (err) {
      await this.compensate(sagaId, err);
      throw err;
    }
  }

  private async compensate(sagaId: string, error: unknown) {
    // Reverse completed steps in reverse order
    await this.payment.refund(sagaId).catch(logCompensationError);
    await this.inventory.release(sagaId).catch(logCompensationError);
    await this.order.cancel(sagaId).catch(logCompensationError);
  }
}
```

| Choreography | Orchestration |
| ------------ | ------------- |
| Decentralized — no single point | Central coordinator — easier to reason |
| Harder to track saga state | Explicit saga state machine |
| Less coupling to orchestrator | Orchestrator = potential bottleneck |

---

## Idempotency và Exactly-Once Semantics

Consumers **phải idempotent** — cùng event xử lý nhiều lần = cùng kết quả:

```typescript
async function handleOrderCreated(event: DomainEvent) {
  // Check đã xử lý chưa
  const processed = await redis.get(`processed:${event.eventId}`);
  if (processed) return; // Already handled — skip

  await sendEmail(event.payload.customerId, event.payload.orderId);

  // Mark processed — TTL 7 days
  await redis.set(`processed:${event.eventId}`, '1', 'EX', 604800);
}
```

| Delivery Guarantee | Ý Nghĩa | Implementation |
| ------------------ | ------- | -------------- |
| **At-most-once** | Có thể mất message | Fire and forget |
| **At-least-once** | Có thể duplicate | Retry + idempotent consumer |
| **Exactly-once** | Khó — thường "effectively once" | Idempotency + dedup + transactional outbox |

### Transactional Outbox Pattern

Đảm bảo DB write và event publish atomic:

```typescript
await prisma.$transaction(async (tx) => {
  await tx.order.create({ data: order });
  await tx.outboxEvent.create({
    data: {
      aggregateId: order.id,
      eventType: 'OrderCreated',
      payload: JSON.stringify(order),
    },
  });
});
// Separate process polls outbox → publishes to Kafka → marks sent
```

---

## Node.js Implementation

### EventEmitter cho In-Process Events

```typescript
import { EventEmitter } from 'node:events';

export const domainEvents = new EventEmitter();
domainEvents.setMaxListeners(20);

// Trong use case
await orderRepo.save(order);
domainEvents.emit('order:created', { orderId: order.id });
```

### NestJS Event Module

```typescript
@Injectable()
export class OrderService {
  constructor(private eventEmitter: EventEmitter2) {}

  async placeOrder(dto: PlaceOrderDto) {
    const order = await this.repo.save(dto);
    this.eventEmitter.emit('order.created', new OrderCreatedEvent(order));
    return order;
  }
}

@Injectable()
export class OrderNotificationHandler {
  @OnEvent('order.created')
  async handle(event: OrderCreatedEvent) {
    await this.emailService.sendConfirmation(event.order);
  }
}
```

---

## Best Practices

- **Start simple:** In-process EventEmitter → Redis Pub/Sub → Kafka khi cần
- **Design events first:** Contract between teams — treat như API
- **Monitor lag:** Consumer lag = backlog indicator
- **Dead Letter Queue:** Failed messages sau N retries → manual review
- **Correlation ID:** Trace event flow across services
- **Don't event everything:** CRUD đơn giản không cần Event Sourcing

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Gợi Ý Trả Lời |
| ------- | ------------- |
| Event-driven vs request-response? | EDA: loose coupling, async. RR: simple, sync, strong consistency |
| Event Sourcing là gì? | Store events not state — rebuild state by replay |
| CQRS benefits? | Optimize read/write independently — denormalized read models |
| Saga vs 2PC? | Saga: local txs + compensate. 2PC: blocking, not recommended distributed |
| Idempotent consumer? | Same event processed twice = same result — dedup by eventId |
| Kafka vs RabbitMQ? | Kafka: log, replay, high throughput. RabbitMQ: routing, traditional queue |
| Transactional outbox? | Atomic DB + event — poll outbox to broker |

---

**Tiếp theo:** [6-monorepo.md](./6-monorepo.md) — Turborepo, Nx, và pnpm workspaces
