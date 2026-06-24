# Connection Pooling — DB Pool Sizing và Pool Exhaustion

> Connection pool (bể kết nối) tái sử dụng database connections — tránh overhead tạo connection mới mỗi query. **Pool exhaustion (cạn kiệt pool)** là nguyên nhân phổ biến gây timeout và cascading failures trong production Node.js APIs.

## Mục Lục

1. [Tại Sao Cần Connection Pool](#tại-sao-cần-connection-pool)
2. [pg Pool Configuration](#pg-pool-configuration)
3. [Prisma Connection Management](#prisma-connection-management)
4. [Pool Sizing Formula](#pool-sizing-formula)
5. [Pool Exhaustion — Triệu Chứng và Fix](#pool-exhaustion--triệu-chứng-và-fix)
6. [Multi-Instance Pool Math](#multi-instance-pool-math)
7. [Monitoring Pool Health](#monitoring-pool-health)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Connection Pool

```
Without Pool:                      With Pool:
Request → New Connection (50ms)   Request → Borrow from pool (0.1ms)
       → Query (5ms)                      → Query (5ms)
       → Close Connection                  → Return to pool
       
1000 RPS = 1000 connections/s     1000 RPS = 10-20 connections reused
→ DB overwhelmed                  → DB stable
```

| Không Pool | Có Pool |
| ---------- | ------- |
| Connection overhead mỗi query | Connections reused |
| Risk vượt `max_connections` | Bounded connection count |
| Slow under load | Predictable performance |
| Connection leak dễ xảy ra | Pool quản lý lifecycle |

---

## pg Pool Configuration

```typescript
import { Pool } from 'pg';

const pool = new Pool({
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT || '5432'),
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,

  // Pool sizing
  max: 20,                    // Maximum connections in pool
  min: 2,                     // Minimum idle connections
  idleTimeoutMillis: 30_000,  // Close idle connections after 30s
  connectionTimeoutMillis: 5_000, // Fail if no connection available in 5s

  // Statement timeout — prevent hung queries
  statement_timeout: 30_000,  // 30 seconds
});

// Pool events
pool.on('connect', () => console.log('New client connected to pool'));
pool.on('error', (err) => console.error('Pool error:', err));
pool.on('remove', () => console.log('Client removed from pool'));

// Graceful shutdown
process.on('SIGTERM', async () => {
  await pool.end();
});
```

### Query Với Pool

```typescript
// ✅ Borrow và return tự động
const result = await pool.query(
  'SELECT * FROM users WHERE id = $1',
  [userId],
);

// ✅ Transaction
const client = await pool.connect();
try {
  await client.query('BEGIN');
  await client.query('INSERT INTO orders ...');
  await client.query('UPDATE inventory ...');
  await client.query('COMMIT');
} catch (err) {
  await client.query('ROLLBACK');
  throw err;
} finally {
  client.release(); // ⚠️ QUAN TRỌNG — luôn release!
}
```

### Connection Leak — Rò Rỉ Connection

```typescript
// ❌ LEAK — không release client
async function leakyQuery() {
  const client = await pool.connect();
  const result = await client.query('SELECT 1');
  return result.rows; // client never released!
}

// ✅ Always release in finally
async function safeQuery() {
  const client = await pool.connect();
  try {
    const result = await client.query('SELECT 1');
    return result.rows;
  } finally {
    client.release();
  }
}
```

---

## Prisma Connection Management

```typescript
// prisma/schema.prisma — connection limit qua DATABASE_URL
// postgresql://user:pass@host:5432/db?connection_limit=10&pool_timeout=10

import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient({
  log: ['query', 'warn', 'error'],
});

// Singleton pattern — tránh multiple PrismaClient instances
declare global {
  var prisma: PrismaClient | undefined;
}

export const db = global.prisma ?? new PrismaClient();
if (process.env.NODE_ENV !== 'production') global.prisma = db;

// Graceful disconnect
process.on('SIGTERM', async () => {
  await db.$disconnect();
});
```

### Prisma Pool Parameters

| URL Parameter | Mô Tả | Default |
| ------------- | ----- | ------- |
| `connection_limit` | Max connections per Prisma Client | `num_cpus * 2 + 1` |
| `pool_timeout` | Seconds wait for connection | 10 |
| `connect_timeout` | Seconds to establish connection | 5 |

```bash
# Production DATABASE_URL
DATABASE_URL="postgresql://user:pass@host:5432/mydb?connection_limit=10&pool_timeout=10&connect_timeout=5"
```

---

## Pool Sizing Formula

### Công Thức Cổ Điển (PostgreSQL Wiki)

```
pool_size = (num_cores * 2) + effective_spindle_count
```

Với SSD: `effective_spindle_count ≈ 1`

```
4 cores → (4 * 2) + 1 = 9 connections per instance
```

### Thực Tế Node.js

| Factor | Recommendation |
| ------ | -------------- |
| **Single instance** | 10–20 connections |
| **I/O-bound API** | Có thể cao hơn (connections idle chờ I/O) |
| **CPU-bound queries** | Thấp hơn (queries chạy lâu, giữ connection) |
| **Serverless** | 1 connection per function instance |

### Tính Toán Tổng Connections

```
Total DB connections =
  num_app_instances × pool_max_per_instance

Ví dụ:
  4 PM2 workers × 10 pool max = 40 connections
  + 2 migration jobs × 5 = 10
  + monitoring tools = 5
  Total = 55 → OK nếu PostgreSQL max_connections = 100
```

---

## Pool Exhaustion — Triệu Chứng và Fix

### Triệu Chứng

```
Error: timeout exceeded when trying to connect
Error: sorry, too many clients already
API latency spike → 503 errors
pool.totalCount === pool.max && pool.waitingCount > 0
```

### Nguyên Nhân

| Nguyên Nhân | Fix |
| ----------- | --- |
| Pool quá nhỏ cho traffic | Tăng `max` (trong giới hạn DB) |
| Connection leak | Audit `client.release()`, dùng try/finally |
| Slow queries giữ connections | Optimize queries, add `statement_timeout` |
| Quá nhiều app instances | Giảm instances hoặc pool size per instance |
| Long-running transactions | Break up transactions, avoid trong request path |
| N+1 queries | Batch queries, eager loading |

### Debug Pool State

```typescript
function logPoolStats() {
  console.log({
    total: pool.totalCount,    // Connections hiện tại (active + idle)
    idle: pool.idleCount,      // Connections available
    waiting: pool.waitingCount, // Requests đang chờ connection
  });
}

setInterval(logPoolStats, 10_000);

// Alert khi waiting > 0
if (pool.waitingCount > 0) {
  console.warn('Pool exhaustion warning!', logPoolStats());
}
```

### Queue Pattern — Giảm Pressure

```typescript
import PQueue from 'p-queue';

// Giới hạn concurrent DB operations
const dbQueue = new PQueue({ concurrency: 15 }); // < pool.max

async function safeQuery(sql: string, params: unknown[]) {
  return dbQueue.add(() => pool.query(sql, params));
}
```

---

## Multi-Instance Pool Math

```
┌─────────────────────────────────────────────────────┐
│              Load Balancer                           │
└────────┬──────────┬──────────┬──────────┬───────────┘
         │          │          │          │
    ┌────▼───┐ ┌────▼───┐ ┌────▼───┐ ┌────▼───┐
    │ Pod 1  │ │ Pod 2  │ │ Pod 3  │ │ Pod 4  │
    │ pool:10│ │ pool:10│ │ pool:10│ │ pool:10│
    └────┬───┘ └────┬───┘ └────┬───┘ └────┬───┘
         └──────────┴──────────┴──────────┘
                         │
              ┌──────────▼──────────┐
              │    PostgreSQL       │
              │  max_connections:100│
              │  used: 40 + overhead│
              └─────────────────────┘
```

### PgBouncer — Connection Pooler Trung Gian

Khi có nhiều instances, dùng **PgBouncer** giữa app và PostgreSQL:

```
100 app instances × 10 pool = 1000 app connections
         ↓ PgBouncer (transaction pooling)
         ↓ 50 actual PostgreSQL connections
```

```ini
# pgbouncer.ini
[databases]
mydb = host=postgres port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
```

---

## Monitoring Pool Health

### Prometheus Metrics

```typescript
import { Gauge } from 'prom-client';

const poolTotal = new Gauge({ name: 'db_pool_total', help: 'Total pool connections' });
const poolIdle = new Gauge({ name: 'db_pool_idle', help: 'Idle pool connections' });
const poolWaiting = new Gauge({ name: 'db_pool_waiting', help: 'Waiting for connection' });

setInterval(() => {
  poolTotal.set(pool.totalCount);
  poolIdle.set(pool.idleCount);
  poolWaiting.set(pool.waitingCount);
}, 10_000);
```

### Alert Rules

```yaml
- alert: DBPoolExhaustion
  expr: db_pool_waiting > 0
  for: 2m
  labels:
    severity: critical
```

### PostgreSQL Monitoring

```sql
-- Active connections
SELECT count(*) FROM pg_stat_activity WHERE state = 'active';

-- Connections by application
SELECT application_name, count(*)
FROM pg_stat_activity
GROUP BY application_name;

-- Long running queries
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active' AND now() - query_start > interval '30 seconds';
```

---

## Best Practices

1. **Một pool per process** — Singleton, không tạo pool mới per request
2. **Always release** — `finally { client.release() }` cho mọi `pool.connect()`
3. **Set timeouts** — `connectionTimeoutMillis`, `statement_timeout`
4. **Size conservatively** — Tổng connections < 80% `max_connections`
5. **PgBouncer** — Khi > 10 app instances hoặc serverless
6. **Monitor waiting count** — Early warning trước exhaustion
7. **Graceful shutdown** — `pool.end()` / `prisma.$disconnect()` on SIGTERM

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Gợi Ý Trả Lời |
| ------- | ------------- |
| Connection pool là gì? | Tập connections tái sử dụng — tránh overhead tạo/đóng mỗi query |
| Pool size bao nhiêu? | `(cores * 2) + 1` per instance; tổng < DB max_connections |
| Pool exhaustion xử lý thế nào? | Tăng pool (cẩn thận), fix leaks, optimize slow queries, PgBouncer |
| Tại sao connection leak nguy hiểm? | Pool cạn → requests queue → timeout → cascading failure |
| PgBouncer là gì? | External pooler — multiplex nhiều app connections vào ít DB connections |
| Prisma vs raw pg pool? | Prisma quản lý pool internally qua connection_limit URL param |

---

**Tiếp theo:** [6-load-testing.md](./6-load-testing.md) — k6, Artillery và throughput benchmarks
