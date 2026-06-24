# Kubernetes Deployment — Deployment, Service, HPA và Probes

> Kubernetes (K8s — Hệ Thống Điều Phối Container) là nền tảng orchestration (điều phối) phổ biến để chạy Node.js app ở quy mô production — tự động scale, self-healing (tự phục hồi), và rolling updates (cập nhật lăn).

## Mục Lục

1. [Kubernetes Concepts Cho Node.js](#kubernetes-concepts-cho-nodejs)
2. [Deployment Manifest](#deployment-manifest)
3. [Service — Expose Application](#service--expose-application)
4. [ConfigMap và Secret](#configmap-và-secret)
5. [Health Probes — Liveness, Readiness, Startup](#health-probes--liveness-readiness-startup)
6. [HPA — Horizontal Pod Autoscaler](#hpa--horizontal-pod-autoscaler)
7. [Resource Limits và Requests](#resource-limits-và-requests)
8. [Rolling Update Strategy](#rolling-update-strategy)
9. [Ingress — External Traffic](#ingress--external-traffic)
10. [Best Practices](#best-practices)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kubernetes Concepts Cho Node.js

```
┌─────────────────────────────────────────────────────────────┐
│                      KUBERNETES CLUSTER                      │
│                                                             │
│  ┌─────────────┐     ┌─────────────────────────────────┐   │
│  │   Ingress   │────►│  Service (ClusterIP/LoadBalancer)│   │
│  │  Controller │     └──────────────┬──────────────────┘   │
│  └─────────────┘                    │                       │
│                          ┌──────────┼──────────┐            │
│                          ▼          ▼          ▼            │
│                     ┌────────┐ ┌────────┐ ┌────────┐       │
│                     │ Pod 1  │ │ Pod 2  │ │ Pod 3  │       │
│                     │ Node.js│ │ Node.js│ │ Node.js│       │
│                     └────────┘ └────────┘ └────────┘       │
│                          ▲          ▲          ▲            │
│                          └──────────┼──────────┘            │
│                                     │                       │
│                              Deployment                     │
│                          (desired state: 3 replicas)        │
│                                     │                       │
│                              HPA (auto-scale 2-10)          │
└─────────────────────────────────────────────────────────────┘
```

| Resource | Vai Trò |
| -------- | ------- |
| **Pod** | Đơn vị nhỏ nhất — một hoặc nhiều containers |
| **Deployment** | Quản lý desired state, rolling updates, rollback |
| **Service** | Stable network endpoint cho pods (load balancing) |
| **ConfigMap** | Non-sensitive configuration |
| **Secret** | Sensitive data (credentials, tokens) |
| **Ingress** | HTTP routing, TLS termination |
| **HPA** | Tự động scale pods theo CPU/memory/custom metrics |

---

## Deployment Manifest

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-api
  labels:
    app: nodejs-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nodejs-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Thêm tối đa 1 pod mới trong khi update
      maxUnavailable: 0  # Không cho phép pod unavailable
  template:
    metadata:
      labels:
        app: nodejs-api
    spec:
      containers:
        - name: api
          image: ghcr.io/myorg/nodejs-api:abc1234
          ports:
            - containerPort: 3000
              protocol: TCP
          envFrom:
            - configMapRef:
                name: nodejs-api-config
            - secretRef:
                name: nodejs-api-secrets
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 30   # 30 × 5s = 150s để start
      terminationGracePeriodSeconds: 30
```

```bash
kubectl apply -f k8s/
kubectl get pods -l app=nodejs-api
kubectl rollout status deployment/nodejs-api
```

---

## Service — Expose Application

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nodejs-api
  labels:
    app: nodejs-api
spec:
  type: ClusterIP          # Internal only
  # type: LoadBalancer     # Cloud LB (AWS ELB, GCP LB)
  selector:
    app: nodejs-api
  ports:
    - port: 80
      targetPort: 3000
      protocol: TCP
```

| Service Type | Use Case |
| ------------ | -------- |
| **ClusterIP** | Internal communication giữa services |
| **NodePort** | Expose trên mỗi node IP (dev/testing) |
| **LoadBalancer** | External traffic qua cloud load balancer |
| **Headless** | Direct pod DNS (stateful, service mesh) |

---

## ConfigMap và Secret

### ConfigMap — Non-sensitive Config

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nodejs-api-config
data:
  NODE_ENV: "production"
  PORT: "3000"
  LOG_LEVEL: "info"
  REDIS_HOST: "redis-service"
  REDIS_PORT: "6379"
```

### Secret — Sensitive Data

```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: nodejs-api-secrets
type: Opaque
stringData:
  DATABASE_URL: "postgresql://user:pass@db-service:5432/mydb"
  JWT_SECRET: "your-jwt-secret-here"
```

```bash
# Tạo secret từ command line (không commit vào git)
kubectl create secret generic nodejs-api-secrets \
  --from-literal=DATABASE_URL="postgresql://..." \
  --from-literal=JWT_SECRET="..."
```

**Quan trọng:** Không commit Secret manifests vào git. Dùng Sealed Secrets, External Secrets Operator, hoặc cloud secret manager.

---

## Health Probes — Liveness, Readiness, Startup

```
Pod Lifecycle với Probes:

  Container Start
       │
       ▼
  ┌─────────────┐
  │ Startup     │ ── Fail 30 lần → Kill & Restart
  │ Probe       │
  └──────┬──────┘
         │ Pass
         ▼
  ┌─────────────┐     ┌─────────────┐
  │ Liveness    │     │ Readiness   │
  │ Probe       │     │ Probe       │
  │             │     │             │
  │ Fail →      │     │ Fail →      │
  │ Restart Pod │     │ Remove from │
  │             │     │ Service LB  │
  └─────────────┘     └─────────────┘
```

| Probe | Mục Đích | Fail Action |
| ----- | -------- | ----------- |
| **Startup** | App khởi động chậm (DB migration, cache warm) | Restart container |
| **Liveness** | Process còn sống, không deadlock | Restart container |
| **Readiness** | Sẵn sàng nhận traffic | Remove khỏi Service endpoints |

### Implementation trong Node.js

```typescript
let isReady = false;

app.get('/health', (_req, res) => {
  res.status(200).json({ status: 'ok' });
});

app.get('/ready', async (_req, res) => {
  if (!isReady) {
    return res.status(503).json({ status: 'starting' });
  }
  try {
    await db.ping();
    res.status(200).json({ status: 'ready' });
  } catch {
    res.status(503).json({ status: 'not ready' });
  }
});

// Sau khi DB connected và app warmed up
async function bootstrap() {
  await db.connect();
  await cache.warmup();
  isReady = true;
  server.listen(PORT);
}
```

---

## HPA — Horizontal Pod Autoscaler

HPA (Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang) tự động tăng/giảm số replicas dựa trên metrics.

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nodejs-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nodejs-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
```

### Custom Metrics (Request Rate)

```yaml
metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
```

Cần Prometheus Adapter hoặc KEDA để scale theo custom metrics.

---

## Resource Limits và Requests

```
Node Resources:
┌────────────────────────────────────────┐
│  CPU: 4 cores    Memory: 8GB           │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐  │
│  │Pod 1 │ │Pod 2 │ │Pod 3 │ │ free │  │
│  │100m  │ │100m  │ │100m  │ │      │  │
│  │256Mi │ │256Mi │ │256Mi │ │      │  │
│  └──────┘ └──────┘ └──────┘ └──────┘  │
└────────────────────────────────────────┘

requests = guaranteed minimum
limits   = maximum allowed (OOM kill nếu vượt memory limit)
```

| Setting | Node.js Khuyến Nghị | Lý Do |
| ------- | ------------------- | ----- |
| memory request | 256Mi | Base heap + dependencies |
| memory limit | 512Mi – 1Gi | Node.js heap grows — set `--max-old-space-size` |
| cpu request | 100m – 250m | I/O-bound, không cần nhiều CPU |
| cpu limit | 500m – 1000m | Tránh CPU throttling gây latency spike |

```dockerfile
# Set Node.js heap limit < container memory limit
ENV NODE_OPTIONS="--max-old-space-size=384"
```

---

## Rolling Update Strategy

```
Rolling Update (maxSurge: 1, maxUnavailable: 0):

v1: [Pod-A] [Pod-B] [Pod-C]     ← 3 pods running v1
         ↓ deploy v2
    [Pod-A] [Pod-B] [Pod-C] [Pod-D-v2]  ← surge: thêm 1 pod v2
         ↓ Pod-D ready
    [Pod-A-v1 terminated]
    [Pod-B] [Pod-C] [Pod-D]     ← 3 pods, mixed versions
         ↓ continue...
    [Pod-D] [Pod-E] [Pod-F]     ← all v2 ✅
```

```bash
# Deploy version mới
kubectl set image deployment/nodejs-api api=ghcr.io/myorg/nodejs-api:new-sha

# Theo dõi rollout
kubectl rollout status deployment/nodejs-api

# Rollback nếu có vấn đề
kubectl rollout undo deployment/nodejs-api

# Rollback về version cụ thể
kubectl rollout history deployment/nodejs-api
kubectl rollout undo deployment/nodejs-api --to-revision=3
```

---

## Ingress — External Traffic

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nodejs-api-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-cert
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nodejs-api
                port:
                  number: 80
```

---

## Best Practices

1. **Một process per pod** — không dùng PM2 trong container
2. **Immutable image tags** — dùng git SHA, không `latest`
3. **Set resource requests/limits** — tránh noisy neighbor và OOM
4. **Startup probe** cho app khởi động chậm — tránh liveness kill sớm
5. **PodDisruptionBudget** — đảm bảo min available pods khi node drain
6. **terminationGracePeriodSeconds: 30** — cho phép graceful shutdown
7. **Horizontal scaling** thay vì vertical — Node.js scale tốt theo replicas

```yaml
# PodDisruptionBudget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nodejs-api-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nodejs-api
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Đáp Án Ngắn |
| ------- | ----------- |
| Deployment vs Pod? | Pod = instance; Deployment quản lý desired replicas và updates |
| Liveness vs Readiness? | Liveness restart pod; Readiness remove khỏi load balancer |
| HPA scale theo gì? | CPU, memory, hoặc custom metrics (request rate, queue depth) |
| Rolling update vs Recreate? | Rolling: zero downtime; Recreate: stop all rồi start new |
| Tại sao set memory limit cho Node.js? | Tránh OOM kill node; kết hợp `--max-old-space-size` |
| ConfigMap vs Secret? | ConfigMap: non-sensitive; Secret: base64 encoded sensitive data |
