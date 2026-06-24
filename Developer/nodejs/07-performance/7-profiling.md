# Profiling — --inspect, Chrome DevTools và perf_hooks

> Profiling (phân tích hiệu năng) xác định **chính xác** function nào, line nào tiêu tốn CPU hoặc memory — khác với load testing cho biết "bao nhiêu" nhưng không cho biết "tại sao". Đây là kỹ năng debug production performance issues.

## Mục Lục

1. [Profiling vs Monitoring vs Load Testing](#profiling-vs-monitoring-vs-load-testing)
2. [Node.js Inspector (--inspect)](#nodejs-inspector---inspect)
3. [Chrome DevTools CPU Profiling](#chrome-devtools-cpu-profiling)
4. [perf_hooks Module](#perf_hooks-module)
5. [V8 Profiler API](#v8-profiler-api)
6. [Production-Safe Profiling](#production-safe-profiling)
7. [Đọc Flame Graph](#đọc-flame-graph)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Profiling vs Monitoring vs Load Testing

| Hoạt Động | Câu Hỏi Trả Lời | Công Cụ |
| --------- | --------------- | ------- |
| **Monitoring** | Hệ thống healthy không? | Prometheus, event loop lag |
| **Load Testing** | Chịu được bao nhiêu load? | k6, Artillery |
| **Profiling** | Code nào chậm và tại sao? | Chrome DevTools, 0x, clinic |

```
Workflow:
  Load test → latency cao → Profile → tìm hot function → fix → verify
```

---

## Node.js Inspector (--inspect)

Node.js built-in debugger/profiler qua Chrome DevTools Protocol.

```bash
# Start với inspector
node --inspect dist/server.js
# Debugger listening on ws://127.0.0.1:9229/...

# Bind external (Docker/remote) — CẨN THẬN security!
node --inspect=0.0.0.0:9229 dist/server.js

# Break at first line (debug startup)
node --inspect-brk dist/server.js
```

### Kết Nối Chrome DevTools

1. Mở Chrome → `chrome://inspect`
2. Click **"Open dedicated DevTools for Node"**
3. Tab **Profiler** → **Start** → reproduce issue → **Stop**

### VS Code Debugging

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug with Inspector",
      "program": "${workspaceFolder}/dist/server.js",
      "runtimeArgs": ["--inspect"],
      "console": "integratedTerminal"
    }
  ]
}
```

---

## Chrome DevTools CPU Profiling

### Sampling vs Instrumentation

| Loại | Cách Hoạt Động | Overhead | Độ Chính Xác |
| ---- | -------------- | -------- | ------------ |
| **Sampling** | Sample call stack định kỳ | Thấp (~5%) | Tốt cho production-like |
| **Instrumentation** | Hook mọi function call | Cao (có thể 10x slower) | Chi tiết hơn |

**Khuyến nghị:** Bắt đầu với **Sampling** — đủ cho hầu hết cases.

### CPU Profile Workflow

```
1. node --inspect dist/server.js
2. Chrome DevTools → Profiler → Start
3. autocannon -c 50 -d 30 http://localhost:3000/api/slow-endpoint
4. Stop profiling
5. Analyze flame chart / Bottom-Up / Call Tree
```

### Đọc Call Tree

```
Bottom-Up View (sorted by Self Time):
┌────────────────────────────┬───────────┬────────────┐
│ Function                   │ Self Time │ Total Time │
├────────────────────────────┼───────────┼────────────┤
│ bcrypt.hashSync            │ 45.2%     │ 45.2%      │  ← Fix this!
│ JSON.parse                 │ 12.1%     │ 12.1%      │
│ express.middleware         │ 3.2%      │ 68.5%      │
│ pg.query                   │ 8.5%      │ 15.3%      │
└────────────────────────────┴───────────┴────────────┘

Self Time = time trong function đó (không tính children)
Total Time = time bao gồm cả children calls
```

---

## perf_hooks Module

Built-in performance measurement — zero dependency.

### Performance Marks và Measures

```typescript
import { performance, PerformanceObserver } from 'node:perf_hooks';

// Đánh dấu điểm bắt đầu/kết thúc
performance.mark('db-query-start');
const users = await db.user.findMany();
performance.mark('db-query-end');

performance.measure('db-query', 'db-query-start', 'db-query-end');

const measure = performance.getEntriesByName('db-query')[0];
console.log(`DB query took ${measure.duration.toFixed(2)}ms`);
```

### Observer Pattern — Log Tự Động

```typescript
const obs = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 100) {
      console.warn(`Slow operation: ${entry.name} = ${entry.duration.toFixed(2)}ms`);
    }
  }
});
obs.observe({ entryTypes: ['measure'] });
```

### Request-Level Timing Middleware

```typescript
import { performance } from 'node:perf_hooks';

function timingMiddleware(req: Request, res: Response, next: NextFunction) {
  const start = performance.now();

  res.on('finish', () => {
    const duration = performance.now() - start;
    console.log({
      method: req.method,
      url: req.url,
      status: res.statusCode,
      duration: `${duration.toFixed(2)}ms`,
    });

  });

  next();
}
```

### `performance.timerify()` — Wrap Functions

```typescript
import { performance } from 'node:perf_hooks';

const timedFetch = performance.timerify(async (url: string) => {
  const res = await fetch(url);
  return res.json();
});

// Mỗi call tự động tạo performance entry
await timedFetch('https://api.example.com/data');
const entries = performance.getEntriesByType('function');
```

---

## V8 Profiler API

Programmatic CPU profiling — useful cho automated profiling.

```typescript
import { Session } from 'node:inspector';
import fs from 'node:fs';

async function takeCpuProfile(durationMs: number, outputPath: string) {
  const session = new Session();
  session.connect();

  await post(session, 'Profiler.enable');
  await post(session, 'Profiler.start');

  console.log(`Profiling for ${durationMs}ms...`);
  await sleep(durationMs);

  const { profile } = await post(session, 'Profiler.stop');
  fs.writeFileSync(outputPath, JSON.stringify(profile));

  await post(session, 'Profiler.disable');
  session.disconnect();

  console.log(`Profile saved to ${outputPath}`);
  // Mở trong Chrome DevTools: DevTools → Profiler → Load profile
}

function post(session: Session, method: string, params?: object): Promise<any> {
  return new Promise((resolve, reject) => {
    session.post(method, params, (err, result) => {
      if (err) reject(err);
      else resolve(result);
    });
  });
}

function sleep(ms: number) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// Usage — chỉ staging/debug!
// await takeCpuProfile(30_000, '/tmp/cpu-profile.json');
```

### Heap Profiling

```typescript
import v8 from 'node:v8';

// Heap snapshot
const snapshotPath = v8.writeHeapSnapshot('./heap.heapsnapshot');

// Heap statistics
const stats = v8.getHeapStatistics();
console.log({
  totalHeapSize: `${(stats.total_heap_size / 1024 / 1024).toFixed(1)} MB`,
  usedHeapSize: `${(stats.used_heap_size / 1024 / 1024).toFixed(1)} MB`,
  heapSizeLimit: `${(stats.heap_size_limit / 1024 / 1024).toFixed(1)} MB`,
});
```

---

## Production-Safe Profiling

### ⚠️ Rủi Ro Profiling Production

| Risk | Mitigation |
| ---- | ---------- |
| Inspector port exposed | Firewall, chỉ bind localhost |
| Profiling overhead | Sampling only, giới hạn duration |
| Heap snapshot pause | Chạy off-peak, snapshot có thể pause vài giây |
| Sensitive data in snapshot | Restrict access, không commit snapshots |

### Continuous Profiling

Tools như **Pyroscope**, **Parca**, **Google Cloud Profiler** — sample production liên tục với overhead thấp.

```typescript
// @google-cloud/profiler — continuous profiling
import * as profiler from '@google-cloud/profiler';

async function startProfiler() {
  await profiler.start({
    serviceContext: {
      service: 'my-api',
      version: process.env.APP_VERSION || '1.0.0',
    },
  });
}
```

### On-Demand Profiling Signal

```typescript
// Trigger profile via admin endpoint — staging only
let profiling = false;

app.post('/admin/profile/start', authAdmin, async (_req, res) => {
  if (profiling) return res.status(409).json({ error: 'Already profiling' });

  profiling = true;
  const session = new Session();
  session.connect();
  session.post('Profiler.enable');
  session.post('Profiler.start');

  setTimeout(async () => {
    session.post('Profiler.stop', async (err, { profile }) => {
      fs.writeFileSync(`/tmp/profile-${Date.now()}.cpuprofile`, JSON.stringify(profile));
      session.disconnect();
      profiling = false;
    });
  }, 30_000);

  res.json({ message: 'Profiling started for 30s' });
});
```

---

## Đọc Flame Graph

Flame graph visualization — x-axis = sample count (không phải thời gian), y-axis = call stack depth.

```
Wide bar = nhiều CPU time
Tall stack = deep call chain

    ┌─────────────────────────────────────────┐
    │███████████ main ████████████████████████│  ← 100% samples
    ├──────────────────┬──────────────────────┤
    │████ handleRequest│██████ other █████████│
    ├────────┬─────────┤                      │
    │██ hash │██ query │                      │
    └────────┴─────────┴──────────────────────┘

"hash" bar rộng → bcrypt.hashSync là bottleneck
```

### Actionable Insights

| Pattern Trong Flame Graph | Action |
| ------------------------- | ------ |
| Wide `*Sync` function | Chuyển sang async hoặc Worker Thread |
| Wide `JSON.parse` | Stream parsing, smaller payloads |
| Wide `RegExp` | Optimize regex, avoid catastrophic backtracking |
| Wide `gc` / `Garbage Collection` | Reduce object allocation |
| Deep Express middleware stack | Remove unused middleware |

---

## Best Practices

1. **Profile with realistic load** — Idle app profile không representative
2. **Compare before/after** — Lưu profiles để verify optimization
3. **Focus top 3 hot functions** — 80/20 rule applies
4. **Don't optimize prematurely** — Profile first, then fix
5. **Use `perf_hooks` for custom metrics** — Business-critical operations
6. **Sampling for production** — Instrumentation chỉ development
7. **Combine with load test** — Profile during k6 run
8. **Document findings** — Team learns from profiling sessions

### Optimization Workflow

```
1. Baseline load test (k6) → p95 = 350ms (target: 200ms)
2. CPU profile during load → bcrypt.hashSync = 60% CPU
3. Fix: bcrypt.hash (async) + Worker Thread pool
4. Re-profile → bcrypt = 5% CPU
5. Re-load test → p95 = 120ms ✅
6. Document in PR/commit message
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Gợi Ý Trả Lời |
| ------- | ------------- |
| Profiling vs benchmarking? | Profiling: where time spent. Benchmarking: how fast overall |
| `--inspect` vs `--inspect-brk`? | inspect: attach anytime. inspect-brk: pause at start for debug |
| Sampling vs instrumentation profiling? | Sampling: low overhead, statistical. Instrumentation: precise, high overhead |
| `perf_hooks` dùng khi nào? | Custom timing metrics, middleware timing, business operation measurement |
| Profile production an toàn không? | Có với sampling + short duration + continuous profilers (Pyroscope) |
| Flame graph đọc thế nào? | Width = CPU time; tìm widest bars ở top = optimization targets |

---

**Quay lại:** [README.md](./README.md) — Tổng quan chủ đề Hiệu Năng Node.js
