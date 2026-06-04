# Production Checklist — Checklist Trước Khi Đưa Lên Production

> Danh sách kiểm tra toàn diện trước khi triển khai ứng dụng lên môi trường production Kubernetes, bao gồm workload configuration, bảo mật, monitoring, networking và khả năng phục hồi.

## Mục Lục

1. [Cách Dùng Checklist Này](#cách-dùng-checklist-này)
2. [Checklist 1: Workload Configuration](#checklist-1-workload-configuration)
3. [Checklist 2: Resource Management](#checklist-2-resource-management)
4. [Checklist 3: Health Checks](#checklist-3-health-checks)
5. [Checklist 4: Bảo Mật](#checklist-4-bảo-mật)
6. [Checklist 5: Networking](#checklist-5-networking)
7. [Checklist 6: Storage](#checklist-6-storage)
8. [Checklist 7: Monitoring và Alerting](#checklist-7-monitoring-và-alerting)
9. [Checklist 8: Scalability và Resilience](#checklist-8-scalability-và-resilience)
10. [Checklist 9: CI/CD và Deployment](#checklist-9-cicd-và-deployment)
11. [Checklist 10: Disaster Recovery](#checklist-10-disaster-recovery)
12. [Công Cụ Tự Động Hóa Kiểm Tra](#công-cụ-tự-động-hóa-kiểm-tra)

---

## Cách Dùng Checklist Này

```
Mức độ ưu tiên:
🔴 CRITICAL — Bắt buộc trước khi go-live, không có lý do bỏ qua
🟡 IMPORTANT — Nên có, bỏ qua cần có lý do kỹ thuật rõ ràng
🟢 RECOMMENDED — Best practice, ưu tiên nếu có thời gian

Cách điền:
✅ — Đã hoàn thành và verified
❌ — Chưa làm, cần bổ sung
⚠️  — Đã biết risk, có lý do kỹ thuật để bỏ qua, đã document
N/A — Không áp dụng cho use case này
```

---

## Checklist 1: Workload Configuration

### Deployment Strategy (Chiến Lược Triển Khai)

- 🔴 [ ] Deployment có ít nhất **2 replica** để đảm bảo High Availability (không single point of failure)
- 🔴 [ ] Deployment dùng `RollingUpdate` strategy với `maxSurge` và `maxUnavailable` phù hợp
- 🟡 [ ] Đã kiểm tra `PodDisruptionBudget` (PDB) để đảm bảo không bị gián đoạn khi drain node

```yaml
# Ví dụ deployment strategy tốt
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # Cho phép tạo thêm 1 Pod khi rolling
      maxUnavailable: 0     # Không bao giờ giảm dưới 3 Pod (zero-downtime)

---
# PodDisruptionBudget khuyến nghị
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2    # Ít nhất 2 Pod phải running khi drain node
  selector:
    matchLabels:
      app: my-app
```

### Pod Specification

- 🔴 [ ] Tất cả container image dùng **tag cụ thể** (không dùng `:latest`)
- 🔴 [ ] `imagePullPolicy` được set là `IfNotPresent` hoặc `Always` (không phụ thuộc behavior mặc định)
- 🟡 [ ] Pod có `restartPolicy: Always` (mặc định, nhưng nên verify)
- 🟡 [ ] `terminationGracePeriodSeconds` đủ dài để application shutdown gracefully (thường 30–60 giây)
- 🟢 [ ] Pod Topology Spread Constraints được cấu hình để Pod phân bổ đều các zone

```yaml
# terminationGracePeriodSeconds đủ cho graceful shutdown
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: app
      image: my-app:1.2.3              # Tag cụ thể, không dùng :latest
      imagePullPolicy: IfNotPresent
```

---

## Checklist 2: Resource Management

### Resource Request và Limit

- 🔴 [ ] Tất cả container có **CPU request** được set
- 🔴 [ ] Tất cả container có **memory request** được set
- 🔴 [ ] Tất cả container có **memory limit** được set (không set memory limit = OOMKilled cả node có thể xảy ra)
- 🟡 [ ] **CPU limit** được set (không set = noisy neighbor nhưng không bị OOMKilled)
- 🟡 [ ] Request và Limit được tính toán dựa trên **load test thực tế**, không đoán mò
- 🟡 [ ] Memory limit >= 1.5x memory request (đủ buffer cho GC spike)

```yaml
# Resource đúng chuẩn
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1000m"      # 4x request — burst khi cần
    memory: "512Mi"   # 2x request — đủ buffer cho GC
```

### Namespace Quota

- 🟡 [ ] Namespace production có `ResourceQuota` giới hạn tổng resource
- 🟡 [ ] Namespace production có `LimitRange` để set default request/limit cho Pod mới

```yaml
# ResourceQuota mẫu
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
```

---

## Checklist 3: Health Checks

### Liveness Probe (Kiểm Tra Sức Sống)

- 🔴 [ ] Liveness probe được cấu hình cho tất cả container production
- 🔴 [ ] `initialDelaySeconds` đủ dài để application khởi động hoàn toàn trước khi probe chạy
- 🟡 [ ] Liveness probe endpoint **không** gọi external dependencies (database, cache) — chỉ kiểm tra process còn sống
- 🟡 [ ] `failureThreshold` đủ lớn để tránh false restart (thường 3–5)

```yaml
livenessProbe:
  httpGet:
    path: /healthz       # Endpoint đơn giản, không gọi DB
    port: 8080
  initialDelaySeconds: 30    # Chờ app khởi động
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

### Readiness Probe (Kiểm Tra Sẵn Sàng)

- 🔴 [ ] Readiness probe được cấu hình (Pod chỉ nhận traffic khi thực sự sẵn sàng)
- 🟡 [ ] Readiness probe kiểm tra các **dependencies quan trọng** (database connection, cache)
- 🟡 [ ] Readiness probe phân biệt với liveness probe (endpoint khác nhau)

```yaml
readinessProbe:
  httpGet:
    path: /ready         # Kiểm tra DB, cache, external deps
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
  successThreshold: 1    # Phải pass 1 lần mới nhận traffic trở lại
```

### Startup Probe (Kiểm Tra Khởi Động)

- 🟡 [ ] Startup probe được cấu hình cho ứng dụng khởi động **chậm** (JVM, large initialization)

```yaml
# Startup probe — chạy đến khi app xong init, sau đó mới dùng liveness
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30   # 30 lần * 10s = 5 phút tối đa để start
  periodSeconds: 10
```

---

## Checklist 4: Bảo Mật

### Container Security Context (Ngữ Cảnh Bảo Mật Container)

- 🔴 [ ] Container **không chạy với quyền root** (`runAsNonRoot: true`)
- 🔴 [ ] `allowPrivilegeEscalation: false` được set
- 🟡 [ ] `readOnlyRootFilesystem: true` nếu ứng dụng không cần ghi vào filesystem
- 🟡 [ ] `seccompProfile` được set (thường dùng `RuntimeDefault`)
- 🟡 [ ] `capabilities.drop: ["ALL"]` và chỉ thêm lại capability cần thiết

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  seccompProfile:
    type: RuntimeDefault
  capabilities:
    drop:
      - ALL
```

### RBAC (Role-Based Access Control)

- 🔴 [ ] Pod có **ServiceAccount riêng** (không dùng `default` ServiceAccount)
- 🔴 [ ] ServiceAccount có quyền **tối thiểu cần thiết** (Principle of Least Privilege)
- 🟡 [ ] `automountServiceAccountToken: false` nếu Pod không cần gọi API Server

```yaml
# ServiceAccount riêng cho mỗi app
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
automountServiceAccountToken: false   # Tắt nếu không cần
```

### Secret Management (Quản Lý Bí Mật)

- 🔴 [ ] Không có secret hardcode trong Docker image, source code, hoặc ConfigMap
- 🔴 [ ] Secret được inject vào Pod qua Kubernetes Secret (không phải environment variable trong Dockerfile)
- 🟡 [ ] Secret sử dụng External Secrets Operator (ESO) với AWS Secrets Manager, Vault, hoặc tương đương
- 🟡 [ ] etcd encryption at rest được bật

### Network Security

- 🟡 [ ] `NetworkPolicy` được áp dụng để giới hạn ingress/egress của Pod
- 🟡 [ ] Pod không có `hostNetwork: true`, `hostPID: true`, hoặc `hostIPC: true` (trừ khi thực sự cần)

---

## Checklist 5: Networking

### Service

- 🔴 [ ] Service `selector` match đúng với Pod `labels`
- 🔴 [ ] `targetPort` trong Service khớp với `containerPort` trong Pod spec
- 🟡 [ ] Service có `sessionAffinity` phù hợp (ClientIP nếu cần sticky session)

### Ingress

- 🔴 [ ] Ingress có TLS được cấu hình (`tls` section trong spec)
- 🔴 [ ] TLS certificate còn hạn ít nhất **30 ngày** (hoặc cert-manager tự động renew)
- 🟡 [ ] Ingress có annotation cho timeout phù hợp (tránh 504 Gateway Timeout)
- 🟡 [ ] Ingress có rate limiting nếu endpoint public (tránh DDoS)
- 🟡 [ ] Health check endpoint của Ingress được cấu hình đúng

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "15"
```

### DNS

- 🟡 [ ] Service có tên ngắn gọn, nhất quán (tránh tên quá dài gây ndots issues)
- 🟡 [ ] CoreDNS có đủ resource (CPU/memory limit)

---

## Checklist 6: Storage

### PersistentVolumeClaim

- 🔴 [ ] PVC đã được **bound** với PV trước khi deploy (không để Pod Pending vì PVC)
- 🔴 [ ] StorageClass phù hợp được sử dụng (đúng performance tier cho use case)
- 🟡 [ ] PVC có đủ capacity với buffer ít nhất 20% (tránh disk full)
- 🟡 [ ] Backup strategy được thiết lập cho PVC quan trọng

### StatefulSet

- 🔴 [ ] StatefulSet có `podManagementPolicy` phù hợp (`Parallel` hoặc `OrderedReady`)
- 🟡 [ ] `volumeClaimTemplates` có storageClass phù hợp
- 🟡 [ ] Data migration plan có sẵn khi thay đổi PVC size

---

## Checklist 7: Monitoring và Alerting

### Metrics (Số Liệu)

- 🔴 [ ] Ứng dụng expose Prometheus metrics endpoint (`/metrics`)
- 🔴 [ ] `ServiceMonitor` hoặc `PodMonitor` được tạo để Prometheus scrape
- 🔴 [ ] Basic metrics có sẵn: request rate, error rate, latency (RED method)
- 🟡 [ ] Custom business metrics được expose (không chỉ infrastructure metrics)

### Alerting (Cảnh Báo)

- 🔴 [ ] Alert cho **error rate** (thường > 1% là ngưỡng để xem xét)
- 🔴 [ ] Alert cho **latency P99** (dựa trên SLA đã cam kết)
- 🔴 [ ] Alert cho **Pod restart rate** (> 5 lần/giờ = cần điều tra)
- 🟡 [ ] Alert cho **memory usage** (> 80% limit = nguy cơ OOMKilled)
- 🟡 [ ] Alert cho **CPU throttling** (> 25% throttle rate)
- 🟡 [ ] Alert cho **disk usage** (> 80%)
- 🟢 [ ] Alert cho **HPA đang ở max replicas** (cần capacity planning)

```yaml
# Ví dụ PrometheusRule (AlertManager rule)
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-alerts
  namespace: production
spec:
  groups:
    - name: my-app
      rules:
        - alert: HighErrorRate
          expr: |
            rate(http_requests_total{status=~"5..",app="my-app"}[5m])
            / rate(http_requests_total{app="my-app"}[5m]) > 0.01
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Error rate > 1% trên my-app"
```

### Logging (Ghi Log)

- 🔴 [ ] Ứng dụng ghi log ra **stdout/stderr** (không ghi vào file trong container)
- 🔴 [ ] Log có structured format (JSON) để dễ query
- 🟡 [ ] Log aggregation được thiết lập (Loki, Elasticsearch, CloudWatch...)
- 🟡 [ ] Log có **correlation ID** (request ID) để trace xuyên suốt nhiều service
- 🟡 [ ] Log level có thể thay đổi mà không cần restart (hoặc ít nhất có env var `LOG_LEVEL`)

### Distributed Tracing (Theo Dõi Phân Tán)

- 🟢 [ ] Ứng dụng instrument tracing (OpenTelemetry, Jaeger, Zipkin)
- 🟢 [ ] Trace được gửi về centralized tracing system

---

## Checklist 8: Scalability và Resilience

### Auto Scaling (Tự Động Mở Rộng)

- 🟡 [ ] HPA (Horizontal Pod Autoscaler) được cấu hình với `minReplicas` >= 2
- 🟡 [ ] HPA target utilization <= 70% (đủ buffer trước khi scale)
- 🟡 [ ] HPA `scaleUp.stabilizationWindowSeconds` phù hợp (ngắn hơn để scale up nhanh)
- 🟡 [ ] Cluster Autoscaler hoặc Karpenter (AWS) được bật để tự thêm node

```yaml
spec:
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60    # Scale khi CPU > 60%, không phải 80%
```

### Resilience (Khả Năng Phục Hồi)

- 🔴 [ ] Ứng dụng có **retry logic** với exponential backoff cho external calls
- 🔴 [ ] Circuit breaker pattern được implement (hoặc dùng Service Mesh)
- 🟡 [ ] Timeout được set cho tất cả external calls
- 🟡 [ ] Pod có `topologySpreadConstraints` để phân bổ đều qua zones/nodes
- 🟡 [ ] Graceful shutdown được implement (cleanup khi nhận SIGTERM)

```yaml
# Topology spread — phân bổ Pod đều qua zone
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-app
```

---

## Checklist 9: CI/CD và Deployment

### Pipeline

- 🔴 [ ] Image được build và push với tag bất biến (commit SHA, không dùng `latest`)
- 🔴 [ ] Image scanning (quét lỗ hổng bảo mật) trong CI pipeline (Trivy, Snyk)
- 🔴 [ ] Unit test và integration test pass trước khi deploy
- 🟡 [ ] SAST (Static Application Security Testing — Kiểm Thử Bảo Mật Ứng Dụng Tĩnh) trong pipeline
- 🟡 [ ] `kube-score` hoặc `polaris` chạy trong pipeline để validate manifest

### Deployment Process

- 🔴 [ ] Deploy sử dụng **Helm** hoặc **ArgoCD/Flux** (không kubectl apply thủ công trong production)
- 🔴 [ ] Có cơ chế **rollback** được test và documented
- 🟡 [ ] Canary deployment hoặc Blue-Green cho service critical
- 🟡 [ ] Deployment được gated bởi smoke test sau khi deploy

### Change Management

- 🟡 [ ] Thay đổi lớn có deployment window (thời gian deploy được plan trước)
- 🟡 [ ] Freeze period (không deploy) trước các sự kiện quan trọng (Black Friday, launch...)

---

## Checklist 10: Disaster Recovery

### Backup (Sao Lưu)

- 🔴 [ ] Backup strategy cho PersistentVolume quan trọng (database, user data)
- 🔴 [ ] Backup được **verify thường xuyên** (restore test, không chỉ backup mù quáng)
- 🟡 [ ] etcd backup được thực hiện định kỳ (đặc biệt với self-managed cluster)
- 🟡 [ ] Backup có retention policy phù hợp với compliance requirements

### Recovery (Phục Hồi)

- 🔴 [ ] RTO (Recovery Time Objective — Mục Tiêu Thời Gian Phục Hồi) được định nghĩa rõ ràng
- 🔴 [ ] RPO (Recovery Point Objective — Mục Tiêu Điểm Phục Hồi) được định nghĩa rõ ràng
- 🟡 [ ] Disaster recovery procedure (quy trình phục hồi thảm họa) được document và test
- 🟢 [ ] Multi-region hoặc Multi-cluster deployment cho tier-1 services

### Documentation

- 🔴 [ ] Runbook cho các sự cố phổ biến được viết và accessible
- 🔴 [ ] On-call rotation được thiết lập với escalation path rõ ràng
- 🟡 [ ] Architecture diagram được cập nhật
- 🟡 [ ] Dependency map (dịch vụ phụ thuộc vào gì) được document

---

## Công Cụ Tự Động Hóa Kiểm Tra

### kube-score — Validate Manifest

```bash
# Chạy kube-score trên manifest trước khi apply
kube-score score deployment.yaml
kube-score score *.yaml

# Kết quả sẽ show score và danh sách cải thiện cần làm
```

### Polaris — Dashboard Bảo Mật Và Best Practice

```bash
# Cài Polaris
kubectl apply -f https://github.com/FairwindsOps/polaris/releases/latest/download/dashboard.yaml

# Hoặc chạy CLI
polaris audit --audit-path ./k8s-manifests/ --format=pretty
```

### Popeye — Quét Cluster Đang Chạy

```bash
# Popeye scan toàn cluster
kubectl run popeye --image=derailed/popeye --rm -it -- scan

# Hoặc cài binary
popeye --save --out html --output-file /tmp/report.html
```

### Trivy — Image Security Scan

```bash
# Quét image
trivy image my-app:1.2.3

# Scan với fail nếu có CRITICAL vulnerability
trivy image --exit-code 1 --severity CRITICAL my-app:1.2.3

# Scan Kubernetes manifest
trivy config ./k8s-manifests/
```

### Checklist Script Tự Động

```bash
#!/bin/bash
# Quick production readiness check
NAMESPACE=$1

echo "=== Production Readiness Check for namespace: $NAMESPACE ==="

echo ""
echo "--- Resource Requests/Limits ---"
kubectl get pods -n $NAMESPACE -o json | jq '.items[].spec.containers[] | 
  {name: .name, requests: .resources.requests, limits: .resources.limits}' 2>/dev/null

echo ""
echo "--- Pods without Health Probes ---"
kubectl get pods -n $NAMESPACE -o json | jq '.items[] | 
  select(.spec.containers[].livenessProbe == null) | .metadata.name' 2>/dev/null

echo ""
echo "--- PVC Status ---"
kubectl get pvc -n $NAMESPACE

echo ""
echo "--- HPA Status ---"
kubectl get hpa -n $NAMESPACE

echo ""
echo "--- Recent Events (Warning) ---"
kubectl get events -n $NAMESPACE --field-selector type=Warning --sort-by=.lastTimestamp | tail -20
```

---

## Tóm Tắt Mức Độ Ưu Tiên

### Phải Có Trước Khi Go-Live (🔴 CRITICAL)

1. Ít nhất 2 replica
2. Resource request/limit đầy đủ
3. Liveness và Readiness probe
4. Container không chạy root
5. Image tag cụ thể, không dùng `:latest`
6. Secret không hardcode
7. Alert cho error rate và latency
8. Backup cho dữ liệu quan trọng
9. Rollback procedure được test

### Nên Có Trong Sprint Đầu Tiên (🟡 IMPORTANT)

1. PodDisruptionBudget
2. HPA với target < 70%
3. NetworkPolicy
4. ResourceQuota cho namespace
5. Structured logging (JSON)
6. Distributed tracing
7. Runbook cho sự cố phổ biến

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
