# Kubernetes Deployment cho .NET — K8s, HPA, ConfigMap, Secret

> Kubernetes — K8s — là hệ thống điều phối container — container orchestration mã nguồn mở, tự động hóa việc triển khai, mở rộng và vận hành ứng dụng container. Bài này hướng dẫn triển khai .NET app lên Kubernetes với các tài nguyên cần thiết: Deployment, Service, ConfigMap, Secret, Ingress và HPA — Horizontal Pod Autoscaler.

---

## 1. Các Khái Niệm Cơ Bản

```
┌──────────────────────────────────────────────────────────────┐
│                      KUBERNETES CLUSTER                       │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    NAMESPACE: production              │   │
│  │                                                      │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │              DEPLOYMENT: myapp-api             │  │   │
│  │  │                                                │  │   │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐    │  │   │
│  │  │  │  POD 1   │  │  POD 2   │  │  POD 3   │    │  │   │
│  │  │  │ container│  │ container│  │ container│    │  │   │
│  │  │  │ :8080    │  │ :8080    │  │ :8080    │    │  │   │
│  │  │  └──────────┘  └──────────┘  └──────────┘    │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  │                          │                            │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │         SERVICE: myapp-api-svc (ClusterIP)     │  │   │
│  │  │         Load balances across pods              │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  │                          │                            │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │              INGRESS: myapp-ingress             │  │   │
│  │  │         api.myapp.com → myapp-api-svc          │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### Thuật Ngữ Quan Trọng

| Thuật Ngữ | Giải Thích |
|-----------|-----------|
| **Pod** | Đơn vị nhỏ nhất trong K8s; chứa một hoặc nhiều containers |
| **Deployment** | Quản lý lifecycle — vòng đời của Pod replicas |
| **ReplicaSet** | Đảm bảo số lượng Pod replicas — bản sao luôn đúng |
| **Service** | Load balancer nội bộ, stable endpoint cho Pods |
| **Ingress** | HTTP/HTTPS routing từ bên ngoài vào Services |
| **ConfigMap** | Lưu cấu hình non-sensitive — không nhạy cảm dưới dạng key-value |
| **Secret** | Lưu dữ liệu nhạy cảm (password, token) dưới dạng base64 |
| **Namespace** | Phân vùng logic trong cluster — cụm |
| **Node** | Máy chủ vật lý hoặc VM tạo nên cluster |
| **HPA** | Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang |
| **PVC** | Persistent Volume Claim — Yêu Cầu Lưu Trữ Bền Vững |

---

## 2. Namespace — Phân Vùng Logic

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: backend
```

```bash
kubectl apply -f namespace.yaml
kubectl get namespaces
```

---

## 3. Deployment — Triển Khai Ứng Dụng

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
  namespace: production
  labels:
    app: myapp-api
    version: "1.0.0"
spec:
  replicas: 3                # số Pod chạy đồng thời
  
  selector:
    matchLabels:
      app: myapp-api         # phải khớp với labels của Pod template
  
  strategy:
    type: RollingUpdate       # RollingUpdate hoặc Recreate
    rollingUpdate:
      maxSurge: 1             # tối đa thêm 1 Pod mới khi update
      maxUnavailable: 0       # không để Pod nào unavailable khi update
  
  template:
    metadata:
      labels:
        app: myapp-api
    spec:
      # Đặt các Pod trên các Node khác nhau (tăng availability)
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: myapp-api

      containers:
        - name: myapp-api
          image: myacr.azurecr.io/myapp-api:1.0.0
          imagePullPolicy: Always
          
          ports:
            - containerPort: 8080
              name: http
          
          # ─── Environment Variables từ ConfigMap ─────────────────
          envFrom:
            - configMapRef:
                name: myapp-config
          
          # ─── Secrets riêng lẻ ───────────────────────────────────
          env:
            - name: ConnectionStrings__Default
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: db-connection-string
            - name: Redis__ConnectionString
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: redis-connection-string
          
          # ─── Resource Requests và Limits ────────────────────────
          # request: K8s scheduler dùng để đặt Pod lên Node phù hợp
          # limit: Pod bị kill nếu vượt memory limit
          resources:
            requests:
              cpu: "250m"        # 250 millicores = 0.25 CPU core
              memory: "256Mi"    # 256 Mebibytes
            limits:
              cpu: "1000m"       # 1 CPU core tối đa
              memory: "512Mi"    # 512 MiB tối đa
          
          # ─── Liveness Probe — Kiểm Tra Sự Sống ─────────────────
          # Nếu fail: K8s restart container
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30   # chờ 30s trước khi bắt đầu probe
            periodSeconds: 10         # probe mỗi 10s
            failureThreshold: 3       # restart sau 3 lần fail liên tiếp
            timeoutSeconds: 5
          
          # ─── Readiness Probe — Kiểm Tra Sẵn Sàng ───────────────
          # Nếu fail: K8s không gửi traffic vào Pod này
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
            timeoutSeconds: 3
          
          # ─── Startup Probe — Kiểm Tra Khởi Động ────────────────
          # Dành cho app khởi động chậm
          startupProbe:
            httpGet:
              path: /health/live
              port: 8080
            failureThreshold: 30   # thử tối đa 30 lần
            periodSeconds: 10      # mỗi 10s → tổng 300s cho app khởi động

      # Pull image từ private registry — registry riêng tư
      imagePullSecrets:
        - name: acr-pull-secret
      
      # Graceful shutdown — tắt nhẹ nhàng
      terminationGracePeriodSeconds: 30
```

---

## 4. ConfigMap — Cấu Hình Không Nhạy Cảm

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: production
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  ASPNETCORE_URLS: "http://+:8080"
  Logging__LogLevel__Default: "Information"
  Logging__LogLevel__Microsoft.AspNetCore: "Warning"
  Cache__DefaultExpiryMinutes: "60"
  
  # File config có thể mount như file
  appsettings.Production.json: |
    {
      "FeatureFlags": {
        "NewCheckout": true,
        "BetaSearch": false
      }
    }
```

### Mount ConfigMap như File

```yaml
# Trong spec.containers
volumeMounts:
  - name: config-volume
    mountPath: /app/config
    readOnly: true

# Trong spec.volumes
volumes:
  - name: config-volume
    configMap:
      name: myapp-config
      items:
        - key: appsettings.Production.json
          path: appsettings.Production.json
```

---

## 5. Secret — Dữ Liệu Nhạy Cảm

```yaml
# secret.yaml — dữ liệu phải được base64 encode
# KHÔNG commit file này vào git!
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: production
type: Opaque
data:
  # echo -n "Server=prod..." | base64
  db-connection-string: U2VydmVyPXByb2Qtc2VydmVyOw==
  redis-connection-string: cHJvZC1yZWRpczozNzk=
  jwt-secret-key: c3VwZXItc2VjcmV0LWtleS10aGF0LWlzLXZlcnktbG9uZw==
```

```bash
# Tạo secret trực tiếp từ command line (không cần base64 thủ công)
kubectl create secret generic myapp-secrets \
    --from-literal=db-connection-string="Server=prod-server;Database=myapp;..." \
    --from-literal=redis-connection-string="prod-redis:6379" \
    --from-literal=jwt-secret-key="super-secret-key" \
    --namespace=production

# Tạo secret từ file
kubectl create secret generic acr-pull-secret \
    --from-file=.dockerconfigjson=$HOME/.docker/config.json \
    --type=kubernetes.io/dockerconfigjson \
    --namespace=production
```

> ⚠️ **Lưu ý:** Kubernetes Secret mặc định chỉ được base64 encode, không phải mã hóa. Cần kết hợp với Azure Key Vault hoặc HashiCorp Vault để bảo mật thực sự.

---

## 6. Service — Expose Pods

### ClusterIP — Chỉ Truy Cập Nội Bộ (Mặc Định)

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-api-svc
  namespace: production
spec:
  type: ClusterIP         # chỉ accessible trong cluster
  selector:
    app: myapp-api        # route traffic đến Pods có label này
  ports:
    - name: http
      port: 80            # port của Service
      targetPort: 8080    # port của container
      protocol: TCP
```

### LoadBalancer — Expose ra Internet

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-api-lb
  namespace: production
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"  # internal LB
spec:
  type: LoadBalancer
  selector:
    app: myapp-api
  ports:
    - port: 80
      targetPort: 8080
```

---

## 7. Ingress — HTTP Routing

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    # Rate limiting — giới hạn tốc độ
    nginx.ingress.kubernetes.io/limit-rps: "100"
    # TLS certificate tự động qua cert-manager
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - api.myapp.com
      secretName: myapp-tls-secret
  
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /api/v1
            pathType: Prefix
            backend:
              service:
                name: myapp-api-svc
                port:
                  number: 80
          - path: /health
            pathType: Prefix
            backend:
              service:
                name: myapp-api-svc
                port:
                  number: 80
```

---

## 8. HPA — Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang

HPA tự động tăng/giảm số lượng Pod dựa trên CPU, memory hoặc custom metrics — chỉ số tùy chỉnh.

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-api
  
  minReplicas: 2    # tối thiểu 2 Pods (cho HA — high availability)
  maxReplicas: 20   # tối đa 20 Pods
  
  metrics:
    # Scale dựa trên CPU
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # scale up khi CPU > 70%
    
    # Scale dựa trên Memory
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80    # scale up khi Memory > 80%
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # chờ 60s trước khi scale up lần nữa
      policies:
        - type: Pods
          value: 4                       # tăng tối đa 4 Pods mỗi lần
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300   # chờ 300s (5 phút) trước khi scale down
      policies:
        - type: Pods
          value: 2                       # giảm tối đa 2 Pods mỗi lần
          periodSeconds: 60
```

---

## 9. Liveness vs Readiness vs Startup Probe

```
┌─────────────────────────────────────────────────────────────┐
│                    VÒNG ĐỜI POD                             │
│                                                             │
│  Container Start                                            │
│       │                                                     │
│       ▼                                                     │
│  ┌──────────────────────────────────────┐                  │
│  │   Startup Probe                       │                  │
│  │   Kiểm tra: app đã khởi xong chưa?   │                  │
│  │   Nếu fail sau N lần: restart         │                  │
│  └──────────────────────────────────────┘                  │
│       │ (pass)                                              │
│       ▼                                                     │
│  ┌──────────────────────────────────────┐                  │
│  │   Liveness Probe (liên tục)           │                  │
│  │   Kiểm tra: app còn sống không?       │                  │
│  │   Nếu fail: K8s restart container     │                  │
│  └──────────────────────────────────────┘                  │
│       │                                                     │
│  ┌──────────────────────────────────────┐                  │
│  │   Readiness Probe (liên tục)          │                  │
│  │   Kiểm tra: app có sẵn sàng nhận     │                  │
│  │   traffic không? (DB connected, etc.) │                  │
│  │   Nếu fail: tạm ngừng gửi traffic    │                  │
│  └──────────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

### Cài Health Checks trong ASP.NET Core

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    // Kiểm tra database connection
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("Default")!,
        name: "database",
        tags: ["ready"])
    // Kiểm tra Redis
    .AddRedis(
        redisConnectionString: builder.Configuration["Redis:ConnectionString"]!,
        name: "redis",
        tags: ["ready"])
    // Custom health check
    .AddCheck<ExternalApiHealthCheck>("external-api", tags: ["ready"]);

// Liveness: app đang chạy bình thường (không cần check dependencies)
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false,  // không chạy check nào → luôn healthy nếu app còn sống
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

// Readiness: app sẵn sàng nhận traffic (phải check dependencies)
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

---

## 10. Rolling Update vs Blue-Green vs Canary

### Rolling Update — Cập Nhật Cuốn Chiếu (Mặc Định K8s)

```
v1 v1 v1 v1 v1
      ↓ (maxSurge=1, maxUnavailable=0)
v2 v1 v1 v1 v1   → xong pod 1
v2 v2 v1 v1 v1   → xong pod 2
v2 v2 v2 v1 v1   → xong pod 3
v2 v2 v2 v2 v1   → xong pod 4
v2 v2 v2 v2 v2   → hoàn tất
```

**Ưu điểm:** Zero-downtime, dùng ít tài nguyên hơn Blue-Green.
**Nhược điểm:** Tại một thời điểm có cả v1 và v2 chạy — phải đảm bảo backward compatible.

### Blue-Green Deployment

```bash
# Blue (v1) đang chạy, Green (v2) được deploy song song
# Sau khi test Green OK, switch Service sang Green

kubectl patch service myapp-api-svc \
    -p '{"spec":{"selector":{"version":"v2"}}}'

# Rollback: switch lại Blue nếu có vấn đề
kubectl patch service myapp-api-svc \
    -p '{"spec":{"selector":{"version":"v1"}}}'
```

### Canary Deployment — Triển Khai Thí Điểm

```yaml
# Stable (v1): 9 replicas
# Canary (v2): 1 replica → ~10% traffic nhận v2
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp-api
      track: canary
  template:
    metadata:
      labels:
        app: myapp-api   # Service dùng label này để route traffic
        track: canary
    spec:
      containers:
        - name: myapp-api
          image: myacr.azurecr.io/myapp-api:2.0.0
```

---

## 11. Resource Quotas — Hạn Ngạch Tài Nguyên

```yaml
# resource-quota.yaml — giới hạn tài nguyên cho namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"          # tổng CPU request tối đa
    requests.memory: "20Gi"     # tổng Memory request tối đa
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"                  # tối đa 50 pods
    services: "10"
    secrets: "20"
    configmaps: "20"
```

---

## 12. PodDisruptionBudget — Ngân Sách Gián Đoạn Pod

```yaml
# pdb.yaml — đảm bảo luôn có ít nhất N pods healthy khi maintenance
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-api-pdb
  namespace: production
spec:
  minAvailable: 2    # luôn giữ ít nhất 2 pods available
  selector:
    matchLabels:
      app: myapp-api
```

---

## 13. Kubectl Lệnh Thực Chiến

```bash
# Deploy / apply config
kubectl apply -f deployment.yaml
kubectl apply -f . # apply tất cả yaml trong thư mục

# Xem trạng thái
kubectl get pods -n production
kubectl get deployments -n production
kubectl describe pod myapp-api-xxx -n production

# Xem logs
kubectl logs myapp-api-xxx -n production
kubectl logs myapp-api-xxx -n production -f          # follow/stream logs
kubectl logs myapp-api-xxx -n production --previous  # log của container trước đó

# Scale thủ công
kubectl scale deployment myapp-api --replicas=5 -n production

# Rolling update image
kubectl set image deployment/myapp-api \
    myapp-api=myacr.azurecr.io/myapp-api:1.1.0 \
    -n production

# Rollback deployment
kubectl rollout history deployment/myapp-api -n production
kubectl rollout undo deployment/myapp-api -n production
kubectl rollout undo deployment/myapp-api --to-revision=2 -n production

# Port-forward để debug local
kubectl port-forward pod/myapp-api-xxx 8080:8080 -n production

# Exec vào container để debug
kubectl exec -it myapp-api-xxx -n production -- /bin/bash

# Xem events
kubectl get events -n production --sort-by=.lastTimestamp
```

---

## Checklist Kubernetes .NET

- [ ] Deployment có `resources.requests` và `resources.limits` rõ ràng
- [ ] Liveness Probe và Readiness Probe đã cấu hình đúng
- [ ] Startup Probe cho app khởi động chậm (EF Core migrations, etc.)
- [ ] HPA cấu hình để auto-scale theo CPU/Memory
- [ ] PodDisruptionBudget đảm bảo HA khi maintenance
- [ ] Secrets không commit vào git — dùng kubectl create secret hoặc sealed-secrets
- [ ] `topologySpreadConstraints` để phân phối Pods đều trên Nodes
- [ ] `terminationGracePeriodSeconds` đủ dài để drain connections
- [ ] Image tag cụ thể, không dùng `latest`
- [ ] Namespace riêng cho mỗi môi trường

---

## Câu Hỏi Phỏng Vấn

**Q: Sự khác nhau giữa Deployment và StatefulSet?**
> Deployment dành cho stateless apps — ứng dụng không trạng thái (API servers). Pods có thể bị thay thế bất kỳ lúc nào, không cần persistent storage riêng. StatefulSet dành cho stateful apps — ứng dụng có trạng thái (databases, Kafka): mỗi Pod có stable network identity, persistent storage riêng, và được scale theo thứ tự.

**Q: Liveness Probe vs Readiness Probe — khác nhau chỗ nào?**
> Liveness: nếu fail → K8s restart container (app bị crash, deadlock). Readiness: nếu fail → K8s ngừng gửi traffic vào Pod nhưng không restart (app đang load, DB chưa kết nối). Startup Probe: chạy trước cả hai, cho app có thời gian khởi động mà không bị restart sớm.

**Q: HPA hoạt động như thế nào?**
> HPA — Horizontal Pod Autoscaler định kỳ (mặc định 15s) query Metrics Server để lấy CPU/Memory usage của các Pods. Nếu average utilization vượt target, K8s tính toán số replicas cần thiết và update Deployment. Có stabilization window để tránh scale up/down liên tục (flapping).

**Cập Nhật Lần Cuối:** 2026-06-02
