# Load Testing — k6, Artillery và Throughput Benchmarks

> Load testing (kiểm thử tải) xác định **capacity (dung lượng)** và **breaking point (điểm gãy)** của API trước khi users làm điều đó. Không load test = deploy blind vào production.

## Mục Lục

1. [Tại Sao Load Test](#tại-sao-load-test)
2. [Metrics Cần Đo](#metrics-cần-đo)
3. [autocannon — Quick Benchmark](#autocannon--quick-benchmark)
4. [k6 — Scriptable Load Testing](#k6--scriptable-load-testing)
5. [Artillery — YAML-Based Testing](#artillery--yaml-based-testing)
6. [Test Scenarios](#test-scenarios)
7. [CI/CD Integration](#cicd-integration)
8. [Đọc Kết Quả và SLIs](#đọc-kết-quả-và-slis)
9. [Best Practices](#best-practices)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Load Test

| Không Load Test | Có Load Test |
| --------------- | ------------ |
| "Chạy OK trên laptop" | Biết chính xác max RPS |
| Black Friday crash | Capacity planned |
| Guessing pool size | Data-driven sizing |
| p99 latency surprise | Latency SLO validated |

**Load test trả lời:**
- API chịu được bao nhiêu requests/giây?
- p95/p99 latency ở load target là bao nhiêu?
- Breaking point ở đâu — CPU, memory, DB, hay event loop?
- Auto-scaling trigger đúng chưa?

---

## Metrics Cần Đo

| Metric | Mô Tả | Target Ví Dụ |
| ------ | ----- | ------------ |
| **RPS (Requests Per Second)** | Throughput (thông lượng) | > 1000 RPS |
| **Latency p50** | 50% requests nhanh hơn | < 50ms |
| **Latency p95** | 95% requests nhanh hơn | < 200ms |
| **Latency p99** | 99% requests nhanh hơn | < 500ms |
| **Error rate** | % failed requests | < 0.1% |
| **Concurrent users** | Virtual users đồng thời | Per business requirement |

```
Latency Distribution:
  p50:  ████████████░░░░░░░░  45ms
  p95:  ████████████████████░  180ms
  p99:  █████████████████████  420ms
  max:  ██████████████████████  2100ms  ← outliers
```

---

## autocannon — Quick Benchmark

[autocannon](https://github.com/mcollina/autocannon) — HTTP benchmark nhanh, zero config.

```bash
npm install -g autocannon

# Basic benchmark
autocannon -c 50 -d 30 http://localhost:3000/api/users

# POST với body
autocannon -c 50 -d 30 -m POST \
  -H "Content-Type: application/json" \
  -b '{"email":"test@example.com"}' \
  http://localhost:3000/api/users

# Pipeline mode — keep connections open
autocannon -c 100 -p 10 -d 60 http://localhost:3000/api/health
```

### Đọc Output

```
┌─────────┬──────┬──────┬───────┬──────┬─────────┬─────────┬─────────┐
│ Stat    │ 2.5% │ 50%  │ 97.5% │ 99%  │ Avg     │ Stdev   │ Max     │
├─────────┼──────┼──────┼───────┼──────┼─────────┼─────────┼─────────┤
│ Latency │ 5 ms │ 12 ms│ 45 ms │ 89 ms│ 15.2 ms │ 12.1 ms │ 234 ms  │
└─────────┴──────┴──────┴───────┴──────┴─────────┴─────────┴─────────┘
Req/Sec: 8,234.56  ← Throughput
```

| Flag | Ý Nghĩa |
| ---- | ------- |
| `-c 50` | 50 concurrent connections |
| `-d 30` | Duration 30 seconds |
| `-p 10` | 10 pipelined requests per connection |

---

## k6 — Scriptable Load Testing

[k6](https://k6.io/) — load testing với JavaScript, metrics mạnh, Grafana integration.

### Cài Đặt

```bash
# Windows (chocolatey) / macOS (brew) / Docker
choco install k6
# hoặc
docker pull grafana/k6
```

### Basic Script

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const apiDuration = new Trend('api_duration');

export const options = {
  stages: [
    { duration: '30s', target: 20 },   // Ramp-up
    { duration: '1m', target: 50 },    // Sustain
    { duration: '30s', target: 100 },  // Peak
    { duration: '30s', target: 0 },    // Ramp-down
  ],
  thresholds: {
    http_req_duration: ['p(95)<200', 'p(99)<500'],
    errors: ['rate<0.01'],
    http_req_failed: ['rate<0.01'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';

export default function () {
  const res = http.get(`${BASE_URL}/api/users`, {
    headers: { Authorization: `Bearer ${__ENV.API_TOKEN}` },
  });

  const success = check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 200ms': (r) => r.timings.duration < 200,
    'has users array': (r) => JSON.parse(r.body).data !== undefined,
  });

  errorRate.add(!success);
  apiDuration.add(res.timings.duration);

  sleep(1); // Think time — realistic user behavior
}
```

### Chạy k6

```bash
k6 run load-test.js
k6 run --vus 100 --duration 5m load-test.js
k6 run -e BASE_URL=https://staging.api.com -e API_TOKEN=xxx load-test.js

# Output to Grafana Cloud
k6 run --out cloud load-test.js
```

### k6 Scenarios — Mixed Workloads

```javascript
export const options = {
  scenarios: {
    read_heavy: {
      executor: 'constant-vus',
      vus: 80,
      duration: '5m',
      exec: 'readTest',
    },
    write_light: {
      executor: 'constant-vus',
      vus: 20,
      duration: '5m',
      exec: 'writeTest',
    },
    spike: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '10s', target: 500 },
        { duration: '1m', target: 500 },
        { duration: '10s', target: 0 },
      ],
      exec: 'readTest',
    },
  },
};

export function readTest() {
  http.get(`${BASE_URL}/api/products`);
}

export function writeTest() {
  http.post(`${BASE_URL}/api/orders`, JSON.stringify({ productId: '123', qty: 1 }), {
    headers: { 'Content-Type': 'application/json' },
  });
}
```

---

## Artillery — YAML-Based Testing

[Artillery](https://www.artillery.io/) — declarative YAML config, Node.js native.

```bash
npm install -g artillery
```

### artillery.yml

```yaml
config:
  target: "http://localhost:3000"
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Warm up"
    - duration: 120
      arrivalRate: 50
      name: "Sustained load"
    - duration: 60
      arrivalRate: 100
      name: "Peak load"
  defaults:
    headers:
      Content-Type: "application/json"
  ensure:
    maxErrorRate: 1
    p95: 200
    p99: 500

scenarios:
  - name: "Get users"
    weight: 70
    flow:
      - get:
          url: "/api/users"
          capture:
            - json: "$.data[0].id"
              as: "userId"
      - think: 2

  - name: "Get user detail"
    weight: 30
    flow:
      - get:
          url: "/api/users/{{ userId }}"
```

```bash
artillery run artillery.yml
artillery run --output report.json artillery.yml
artillery report report.json  # HTML report
```

---

## Test Scenarios

### 1. Smoke Test — Kiểm Tra Cơ Bản

```javascript
// k6 smoke test — ít VUs, verify app hoạt động
export const options = {
  vus: 5,
  duration: '30s',
  thresholds: { http_req_failed: ['rate<0'] },
};
```

### 2. Load Test — Normal Traffic

Simulate expected production traffic + 20% headroom.

### 3. Stress Test — Tìm Breaking Point

```javascript
export const options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 200 },
    { duration: '5m', target: 400 },
    { duration: '5m', target: 600 },
    { duration: '2m', target: 0 },
  ],
};
```

### 4. Spike Test — Traffic Đột Biến

Đột ngột 10x traffic — test auto-scaling và circuit breakers.

### 5. Soak Test — Endurance

Chạy 4–24 giờ ở moderate load — phát hiện memory leaks.

```javascript
export const options = {
  stages: [{ duration: '4h', target: 50 }],
};
```

---

## CI/CD Integration

### GitHub Actions với k6

```yaml
name: Load Test

on:
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'  # Weekly Monday 2AM

jobs:
  load-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - name: Start API
        run: |
          npm ci && npm run build
          npm start &
          sleep 10

      - name: Run k6
        uses: grafana/k6-action@v0.3.1
        with:
          filename: tests/load/smoke.js
          flags: -e BASE_URL=http://localhost:3000
```

### Performance Regression Gate

```javascript
// Fail CI nếu p95 regression > 20% so với baseline
export const options = {
  thresholds: {
    http_req_duration: ['p(95)<200'],
  },
};
```

---

## Đọc Kết Quả và SLIs

### k6 Summary Output

```
✓ status is 200
✓ response time < 200ms

checks.........................: 99.82% ✓ 29845    ✗ 54
data_received..................: 45 MB  150 kB/s
data_sent......................: 12 MB  40 kB/s
http_req_duration..............: avg=45ms  min=3ms  med=28ms  max=1.2s  p(90)=89ms  p(95)=156ms
http_req_failed................: 0.18%  ✓ 54       ✗ 29845
http_reqs......................: 29900  99.6/s
iteration_duration.............: avg=1.04s min=1s   med=1.02s max=2.1s  p(90)=1.08s p(95)=1.15s
vus............................: 50     min=0      max=50
```

### Phân Tích Bottleneck

| Symptom Trong Load Test | Likely Bottleneck |
| ----------------------- | ----------------- |
| Latency tăng linear với VUs | DB connections, thread pool |
| Latency spike đột ngột | GC pause, event loop block |
| Error rate tăng ở high VUs | Pool exhaustion, OOM |
| Throughput plateau | CPU single-core limit → cluster |
| p99 >> p95 | Outliers — slow queries, cold cache |

---

## Best Practices

1. **Test against staging** — Production-like environment, không test prod trực tiếp
2. **Isolate variables** — Một thay đổi mỗi test run
3. **Warm up** — JVM/Node/V8 warmup trước khi đo chính thức
4. **Realistic data** — Test với data volume production-like
5. **Think time** — `sleep()` simulate user behavior
6. **Baseline first** — Lưu kết quả trước optimization để so sánh
7. **Monitor server** — Chạy load test đồng thời với server metrics (CPU, memory, DB)
8. **Không test từ cùng machine** — Network loopback không realistic

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Gợi Ý Trả Lời |
| ------- | ------------- |
| Load test vs stress test? | Load: normal/expected traffic. Stress: beyond capacity để tìm breaking point |
| p95 vs average latency? | Average bị skew bởi outliers; p95 phản ánh UX thực tế hơn |
| k6 vs Artillery? | k6: JS scripts, Grafana ecosystem. Artillery: YAML, Node-native, dễ cho non-devs |
| Soak test là gì? | Extended duration test — phát hiện memory leaks, resource exhaustion |
| Load test trong CI? | Smoke test mỗi PR; full load test weekly hoặc pre-release |
| Virtual Users vs Connections? | VUs simulate users với think time; connections là concurrent TCP connections |

---

**Tiếp theo:** [7-profiling.md](./7-profiling.md) — `--inspect`, Chrome DevTools và `perf_hooks`
