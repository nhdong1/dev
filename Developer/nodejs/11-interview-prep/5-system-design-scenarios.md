# System Design Scenarios — Tình Huống Thiết Kế Hệ Thống

> Bài toán thiết kế hệ thống phỏng vấn Backend Node.js — URL shortener, chat app, notification system, rate limiter — với Node.js constraints và trade-offs thực tế.

## Mục Lục

1. [Framework Thiết Kế Hệ Thống](#framework-thiết-kế-hệ-thống)
2. [Node.js Constraints](#nodejs-constraints)
3. [Scenario 1: URL Shortener](#scenario-1-url-shortener)
4. [Scenario 2: Real-time Chat](#scenario-2-real-time-chat)
5. [Scenario 3: Notification System](#scenario-3-notification-system)
6. [Scenario 4: Rate Limiter Service](#scenario-4-rate-limiter-service)
7. [Scenario 5: E-commerce Order System](#scenario-5-e-commerce-order-system)
8. [Scenario 6: File Upload Service](#scenario-6-file-upload-service)
9. [Tips Phỏng Vấn System Design](#tips-phỏng-vấn-system-design)

---

## Framework Thiết Kế Hệ Thống

```
┌─────────────────────────────────────────────────────────┐
│  1. CLARIFY REQUIREMENTS (5 phút)                       │
│     Functional: features cần có                         │
│     Non-functional: scale, latency, availability        │
├─────────────────────────────────────────────────────────┤
│  2. ESTIMATION (5 phút)                                 │
│     QPS, storage, bandwidth — back-of-envelope          │
├─────────────────────────────────────────────────────────┤
│  3. HIGH-LEVEL DESIGN (10 phút)                         │
│     Components diagram: API, DB, cache, queue           │
├─────────────────────────────────────────────────────────┤
│  4. DEEP DIVE (20 phút)                                 │
│     API design, DB schema, algorithms                 │
├─────────────────────────────────────────────────────────┤
│  5. BOTTLENECKS & SCALING (10 phút)                     │
│     Single points of failure, Node.js specific issues   │
├─────────────────────────────────────────────────────────┤
│  6. TRADE-OFFS (5 phút)                                 │
│     Alternatives considered, why chose this approach    │
└─────────────────────────────────────────────────────────┘
```

---

## Node.js Constraints

Khi thiết kế với Node.js, **luôn đề cập** các constraints sau:

| Constraint | Impact | Giải Pháp |
| ---------- | ------ | --------- |
| **Single-threaded** | CPU-bound block Event Loop | Worker Threads, offload sang service khác |
| **Memory per process** | ~1.5GB default heap limit | Cluster mode, horizontal scaling |
| **I/O-bound strength** | Excellent cho concurrent I/O | Tận dụng non-blocking I/O |
| **No native threads** | Không phù hợp image processing | Child process, external service |
| **Callback/Promise overhead** | Nhỏ nhưng cần handle errors | async/await, proper error boundaries |

```
Node.js phù hợp:  API Gateway, BFF, I/O-heavy services, real-time
Node.js không phù hợp: Video encoding, ML inference, heavy computation
```

---

## Scenario 1: URL Shortener

### Requirements

**Functional:**
- Shorten long URL → short code (e.g., `abc123`)
- Redirect short URL → original URL
- Optional: custom alias, expiration, analytics

**Non-functional:**
- 100M URLs stored
- 1000:1 read:write ratio
- Redirect latency < 100ms
- 99.9% availability

### Estimation

```
Writes: 100M URLs / (365 * 24 * 3600) ≈ 3 writes/sec
Reads:  3 * 1000 = 3000 reads/sec

Storage: 100M * 500 bytes ≈ 50GB
```

### High-Level Design

```
Client → Load Balancer → Node.js API (Cluster) → Redis Cache
                                                   → PostgreSQL
```

### Deep Dive

**Short code generation:**

| Approach | Ưu | Nhược |
| -------- | -- | ----- |
| Auto-increment ID → Base62 | Unique, sortable | Predictable, single DB bottleneck |
| Random 6-char Base62 | Unpredictable | Collision risk (birthday paradox) |
| Pre-generated pool | Fast insert | Complexity |

```javascript
// Base62 encode
const CHARS = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
function encode(num) {
  let result = '';
  while (num > 0) {
    result = CHARS[num % 62] + result;
    num = Math.floor(num / 62);
  }
  return result;
}
```

**API Design:**

```
POST /api/shorten    { "url": "https://...", "customAlias?": "mylink" }
GET  /:code          → 301 Redirect
GET  /api/stats/:code → { clicks, createdAt }
```

**Caching strategy:**
- Cache-aside: `GET /:code` → check Redis → miss → PostgreSQL → cache (TTL 24h)
- 99% cache hit rate → 30 reads/sec to DB

**DB Schema:**

```sql
CREATE TABLE urls (
  id          BIGSERIAL PRIMARY KEY,
  short_code  VARCHAR(10) UNIQUE NOT NULL,
  long_url    TEXT NOT NULL,
  created_at  TIMESTAMP DEFAULT NOW(),
  expires_at  TIMESTAMP,
  user_id     BIGINT
);
CREATE INDEX idx_short_code ON urls(short_code);
```

---

## Scenario 2: Real-time Chat

### Requirements

**Functional:**
- 1:1 và group chat
- Message history
- Online/offline status
- Typing indicators

**Non-functional:**
- 1M concurrent connections
- Message delivery < 500ms
- Message ordering trong conversation

### High-Level Design

```
Client (WebSocket) → Socket.io Server (Node.js Cluster)
                          ↓
                    Redis Pub/Sub (cross-instance messaging)
                          ↓
                    PostgreSQL (message persistence)
                    Redis (online status, recent messages cache)
```

### Deep Dive

**Tại sao WebSocket + Node.js?**
- Persistent connections — Node.js handle 10k+ connections per instance tốt
- Event-driven phù hợp push notifications
- Socket.io có fallback (long polling) và room management

```javascript
io.on('connection', (socket) => {
  socket.on('join', (roomId) => socket.join(roomId));

  socket.on('message', async (data) => {
    const message = await saveMessage(data);
    io.to(data.roomId).emit('message', message);
    // Cross-instance: redis.publish(`room:${roomId}`, JSON.stringify(message))
  });
});
```

**Scaling strategy:**
- Sticky sessions (load balancer) hoặc Redis adapter cho Socket.io
- Horizontal scaling: thêm Node.js instances
- Message sharding by conversation ID

**Message ordering:**
- Server-assigned sequence number per conversation
- Client sort by sequence, handle gaps với fetch missing

---

## Scenario 3: Notification System

### Requirements

**Functional:**
- Push notification (mobile), email, SMS, in-app
- User preferences (opt-in/opt-out per channel)
- Template-based messages
- Scheduled notifications

**Non-functional:**
- 10M notifications/day
- At-least-once delivery
- Priority queue (urgent vs marketing)

### High-Level Design

```
API Service (Node.js) → BullMQ (Redis) → Worker Processes
                                              ├─ Email Worker (SendGrid)
                                              ├─ Push Worker (FCM/APNs)
                                              ├─ SMS Worker (Twilio)
                                              └─ In-app Worker (WebSocket)
```

### Deep Dive

```javascript
// Producer — API service
await notificationQueue.add('send-email', {
  userId: 123,
  template: 'order-confirmation',
  data: { orderId: 456 },
}, {
  priority: 1,        // 1 = highest
  attempts: 3,
  backoff: { type: 'exponential', delay: 5000 },
});

// Consumer — worker
notificationQueue.process('send-email', async (job) => {
  const user = await getUserPreferences(job.data.userId);
  if (!user.emailOptIn) return;
  await sendGrid.send(renderTemplate(job.data));
});
```

**Idempotency:** Dùng `notificationId` dedup — tránh gửi duplicate khi retry.

**Rate limiting per channel:** SendGrid 100 emails/sec → worker concurrency limit.

---

## Scenario 4: Rate Limiter Service

### Requirements

**Functional:**
- Rate limit per user/IP/API key
- Configurable limits per endpoint
- Return remaining quota in headers

**Non-functional:**
- Distributed (multiple API instances)
- Low latency (< 5ms overhead)
- Accurate counting

### High-Level Design

```
API Request → Rate Limiter Middleware → Redis (sliding window counter) → API Handler
```

### Deep Dive — Sliding Window với Redis

```javascript
async function isRateLimited(key, limit, windowSec) {
  const now = Date.now();
  const windowStart = now - windowSec * 1000;

  const multi = redis.multi();
  multi.zremrangebyscore(key, 0, windowStart);  // Remove old entries
  multi.zadd(key, now, `${now}`);              // Add current request
  multi.zcard(key);                            // Count in window
  multi.expire(key, windowSec);

  const [, , count] = await multi.exec();
  return count > limit;
}

// Middleware
async function rateLimiter(req, res, next) {
  const key = `rate:${req.user?.id || req.ip}`;
  const limited = await isRateLimited(key, 100, 60);
  if (limited) return res.status(429).json({ error: 'Rate limited' });
  next();
}
```

**Alternatives:**
- **Token bucket:** Cho phép burst, phức tạp hơn
- **Fixed window:** Đơn giản nhưng boundary burst problem
- **Dedicated service:** Kong, Envoy rate limiting — offload khỏi app

---

## Scenario 5: E-commerce Order System

### Requirements

**Functional:**
- Place order, payment, inventory management
- Order status tracking
- Handle concurrent purchases (last item)

**Non-functional:**
- 1000 orders/minute peak
- No overselling
- Payment idempotency

### High-Level Design

```
Order API (Node.js) → PostgreSQL (orders, inventory)
                   → Redis (inventory cache, idempotency keys)
                   → Payment Service (Stripe)
                   → BullMQ (order processing, email)
```

### Deep Dive — Prevent Overselling

```javascript
async function placeOrder(userId, items, idempotencyKey) {
  // Idempotency check
  const existing = await redis.get(`idempotency:${idempotencyKey}`);
  if (existing) return JSON.parse(existing);

  return prisma.$transaction(async (tx) => {
    for (const item of items) {
      const updated = await tx.product.updateMany({
        where: {
          id: item.productId,
          stock: { gte: item.quantity },
        },
        data: { stock: { decrement: item.quantity } },
      });
      if (updated.count === 0) {
        throw new InsufficientStockError(item.productId);
      }
    }

    const order = await tx.order.create({
      data: { userId, items: { create: items }, status: 'PENDING' },
    });

    await redis.setex(`idempotency:${idempotencyKey}`, 86400, JSON.stringify(order));
    return order;
  }, { isolationLevel: 'Serializable' }); // Hoặc Optimistic locking
});
```

**Saga pattern** cho distributed transaction (order → payment → shipping):

```
Order Created → Payment Requested → Payment Confirmed → Shipping Initiated
                     ↓ fail
              Compensate: Release Inventory
```

---

## Scenario 6: File Upload Service

### Requirements

**Functional:**
- Upload files up to 5GB
- Resume interrupted uploads
- Generate thumbnails (images)

**Non-functional:**
- High throughput
- No memory exhaustion on server

### High-Level Design

```
Client → Node.js API (presigned URL) → S3/MinIO (direct upload)
                                     → BullMQ → Worker (thumbnail generation)
```

### Deep Dive — Tại sao không upload qua Node.js?

```javascript
// ❌ Upload qua Node.js — memory risk với file lớn
app.post('/upload', upload.single('file'), handler);

// ✅ Presigned URL — client upload trực tiếp lên S3
app.post('/upload/init', async (req, res) => {
  const { filename, contentType } = req.body;
  const key = `uploads/${uuid()}/${filename}`;

  const presignedUrl = await s3.getSignedUrl('putObject', {
    Bucket: 'my-bucket',
    Key: key,
    ContentType: contentType,
    Expires: 3600,
  });

  res.json({ uploadUrl: presignedUrl, key });
});
```

**Multipart upload:** S3 multipart cho file > 100MB, client upload từng part, resume khi fail.

**Thumbnail:** Worker Thread hoặc separate service (không block Event Loop):

```javascript
// Job queue cho image processing
await thumbnailQueue.add('generate', { s3Key, sizes: [150, 300, 600] });
```

---

## Tips Phỏng Vấn System Design

### Nên Làm

1. **Hỏi clarifying questions** — "Có bao nhiêu users?", "Read-heavy hay write-heavy?"
2. **Bắt đầu simple** — monolith + single DB, rồi scale khi cần
3. **Đề cập trade-offs** — "Chọn Redis vì latency thấp, trade-off là consistency"
4. **Vẽ diagram** — boxes và arrows, label protocols
5. **Nói về monitoring** — "Cần alert khi error rate > 1%"

### Không Nên

1. Nhảy thẳng vào microservices mà chưa justify
2. Over-engineer cho scale không realistic
3. Quên single points of failure
4. Bỏ qua data consistency requirements
5. Không đề cập Node.js limitations khi relevant

### Câu Hỏi Thường Gặp Thêm

| Bài Toán | Focus |
| -------- | ----- |
| Twitter/X feed | Fan-out on write vs read, caching |
| Uber matching | Geospatial indexing, real-time |
| Netflix streaming | CDN, adaptive bitrate (không phải Node.js core) |
| WhatsApp messaging | WebSocket scaling, message queue |
| Distributed cache | Consistent hashing, eviction policies |

---

## Tài Liệu Tham Khảo

- [08-architecture/4-microservices.md](../08-architecture/4-microservices.md)
- [07-performance/4-caching-strategies.md](../07-performance/4-caching-strategies.md)
- [10-advanced/4-websockets.md](../10-advanced/4-websockets.md)
- [10-advanced/7-job-queues.md](../10-advanced/7-job-queues.md)
- [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) — Câu 41–50
