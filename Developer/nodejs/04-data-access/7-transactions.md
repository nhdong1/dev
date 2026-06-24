# Database Transactions — ACID và Optimistic Locking

> Transactions (Giao Dịch) đảm bảo data consistency khi nhiều thao tác database phải thành công hoặc thất bại cùng nhau. Hiểu ACID và locking strategies là kỹ năng bắt buộc cho Backend production.

## Mục Lục

1. [ACID Là Gì](#acid-là-gì)
2. [Isolation Levels (Mức Cô Lập)](#isolation-levels-mức-cô-lập)
3. [Transactions với `pg`](#transactions-với-pg)
4. [Transactions với Prisma](#transactions-với-prisma)
5. [Transactions với TypeORM](#transactions-với-typeorm)
6. [Optimistic vs Pessimistic Locking](#optimistic-vs-pessimistic-locking)
7. [Distributed Transactions](#distributed-transactions)
8. [Common Pitfalls](#common-pitfalls)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## ACID Là Gì

| Property | Mô Tả | Ví Dụ |
| -------- | ----- | ----- |
| **Atomicity (Tính Nguyên Tử)** | Tất cả operations thành công hoặc tất cả rollback | Chuyển tiền: debit + credit cùng commit hoặc cùng rollback |
| **Consistency (Tính Nhất Quán)** | DB chuyển từ valid state sang valid state | Balance không âm sau transaction |
| **Isolation (Tính Cô Lập)** | Concurrent transactions không interfere | Hai user cùng mua item cuối — chỉ một thành công |
| **Durability (Tính Bền Vững)** | Committed data survive crash | Sau COMMIT, data persist dù server restart |

---

## Isolation Levels (Mức Cô Lập)

PostgreSQL hỗ trợ 4 isolation levels:

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
| ----- | ---------- | ------------------- | ------------ | ----------- |
| **READ UNCOMMITTED** | Có thể | Có thể | Có thể | Cao nhất |
| **READ COMMITTED** (default PG) | Không | Có thể | Có thể | Cao |
| **REPEATABLE READ** | Không | Không | Không* | Trung bình |
| **SERIALIZABLE** | Không | Không | Không | Thấp nhất |

*PostgreSQL REPEATABLE READ ngăn phantom reads nhờ MVCC (Multi-Version Concurrency Control).

### Anomalies Giải Thích

```
Dirty Read:        Đọc data chưa commit của transaction khác
Non-Repeatable:    Cùng query 2 lần trong transaction → kết quả khác
Phantom Read:      Cùng query 2 lần → số rows khác (rows mới xuất hiện)
```

---

## Transactions với `pg`

```javascript
const { pool } = require('./db');

async function transferFunds(fromId, toId, amount) {
  const client = await pool.connect();

  try {
    await client.query('BEGIN');

    // Set isolation level nếu cần
    await client.query('SET TRANSACTION ISOLATION LEVEL SERIALIZABLE');

    const { rows: [fromAccount] } = await client.query(
      'SELECT balance FROM accounts WHERE id = $1 FOR UPDATE', // Row lock
      [fromId]
    );

    if (!fromAccount || fromAccount.balance < amount) {
      throw new Error('Insufficient funds');
    }

    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
      [amount, fromId]
    );

    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amount, toId]
    );

    await client.query(
      'INSERT INTO transactions (from_id, to_id, amount) VALUES ($1, $2, $3)',
      [fromId, toId, amount]
    );

    await client.query('COMMIT');
    return { success: true };
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}
```

### `FOR UPDATE` — Pessimistic Row Lock

```sql
SELECT * FROM products WHERE id = $1 FOR UPDATE;
-- Lock row cho đến khi transaction commit/rollback
-- Ngăn concurrent updates trên cùng row
```

---

## Transactions với Prisma

### Interactive Transaction

```typescript
const result = await prisma.$transaction(
  async (tx) => {
    const product = await tx.product.update({
      where: { id: productId },
      data: { stock: { decrement: quantity } },
    });

    if (product.stock < 0) {
      throw new Error('Out of stock');
    }

    const order = await tx.order.create({
      data: {
        userId,
        items: {
          create: [{ productId, quantity, price: product.price }],
        },
        total: product.price * quantity,
      },
    });

    return order;
  },
  {
    maxWait: 5000,      // Chờ acquire connection
    timeout: 10000,     // Max transaction duration
    isolationLevel: 'Serializable',
  }
);
```

### Sequential Transaction

```typescript
await prisma.$transaction([
  prisma.account.update({
    where: { id: fromId },
    data: { balance: { decrement: amount } },
  }),
  prisma.account.update({
    where: { id: toId },
    data: { balance: { increment: amount } },
  }),
]);
```

---

## Transactions với TypeORM

```typescript
await AppDataSource.transaction(async (manager) => {
  const accountRepo = manager.getRepository(Account);

  const fromAccount = await accountRepo.findOne({
    where: { id: fromId },
    lock: { mode: 'pessimistic_write' },
  });

  if (!fromAccount || fromAccount.balance < amount) {
    throw new Error('Insufficient funds');
  }

  fromAccount.balance -= amount;
  await accountRepo.save(fromAccount);

  await accountRepo.increment({ id: toId }, 'balance', amount);
});
```

---

## Optimistic vs Pessimistic Locking

### Pessimistic Locking

Lock row ngay khi đọc — ngăn concurrent modifications:

```typescript
// Prisma — raw query cho FOR UPDATE
await prisma.$queryRaw`
  SELECT * FROM products WHERE id = ${id} FOR UPDATE
`;

// TypeORM
lock: { mode: 'pessimistic_write' }

// Use case: High contention — nhiều users cùng compete resource (flash sale)
```

### Optimistic Locking

Không lock khi đọc — verify version khi write:

```prisma
model Product {
  id      String @id
  name    String
  stock   Int
  version Int    @default(0) // Version column
}
```

```typescript
async function purchaseWithOptimisticLock(productId: string, quantity: number) {
  const product = await prisma.product.findUnique({ where: { id: productId } });

  if (!product || product.stock < quantity) {
    throw new Error('Out of stock');
  }

  try {
    await prisma.product.update({
      where: {
        id: productId,
        version: product.version, // Chỉ update nếu version chưa đổi
      },
      data: {
        stock: { decrement: quantity },
        version: { increment: 1 },
      },
    });
  } catch (err) {
    if (err.code === 'P2025') {
      // Record not found — version đã thay đổi
      throw new Error('Concurrent modification — please retry');
    }
    throw err;
  }
}
```

### So Sánh

| | Pessimistic | Optimistic |
| - | ----------- | ---------- |
| **Lock timing** | Khi đọc | Khi ghi (check version) |
| **Contention** | High contention OK | Low contention tốt hơn |
| **Deadlock risk** | Có | Không |
| **Retry logic** | Không cần | Cần retry khi conflict |
| **Use case** | Banking, inventory flash sale | Blog edits, profile updates |

---

## Distributed Transactions

Khi operation span nhiều services/databases — **không dùng 2PC (Two-Phase Commit)** trong microservices hiện đại.

### Saga Pattern

```
Order Service          Payment Service       Inventory Service
     │                       │                       │
     ├── Create Order ──────►│                       │
     │                       ├── Charge Payment ────►│
     │                       │                       ├── Reserve Stock
     │                       │                       │
     │  (nếu fail)           │                       │
     │◄── Compensate ────────┤◄── Refund ────────────┤◄── Release Stock
```

```typescript
// Choreography-based saga — mỗi service publish events
// Orchestration-based saga — central coordinator

// Trong Node.js: dùng BullMQ, Kafka, hoặc Temporal cho saga orchestration
```

### Outbox Pattern

Đảm bảo DB write và event publish atomic:

```
1. Write to business table + outbox table trong cùng transaction
2. Background worker đọc outbox → publish events
3. Mark outbox entries as processed
```

---

## Common Pitfalls

### 1. Long-Running Transactions

```typescript
// ❌ BAD — giữ transaction mở trong khi gọi external API
await prisma.$transaction(async (tx) => {
  const order = await tx.order.create({ data });
  await sendEmailNotification(order); // External call trong transaction!
  await chargePaymentGateway(order);  // Có thể timeout
});

// ✅ GOOD — transaction chỉ cho DB operations
const order = await prisma.$transaction(async (tx) => {
  return tx.order.create({ data });
});
await sendEmailNotification(order);
await chargePaymentGateway(order);
```

### 2. Nested Transactions

PostgreSQL không có true nested transactions — `SAVEPOINT` simulate:

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  SAVEPOINT sp1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 999; -- Fail FK
  ROLLBACK TO sp1;  -- Chỉ rollback phần sau savepoint
COMMIT;
```

### 3. Connection Leak trong Transaction

```typescript
// Luôn release client trong finally
const client = await pool.connect();
try {
  await client.query('BEGIN');
  // ...
  await client.query('COMMIT');
} catch (err) {
  await client.query('ROLLBACK');
  throw err;
} finally {
  client.release(); // BẮT BUỘC
}
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Khi nào cần transaction?

**Trả lời:** Khi nhiều DB operations phải atomic — financial transfers, order + inventory update, user + default settings creation. Single CRUD operation không cần transaction.

### Câu 2: Optimistic vs pessimistic locking — chọn khi nào?

**Trả lời:** Pessimistic khi high contention, data critical (banking, limited inventory). Optimistic khi low contention, read-heavy, conflict rare (user profile edit). Optimistic scale tốt hơn.

### Câu 3: Transaction isolation level default của PostgreSQL?

**Trả lời:** READ COMMITTED — mỗi statement thấy snapshot mới nhất đã commit. REPEATABLE READ cho consistent reads trong transaction. SERIALIZABLE khi cần strictest isolation.

### Câu 4: Microservices có dùng distributed transaction không?

**Trả lời:** Tránh 2PC — latency cao, tight coupling. Dùng Saga pattern (choreography hoặc orchestration) với compensating transactions. Outbox pattern cho reliable event publishing.

---

**Xem tiếp:** [8-n-plus-one.md](./8-n-plus-one.md) — anti-pattern phổ biến nhất gây API chậm.
