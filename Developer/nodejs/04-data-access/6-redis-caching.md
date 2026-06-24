# Redis Caching — ioredis, Cache Patterns và Pub/Sub

> Redis là in-memory data store (kho lưu trữ dữ liệu trong bộ nhớ) phổ biến nhất cho caching, session storage, rate limiting, và real-time features trong Node.js.

## Mục Lục

1. [Redis Là Gì](#redis-là-gì)
2. [ioredis Setup](#ioredis-setup)
3. [Data Structures](#data-structures)
4. [Cache Patterns](#cache-patterns)
5. [TTL và Cache Invalidation](#ttl-và-cache-invalidation)
6. [Session Store](#session-store)
7. [Pub/Sub (Publish/Subscribe)](#pubsub-publishsubscribe)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Redis Là Gì

| Use Case | Mô Tả |
| -------- | ----- |
| **Caching** | Cache DB query results — giảm latency và DB load |
| **Session Store** | Lưu user sessions thay vì in-memory (scale horizontal) |
| **Rate Limiting** | Counter với TTL — giới hạn requests per IP/user |
| **Pub/Sub** | Real-time messaging giữa services |
| **Job Queues** | BullMQ dùng Redis làm backend |
| **Leaderboards** | Sorted Sets cho ranking |

```bash
npm install ioredis
# Session với Express
npm install express-session connect-redis
```

---

## ioredis Setup

```typescript
import Redis from 'ioredis';

const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD,
  db: 0,
  maxRetriesPerRequest: 3,
  retryStrategy(times) {
    const delay = Math.min(times * 50, 2000);
    return delay;
  },
  lazyConnect: true,
});

redis.on('connect', () => console.log('Redis connected'));
redis.on('error', (err) => console.error('Redis error:', err));

// Graceful shutdown
process.on('SIGTERM', async () => {
  await redis.quit();
});
```

### Connection cho Production

```typescript
// Redis Cluster
const cluster = new Redis.Cluster([
  { host: 'redis-node-1', port: 6379 },
  { host: 'redis-node-2', port: 6379 },
]);

// TLS + URL
const redis = new Redis(process.env.REDIS_URL, {
  tls: process.env.NODE_ENV === 'production' ? {} : undefined,
});
```

---

## Data Structures

```typescript
// String — cache JSON, counters
await redis.set('user:123', JSON.stringify(userData), 'EX', 3600);
const cached = await redis.get('user:123');
const user = cached ? JSON.parse(cached) : null;

// Hash — object fields
await redis.hset('user:123:profile', 'name', 'Alice', 'email', 'alice@example.com');
await redis.hgetall('user:123:profile');

// List — queues, recent items
await redis.lpush('recent:views', 'product:456');
await redis.ltrim('recent:views', 0, 99); // Giữ 100 items

// Set — unique members, tags
await redis.sadd('user:123:tags', 'nodejs', 'typescript');
await redis.smembers('user:123:tags');

// Sorted Set — leaderboards, rankings
await redis.zadd('leaderboard', 1500, 'player:alice', 1200, 'player:bob');
const top10 = await redis.zrevrange('leaderboard', 0, 9, 'WITHSCORES');

// Counter — rate limiting
const count = await redis.incr('rate:ip:192.168.1.1');
if (count === 1) await redis.expire('rate:ip:192.168.1.1', 60);
```

---

## Cache Patterns

### Cache-Aside (Lazy Loading)

Pattern phổ biến nhất — app quản lý cache explicitly:

```typescript
async function getUserById(userId: string) {
  const cacheKey = `user:${userId}`;

  // 1. Check cache
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  // 2. Cache miss — query DB
  const user = await prisma.user.findUnique({ where: { id: userId } });
  if (!user) return null;

  // 3. Populate cache
  await redis.set(cacheKey, JSON.stringify(user), 'EX', 3600);

  return user;
}
```

```
Request ──► Check Redis ──► Hit? ──► Return cached
                │
                ▼ Miss
           Query Database ──► Set Redis ──► Return data
```

### Write-Through

Ghi DB và cache đồng thời:

```typescript
async function updateUser(userId: string, data: UpdateUserDto) {
  const user = await prisma.user.update({ where: { id: userId }, data });

  // Update cache immediately
  await redis.set(`user:${userId}`, JSON.stringify(user), 'EX', 3600);

  return user;
}
```

### Write-Behind (Write-Back)

Ghi cache trước, DB async sau — high throughput nhưng risk data loss.

### Read-Through

Cache layer tự fetch từ DB khi miss — thường implement bằng library (Redis không native).

### So Sánh Patterns

| Pattern | Read | Write | Consistency | Phù Hợp |
| ------- | ---- | ----- | ----------- | ------- |
| **Cache-Aside** | App check cache | App update cache + DB | Eventual | General purpose |
| **Write-Through** | App check cache | Sync cache + DB | Strong hơn | Read-heavy, consistency |
| **Write-Behind** | App check cache | Cache first, DB async | Weak | Write-heavy, analytics |

---

## TTL và Cache Invalidation

```typescript
// TTL (Time To Live — Thời Gian Sống)
await redis.set('key', 'value', 'EX', 300);     // Expire sau 300 giây
await redis.setex('key', 300, 'value');          // Tương đương
await redis.expire('key', 300);                  // Set TTL cho key existing

// Invalidation strategies
async function invalidateUserCache(userId: string) {
  await redis.del(`user:${userId}`);
  await redis.del(`user:${userId}:posts`);
}

// Pattern-based delete (Redis 6.2+)
await redis.del(await redis.keys('user:*')); // ⚠️ Cẩn thận trên production lớn

// SCAN thay KEYS cho production
async function deleteByPattern(pattern: string) {
  let cursor = '0';
  do {
    const [nextCursor, keys] = await redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
    cursor = nextCursor;
    if (keys.length) await redis.del(...keys);
  } while (cursor !== '0');
}
```

### Cache Stampede Prevention

```typescript
async function getUserWithLock(userId: string) {
  const cacheKey = `user:${userId}`;
  const lockKey = `lock:${cacheKey}`;

  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  // Acquire lock — chỉ một process rebuild cache
  const acquired = await redis.set(lockKey, '1', 'EX', 10, 'NX');
  if (!acquired) {
    await new Promise((r) => setTimeout(r, 100));
    return getUserWithLock(userId); // Retry
  }

  try {
    const user = await prisma.user.findUnique({ where: { id: userId } });
    if (user) await redis.set(cacheKey, JSON.stringify(user), 'EX', 3600);
    return user;
  } finally {
    await redis.del(lockKey);
  }
}
```

---

## Session Store

```typescript
import session from 'express-session';
import RedisStore from 'connect-redis';

const redisStore = new RedisStore({
  client: redis,
  prefix: 'sess:',
  ttl: 86400, // 24 giờ
});

app.use(
  session({
    store: redisStore,
    secret: process.env.SESSION_SECRET!,
    resave: false,
    saveUninitialized: false,
    cookie: {
      secure: process.env.NODE_ENV === 'production',
      httpOnly: true,
      maxAge: 86400000,
      sameSite: 'lax',
    },
  })
);
```

> Redis session store cho phép **horizontal scaling** — nhiều Node.js instances share sessions.

---

## Pub/Sub (Publish/Subscribe)

```typescript
// Publisher
const publisher = new Redis();

async function notifyOrderCreated(orderId: string) {
  await publisher.publish('orders:created', JSON.stringify({ orderId, timestamp: Date.now() }));
}

// Subscriber — dedicated connection (blocking)
const subscriber = new Redis();

subscriber.subscribe('orders:created', 'orders:updated', (err, count) => {
  console.log(`Subscribed to ${count} channels`);
});

subscriber.on('message', (channel, message) => {
  const data = JSON.parse(message);
  console.log(`[${channel}]`, data);

  if (channel === 'orders:created') {
    // Send notification, update cache, etc.
  }
});
```

### Pub/Sub vs Message Queue

| | Redis Pub/Sub | BullMQ / RabbitMQ |
| - | ------------- | ----------------- |
| **Persistence** | Không — fire and forget | Có — messages survive restart |
| **Delivery** | At-most-once | At-least-once / exactly-once |
| **Use Case** | Real-time notifications | Background jobs, reliable processing |

---

## Best Practices

1. **Luôn set TTL** — tránh memory leak từ stale cache
2. **Serialize JSON** — Redis chỉ lưu strings
3. **Key naming convention** — `entity:id:field` (ví dụ: `user:123:profile`)
4. **Connection pooling** — một Redis instance per process
5. **Không cache sensitive data** không encrypted (passwords, tokens)
6. **Monitor memory** — `INFO memory`, set `maxmemory-policy`
7. **Pipeline** cho batch operations — giảm round-trips

```typescript
const pipeline = redis.pipeline();
pipeline.set('key1', 'val1');
pipeline.set('key2', 'val2');
pipeline.incr('counter');
await pipeline.exec();
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Cache-aside vs write-through?

**Trả lời:** Cache-aside: app đọc cache, miss thì query DB và populate — đơn giản, phổ biến. Write-through: mỗi write update cả DB và cache — consistency tốt hơn nhưng write latency cao hơn.

### Câu 2: Redis single-threaded — tại sao vẫn nhanh?

**Trả lời:** In-memory operations (~100k ops/s), không disk I/O cho reads, simple data structures. Bottleneck thường là network latency, không phải CPU. Scale bằng Redis Cluster sharding.

### Câu 3: Khi nào KHÔNG nên dùng Redis cache?

**Trả lời:** Data thay đổi liên tục (real-time stock prices), strong consistency required, dataset lớn hơn RAM, hoặc khi invalidation logic quá phức tạp gây stale data bugs.

### Câu 4: `KEYS *` vs `SCAN`?

**Trả lời:** `KEYS` block Redis server — O(N) trên toàn bộ keyspace. `SCAN` iterate incrementally, không block — dùng trong production cho pattern delete.

---

**Xem tiếp:** [7-transactions.md](./7-transactions.md) — ACID transactions trong Node.js.
