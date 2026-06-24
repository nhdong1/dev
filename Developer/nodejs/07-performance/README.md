# Hiệu Năng Node.js — Tổng Quan

> Phương pháp đo lường, phân tích và tối ưu hiệu năng (performance) cho Backend Node.js trong production — từ Event Loop lag đến horizontal scaling (mở rộng theo chiều ngang) và load testing (kiểm thử tải).

## Mục Lục

1. [Tại Sao Hiệu Năng Quan Trọng](#tại-sao-hiệu-năng-quan-trọng)
2. [Ràng Buộc Kiến Trúc Node.js](#ràng-buộc-kiến-trúc-nodejs)
3. [Performance Workflow — Quy Trình Tối Ưu](#performance-workflow--quy-trình-tối-ưu)
4. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
5. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
6. [Bài Tập Thực Hành](#bài-tập-thực-hành)
7. [Performance Checklist Production](#performance-checklist-production)
8. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Hiệu Năng Quan Trọng

Node.js thường được chọn vì **I/O throughput (thông lượng I/O) cao** và **developer velocity (tốc độ phát triển)** — nhưng production incidents thường đến từ:

| Triệu Chứng | Nguyên Nhân Thường Gặp |
| ----------- | ---------------------- |
| API response chậm đột ngột | Event Loop blocked (bị chặn), N+1 queries, thiếu cache |
| Memory tăng liên tục | Memory leak (rò rỉ bộ nhớ), closure giữ reference |
| CPU 100% trên 1 core | CPU-bound task trên main thread |
| Connection timeout | Pool exhaustion (cạn kiệt connection pool) |
| Crash OOM (Out Of Memory — Hết Bộ Nhớ) | Heap quá lớn, không set memory limits |

**Nguyên tắc cốt lõi:** Đo trước khi tối ưu (measure before optimize). Không đoán — dùng metrics (chỉ số), profiling (phân tích hiệu năng), và load test để xác định bottleneck (điểm nghẽn).

---

## Ràng Buộc Kiến Trúc Node.js

```
┌─────────────────────────────────────────────────────────┐
│                    Node.js Process                       │
│  ┌─────────────────────────────────────────────────┐   │
│  │         Main Thread (Event Loop)                 │   │
│  │  JavaScript execution — SINGLE-THREADED          │   │
│  │  ⚠️ Blocking code → toàn bộ app bị ảnh hưởng    │   │
│  └─────────────────────────────────────────────────┘   │
│         │                    │                          │
│         ▼                    ▼                          │
│  ┌──────────────┐    ┌──────────────────┐              │
│  │  libuv Pool  │    │  Worker Threads  │              │
│  │  (I/O async) │    │  (CPU offload)   │              │
│  └──────────────┘    └──────────────────┘              │
└─────────────────────────────────────────────────────────┘
```

| Đặc Điểm | Ý Nghĩa Thực Tế |
| -------- | --------------- |
| **Single-threaded Event Loop** | Một request chậm do sync code có thể làm chậm tất cả requests khác |
| **Non-blocking I/O** | Phù hợp I/O-bound workloads — DB, HTTP, file system |
| **Không phù hợp CPU-heavy** | Image processing, crypto, ML inference → Worker Threads hoặc service riêng |
| **Horizontal scaling** | Cluster module, PM2, K8s replicas — tận dụng multi-core |

---

## Performance Workflow — Quy Trình Tối Ưu

```
1. DEFINE SLIs/SLOs          → p95 latency < 200ms, error rate < 0.1%
        ↓
2. BASELINE METRICS          → Prometheus, APM, event loop lag
        ↓
3. LOAD TEST                 → k6/Artillery — tìm breaking point
        ↓
4. PROFILE                   → Flame graph, heap snapshot — xác định bottleneck
        ↓
5. FIX & VERIFY              → Deploy, so sánh metrics trước/sau
        ↓
6. REPEAT                    → Performance là continuous process
```

| Giai Đoạn | Công Cụ |
| --------- | ------- |
| Monitoring (giám sát) | `perf_hooks`, clinic.js, Prometheus |
| Profiling (phân tích) | `--inspect`, Chrome DevTools, 0x |
| Load testing | k6, Artillery, autocannon |
| Scaling | Cluster, Worker Threads, Redis cache |

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 8–10 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-event-loop-monitoring.md](./1-event-loop-monitoring.md) | Event loop lag, clinic.js, 0x flame graphs | 1.5 giờ |
| 2 | [2-memory-management.md](./2-memory-management.md) | Heap snapshots, GC, memory leak detection | 1.5 giờ |
| 3 | [3-cluster-worker-threads.md](./3-cluster-worker-threads.md) | Multi-process scaling, CPU offloading | 1.5 giờ |
| 4 | [4-caching-strategies.md](./4-caching-strategies.md) | Cache-aside, write-through, TTL patterns | 1 giờ |
| 5 | [5-connection-pooling.md](./5-connection-pooling.md) | DB pool sizing, pool exhaustion | 1 giờ |
| 6 | [6-load-testing.md](./6-load-testing.md) | k6, Artillery, throughput benchmarks | 1.5 giờ |
| 7 | [7-profiling.md](./7-profiling.md) | `--inspect`, Chrome DevTools, `perf_hooks` | 1 giờ |

---

## Các Tài Liệu Chi Tiết

| File | Chủ Đề Chính |
| ---- | ------------ |
| [1-event-loop-monitoring.md](./1-event-loop-monitoring.md) | `monitorEventLoopDelay`, clinic.js Doctor/Bubbleprof, 0x |
| [2-memory-management.md](./2-memory-management.md) | V8 heap spaces, `--expose-gc`, Chrome Memory tab |
| [3-cluster-worker-threads.md](./3-cluster-worker-threads.md) | `cluster` module, `worker_threads`, `isMainThread` |
| [4-caching-strategies.md](./4-caching-strategies.md) | Cache-aside, stampede prevention, Redis patterns |
| [5-connection-pooling.md](./5-connection-pooling.md) | `pg` pool, Prisma connection limit, sizing formula |
| [6-load-testing.md](./6-load-testing.md) | k6 scenarios, Artillery phases, SLI interpretation |
| [7-profiling.md](./7-profiling.md) | CPU profiling, sampling vs instrumentation |

---

## Bài Tập Thực Hành

### Bài 1: Đo Event Loop Lag

1. Thêm `monitorEventLoopDelay` vào Express app
2. Tạo endpoint chạy `bcrypt.hashSync` (blocking) — quan sát lag tăng
3. Chuyển sang `bcrypt.hash` async hoặc Worker Thread — so sánh lag

### Bài 2: Tìm Memory Leak

1. Tạo endpoint leak array global mỗi request
2. Chụp 2 heap snapshots cách nhau 1000 requests
3. Dùng Chrome DevTools so sánh — tìm object retained

### Bài 3: Cluster Mode

1. Chạy Express app với `cluster` module — 1 worker per CPU core
2. Load test với autocannon — so sánh throughput single vs cluster

### Bài 4: Load Test với k6

1. Viết k6 script cho `GET /api/users`
2. Ramp-up từ 10 → 100 VUs (Virtual Users — Người Dùng Ảo)
3. Ghi nhận p95 latency và error rate tại breaking point

---

## Performance Checklist Production

### Trước Deploy

- [ ] Baseline load test đã chạy và documented
- [ ] Event loop lag monitoring enabled
- [ ] Memory limits set (`--max-old-space-size` hoặc K8s limits)
- [ ] Connection pool sized đúng (không quá `max_connections` DB)
- [ ] Caching strategy cho hot paths
- [ ] Không có sync blocking code trên hot path

### Runtime Monitoring

- [ ] Metrics: RPS, latency percentiles (p50/p95/p99), error rate
- [ ] Event loop delay < 10ms (warning), < 50ms (critical)
- [ ] Heap usage trend — không tăng tuyến tính theo thời gian
- [ ] DB connection pool utilization < 80%
- [ ] Alerts configured cho SLO violations

### Khi Incident

- [ ] Thu thập heap snapshot / CPU profile trước khi restart
- [ ] Kiểm tra recent deploys và config changes
- [ ] Scale horizontally tạm thời nếu CPU-bound
- [ ] Root cause analysis — không chỉ restart

---

## Câu Hỏi Phỏng Vấn Thường Gặp

| Câu Hỏi | Điểm Cần Trả Lời |
| ------- | ---------------- |
| Node.js single-threaded — scale thế nào? | Cluster/PM2 multi-process, horizontal scaling với load balancer |
| Event loop lag là gì? | Độ trễ giữa scheduled callback và thực thi — dấu hiệu blocking code |
| Memory leak debug như thế nào? | Heap snapshot comparison, tìm retained objects, fix reference leaks |
| Cluster vs Worker Threads? | Cluster: multi-process cho I/O scaling. Worker Threads: shared memory CPU tasks |
| Cache-aside pattern? | App đọc cache trước, miss thì đọc DB và populate cache |
| Connection pool sizing? | `(cores * 2) + effective_spindle_count` — điều chỉnh theo workload |
| k6 vs Artillery? | k6: Go-based, scripting JS, metrics mạnh. Artillery: YAML config, Node-native |
| Khi nào dùng profiling vs load test? | Load test tìm capacity. Profiling tìm code path chậm cụ thể |

---

## Liên Kết Chủ Đề Liên Quan

- **Event Loop:** [02-async-programming/1-event-loop.md](../02-async-programming/1-event-loop.md) — Nền tảng hiểu blocking behavior
- **Redis Caching:** [04-data-access/6-redis-caching.md](../04-data-access/6-redis-caching.md) — Implementation chi tiết cache layer
- **Connection Pool:** [04-data-access/1-postgresql-pg.md](../04-data-access/1-postgresql-pg.md) — pg pool configuration
- **PM2 & K8s:** [09-cloud-deployment/](../09-cloud-deployment/) — Production scaling và deployment
- **Streams:** [10-advanced/1-streams-api.md](../10-advanced/1-streams-api.md) — Xử lý file lớn hiệu quả

---

**Tiếp theo:** [1-event-loop-monitoring.md](./1-event-loop-monitoring.md) — Giám sát Event Loop và phát hiện blocking code
