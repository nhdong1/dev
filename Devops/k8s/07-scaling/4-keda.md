# KEDA — Kubernetes Event-Driven Autoscaling (Tự Động Mở Rộng Dựa Trên Sự Kiện)

> Hướng dẫn chi tiết về KEDA (Kubernetes Event-Driven Autoscaling — Tự Động Mở Rộng Dựa Trên Sự Kiện): kiến trúc, ScaledObject, ScaledJob, TriggerAuthentication, và các scaler phổ biến: Kafka, SQS, RabbitMQ, Prometheus, Cron.

## Mục Lục

1. [KEDA Là Gì?](#keda-là-gì)
2. [Kiến Trúc KEDA](#kiến-trúc-keda)
3. [ScaledObject — Scale Deployment](#scaledobject--scale-deployment)
4. [ScaledJob — Scale Job](#scaledjob--scale-job)
5. [TriggerAuthentication — Xác Thực Nguồn Sự Kiện](#triggerauthentication--xác-thực-nguồn-sự-kiện)
6. [Scaler Phổ Biến](#scaler-phổ-biến)
7. [Scale To Zero — Giảm Về 0 Replica](#scale-to-zero--giảm-về-0-replica)
8. [Cài Đặt KEDA](#cài-đặt-keda)
9. [Debug và Giám Sát](#debug-và-giám-sát)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## KEDA Là Gì?

**KEDA (Kubernetes Event-Driven Autoscaling)** là CNCF project mở rộng khả năng tự động scaling của Kubernetes. Trong khi HPA chỉ scale theo CPU và memory (hoặc custom metric qua adapter phức tạp), KEDA cho phép scale dựa trên **sự kiện từ hầu hết hệ thống bên ngoài**:

```
Nguồn sự kiện KEDA hỗ trợ (>50 scaler):
─────────────────────────────────────────
Message Queue:   Apache Kafka, RabbitMQ, AWS SQS, Azure Service Bus, NATS
Database:        PostgreSQL, MySQL, MongoDB, Redis
Cloud Native:    AWS CloudWatch, Azure Monitor, GCP Stackdriver
Observability:   Prometheus, Datadog, New Relic
Compute:         Cron Schedule, External HTTP
CI/CD:           GitHub, ArgoCD
```

**Ưu điểm KEDA so với HPA thuần:**

| Khả Năng | HPA | KEDA |
| -------- | --- | ---- |
| Scale theo CPU/memory | ✅ | ✅ |
| Scale theo custom metric | ✅ (cần adapter riêng) | ✅ (tích hợp sẵn) |
| Scale về 0 replica | ❌ | ✅ |
| Trigger từ queue/event | ❌ | ✅ |
| Scale Job (không phải Deployment) | ❌ | ✅ (ScaledJob) |

---

## Kiến Trúc KEDA

```
┌─────────────────────────────────────────────────────────────┐
│                    KEDA System                              │
│                                                             │
│  ┌──────────────────────┐    ┌──────────────────────────┐  │
│  │   KEDA Operator      │    │   KEDA Metrics Adapter   │  │
│  │                      │    │                          │  │
│  │ - Quản lý            │    │ - Implement Custom       │  │
│  │   ScaledObject /     │    │   Metrics API            │  │
│  │   ScaledJob          │    │ - Cung cấp metric        │  │
│  │ - Tạo và quản lý     │    │   từ external source     │  │
│  │   HPA object         │    │   cho HPA                │  │
│  │   tự động            │    │                          │  │
│  └──────────────────────┘    └──────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

       │                                    │
       │ tạo/quản lý HPA                   │ cung cấp metric
       ▼                                    ▼
  Kubernetes HPA ←──────────────── External Source
  (KEDA tạo ra)                (Kafka, SQS, RabbitMQ...)
       │
       │ điều khiển replica
       ▼
  Deployment / StatefulSet / Job
```

KEDA **không thay thế HPA** — KEDA tạo và quản lý một HPA object bên dưới. Khi bạn tạo `ScaledObject`, KEDA tự tạo HPA tương ứng. Bạn không cần và không nên tạo HPA thủ công cho cùng Deployment.

---

## ScaledObject — Scale Deployment

**ScaledObject** là CRD (Custom Resource Definition) chính của KEDA — ánh xạ workload (Deployment, StatefulSet) với một hoặc nhiều trigger.

### Cấu Trúc ScaledObject

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-worker-scaler
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kafka-consumer        # Deployment cần scale

  pollingInterval: 30           # Kiểm tra trigger mỗi 30 giây (mặc định: 30)
  cooldownPeriod: 300           # Chờ 300s trước khi scale về 0 (mặc định: 300)
  idleReplicaCount: 0           # Số replica khi không có sự kiện (mặc định: 0 — scale về 0)
  minReplicaCount: 1            # Số replica tối thiểu khi có sự kiện (0 nếu muốn scale to zero)
  maxReplicaCount: 20           # Số replica tối đa

  fallback:                     # Fallback khi scaler không lấy được metric
    failureThreshold: 3         # Sau 3 lần fail liên tiếp
    replicas: 5                 # Giữ 5 replica thay vì scale về 0

  triggers:
    - type: kafka               # Loại scaler
      metadata:
        bootstrapServers: kafka-broker:9092
        consumerGroup: order-processors
        topic: orders
        lagThreshold: "100"     # Mỗi Pod xử lý 100 message lag

    - type: prometheus          # Trigger thứ 2 — KEDA chọn max
      metadata:
        serverAddress: http://prometheus:9090
        metricName: http_requests_queue
        query: sum(rate(http_requests_total[2m]))
        threshold: "50"
```

---

## ScaledJob — Scale Job

**ScaledJob** tạo **Job mới** cho mỗi batch của sự kiện — thay vì scale Deployment (chạy liên tục), ScaledJob tạo Job chạy một lần rồi kết thúc. Phù hợp cho:

- Xử lý message batch nặng (video encoding, report generation)
- Task có thời gian chạy dài (ETL pipeline)
- Workload cần tài nguyên lớn nhưng không liên tục

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: image-processor-job
  namespace: production
spec:
  jobTargetRef:
    template:
      spec:
        containers:
          - name: image-processor
            image: image-processor:1.0
            env:
              - name: QUEUE_URL
                value: "https://sqs.us-east-1.amazonaws.com/123456/image-queue"
            resources:
              requests:
                cpu: "2"
                memory: "4Gi"
        restartPolicy: Never

  pollingInterval: 10
  maxReplicaCount: 10           # Tối đa 10 Job chạy song song
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3

  scalingStrategy:
    strategy: "accurate"        # Tạo đúng số Job theo số message (accurate/default/custom)

  triggers:
    - type: aws-sqs-queue
      authenticationRef:
        name: sqs-trigger-auth  # tham chiếu TriggerAuthentication
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123456/image-queue
        queueLength: "1"        # 1 Job cho mỗi message trong queue
        awsRegion: us-east-1
```

---

## TriggerAuthentication — Xác Thực Nguồn Sự Kiện

**TriggerAuthentication** lưu thông tin xác thực tách biệt khỏi ScaledObject — tái sử dụng được và bảo mật hơn.

### Xác Thực Qua Kubernetes Secret

```yaml
# Bước 1: Tạo Secret chứa credential
apiVersion: v1
kind: Secret
metadata:
  name: kafka-credentials
  namespace: production
type: Opaque
stringData:
  username: kafka-consumer-user
  password: supersecret

---
# Bước 2: TriggerAuthentication trỏ đến Secret
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: kafka-trigger-auth
  namespace: production
spec:
  secretTargetRef:
    - parameter: username         # tên parameter KEDA scaler dùng
      name: kafka-credentials     # tên Secret
      key: username               # key trong Secret
    - parameter: password
      name: kafka-credentials
      key: password

---
# Bước 3: ScaledObject dùng TriggerAuthentication
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: kafka-consumer
  triggers:
    - type: kafka
      authenticationRef:
        name: kafka-trigger-auth  # tham chiếu TriggerAuthentication
      metadata:
        bootstrapServers: kafka-broker:9092
        consumerGroup: my-group
        topic: my-topic
        lagThreshold: "50"
        sasl: plaintext
        tls: disable
```

### Xác Thực Qua Pod Identity (IRSA Trên EKS)

```yaml
# Không cần Secret — dùng ServiceAccount với IAM role
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: sqs-pod-identity-auth
  namespace: production
spec:
  podIdentity:
    provider: aws               # dùng IRSA (IAM Roles for Service Accounts)
    identityId: arn:aws:iam::123456789:role/keda-sqs-role
```

---

## Scaler Phổ Biến

### Apache Kafka — Scale Theo Consumer Lag

```yaml
triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka-service.kafka.svc:9092
      consumerGroup: order-consumer-group
      topic: orders
      lagThreshold: "100"         # scale nếu lag > lagThreshold × số replica hiện tại
      offsetResetPolicy: latest   # earliest | latest
    authenticationRef:
      name: kafka-auth
```

**Giải thích `lagThreshold`:** Nếu total lag = 500 và lagThreshold = 100, KEDA muốn 5 replica (500/100). Nếu total lag = 0, KEDA scale về `minReplicaCount` (hoặc 0 nếu `idleReplicaCount: 0`).

### Amazon SQS — Scale Theo Queue Length

```yaml
triggers:
  - type: aws-sqs-queue
    authenticationRef:
      name: sqs-pod-identity-auth   # dùng IRSA
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123456/my-queue
      queueLength: "10"             # 1 Pod xử lý 10 message
      awsRegion: us-east-1
      scaleOnInFlight: "true"       # tính cả message đang được xử lý (in-flight)
```

### RabbitMQ — Scale Theo Queue Depth

```yaml
triggers:
  - type: rabbitmq
    metadata:
      protocol: amqp               # amqp hoặc http
      queueName: task-queue
      mode: QueueLength            # QueueLength hoặc MessageRate
      value: "20"                  # 1 Pod xử lý 20 message
    authenticationRef:
      name: rabbitmq-auth
```

### Prometheus — Scale Theo Custom Query

```yaml
triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-service.monitoring.svc:9090
      metricName: active_users
      query: |
        sum(increase(http_requests_total{job="web-api"}[1m]))
      threshold: "100"             # scale nếu query result > threshold
      ignoreNullValues: "false"    # fail scaler nếu query trả về null
```

### Cron Schedule — Scale Theo Lịch

```yaml
triggers:
  - type: cron
    metadata:
      timezone: Asia/Ho_Chi_Minh   # timezone IANA
      start: "0 8 * * 1-5"         # 8:00 SA thứ Hai đến thứ Sáu
      end: "0 20 * * 1-5"          # 8:00 PM thứ Hai đến thứ Sáu
      desiredReplicas: "5"         # giữ 5 replica trong giờ làm việc
```

**Kết hợp cron và metric khác — giờ hành chính + khả năng mở rộng thêm:**
```yaml
triggers:
  - type: cron                     # Giữ 5 replica giờ làm việc
    metadata:
      timezone: Asia/Ho_Chi_Minh
      start: "0 8 * * 1-5"
      end: "0 20 * * 1-5"
      desiredReplicas: "5"

  - type: kafka                    # Tăng thêm nếu Kafka lag cao
    metadata:
      bootstrapServers: kafka:9092
      consumerGroup: my-group
      topic: events
      lagThreshold: "50"
```

---

## Scale To Zero — Giảm Về 0 Replica

Scale về 0 là tính năng đặc biệt của KEDA — HPA không làm được vì HPA luôn yêu cầu `minReplicas >= 1`.

```yaml
spec:
  minReplicaCount: 0        # cho phép scale về 0
  idleReplicaCount: 0       # replica count khi không có trigger
  cooldownPeriod: 300       # chờ 300s không có event mới scale về 0
```

**Lưu ý quan trọng khi scale về 0:**
- **Cold start (Khởi Động Nguội):** Khi có request/event đến sau khi scale về 0, cần thời gian để Pod khởi động (thường 10–60s). Không phù hợp cho service cần latency thấp.
- **Readiness probe:** Cần cấu hình đúng để request không route vào Pod chưa sẵn sàng.
- **Fallback:** Nếu scaler lỗi, dùng `fallback.replicas` để giữ một số replica tránh downtime.

```yaml
spec:
  minReplicaCount: 0
  fallback:
    failureThreshold: 3     # sau 3 lần lấy metric thất bại
    replicas: 2             # giữ 2 replica (không scale về 0)
```

---

## Cài Đặt KEDA

```bash
# Cài KEDA bằng Helm (khuyến nghị)
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace

# Kiểm tra KEDA đang chạy
kubectl get pods -n keda
# keda-operator-xxx               Running
# keda-operator-metrics-apiserver Running

# Kiểm tra CRD được tạo
kubectl get crd | grep keda
# scaledjobs.keda.sh
# scaledobjects.keda.sh
# triggerauthentications.keda.sh
# clustertriggerauthentications.keda.sh
```

---

## Debug và Giám Sát

### Kiểm Tra ScaledObject

```bash
# Xem trạng thái ScaledObject
kubectl get scaledobject -n production

# Output:
# NAME                  SCALETARGETKIND   SCALETARGETNAME   MIN   MAX   READY   ACTIVE   AGE
# kafka-worker-scaler   apps/Deployment   kafka-consumer    1     20    True    True     2d

# Chi tiết và conditions
kubectl describe scaledobject kafka-worker-scaler -n production
```

**Giải thích trạng thái:**
- `READY: True` — KEDA kết nối được với trigger source
- `ACTIVE: True` — đang có event (scale > 0)
- `ACTIVE: False` — không có event (hoặc đã scale về 0)

### Kiểm Tra HPA Được KEDA Tạo

```bash
# KEDA tạo HPA với prefix "keda-hpa-"
kubectl get hpa -n production
# keda-hpa-kafka-worker-scaler   Deployment/kafka-consumer   <unknown>/100   1   20   3   2d
```

### Xem KEDA Operator Logs

```bash
# Log KEDA operator — xem có lỗi kết nối đến scaler không
kubectl logs -n keda -l app=keda-operator --tail=100

# Log metrics server
kubectl logs -n keda -l app=keda-operator-metrics-apiserver --tail=100
```

### Kiểm Tra Metric KEDA Đang Lấy

```bash
# Xem raw metric KEDA báo cáo qua external metrics API
kubectl get --raw \
  "/apis/external.metrics.k8s.io/v1beta1/namespaces/production/s0-kafka-orders?labelSelector=scaledobject.keda.sh%2Fname%3Dkafka-worker-scaler" \
  | jq .
```

---

## Câu Hỏi Phỏng Vấn

**KEDA khác HPA thế nào? Khi nào nên dùng KEDA thay HPA?**

> HPA là controller tích hợp sẵn trong Kubernetes — scale dựa trên CPU, memory, hoặc custom metric qua riêng một adapter (Prometheus Adapter). KEDA là CNCF operator cài thêm — tích hợp sẵn >50 scaler, đặc biệt mạnh với event-driven workload. Dùng KEDA khi: (1) Cần scale theo queue depth (Kafka lag, SQS length, RabbitMQ message count); (2) Cần scale về 0 replica khi không có traffic (tiết kiệm chi phí Spot instance hoặc batch workload không liên tục); (3) Cần scale theo lịch cron kết hợp với metric; (4) Cần ScaledJob — tạo Job mới cho mỗi batch thay vì Deployment thường trực.

**Scale to zero của KEDA hoạt động thế nào? Rủi ro gì?**

> Khi không có event trong `cooldownPeriod` giây (mặc định 300s), KEDA set `minReplicas = 0` trên HPA → HPA scale Deployment về 0 Pod. Khi có event mới, KEDA phát hiện qua polling interval (mặc định 30s), set `minReplicas` về giá trị cấu hình, HPA scale lên. **Rủi ro:** (1) **Cold start latency** — Pod cần thời gian khởi động (image pull, init, warmup) trước khi có thể serve request — không phù hợp service web cần P99 latency thấp; (2) **Polling gap** — có thể mất tối đa `pollingInterval` giây trước khi KEDA phát hiện cần scale up; (3) **Request loss** — nếu không có load balancer queue request trong khi Pod đang khởi động. Giải pháp: dùng `fallback` để giữ replica tối thiểu khi cần, hoặc chỉ scale to zero cho batch/async workload.

**TriggerAuthentication dùng để làm gì? Tại sao không hardcode credential vào ScaledObject?**

> `TriggerAuthentication` tách thông tin xác thực khỏi ScaledObject để: (1) **Tái sử dụng** — nhiều ScaledObject có thể dùng chung một TriggerAuthentication thay vì duplicate credential; (2) **Bảo mật** — credential lưu trong Kubernetes Secret (được encrypt at rest) thay vì nằm inline trong ScaledObject YAML (có thể bị lưu trong Git); (3) **Separation of concerns** — team platform quản lý credential, team app chỉ reference tên TriggerAuthentication; (4) **Pod Identity** — hỗ trợ IRSA (AWS), Workload Identity (GCP), AAD Pod Identity (Azure) để không cần credential nào cả.
