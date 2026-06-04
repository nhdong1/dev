# Thiết Kế Hệ Thống với Kubernetes — System Design

> Hướng dẫn thiết kế các hệ thống thực tế trên Kubernetes, bao gồm kiến trúc mẫu, quyết định trade-off, và cách trình bày trong phỏng vấn system design.

## Mục Lục

1. [Framework Trả Lời System Design](#framework-trả-lời-system-design)
2. [Case Study 1: Multi-tier Web Application](#case-study-1-multi-tier-web-application)
3. [Case Study 2: Microservices E-Commerce](#case-study-2-microservices-e-commerce)
4. [Case Study 3: Data Pipeline với Kubernetes](#case-study-3-data-pipeline-với-kubernetes)
5. [Case Study 4: Multi-region High Availability](#case-study-4-multi-region-high-availability)
6. [Quyết Định Trade-off Quan Trọng](#quyết-định-trade-off-quan-trọng)

---

## Framework Trả Lời System Design

### Quy Trình 4 Bước (RATS)

```
R — Requirements (Yêu Cầu):     Làm rõ yêu cầu trước khi thiết kế
A — Architecture (Kiến Trúc):   Vẽ sơ đồ high-level, giải thích thành phần
T — Trade-offs (Đánh Đổi):      Giải thích tại sao chọn giải pháp này, không phải giải pháp khác
S — Scale (Mở Rộng):            Hệ thống xử lý tải như thế nào khi lớn lên
```

### Câu Hỏi Cần Hỏi Trước Khi Thiết Kế

```
Scale:
- Bao nhiêu request/giây hiện tại và tối đa?
- Bao nhiêu user đồng thời?
- Dữ liệu có hot/cold pattern không?

Reliability:
- SLA (Service Level Agreement) là bao nhiêu? (99.9%, 99.99%?)
- RTO (Recovery Time Objective — Mục Tiêu Thời Gian Phục Hồi) là bao lâu?
- RPO (Recovery Point Objective — Mục Tiêu Điểm Phục Hồi) là bao nhiêu?

Constraints:
- On-premise hay cloud?
- Ngân sách? (ảnh hưởng lựa chọn instance type, region)
- Team size? (ảnh hưởng độ phức tạp cho phép)
```

---

## Case Study 1: Multi-tier Web Application

### Yêu Cầu

```
- Web app 3 tier: Frontend (React), Backend (Node.js), Database (PostgreSQL)
- 1.000 request/giây bình thường, peak 5.000 request/giây
- Uptime 99.9% (8.7 giờ downtime/năm cho phép)
- Deploy trên AWS EKS
```

### Kiến Trúc

```
Internet
    │
    ▼
[AWS ALB / Route53]
    │
    ▼
[Ingress Controller — NGINX]
    │
    ├──── /  ──────► [Frontend Service — ClusterIP]
    │                        │
    │                    [Frontend Pods x3]
    │                    React static served by nginx
    │
    └──── /api ────► [Backend Service — ClusterIP]
                             │
                         [Backend Pods x3-10]
                         Node.js + HPA
                             │
                    [PostgreSQL Service — ClusterIP]
                             │
                    [PostgreSQL StatefulSet x1]
                    (primary) + [Read Replica x2]
                             │
                    [PersistentVolumeClaim — EBS gp3]
```

### YAML Thiết Kế Chính

```yaml
# Namespace phân vùng
---
apiVersion: v1
kind: Namespace
metadata:
  name: web-app
  labels:
    pod-security.kubernetes.io/enforce: restricted

# Backend Deployment với HPA
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: backend
    spec:
      serviceAccountName: backend-sa
      affinity:
        podAntiAffinity:         # Spread ra nhiều node
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: backend
              topologyKey: kubernetes.io/hostname
      containers:
      - name: backend
        image: my-backend:v2.1.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
      terminationGracePeriodSeconds: 30

# HPA cho Backend
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: web-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 3
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Giảm chậm hơn tăng

# PodDisruptionBudget
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: backend-pdb
  namespace: web-app
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: backend
```

### Tính Toán Resource

```
Backend Request: 250m CPU, 256Mi RAM
Peak load 5.000 RPS: HPA scale lên 15 Pod
Total CPU request: 15 × 250m = 3.75 CPU
Total RAM request: 15 × 256Mi = 3.75 GB

Node type: t3.large (2 vCPU, 8GB RAM)
Số node cần: 3 node (với buffer 30%: ~4 node)
Chi phí ước tính: 4 × $0.083/h = $0.33/h = ~$240/tháng
```

---

## Case Study 2: Microservices E-Commerce

### Yêu Cầu

```
- Platform thương mại điện tử: order, inventory, payment, notification
- 10.000 đơn hàng/giờ, peak ngày sale 100.000 đơn/giờ (10x)
- Zero-downtime deployment
- Audit log đầy đủ cho mọi giao dịch
```

### Kiến Trúc Tổng Thể

```
                        [API Gateway — Kong/NGINX]
                                │
            ┌───────────────────┼───────────────────┐
            │                   │                   │
    [Order Service]    [Inventory Service]   [User Service]
            │                   │
            └──────► [Message Broker — Kafka]
                           │
            ┌──────────────┼──────────────┐
            │              │              │
    [Payment Service] [Notification]  [Analytics]
                      Service         (batch job)
            │
    [Payment Gateway — external]
```

### Thiết Kế Các Thành Phần K8s

**1. Service Mesh với Istio — tạo mTLS tự động:**
```yaml
# Namespace với Istio injection
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
  labels:
    istio-injection: enabled  # Tự động inject Envoy sidecar

# PeerAuthentication — bắt buộc mTLS trong namespace
---
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: require-mtls
  namespace: ecommerce
spec:
  mtls:
    mode: STRICT  # Từ chối HTTP không encrypted
```

**2. Circuit Breaker (Ngắt Mạch) với Istio:**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutiveGatewayErrors: 5   # Sau 5 lỗi liên tiếp
      interval: 30s
      baseEjectionTime: 30s         # Loại khỏi load balancer 30s
      maxEjectionPercent: 50        # Tối đa 50% Pod bị loại
```

**3. Kafka Deployment trên K8s (StatefulSet):**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: ecommerce
spec:
  serviceName: kafka
  replicas: 3
  selector:
    matchLabels:
      app: kafka
  template:
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:7.4.0
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
  volumeClaimTemplates:
  - metadata:
      name: kafka-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
```

**4. KEDA (Kubernetes Event-Driven Autoscaling) Scale theo Kafka lag:**
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 2
  maxReplicaCount: 50
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka.ecommerce.svc:9092
      consumerGroup: order-processor-group
      topic: orders
      lagThreshold: "100"   # Scale lên nếu lag > 100 messages/replica
```

### Quyết Định Trade-off

**Q: Tại sao dùng Kafka thay vì gọi trực tiếp giữa service?**

```
Direct call (synchronous):
  + Đơn giản hơn, dễ debug
  - Order service phải chờ payment service (latency tăng)
  - Nếu payment service down → order service fail
  - Coupling chặt giữa service

Kafka (asynchronous):
  + Decoupling: order service không cần biết payment service tồn tại
  + Resilience: payment service down → message tích lũy, xử lý sau
  + Scale độc lập: payment service scale riêng theo Kafka lag
  - Complexity: cần Kafka cluster, consumer group, offset management
  - Eventual consistency: không phải real-time response

→ Với e-commerce xử lý 10.000 đơn/giờ, Kafka là lựa chọn đúng
```

---

## Case Study 3: Data Pipeline với Kubernetes

### Yêu Cầu

```
- ETL pipeline: collect data từ 10 sources → transform → load vào warehouse
- Xử lý 1TB data mỗi ngày
- Job chạy định kỳ: mỗi giờ (incremental), mỗi ngày (full load)
- Cost-efficient: chỉ dùng compute khi có job chạy
```

### Kiến Trúc

```
[CronJob — hourly] → [Job Pod: Extract]
                             │
                     [S3/GCS raw bucket]
                             │
                    [Job Pod: Transform]
                    (Apache Spark on K8s)
                             │
                    [S3/GCS processed bucket]
                             │
                    [Job Pod: Load]
                             │
                    [Data Warehouse — BigQuery/Redshift]
```

### Thiết Kế CronJob và Job

```yaml
# CronJob chạy mỗi giờ
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hourly-etl
  namespace: data-pipeline
spec:
  schedule: "0 * * * *"           # Mỗi giờ
  concurrencyPolicy: Forbid        # Không chạy 2 job song song
  failedJobsHistoryLimit: 3
  successfulJobsHistoryLimit: 5
  jobTemplate:
    spec:
      backoffLimit: 2              # Retry tối đa 2 lần nếu fail
      activeDeadlineSeconds: 3600  # Timeout sau 1 giờ
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: etl-sa
          containers:
          - name: etl-extract
            image: my-etl:v1.2.0
            command: ["python", "extract.py", "--mode=incremental"]
            resources:
              requests:
                cpu: "2"
                memory: "4Gi"
              limits:
                cpu: "4"
                memory: "8Gi"
            env:
            - name: SOURCE_DB_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
          nodeSelector:
            workload-type: batch   # Chạy trên node dành riêng cho batch
```

**Spot Instance (Instance Tiết Kiệm Chi Phí) cho batch job:**
```yaml
# Node group với Spot instances — tiết kiệm 60–70% chi phí
nodeSelector:
  node.kubernetes.io/instance-type: spot

tolerations:
- key: "spot-instance"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
```

**Apache Spark on Kubernetes:**
```bash
# Submit Spark job thẳng lên K8s
spark-submit \
  --master k8s://https://kubernetes-api:6443 \
  --deploy-mode cluster \
  --conf spark.kubernetes.namespace=data-pipeline \
  --conf spark.kubernetes.container.image=my-spark:3.4.0 \
  --conf spark.executor.instances=10 \
  --conf spark.kubernetes.executor.request.cores=1 \
  --conf spark.executor.memory=4g \
  local:///opt/spark/jobs/transform.py
```

---

## Case Study 4: Multi-region High Availability

### Yêu Cầu

```
- Service phục vụ users tại Đông Nam Á (Singapore) và Mỹ (US-East)
- SLA 99.99% = tối đa 52 phút downtime/năm
- RPO = 0 (không mất data), RTO = 5 phút
- Hệ thống banking — compliance yêu cầu data không rời khỏi region
```

### Kiến Trúc Multi-cluster

```
[Global DNS — Route53/CloudDNS]
        │ (Latency-based routing)
        │
   ┌────┴────┐
   │         │
[SEA Cluster]  [US-East Cluster]
(EKS Singapore) (EKS us-east-1)
   │         │
[SEA DB]   [US DB]
(RDS)      (RDS)
   │         │
   └────┬────┘
        │
[DB Replication — cross-region]
(Aurora Global Database)
```

### Thiết Kế Chính

**1. Multi-cluster Service Discovery với ArgoCD:**
```yaml
# ArgoCD ApplicationSet — deploy cùng app lên nhiều cluster
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: banking-app
spec:
  generators:
  - list:
      elements:
      - cluster: sea-cluster
        url: https://sea-eks.example.com
        region: ap-southeast-1
        env: production
      - cluster: us-east-cluster
        url: https://us-eks.example.com
        region: us-east-1
        env: production
  template:
    metadata:
      name: '{{cluster}}-banking-app'
    spec:
      project: banking
      source:
        repoURL: https://github.com/company/banking-app
        targetRevision: HEAD
        path: 'helm/banking-app'
        helm:
          valueFiles:
          - 'values-{{env}}.yaml'
          - 'values-{{region}}.yaml'
      destination:
        server: '{{url}}'
        namespace: banking
```

**2. Failover Strategy (Chiến Lược Chuyển Đổi Dự Phòng):**
```yaml
# Health Check cho Route53 failover
# Nếu SEA cluster unhealthy → Route53 tự động chuyển sang US-East
Type: HTTPS
ResourcePath: /health/cluster
FailureThreshold: 3
RequestInterval: 10
```

**3. Database Replication với Aurora Global:**
```
Primary (Singapore):    Write + Read
Secondary (US-East):    Read only + Standby
Replication lag:        < 1 giây thông thường

Failover:
- Tự động (managed): 1–2 phút
- Thủ công (promote secondary): < 1 phút
→ Đáp ứng RTO = 5 phút
```

### Trade-off Multi-cluster

| Khía Cạnh | Single Cluster | Multi-cluster |
|-----------|---------------|---------------|
| **Độ phức tạp** | Thấp | Cao (2x infrastructure) |
| **Chi phí** | Thấp | 2x chi phí cluster |
| **Availability** | Single point | Chịu được region failure |
| **Latency** | Tốt trong region | Tốt cho user local |
| **Data compliance** | Khó | Data có thể giữ trong region |
| **Kỹ năng vận hành** | Tiêu chuẩn | Cần multi-cluster expertise |

**Kết luận:** Multi-cluster chỉ nên dùng khi SLA > 99.99% hoặc có yêu cầu data residency.

---

## Quyết Định Trade-off Quan Trọng

### Deployment vs StatefulSet cho Database

```
Deployment + RWX Storage:
  + Đơn giản, quen thuộc
  - Không có stable network identity → client không biết kết nối node nào
  - Không có ordered deployment → race condition khi primary/replica init

StatefulSet:
  + Stable hostname (db-0.db-service, db-1.db-service)
  + Ordered startup/shutdown → primary khởi động trước replica
  + Dedicated PVC cho mỗi Pod
  - Phức tạp hơn Deployment
  - Xoá khó hơn (PVC không tự xoá)

→ Database luôn dùng StatefulSet
```

### Helm vs Kustomize vs Raw YAML

```
Raw YAML:
  + Đơn giản nhất, không cần tool
  - Duplicate config giữa môi trường (dev/staging/prod)
  - Không có versioning

Kustomize:
  + Built-in kubectl, không cần cài thêm
  + Overlay pattern: base + patch per environment
  - Không có loop, phức tạp khi có nhiều biến

Helm:
  + Template engine mạnh (loop, condition, function)
  + Package management (versioning, dependencies)
  + Rollback built-in
  - Học đường cong cao hơn
  - Template syntax phức tạp với whitespace

→ Dự án nhỏ/team nhỏ: Kustomize
→ Internal platform / nhiều môi trường: Helm
→ Public chart (chia sẻ cộng đồng): Helm bắt buộc
```

### Ingress vs Service Mesh cho traffic management

```
Ingress Controller:
  + Đơn giản, ít overhead
  + NGINX/Traefik có hầu hết tính năng thông thường
  - Không có mTLS tự động giữa service
  - Không có circuit breaker, retry policy per service

Service Mesh (Istio/Linkerd):
  + mTLS tự động (zero-trust networking)
  + Advanced traffic management: canary, A/B, circuit breaker
  + Distributed tracing tích hợp
  + Fine-grained observability
  - CPU/RAM overhead (Envoy sidecar ~50MB/Pod)
  - Complexity rất cao
  - Debug khó hơn (thêm 1 lớp abstraction)

→ < 10 microservices hoặc team nhỏ: Ingress là đủ
→ > 20 microservices / cần zero-trust / compliance: Service Mesh
```

### EKS vs GKE vs AKS

```
Chọn theo cloud provider hiện có:
- Đang dùng AWS (RDS, S3, CloudFront): EKS
- Đang dùng GCP (BigQuery, Cloud Run): GKE
- Đang dùng Azure (AD, SQL Server): AKS

Nếu chọn từ đầu:
GKE Autopilot   → Ít ops nhất, Google quản lý hoàn toàn node
EKS             → Tích hợp tốt nhất với AWS ecosystem
AKS             → Enterprise với Active Directory integration
Self-managed    → Toàn quyền kiểm soát, on-premise, tiết kiệm chi phí
```

---

## Checklist System Design K8s

### Khi Thiết Kế Deployment

- [ ] Đặt resource request và limit cho mọi container
- [ ] Cấu hình readiness probe và liveness probe
- [ ] Đặt maxUnavailable=0 nếu cần zero-downtime
- [ ] Thêm preStop hook để graceful shutdown
- [ ] Dùng podAntiAffinity để spread Pod ra nhiều node
- [ ] Tạo PodDisruptionBudget cho critical service
- [ ] Dùng ServiceAccount riêng (không dùng default)

### Khi Thiết Kế Scaling

- [ ] HPA với đúng metric (CPU %, custom metric)
- [ ] Cấu hình scaleDown stabilization window
- [ ] Cluster Autoscaler cho môi trường cloud
- [ ] KEDA nếu cần scale theo event (Kafka, SQS)
- [ ] ResourceQuota per namespace để tránh noisy neighbor

### Khi Thiết Kế Security

- [ ] RBAC principle of least privilege
- [ ] NetworkPolicy để cô lập namespace
- [ ] Secret không lưu trong Git (dùng External Secrets)
- [ ] runAsNonRoot và readOnlyRootFilesystem
- [ ] Image từ trusted registry, có scan kết quả

### Khi Thiết Kế Reliability

- [ ] ≥ 3 replica cho critical service
- [ ] Multi-AZ deployment
- [ ] Database backup và point-in-time recovery
- [ ] Monitoring với Prometheus + Grafana
- [ ] Alert trên SLO breach
- [ ] Runbook cho mọi alert

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
