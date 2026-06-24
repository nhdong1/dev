# Caching Strategies — Cache-Aside, Write-Through và TTL Patterns

> Caching (bộ nhớ đệm) là một trong những optimization hiệu quả nhất cho read-heavy applications — giảm database load và latency. Chọn đúng **cache pattern (mẫu cache)** và **invalidation strategy (chiến lược vô hiệu hóa)** quan trọng hơn chọn Redis vs in-memory.

## Mục Lục

1. [Tại Sao Cache](#tại-sao-cache)
2. [Cache-Aside Pattern](#cache-aside-pattern)
3. [Write-Through & Write-Behind](#write-through--write-behind)
4. [Read-Through Pattern](#read-through-pattern)
5. [TTL và Eviction](#ttl-và-eviction)
6. [Cache Stampede](#cache-stampede)
7. [Multi-Layer Caching](#multi-layer-caching)
8. [Implementation với Redis](#implementation-với-redis)
9. [Best Practices](#best-practices)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cache

```
Without Cache:                    With Cache:
Client → API → DB (50ms)          Client → API → Redis (1ms) ✅
                                              ↘ DB (50ms) chỉ khi cache miss
```

| Metric | Không Cache | Có Cache (hit rate 90%) |
| ------ | ----------- | ----------------------- |
| Avg latency | 50ms | ~6ms (0.9×1 + 0.1×50) |
| DB queries/s | 10,000 | 1,000 |
| DB CPU | 80% | 15% |

**Cache khi:** Data đọc nhiều, ít thay đổi, chấp nhận eventual consistency (nhất quán cuối cùng).

**Không cache khi:** Financial transactions, real-time inventory chính xác tuyệt đối.

---

## Cache-Aside Pattern

Ứng dụng tự quản lý cache — pattern phổ biến nhất.

```
READ:
  1. App đọc cache
  2. Cache HIT → return data
  3. Cache MISS → đọc DB → ghi cache → return data

WRITE:
  1. App ghi DB
  2. Invalidate (xóa) hoặc update cache
```

```typescript
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);
const CACHE_TTL = 300; // 5 phút

async function getUser(userId: string) {
  const cacheKey = `user:${userId}`;

  // 1. Thử cache
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  // 2. Cache miss — đọc DB
  const user = await db.user.findUnique({ where: { id: userId } });
  if (!user) return null;

  // 3. Populate cache
  await redis.set(cacheKey, JSON.stringify(user), 'EX', CACHE_TTL);

  return user;
}

async function updateUser(userId: string, data: UpdateUserDto) {
  // 1. Ghi DB trước
  const user = await db.user.update({ where: { id: userId }, data });

  // 2. Invalidate cache
  await redis.del(`user:${userId}`);

  return user;
}
```

| Ưu Điểm | Nhược Điểm |
| ------- | ---------- |
| App kiểm soát hoàn toàn | Code lặp ở nhiều nơi |
| Chỉ cache data cần thiết | Cache miss penalty |
| DB là source of truth | Race condition khi concurrent writes |

---

## Write-Through & Write-Behind

### Write-Through — Ghi Xuyên Suốt

Ghi đồng thời vào cache và DB — cache luôn fresh.

```typescript
async function updateUserWriteThrough(userId: string, data: UpdateUserDto) {
  const user = await db.user.update({ where: { id: userId }, data });

  // Ghi cache ngay sau DB
  await redis.set(`user:${userId}`, JSON.stringify(user), 'EX', CACHE_TTL);

  return user;
}
```

```
Write-Through:
  App → Cache (write) → DB (write)
  Read luôn từ cache — fresh data
```

### Write-Behind (Write-Back) — Ghi Trễ

Ghi cache trước, batch ghi DB sau — throughput cao nhưng risk mất data.

```typescript
// Simplified write-behind với queue
async function updateUserWriteBehind(userId: string, data: UpdateUserDto) {
  const user = { id: userId, ...data, updatedAt: new Date() };

  // Ghi cache ngay
  await redis.set(`user:${userId}`, JSON.stringify(user), 'EX', CACHE_TTL);

  // Queue DB write — async, batched
  await writeQueue.add('user:update', { userId, data });

  return user;
}
```

| Pattern | Consistency | Latency Write | Use Case |
| ------- | ----------- | ------------- | -------- |
| Cache-Aside | Eventual | Thấp | General purpose |
| Write-Through | Strong | Trung bình | Read-heavy, cần fresh cache |
| Write-Behind | Eventual | Rất thấp | Analytics, counters, logs |

---

## Read-Through Pattern

Cache layer tự load data khi miss — app chỉ gọi cache API.

```typescript
class ReadThroughCache<T> {
  constructor(
    private redis: Redis,
    private loader: (key: string) => Promise<T | null>,
    private keyPrefix: string,
    private ttl: number,
  ) {}

  async get(id: string): Promise<T | null> {
    const key = `${this.keyPrefix}:${id}`;
    const cached = await this.redis.get(key);

    if (cached) return JSON.parse(cached);

    const data = await this.loader(id);
    if (data) {
      await this.redis.set(key, JSON.stringify(data), 'EX', this.ttl);
    }
    return data;
  }
}

const userCache = new ReadThroughCache(
  redis,
  (id) => db.user.findUnique({ where: { id } }),
  'user',
  300,
);
```

---

## TTL và Eviction

### TTL (Time To Live — Thời Gian Sống)

```typescript
// Fixed TTL
await redis.set('key', value, 'EX', 300); // 5 phút

// TTL khác nhau theo data type
const TTL = {
  userProfile: 300,      // 5 phút — thay đổi thường xuyên
  productCatalog: 3600,  // 1 giờ — ít thay đổi
  staticConfig: 86400,   // 24 giờ — gần như static
};
```

### TTL + Jitter — Tránh Thundering Herd

```typescript
function ttlWithJitter(baseTtl: number): number {
  const jitter = Math.floor(Math.random() * baseTtl * 0.1); // ±10%
  return baseTtl + jitter;
}

await redis.set(key, value, 'EX', ttlWithJitter(300));
```

### Cache Invalidation Strategies

| Strategy | Mô Tả | Khi Dùng |
| -------- | ----- | -------- |
| **TTL-based** | Tự expire sau thời gian | Data chấp nhận stale |
| **Event-based** | Invalidate khi data thay đổi | Cần consistency cao hơn |
| **Version-based** | Cache key có version | Schema migration |
| **Tag-based** | Nhóm keys theo tag, invalidate cả nhóm | Related entities |

```typescript
// Tag-based invalidation
async function invalidateProductCache(productId: string) {
  const tag = `tag:product:${productId}`;
  const keys = await redis.smembers(tag);
  if (keys.length > 0) {
    await redis.del(...keys, tag);
  }
}

async function cacheWithTag(key: string, tag: string, data: unknown, ttl: number) {
  await redis.set(key, JSON.stringify(data), 'EX', ttl);
  await redis.sadd(tag, key);
}
```

---

## Cache Stampede

**Cache stampede (bão cache)** — nhiều requests cùng cache miss, cùng query DB.

```
Cache expires at T=0
  Request 1 ──► MISS ──► DB query
  Request 2 ──► MISS ──► DB query  } 100 concurrent
  Request 3 ──► MISS ──► DB query  } requests
  ...
  Request 100 ──► MISS ──► DB query → DB overload!
```

### Giải Pháp: Lock/Mutex

```typescript
async function getWithLock(key: string, loader: () => Promise<unknown>, ttl: number) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const lockKey = `lock:${key}`;
  const acquired = await redis.set(lockKey, '1', 'EX', 10, 'NX');

  if (!acquired) {
    // Đợi và retry — request khác đang load
    await sleep(50);
    return getWithLock(key, loader, ttl);
  }

  try {
    const data = await loader();
    await redis.set(key, JSON.stringify(data), 'EX', ttl);
    return data;
  } finally {
    await redis.del(lockKey);
  }
}
```

### Giải Pháp: Probabilistic Early Expiration

Refresh cache trước khi expire nếu gần hết TTL và có request.

---

## Multi-Layer Caching

```
Request
   │
   ▼
┌─────────────┐  L1: In-Memory (node-cache, LRU)  ~0.01ms
│  L1 Cache   │
└──────┬──────┘
       │ miss
       ▼
┌─────────────┐  L2: Redis (shared)               ~1ms
│  L2 Cache   │
└──────┬──────┘
       │ miss
       ▼
┌─────────────┐  L3: Database                     ~50ms
│  Database   │
└─────────────┘
```

```typescript
import LRU from 'lru-cache';

const l1Cache = new LRU<string, object>({ max: 1000, ttl: 60_000 });

async function getProduct(id: string) {
  // L1
  const l1 = l1Cache.get(id);
  if (l1) return l1;

  // L2
  const l2 = await redis.get(`product:${id}`);
  if (l2) {
    const product = JSON.parse(l2);
    l1Cache.set(id, product);
    return product;
  }

  // L3
  const product = await db.product.findUnique({ where: { id } });
  if (product) {
    await redis.set(`product:${id}`, JSON.stringify(product), 'EX', 3600);
    l1Cache.set(id, product);
  }
  return product;
}
```

**Lưu ý cluster:** L1 cache per worker — inconsistency giữa workers. Chỉ cache truly static data ở L1.

---

## Implementation với Redis

Xem chi tiết tại [04-data-access/6-redis-caching.md](../04-data-access/6-redis-caching.md).

### Cache Key Design

```typescript
// ✅ Namespaced, versioned keys
const key = `v1:user:${userId}`;
const key = `v1:product:${productId}:details`;
const key = `v1:search:${hashQuery(params)}`;

// ❌ Tránh
const key = userId; // Collision risk
const key = `user_${userId}`; // Không có namespace
```

### Serialization

```typescript
// JSON — đơn giản, human-readable
await redis.set(key, JSON.stringify(data));

// MessagePack — nhỏ hơn, nhanh hơn (high throughput)
import { encode, decode } from '@msgpack/msgpack';
await redis.setBuffer(key, Buffer.from(encode(data)));
```

---

## Best Practices

1. **Cache what hurts** — Profile trước, cache hot paths
2. **Always set TTL** — Tránh stale data vĩnh viễn
3. **Monitor hit rate** — `cache_hits / (cache_hits + cache_misses)`
4. **Graceful degradation** — App vẫn chạy khi Redis down
5. **Không cache errors** — Tránh cache 404/500 responses (trừ intentional)
6. **Size limits** — Không cache objects quá lớn (> 1MB)
7. **Security** — Không cache sensitive data không encrypted

```typescript
// Graceful degradation
async function getCached(key: string, loader: () => Promise<unknown>) {
  try {
    const cached = await redis.get(key);
    if (cached) return JSON.parse(cached);
  } catch (err) {
    console.warn('Redis unavailable, falling back to DB', err);
  }
  return loader();
}
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Gợi Ý Trả Lời |
| ------- | ------------- |
| Cache-aside vs write-through? | Cache-aside: app manages. Write-through: write cả cache lẫn DB đồng thời |
| Cache invalidation strategy? | TTL cho eventual consistency; event-based khi cần fresher data |
| Cache stampede là gì? | Nhiều requests miss cùng lúc → DB overload. Fix: lock, early expiration |
| Redis vs in-memory cache? | In-memory: nhanh nhất, per-process. Redis: shared, persistent, scale horizontal |
| Cache hit rate bao nhiêu là tốt? | > 80% cho hot paths; monitor và tune TTL |
| Khi nào không nên cache? | Data thay đổi liên tục, personalized per-user sensitive data |

---

**Tiếp theo:** [5-connection-pooling.md](./5-connection-pooling.md) — Database pool sizing và pool exhaustion
