# Load Testing — Kiểm Thử Tải Với Gatling & k6

> Load Testing (Kiểm Thử Tải) là quá trình giả lập nhiều người dùng đồng thời truy cập
> hệ thống để đo khả năng chịu tải, tìm bottleneck (Điểm Thắt Cổ Chai), và xác định giới
> hạn của hệ thống. Không load test = không biết hệ thống sẽ sập khi nào.

---

## 📋 Mục Tiêu

- [ ] Phân biệt **Load**, **Stress**, **Spike**, và **Soak Testing**
- [ ] Viết test script với **Gatling** (JVM-based) và **k6** (JavaScript-based)
- [ ] Thiết kế **test scenarios** (Kịch Bản Kiểm Thử) phản ánh đúng production traffic
- [ ] Phân tích kết quả: **throughput**, **latency percentiles**, **error rate**
- [ ] Tích hợp load test vào **CI/CD pipeline** (Quy Trình Tích Hợp Liên Tục)
- [ ] Xác định và document **system capacity** (Dung Lượng Hệ Thống)

---

## 1. Các Loại Load Test

### Load Test (Kiểm Thử Tải Thông Thường)

```
Mục đích: Kiểm tra hành vi ứng dụng ở tải dự kiến
Cách thực hiện: Tăng dần đến expected load, duy trì trong 30–60 phút
Câu hỏi trả lời: "Hệ thống có hoạt động đúng ở tải bình thường không?"

         Users
           ^
  200 ─── │    ████████████████████
  150 ─── │  ██                    ██
  100 ─── │ █                        █
   50 ─── │█                          █
    0 ─── └─────────────────────────────→ Time
          ramp-up     sustained        ramp-down
          (Tăng Dần)  (Duy Trì)       (Giảm Dần)
```

### Stress Test (Kiểm Thử Áp Lực)

```
Mục đích: Tìm điểm giới hạn (breaking point) của hệ thống
Cách thực hiện: Tăng liên tục vượt quá expected load
Câu hỏi trả lời: "Hệ thống sẽ sập khi nào? Và sập thế nào?"

         Users
           ^
  500 ─── │              ████ ← Hệ thống bắt đầu gặp lỗi
  400 ─── │         ████
  300 ─── │    ████
  200 ─── │ ████
  100 ─── │█
    0 ─── └─────────────────────────────→ Time
```

### Spike Test (Kiểm Thử Đột Biến)

```
Mục đích: Kiểm tra hành vi khi tải tăng đột ngột (flash sale, viral event)
Cách thực hiện: Tăng đột ngột lên rất cao, sau đó giảm xuống
Câu hỏi trả lời: "Hệ thống có recover được sau spike không?"

         Users
           ^
 1000 ─── │    █ ← Spike đột ngột
  500 ─── │    █
  200 ─── │████████ ← Baseline (Tải Cơ Bản)
    0 ─── └─────────────────────────────→ Time
```

### Soak Test / Endurance Test (Kiểm Thử Độ Bền)

```
Mục đích: Phát hiện memory leak và resource exhaustion theo thời gian
Cách thực hiện: Duy trì tải bình thường trong thời gian dài (6–24 giờ)
Câu hỏi trả lời: "Hệ thống có bị degradation (Suy Giảm) theo thời gian không?"

         Memory
           ^
  8GB  ─── │                                    ▁▂▃▄▅ ← Memory leak!
  4GB  ─── │████████████████████████████████████
  2GB  ─── │
    0  ─── └─────────────────────────────────────→ Time (24h)
```

---

## 2. k6 — Modern Load Testing Tool

### Cài Đặt

```bash
# macOS
brew install k6

# Windows (winget)
winget install k6

# Docker
docker run --rm -i grafana/k6 run - <test-script.js

# Linux
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
  --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" \
  | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update && sudo apt-get install k6
```

### Script Cơ Bản

```javascript
// k6-basic.js
import http from 'k6/http';
import { check, sleep } from 'k6';

// Cấu hình test scenario
export const options = {
  stages: [
    { duration: '30s', target: 10 },   // Ramp up (Tăng Dần): 0→10 users trong 30s
    { duration: '1m', target: 50 },    // Tăng lên: 10→50 users trong 1 phút
    { duration: '2m', target: 50 },    // Duy trì: 50 users trong 2 phút
    { duration: '30s', target: 0 },    // Ramp down (Giảm Dần): về 0
  ],
  thresholds: {
    // SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ)
    http_req_duration: ['p(95)<200', 'p(99)<500'],  // P95 < 200ms, P99 < 500ms
    http_req_failed: ['rate<0.01'],                  // Error rate (Tỷ Lệ Lỗi) < 1%
    http_reqs: ['rate>100'],                         // Throughput (Thông Lượng) > 100 RPS
  },
};

// Virtual User (VU — Người Dùng Ảo) behavior
export default function () {
  const response = http.get('http://localhost:8080/api/products');

  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'body is not empty': (r) => r.body.length > 0,
  });

  sleep(1); // Pause 1 giây giữa requests (think time — Thời Gian Suy Nghĩ)
}
```

### Test Scenario Phức Tạp — Mô Phỏng User Journey

```javascript
// k6-user-journey.js
import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics (Số Liệu Tùy Chỉnh)
const loginErrorRate = new Rate('login_error_rate');
const checkoutDuration = new Trend('checkout_duration');

export const options = {
  scenarios: {
    // Scenario 1: Browsing users (Người dùng duyệt)
    browsing: {
      executor: 'ramping-vus',
      stages: [
        { duration: '1m', target: 100 },
        { duration: '3m', target: 100 },
        { duration: '1m', target: 0 },
      ],
    },
    // Scenario 2: Shopping users (Người dùng mua sắm) — ít hơn nhưng tác động lớn hơn
    shopping: {
      executor: 'constant-vus',
      vus: 20,
      duration: '5m',
    },
  },
  thresholds: {
    'http_req_duration{scenario:browsing}': ['p(95)<300'],
    'http_req_duration{scenario:shopping}': ['p(95)<500'],
    'login_error_rate': ['rate<0.05'],
  },
};

const BASE_URL = 'http://localhost:8080';

export default function () {
  // Group 1: Login
  let loginToken;
  group('Authentication (Xác Thực)', () => {
    const loginRes = http.post(`${BASE_URL}/api/auth/login`, JSON.stringify({
      email: 'test@example.com',
      password: 'password123',
    }), {
      headers: { 'Content-Type': 'application/json' },
    });

    loginErrorRate.add(loginRes.status !== 200);

    check(loginRes, {
      'login successful': (r) => r.status === 200,
    });

    loginToken = loginRes.json('accessToken');
  });

  sleep(1);

  // Group 2: Browse products
  group('Product Browsing (Duyệt Sản Phẩm)', () => {
    const headers = { Authorization: `Bearer ${loginToken}` };

    const productsRes = http.get(`${BASE_URL}/api/products?page=0&size=20`, { headers });
    check(productsRes, { 'products loaded': (r) => r.status === 200 });

    sleep(2);

    // View product detail
    const productId = productsRes.json('content.0.id');
    const detailRes = http.get(`${BASE_URL}/api/products/${productId}`, { headers });
    check(detailRes, { 'product detail loaded': (r) => r.status === 200 });
  });

  sleep(1);

  // Group 3: Checkout
  group('Checkout (Thanh Toán)', () => {
    const headers = {
      Authorization: `Bearer ${loginToken}`,
      'Content-Type': 'application/json',
    };

    const startTime = Date.now();

    // Add to cart
    http.post(`${BASE_URL}/api/cart/items`, JSON.stringify({ productId: 1, quantity: 2 }), { headers });

    // Place order
    const orderRes = http.post(`${BASE_URL}/api/orders`, JSON.stringify({ paymentMethod: 'CARD' }), { headers });
    check(orderRes, { 'order created': (r) => r.status === 201 });

    checkoutDuration.add(Date.now() - startTime);
  });

  sleep(2);
}
```

### Chạy k6

```bash
# Chạy test script
k6 run k6-basic.js

# Chạy với nhiều VUs hơn (override script options)
k6 run --vus 50 --duration 30s k6-basic.js

# Output kết quả ra JSON để phân tích
k6 run --out json=results.json k6-basic.js

# Stream metrics sang Grafana
k6 run --out influxdb=http://localhost:8086/k6 k6-basic.js

# Chạy trên K6 Cloud
k6 cloud k6-basic.js
```

### Đọc Kết Quả k6

```
          /\      |‾‾| /‾‾/   /‾‾/
     /\  /  \     |  |/  /   /  /
    /  \/    \    |     (   /   ‾‾\
   /          \   |  |\  \ |  (‾)  |
  / __________ \  |__| \__\ \_____/ .io

  execution: local
     script: k6-basic.js
     output: -

  scenarios: (100.00%) 1 scenario, 50 max VUs, 3m0s max duration:
           * default: Up to 50 looping VUs for 2m30s (gracefulRampDown: 30s)

✓ status is 200        ← Check pass
✓ response time < 500ms

     checks.........................: 100.00% ✓ 15234 ✗ 0
     data_received..................: 12 MB 83 kB/s
     data_sent......................: 1.5 MB 10 kB/s
     http_req_blocked...............: avg=1.2ms  min=1µs   med=3µs    max=156ms  p(90)=5µs    p(95)=9µs
     http_req_connecting............: avg=0.98ms min=0µs   med=0µs    max=155ms  p(90)=0µs    p(95)=0µs
   ✓ http_req_duration..............: avg=45ms   min=12ms  med=38ms   max=512ms  p(90)=78ms   p(95)=102ms
       { expected_response:true }...: avg=45ms   min=12ms  med=38ms   max=512ms  p(90)=78ms   p(95)=102ms
   ✓ http_req_failed................: 0.00%  ✓ 0 ✗ 15234
     http_req_receiving.............: avg=0.2ms  min=13µs  med=0.17ms max=12ms   p(90)=0.42ms p(95)=0.58ms
     http_req_sending...............: avg=0.04ms min=6µs   med=0.03ms max=2.3ms  p(90)=0.07ms p(95)=0.1ms
     http_req_tls_handshaking.......: avg=0s     min=0s    med=0s     max=0s     p(90)=0s     p(95)=0s
     http_req_waiting...............: avg=44ms   min=12ms  med=38ms   max=511ms  p(90)=77ms   p(95)=101ms
     http_reqs......................: 15234  101.56/s   ← Throughput: ~101 RPS
     iteration_duration.............: avg=1.05s  min=1.01s med=1.04s  max=1.63s
     iterations.....................: 15234  101.56/s
     vus............................: 1      min=1       max=50
     vus_max........................: 50     min=50      max=50

Chỉ Số Quan Trọng:
  avg=45ms    : Average latency (Độ Trễ Trung Bình)
  p(90)=78ms  : P90 — 90% requests hoàn thành trong 78ms
  p(95)=102ms : P95 — 95% requests hoàn thành trong 102ms
  101.56/s    : Throughput — 101 requests mỗi giây
  0.00% error : Không có lỗi
```

---

## 3. Gatling — JVM-Based Load Testing

### Cài Đặt Với Maven

```xml
<!-- pom.xml -->
<plugin>
    <groupId>io.gatling</groupId>
    <artifactId>gatling-maven-plugin</artifactId>
    <version>4.9.0</version>
</plugin>
<dependency>
    <groupId>io.gatling.highcharts</groupId>
    <artifactId>gatling-charts-highcharts</artifactId>
    <version>3.11.0</version>
    <scope>test</scope>
</dependency>
```

### Script Gatling Cơ Bản (Scala DSL)

```scala
// src/test/scala/simulations/ProductApiSimulation.scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class ProductApiSimulation extends Simulation {

  // HTTP Protocol (Giao Thức HTTP) configuration
  val httpProtocol = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")

  // Scenario: Duyệt sản phẩm
  val browseProducts = scenario("Browse Products")
    .exec(
      http("Get Product List")
        .get("/api/products?page=0&size=20")
        .check(status.is(200))
        .check(jsonPath("$.content").exists)
    )
    .pause(1.second)
    .exec(
      http("Get Product Detail")
        .get("/api/products/1")
        .check(status.is(200))
        .check(responseTimeInMillis.lt(500))
    )
    .pause(2.seconds)

  // Scenario: Tạo đơn hàng
  val createOrder = scenario("Create Order")
    .exec(
      http("Create Order")
        .post("/api/orders")
        .body(StringBody("""{"productId": 1, "quantity": 2}"""))
        .check(status.is(201))
        .check(jsonPath("$.id").exists)
    )
    .pause(1.second)

  // Load injection profile (Hồ Sơ Tiêm Tải)
  setUp(
    browseProducts.inject(
      rampUsers(50).during(30.seconds),         // Ramp lên 50 users trong 30s
      constantUsersPerSec(20).during(2.minutes) // Duy trì 20 users/giây trong 2 phút
    ),
    createOrder.inject(
      rampUsers(10).during(30.seconds),
      constantUsersPerSec(5).during(2.minutes)
    )
  ).protocols(httpProtocol)
   .assertions(
     global.responseTime.percentile3.lt(500),   // P99 < 500ms
     global.successfulRequests.percent.gt(99),   // Success rate > 99%
     global.requestsPerSec.gt(100)               // Throughput > 100 RPS
   )
}
```

### Chạy Gatling

```bash
# Chạy simulation
mvn gatling:test

# Chạy simulation cụ thể
mvn gatling:test -Dgatling.simulationClass=simulations.ProductApiSimulation

# Kết quả HTML report tại:
# target/gatling/productsapisimulation-<timestamp>/index.html
```

---

## 4. Tích Hợp Vào CI/CD Pipeline

### GitHub Actions

```yaml
# .github/workflows/load-test.yml
name: Load Test

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # Chạy mỗi đêm lúc 2 AM

jobs:
  load-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Start application
        run: |
          docker-compose up -d
          sleep 30  # Chờ app khởi động

      - name: Install k6
        run: |
          sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
            --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
          echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" \
            | sudo tee /etc/apt/sources.list.d/k6.list
          sudo apt-get update && sudo apt-get install k6

      - name: Run load test
        run: k6 run --out json=results.json tests/load/k6-basic.js

      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: k6-results
          path: results.json

      - name: Tear down
        if: always()
        run: docker-compose down
```

---

## 5. Thiết Kế Test Scenario Thực Tế

### Phân Tích Traffic Patterns (Mẫu Lưu Lượng)

```
Trước khi viết test, phân tích production traffic:

1. Peak hours (Giờ Cao Điểm): Khi nào traffic cao nhất?
   → E-commerce: 8–10 PM weekdays, weekend mornings

2. User behavior distribution (Phân Bổ Hành Vi Người Dùng):
   → 60% browse only (chỉ xem)
   → 30% add to cart (thêm vào giỏ)
   → 10% complete purchase (hoàn thành mua)

3. API endpoint distribution:
   → GET /api/products — 50% traffic
   → GET /api/products/:id — 30% traffic
   → POST /api/orders — 5% traffic

→ Test phải phản ánh đúng tỷ lệ này, không chỉ test 1 endpoint
```

### Realistic Think Time (Thời Gian Suy Nghĩ Thực Tế)

```javascript
// ❌ SAI: Không có think time — không thực tế
export default function () {
  http.get('http://localhost:8080/api/products');
  http.get('http://localhost:8080/api/products/1');
  // Gửi requests liên tục không nghỉ → test không phản ánh thực tế
}

// ✅ ĐÚNG: Thêm think time giữa actions
export default function () {
  http.get('http://localhost:8080/api/products');
  sleep(1.5);  // User đọc danh sách sản phẩm

  http.get('http://localhost:8080/api/products/1');
  sleep(3);    // User xem chi tiết sản phẩm

  http.post('http://localhost:8080/api/cart/items', ...);
  sleep(0.5);
}
```

---

## 6. Phân Tích Kết Quả Và Báo Cáo

### Chỉ Số Cần Theo Dõi

```
Latency Percentiles (Phần Vị Độ Trễ):
  P50 (median): Latency của 50% requests thấp hơn
  P90: 90% requests hoàn thành trong thời gian này
  P95: 95% requests — thường dùng trong SLA
  P99: 99% requests — phát hiện worst-case scenarios (Tình Huống Tệ Nhất)
  P99.9: Dùng khi hệ thống cần ultra-high reliability (Độ Tin Cậy Cực Cao)

Throughput (Thông Lượng):
  RPS (Requests Per Second — Yêu Cầu Mỗi Giây): Tổng requests thành công
  TPS (Transactions Per Second — Giao Dịch Mỗi Giây): Hoàn tất user journeys

Error Rate (Tỷ Lệ Lỗi):
  < 0.1%  : Xuất sắc
  0.1–1%  : Chấp nhận được
  > 1%    : Cần investigate ngay
```

### Grafana Dashboard Cho Load Test

```
Metrics cần hiển thị trong dashboard:
  - Request rate (requests/sec)
  - Error rate (%)
  - Response time percentiles (P50, P90, P95, P99)
  - Active VUs
  - CPU/Memory của application server
  - JVM heap usage
  - HikariCP active connections
  - Database query time
```

---

## 7. Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa Load Test, Stress Test, và Spike Test?**

```
Load Test:   Tải bình thường, duy trì trong giờ
             → "Hệ thống có ổn ở expected load?"

Stress Test: Vượt quá expected load, tăng liên tục
             → "Điểm giới hạn là bao nhiêu? Sập thế nào?"

Spike Test:  Tải tăng đột ngột trong thời gian ngắn
             → "Có xử lý được flash sale không?"

Soak Test:   Tải bình thường nhưng trong 6–24 giờ
             → "Có bị memory leak hay resource leak không?"
```

**Q: k6 và Gatling khác nhau thế nào? Khi nào dùng cái nào?**

```
k6:
  + JavaScript — dev team dễ viết
  + CLI-first (Giao Diện Dòng Lệnh Đầu Tiên), CI/CD friendly
  + SaaS cloud option (k6 Cloud)
  + Nhẹ, nhanh khởi động
  - Không generate HTML report sẵn (cần Grafana)

Gatling:
  + Scala DSL — expressive, type-safe
  + HTML report tự động đẹp
  + JVM — tích hợp tốt với Java ecosystem
  + Maven/Gradle plugin
  - Cần biết Scala/JVM

Khuyến nghị:
  - Backend team Java: Gatling
  - Fullstack/DevOps: k6
  - CI/CD integration đơn giản: k6
  - Detailed HTML reports: Gatling
```

**Q: Làm thế nào để xác định `maximumPoolSize` từ kết quả load test?**

```
1. Chạy load test với expected traffic
2. Monitor: hikaricp_connections_pending metric
3. Nếu connections_pending > 0 và tăng → pool exhaustion → tăng pool size
4. Nếu connections_active / connections_max < 50% ổn định → pool size có thể giảm
5. Tăng dần pool size cho đến khi connections_pending = 0 ở peak load
6. Thêm 20% buffer: final_pool_size = observed_peak × 1.2
```

---

## ✅ Checklist

- [ ] Load test được chạy trước mỗi major release
- [ ] Test scenario phản ánh đúng production traffic distribution
- [ ] Think time được thêm vào giữa actions
- [ ] Thresholds được định nghĩa dựa trên SLA thực tế
- [ ] Results được lưu và so sánh giữa các releases
- [ ] Stress test đã được chạy để biết breaking point
- [ ] Soak test đã được chạy ít nhất 1 lần để kiểm tra memory leak
- [ ] Load test tích hợp vào CI/CD với fail condition rõ ràng

---

## 🔗 Tổng Kết Module 07-performance

```
07-performance/ hoàn thành với:

1-caching-strategies.md  → Spring Cache + Redis, cache patterns, anti-patterns
2-connection-pooling.md  → HikariCP sizing, leak detection, monitoring
3-jvm-tuning.md          → Heap sizing, G1GC vs ZGC, GC logging, OOM handling
4-query-optimization.md  → EXPLAIN ANALYZE, indexes, batch ops, pagination
5-profiling.md           → async-profiler, JFR, Actuator metrics, heap/thread dumps
6-load-testing.md        → k6, Gatling, test types, CI/CD integration
```

**Xem tiếp:** [08-architecture/README.md](../08-architecture/README.md) — Kiến Trúc Ứng Dụng
