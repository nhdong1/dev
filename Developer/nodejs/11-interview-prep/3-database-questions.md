# Câu Hỏi Database & Data Access

> Bộ câu hỏi phỏng vấn về PostgreSQL, ORM (Object-Relational Mapping — Ánh Xạ Đối Tượng Quan Hệ), N+1 problem, transactions, Redis caching, và query optimization trong Node.js.

## Mục Lục

1. [SQL & ORM Fundamentals](#sql--orm-fundamentals)
2. [N+1 Problem & Query Optimization](#n1-problem--query-optimization)
3. [Transactions & Data Integrity](#transactions--data-integrity)
4. [Caching với Redis](#caching-với-redis)
5. [SQL vs NoSQL](#sql-vs-nosql)
6. [Live Coding Challenges](#live-coding-challenges)

---

## SQL & ORM Fundamentals

### Q1: ORM là gì? Ưu và nhược điểm?

**ORM (Object-Relational Mapping)** — ánh xạ bảng database thành objects trong code.

| Ưu Điểm | Nhược Điểm |
| ------- | ----------- |
| Type-safe queries (Prisma, TypeORM) | Abstraction leak — khó debug query phức tạp |
| Migrations tự động | Performance overhead so với raw SQL |
| Relations dễ quản lý | N+1 problem nếu không cẩn thận |
| Database-agnostic (một phần) | Learning curve, magic behavior |

**Khi dùng raw SQL:** Complex reports, bulk operations, performance-critical queries.

---

### Q2: Connection Pool (Bể Kết Nối) — tại sao cần và sizing?

**Vấn đề:** Mỗi DB connection tốn ~1–5MB RAM trên DB server. Tạo connection mới tốn ~20–50ms.

**Connection pool** tái sử dụng connections:

```javascript
const { Pool } = require('pg');
const pool = new Pool({
  host: 'localhost',
  database: 'myapp',
  max: 20,              // Max connections trong pool
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
});

const result = await pool.query('SELECT * FROM users WHERE id = $1', [userId]);
```

**Sizing formula (rule of thumb):**
```
pool_size = (num_cores * 2) + effective_spindle_count
// Node.js: thường 10–20 per instance
// Tổng connections tất cả instances < max_connections của PostgreSQL
```

---

### Q3: Parameterized Queries — tại sao bắt buộc?

```javascript
// ❌ SQL Injection vulnerability
const query = `SELECT * FROM users WHERE email = '${email}'`;

// ✅ Parameterized — an toàn
const query = 'SELECT * FROM users WHERE email = $1';
await pool.query(query, [email]);
```

ORM như Prisma tự parameterize, nhưng `$queryRaw` cần cẩn thận:

```javascript
// ❌ Nguy hiểm
await prisma.$queryRaw`SELECT * FROM users WHERE id = ${userId}`; // OK — tagged template
await prisma.$queryRawUnsafe(`SELECT * FROM users WHERE id = ${userId}`); // ❌ Injection!
```

---

### Q4: Prisma vs TypeORM vs Sequelize — khi nào dùng gì?

| | Prisma | TypeORM | Sequelize |
| - | ------ | ------- | --------- |
| **Approach** | Schema-first, code generation | Decorator-based, Active Record | Model-based, mature |
| **TypeScript** | Excellent | Good | Fair |
| **Migrations** | `prisma migrate` | Manual/auto | `sequelize-cli` |
| **Query style** | Fluent API | Query Builder + Repository | Sequelize API |
| **Phù hợp** | Greenfield TS projects | NestJS, enterprise | Legacy Node.js |

---

### Q5: Database Indexing — khi nào cần index?

```sql
-- Query chậm — full table scan
SELECT * FROM orders WHERE user_id = 123 AND status = 'pending';

-- Fix: composite index
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

**Khi index:**
- Columns trong WHERE, JOIN, ORDER BY thường xuyên
- Foreign keys
- Columns có cardinality cao (nhiều giá trị unique)

**Khi không index:**
- Bảng nhỏ (<1000 rows)
- Columns ít thay đổi (write-heavy)
- Low cardinality (boolean column)

---

## N+1 Problem & Query Optimization

### Q6: N+1 Problem là gì? Cách phát hiện và fix?

**Vấn đề:** 1 query lấy N records + N queries lấy related data = N+1 queries.

```javascript
// ❌ N+1 — 1 + 100 = 101 queries
const users = await prisma.user.findMany();
for (const user of users) {
  user.posts = await prisma.post.findMany({ where: { userId: user.id } });
}

// ✅ Fix 1: Eager loading (include)
const users = await prisma.user.findMany({
  include: { posts: true },
});

// ✅ Fix 2: DataLoader (batch + cache trong 1 request)
const postLoader = new DataLoader(async (userIds) => {
  const posts = await prisma.post.findMany({
    where: { userId: { in: userIds } },
  });
  return userIds.map(id => posts.filter(p => p.userId === id));
});
```

---

### Q7: DataLoader pattern — giải thích?

**DataLoader** batch multiple requests trong cùng tick của Event Loop:

```
Request 1: loadUser(1)  ─┐
Request 2: loadUser(2)  ─┼─► Batch: SELECT * FROM users WHERE id IN (1,2,3)
Request 3: loadUser(3)  ─┘
```

```javascript
const DataLoader = require('dataloader');

const userLoader = new DataLoader(async (ids) => {
  const users = await db.users.findByIds(ids);
  const map = new Map(users.map(u => [u.id, u]));
  return ids.map(id => map.get(id) || null);
});

// Trong resolver/handler — tự động batch
const user1 = await userLoader.load(1);
const user2 = await userLoader.load(2);
```

**Lưu ý:** Tạo DataLoader mới per request (không share across requests).

---

### Q8: EXPLAIN ANALYZE — đọc query plan?

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE created_at > '2024-01-01';
```

| Term | Ý Nghĩa |
| ---- | ------- |
| **Seq Scan** | Full table scan — chậm với bảng lớn |
| **Index Scan** | Dùng index — nhanh |
| **Nested Loop** | Join nhỏ — OK |
| **Hash Join** | Join lớn — cần memory |
| **cost=0.00..X** | Estimated cost (không phải ms) |
| **actual time** | Thời gian thực tế |

---

### Q9: Pagination với large dataset — best practice?

```javascript
// ❌ Offset pagination — chậm với page lớn
const users = await prisma.user.findMany({
  skip: 100000,
  take: 20,
});

// ✅ Cursor-based pagination
const users = await prisma.user.findMany({
  take: 20,
  cursor: lastId ? { id: lastId } : undefined,
  skip: lastId ? 1 : 0,
  orderBy: { id: 'asc' },
});
```

---

### Q10: Soft Delete vs Hard Delete?

```javascript
// Soft delete — thêm deletedAt column
model User {
  id        Int       @id
  email     String
  deletedAt DateTime?
}

// Query luôn filter
const users = await prisma.user.findMany({
  where: { deletedAt: null },
});

// "Xóa"
await prisma.user.update({
  where: { id },
  data: { deletedAt: new Date() },
});
```

**Trade-off:** Soft delete giữ data cho audit/recovery; hard delete đơn giản hơn, tiết kiệm storage.

---

## Transactions & Data Integrity

### Q11: ACID trong Node.js — ví dụ transaction?

```javascript
// Prisma transaction
await prisma.$transaction(async (tx) => {
  const order = await tx.order.create({ data: orderData });

  for (const item of items) {
    const product = await tx.product.findUnique({ where: { id: item.productId } });
    if (product.stock < item.quantity) {
      throw new Error('Insufficient stock'); // Rollback toàn bộ
    }
    await tx.product.update({
      where: { id: item.productId },
      data: { stock: { decrement: item.quantity } },
    });
    await tx.orderItem.create({ data: { orderId: order.id, ...item } });
  }
});
```

**ACID:**
- **Atomicity** — Tất cả hoặc không
- **Consistency** — Data luôn valid
- **Isolation** — Concurrent transactions không interfere
- **Durability** — Committed data persist

---

### Q12: Optimistic vs Pessimistic Locking?

| | Optimistic | Pessimistic |
| - | ---------- | ----------- |
| **Cơ chế** | Version column, check trước update | `SELECT FOR UPDATE` lock row |
| **Conflict** | Retry khi version mismatch | Block cho đến khi lock release |
| **Phù hợp** | Low contention, read-heavy | High contention, financial |
| **Prisma** | `@version` field | `$executeRaw` FOR UPDATE |

```javascript
// Optimistic locking
const updated = await prisma.product.updateMany({
  where: { id: productId, version: currentVersion },
  data: { stock: newStock, version: { increment: 1 } },
});
if (updated.count === 0) throw new ConflictError('Concurrent modification');
```

---

### Q13: Database Migration — best practices?

1. **Migrations phải reversible** — có `down` migration
2. **Không sửa migration đã deploy** — tạo migration mới
3. **Backward compatible changes** — add column nullable trước, deploy code, rồi backfill
4. **Test migration trên staging** với production-like data volume
5. **Zero-downtime migration** — expand-contract pattern

```
Expand: ADD COLUMN nullable → Deploy code writes to both → Backfill → Deploy code reads new only → Contract: DROP old column
```

---

## Caching với Redis

### Q14: Cache-Aside pattern — implement thế nào?

```javascript
async function getUser(id) {
  const cacheKey = `user:${id}`;

  // 1. Check cache
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  // 2. Cache miss — query DB
  const user = await prisma.user.findUnique({ where: { id } });
  if (!user) return null;

  // 3. Populate cache
  await redis.setex(cacheKey, 3600, JSON.stringify(user)); // TTL 1h
  return user;
}
```

---

### Q15: Cache Invalidation — strategies?

| Strategy | Mô Tả | Use Case |
| -------- | ----- | -------- |
| **TTL** | Expire sau thời gian cố định | Data ít thay đổi |
| **Write-through** | Update cache khi write DB | Consistency cao |
| **Write-behind** | Update cache trước, DB sau | Write-heavy |
| **Event-driven** | Invalidate khi có change event | Distributed systems |

```javascript
// Invalidate on update
async function updateUser(id, data) {
  const user = await prisma.user.update({ where: { id }, data });
  await redis.del(`user:${id}`);
  return user;
}
```

**Nguyên tắc:** "There are only two hard things in Computer Science: cache invalidation and naming things."

---

### Q16: Redis cho Session Store?

```javascript
const session = require('express-session');
const RedisStore = require('connect-redis').default;

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: { secure: true, httpOnly: true, maxAge: 86400000 },
}));
```

**Ưu điểm so với in-memory session:** Shared across multiple Node.js instances, survive restart.

---

## SQL vs NoSQL

### Q17: Khi nào chọn PostgreSQL vs MongoDB?

| Tiêu Chí | PostgreSQL | MongoDB |
| -------- | ---------- | ------- |
| **Data model** | Structured, relations | Flexible schema, documents |
| **Transactions** | Full ACID | Multi-doc ACID (v4+) |
| **Scaling** | Vertical + read replicas | Horizontal sharding native |
| **Queries** | Complex JOINs, analytics | Aggregation pipeline |
| **Use case** | E-commerce, finance, ERP | CMS, IoT, real-time analytics |

**Node.js phổ biến:** PostgreSQL + Prisma cho hầu hết apps; MongoDB khi schema thay đổi liên tục hoặc document-heavy.

---

### Q18: CAP Theorem — giải thích trong context database?

Trong distributed system, chỉ chọn được **2 trong 3**:

- **Consistency** — Mọi node thấy cùng data
- **Availability** — Mọi request nhận response
- **Partition Tolerance** — Hệ thống hoạt động khi network partition

| Database | Trade-off |
| -------- | --------- |
| PostgreSQL (single node) | CA |
| MongoDB (replica set) | CP hoặc AP tùy read preference |
| Redis Cluster | AP |
| CockroachDB | CP (distributed SQL) |

---

## Live Coding Challenges

### Challenge 1: Implement simple in-memory cache với TTL

```javascript
class TTLCache {
  constructor() {
    this.store = new Map();
  }

  set(key, value, ttlMs) {
    this.store.set(key, {
      value,
      expiresAt: Date.now() + ttlMs,
    });
  }

  get(key) {
    const entry = this.store.get(key);
    if (!entry) return undefined;
    if (Date.now() > entry.expiresAt) {
      this.store.delete(key);
      return undefined;
    }
    return entry.value;
  }
}
```

### Challenge 2: Batch insert với transaction

```javascript
async function batchCreateUsers(users) {
  return prisma.$transaction(
    users.map(user =>
      prisma.user.create({ data: user })
    )
  );
}
```

### Challenge 3: Fix N+1 trong code cho sẵn

```javascript
// Input: getOrdersWithItems() trả orders, mỗi order cần items
// Output: 2 queries thay vì N+1

async function getOrdersWithItems() {
  const orders = await prisma.order.findMany({
    include: { items: true },
    orderBy: { createdAt: 'desc' },
    take: 50,
  });
  return orders;
}
```

---

## Tài Liệu Tham Khảo

- [04-data-access/2-prisma-orm.md](../04-data-access/2-prisma-orm.md)
- [04-data-access/7-transactions.md](../04-data-access/7-transactions.md)
- [04-data-access/8-n-plus-one.md](../04-data-access/8-n-plus-one.md)
- [04-data-access/6-redis-caching.md](../04-data-access/6-redis-caching.md)
- [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) — Câu 21–30
