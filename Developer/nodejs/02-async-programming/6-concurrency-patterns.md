# Concurrency Patterns — Throttle, Debounce, Queue và Batch Processing

> Trong production Node.js API, không chỉ cần async/await mà còn cần **kiểm soát concurrency (đồng thời)** — tránh overwhelm database, rate limit external APIs, và xử lý burst traffic hiệu quả.

## Mục Lục

1. [Concurrency vs Parallelism Trong Node.js](#concurrency-vs-parallelism-trong-nodejs)
2. [Debounce](#debounce)
3. [Throttle](#throttle)
4. [Mutex và Lock](#mutex-và-lock)
5. [Job Queue Pattern](#job-queue-pattern)
6. [Concurrency Limit (p-limit)](#concurrency-limit-p-limit)
7. [Batch Processing](#batch-processing)
8. [Retry với Exponential Backoff](#retry-với-exponential-backoff)
9. [Use Cases Thực Tế](#use-cases-thực-tế)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Concurrency vs Parallelism Trong Node.js

```
Concurrency (Single Thread):
  Task A: ████░░░░████░░░░  (interleaved)
  Task B: ░░░░████░░░░████

Parallelism (Multi Thread/Process):
  Task A: ████████████████  (Worker Thread)
  Task B: ████████████████  (Worker Thread)
```

| Pattern | Mô Tả | Node.js Context |
| ------- | ----- | --------------- |
| **Concurrency** | Nhiều task tiến triển xen kẽ trên single thread | async I/O, Event Loop |
| **Parallelism** | Nhiều task chạy thực sự đồng thời | Cluster, Worker Threads |
| **Concurrency limit** | Giới hạn số async ops cùng lúc | p-limit, semaphore |
| **Batching** | Gom nhiều ops thành một | DataLoader, bulk insert |

---

## Debounce

Debounce (Chống Rung) — chỉ thực thi sau khi **ngừng gọi** trong khoảng thời gian `delay`.

```javascript
function debounce(fn, delayMs) {
  let timerId;

  return function debounced(...args) {
    clearTimeout(timerId);
    timerId = setTimeout(() => {
      fn.apply(this, args);
    }, delayMs);
  };
}

// Use case: search autocomplete
const search = debounce(async (query) => {
  const results = await fetch(`/api/search?q=${query}`);
  return results.json();
}, 300);

// User gõ "hello" nhanh → chỉ 1 API call sau 300ms ngừng gõ
```

### Debounce vs Throttle

| | Debounce | Throttle |
| --- | -------- | -------- |
| **Trigger** | Sau khi ngừng events | Đều đặn theo interval |
| **Use case** | Search input, resize handler | Scroll, rate limit API calls |
| **Calls** | 1 call sau burst | Max 1 call per interval |

```
Events:  |--|--|--|--|----|----|
Debounce:                    X (1 call)
Throttle:  X     X     X     X  (max 1 per window)
```

---

## Throttle

Throttle (Giới Hạn Tần Suất) — thực thi tối đa **một lần** trong mỗi time window.

```javascript
function throttle(fn, intervalMs) {
  let lastRun = 0;
  let timerId;

  return function throttled(...args) {
    const now = Date.now();
    const remaining = intervalMs - (now - lastRun);

    if (remaining <= 0) {
      lastRun = now;
      fn.apply(this, args);
    } else if (!timerId) {
      // Trailing call — đảm bảo gọi lần cuối
      timerId = setTimeout(() => {
        lastRun = Date.now();
        timerId = null;
        fn.apply(this, args);
      }, remaining);
    }
  };
}

// Rate limit log writes
const logMetric = throttle((data) => {
  metricsClient.gauge('api.latency', data.ms);
}, 1000);
```

### Token Bucket — Rate Limiting API

```javascript
class TokenBucket {
  constructor({ capacity, refillRate }) {
    this.capacity = capacity;       // max tokens
    this.tokens = capacity;
    this.refillRate = refillRate;   // tokens per second
    this.lastRefill = Date.now();
  }

  refill() {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    this.tokens = Math.min(
      this.capacity,
      this.tokens + elapsed * this.refillRate
    );
    this.lastRefill = now;
  }

  tryConsume(count = 1) {
    this.refill();
    if (this.tokens >= count) {
      this.tokens -= count;
      return true;
    }
    return false;
  }
}

const bucket = new TokenBucket({ capacity: 100, refillRate: 10 });

function rateLimitedHandler(req, res, next) {
  if (!bucket.tryConsume()) {
    return res.status(429).json({ error: 'Too Many Requests' });
  }
  next();
}
```

---

## Mutex và Lock

Mutex (Mutual Exclusion — Loại Trừ Tương Hộ) — đảm bảo chỉ một async operation truy cập critical section tại một thời điểm.

```javascript
class Mutex {
  constructor() {
    this.queue = [];
    this.locked = false;
  }

  async acquire() {
    if (!this.locked) {
      this.locked = true;
      return;
    }
    await new Promise((resolve) => this.queue.push(resolve));
    this.locked = true;
  }

  release() {
    const next = this.queue.shift();
    if (next) {
      next();
    } else {
      this.locked = false;
    }
  }

  async runExclusive(fn) {
    await this.acquire();
    try {
      return await fn();
    } finally {
      this.release();
    }
  }
}

const fileMutex = new Mutex();

async function writeConfig(data) {
  await fileMutex.runExclusive(async () => {
    const current = await fs.readFile('config.json', 'utf8');
    const merged = { ...JSON.parse(current), ...data };
    await fs.writeFile('config.json', JSON.stringify(merged));
  });
}
```

---

## Job Queue Pattern

Queue (Hàng Đợi) — xử lý jobs tuần tự hoặc với concurrency limit.

```javascript
class AsyncQueue {
  constructor({ concurrency = 1 } = {}) {
    this.concurrency = concurrency;
    this.running = 0;
    this.queue = [];
  }

  push(task) {
    return new Promise((resolve, reject) => {
      this.queue.push({ task, resolve, reject });
      this.process();
    });
  }

  async process() {
    if (this.running >= this.concurrency || this.queue.length === 0) {
      return;
    }

    this.running++;
    const { task, resolve, reject } = this.queue.shift();

    try {
      const result = await task();
      resolve(result);
    } catch (err) {
      reject(err);
    } finally {
      this.running--;
      this.process();
    }
  }
}

// Xử lý 100 emails, tối đa 5 concurrent
const emailQueue = new AsyncQueue({ concurrency: 5 });

for (const recipient of recipients) {
  emailQueue.push(() => sendEmail(recipient));
}
```

### BullMQ — Production Job Queue

```javascript
// npm install bullmq ioredis
const { Queue, Worker } = require('bullmq');

const emailQueue = new Queue('emails', {
  connection: { host: 'localhost', port: 6379 },
});

// Producer
await emailQueue.add('send-welcome', {
  to: 'user@example.com',
  template: 'welcome',
}, {
  attempts: 3,
  backoff: { type: 'exponential', delay: 1000 },
});

// Consumer
const worker = new Worker('emails', async (job) => {
  await sendEmail(job.data);
}, { connection: { host: 'localhost', port: 6379 }, concurrency: 5 });
```

---

## Concurrency Limit (p-limit)

```javascript
// npm install p-limit
const pLimit = require('p-limit');

const limit = pLimit(5); // tối đa 5 concurrent

async function processAllUsers(userIds) {
  const tasks = userIds.map((id) =>
    limit(async () => {
      const user = await db.users.findById(id);
      await enrichUserData(user);
      return user;
    })
  );
  return Promise.all(tasks);
}

// 10,000 users — chỉ 5 DB queries cùng lúc
```

### Manual Implementation

```javascript
async function mapWithConcurrency(items, fn, concurrency) {
  const results = new Array(items.length);
  let index = 0;

  async function worker() {
    while (index < items.length) {
      const i = index++;
      results[i] = await fn(items[i], i);
    }
  }

  const workers = Array.from(
    { length: Math.min(concurrency, items.length) },
    () => worker()
  );
  await Promise.all(workers);
  return results;
}

await mapWithConcurrency(urls, fetchUrl, 10);
```

---

## Batch Processing

### Chunk + Process

```javascript
function chunk(array, size) {
  const chunks = [];
  for (let i = 0; i < array.length; i += size) {
    chunks.push(array.slice(i, i + size));
  }
  return chunks;
}

async function bulkInsert(records, batchSize = 500) {
  const batches = chunk(records, batchSize);

  for (const batch of batches) {
    await db.query(
      'INSERT INTO users (name, email) VALUES ' +
      batch.map((_, i) => `($${i * 2 + 1}, $${i * 2 + 2})`).join(', '),
      batch.flatMap((r) => [r.name, r.email])
    );
  }
}
```

### DataLoader — Batch + Cache

```javascript
// npm install dataloader
const DataLoader = require('dataloader');

const userLoader = new DataLoader(async (ids) => {
  // Gom tất cả IDs trong một event loop tick
  const users = await db.users.findMany({ where: { id: { in: ids } } });
  const map = new Map(users.map((u) => [u.id, u]));
  return ids.map((id) => map.get(id) ?? new Error(`User ${id} not found`));
});

// Nhiều resolvers gọi cùng lúc → 1 query
const [user1, user2] = await Promise.all([
  userLoader.load(1),
  userLoader.load(2),
]);
```

---

## Retry với Exponential Backoff

```javascript
async function retry(fn, {
  maxAttempts = 3,
  baseDelayMs = 1000,
  maxDelayMs = 30000,
  shouldRetry = () => true,
} = {}) {
  let lastError;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn(attempt);
    } catch (err) {
      lastError = err;
      if (attempt === maxAttempts || !shouldRetry(err)) {
        throw err;
      }
      const delay = Math.min(
        baseDelayMs * Math.pow(2, attempt - 1) + Math.random() * 1000,
        maxDelayMs
      );
      await new Promise((r) => setTimeout(r, delay));
    }
  }
  throw lastError;
}

// Gọi external API với retry
const data = await retry(
  () => fetch('https://api.example.com/data'),
  {
    shouldRetry: (err) => err.status >= 500 || err.code === 'ECONNRESET',
  }
);
```

### Jitter (Nhiễu Ngẫu Nhiên)

Thêm random jitter tránh **thundering herd (bầy đàn ồ ạt)** — nhiều clients retry cùng lúc sau outage.

```javascript
const delay = baseDelay * Math.pow(2, attempt) * (0.5 + Math.random() * 0.5);
```

---

## Use Cases Thực Tế

| Pattern | Use Case | Implementation |
| ------- | -------- | -------------- |
| **Debounce** | Search autocomplete, form validation | lodash.debounce hoặc custom |
| **Throttle** | Scroll events, metrics sampling | lodash.throttle |
| **Token bucket** | API rate limiting | express-rate-limit, custom |
| **Queue** | Email sending, image processing | BullMQ, custom AsyncQueue |
| **p-limit** | Bulk DB operations | p-limit package |
| **DataLoader** | GraphQL N+1 prevention | dataloader |
| **Retry + backoff** | External API calls | axios-retry, custom |
| **Batch insert** | Import CSV, bulk sync | chunk + SQL bulk |

### Express Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 phút
  max: 100,                 // 100 requests per IP
  standardHeaders: true,
  legacyHeaders: false,
});

app.use('/api/', limiter);
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Debounce vs Throttle?

**Gợi ý trả lời:** Debounce: chờ ngừng events rồi mới gọi (search input). Throttle: gọi tối đa 1 lần per interval dù events liên tục (scroll handler, API rate limit). Debounce = "chờ yên"; throttle = "giới hạn tần suất".

### Câu 2: Làm sao xử lý 10,000 records mà không overwhelm DB?

**Gợi ý trả lời:** Concurrency limit (p-limit, queue với concurrency=5-10), batch processing (chunk 500 records), bulk insert thay vì 10,000 individual queries. Monitor connection pool usage.

### Câu 3: DataLoader giải quyết vấn đề gì?

**Gợi ý trả lời:** N+1 query problem trong GraphQL/resolvers. Batch multiple `.load(id)` calls trong cùng tick thành một DB query. Có per-request cache — cùng id chỉ query một lần.

### Câu 4: Exponential backoff tại sao cần jitter?

**Gợi ý trả lời:** Sau outage, nhiều clients retry cùng lúc (thundering herd) gây spike load. Jitter randomize delay, spread retries over time, giúp service recover ổn định hơn.

### Câu 5: Mutex trong single-threaded Node.js có cần không?

**Gợi ý trả lời:** Có — cho async critical sections. Hai concurrent requests có thể interleave await points, gây race condition khi read-modify-write file hoặc shared resource. Mutex serialize async access.

---

**Hoàn thành chủ đề:** Quay lại [README.md](./README.md) hoặc tiếp tục [03-web-frameworks/](../03-web-frameworks/).
