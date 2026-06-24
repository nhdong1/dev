# Event Loop Monitoring — clinic.js, 0x Flame Graphs và Event Loop Lag

> Event Loop (Vòng Lặp Sự Kiện) là trái tim của Node.js — khi nó bị block (chặn), toàn bộ ứng dụng ngừng phản hồi. Giám sát event loop lag (độ trễ vòng lặp) là bước đầu tiên khi troubleshoot API chậm.

## Mục Lục

1. [Event Loop Lag Là Gì](#event-loop-lag-là-gì)
2. [monitorEventLoopDelay — Built-in API](#monitoreventloopdelay--built-in-api)
3. [Blocking Code — Nguyên Nhân Phổ Biến](#blocking-code--nguyên-nhân-phổ-biến)
4. [clinic.js — Performance Suite](#clinicjs--performance-suite)
5. [0x — Flame Graph Generator](#0x--flame-graph-generator)
6. [Metrics và Alerting](#metrics-và-alerting)
7. [Best Practices](#best-practices)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Event Loop Lag Là Gì

**Event loop lag** đo khoảng thời gian giữa khi một timer/callback được lên lịch và khi nó thực sự chạy. Lag cao = main thread đang bận xử lý việc khác (thường là **blocking synchronous code** — mã đồng bộ chặn luồng).

```
Timeline bình thường:
  Timer scheduled ──[2ms]──► Callback executed   ✅ Lag thấp

Timeline khi blocked:
  Timer scheduled ──[500ms]──► Callback executed   ❌ Lag cao — users timeout
```

| Mức Lag | Ý Nghĩa |
| ------- | ------- |
| < 10ms | Healthy — bình thường |
| 10–50ms | Warning — có thể có blocking spikes |
| > 50ms | Critical — UX degraded, timeouts likely |
| > 100ms | Severe — health checks fail, cascading failures |

---

## monitorEventLoopDelay — Built-in API

Node.js 16+ cung cấp `perf_hooks.monitorEventLoopDelay` — zero dependency monitoring.

```typescript
import { monitorEventLoopDelay, performance } from 'node:perf_hooks';

const histogram = monitorEventLoopDelay({ resolution: 10 });
histogram.enable();

// Đọc metrics định kỳ (ví dụ mỗi 10 giây)
setInterval(() => {
  const lag = {
    min: histogram.min / 1e6,      // nanoseconds → milliseconds
    max: histogram.max / 1e6,
    mean: histogram.mean / 1e6,
    p50: histogram.percentile(50) / 1e6,
    p99: histogram.percentile(99) / 1e6,
  };

  console.log('Event loop lag (ms):', lag);

  // Reset histogram cho window tiếp theo
  histogram.reset();
}, 10_000);
```

### Tích Hợp Prometheus

```typescript
import { Registry, Gauge } from 'prom-client';

const register = new Registry();
const eventLoopLag = new Gauge({
  name: 'nodejs_event_loop_lag_p99_ms',
  help: 'Event loop lag p99 in milliseconds',
  registers: [register],
});

setInterval(() => {
  eventLoopLag.set(histogram.percentile(99) / 1e6);
  histogram.reset();
}, 10_000);

// Expose /metrics endpoint
app.get('/metrics', async (_req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

### `performance.eventLoopUtilization()` (Node.js 16+)

Đo **tỷ lệ thời gian Event Loop active** so với idle:

```typescript
import { performance, PerformanceObserver } from 'node:perf_hooks';

const elu = performance.eventLoopUtilization();

setInterval(() => {
  const current = performance.eventLoopUtilization(elu);
  // current.utilization: 0.0 → 1.0 (100% busy)
  console.log(`ELU: ${(current.utilization * 100).toFixed(1)}%`);
}, 5000);
```

---

## Blocking Code — Nguyên Nhân Phổ Biến

### Ví Dụ Blocking (Tránh Trên Hot Path)

```typescript
// ❌ BLOCKING — bcrypt sync trên mỗi login
import bcrypt from 'bcrypt';
app.post('/login', (req, res) => {
  const hash = bcrypt.hashSync(req.body.password, 12); // Block main thread!
  // ...
});

// ✅ NON-BLOCKING — async version
app.post('/login', async (req, res) => {
  const hash = await bcrypt.hash(req.body.password, 12);
  // ...
});
```

| Blocking Pattern | Giải Pháp |
| ---------------- | --------- |
| `*Sync()` APIs (`readFileSync`, `hashSync`) | Dùng async variant hoặc Worker Threads |
| `JSON.parse` object cực lớn | Stream parsing, chunk processing |
| Regex catastrophic backtracking | Test regex với [regexploit](https://github.com/davisjam/regexploit) |
| Tight loops không yield | `setImmediate` chunking hoặc Worker Thread |
| `while(true)` busy wait | Không bao giờ — dùng queue/worker |

### Chunking CPU Work

```typescript
function processLargeArray(items: number[], callback: () => void) {
  let index = 0;
  const CHUNK_SIZE = 1000;

  function processChunk() {
    const end = Math.min(index + CHUNK_SIZE, items.length);
    for (; index < end; index++) {
      // CPU work per item
      Math.sqrt(items[index]);
    }
    if (index < items.length) {
      setImmediate(processChunk); // Yield to event loop
    } else {
      callback();
    }
  }
  processChunk();
}
```

---

## clinic.js — Performance Suite

[clinic.js](https://clinicjs.org/) là bộ công cụ profiling từ NearForm — 3 tools chính:

| Tool | Mục Đích |
| ---- | -------- |
| **Doctor** | Tổng quan health — event loop, memory, I/O issues |
| **Bubbleprof** | Async delay visualization — tìm async bottlenecks |
| **Flame** | CPU flame graph — tìm hot functions |

### Cài Đặt và Sử Dụng

```bash
npm install -g clinic

# Doctor — chẩn đoán tổng quan
clinic doctor -- node dist/server.js

# Bubbleprof — async delays
clinic bubbleprof -- node dist/server.js

# Flame — CPU profiling
clinic flame -- node dist/server.js
```

Sau khi chạy và reproduce load (dùng `autocannon` hoặc `wrk`), nhấn `Ctrl+C` — clinic mở HTML report trong browser.

### Đọc Doctor Report

```
Doctor phát hiện:
├── "Event Loop Delay" spike     → Blocking sync code
├── "GC Activity" cao            → Memory pressure, object churn
├── "Active Handles" tăng        → Connection leak, timer leak
└── "CPU Usage" uneven           → Cần cluster/worker threads
```

---

## 0x — Flame Graph Generator

[0x](https://github.com/davidmarkgardner/0x) tạo **flame graph (biểu đồ ngọn lửa)** — visualization stack traces theo CPU time.

```bash
npm install -g 0x

# Profile app trong 30 giây
0x -- node dist/server.js

# Trong terminal khác — generate load
npx autocannon -c 50 -d 30 http://localhost:3000/api/users
```

### Đọc Flame Graph

```
┌────────────────────────────────────────┐  ← Top = functions tiêu tốn CPU nhất
│████████ bcrypt.hashSync ████████████████│  ← Rộng = nhiều CPU time
├────────────────────────────────────────┤
│██████ Express middleware ██████████████│
├────────────────────────────────────────┤
│████████████ libuv █████████████████████│  ← Bottom = root callers
└────────────────────────────────────────┘
```

- **Plateau rộng ở top** = function cần optimize hoặc offload
- **Tower cao** = deep call stack — có thể refactor
- So sánh flame graph trước/sau optimization để verify improvement

---

## Metrics và Alerting

### SLIs (Service Level Indicators — Chỉ Số Mức Dịch Vụ) Cho Event Loop

| Metric | Alert Threshold |
| ------ | --------------- |
| `event_loop_lag_p99_ms` | > 50ms for 5 minutes |
| `event_loop_utilization` | > 0.9 for 5 minutes |
| `nodejs_active_handles` | Tăng liên tục (leak) |

### Correlation với Request Latency

Event loop lag thường **correlate (tương quan)** với p99 latency spike:

```
Event loop lag spike at 14:32
    ↓
p99 latency spike at 14:32
    ↓
Root cause: deploy mới có JSON.parse(largePayload) sync
```

---

## Best Practices

1. **Monitor mặc định** — Thêm event loop metrics vào mọi production service
2. **Không dùng sync APIs** — Audit codebase với `grep -r 'Sync(' src/`
3. **Load test sau mỗi deploy** — Catch regression sớm
4. **Set timeouts** — HTTP client, DB query timeouts prevent hung requests
5. **Graceful degradation** — Circuit breaker khi downstream chậm
6. **Separate concerns** — CPU-heavy work → Worker Threads hoặc microservice riêng

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Gợi Ý Trả Lời |
| ------- | ------------- |
| Event loop lag khác gì response latency? | Lag đo internal scheduling delay; latency đo end-to-end request time — lag cao thường gây latency cao |
| Làm sao phát hiện blocking code trong production? | `monitorEventLoopDelay`, clinic.js Doctor, flame graph |
| `setImmediate` vs `setTimeout(0)` để yield? | `setImmediate` chạy sau I/O phase; `setTimeout` qua timers phase — `setImmediate` phù hợp hơn để yield trong I/O callbacks |
| Tại sao `bcrypt.hashSync` nguy hiểm? | Bcrypt intentionally slow — sync version block main thread hàng trăm ms mỗi call |
| clinic.js Doctor vs Flame? | Doctor: health overview. Flame: deep CPU profiling với flame graph |

---

**Tiếp theo:** [2-memory-management.md](./2-memory-management.md) — Heap snapshots, Garbage Collection và memory leak detection
