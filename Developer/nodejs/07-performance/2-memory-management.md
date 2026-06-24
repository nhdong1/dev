# Memory Management — Heap Snapshots, GC và Memory Leak Detection

> V8 Engine quản lý memory (bộ nhớ) tự động qua Garbage Collection (GC — Thu Gom Rác), nhưng memory leaks (rò rỉ bộ nhớ) vẫn xảy ra khi objects không được giải phóng. Đây là nguyên nhân phổ biến khiến Node.js process crash với OOM (Out Of Memory — Hết Bộ Nhớ).

## Mục Lục

1. [V8 Memory Model](#v8-memory-model)
2. [Đọc Memory Usage](#đọc-memory-usage)
3. [Garbage Collection](#garbage-collection)
4. [Memory Leak Patterns](#memory-leak-patterns)
5. [Heap Snapshots](#heap-snapshots)
6. [Chrome DevTools Memory Profiling](#chrome-devtools-memory-profiling)
7. [Production Memory Monitoring](#production-memory-monitoring)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## V8 Memory Model

```
┌─────────────────────────────────────────────────┐
│                  V8 Heap                         │
│  ┌──────────────┐  ┌──────────────────────────┐ │
│  │  New Space   │  │      Old Space           │ │
│  │  (Young Gen) │  │      (Old Generation)    │ │
│  │  Short-lived │  │      Long-lived objects  │ │
│  └──────────────┘  └──────────────────────────┘ │
│  ┌──────────────┐  ┌──────────────────────────┐ │
│  │  Code Space  │  │   Large Object Space     │ │
│  └──────────────┘  └──────────────────────────┘ │
└─────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│  Stack          │  Primitives, function call frames
│  (per thread)   │
└─────────────────┘
```

| Vùng | Mô Tả |
| ---- | ----- |
| **New Space (Young Generation)** | Objects mới tạo — GC nhanh (Scavenge) |
| **Old Space (Old Generation)** | Objects survive nhiều GC cycles — GC chậm hơn (Mark-Sweep-Compact) |
| **Large Object Space** | Objects > ~1MB — allocate riêng |
| **Code Space** | JIT-compiled code |

---

## Đọc Memory Usage

### `process.memoryUsage()`

```typescript
function logMemory(label: string) {
  const mem = process.memoryUsage();
  console.log(`[${label}]`, {
    rss: `${(mem.rss / 1024 / 1024).toFixed(1)} MB`,           // Resident Set Size — tổng RAM process dùng
    heapTotal: `${(mem.heapTotal / 1024 / 1024).toFixed(1)} MB`,
    heapUsed: `${(mem.heapUsed / 1024 / 1024).toFixed(1)} MB`,
    external: `${(mem.external / 1024 / 1024).toFixed(1)} MB`, // C++ objects (Buffers)
    arrayBuffers: `${(mem.arrayBuffers / 1024 / 1024).toFixed(1)} MB`,
  });
}

// Gọi định kỳ
setInterval(() => logMemory('periodic'), 30_000);
```

| Field | Ý Nghĩa |
| ----- | ------- |
| **rss** | Tổng RAM process chiếm — bao gồm heap, stack, code |
| **heapUsed** | JavaScript objects đang dùng |
| **external** | Memory của Buffer, native addons |
| **arrayBuffers** | ArrayBuffer và SharedArrayBuffer |

### Dấu Hiệu Memory Leak

```
Heap Used over time:
     │
 500 │                              ╱
 MB  │                         ╱╱╱╱
     │                    ╱╱╱╱
 200 │              ╱╱╱╱╱
     │         ╱╱╱╱
 100 │────────╱  ← Healthy: plateau sau warmup
     └────────────────────────────► Time
```

- **Healthy:** Heap tăng lúc startup, plateau ổn định
- **Leak:** Heap tăng tuyến tính, không giảm sau GC

---

## Garbage Collection

### GC Types

| Loại GC | Vùng | Tần Suất | Pause Time |
| ------- | ---- | -------- | ---------- |
| **Scavenge** | New Space | Thường xuyên | < 5ms |
| **Mark-Sweep** | Old Space | Khi Old Space đầy | 10–100ms+ |
| **Mark-Compact** | Old Space | Khi fragmentation cao | Có thể > 100ms |

### Trace GC Logs

```bash
# Log mọi GC event
node --trace-gc dist/server.js

# Chi tiết hơn
node --trace-gc --trace-gc-verbose dist/server.js

# Expose manual GC (chỉ development!)
node --expose-gc dist/server.js
```

```typescript
// Manual GC — chỉ dùng khi debug, không production
if (global.gc) {
  global.gc();
  logMemory('after manual GC');
}
```

### GC Pressure — Áp Lực GC

Tạo quá nhiều short-lived objects → GC chạy liên tục → **GC pause** ảnh hưởng latency:

```typescript
// ❌ Tạo object mới mỗi request — GC pressure
app.use((req, res, next) => {
  req.context = { timestamp: Date.now(), id: crypto.randomUUID(), meta: {} };
  next();
});

// ✅ Reuse hoặc chỉ tạo khi cần
app.use((req, res, next) => {
  req.requestId = req.headers['x-request-id'] ?? crypto.randomUUID();
  next();
});
```

---

## Memory Leak Patterns

### 1. Global Array/Map Không Giới Hạn

```typescript
// ❌ LEAK — cache không có eviction
const requestLog: object[] = [];

app.use((req, res, next) => {
  requestLog.push({ url: req.url, body: req.body, time: Date.now() });
  next();
});

// ✅ Bounded cache với LRU
import LRU from 'lru-cache';
const cache = new LRU({ max: 1000, ttl: 1000 * 60 * 5 });
```

### 2. Closure Giữ Reference

```typescript
// ❌ LEAK — closure giữ large object
function createHandler() {
  const largeData = new Array(1_000_000).fill('x');

  return function handler(req: Request, res: Response) {
    // largeData vẫn referenced dù không dùng
    res.json({ ok: true });
  };
}

// ✅ Chỉ capture cần thiết
function createHandler() {
  return function handler(req: Request, res: Response) {
    res.json({ ok: true });
  };
}
```

### 3. Event Listener Không Remove

```typescript
// ❌ LEAK — listener tích lũy
class UserService {
  constructor(private emitter: EventEmitter) {
    this.emitter.on('user:created', this.onUserCreated);
  }
  onUserCreated = (user: User) => { /* ... */ };
  // Nếu UserService bị destroy mà không remove listener → leak
}

// ✅ Cleanup
destroy() {
  this.emitter.off('user:created', this.onUserCreated);
}
```

### 4. Timer/Interval Không Clear

```typescript
// ❌ LEAK
setInterval(() => {
  cache.set(`key:${Date.now()}`, fetchData());
}, 1000);

// ✅ Có cleanup và bounded keys
const interval = setInterval(updateCache, 60_000);
process.on('SIGTERM', () => clearInterval(interval));
```

### 5. Detached DOM-like Patterns (Server)

```typescript
// ❌ Map giữ user sessions mãi mãi
const sessions = new Map<string, Session>();

// ✅ TTL eviction
setInterval(() => {
  const now = Date.now();
  for (const [id, session] of sessions) {
    if (now - session.lastAccess > SESSION_TTL) {
      sessions.delete(id);
    }
  }
}, 60_000);
```

---

## Heap Snapshots

### Chụp Snapshot Từ Code

```typescript
import v8 from 'node:v8';
import fs from 'node:fs';
import path from 'node:path';

function takeHeapSnapshot(label: string) {
  const snapshotPath = path.join('/tmp', `heap-${label}-${Date.now()}.heapsnapshot`);
  const snapshot = v8.writeHeapSnapshot(snapshotPath);
  console.log(`Heap snapshot written to: ${snapshot}`);
  return snapshot;
}

// API endpoint — CHỈ enable trong staging/debug, có auth!
app.post('/debug/heap-snapshot', authAdmin, (_req, res) => {
  const path = takeHeapSnapshot('manual');
  res.json({ path });
});
```

### Chụp Qua Inspector

```bash
node --inspect dist/server.js
# Mở chrome://inspect → Memory tab → Take snapshot
```

### So Sánh Snapshots

1. Chụp snapshot **baseline** (app vừa start)
2. Gửi 10,000 requests
3. Chụp snapshot **after load**
4. Trong Chrome DevTools: **Comparison view** — tìm objects tăng nhiều nhất

```
Comparison Results:
  (array)     +15,234 instances  +45.2 MB  ← Nghi ngờ leak
  (string)    +8,102 instances   +12.1 MB
  UserSession +10,000 instances  +30.0 MB  ← Root cause!
```

---

## Chrome DevTools Memory Profiling

### 3 Chế Độ Profiling

| Chế Độ | Mục Đích |
| ------ | -------- |
| **Heap Snapshot** | Point-in-time object graph |
| **Allocation instrumentation** | Track mọi allocation — tìm nơi tạo objects |
| **Allocation sampling** | Sample allocations — overhead thấp hơn |

### Retainers (Giữ Reference)

Trong snapshot, click object → xem **Retainers chain** — ai đang giữ reference khiến GC không thu hồi:

```
UserSession instance
  └── retained by Map (sessions)
        └── retained by global object
              └── retained by Module (server.ts)
```

---

## Production Memory Monitoring

### Kubernetes Memory Limits

```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"   # OOMKill khi vượt — set đủ headroom
```

### Node.js `--max-old-space-size`

```bash
# Giới hạn Old Space heap — tránh chiếm hết RAM host
node --max-old-space-size=384 dist/server.js
```

### Prometheus Metrics

```typescript
import { Gauge } from 'prom-client';

const heapUsed = new Gauge({
  name: 'nodejs_heap_used_bytes',
  help: 'Heap used in bytes',
});

setInterval(() => {
  heapUsed.set(process.memoryUsage().heapUsed);
}, 10_000);
```

### Alert Rules

```yaml
# heap tăng liên tục 30 phút
- alert: NodeJSMemoryLeakSuspected
  expr: increase(nodejs_heap_used_bytes[30m]) > 50 * 1024 * 1024
  for: 10m
```

---

## Best Practices

1. **Bounded caches** — Luôn có `max` size hoặc TTL
2. **Cleanup listeners** — `off()` / `removeListener()` khi destroy
3. **Avoid global state** — Module-level Maps cần eviction policy
4. **Stream large data** — Không load toàn bộ file vào memory
5. **Object pooling** — Cho high-frequency allocations (advanced)
6. **Monitor heap trend** — Không chỉ absolute value
7. **Load test memory** — Leak thường chỉ lộ sau extended run

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Gợi Ý Trả Lời |
| ------- | ------------- |
| Memory leak trong Node.js là gì? | Objects không còn cần nhưng vẫn referenced → GC không thu hồi → heap tăng |
| `rss` vs `heapUsed`? | rss = tổng RAM process; heapUsed = chỉ V8 JS heap |
| Làm sao debug memory leak? | Heap snapshot comparison, tìm retained objects và retainers chain |
| WeakMap dùng khi nào? | Cache metadata cho objects — key bị GC thì entry tự remove |
| Tại sao GC pause ảnh hưởng latency? | Mark-Sweep stop-the-world — block main thread trong lúc GC |
| `--max-old-space-size` vs K8s memory limit? | Node flag giới hạn V8 heap; K8s limit giới hạn toàn process — cần cả hai |

---

**Tiếp theo:** [3-cluster-worker-threads.md](./3-cluster-worker-threads.md) — Multi-process scaling và CPU offloading
