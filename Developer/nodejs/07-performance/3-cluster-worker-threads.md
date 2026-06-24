# Cluster Module & Worker Threads — Multi-Process Scaling và CPU Offloading

> Node.js single-threaded nhưng có thể scale qua **Cluster module (mô-đun cụm tiến trình)** cho I/O workloads và **Worker Threads (luồng worker)** cho CPU-intensive tasks (tác vụ nặng CPU). Hiểu trade-offs giữa hai approach là kỹ năng production quan trọng.

## Mục Lục

1. [Tại Sao Cần Scale](#tại-sao-cần-scale)
2. [Cluster Module](#cluster-module)
3. [PM2 Cluster Mode](#pm2-cluster-mode)
4. [Worker Threads](#worker-threads)
5. [Cluster vs Worker Threads](#cluster-vs-worker-threads)
6. [Sticky Sessions](#sticky-sessions)
7. [Best Practices](#best-practices)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Scale

Modern servers có **multi-core CPU** — một Node.js process chỉ dùng **1 core** cho JavaScript execution.

```
Single Process:                    Cluster (4 workers):
┌─────────┐                       ┌─────────┐
│ Core 1  │ ← 100% Node.js        │ Core 1  │ ← Worker 1
│ Core 2  │   idle                │ Core 2  │ ← Worker 2
│ Core 3  │   idle                │ Core 3  │ ← Worker 3
│ Core 4  │   idle                │ Core 4  │ ← Worker 4
└─────────┘                       └─────────┘
Throughput: ~5k RPS                Throughput: ~18k RPS (không linear 4x)
```

**Lưu ý:** Scaling không linear vì shared resources (DB connections, Redis) và OS scheduling overhead.

---

## Cluster Module

Cluster tạo **multiple child processes** chia sẻ cùng server port — OS load balance connections.

```typescript
import cluster from 'node:cluster';
import os from 'node:os';
import process from 'node:process';

const NUM_WORKERS = parseInt(process.env.WEB_CONCURRENCY || String(os.cpus().length));

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} starting ${NUM_WORKERS} workers`);

  for (let i = 0; i < NUM_WORKERS; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died (${signal || code}). Restarting...`);
    cluster.fork(); // Auto-restart crashed worker
  });

  // Graceful shutdown
  process.on('SIGTERM', () => {
    for (const id in cluster.workers) {
      cluster.workers[id]?.kill('SIGTERM');
    }
  });
} else {
  // Worker process — chạy Express app
  import('./server.js');
  console.log(`Worker ${process.pid} started`);
}
```

### Express Server Trong Worker

```typescript
// server.ts — được import bởi worker
import express from 'express';

const app = express();
const PORT = process.env.PORT || 3000;

app.get('/api/health', (_req, res) => {
  res.json({ pid: process.pid, status: 'ok' });
});

app.listen(Number(PORT), () => {
  console.log(`Worker ${process.pid} listening on ${PORT}`);
});

export default app;
```

### Connection Distribution

Node.js cluster dùng **round-robin** (mặc định trên Windows và Node 16+) để phân phối connections:

```
Client connections → Primary (optional) → Workers
                      ├── Worker 1
                      ├── Worker 2
                      └── Worker 3
```

---

## PM2 Cluster Mode

[PM2](https://pm2.keymetrics.io/) là process manager (trình quản lý tiến trình) production phổ biến — abstract cluster logic.

```bash
npm install -g pm2

# Cluster mode — 1 instance per CPU core
pm2 start dist/server.js -i max --name api

# Hoặc chỉ định số workers
pm2 start dist/server.js -i 4 --name api

# Zero-downtime reload
pm2 reload api

# Monitoring
pm2 monit
pm2 logs api
```

### ecosystem.config.js

```javascript
module.exports = {
  apps: [{
    name: 'api',
    script: './dist/server.js',
    instances: 'max',        // hoặc số cụ thể: 4
    exec_mode: 'cluster',
    max_memory_restart: '512M',
    env: {
      NODE_ENV: 'production',
      PORT: 3000,
    },
    kill_timeout: 5000,      // Graceful shutdown window
    listen_timeout: 10000,
  }],
};
```

| PM2 Feature | Mô Tả |
| ----------- | ----- |
| `instances: 'max'` | 1 worker per CPU core |
| `max_memory_restart` | Auto-restart khi vượt memory |
| `pm2 reload` | Zero-downtime rolling restart |
| `pm2 scale api 8` | Scale workers runtime |

---

## Worker Threads

Worker Threads chạy JavaScript trên **threads riêng** trong cùng process — phù hợp **CPU-bound tasks** mà cần share memory.

```typescript
// main.ts
import { Worker, isMainThread, parentPort, workerData } from 'node:worker_threads';

if (isMainThread) {
  export function hashPassword(password: string): Promise<string> {
    return new Promise((resolve, reject) => {
      const worker = new Worker(new URL('./hash-worker.ts', import.meta.url), {
        workerData: { password },
      });

      worker.on('message', resolve);
      worker.on('error', reject);
      worker.on('exit', (code) => {
        if (code !== 0) reject(new Error(`Worker stopped with code ${code}`));
      });
    });
  }
}
```

```typescript
// hash-worker.ts
import { parentPort, workerData } from 'node:worker_threads';
import bcrypt from 'bcrypt';

const hash = bcrypt.hashSync(workerData.password, 12);
parentPort?.postMessage(hash);
```

### Worker Pool Pattern

Tạo pool workers tái sử dụng — tránh spawn worker mỗi request:

```typescript
import { Worker } from 'node:worker_threads';
import os from 'node:os';

class WorkerPool {
  private workers: Worker[] = [];
  private queue: Array<{ data: unknown; resolve: Function; reject: Function }> = [];
  private available: Worker[] = [];

  constructor(
    private workerPath: string,
    poolSize = os.cpus().length,
  ) {
    for (let i = 0; i < poolSize; i++) {
      const worker = new Worker(workerPath);
      worker.on('message', (result) => this.onWorkerDone(worker, result));
      worker.on('error', (err) => this.onWorkerError(worker, err));
      this.workers.push(worker);
      this.available.push(worker);
    }
  }

  exec<T>(data: unknown): Promise<T> {
    return new Promise((resolve, reject) => {
      const worker = this.available.pop();
      if (worker) {
        worker.postMessage(data);
        (worker as any).__currentResolve = resolve;
        (worker as any).__currentReject = reject;
      } else {
        this.queue.push({ data, resolve, reject });
      }
    });
  }

  private onWorkerDone(worker: Worker, result: unknown) {
    (worker as any).__currentResolve?.(result);
    this.processQueue(worker);
  }

  private processQueue(worker: Worker) {
    const next = this.queue.shift();
    if (next) {
      worker.postMessage(next.data);
      (worker as any).__currentResolve = next.resolve;
      (worker as any).__currentReject = next.reject;
    } else {
      this.available.push(worker);
    }
  }

  private onWorkerError(worker: Worker, err: Error) {
    (worker as any).__currentReject?.(err);
    this.available.push(worker);
  }

  async destroy() {
    await Promise.all(this.workers.map(w => w.terminate()));
  }
}
```

### SharedArrayBuffer (Advanced)

```typescript
// Chia sẻ memory giữa main thread và workers
const sharedBuffer = new SharedArrayBuffer(1024);
const sharedArray = new Int32Array(sharedBuffer);

const worker = new Worker('./worker.js', {
  workerData: { sharedBuffer },
});
```

---

## Cluster vs Worker Threads

| Tiêu Chí | Cluster Module | Worker Threads |
| -------- | -------------- | -------------- |
| **Đơn vị** | Process (riêng memory) | Thread (shared memory) |
| **Use case** | Scale HTTP server, I/O | CPU-intensive computation |
| **Memory** | Riêng mỗi worker | Shared heap (tiết kiệm hơn) |
| **Crash isolation** | Worker crash không ảnh hưởng others | Thread crash có thể crash process |
| **Startup cost** | Cao hơn (fork process) | Thấp hơn |
| **State sharing** | Không — cần Redis/DB | SharedArrayBuffer, MessageChannel |
| **Production tool** | PM2, K8s replicas | Custom pool |

```
HTTP API scaling:     Cluster / PM2 / K8s pods
Image resize:         Worker Threads pool
PDF generation:       Worker Threads hoặc separate microservice
Blockchain hashing:   Worker Threads hoặc child_process
```

---

## Sticky Sessions

Một số use cases cần **sticky sessions (phiên dính)** — cùng client luôn đến cùng worker:

| Use Case | Cần Sticky? |
| -------- | ----------- |
| Stateless REST API | Không |
| In-memory session store | Có — hoặc dùng Redis sessions |
| WebSocket connections | Có — connection persistent |
| Local cache per worker | Có — hoặc dùng distributed cache |

```typescript
// cluster với sticky (legacy — prefer Redis sessions)
import cluster from 'node:cluster';
import http from 'node:http';

if (cluster.isPrimary) {
  const workers: typeof cluster.Worker[] = [];
  let nextWorker = 0;

  for (let i = 0; i < os.cpus().length; i++) {
    workers.push(cluster.fork());
  }

  http.createServer((req, res) => {
    const worker = workers[nextWorker];
    nextWorker = (nextWorker + 1) % workers.length;
    worker?.send('sticky:session', req, res); // Simplified — dùng thư viện sticky-session
  }).listen(8000);
}
```

**Khuyến nghị production:** Dùng **Redis session store** hoặc **JWT stateless auth** — tránh sticky sessions phức tạp.

---

## Best Practices

1. **Cluster cho HTTP servers** — PM2 hoặc K8s replicas thay vì tự implement
2. **Worker pool cho CPU tasks** — Không spawn worker per request
3. **Size workers = CPU cores** — Nhiều hơn không giúp CPU-bound, có thể gây context switching
4. **Graceful shutdown** — Drain connections trước khi kill worker
5. **Connection pool per worker** — Tổng DB connections = `pool_size × num_workers`
6. **Health checks per worker** — K8s liveness probe hit load balancer
7. **Không share mutable state** — Dùng Redis/DB cho shared state

### DB Pool Sizing Với Cluster

```
4 workers × 10 connections/worker = 40 total DB connections
→ Đảm bảo < PostgreSQL max_connections (thường 100)
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Gợi Ý Trả Lời |
| ------- | ------------- |
| Scale Node.js như thế nào? | Cluster/PM2 cho multi-core, horizontal scaling với load balancer + multiple instances |
| Cluster vs Worker Threads? | Cluster: separate processes, I/O scaling. Worker Threads: same process, CPU offload |
| Tại sao không spawn 100 workers? | Diminishing returns, DB connection exhaustion, context switching overhead |
| Worker Thread vs child_process? | Worker Threads: lighter, shared memory. child_process: full isolation, IPC overhead |
| Sticky sessions cần khi nào? | WebSocket, in-memory state — prefer external session store thay vì sticky |
| PM2 cluster vs K8s replicas? | PM2: single host. K8s: multi-host, auto-scaling, self-healing |

---

**Tiếp theo:** [4-caching-strategies.md](./4-caching-strategies.md) — Cache-aside, write-through và TTL patterns
