# HPA — Horizontal Pod Autoscaler (Tự Động Mở Rộng Pod Theo Chiều Ngang)

> Hướng dẫn chi tiết về HPA (Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang): cơ chế hoạt động, cấu hình theo CPU/memory/custom metric, tinh chỉnh hành vi scale, debug và các pattern thực chiến.

## Mục Lục

1. [HPA Là Gì?](#hpa-là-gì)
2. [Cơ Chế Hoạt Động](#cơ-chế-hoạt-động)
3. [Cấu Hình HPA Cơ Bản](#cấu-hình-hpa-cơ-bản)
4. [Scale Theo Custom Metric](#scale-theo-custom-metric)
5. [Behaviour — Tinh Chỉnh Hành Vi Scale](#behaviour--tinh-chỉnh-hành-vi-scale)
6. [Kết Hợp Nhiều Metric](#kết-hợp-nhiều-metric)
7. [Debug và Giám Sát HPA](#debug-và-giám-sát-hpa)
8. [Vấn Đề Thường Gặp](#vấn-đề-thường-gặp)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## HPA Là Gì?

**HPA (Horizontal Pod Autoscaler)** là controller tích hợp sẵn trong Kubernetes tự động điều chỉnh số lượng Pod replica của Deployment, StatefulSet, hoặc ReplicaSet dựa trên các metric quan sát được.

```
              metrics-server
              (hoặc Prometheus Adapter)
                    │
                    │ cung cấp metric mỗi 15s
                    ▼
          ┌─────────────────┐
          │  HPA Controller │  ← chạy trong kube-controller-manager
          │  (control loop) │
          └────────┬────────┘
                   │
         tính toán desired replicas
         = currentReplicas × (currentMetric / targetMetric)
                   │
                   ▼
          Deployment / StatefulSet
          ┌──────────────────────┐
          │  Pod  Pod  Pod  Pod  │  ← thêm hoặc xoá Pod
          └──────────────────────┘
```

**Điều kiện để HPA hoạt động:**
1. Workload phải có `resources.requests` được khai báo (HPA dùng `requests` làm denominator)
2. **metrics-server** (hoặc custom metrics adapter) phải cài đặt trong cluster
3. HPA phải trỏ đến đúng `scaleTargetRef` (Deployment, StatefulSet, ReplicaSet)

---

## Cơ Chế Hoạt Động

### Vòng Lặp Điều Khiển (Control Loop)

HPA controller chạy mỗi **15 giây** (cấu hình bởi `--horizontal-pod-autoscaler-sync-period`):

```
1. Lấy metric hiện tại từ metrics API
   currentMetric = avg(CPU usage across all Pods)

2. Tính desired replicas
   desiredReplicas = ceil(currentReplicas × currentMetric / targetMetric)

   Ví dụ:
   - currentReplicas = 3
   - currentCPU = 80% (trung bình trên 3 Pod)
   - targetCPU = 50%
   - desiredReplicas = ceil(3 × 80/50) = ceil(4.8) = 5

3. Áp dụng giới hạn min/max
   clampedReplicas = clamp(desiredReplicas, minReplicas, maxReplicas)

4. So sánh với current replicas
   Nếu clampedReplicas != currentReplicas → scale
```

### Cooldown / Stabilization Window

HPA có **stabilization window (cửa sổ ổn định)** để tránh scale quá nhanh (flapping):

```
Scale Up:   stabilizationWindowSeconds = 0   (scale up ngay lập tức — mặc định)
Scale Down: stabilizationWindowSeconds = 300 (đợi 5 phút — mặc định)
```

Trong stabilization window, HPA **nhớ lại tất cả recommendation** và chọn recommendation **bảo thủ nhất** — tức là không scale down vội khi metric giảm tạm thời.

---

## Cấu Hình HPA Cơ Bản

### Scale Theo CPU Utilization

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-api
  minReplicas: 2        # tối thiểu 2 Pod (đảm bảo high availability)
  maxReplicas: 20       # tối đa 20 Pod

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60   # mục tiêu 60% CPU utilization trung bình
```

**Lưu ý về target utilization:** Đặt ở 60–70%, không phải 80–90%. Nếu đặt 80%, khi traffic tăng đột biến, HPA cần thêm vài chục giây để phát hiện và scale — trong khoảng đó CPU có thể vọt lên 100% và request timeout.

### Scale Theo Memory

```yaml
metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 512Mi   # trung bình memory mỗi Pod không vượt 512Mi
```

> **Cảnh báo:** Scale theo memory thường không hiệu quả — memory không giảm ngay khi load giảm (process giữ lại bộ nhớ đã dùng), dẫn đến HPA không scale down đúng lúc. Dùng CPU hoặc custom metric tốt hơn cho hầu hết trường hợp.

### Deployment Tương Ứng (phải khai báo requests)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-api
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-api
  template:
    metadata:
      labels:
        app: web-api
    spec:
      containers:
        - name: web-api
          image: myapp:1.0.0
          resources:
            requests:
              cpu: "250m"       # HPA dùng giá trị này làm 100% baseline
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

---

## Scale Theo Custom Metric

### Prometheus Adapter — Cầu Nối Prometheus và HPA

Để HPA dùng metric từ Prometheus (VD: số request/giây, queue depth), cần cài **Prometheus Adapter** — một implementation của Kubernetes Custom Metrics API.

```
Prometheus ────scrape────→ metric từ app
     │
     │  Prometheus Adapter chuyển đổi
     │  thành custom metrics API
     ▼
HPA ─── query custom metrics API ──→ quyết định scale
```

### Scale Theo Số Request/Giây

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-api-rps-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-api
  minReplicas: 2
  maxReplicas: 30

  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second     # metric name trong Prometheus Adapter config
        target:
          type: AverageValue
          averageValue: "100"               # mỗi Pod xử lý tối đa 100 req/s
```

### Scale Theo External Metric (Queue Length)

```yaml
metrics:
  - type: External
    external:
      metric:
        name: sqs_queue_depth               # metric từ CloudWatch Exporter
        selector:
          matchLabels:
            queue: "order-processing"
      target:
        type: Value
        value: "500"                        # scale khi queue > 500 message
```

---

## Behaviour — Tinh Chỉnh Hành Vi Scale

`behavior` field (autoscaling/v2) cho phép kiểm soát tốc độ scale up và scale down:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-api
  minReplicas: 2
  maxReplicas: 50

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0        # scale up ngay khi metric vượt ngưỡng
      policies:
        - type: Pods
          value: 4                          # tối đa thêm 4 Pod mỗi lần
          periodSeconds: 60
        - type: Percent
          value: 100                        # hoặc tối đa tăng 100% số hiện tại mỗi lần
          periodSeconds: 60
      selectPolicy: Max                     # chọn policy cho phép scale nhiều nhất

    scaleDown:
      stabilizationWindowSeconds: 300       # đợi 5 phút trước khi scale down
      policies:
        - type: Pods
          value: 2                          # tối đa giảm 2 Pod mỗi lần
          periodSeconds: 60
      selectPolicy: Min                     # chọn policy scale down ít nhất (bảo thủ hơn)
```

**Giải thích `selectPolicy`:**
- `Max` — chọn policy cho phép **thay đổi nhiều nhất** (thường dùng cho scale up — muốn scale nhanh)
- `Min` — chọn policy cho phép **thay đổi ít nhất** (thường dùng cho scale down — muốn scale chậm để tránh flapping)
- `Disabled` — vô hiệu hoá scale theo hướng đó hoàn toàn

---

## Kết Hợp Nhiều Metric

Khi HPA có nhiều metric, nó chọn **số replica cao nhất** trong tất cả các tính toán:

```yaml
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60    # tính toán từ CPU → VD: cần 5 Pod

  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"       # tính toán từ RPS → VD: cần 8 Pod

# HPA chọn max(5, 8) = 8 Pod → scale lên 8 Pod
```

Điều này đảm bảo ứng dụng không bị quá tải theo bất kỳ chiều nào.

---

## Debug và Giám Sát HPA

### Xem Trạng Thái HPA

```bash
# Xem tất cả HPA và metric hiện tại
kubectl get hpa -n production

# Output ví dụ:
# NAME          REFERENCE        TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
# web-api-hpa   Deployment/api   48%/60%         2         20        5          2d

# Xem chi tiết — bao gồm conditions và events
kubectl describe hpa web-api-hpa -n production
```

### Kiểm Tra Metric API

```bash
# Kiểm tra metrics-server đang chạy
kubectl get pods -n kube-system | grep metrics-server

# Xem CPU/memory của Pod trực tiếp từ metrics API
kubectl top pods -n production
kubectl top nodes

# Kiểm tra custom metrics API
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq .
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1" | jq .
```

### Xem Events của HPA

```bash
# Xem events liên quan đến scaling
kubectl describe hpa web-api-hpa -n production | grep -A 20 "Events:"

# Output ví dụ:
# Events:
#   Normal  SuccessfulRescale   2m    horizontal-pod-autoscaler
#           New size: 8; reason: cpu resource utilization (percentage of request)
#           above target
```

### TARGETS Hiện Thị `<unknown>`

```bash
# Lỗi phổ biến: metrics-server không chạy hoặc Pod thiếu resource requests
kubectl get hpa -n production
# NAME          TARGETS           → <unknown>/60%  ← vấn đề!

# Kiểm tra 1: metrics-server
kubectl top pods -n production
# Nếu lỗi "error: metrics not available yet" → metrics-server chưa ready

# Kiểm tra 2: Pod có resource requests không?
kubectl get deployment web-api -n production -o jsonpath='{.spec.template.spec.containers[*].resources}'
```

---

## Vấn Đề Thường Gặp

### 1. HPA Scale Quá Nhanh Rồi Scale Xuống Liên Tục (Flapping)

**Triệu chứng:** Replica count dao động liên tục: 5 → 10 → 5 → 10...

**Nguyên nhân:** Target utilization quá gần với load thực tế, hoặc stabilizationWindowSeconds quá ngắn.

**Giải pháp:**
```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300   # tăng lên 5 phút
    policies:
      - type: Pods
        value: 2
        periodSeconds: 120            # scale down chậm hơn
```

### 2. HPA Không Scale Dù CPU Cao

**Nguyên nhân có thể:**
- Pod thiếu `resources.requests.cpu` → HPA không có denominator
- metrics-server chưa chạy hoặc crash
- HPA đang trong cooldown period
- Đã đạt `maxReplicas`

```bash
# Kiểm tra condition của HPA
kubectl describe hpa web-api-hpa | grep -A 5 "Conditions:"
# ScalingActive  False  FailedGetScale  → thiếu target ref
# AbleToScale    False  BackoffBoth     → đang trong cooldown
```

### 3. Scale Chậm Khi Traffic Spike

**Nguyên nhân:** `stabilizationWindowSeconds` cho scale up quá lớn, hoặc policy `value` quá nhỏ.

**Giải pháp:**
```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0     # scale up ngay
    policies:
      - type: Percent
        value: 200                    # tăng 200% mỗi 30s (aggressive scale up)
        periodSeconds: 30
```

---

## Câu Hỏi Phỏng Vấn

**HPA tính toán số replica như thế nào?**

> HPA dùng công thức: `desiredReplicas = ceil(currentReplicas × currentMetricValue / targetMetricValue)`. Ví dụ: đang có 4 Pod, trung bình CPU là 80%, target là 50% → `ceil(4 × 80/50) = ceil(6.4) = 7 Pod`. Khi metric giảm về 30%: `ceil(7 × 30/50) = ceil(4.2) = 5 Pod`, nhưng scale down chưa xảy ra ngay vì `stabilizationWindowSeconds` mặc định 300 giây cho scale down.

**Tại sao cần khai báo `resources.requests` để HPA hoạt động?**

> HPA tính CPU utilization theo phần trăm = `actualCPUUsage / requestedCPU × 100%`. Nếu không có `requests`, HPA không biết "100% CPU" của Pod là bao nhiêu millicore, không thể tính utilization percentage. Container không có `requests` sẽ bị HPA bỏ qua trong phép tính trung bình, dẫn đến kết quả sai hoặc `TARGETS: <unknown>`.

**Phân biệt `type: Utilization` và `type: AverageValue`?**

> `type: Utilization` — tính **phần trăm** so với `resources.requests`. Dùng cho Resource metric (CPU, memory). Ví dụ: CPU request 500m, target utilization 60% → HPA muốn trung bình mỗi Pod dùng 300m CPU. `type: AverageValue` — giá trị tuyệt đối **trên mỗi Pod**, không cần request. Dùng cho custom metric. Ví dụ: `averageValue: 100req/s` → HPA tính tổng RPS chia cho 100 để biết cần bao nhiêu Pod.

**Làm thế nào để tránh HPA scale down gây downtime?**

> 4 biện pháp: (1) `stabilizationWindowSeconds` cho scale down đủ dài (300–600s) — không scale down ngay khi metric giảm tạm thời; (2) `behavior.scaleDown.policies` giới hạn số Pod bị xoá mỗi lần — ví dụ tối đa 2 Pod mỗi 60 giây; (3) Kết hợp với `PodDisruptionBudget` — giới hạn số Pod gián đoạn đồng thời tối đa; (4) Deployment `rollingUpdate.maxUnavailable: 0` — không bao giờ để Pod count < desired khi scale. Cùng với đó, `minReplicas >= 2` đảm bảo không bao giờ scale về 1 Pod duy nhất.
