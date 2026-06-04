# Progressive Delivery — Triển Khai Dần Dần và An Toàn

> Progressive Delivery (Triển Khai Dần Dần) là phương pháp giảm thiểu rủi ro khi deploy bằng cách đưa version mới ra dần dần, kết hợp với phân tích metric tự động để quyết định promote (tiến lên) hay rollback (lùi lại). Công cụ chính trong Kubernetes ecosystem là Argo Rollouts.

## Mục Lục

1. [Progressive Delivery Là Gì](#progressive-delivery-là-gì)
2. [Argo Rollouts — Kiến Trúc](#argo-rollouts--kiến-trúc)
3. [Canary Deployment — Triển Khai Canary](#canary-deployment--triển-khai-canary)
4. [Blue-Green Deployment — Triển Khai Xanh-Lam](#blue-green-deployment--triển-khai-xanh-lam)
5. [AnalysisTemplate — Phân Tích Metric Tự Động](#analysistemplate--phân-tích-metric-tự-động)
6. [Tích Hợp Với Ingress Controller](#tích-hợp-với-ingress-controller)
7. [Tích Hợp Với ArgoCD](#tích-hợp-với-argocd)
8. [So Sánh Các Chiến Lược Deploy](#so-sánh-các-chiến-lược-deploy)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Progressive Delivery Là Gì

### Vấn Đề Của Deploy Truyền Thống

```
Rolling Update truyền thống (Kubernetes mặc định):
Version cũ: [Pod v1] [Pod v1] [Pod v1] [Pod v1]
                         │ kubectl apply new deployment
                         ▼
Trong quá trình:  [Pod v2] [Pod v1] [Pod v1] [Pod v2]
                         │
                         ▼
Version mới:  [Pod v2] [Pod v2] [Pod v2] [Pod v2]

Vấn đề:
- Nếu v2 có bug, 100% user bị ảnh hưởng trước khi phát hiện
- Không có cơ chế tự động phát hiện và rollback dựa trên metric
- Không biết chính xác bao nhiêu % traffic đang đến v2
```

### Progressive Delivery Giải Quyết

```
Progressive Delivery (Argo Rollouts):

Bước 1 — 20% traffic đến v2:
  [Pod v2] [Pod v2] [Pod v1] [Pod v1] [Pod v1] [Pod v1] [Pod v1] [Pod v1]
      │
      ├── Đo metric 5 phút: error rate, latency
      ├── metric OK? → tự động tiến sang bước 2
      └── metric FAIL? → tự động rollback 100% về v1

Bước 2 — 50% traffic đến v2:
  [Pod v2] [Pod v2] [Pod v2] [Pod v2] [Pod v1] [Pod v1] [Pod v1] [Pod v1]
      │
      ├── Đo metric 10 phút
      └── ...

Bước 3 — 100% traffic đến v2:
  [Pod v2] [Pod v2] [Pod v2] [Pod v2] [Pod v2] [Pod v2] [Pod v2] [Pod v2]
```

### Các Hình Thức Progressive Delivery

| Chiến Lược | Mô Tả | Khi Nào Dùng |
|---|---|---|
| **Canary** | Tăng dần % traffic đến version mới | Production thông thường, thay đổi API |
| **Blue-Green** | Chạy song song 2 version, switch toàn bộ | Zero-downtime, cần rollback ngay |
| **A/B Testing** | Route traffic theo header/cookie | Test feature với nhóm user cụ thể |
| **Shadow** | Copy traffic đến version mới nhưng không trả response | Test performance mà không ảnh hưởng user |

---

## Argo Rollouts — Kiến Trúc

**Argo Rollouts** thay thế Deployment bằng **Rollout** CRD, cung cấp các chiến lược deploy nâng cao với khả năng phân tích metric tự động.

```bash
# Cài Argo Rollouts
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Cài kubectl plugin để quản lý
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

### Rollout vs Deployment

```yaml
# Deployment truyền thống
apiVersion: apps/v1
kind: Deployment
spec:
  strategy:
    type: RollingUpdate

# Argo Rollouts — thay Deployment bằng Rollout
apiVersion: argoproj.io/v1alpha1
kind: Rollout                   # ← thay đổi duy nhất ở apiVersion và kind
spec:
  strategy:
    canary:                     # ← thêm strategy nâng cao
      steps: [...]
    # hoặc
    blueGreen:
      activeService: my-app-active
      previewService: my-app-preview
```

---

## Canary Deployment — Triển Khai Canary

### Canary Cơ Bản Dựa Trên Replica

```yaml
# rollout-canary.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 10
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-registry.io/my-app:v1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10

  strategy:
    canary:
      steps:
        - setWeight: 10          # bước 1: 10% traffic đến canary
        - pause:
            duration: 5m         # đợi 5 phút
        - setWeight: 30
        - pause:
            duration: 10m
        - setWeight: 60
        - pause:
            duration: 10m
        - setWeight: 100         # 100% → promote hoàn toàn

      # Tự động rollback nếu vượt ngưỡng
      maxSurge: "25%"            # tối đa thêm 25% Pod tạm thời
      maxUnavailable: 0          # không có Pod nào down trong quá trình
```

### Canary Với Automatic Analysis (Phân Tích Tự Động)

```yaml
strategy:
  canary:
    steps:
      - setWeight: 20
      - analysis:
          templates:
            - templateName: error-rate-analysis    # tham chiếu AnalysisTemplate
          args:
            - name: service-name
              value: my-app-canary
      - pause:
          duration: 5m
      - setWeight: 50
      - analysis:
          templates:
            - templateName: error-rate-analysis
            - templateName: latency-analysis
      - setWeight: 100

    canaryService: my-app-canary    # Service riêng cho canary traffic
    stableService: my-app-stable    # Service cho stable traffic
    trafficRouting:
      nginx:
        stableIngress: my-app-ingress
```

---

## Blue-Green Deployment — Triển Khai Xanh-Lam

### Nguyên Tắc Blue-Green

```
Blue-Green Architecture:

"Active Service" (traffic thực): → Blue (version hiện tại)
"Preview Service" (test nội bộ): → Green (version mới)

Before switch:
User traffic → Active Service → Blue Pods (v1) ← 100% traffic
                                Green Pods (v2) ← 0% user traffic, chỉ smoke test

After switch (instant):
User traffic → Active Service → Green Pods (v2) ← 100% traffic
                                Blue Pods (v1) ← sẵn sàng rollback ngay

Ưu điểm:
- Switch ngay lập tức, không có trạng thái "đang chuyển"
- Rollback cực nhanh — chỉ switch service selector
- Blue Pods giữ nguyên X phút/giờ → rollback không cần rebuild

Nhược điểm:
- Tốn gấp đôi tài nguyên trong thời gian deploy
- Database schema phải backward-compatible (tương thích ngược)
```

### Rollout Với Blue-Green Strategy

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  replicas: 5
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-registry.io/my-app:v1.0.0

  strategy:
    blueGreen:
      activeService: my-app-active      # Service nhận traffic thực
      previewService: my-app-preview    # Service cho version mới (test)

      autoPromotionEnabled: false       # false = chờ manual approval trước khi switch

      # Hoặc tự động promote sau X giây
      # autoPromotionSeconds: 60

      # Giữ Blue Pods X giây sau khi switch (để rollback nhanh)
      scaleDownDelaySeconds: 600        # 10 phút

      # Chạy analysis trước khi auto promote
      prePromotionAnalysis:
        templates:
          - templateName: smoke-test
      postPromotionAnalysis:
        templates:
          - templateName: production-validation
```

### Service Manifest Cho Blue-Green

```yaml
# Service nhận traffic thực của user
apiVersion: v1
kind: Service
metadata:
  name: my-app-active
spec:
  selector:
    app: my-app                    # ArgoCD sẽ update rollouts-pod-template-hash để chỉ đúng version
  ports:
    - port: 80
      targetPort: 8080
---
# Service cho preview (version mới, chỉ nội bộ test)
apiVersion: v1
kind: Service
metadata:
  name: my-app-preview
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

### Các Lệnh Quản Lý Blue-Green

```bash
# Xem trạng thái rollout
kubectl argo rollouts get rollout my-app -n production --watch

# Promote (switch active → green) thủ công
kubectl argo rollouts promote my-app -n production

# Abort (rollback về blue) trước khi switch
kubectl argo rollouts abort my-app -n production

# Undo (rollback về version trước) sau khi đã switch
kubectl argo rollouts undo my-app -n production
```

---

## AnalysisTemplate — Phân Tích Metric Tự Động

**AnalysisTemplate** định nghĩa các metric query (truy vấn metric) để đánh giá sức khoẻ của canary/green version. Nếu metric fail, Rollout tự động rollback.

### AnalysisTemplate Với Prometheus

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-analysis
  namespace: production
spec:
  args:
    - name: service-name                    # tham số được truyền từ Rollout

  metrics:
    # Metric 1: Error rate không vượt 5%
    - name: error-rate
      interval: 1m                          # đo mỗi 1 phút
      count: 5                              # đo 5 lần
      successCondition: result[0] <= 0.05  # pass nếu error rate <= 5%
      failureLimit: 1                       # cho phép fail 1 lần trước khi abort
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc.cluster.local:9090
          query: |
            sum(rate(http_requests_total{
              job="{{args.service-name}}",
              status=~"5.."
            }[5m]))
            /
            sum(rate(http_requests_total{
              job="{{args.service-name}}"
            }[5m]))

    # Metric 2: P99 latency dưới 500ms
    - name: p99-latency
      interval: 1m
      count: 5
      successCondition: result[0] <= 500     # milliseconds
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc.cluster.local:9090
          query: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_milliseconds_bucket{
                job="{{args.service-name}}"
              }[5m])) by (le)
            )

    # Metric 3: Tỉ lệ Pod healthy
    - name: pods-healthy
      interval: 30s
      count: 3
      successCondition: result[0] >= 0.9    # >= 90% pod phải healthy
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc.cluster.local:9090
          query: |
            sum(kube_pod_status_ready{
              namespace="production",
              pod=~"my-app.*",
              condition="true"
            })
            /
            sum(kube_pod_status_ready{
              namespace="production",
              pod=~"my-app.*"
            })
```

### AnalysisTemplate Với Web Hook (Tích Hợp Datadog, New Relic)

```yaml
metrics:
  - name: datadog-error-rate
    interval: 1m
    count: 5
    successCondition: result < 5
    provider:
      web:
        url: "https://api.datadoghq.com/api/v1/query"
        method: GET
        headers:
          - key: "DD-API-KEY"
            value: "{{args.datadog-api-key}}"
        jsonPath: ".series[0].pointlist[-1][1]"  # lấy giá trị cuối từ response JSON
```

### ClusterAnalysisTemplate (Dùng Chung Nhiều Namespace)

```yaml
# ClusterAnalysisTemplate áp dụng cho toàn cluster
apiVersion: argoproj.io/v1alpha1
kind: ClusterAnalysisTemplate
metadata:
  name: standard-canary-checks
spec:
  metrics:
    - name: error-rate
      # ... giống AnalysisTemplate nhưng scope toàn cluster
```

---

## Tích Hợp Với Ingress Controller

Để kiểm soát traffic weight chính xác (ví dụ: đúng 10%, không phụ thuộc số replica), Argo Rollouts tích hợp với Ingress Controller.

### Tích Hợp Với NGINX Ingress

```yaml
# Ingress cho stable service (traffic thực)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: my-app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-stable
                port:
                  number: 80
---
# Rollout cấu hình traffic splitting qua NGINX annotation
strategy:
  canary:
    stableService: my-app-stable
    canaryService: my-app-canary
    trafficRouting:
      nginx:
        stableIngress: my-app          # tên Ingress
        additionalIngressAnnotations:  # annotation thêm vào canary ingress
          canary-by-header: X-Canary   # route canary nếu có header này
    steps:
      - setWeight: 5      # 5% traffic → canary (chính xác, không phụ thuộc replica count)
      - pause:
          duration: 2m
      - setWeight: 20
      - pause:
          duration: 5m
```

### Tích Hợp Với AWS ALB (Application Load Balancer — Cân Bằng Tải Ứng Dụng)

```yaml
strategy:
  canary:
    stableService: my-app-stable
    canaryService: my-app-canary
    trafficRouting:
      alb:
        ingress: my-app-alb-ingress
        servicePort: 80
        annotationPrefix: kubernetes.io/ingress   # annotation prefix cho AWS ALB
    steps:
      - setWeight: 10
      - pause:
          duration: 5m
```

---

## Tích Hợp Với ArgoCD

Argo Rollouts tích hợp tự nhiên với ArgoCD. ArgoCD nhận biết Rollout resource và hiển thị trạng thái canary/blue-green trong UI.

```yaml
# Application ArgoCD quản lý Rollout
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/my-org/my-app-config
    path: helm/my-app
    helm:
      valueFiles: [values-prod.yaml]
  destination:
    namespace: production
  # ArgoCD health check tùy chỉnh cho Rollout
  # (ArgoCD tự động nhận biết nếu cài argo-rollouts trong cluster)
```

```bash
# ArgoCD CLI với Rollouts
argocd app get my-app              # xem health kể cả rollout status
kubectl argo rollouts dashboard    # mở Argo Rollouts Dashboard trên localhost
```

---

## So Sánh Các Chiến Lược Deploy

| Tiêu Chí | Rolling Update (K8s mặc định) | Canary | Blue-Green |
|---|---|---|---|
| **Rủi ro** | Trung bình (thay dần) | Thấp (kiểm soát % traffic) | Thấp (test trước khi switch) |
| **Thời gian deploy** | Nhanh | Chậm hơn (nhiều bước) | Tương đương nhưng cần chờ test |
| **Tài nguyên** | Thấp (maxSurge) | Thấp (một phần replica) | Cao (gấp đôi trong thời gian deploy) |
| **Rollback** | Slow (rolling back) | Nhanh (về stable service) | Rất nhanh (switch service) |
| **Phát hiện lỗi** | Thủ công | Tự động qua metric | Thủ công (smoke test) hoặc tự động |
| **Zero downtime** | Có (nếu readinessProbe đúng) | Có | Có (switch ngay lập tức) |
| **Database migration** | Backward-compatible cần thiết | Backward-compatible cần thiết | Backward-compatible bắt buộc |
| **Phù hợp** | Thay đổi nhỏ, backend service | API thay đổi, cần kiểm soát rủi ro | Database schema change, critical service |

---

## Câu Hỏi Phỏng Vấn

**Canary deployment trong Kubernetes thuần có hạn chế gì so với Argo Rollouts?**

> Kubernetes thuần hỗ trợ canary cơ bản bằng cách chạy hai Deployment (stable và canary) với label selector khác nhau. Có 4 hạn chế chính: (1) **Kiểm soát traffic thô:** Tỉ lệ traffic phụ thuộc vào số replica — muốn 1% canary phải có 99 stable + 1 canary = 100 Pod, rất tốn tài nguyên; (2) **Không có metric analysis:** Phải tự viết script monitor và trigger rollback; (3) **Không có header-based routing:** Không thể route chỉ request từ internal team hoặc có header `X-Beta: true`; (4) **Quản lý thủ công:** Phải tự tăng dần replica và monitor mỗi bước. Argo Rollouts giải quyết tất cả bằng cách tích hợp với ingress/service mesh cho weight chính xác, AnalysisTemplate cho auto-rollback, và step-based workflow tự động.

**Blue-Green deployment có thể gặp vấn đề gì với database?**

> Vấn đề lớn nhất là **database schema migration (thay đổi schema database)**. Nếu version mới (Green) cần schema change (thêm/xóa/đổi tên column), trong khi Blue vẫn đang chạy với schema cũ, có xung đột: Green cần schema mới để hoạt động, nhưng nếu phải rollback về Blue, Blue có thể crash vì schema đã thay đổi. Giải pháp: **expand-contract pattern** (mô hình mở rộng-thu hẹp) — deploy schema change theo hai giai đoạn: (1) Expand: thêm column mới nhưng không xóa column cũ → cả Blue và Green đều chạy được với schema này; (2) Sau khi Blue-Green switch ổn định: Contract — xóa column cũ trong deploy tiếp theo. Cách này phức tạp hơn nhưng đảm bảo khả năng rollback an toàn.

**AnalysisTemplate có thể tích hợp với những gì ngoài Prometheus?**

> AnalysisTemplate hỗ trợ nhiều provider. (1) **Prometheus/Thanos/Cortex** — phổ biến nhất, dùng PromQL query; (2) **Datadog** — dùng Datadog Metrics API, phù hợp team đang dùng Datadog; (3) **New Relic** — NRQL query; (4) **CloudWatch** — AWS metrics, phù hợp workload trên AWS; (5) **Web (HTTP)** — gọi bất kỳ HTTP endpoint nào trả về metric — linh hoạt nhất, có thể tích hợp với bất kỳ hệ thống nào; (6) **Kubernetes Jobs** — chạy một Job tùy chỉnh và dựa trên exit code để quyết định pass/fail — phù hợp cho integration test hay smoke test phức tạp. Thực tế production thường kết hợp: Prometheus cho metric cơ bản (error rate, latency) + Web hook để trigger smoke test + Job để chạy end-to-end test.
