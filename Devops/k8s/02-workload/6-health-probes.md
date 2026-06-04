# Health Probes — Kiểm Tra Sức Khoẻ Container

> Kubernetes dùng ba loại probe để liên tục kiểm tra trạng thái container — quyết định khi nào restart, khi nào gửi traffic, và khi nào coi ứng dụng đã khởi động xong.

## Mục Lục

1. [Tổng Quan Ba Loại Probe](#tổng-quan-ba-loại-probe)
2. [Liveness Probe — Kiểm Tra Sức Sống](#liveness-probe--kiểm-tra-sức-sống)
3. [Readiness Probe — Kiểm Tra Sẵn Sàng](#readiness-probe--kiểm-tra-sẵn-sàng)
4. [Startup Probe — Kiểm Tra Khởi Động](#startup-probe--kiểm-tra-khởi-động)
5. [Cơ Chế Kiểm Tra (Handler)](#cơ-chế-kiểm-tra-handler)
6. [Tham Số Cấu Hình](#tham-số-cấu-hình)
7. [Chiến Lược Cấu Hình Thực Tế](#chiến-lược-cấu-hình-thực-tế)
8. [Ví Dụ Đầy Đủ](#ví-dụ-đầy-đủ)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Ba Loại Probe

| Probe | Mục Đích | Hành Động Khi Thất Bại | Gây Restart? |
| ----- | -------- | ----------------------- | ------------ |
| **Liveness Probe** (kiểm tra sức sống) | Container có đang chạy bình thường không? Có bị deadlock không? | Restart container | Có |
| **Readiness Probe** (kiểm tra sẵn sàng) | Container có sẵn sàng nhận traffic không? | Tạm ngừng gửi traffic đến Pod | Không |
| **Startup Probe** (kiểm tra khởi động) | Ứng dụng đã khởi động hoàn toàn chưa? | Restart container nếu không hoàn thành trong thời gian cho phép | Có |

### Vòng Đời Probe

```
Container khởi động
        ↓
Startup Probe chạy (nếu có)
  → Thất bại: restart container
  → Thành công: tiếp tục
        ↓
Liveness Probe + Readiness Probe chạy song song (liên tục)
  │
  ├── Liveness thất bại: restart container
  └── Readiness thất bại: xoá Pod khỏi Service Endpoints (không nhận traffic)
                          Pod vẫn chạy, không restart
```

---

## Liveness Probe — Kiểm Tra Sức Sống

**Liveness Probe** trả lời câu hỏi: *"Container có còn sống và hoạt động bình thường không?"*

Dùng để phát hiện:
- **Deadlock** (khóa chết): ứng dụng bị kẹt, không xử lý request nhưng process vẫn chạy
- **Memory leak** (rò rỉ bộ nhớ) dẫn đến ứng dụng không phản hồi
- **Infinite loop** (vòng lặp vô tận) nội bộ

**Hành động khi thất bại:** Kubelet **kill và restart** container. Thông thường sẽ thấy `RESTARTS` count tăng trong `kubectl get pods`.

### Khi Nào Cần Liveness Probe?

- Ứng dụng có thể bị deadlock và không tự phục hồi
- Ứng dụng cần cơ chế "rebooting" tự động khi stuck
- **Không cần** nếu ứng dụng tự exit khi gặp lỗi nghiêm trọng (tốt hơn)

### Ví Dụ

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15    # chờ 15 giây sau khi container bắt đầu
  periodSeconds: 20          # kiểm tra mỗi 20 giây
  failureThreshold: 3        # thất bại 3 lần liên tiếp mới restart
  successThreshold: 1        # 1 lần thành công là đủ (liveness không thể > 1)
  timeoutSeconds: 5          # timeout sau 5 giây
```

---

## Readiness Probe — Kiểm Tra Sẵn Sàng

**Readiness Probe** trả lời câu hỏi: *"Container có sẵn sàng nhận request từ người dùng không?"*

Dùng để ngăn traffic đến Pod khi:
- Ứng dụng đang **khởi động** và chưa sẵn sàng
- Ứng dụng đang **nạp dữ liệu** (warm up cache, load ML model)
- Ứng dụng đang **bận tạm thời** (kết nối database bị ngắt, đang xử lý công việc nặng)
- Trong quá trình **rolling update**: Pod mới chưa ready thì không nhận traffic

**Hành động khi thất bại:** Pod bị xoá khỏi **Endpoints** của Service — kubelet KHÔNG restart container. Khi Readiness Probe pass lại, Pod được thêm trở lại vào Endpoints.

```
Service → Endpoints → [Pod-1 ✓] [Pod-2 ✗] [Pod-3 ✓]
                                    ↑
                         Bị xoá khi readiness fail
                         Không nhận traffic
                         Container vẫn chạy
```

### Khác Biệt Quan Trọng So Với Liveness

| | Liveness Probe | Readiness Probe |
| - | -------------- | --------------- |
| Thất bại → | Restart container | Xoá khỏi Service Endpoints |
| Ứng dụng vẫn chạy? | Không (bị kill) | Có |
| Mục đích | Phát hiện ứng dụng bị lỗi | Kiểm soát traffic routing |

### Ví Dụ

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
  successThreshold: 1   # readiness: 1 lần thành công là đủ (trở lại nhận traffic)
```

**Endpoint HTTP khuyến nghị:**
- `/health` hoặc `/healthz` cho Liveness — trả về 200 nếu process còn sống
- `/ready` hoặc `/readyz` cho Readiness — kiểm tra database, cache, dependencies

```python
# Ví dụ Python/Flask
@app.route('/health')
def health():
    return {'status': 'ok'}, 200   # Liveness: chỉ cần process chạy

@app.route('/ready')
def ready():
    # Readiness: kiểm tra các dependencies
    try:
        db.execute('SELECT 1')          # database accessible?
        redis_client.ping()             # cache accessible?
        return {'status': 'ready'}, 200
    except Exception as e:
        return {'status': 'not ready', 'reason': str(e)}, 503
```

---

## Startup Probe — Kiểm Tra Khởi Động

**Startup Probe** giải quyết vấn đề: ứng dụng khởi động chậm bị Liveness Probe kill trước khi kịp sẵn sàng.

**Vấn đề trước khi có Startup Probe:**
```
Ứng dụng Java cần 60 giây để khởi động
Liveness Probe: initialDelaySeconds: 30, failureThreshold: 3, periodSeconds: 10
→ Liveness bắt đầu lúc t=30
→ t=30, t=40, t=50: app chưa ready → 3 lần fail
→ t=50: kubelet restart container
→ Ứng dụng không bao giờ khởi động được! (CrashLoopBackOff)
```

**Giải pháp với Startup Probe:**
```
Startup Probe: failureThreshold: 30, periodSeconds: 10
→ Startup Probe có tối đa 30 * 10 = 300 giây để ứng dụng khởi động
→ Trong thời gian đó, Liveness và Readiness Probe bị TẮT
→ Khi Startup Probe thành công, Liveness và Readiness mới bắt đầu
```

### Ví Dụ — Ứng Dụng Khởi Động Chậm

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30   # thử tối đa 30 lần
  periodSeconds: 10      # mỗi 10 giây → tổng 300 giây = 5 phút
  timeoutSeconds: 5
  successThreshold: 1

livenessProbe:
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 20
  failureThreshold: 3
  # Không cần initialDelaySeconds khi đã có Startup Probe

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 10
  failureThreshold: 3
```

**Tính thời gian tối đa:**
```
Thời gian startup tối đa = failureThreshold × periodSeconds
                         = 30 × 10 = 300 giây
```

---

## Cơ Chế Kiểm Tra (Handler)

Có ba cơ chế để thực hiện probe:

### 1. httpGet — HTTP Request

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
      - name: Custom-Header
        value: Awesome
    scheme: HTTPS   # mặc định: HTTP
```

Thành công khi HTTP response code từ 200 đến 399. Phù hợp cho web service, API server.

### 2. exec — Chạy Lệnh Trong Container

```yaml
livenessProbe:
  exec:
    command:
      - sh
      - -c
      - "pg_isready -U postgres && psql -U postgres -c 'SELECT 1'"
```

Thành công khi lệnh thoát với exit code 0. Phù hợp cho database, service không có HTTP interface.

```yaml
# Ví dụ: Kiểm tra Redis
livenessProbe:
  exec:
    command:
      - redis-cli
      - ping
# → trả về "PONG" và exit 0 nếu Redis sống
```

### 3. tcpSocket — Kiểm Tra TCP Connection

```yaml
livenessProbe:
  tcpSocket:
    port: 5432
```

Thành công khi kết nối TCP đến port được thiết lập. Đơn giản nhất, dùng khi ứng dụng không có HTTP endpoint (database, message broker raw TCP).

### 4. grpc — gRPC Health Check (Kubernetes 1.24+)

```yaml
livenessProbe:
  grpc:
    port: 50051
    service: "my.grpc.HealthService"   # tuỳ chọn
```

Dùng chuẩn [gRPC Health Checking Protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md).

---

## Tham Số Cấu Hình

| Tham Số | Mặc Định | Ý Nghĩa |
| ------- | -------- | ------- |
| `initialDelaySeconds` | 0 | Chờ N giây sau khi container bắt đầu trước khi chạy probe đầu tiên |
| `periodSeconds` | 10 | Khoảng thời gian giữa các lần probe |
| `timeoutSeconds` | 1 | Timeout cho mỗi lần probe |
| `successThreshold` | 1 | Số lần thành công liên tiếp để chuyển từ Failed → Healthy |
| `failureThreshold` | 3 | Số lần thất bại liên tiếp trước khi thực hiện hành động |

**Lưu ý về `successThreshold`:**
- Liveness Probe: **phải là 1** (không cho phép giá trị khác)
- Readiness Probe: có thể > 1 — Pod phải pass nhiều lần liên tiếp mới được thêm lại vào Endpoints. Dùng khi ứng dụng thỉnh thoảng trả về false positive.

### Tính Toán Thời Gian Tolerance

```
Thời gian phát hiện lỗi tối đa =
    initialDelaySeconds + (failureThreshold × periodSeconds) + (failureThreshold × timeoutSeconds)

Ví dụ:
    initialDelaySeconds: 10
    failureThreshold: 3
    periodSeconds: 10
    timeoutSeconds: 5
→   10 + (3 × 10) + (3 × 5) = 10 + 30 + 15 = 55 giây
```

---

## Chiến Lược Cấu Hình Thực Tế

### Ứng Dụng Khởi Động Nhanh (< 30 giây)

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 15
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
```

### Ứng Dụng Khởi Động Chậm (30 giây đến 5 phút)

```yaml
# Startup Probe: tối đa 5 phút để khởi động
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30
  periodSeconds: 10

# Liveness và Readiness: không cần initialDelaySeconds
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 20
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 10
  failureThreshold: 3
```

### Database (PostgreSQL, MySQL)

```yaml
livenessProbe:
  exec:
    command:
      - pg_isready
      - -U
      - postgres
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 6
  timeoutSeconds: 5

readinessProbe:
  exec:
    command:
      - pg_isready
      - -U
      - postgres
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 5
```

### Message Queue (Redis)

```yaml
livenessProbe:
  exec:
    command:
      - redis-cli
      - ping
  initialDelaySeconds: 15
  periodSeconds: 5
  failureThreshold: 3

readinessProbe:
  exec:
    command:
      - redis-cli
      - ping
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

---

## Ví Dụ Đầy Đủ

Ứng dụng microservice Node.js với cả ba probe:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service

spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-service

  template:
    metadata:
      labels:
        app: api-service

    spec:
      containers:
        - name: api
          image: my-api:1.5.0

          ports:
            - containerPort: 3000

          # ─── Startup Probe ──────────────────────────────
          # Node.js app: cho phép 60 giây để khởi động
          startupProbe:
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 12    # 12 × 5 giây = 60 giây tối đa
            periodSeconds: 5
            timeoutSeconds: 3

          # ─── Liveness Probe ─────────────────────────────
          # Phát hiện deadlock: restart nếu không phản hồi trong 60 giây
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            periodSeconds: 20          # kiểm tra mỗi 20 giây
            failureThreshold: 3        # 3 × 20 = 60 giây trước khi restart
            timeoutSeconds: 5
            successThreshold: 1

          # ─── Readiness Probe ────────────────────────────
          # Kiểm tra database và cache trước khi nhận traffic
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            periodSeconds: 10
            failureThreshold: 3
            successThreshold: 1
            timeoutSeconds: 3
```

Endpoint `/health` và `/ready` trong ứng dụng:

```javascript
// health.js — endpoints cho Kubernetes probes
const express = require('express');
const router = express.Router();

// Liveness: chỉ cần process còn sống
router.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// Readiness: kiểm tra tất cả dependencies
router.get('/ready', async (req, res) => {
  const checks = {};

  try {
    // Kiểm tra database
    await db.query('SELECT 1');
    checks.database = 'ok';
  } catch (err) {
    checks.database = 'error: ' + err.message;
  }

  try {
    // Kiểm tra Redis cache
    await redis.ping();
    checks.cache = 'ok';
  } catch (err) {
    checks.cache = 'error: ' + err.message;
  }

  const allOk = Object.values(checks).every(v => v === 'ok');

  res.status(allOk ? 200 : 503).json({
    status: allOk ? 'ready' : 'not ready',
    checks
  });
});

module.exports = router;
```

---

## Câu Hỏi Phỏng Vấn

### Q: Sự khác biệt giữa Liveness và Readiness Probe?

**Liveness** xác định container có còn "sống" không — thất bại dẫn đến **restart container**. Dùng để phát hiện deadlock hay trạng thái không thể tự phục hồi.

**Readiness** xác định container có sẵn sàng nhận traffic không — thất bại dẫn đến **xoá Pod khỏi Service Endpoints** nhưng không restart. Container vẫn tiếp tục chạy và sẽ nhận traffic lại khi Readiness pass.

Cần cả hai: Liveness tự động phục hồi ứng dụng khi bị stuck, Readiness đảm bảo chỉ Pod healthy mới nhận request.

### Q: Tại sao cần Startup Probe? Không thể tăng `initialDelaySeconds` thay thế?

`initialDelaySeconds` là fixed delay — nếu ứng dụng khởi động nhanh hơn dự kiến, vẫn phải chờ. Nếu đặt quá thấp, ứng dụng chậm bị kill. Startup Probe thích nghi được: chờ đến khi ứng dụng thực sự sẵn sàng, tối đa `failureThreshold × periodSeconds`. Khi ứng dụng sẵn sàng, Liveness/Readiness bắt đầu ngay — không lãng phí thời gian.

### Q: Điều gì xảy ra trong rolling update khi Readiness Probe fail?

Kubernetes dừng rolling update. Pod mới được tạo nhưng Readiness Probe fail → Pod không được thêm vào Service Endpoints → Deployment không xoá Pod cũ tiếp theo. Sau `progressDeadlineSeconds`, Deployment được đánh dấu là failed. Để tiếp tục: rollback với `kubectl rollout undo` hoặc fix bug rồi deploy lại. Đây là cơ chế bảo vệ quan trọng giúp zero-downtime deployment thực sự hoạt động.

### Q: Khi nào không nên dùng Liveness Probe?

Không nên dùng Liveness Probe nếu ứng dụng không có vấn đề deadlock rõ ràng — thêm Liveness Probe không cần thiết có thể gây restart không mong muốn do tạm thời tải cao, mạng chậm. Tốt hơn là để ứng dụng tự exit khi gặp lỗi nghiêm trọng (exit code ≠ 0) và để `restartPolicy: Always` xử lý phần còn lại. Liveness chỉ cần thiết cho ứng dụng bị stuck mà không tự exit.

---

**Xem Tiếp:** [deployment.md](./deployment.md) — Cách Deployment dùng Readiness Probe để đảm bảo Zero-Downtime Rolling Update
