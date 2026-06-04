# 07 — Hiệu Năng (Performance) — Tổng Quan

> Hiệu năng không phải là tính năng — đó là yêu cầu cơ bản của mọi hệ thống production.
> Module này bao quát toàn bộ vòng đời tối ưu: từ caching, connection pool, JVM tuning,
> query optimization, profiling cho đến load testing.

---

## 📋 Mục Tiêu Module

Sau khi hoàn thành module này, bạn có thể:

- [ ] Thiết kế và triển khai **Caching Strategy** (Chiến Lược Bộ Nhớ Đệm) đa tầng với Spring Cache + Redis
- [ ] Cấu hình **HikariCP** (Connection Pool — Bể Kết Nối) tối ưu cho production
- [ ] Tune (Tinh Chỉnh) **JVM** — chọn đúng GC (Garbage Collector — Bộ Thu Gom Rác), cấu hình heap
- [ ] Phân tích và tối ưu **slow queries** (truy vấn chậm), sử dụng index đúng cách
- [ ] Profiling (Phân Tích Hiệu Năng) ứng dụng với **async-profiler** và **JFR** (Java Flight Recorder)
- [ ] Thiết kế và chạy **Load Test** (Kiểm Thử Tải) với Gatling / k6

---

## 🗂️ Nội Dung Module

| File | Chủ Đề | Độ Khó | Ưu Tiên |
| ---- | ------- | ------- | ------- |
| [1-caching-strategies.md](1-caching-strategies.md) | Spring Cache, Redis, Cache Eviction | ⭐⭐ | Cao |
| [2-connection-pooling.md](2-connection-pooling.md) | HikariCP, pool sizing, leak detection | ⭐⭐ | Cao |
| [3-jvm-tuning.md](3-jvm-tuning.md) | Heap, GC, G1GC vs ZGC, tuning flags | ⭐⭐⭐ | Cao |
| [4-query-optimization.md](4-query-optimization.md) | Slow queries, indexes, batch ops | ⭐⭐ | Cao |
| [5-profiling.md](5-profiling.md) | async-profiler, JFR, flame graphs | ⭐⭐⭐ | Trung bình |
| [6-load-testing.md](6-load-testing.md) | Gatling, k6, load vs stress vs spike | ⭐⭐ | Trung bình |

---

## 🧠 Tại Sao Hiệu Năng Quan Trọng?

```
Vấn đề hiệu năng điển hình trong Spring Boot production:

                    [Người Dùng]
                         ↓
               Response Time: 2–5 giây ← KHÔNG THỂ CHẤP NHẬN
                         ↓
        ┌────────────────────────────────────┐
        │     Nguyên nhân phổ biến:          │
        │  1. N+1 queries → DB quá tải       │
        │  2. Connection pool exhausted       │
        │  3. GC pauses quá dài              │
        │  4. Thiếu caching → tính toán lại  │
        │  5. Slow queries không có index     │
        └────────────────────────────────────┘
```

### Chi Phí Thực Tế Của Performance Bug

```
Latency (Độ Trễ) tăng gấp đôi = Revenue (Doanh Thu) giảm ~7% (Amazon research)

1ms latency increase → $6.7M annual revenue loss (Amazon)
100ms latency increase → Sales drop 1% (Google)
3 second load time → 40% users abandon (Google)
```

---

## 🏗️ Kiến Trúc Tối Ưu Hiệu Năng

```
┌─────────────────────────────────────────────────────────────┐
│                   Spring Boot Application                    │
│                                                             │
│  ┌─────────────┐    ┌──────────────┐    ┌───────────────┐  │
│  │  Web Layer  │───▶│ Service Layer│───▶│  Data Layer   │  │
│  │             │    │              │    │               │  │
│  │ - Rate limit│    │ - @Cacheable │    │ - HikariCP    │  │
│  │ - Compress  │    │ - @Async     │    │ - Query opt.  │  │
│  │ - ETags     │    │ - Pagination │    │ - Batch ops   │  │
│  └─────────────┘    └──────────────┘    └───────────────┘  │
│           │                 │                   │           │
│           ▼                 ▼                   ▼           │
│     [CDN Cache]      [Redis Cache]         [DB Index]       │
│                                                             │
│  JVM: Heap + GC tuning     Monitoring: Micrometer + JFR    │
└─────────────────────────────────────────────────────────────┘
```

---

## 📊 Bốn Tầng Tối Ưu Hiệu Năng

### Tầng 1 — Application Layer (Tầng Ứng Dụng)

| Kỹ Thuật | Công Cụ | Độ Ưu Tiên |
| -------- | ------- | ---------- |
| Caching (Bộ Nhớ Đệm) | Spring Cache + Redis | ⭐⭐⭐ |
| Async Processing (Xử Lý Bất Đồng Bộ) | @Async + CompletableFuture | ⭐⭐ |
| Pagination (Phân Trang) | Pageable, Slice | ⭐⭐ |
| Response Compression (Nén Phản Hồi) | GZIP via server.compression | ⭐⭐ |

### Tầng 2 — Database Layer (Tầng Cơ Sở Dữ Liệu)

| Kỹ Thuật | Công Cụ | Độ Ưu Tiên |
| -------- | ------- | ---------- |
| Connection Pooling (Bể Kết Nối) | HikariCP | ⭐⭐⭐ |
| Query Optimization (Tối Ưu Truy Vấn) | EXPLAIN ANALYZE, indexes | ⭐⭐⭐ |
| Batch Operations (Thao Tác Hàng Loạt) | JDBC batch, JPA bulk | ⭐⭐ |
| Read Replicas (Bản Sao Chỉ Đọc) | @Transactional(readOnly=true) | ⭐⭐ |

### Tầng 3 — JVM Layer

| Kỹ Thuật | Công Cụ | Độ Ưu Tiên |
| -------- | ------- | ---------- |
| Heap Sizing (Cấu Hình Heap) | -Xms, -Xmx | ⭐⭐⭐ |
| GC Selection (Chọn Bộ Thu Gom Rác) | G1GC, ZGC, Shenandoah | ⭐⭐⭐ |
| GC Tuning (Tinh Chỉnh GC) | Pause target, region size | ⭐⭐ |
| JIT Compilation (Biên Dịch Tức Thì) | C1/C2 compilers | ⭐ |

### Tầng 4 — Infrastructure Layer (Tầng Hạ Tầng)

| Kỹ Thuật | Công Cụ | Độ Ưu Tiên |
| -------- | ------- | ---------- |
| Load Balancing (Cân Bằng Tải) | K8s Service, NGINX | ⭐⭐⭐ |
| Auto Scaling (Tự Động Mở Rộng) | HPA (Horizontal Pod Autoscaler) | ⭐⭐ |
| CDN Caching (Bộ Nhớ Đệm CDN) | CloudFront, Cloudflare | ⭐⭐ |
| Network Optimization (Tối Ưu Mạng) | HTTP/2, keep-alive | ⭐⭐ |

---

## 🔄 Quy Trình Tối Ưu Hiệu Năng

```
1. MEASURE (Đo Lường)
   └─ Baseline metrics: latency, throughput, error rate, CPU, memory
   └─ Công cụ: Spring Boot Actuator + Micrometer + Prometheus

2. PROFILE (Phân Tích)
   └─ Tìm bottleneck (điểm thắt cổ chai) thực sự
   └─ Công cụ: async-profiler, JFR, Hibernate Statistics

3. OPTIMIZE (Tối Ưu)
   └─ Sửa đúng vấn đề — không đoán mò
   └─ Một thay đổi một lần để đo impact

4. VALIDATE (Xác Nhận)
   └─ So sánh metrics trước/sau
   └─ Load test để verify improvement

5. MONITOR (Theo Dõi)
   └─ Alert khi metrics vượt threshold
   └─ Công cụ: Grafana dashboards, PagerDuty
```

> **Quy tắc vàng:** Không bao giờ tối ưu trước khi đo lường.
> "Premature optimization is the root of all evil" — Donald Knuth

---

## ⚡ Checklist Hiệu Năng Production

### Caching
- [ ] Đã bật Spring Cache cho các methods tốn tài nguyên
- [ ] TTL (Time To Live — Thời Gian Sống) phù hợp với business requirements
- [ ] Cache eviction (Thu Hồi Bộ Nhớ Đệm) được xử lý khi data thay đổi
- [ ] Redis connection pool được cấu hình đúng

### Database
- [ ] HikariCP `maximumPoolSize` được tính toán dựa trên tải thực tế
- [ ] Tất cả foreign keys đều có index
- [ ] Không có N+1 query trong production code
- [ ] Slow query log (Log Truy Vấn Chậm) được bật và theo dõi
- [ ] Batch operations thay vì single-row inserts

### JVM
- [ ] Heap size phù hợp với container memory limits
- [ ] GC được chọn và cấu hình phù hợp với workload
- [ ] GC pause time (Thời Gian Dừng GC) < 200ms ở P99
- [ ] JVM flags được document và version controlled

### Load Testing
- [ ] Load test đã được chạy trước mỗi lần release production
- [ ] SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ) định nghĩa rõ
- [ ] Stress test để biết điểm giới hạn của hệ thống

---

## 🎯 SLA (Service Level Agreement) Tiêu Chuẩn

```
Web API điển hình:
  P50 (median): < 50ms
  P95: < 200ms
  P99: < 500ms
  P99.9: < 1s

Throughput (Thông Lượng):
  Minimum: 100 RPS (Requests Per Second — Yêu Cầu Mỗi Giây)
  Target: 500–1000 RPS
  Peak: 2000+ RPS với horizontal scaling (Mở Rộng Theo Chiều Ngang)

Availability (Khả Năng Sẵn Sàng):
  99.9% = 8.7 hours downtime/year
  99.95% = 4.4 hours downtime/year
  99.99% = 52.5 minutes downtime/year ← Production target
```

---

## 🔗 Điều Hướng

| Chủ Đề Trước | Module Này | Chủ Đề Tiếp |
| ------------ | ---------- | ------------ |
| [06-testing](../06-testing/README.md) | **07-performance** | [08-architecture](../08-architecture/README.md) |

### Trong Module Này

```
Bắt đầu từ:
1. 1-caching-strategies.md   ← Tác động cao nhất, dễ implement nhất
2. 2-connection-pooling.md   ← Hay bị bỏ qua, gây production outage
3. 3-jvm-tuning.md           ← Cần thiết cho production sizing
4. 4-query-optimization.md   ← Làm kết hợp với 03-data-access
5. 5-profiling.md            ← Khi cần debug performance issue
6. 6-load-testing.md         ← Trước mỗi lần release
```

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
