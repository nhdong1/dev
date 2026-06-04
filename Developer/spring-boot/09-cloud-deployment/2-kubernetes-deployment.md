# Kubernetes Deployment — Triển Khai Spring Boot Lên K8s

> Hướng dẫn toàn diện về triển khai ứng dụng Spring Boot lên Kubernetes: viết manifest YAML, cấu hình Service, quản lý cấu hình với ConfigMap/Secret, thiết lập HPA (Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang) và cấu hình Probes để đảm bảo high availability (tính khả dụng cao).

---

## 1. Các Khái Niệm Nền Tảng K8s

```
Cluster (Cụm)
├── Node (Máy chủ — vật lý hoặc VM)
│   ├── Pod (Đơn vị triển khai nhỏ nhất)
│   │   ├── Container 1 (Spring Boot App)
│   │   └── Container 2 (Sidecar — ví dụ: logging agent)
│   └── Pod ...
└── Node ...

Control Plane (Mặt Phẳng Điều Khiển):
├── API Server — cổng vào của cluster
├── etcd — key-value store lưu trạng thái cluster
├── Scheduler — lên lịch Pod lên Node
└── Controller Manager — duy trì trạng thái mong muốn
```

### Các Tài Nguyên K8s Cần Biết

| Tài Nguyên | Mục Đích |
|-----------|----------|
| **Pod** | Nhóm container chia sẻ network và storage |
| **Deployment** | Quản lý Pod, rolling update, rollback |
| **ReplicaSet** | Đảm bảo số lượng Pod mong muốn |
| **Service** | Load balancing và service discovery nội bộ |
| **ConfigMap** | Lưu cấu hình không nhạy cảm dạng key-value |
| **Secret** | Lưu dữ liệu nhạy cảm (mã hóa base64) |
| **Ingress** | HTTP/HTTPS routing từ bên ngoài vào cluster |
| **HPA** | HPA — Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang |
| **PDB** | PDB — PodDisruptionBudget — Ngân Sách Gián Đoạn Pod |
| **Namespace** | Phân tách logic trong cluster |

---

## 2. Namespace — Phân Vùng Logic

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
    team: backend
```

```bash
kubectl apply -f namespace.yaml
kubectl get namespaces
```

---

## 3. ConfigMap — Cấu Hình Không Nhạy Cảm

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: production
data:
  # Các key dạng flat (phẳng)
  APP_PORT: "8080"
  SPRING_PROFILES_ACTIVE: "production"
  LOG_LEVEL: "INFO"
  
  # File cấu hình nhúng (inline file)
  application.properties: |
    server.port=8080
    spring.datasource.url=jdbc:postgresql://postgres-service:5432/myapp
    spring.cache.type=redis
    spring.data.redis.host=redis-service
    management.endpoints.web.exposure.include=health,info,metrics,prometheus
    management.endpoint.health.show-details=always
```

```bash
kubectl apply -f configmap.yaml -n production

# Xem ConfigMap
kubectl get configmap myapp-config -n production -o yaml

# Cập nhật ConfigMap
kubectl edit configmap myapp-config -n production
```

---

## 4. Secret — Dữ Liệu Nhạy Cảm

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: production
type: Opaque
# Giá trị phải được mã hóa base64
# echo -n "mypassword" | base64
data:
  DB_PASSWORD: bXlwYXNzd29yZA==
  JWT_SECRET: c2VjcmV0a2V5MTIzNDU2Nzg5MA==
  REDIS_PASSWORD: cmVkaXNwYXNz
```

> **Lưu ý:** Base64 không phải mã hóa bảo mật. Trong production, dùng External Secrets Operator (Toán Tử Secret Bên Ngoài) kết hợp HashiCorp Vault hoặc AWS Secrets Manager.

```bash
# Tạo Secret từ command line (không lộ trong shell history)
kubectl create secret generic myapp-secrets \
  --from-literal=DB_PASSWORD=mypassword \
  --from-literal=JWT_SECRET=secretkey123 \
  -n production

# Hoặc từ file
kubectl create secret generic myapp-secrets \
  --from-env-file=.env.production \
  -n production
```

---

## 5. Deployment — Triển Khai Ứng Dụng

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
    version: "1.0.0"
spec:
  # Số lượng Pod mong muốn
  replicas: 3
  
  # Selector — bộ chọn để tìm Pod thuộc Deployment này
  selector:
    matchLabels:
      app: myapp
  
  # Rolling Update Strategy — Chiến Lược Cập Nhật Cuốn
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Tối đa thêm 1 Pod khi deploy (vượt quá replicas)
      maxUnavailable: 0    # 0 Pod bị down trong quá trình deploy
  
  template:
    metadata:
      labels:
        app: myapp
        version: "1.0.0"
    spec:
      # Graceful termination — Kết Thúc Mềm
      terminationGracePeriodSeconds: 60
      
      # Anti-affinity — chống đặt các Pod cùng Node
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - myapp
                topologyKey: kubernetes.io/hostname
      
      containers:
        - name: myapp
          image: registry.example.com/my-org/myapp:1.0.0
          imagePullPolicy: IfNotPresent
          
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
          
          # ──────────────────────────────
          # Environment Variables — Biến Môi Trường
          # ──────────────────────────────
          env:
            # Từ ConfigMap
            - name: SPRING_PROFILES_ACTIVE
              valueFrom:
                configMapKeyRef:
                  name: myapp-config
                  key: SPRING_PROFILES_ACTIVE
            
            # Từ Secret
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: DB_PASSWORD
            
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: JWT_SECRET
            
            # Pod metadata (downward API — API xuống dưới)
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          
          # Mount toàn bộ ConfigMap như file
          volumeMounts:
            - name: config-volume
              mountPath: /app/config
              readOnly: true
          
          # ──────────────────────────────
          # Resource Management — Quản Lý Tài Nguyên
          # ──────────────────────────────
          resources:
            requests:
              memory: "256Mi"  # RAM tối thiểu cần để schedule
              cpu: "250m"      # 250 millicores = 0.25 CPU core
            limits:
              memory: "512Mi"  # RAM tối đa (vượt = OOMKilled)
              cpu: "500m"      # CPU tối đa (vượt = throttled)
          
          # ──────────────────────────────
          # Probes — Kiểm Tra Sức Khỏe
          # ──────────────────────────────
          
          # Startup Probe — Kiểm Tra Khởi Động
          # Tắt liveness/readiness probe cho đến khi app khởi động xong
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 10   # Chờ 10s trước khi bắt đầu kiểm tra
            periodSeconds: 10         # Kiểm tra mỗi 10s
            failureThreshold: 30      # Cho phép thất bại 30 lần (= 5 phút tổng)
            successThreshold: 1
          
          # Liveness Probe — Kiểm Tra Còn Sống
          # Nếu fail → K8s restart Pod
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 0
            periodSeconds: 10
            failureThreshold: 3       # Fail 3 lần liên tiếp → restart
            successThreshold: 1
            timeoutSeconds: 5
          
          # Readiness Probe — Kiểm Tra Sẵn Sàng
          # Nếu fail → K8s bỏ Pod ra khỏi Service endpoints (không gửi traffic)
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 0
            periodSeconds: 5
            failureThreshold: 3
            successThreshold: 1
            timeoutSeconds: 3
      
      volumes:
        - name: config-volume
          configMap:
            name: myapp-config
      
      # Image Pull Secret nếu registry yêu cầu xác thực
      imagePullSecrets:
        - name: registry-credentials
```

---

## 6. Service — Mạng Nội Bộ

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: production
  labels:
    app: myapp
spec:
  # ClusterIP: chỉ accessible nội bộ cluster (default)
  # NodePort: expose ra node port (dev/test)
  # LoadBalancer: tạo external load balancer (cloud providers)
  type: ClusterIP
  
  selector:
    app: myapp    # Route traffic đến Pod có label app=myapp
  
  ports:
    - name: http
      port: 80          # Port của Service
      targetPort: 8080  # Port của container
      protocol: TCP
```

### Headless Service Cho StatefulSet

```yaml
# Headless Service — không có ClusterIP, DNS trực tiếp đến Pod
apiVersion: v1
kind: Service
metadata:
  name: myapp-headless
  namespace: production
spec:
  clusterIP: None    # Headless
  selector:
    app: myapp
  ports:
    - port: 8080
```

---

## 7. Ingress — HTTP Routing Từ Bên Ngoài

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    # Annotations cho nginx-ingress-controller
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    # Rate limiting — Giới Hạn Tốc Độ
    nginx.ingress.kubernetes.io/limit-rps: "100"
    # TLS redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # cert-manager tự động cấp TLS certificate
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  
  tls:
    - hosts:
        - api.example.com
      secretName: myapp-tls-cert
  
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /api/v1
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
```

---

## 8. HPA — Horizontal Pod Autoscaler (Tự Động Mở Rộng Pod Theo Chiều Ngang)

HPA tự động tăng/giảm số Pod dựa trên CPU, memory, hoặc custom metrics (chỉ số tùy chỉnh).

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  
  minReplicas: 2    # Tối thiểu 2 Pod (high availability)
  maxReplicas: 10   # Tối đa 10 Pod
  
  metrics:
    # Scale theo CPU
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # Scale up khi CPU > 70%
    
    # Scale theo memory
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80   # Scale up khi memory > 80%
    
    # Scale theo custom metric (ví dụ: request per second)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"      # 100 req/s per Pod
  
  # Hành vi scale — điều chỉnh tốc độ scale
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30    # Chờ 30s trước khi scale up lại
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60             # Tối đa scale up 2 Pod mỗi 60s
    scaleDown:
      stabilizationWindowSeconds: 300   # Chờ 5 phút trước khi scale down
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60             # Scale down tối đa 25% mỗi 60s
```

```bash
# Xem trạng thái HPA
kubectl get hpa -n production
kubectl describe hpa myapp-hpa -n production

# Kết quả mẫu:
# NAME        REFERENCE          TARGETS         MINPODS   MAXPODS   REPLICAS
# myapp-hpa   Deployment/myapp   45%/70%         2         10        3
```

---

## 9. PodDisruptionBudget — Ngân Sách Gián Đoạn Pod

PDB — PodDisruptionBudget — đảm bảo một số lượng Pod tối thiểu luôn chạy trong quá trình drain node hoặc rolling update.

```yaml
# pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  # Đảm bảo ít nhất 2 Pod luôn Available
  minAvailable: 2
  # Hoặc: cho phép tối đa 1 Pod gián đoạn
  # maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

---

## 10. Liveness vs Readiness vs Startup Probes

### So Sánh Ba Loại Probe

| Probe | Khi Fail | Mục Đích |
|-------|---------|----------|
| **startupProbe** | Tiếp tục thử, không restart ngay | Chờ app khởi động xong |
| **livenessProbe** | Restart Pod | Phát hiện app bị deadlock/hang |
| **readinessProbe** | Bỏ Pod khỏi Service endpoints | Chờ app sẵn sàng nhận traffic |

### Cấu Hình Spring Boot Actuator Cho Probes

```properties
# application.properties
management.health.livenessState.enabled=true
management.health.readinessState.enabled=true

# Expose endpoints
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.health.show-details=always
management.endpoint.health.probes.enabled=true
```

```bash
# Kiểm tra liveness
curl http://localhost:8080/actuator/health/liveness
# {"status":"UP"}

# Kiểm tra readiness  
curl http://localhost:8080/actuator/health/readiness
# {"status":"UP","components":{"db":{"status":"UP"},"redis":{"status":"UP"}}}
```

### Graceful Shutdown Khi Pod Bị Xóa

```
1. K8s gửi SIGTERM đến container
2. K8s bỏ Pod khỏi Service endpoints (không nhận traffic mới)
3. Spring Boot nhận SIGTERM → bắt đầu graceful shutdown
4. Chờ các request hiện tại hoàn thành (tối đa 30s)
5. Spring Boot đóng ứng dụng
6. K8s gửi SIGKILL nếu quá terminationGracePeriodSeconds
```

```properties
# Bật graceful shutdown
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

---

## 11. Rolling Update & Rollback

### Rolling Update — Cập Nhật Cuốn

```bash
# Update image của deployment
kubectl set image deployment/myapp \
  myapp=registry.example.com/my-org/myapp:1.1.0 \
  -n production

# Theo dõi tiến trình
kubectl rollout status deployment/myapp -n production

# Xem lịch sử rollout
kubectl rollout history deployment/myapp -n production
```

### Rollback — Quay Lại Phiên Bản Trước

```bash
# Rollback về revision trước đó
kubectl rollout undo deployment/myapp -n production

# Rollback về revision cụ thể
kubectl rollout undo deployment/myapp --to-revision=2 -n production

# Kiểm tra trạng thái sau rollback
kubectl get pods -n production -l app=myapp
```

---

## 12. Các Lệnh kubectl Thiết Yếu

```bash
# ─── Xem trạng thái ───
kubectl get pods -n production
kubectl get pods -n production -o wide           # Thêm thông tin Node, IP
kubectl describe pod myapp-xxx -n production     # Chi tiết Pod + Events
kubectl logs myapp-xxx -n production             # Xem logs
kubectl logs myapp-xxx -n production -f          # Theo dõi logs real-time
kubectl logs myapp-xxx -n production --previous  # Logs của container đã restart

# ─── Debug ───
kubectl exec -it myapp-xxx -n production -- /bin/sh  # Vào bên trong container
kubectl port-forward pod/myapp-xxx 8080:8080 -n production  # Port forwarding

# ─── Scale thủ công ───
kubectl scale deployment myapp --replicas=5 -n production

# ─── Apply manifests ───
kubectl apply -f k8s/ -n production              # Apply toàn bộ folder
kubectl diff -f k8s/ -n production              # Xem thay đổi trước khi apply
kubectl delete -f k8s/ -n production            # Xóa tài nguyên

# ─── Kiểm tra events ───
kubectl get events -n production --sort-by='.lastTimestamp'
```

---

## 13. Cấu Trúc Thư Mục K8s Tiêu Chuẩn

```
k8s/
├── base/                          # Cấu hình gốc (dùng Kustomize)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── development/
│   │   ├── kustomization.yaml     # Patch cho dev (replicas=1)
│   │   └── patch-replicas.yaml
│   ├── staging/
│   │   └── kustomization.yaml
│   └── production/
│       ├── kustomization.yaml     # Patch cho prod (replicas=3, resources)
│       ├── hpa.yaml
│       └── pdb.yaml
└── monitoring/
    ├── servicemonitor.yaml        # Prometheus ServiceMonitor
    └── grafana-dashboard.yaml
```

### Kustomize — Tùy Chỉnh Manifest

```yaml
# k8s/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production
namePrefix: prod-

bases:
  - ../../base

patches:
  - patch-replicas.yaml

images:
  - name: registry.example.com/my-org/myapp
    newTag: "1.0.0"

resources:
  - hpa.yaml
  - pdb.yaml
```

```bash
# Apply với Kustomize
kubectl apply -k k8s/overlays/production/
```

---

## 14. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Sự khác biệt giữa liveness và readiness probe?**

A: `livenessProbe` kiểm tra ứng dụng còn sống không — nếu fail, K8s restart Pod. `readinessProbe` kiểm tra ứng dụng sẵn sàng nhận traffic không — nếu fail, K8s bỏ Pod khỏi Service endpoints nhưng không restart. Ví dụ: ứng dụng đang khởi tạo kết nối DB → readiness fail (không nhận traffic), nhưng liveness vẫn pass (không cần restart).

**Q: ConfigMap vs Secret khác nhau thế nào?**

A: Cả hai đều lưu key-value, nhưng Secret dành cho dữ liệu nhạy cảm: được encode base64, có thể encrypt at rest (mã hóa khi lưu trữ) và kiểm soát RBAC riêng biệt. ConfigMap cho cấu hình thông thường. Tuy nhiên, base64 không phải mã hóa bảo mật — production nên dùng External Secrets Operator với Vault.

**Q: HPA hoạt động thế nào?**

A: HPA — Horizontal Pod Autoscaler — poll metrics từ Metrics Server mỗi 15s. Khi CPU/memory vượt ngưỡng target, HPA tính toán số Pod mới = `ceil(current × currentMetric / targetMetric)`. ScaleDown có stabilization window (cửa sổ ổn định) để tránh flapping (dao động liên tục).

**Q: Làm thế nào để đảm bảo zero-downtime deployment?**

A: (1) `maxUnavailable: 0` trong RollingUpdate strategy, (2) `readinessProbe` đúng — K8s chỉ route traffic khi Pod sẵn sàng, (3) `graceful shutdown` với `server.shutdown=graceful`, (4) PodDisruptionBudget đảm bảo minimum availability trong quá trình drain node.

---

## ✅ Checklist

- [ ] Khai báo `resources.requests` và `resources.limits` cho mọi container
- [ ] Cấu hình đủ ba loại probe: startup, liveness, readiness
- [ ] Dùng `/actuator/health/liveness` và `/actuator/health/readiness`
- [ ] Bật `server.shutdown=graceful` và `terminationGracePeriodSeconds` hợp lý
- [ ] HPA với `minReplicas >= 2` cho high availability
- [ ] PodDisruptionBudget để bảo vệ khi drain node
- [ ] ConfigMap cho config, Secret cho credentials
- [ ] Không hardcode image tag `latest` trong Deployment
- [ ] Pod Anti-Affinity để tránh tất cả Pod cùng 1 Node
- [ ] Ingress với TLS termination cho production
