# Scaling Strategies — Chiến Lược Co Giãn Tự Động

> Scaling (Co Giãn) là khả năng thêm hoặc bớt tài nguyên compute tự động theo nhu cầu. Đúng scaling strategy giúp đảm bảo ứng dụng luôn có đủ năng lực xử lý mà không lãng phí chi phí khi traffic thấp.

## 📚 Mục Lục

1. [Horizontal vs Vertical Scaling](#horizontal-vs-vertical)
2. [Reactive Scaling — Co Giãn Phản Ứng](#reactive-scaling)
3. [Predictive Scaling — Co Giãn Dự Báo](#predictive-scaling)
4. [Scheduled Scaling — Co Giãn Theo Lịch](#scheduled-scaling)
5. [Scaling cho ECS & EKS](#scaling-container)
6. [Scaling cho Lambda](#scaling-lambda)
7. [Kết Hợp Các Chiến Lược](#kết-hợp)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## ↔️ Horizontal vs Vertical Scaling {#horizontal-vs-vertical}

### Horizontal Scaling (Scale Out/In — Co Giãn Ngang)

**Thêm/bớt số lượng instances (máy chủ):**

```
Traffic tăng:
[EC2-1] [EC2-2]  →  [EC2-1] [EC2-2] [EC2-3] [EC2-4]

Traffic giảm:
[EC2-1] [EC2-2] [EC2-3] [EC2-4]  →  [EC2-1] [EC2-2]
```

**Ưu điểm:**
- Không có downtime khi scale
- Tăng capacity không giới hạn lý thuyết
- Phù hợp với stateless applications (ứng dụng không lưu trạng thái)
- Tăng tính sẵn sàng (HA) — nhiều instances hơn

**Nhược điểm:**
- Ứng dụng phải stateless hoặc dùng shared session store
- Load balancer cần phân phối traffic
- Khó hơn cho database (stateful)

### Vertical Scaling (Scale Up/Down — Co Giãn Dọc)

**Thay đổi kích thước instance (CPU/RAM):**

```
t3.small (2 vCPU, 2GB)  →  t3.xlarge (4 vCPU, 16GB)
```

**Ưu điểm:**
- Đơn giản — không cần thay đổi kiến trúc
- Phù hợp cho database, legacy apps
- Không cần load balancer

**Nhược điểm:**
- Downtime khi resize (stop/start instance)
- Giới hạn bởi kích thước instance lớn nhất
- SPOF — một instance vẫn là một instance

### So Sánh

| Tiêu Chí               | Horizontal                        | Vertical                          |
| ---------------------- | --------------------------------- | --------------------------------- |
| Downtime               | Không                             | Có (vài phút)                     |
| Giới hạn               | Hầu như không                     | Kích thước instance lớn nhất      |
| Chi phí                | Linh hoạt (pay per instance)      | Tier pricing                      |
| Fault Tolerance        | Cao (nhiều instances)             | Thấp (SPOF)                       |
| Phù hợp cho            | Web/API stateless                 | Database, stateful apps           |

**AWS Best Practice:** Ưu tiên horizontal scaling. Dùng vertical scaling cho database hoặc khi horizontal không khả thi.

---

## ⚡ Reactive Scaling — Co Giãn Phản Ứng {#reactive-scaling}

Reactive Scaling scale dựa trên metric hiện tại — hệ thống **phản ứng** với thay đổi đã xảy ra.

### Target Tracking Scaling (Co Giãn Theo Mục Tiêu)

Cách đơn giản nhất. Bạn chỉ định một metric target, AWS tự tính toán số instances cần thiết.

```bash
# Scale để giữ CPU ở mức 50%
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 50.0,
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }'
```

**Predefined Metrics cho Target Tracking:**

| Metric                              | Dùng Khi                                          |
| ----------------------------------- | ------------------------------------------------- |
| `ASGAverageCPUUtilization`          | CPU-bound workload phổ biến                       |
| `ASGAverageNetworkIn`               | Network-intensive (data ingestion)                |
| `ASGAverageNetworkOut`              | High-bandwidth applications                       |
| `ALBRequestCountPerTarget`          | Web apps — scale theo requests                    |

**Custom Metric Target Tracking:**

```bash
# Scale theo số message trong SQS queue
aws autoscaling put-scaling-policy \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "CustomizedMetricSpecification": {
      "MetricName": "ApproximateNumberOfMessagesVisible",
      "Namespace": "AWS/SQS",
      "Dimensions": [{"Name": "QueueName", "Value": "my-queue"}],
      "Statistic": "Average"
    },
    "TargetValue": 100.0
  }'
```

### Step Scaling (Co Giãn Theo Bước)

Scale với số lượng khác nhau tùy theo mức độ vi phạm ngưỡng.

```
CPU < 40%:   scale in 2 instances
CPU 40-60%:  không thay đổi
CPU 60-70%:  scale out 1 instance
CPU 70-85%:  scale out 2 instances
CPU > 85%:   scale out 4 instances
```

```bash
aws autoscaling put-scaling-policy \
  --policy-name cpu-step-scale-out \
  --policy-type StepScaling \
  --adjustment-type ChangeInCapacity \
  --step-adjustments '[
    {
      "MetricIntervalLowerBound": 0.0,
      "MetricIntervalUpperBound": 10.0,
      "ScalingAdjustment": 1
    },
    {
      "MetricIntervalLowerBound": 10.0,
      "MetricIntervalUpperBound": 25.0,
      "ScalingAdjustment": 2
    },
    {
      "MetricIntervalLowerBound": 25.0,
      "ScalingAdjustment": 4
    }
  ]'
```

### Cooldown Period (Giai Đoạn Hồi Phục)

Sau khi scaling action xảy ra, ASG chờ cooldown trước khi scale tiếp.

```
Scale Out Cooldown (60–300 giây):
  - Ngắn: Phản ứng nhanh với traffic đột biến
  - Thường 60 giây là đủ cho scale-out

Scale In Cooldown (300 giây):
  - Dài hơn: Tránh terminate instance quá sớm khi traffic vừa giảm
  - Thường 300 giây để đảm bảo traffic đã thực sự giảm
```

---

## 🔮 Predictive Scaling — Co Giãn Dự Báo {#predictive-scaling}

Predictive Scaling (Co Giãn Dự Báo) dùng **Machine Learning** phân tích lịch sử 14 ngày để dự báo traffic và **scale trước** khi nhu cầu tăng.

### Hoạt Động Thế Nào?

```
Phân Tích Lịch Sử (14 ngày)
         │
         ▼
Dự Báo Nhu Cầu 48 Giờ Tới
         │
         ▼
Tạo Scheduled Actions Tự Động
         │
         ▼
Scale 5-10 Phút Trước Khi Cần
```

### Ví Dụ Thực Tế

```
Ứng dụng thương mại điện tử:
- Mỗi 8:00 sáng thứ Hai traffic tăng mạnh
- Predictive Scaling nhận ra pattern này
- Tự động tạo scheduled scale-out lúc 7:50
- Instances đã warm và sẵn sàng trước 8:00
```

```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-asg \
  --policy-name predictive-policy \
  --policy-type PredictiveScaling \
  --predictive-scaling-configuration '{
    "MetricSpecifications": [{
      "TargetValue": 50,
      "PredefinedMetricPairSpecification": {
        "PredefinedMetricType": "ASGCPUUtilization"
      }
    }],
    "Mode": "ForecastAndScale",
    "SchedulingBufferTime": 300,
    "MaxCapacityBreachBehavior": "HonorMaxCapacity"
  }'
```

**Chế Độ Predictive Scaling:**

| Mode                  | Mô Tả                                          |
| --------------------- | ---------------------------------------------- |
| `ForecastOnly`        | Chỉ dự báo, không tự động scale               |
| `ForecastAndScale`    | Dự báo và tự động scale (dùng cho production) |

**Điều Kiện Dùng Predictive Scaling:**
- Traffic có pattern tuần hoàn (ngày, tuần, tháng)
- Ứng dụng cần thời gian warmup
- Muốn giảm spike do scale-out chậm

---

## 📅 Scheduled Scaling — Co Giãn Theo Lịch {#scheduled-scaling}

Khi bạn **biết trước** thời điểm traffic tăng/giảm, schedule scaling action.

### Trường Hợp Sử Dụng

- Giờ làm việc (8:00-18:00 weekdays)
- Flash sale được lên kế hoạch
- Batch jobs chạy vào 2:00 sáng
- Cuối tháng — peak billing/reporting

```bash
# Scale up vào đầu giờ làm việc
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name my-asg \
  --scheduled-action-name scale-up-morning \
  --recurrence "0 1 * * MON-FRI" \  # 8:00 ICT = 1:00 UTC, thứ 2-6
  --min-size 4 \
  --desired-capacity 8

# Scale down vào cuối giờ làm việc
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name my-asg \
  --scheduled-action-name scale-down-evening \
  --recurrence "0 12 * * MON-FRI" \  # 19:00 ICT = 12:00 UTC
  --min-size 2 \
  --desired-capacity 2

# Scale up cụ thể cho flash sale
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name my-asg \
  --scheduled-action-name flash-sale-2026 \
  --start-time "2026-11-11T00:00:00Z" \
  --end-time "2026-11-11T23:59:00Z" \
  --min-size 20 \
  --desired-capacity 40
```

### Cron Syntax (Cú Pháp Lịch)

```
* * * * *
│ │ │ │ └── Ngày trong tuần (0-7, 0 và 7 là Chủ Nhật)
│ │ │ └──── Tháng (1-12)
│ │ └────── Ngày trong tháng (1-31)
│ └──────── Giờ (0-23, UTC)
└────────── Phút (0-59)

Ví dụ:
0 1 * * MON-FRI    → 8:00 ICT mỗi ngày thứ 2-6
0 12 * * MON-FRI   → 19:00 ICT mỗi ngày thứ 2-6
0 0 1 * *          → Đầu tháng lúc 7:00 ICT
```

---

## 🐳 Scaling cho ECS & EKS {#scaling-container}

### ECS Application Auto Scaling

ECS dùng **Application Auto Scaling** (không phải EC2 Auto Scaling) để scale số lượng tasks.

```bash
# Đăng ký ECS Service với Application Auto Scaling
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/my-cluster/my-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 20

# Target Tracking cho ECS
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/my-cluster/my-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name ecs-cpu-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "TargetValue": 60.0
  }'
```

**ECS Predefined Metrics:**
- `ECSServiceAverageCPUUtilization`
- `ECSServiceAverageMemoryUtilization`
- `ALBRequestCountPerTarget`

### Kubernetes HPA — Horizontal Pod Autoscaler

HPA (Horizontal Pod Autoscaler — Tự Động Co Giãn Pod Theo Chiều Ngang) tự động scale số pods.

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
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60    # Scale khi CPU > 60%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70    # Scale khi Memory > 70%
```

### Kubernetes VPA — Vertical Pod Autoscaler

VPA (Vertical Pod Autoscaler — Tự Động Co Giãn Pod Theo Chiều Dọc) điều chỉnh CPU/Memory requests của pods.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"    # Tự động restart pods với resources mới
  resourcePolicy:
    containerPolicies:
      - containerName: my-app
        minAllowed:
          cpu: 100m
          memory: 50Mi
        maxAllowed:
          cpu: 2
          memory: 2Gi
```

### Kubernetes Cluster Autoscaler — Tự Động Mở Rộng Cluster

Cluster Autoscaler (Bộ Tự Động Co Giãn Cluster) thêm/bớt EC2 nodes khi pods không thể schedule.

```yaml
# Cấu hình Cluster Autoscaler trong EKS
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
        - image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.0
          command:
            - ./cluster-autoscaler
            - --cloud-provider=aws
            - --nodes=2:10:my-node-group
            - --scale-down-delay-after-add=10m
            - --scale-down-unneeded-time=10m
```

**KEDA — Kubernetes Event-Driven Autoscaling (Co Giãn Theo Sự Kiện):**

KEDA scale pods dựa trên nguồn sự kiện bên ngoài (SQS, Kafka, Redis...).

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: sqs-scaler
spec:
  scaleTargetRef:
    name: my-worker
  minReplicaCount: 0      # Scale to zero khi không có việc
  maxReplicaCount: 50
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.ap-southeast-1.amazonaws.com/123/my-queue
        queueLength: "10"  # 1 pod cho mỗi 10 messages
```

---

## ⚡ Scaling cho Lambda {#scaling-lambda}

Lambda scale hoàn toàn tự động — bạn chỉ cần hiểu giới hạn.

### Concurrency Model (Mô Hình Đồng Thời)

```
Request 1 ──→ Lambda Instance #1
Request 2 ──→ Lambda Instance #2   (mới tạo)
Request 3 ──→ Lambda Instance #3   (mới tạo)
...
Request 1000 ──→ Lambda Instance #1000  (mới tạo)

Tất cả chạy song song trong vài giây!
Giới hạn mặc định: 1000 concurrent executions / Region
```

**Provisioned Concurrency (Đồng Thời Được Cung Cấp Sẵn):**

```bash
# Giữ sẵn 10 instances warm
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier production \
  --provisioned-concurrent-executions 10
```

**Reserved Concurrency (Đồng Thời Được Đặt Trước):**

```bash
# Giới hạn function chỉ dùng tối đa 100 concurrent
aws lambda put-function-concurrency \
  --function-name my-function \
  --reserved-concurrent-executions 100
```

---

## 🔀 Kết Hợp Các Chiến Lược {#kết-hợp}

Trong thực tế, bạn kết hợp nhiều chiến lược:

```
Ứng Dụng E-commerce Production:

1. Scheduled Scaling:
   - Thứ 2-6 sáng: desired=8 (giờ làm việc)
   - Tối cuối tuần: desired=4 (traffic thấp)
   - 11/11: desired=40 (flash sale đã biết)

2. Predictive Scaling:
   - Học pattern traffic thứ 6 tối luôn tăng
   - Tự scale lên trước khi cần

3. Reactive Scaling (Target Tracking):
   - CPU > 60%: scale out thêm
   - CPU < 30%: scale in
   - Xử lý traffic đột biến không dự đoán được

Kết quả: 
- Giảm chi phí 40% so với overprovisioning
- Zero downtime khi traffic spike
- Tự động thích nghi với mọi pattern
```

### Thứ Tự Ưu Tiên

Khi nhiều policies xung đột:
1. **Scheduled actions** được thực hiện trước
2. **Predictive Scaling** tạo scheduled actions tự động
3. **Reactive Scaling** (Target Tracking, Step) điều chỉnh liên tục

---

## ❓ Câu Hỏi Phỏng Vấn

**Q: Target Tracking vs Step Scaling — khi nào dùng cái nào?**
> Target Tracking: Đơn giản, phù hợp hầu hết trường hợp, AWS tự tính toán. Step Scaling: Khi cần kiểm soát chính xác số instances scale ở mỗi ngưỡng khác nhau. Ưu tiên Target Tracking cho mới bắt đầu.

**Q: Tại sao Scale-In Cooldown thường dài hơn Scale-Out Cooldown?**
> Scale-Out: Cần phản ứng nhanh khi traffic tăng → cooldown ngắn (60s). Scale-In: Tránh terminate instances rồi lại phải tạo mới ngay sau đó → cooldown dài (300s). Scale-In vội vã tốn chi phí và tạo overhead.

**Q: Predictive Scaling có thể thay thế Reactive Scaling không?**
> Không. Predictive Scaling chỉ hoạt động tốt với traffic có pattern tuần hoàn. Traffic đột biến không dự đoán được (viral post, DDoS) vẫn cần Reactive Scaling. Kết hợp cả hai cho coverage tối đa.

**Q: KEDA là gì và khi nào dùng?**
> KEDA (Kubernetes Event-Driven Autoscaling) scale pods dựa trên số events trong queue (SQS, Kafka...) thay vì CPU/Memory. Dùng khi bạn có worker pods xử lý queue — scale số pods theo số messages, kể cả scale to zero.

---

**Tiếp Theo:** [3-health-checks-recovery.md](./3-health-checks-recovery.md) — Health Checks & Tự Động Phục Hồi
