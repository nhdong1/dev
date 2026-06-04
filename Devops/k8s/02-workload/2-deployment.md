# Deployment — Triển Khai Và Cập Nhật Không Gián Đoạn

> Deployment quản lý ReplicaSet (tập bản sao) để đảm bảo số lượng Pod mong muốn luôn chạy, đồng thời cung cấp cơ chế rolling update và rollback có kiểm soát.

## Mục Lục

1. [Deployment Là Gì?](#deployment-là-gì)
2. [Quan Hệ Deployment → ReplicaSet → Pod](#quan-hệ-deployment--replicaset--pod)
3. [Update Strategy — Chiến Lược Cập Nhật](#update-strategy--chiến-lược-cập-nhật)
4. [Rolling Update Chi Tiết](#rolling-update-chi-tiết)
5. [Rollback — Quay Lại Phiên Bản Cũ](#rollback--quay-lại-phiên-bản-cũ)
6. [Scaling — Mở Rộng](#scaling--mở-rộng)
7. [Deployment Manifest Đầy Đủ](#deployment-manifest-đầy-đủ)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Deployment Là Gì?

**Deployment** là một higher-level abstraction (lớp trừu tượng cao hơn) trên Pod và ReplicaSet. Bạn khai báo trạng thái mong muốn (số lượng replica, image version, cấu hình) và Deployment Controller tự đảm bảo trạng thái thực tế khớp với khai báo.

**Chức năng chính:**
- Duy trì N bản sao Pod (high availability — tính sẵn sàng cao)
- Cập nhật ứng dụng mà không gây downtime (rolling update)
- Quay lại phiên bản trước khi có sự cố (rollback)
- Tạm dừng và tiếp tục quá trình cập nhật (pause/resume)
- Scaling thủ công hoặc tự động (với HPA)

---

## Quan Hệ Deployment → ReplicaSet → Pod

```
Deployment (my-app)
    │
    ├── ReplicaSet (my-app-7d4b8c9f5)   ← phiên bản cũ (0 replica)
    │
    └── ReplicaSet (my-app-6a2f3e1b7)   ← phiên bản hiện tại (3 replicas)
            ├── Pod (my-app-6a2f3e1b7-xk9p2)
            ├── Pod (my-app-6a2f3e1b7-r7m4n)
            └── Pod (my-app-6a2f3e1b7-h2v8q)
```

**Khi rolling update xảy ra:**
1. Deployment tạo **ReplicaSet mới** cho phiên bản mới
2. Tăng dần replica của ReplicaSet mới
3. Giảm dần replica của ReplicaSet cũ
4. ReplicaSet cũ vẫn tồn tại (có 0 replica) để hỗ trợ rollback

Số lượng ReplicaSet lịch sử được giữ lại được kiểm soát bởi `revisionHistoryLimit` (mặc định: 10).

---

## Update Strategy — Chiến Lược Cập Nhật

### RollingUpdate (Cập Nhật Cuốn) — Mặc Định

Thay thế Pod cũ bằng Pod mới **từng phần**, đảm bảo luôn có một số Pod đang phục vụ.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1     # số Pod tối đa có thể không available trong khi update
    maxSurge: 1           # số Pod mới tối đa có thể tạo vượt quá replicas
```

**Ví dụ với 3 replicas, maxUnavailable=1, maxSurge=1:**

```
Trước:  [v1] [v1] [v1]          (3 Pod v1)
Bước 1: [v1] [v1] [v1] [v2]    (tạo 1 Pod v2 — dùng maxSurge)
Bước 2: [v1] [v1]      [v2]    (xoá 1 Pod v1 — dùng maxUnavailable)
Bước 3: [v1] [v1] [v2] [v2]    (tạo thêm 1 Pod v2)
Bước 4: [v1]      [v2] [v2]    (xoá 1 Pod v1)
Bước 5: [v1] [v2] [v2] [v2]    (tạo thêm 1 Pod v2)
Hoàn:        [v2] [v2] [v2]    (xoá Pod v1 cuối)
```

**Lựa chọn giá trị:**

| Tình Huống | maxUnavailable | maxSurge | Kết Quả |
| ---------- | -------------- | -------- | ------- |
| Tiết kiệm resource | 1 | 0 | Xoá trước tạo sau — có downtime cục bộ |
| Tối ưu tốc độ | 0 | 1 | Tạo trước xoá sau — không downtime, cần thêm resource |
| Cân bằng | 1 | 1 | Mặc định — phù hợp hầu hết trường hợp |
| Update nhanh | 25% | 25% | Kubernetes mặc định khi không khai báo |

### Recreate (Tạo Lại) — Có Downtime

Xoá **tất cả** Pod cũ, rồi mới tạo Pod mới. Có downtime trong khoảng thời gian giữa hai lần.

```yaml
strategy:
  type: Recreate
```

**Dùng khi:** ứng dụng không thể chạy hai phiên bản cùng lúc (database schema migration không tương thích ngược, ứng dụng giữ lock trên tài nguyên chia sẻ).

---

## Rolling Update Chi Tiết

### Trigger Rolling Update

Rolling update tự động xảy ra khi bạn thay đổi Pod template (`.spec.template`):

```bash
# Cập nhật image
kubectl set image deployment/my-app app=my-app:2.0

# Cập nhật biến môi trường
kubectl set env deployment/my-app APP_VERSION=2.0

# Chỉnh sửa trực tiếp
kubectl edit deployment/my-app

# Apply file manifest mới
kubectl apply -f deployment-v2.yaml
```

Thay đổi ở cấp Deployment không trigger rolling update (ví dụ: chỉ thay đổi `replicas`).

### Theo Dõi Tiến Trình

```bash
# Xem trạng thái rollout
kubectl rollout status deployment/my-app

# Xem lịch sử rollout
kubectl rollout history deployment/my-app

# Xem chi tiết một revision cụ thể
kubectl rollout history deployment/my-app --revision=3

# Tạm dừng rollout (để test một phần Pod mới trước)
kubectl rollout pause deployment/my-app

# Tiếp tục rollout đã tạm dừng
kubectl rollout resume deployment/my-app
```

### minReadySeconds — Thời Gian Chờ Xác Nhận

`minReadySeconds` (mặc định: 0) — Kubernetes chờ Pod mới ở trạng thái `Ready` ít nhất bao nhiêu giây trước khi tiếp tục.

```yaml
spec:
  minReadySeconds: 10   # chờ 10 giây sau khi Pod mới Ready trước khi tạo Pod mới tiếp theo
```

Kết hợp với `readinessProbe`, đây là cơ chế đảm bảo Pod mới thực sự hoạt động trước khi tiếp tục update.

### progressDeadlineSeconds — Thời Hạn Hoàn Thành

```yaml
spec:
  progressDeadlineSeconds: 600   # rollout phải hoàn thành trong 10 phút, không thì báo lỗi
```

---

## Rollback — Quay Lại Phiên Bản Cũ

### Rollback Lập Tức

```bash
# Quay lại phiên bản trước
kubectl rollout undo deployment/my-app

# Quay lại revision cụ thể
kubectl rollout undo deployment/my-app --to-revision=2
```

### Xem Lịch Sử Để Chọn Revision

```bash
kubectl rollout history deployment/my-app
# REVISION  CHANGE-CAUSE
# 1         kubectl apply --filename=deployment.yaml
# 2         kubectl set image deployment/my-app app=my-app:1.1
# 3         kubectl set image deployment/my-app app=my-app:2.0  ← hiện tại
```

Để ghi lại `CHANGE-CAUSE` (nguyên nhân thay đổi), thêm annotation:

```bash
kubectl annotate deployment/my-app kubernetes.io/change-cause="deploy v2.0 — thêm tính năng OAuth"
```

### Giữ Đủ Lịch Sử Để Rollback

```yaml
spec:
  revisionHistoryLimit: 5   # giữ 5 ReplicaSet cũ (mặc định: 10)
```

---

## Scaling — Mở Rộng

### Scaling Thủ Công

```bash
# Scale lên 5 replicas
kubectl scale deployment/my-app --replicas=5

# Scale xuống 2 replicas
kubectl scale deployment/my-app --replicas=2
```

### Scaling Tự Động với HPA

HPA (Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang) tự động điều chỉnh số replicas dựa trên metric (CPU, memory, custom metric).

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # scale khi CPU trung bình vượt 70%
```

> Xem chi tiết tại `07-scaling/hpa.md`

---

## Deployment Manifest Đầy Đủ

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
    team: backend
  annotations:
    kubernetes.io/change-cause: "deploy v1.2.3 — fix memory leak"

spec:
  # ─── Số lượng bản sao ────────────────────────────────
  replicas: 3

  # ─── Giữ tối đa 5 ReplicaSet cũ để rollback ─────────
  revisionHistoryLimit: 5

  # ─── Selector — kết nối Deployment với Pod ──────────
  selector:
    matchLabels:
      app: my-app

  # ─── Chiến lược cập nhật ─────────────────────────────
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0   # không có Pod nào unavailable trong khi update
      maxSurge: 1         # tối đa thêm 1 Pod vượt quá replicas

  # ─── Xác nhận Pod ổn định trước khi tiếp tục ────────
  minReadySeconds: 10

  # ─── Thời hạn rollout tối đa ─────────────────────────
  progressDeadlineSeconds: 300

  # ─── Pod Template ────────────────────────────────────
  template:
    metadata:
      labels:
        app: my-app
        version: "1.2.3"

    spec:
      terminationGracePeriodSeconds: 30

      # Phân tán Pod trên nhiều node
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values: ["my-app"]
              topologyKey: kubernetes.io/hostname

      containers:
        - name: app
          image: my-app:1.2.3
          imagePullPolicy: IfNotPresent

          ports:
            - name: http
              containerPort: 8080

          env:
            - name: APP_ENV
              value: production
            - name: DB_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: database-url

          resources:
            requests:
              memory: "256Mi"
              cpu: "200m"
            limits:
              memory: "512Mi"
              cpu: "500m"

          # Chờ ứng dụng khởi động hoàn toàn trước khi chạy liveness/readiness
          startupProbe:
            httpGet:
              path: /health
              port: 8080
            failureThreshold: 30
            periodSeconds: 10

          # Kiểm tra ứng dụng còn sống không
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 15
            failureThreshold: 3

          # Kiểm tra ứng dụng sẵn sàng nhận traffic không
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3

          # Chờ 5 giây trước khi dừng để load balancer drain connections
          lifecycle:
            preStop:
              exec:
                command: ["sleep", "5"]

          # Bảo mật container
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true

          volumeMounts:
            - name: tmp-dir
              mountPath: /tmp

      volumes:
        - name: tmp-dir
          emptyDir: {}
```

---

## Câu Hỏi Phỏng Vấn

### Q: Sự khác biệt giữa RollingUpdate và Recreate?

**RollingUpdate**: thay thế Pod cũ từng phần — không có downtime nhưng hai phiên bản cùng chạy một lúc. Phù hợp cho hầu hết ứng dụng stateless.

**Recreate**: xoá hết Pod cũ rồi mới tạo Pod mới — có downtime nhưng không bao giờ chạy hai phiên bản cùng lúc. Dùng khi ứng dụng không tương thích ngược (database schema migration).

### Q: maxUnavailable=0 và maxSurge=1 có ý nghĩa gì?

Đây là chiến lược **"blue-green" cục bộ**: luôn tạo Pod mới trước khi xoá Pod cũ. Đảm bảo 100% capacity trong suốt quá trình update nhưng cần thêm tài nguyên. Phù hợp cho production khi không chấp nhận giảm capacity.

### Q: Làm thế nào để biết Deployment có đang healthy không?

```bash
kubectl rollout status deployment/my-app
# "deployment/my-app" successfully rolled out = bình thường
# Nếu bị stuck: xem events và pod logs

kubectl get deployment my-app
# READY = số Pod ready / số Pod mong muốn
# AVAILABLE = số Pod actually available
```

### Q: Tại sao cần `minReadySeconds`?

Không có `minReadySeconds`, Kubernetes chỉ chờ Pod đạt trạng thái `Ready` (readinessProbe pass) rồi tiếp tục xoá Pod cũ. Nhưng `Ready` không đồng nghĩa ứng dụng đã hoàn toàn ổn định — có thể vừa start xong và chưa warm up cache. `minReadySeconds` thêm khoảng đệm thời gian để ứng dụng thực sự ổn định.

### Q: Deployment có thể deploy lên Kubernetes không khi không có Service?

Có, Deployment và Service là hai object độc lập. Deployment quản lý Pod, Service quản lý network access. Bạn có thể tạo Deployment mà không cần Service (ví dụ: worker không cần expose port). Chỉ cần Service khi Pod cần nhận traffic từ bên ngoài hoặc từ Pod khác.

---

**Xem Tiếp:** [statefulset.md](./statefulset.md) — StatefulSet cho Workload Có Trạng Thái
