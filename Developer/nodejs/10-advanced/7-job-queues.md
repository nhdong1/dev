# Job Queues — BullMQ, Agenda, Scheduled Tasks và Retry

> Job Queues (Hàng Đợi Công Việc) cho phép xử lý tác vụ bất đồng bộ (asynchronous tasks) — email sending, report generation, image processing — tách khỏi request/response cycle chính của API.

## Mục Lục

1. [Tại Sao Cần Job Queues](#tại-sao-cần-job-queues)
2. [BullMQ Fundamentals](#bullmq-fundamentals)
3. [Queue, Worker, và Job Lifecycle](#queue-worker-và-job-lifecycle)
4. [Retry Policies và Error Handling](#retry-policies-và-error-handling)
5. [Scheduled Jobs và Cron](#scheduled-jobs-và-cron)
6. [Job Priorities và Concurrency](#job-priorities-và-concurrency)
7. [Agenda — MongoDB-based Scheduler](#agenda--mongodb-based-scheduler)
8. [Monitoring và Dashboard](#monitoring-và-dashboard)
9. [Production Patterns](#production-patterns)
10. [Best Practices](#best-practices)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Job Queues

### Vấn Đề: Sync Processing trong API

```javascript
// ❌ User đợi 30 giây cho email + report generation
app.post('/orders', async (req, res) => {
  const order = await createOrder(req.body);
  await sendConfirmationEmail(order);      // 5 seconds
  await generateInvoicePDF(order);         // 10 seconds
  await notifyWarehouse(order);            // 3 seconds
  await updateAnalytics(order);            // 2 seconds
  res.json(order); // User waited 20+ seconds!
});
```

### Giải Pháp: Queue Background Jobs

```javascript
// ✅ User nhận response ngay lập tức
app.post('/orders', async (req, res) => {
  const order = await createOrder(req.body);

  await orderQueue.add('process-order', {
    orderId: order.id,
    email: order.customerEmail,
  });

  res.status(202).json({ order, message: 'Order received, processing...' });
});

// Worker xử lý background
orderWorker.process(async (job) => {
  const { orderId } = job.data;
  await sendConfirmationEmail(orderId);
  await generateInvoicePDF(orderId);
  await notifyWarehouse(orderId);
});
```

```
API Request Flow:
Client ──POST /orders──► API Server ──► Response 202 (immediate)
                              │
                              ▼
                         Job Queue (Redis)
                              │
                              ▼
                         Worker Process ──► Email, PDF, Notify
```

| Use Case | Ví Dụ |
| -------- | ----- |
| Email/SMS notifications | Welcome email, password reset |
| Report generation | Daily sales report, PDF export |
| Image/video processing | Resize, transcode, thumbnail |
| Data import/export | CSV import, bulk operations |
| Webhook delivery | Retry failed webhook calls |
| Scheduled tasks | Cleanup, reminders, billing |

---

## BullMQ Fundamentals

BullMQ là job queue library hiện đại cho Node.js — built on Redis, successor của Bull.

### Cài Đặt

```bash
npm install bullmq ioredis
```

### Basic Setup

```javascript
// queue.js
const { Queue } = require('bullmq');
const IORedis = require('ioredis');

const connection = new IORedis({
  host: process.env.REDIS_HOST || 'localhost',
  port: 6379,
  maxRetriesPerRequest: null, // Required for BullMQ
});

const emailQueue = new Queue('email', { connection });

module.exports = { emailQueue, connection };
```

```javascript
// worker.js
const { Worker } = require('bullmq');
const { connection } = require('./queue');

const emailWorker = new Worker(
  'email',
  async (job) => {
    const { to, subject, body } = job.data;
    console.log(`Sending email to ${to}: ${subject}`);
    await sendEmail({ to, subject, body });
    return { sent: true, timestamp: Date.now() };
  },
  {
    connection,
    concurrency: 5, // Process 5 jobs simultaneously
  }
);

emailWorker.on('completed', (job, result) => {
  console.log(`Job ${job.id} completed:`, result);
});

emailWorker.on('failed', (job, err) => {
  console.error(`Job ${job.id} failed:`, err.message);
});

module.exports = { emailWorker };
```

### Add Jobs

```javascript
const { emailQueue } = require('./queue');

// Simple job
await emailQueue.add('welcome-email', {
  to: 'user@example.com',
  subject: 'Welcome!',
  body: 'Thanks for signing up',
});

// Job với options
await emailQueue.add(
  'password-reset',
  { to: 'user@example.com', token: 'abc123' },
  {
    attempts: 3,
    backoff: { type: 'exponential', delay: 1000 },
    removeOnComplete: 100,
    removeOnFail: 50,
    priority: 1, // Lower number = higher priority
  }
);
```

---

## Queue, Worker, và Job Lifecycle

### Job States

```
waiting ──► active ──► completed
    │          │
    │          └──► failed ──► waiting (retry)
    │
    └──► delayed ──► waiting
              │
              └──► paused
```

| State | Mô Tả |
| ----- | ----- |
| `waiting` | Trong queue, chờ worker |
| `active` | Worker đang xử lý |
| `completed` | Xử lý thành công |
| `failed` | Xử lý thất bại (sau hết retries) |
| `delayed` | Scheduled cho tương lai |
| `paused` | Queue bị pause |

### Job Events

```javascript
const { QueueEvents } = require('bullmq');

const queueEvents = new QueueEvents('email', { connection });

queueEvents.on('completed', ({ jobId, returnvalue }) => {
  console.log(`Job ${jobId} completed with:`, returnvalue);
});

queueEvents.on('failed', ({ jobId, failedReason }) => {
  console.error(`Job ${jobId} failed:`, failedReason);
});

queueEvents.on('progress', ({ jobId, data }) => {
  console.log(`Job ${jobId} progress:`, data);
});
```

### Progress Reporting

```javascript
const reportWorker = new Worker('reports', async (job) => {
  const total = 1000;
  const results = [];

  for (let i = 0; i < total; i++) {
    results.push(await processRecord(i));

    if (i % 100 === 0) {
      await job.updateProgress((i / total) * 100);
    }
  }

  return { processed: results.length };
}, { connection });
```

---

## Retry Policies và Error Handling

### Exponential Backoff

```javascript
await queue.add('unreliable-task', data, {
  attempts: 5,
  backoff: {
    type: 'exponential',
    delay: 1000, // 1s, 2s, 4s, 8s, 16s
  },
});

// Custom backoff
backoff: {
  type: 'custom',
},

// Trong worker
const worker = new Worker('queue', processor, {
  settings: {
    backoffStrategy: (attemptsMade) => {
      return Math.min(attemptsMade * 2000, 30000); // Max 30s
    },
  },
});
```

### Dead Letter Queue (DLQ)

```javascript
const { Queue, Worker } = require('bullmq');

const mainQueue = new Queue('orders', { connection });
const dlq = new Queue('orders-dlq', { connection });

const worker = new Worker('orders', async (job) => {
  try {
    await processOrder(job.data);
  } catch (err) {
    if (job.attemptsMade >= job.opts.attempts) {
      // Move to DLQ after all retries exhausted
      await dlq.add('failed-order', {
        originalJob: job.data,
        error: err.message,
        failedAt: new Date().toISOString(),
        attempts: job.attemptsMade,
      });
    }
    throw err; // Re-throw để BullMQ handle retry
  }
}, { connection });
```

### Idempotent Job Processing

```javascript
const processedJobs = new Set(); // Hoặc Redis SET

const worker = new Worker('payments', async (job) => {
  const idempotencyKey = `payment:${job.data.orderId}`;

  const alreadyProcessed = await redis.get(idempotencyKey);
  if (alreadyProcessed) {
    console.log('Already processed, skipping');
    return JSON.parse(alreadyProcessed);
  }

  const result = await chargePayment(job.data);

  await redis.setex(idempotencyKey, 86400, JSON.stringify(result));
  return result;
}, { connection });
```

---

## Scheduled Jobs và Cron

### Delayed Jobs

```javascript
// Chạy sau 1 giờ
await queue.add('reminder', { userId: '123' }, {
  delay: 60 * 60 * 1000,
});

// Chạy vào thời điểm cụ thể
const runAt = new Date('2026-12-25T00:00:00Z');
await queue.add('christmas-promo', data, {
  delay: runAt.getTime() - Date.now(),
});
```

### Repeatable Jobs (Cron)

```javascript
const { Queue } = require('bullmq');

const cronQueue = new Queue('cron-jobs', { connection });

// Mỗi ngày 8:00 AM
await cronQueue.add(
  'daily-report',
  { type: 'sales' },
  {
    repeat: {
      pattern: '0 8 * * *', // Cron expression
      tz: 'Asia/Ho_Chi_Minh',
    },
    jobId: 'daily-sales-report', // Unique ID — prevent duplicates
  }
);

// Mỗi 5 phút
await cronQueue.add('health-check', {}, {
  repeat: { every: 5 * 60 * 1000 },
});

// Worker xử lý cron jobs
const cronWorker = new Worker('cron-jobs', async (job) => {
  switch (job.name) {
    case 'daily-report':
      return await generateDailyReport(job.data);
    case 'health-check':
      return await runHealthCheck();
  }
}, { connection });
```

### Remove Repeatable Job

```javascript
const repeatableJobs = await cronQueue.getRepeatableJobs();
for (const job of repeatableJobs) {
  if (job.name === 'daily-report') {
    await cronQueue.removeRepeatableByKey(job.key);
  }
}
```

---

## Job Priorities và Concurrency

### Priority Queue

```javascript
// Priority: 1 = highest, higher number = lower priority
await queue.add('urgent-notification', data, { priority: 1 });
await queue.add('regular-email', data, { priority: 5 });
await queue.add('bulk-newsletter', data, { priority: 10 });
```

### Concurrency Control

```javascript
// Global concurrency per worker
const worker = new Worker('api-calls', processor, {
  concurrency: 10,
});

// Rate limiting
const worker = new Worker('api-calls', processor, {
  limiter: {
    max: 100,      // Max 100 jobs
    duration: 60000, // Per 60 seconds
  },
});
```

### Multiple Workers cho Different Job Types

```javascript
// email-worker.js — high concurrency
new Worker('notifications', emailProcessor, { concurrency: 20 });

// pdf-worker.js — low concurrency (CPU intensive)
new Worker('reports', pdfProcessor, { concurrency: 2 });

// Mỗi worker có thể chạy separate process/container
```

---

## Agenda — MongoDB-based Scheduler

Agenda dùng MongoDB thay vì Redis — phù hợp nếu đã có MongoDB trong stack.

```bash
npm install agenda
```

```javascript
const Agenda = require('agenda');

const agenda = new Agenda({
  db: { address: process.env.MONGODB_URI, collection: 'jobs' },
  processEvery: '30 seconds',
});

// Define jobs
agenda.define('send email', async (job) => {
  const { to, subject, body } = job.attrs.data;
  await sendEmail({ to, subject, body });
});

agenda.define('generate report', { priority: 'high', concurrency: 2 }, async (job) => {
  await generateReport(job.attrs.data);
});

// Schedule jobs
(async () => {
  await agenda.start();

  // Run once
  await agenda.now('send email', { to: 'user@example.com', subject: 'Hi' });

  // Schedule for later
  await agenda.schedule('in 1 hour', 'send email', { to: 'user@example.com' });

  // Recurring
  await agenda.every('0 8 * * *', 'generate report', { type: 'daily' });
  await agenda.every('15 minutes', 'health-check');
})();
```

### BullMQ vs Agenda

| Feature | BullMQ | Agenda |
| ------- | ------ | ------ |
| Storage | Redis | MongoDB |
| Performance | Rất cao | Trung bình |
| Persistence | Redis AOF/RDB | MongoDB native |
| UI/Dashboard | Bull Board | Không built-in |
| Job events | Rich events | Basic |
| Best for | High throughput | MongoDB shops, simpler needs |

---

## Monitoring và Dashboard

### Bull Board — Web UI

```bash
npm install @bull-board/api @bull-board/express
```

```javascript
const express = require('express');
const { createBullBoard } = require('@bull-board/api');
const { BullMQAdapter } = require('@bull-board/api/bullMQAdapter');
const { ExpressAdapter } = require('@bull-board/express');
const { emailQueue, orderQueue } = require('./queues');

const serverAdapter = new ExpressAdapter();
serverAdapter.setBasePath('/admin/queues');

createBullBoard({
  queues: [
    new BullMQAdapter(emailQueue),
    new BullMQAdapter(orderQueue),
  ],
  serverAdapter,
});

const app = express();
app.use('/admin/queues', serverAdapter.getRouter());
app.listen(3002, () => {
  console.log('Bull Board at http://localhost:3002/admin/queues');
});
```

### Metrics với Prometheus

```javascript
const { Queue } = require('bullmq');
const client = require('prom-client');

const queueSize = new client.Gauge({
  name: 'bullmq_queue_size',
  help: 'Number of jobs in queue',
  labelNames: ['queue', 'status'],
});

async function collectMetrics(queue) {
  const waiting = await queue.getWaitingCount();
  const active = await queue.getActiveCount();
  const failed = await queue.getFailedCount();

  queueSize.set({ queue: queue.name, status: 'waiting' }, waiting);
  queueSize.set({ queue: queue.name, status: 'active' }, active);
  queueSize.set({ queue: queue.name, status: 'failed' }, failed);
}

setInterval(() => collectMetrics(emailQueue), 15000);
```

---

## Production Patterns

### Graceful Shutdown

```javascript
const { Worker } = require('bullmq');

const worker = new Worker('orders', processor, { connection });

async function shutdown() {
  console.log('Shutting down worker...');
  await worker.close(); // Wait for current jobs to complete
  await connection.quit();
  process.exit(0);
}

process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);
```

### Separate API và Worker Processes

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  API Server │────►│    Redis    │◄────│   Worker    │
│  (add jobs) │     │   (queue)   │     │ (process)   │
└─────────────┘     └─────────────┘     └─────────────┘
     x3 replicas          │               x2 replicas
                           │
                    ┌──────┴──────┐
                    │   Worker    │
                    │  (process)  │
                    └─────────────┘
```

```dockerfile
# docker-compose.yml
services:
  api:
    build: .
    command: node dist/api.js
    environment:
      - REDIS_URL=redis://redis:6379

  worker:
    build: .
    command: node dist/worker.js
    deploy:
      replicas: 2
    environment:
      - REDIS_URL=redis://redis:6379

  redis:
    image: redis:7-alpine
```

### Flow Producer — Job Dependencies

```javascript
const { FlowProducer } = require('bullmq');

const flowProducer = new FlowProducer({ connection });

// Parent job chờ tất cả children complete
await flowProducer.add({
  name: 'process-order',
  queueName: 'orders',
  data: { orderId: '123' },
  children: [
    { name: 'validate-payment', queueName: 'payments', data: { orderId: '123' } },
    { name: 'check-inventory', queueName: 'inventory', data: { orderId: '123' } },
  ],
});
```

---

## Best Practices

| Practice | Lý Do |
| -------- | ----- |
| Idempotent job handlers | Jobs có thể retry/duplicate |
| Set job timeouts | Prevent stuck jobs |
| Limit payload size | Large data → store in DB/S3, pass ID only |
| Monitor queue depth | Alert khi backlog tăng |
| Separate queues by priority/type | Isolate failures, scale independently |
| Dead letter queue | Manual review failed jobs |
| Graceful shutdown | Không mất jobs khi deploy |

### Job Data Design

```javascript
// ❌ Large payload in queue
await queue.add('process', { csvData: hugeCSVString });

// ✅ Reference only
await queue.add('process', { fileId: 'file-123', s3Key: 'uploads/data.csv' });
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Trả Lời Ngắn |
| ------- | ------------ |
| Tại sao cần job queue? | Decouple long-running tasks khỏi API response, retry, scheduling |
| BullMQ vs Agenda? | BullMQ: Redis, high performance; Agenda: MongoDB, simpler |
| Exponential backoff là gì? | Retry delay tăng dần — 1s, 2s, 4s, 8s — tránh overwhelm |
| Dead letter queue (DLQ)? | Queue chứa jobs failed sau hết retries — manual review |
| Idempotency trong jobs? | Same job chạy nhiều lần → same result, không duplicate side effects |
| Worker concurrency? | Số jobs xử lý parallel per worker instance |
| Làm sao scale workers? | Thêm worker instances/containers, Redis handle distribution |
| Cron job duplicate prevention? | Unique `jobId` cho repeatable jobs |
