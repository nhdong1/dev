# 🐳 Sự Cố Container — ECS & EKS

> Hướng dẫn chẩn đoán và khắc phục toàn diện các sự cố container: ECS task failures, OOM (Out of Memory — Hết Bộ Nhớ), lỗi kéo image, sự cố mạng, và EKS pod issues.

## 📚 Mục Lục

1. [ECS Task Failures — Task Không Thể Khởi Động](#1-ecs-task-failures)
2. [OOM — Container Bị Kill Vì Hết Bộ Nhớ](#2-oom-killer)
3. [Image Pull Failures — Không Kéo Được Image](#3-image-pull-failures)
4. [Container Networking Issues — Sự Cố Mạng Container](#4-container-networking)
5. [EKS Pod Issues — Pod EKS Không Hoạt Động](#5-eks-pod-issues)
6. [ECS Service Deployment Failures](#6-ecs-service-deployment-failures)

---

## 1. ECS Task Failures

### Tìm Lý Do Task Dừng

```bash
# Xem stopped tasks trong cluster (trong 1 giờ qua)
aws ecs list-tasks \
  --cluster my-cluster \
  --desired-status STOPPED \
  --query 'taskArns'

# Xem chi tiết tại sao task dừng — đây là thông tin quan trọng nhất
aws ecs describe-tasks \
  --cluster my-cluster \
  --tasks TASK_ARN_1 TASK_ARN_2 \
  --query 'tasks[*].{
    TaskArn:taskArn,
    StopCode:stopCode,
    StoppedReason:stoppedReason,
    Containers:containers[*].{
      Name:name,
      ExitCode:exitCode,
      Reason:reason
    }
  }'
```

### Giải Mã StopCode và StoppedReason

| `stopCode` | Ý Nghĩa | Hành Động |
|-----------|---------|----------|
| `TaskFailedToStart` | Task không khởi động được | Xem container `reason` — thường là image pull lỗi hoặc resource thiếu |
| `EssentialContainerExited` | Container essential (cốt lõi) thoát | Xem exit code, kiểm tra application logs |
| `ServiceSchedulerInitiated` | Service stop task có chủ đích | Health check fail, deployment, hay scale-in |
| `SpotInterruption` | Spot instance bị AWS thu hồi | Thêm On-Demand capacity vào service |
| `TerminationNotice` | Instance sắp bị terminate | Graceful shutdown không hoàn tất kịp |

### Exit Codes Thường Gặp

```
Exit Code 0  → Thoát bình thường (không phải lỗi)
Exit Code 1  → Lỗi chung trong ứng dụng
Exit Code 137→ OOM Killed (Kernel kill do hết memory) — xem phần 2
Exit Code 139→ Segmentation Fault (Lỗi Phân Đoạn Bộ Nhớ) — bug trong app
Exit Code 143→ SIGTERM — graceful shutdown, bình thường
Exit Code 255→ Entrypoint hoặc command không tìm thấy
```

### Xem Logs Container Bị Dừng

```bash
# ECS với CloudWatch Logs driver
aws logs get-log-events \
  --log-group-name /ecs/my-service \
  --log-stream-name ecs/my-container/TASK_ID \
  --limit 100

# CloudWatch Log Insights — tìm lỗi trong service
aws logs start-query \
  --log-group-name /ecs/my-service \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    fields @timestamp, @message
    | filter @message like /ERROR|FATAL|Exception/
    | sort @timestamp desc
    | limit 50'
```

---

## 2. OOM Killer

### Nhận Biết OOM Killed

```
ECS: Container exit code = 137
     StoppedReason: "Essential container in task exited"
     CloudWatch Metric: MemoryUtilization tiệm cận 100%

EKS: kubectl describe pod → Events:
     "OOMKilled"
     "Container my-app was OOM killed"
```

### Phân Tích Memory Usage ECS

```bash
# Xem memory utilization của service (metrics CloudWatch)
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name MemoryUtilization \
  --dimensions \
    Name=ClusterName,Value=my-cluster \
    Name=ServiceName,Value=my-service \
  --start-time $(date -u -d '2 hours ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Maximum,Average

# Xem memory limit hiện tại trong Task Definition
aws ecs describe-task-definition \
  --task-definition my-task-def \
  --query 'taskDefinition.containerDefinitions[*].{
    Name:name,
    MemoryHard:memory,
    MemorySoft:memoryReservation,
    CPU:cpu
  }'
```

### Cập Nhật Memory Limit Đúng Cách

```json
{
  "containerDefinitions": [
    {
      "name": "my-app",
      "image": "my-image:latest",
      "memory": 1024,
      "memoryReservation": 768,
      "cpu": 512
    }
  ]
}
```

```
Giải thích:
  memory (Hard Limit — Giới Hạn Cứng): 1024 MB
    → Container bị kill ngay khi vượt quá mức này
    → Nên đặt cao hơn memory thực tế cần khoảng 20-30%

  memoryReservation (Soft Limit — Giới Hạn Mềm): 768 MB
    → ECS scheduler dùng giá trị này để đặt container lên host
    → Container có thể dùng hơn mức này nếu host còn tài nguyên
    → Luôn đặt memoryReservation ≤ memory (hard limit)

  cpu: 512
    → 512 CPU units = 0.5 vCPU
    → 1024 CPU units = 1 vCPU
    → Với Fargate, phải theo bảng CPU/Memory hợp lệ
```

### Fargate CPU/Memory Combinations Hợp Lệ

```
CPU     | Memory Range
--------|----------------------------------
0.25 vCPU | 0.5 GB - 2 GB (bước 0.5 GB)
0.5 vCPU  | 1 GB - 4 GB
1 vCPU    | 2 GB - 8 GB
2 vCPU    | 4 GB - 16 GB
4 vCPU    | 8 GB - 30 GB
8 vCPU    | 16 GB - 60 GB
16 vCPU   | 32 GB - 120 GB
```

---

## 3. Image Pull Failures

### Triệu Chứng

```bash
# ECS: Xem stopped task
aws ecs describe-tasks --cluster my-cluster --tasks TASK_ARN \
  --query 'tasks[].containers[].reason'
# Output: "CannotPullContainerError: Error response from daemon:
#  pull access denied for my-repo/my-image, repository does not exist
#  or may require 'docker login'"

# EKS: Xem pod events
kubectl describe pod my-pod
# Events:
#   Failed to pull image "ecr.amazonaws.com/my-repo:tag":
#   rpc error: code = Unknown desc = failed to pull and unpack image:
#   ... ErrImagePull
```

### Các Nguyên Nhân và Cách Xử Lý

#### A. ECR — Amazon Elastic Container Registry (Kho Lưu Trữ Container)

```bash
# Kiểm tra repository tồn tại
aws ecr describe-repositories \
  --repository-names my-repo \
  --region us-east-1

# Kiểm tra image tag tồn tại
aws ecr list-images \
  --repository-name my-repo \
  --filter tagStatus=TAGGED \
  --query 'imageIds[*].imageTag'

# Kiểm tra IAM permissions để pull từ ECR
# ECS Task Execution Role cần có AmazonECSTaskExecutionRolePolicy
# hoặc ecr:GetAuthorizationToken + ecr:BatchGetImage + ecr:GetDownloadUrlForLayer

aws iam list-attached-role-policies \
  --role-name ecsTaskExecutionRole

# ECR authentication token expired (hết hạn mỗi 12 tiếng)
# ECS tự xử lý — nhưng với self-managed EKS cần cấu hình đúng
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com
```

#### B. Image Tag :latest Không Ổn Định

```
Vấn đề: Tag :latest không guarantee cùng một image
        Khi rebuild image với :latest, instances khác có thể pull version khác nhau

Giải pháp: Luôn dùng immutable tags (tag bất biến)
  - Dùng Git commit SHA: my-image:a1b2c3d
  - Dùng version: my-image:v1.2.3
  - Dùng build number: my-image:build-456

# Bật ECR Image Tag Immutability (Bất Biến Tag)
aws ecr put-image-tag-mutability \
  --repository-name my-repo \
  --image-tag-mutability IMMUTABLE
```

#### C. Cross-Account ECR Pull

```bash
# Pull ECR image từ account khác cần Repository Policy
aws ecr set-repository-policy \
  --repository-name my-repo \
  --policy-text '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::OTHER_ACCOUNT_ID:root"
        },
        "Action": [
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:BatchCheckLayerAvailability"
        ]
      }
    ]
  }'
```

---

## 4. Container Networking

### ECS awsvpc Mode — Không Kết Nối Được Giữa Services

```
Trong ECS awsvpc mode (chế độ mạng riêng cho mỗi task):
  - Mỗi task có một ENI riêng với Private IP riêng
  - IP thay đổi mỗi lần task được tạo mới
  - Không nên dùng IP trực tiếp để giao tiếp
```

```bash
# Giải pháp 1: ECS Service Connect (Khuyến Nghị — AWS managed service mesh)
# Cho phép gọi nhau bằng service name thay vì IP

# Giải pháp 2: Cloud Map (AWS Cloud Map — Service Discovery)
# Service tự đăng ký DNS name khi khởi động
# my-service.local hoặc my-service.my-namespace.local

# Giải pháp 3: Application Load Balancer
# Services giao tiếp qua ALB endpoint cố định

# Kiểm tra Service Connect endpoints
aws ecs describe-services \
  --cluster my-cluster \
  --services my-service \
  --query 'services[].serviceConnectConfiguration'
```

### Port Mapping & Security Groups

```bash
# ECS Security Group cần mở port từ:
# 1. Load Balancer Security Group → Container port
# 2. Các ECS services khác (nếu giao tiếp trực tiếp)

# Xem port mapping của task
aws ecs describe-tasks \
  --cluster my-cluster \
  --tasks TASK_ARN \
  --query 'tasks[].containers[].networkBindings'

# Xem Security Group của task (awsvpc mode)
aws ecs describe-tasks \
  --cluster my-cluster \
  --tasks TASK_ARN \
  --query 'tasks[].attachments[].details'
# Tìm "networkInterfaceId" rồi describe để xem Security Groups
```

---

## 5. EKS Pod Issues

### Pod Stuck Pending — Pod Chờ Mãi Không Chạy

```bash
# Xem lý do pod pending
kubectl describe pod my-pod -n my-namespace

# Events thường gặp:
# "0/3 nodes are available: 3 Insufficient cpu"
#   → Node không đủ CPU, cần scale up node group

# "0/3 nodes are available: 3 node(s) had untolerated taint"
#   → Pod không có toleration (Dung Sai) cho taint của node

# "0/3 nodes are available: 3 pod has unbound immediate PersistentVolumeClaims"
#   → PVC chưa được bound với PV

# "didn't match pod anti-affinity rules"
#   → Pod Anti-Affinity (Chống Gần Gũi) ngăn pod được schedule trên node
```

```bash
# Kiểm tra resource request vs node capacity
kubectl get nodes -o custom-columns=\
'NAME:.metadata.name,CPU:.status.capacity.cpu,MEM:.status.capacity.memory'

kubectl describe nodes | grep -A 5 "Allocated resources"

# Xem tất cả pods đang dùng resource
kubectl top pods -A
kubectl top nodes
```

### CrashLoopBackOff — Container Crash Liên Tục

```bash
# Đây là dấu hiệu container chạy xong rồi crash, Kubernetes restart → crash → restart...

# Xem logs của lần chạy hiện tại
kubectl logs my-pod -n my-namespace

# Xem logs của lần chạy trước (trước khi crash)
kubectl logs my-pod -n my-namespace --previous

# Xem exit code
kubectl describe pod my-pod -n my-namespace \
  | grep -A 5 "Last State"
# "Exit Code: 1" → Application error
# "Exit Code: 137" → OOM Killed
# "Exit Code: 139" → Segfault
```

**Nguyên Nhân CrashLoopBackOff:**

```
1. Application lỗi ngay khi start (config sai, database không kết nối được)
2. OOM Killed → Tăng memory limit
3. Liveness Probe (Kiểm Tra Tính Sống) quá nhạy → Giảm threshold hoặc tăng initialDelaySeconds
4. Command/Entrypoint không đúng trong container image
5. Secrets/ConfigMaps bị missing → Container không start được
```

### ImagePullBackOff / ErrImagePull

```bash
# Nguyên nhân:
# 1. Image không tồn tại hoặc tag sai
# 2. Private registry cần authentication (Xác Thực)
# 3. ECR token hết hạn

# Với ECR trên EKS, dùng IAM Roles for Service Accounts (IRSA):
# Node IAM Role hoặc Service Account cần có permission ecr:GetAuthorizationToken

# Kiểm tra node có pull quyền không
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789:role/eks-node-role \
  --action-names ecr:GetAuthorizationToken ecr:BatchGetImage \
  --resource-arns "*"
```

### OOMKilled trong EKS

```yaml
# Tăng memory limit trong Pod spec
resources:
  requests:
    memory: "256Mi"  # Lượng memory tối thiểu được đảm bảo (Scheduler dùng giá trị này)
    cpu: "250m"
  limits:
    memory: "512Mi"  # Hard limit — bị kill nếu vượt quá
    cpu: "500m"      # Soft limit cho CPU — bị throttle không bị kill
```

```
Lưu ý:
  - CPU limit: Container bị CPU throttle khi vượt quá, KHÔNG bị kill
  - Memory limit: Container BỊ KILL ngay khi vượt quá (OOMKilled)
  - requests vs limits: Best practice đặt requests < limits
  - Không nên set CPU limit (gây throttle không cần thiết)
    → Chỉ set CPU requests để scheduler hoạt động đúng
```

### HPA — Horizontal Pod Autoscaler (Tự Động Mở Rộng Pod Theo Chiều Ngang) Không Hoạt Động

```bash
# Kiểm tra trạng thái HPA
kubectl get hpa -n my-namespace
# NAME          REFERENCE            TARGETS    MINPODS   MAXPODS   REPLICAS
# my-hpa        Deployment/my-app    <unknown>  2         10        2
# → "<unknown>" nghĩa là metrics không được thu thập

# Nguyên nhân thường gặp:
# 1. Metrics Server không được cài (cần cho CPU/Memory based HPA)
kubectl get deployment metrics-server -n kube-system

# 2. Pods không có CPU/Memory requests → HPA không tính được %
kubectl describe hpa my-hpa -n my-namespace
# "the HPA was unable to compute the replica count: failed to get cpu
#  utilization: missing request for cpu"
```

---

## 6. ECS Service Deployment Failures

### Circuit Breaker — Tự Động Dừng Deployment Lỗi

```
ECS Deployment Circuit Breaker (Cơ Chế Ngắt Mạch Triển Khai):
  Tự động phát hiện và dừng deployment không thành công
  Tránh replacement endless loop khi image mới bị lỗi

Điều kiện trigger:
  - % tasks healthy thấp hơn minimumHealthyPercent
  - Không đủ tasks healthy sau maximumPercent replacement
```

```bash
# Kiểm tra deployment status
aws ecs describe-services \
  --cluster my-cluster \
  --services my-service \
  --query 'services[].{
    Status:status,
    Deployments:deployments[*].{
      Status:status,
      Desired:desiredCount,
      Running:runningCount,
      Failed:failedTasks,
      RolloutState:rolloutState,
      RolloutStateReason:rolloutStateReason
    }
  }'

# Nếu bị kẹt ở "IN_PROGRESS":
# 1. Xem stopped tasks để hiểu lý do fail
# 2. Rollback bằng cách update service với task definition cũ

# Force rollback về task definition trước đó
aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --task-definition my-task-def:PREVIOUS_REVISION \
  --force-new-deployment
```

### Service Không Đạt Steady State

```
Triệu chứng: ECS Console hiện "Service is not in a steady state"
             Service cứ mãi khởi động rồi dừng tasks

Checklist:
  □ Task definition có lỗi không? (image pull, resources)
  □ Application health check endpoint có trả về 200 không?
  □ Health check grace period có đủ dài không?
  □ ELB target group health check có đúng path không?
  □ IAM Task Role có đủ permissions không?
  □ Application có đọc được Secrets/Environment Variables không?
```

```bash
# Enable ECS Exec để SSH vào running container (debug trực tiếp)
# Yêu cầu: ECS Exec enabled + SSM agent trong container image

aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --enable-execute-command

# Sau khi service update, exec vào container
aws ecs execute-command \
  --cluster my-cluster \
  --task TASK_ARN \
  --container my-container \
  --interactive \
  --command "/bin/sh"
```

---

## 🔁 Tổng Kết — Quy Trình Chẩn Đoán Container

```
Container có vấn đề?
│
├─ ECS TASK KHÔNG KHỞI ĐỘNG?
│   ├─ describe-tasks → xem stoppedReason
│   ├─ Exit code 137 → OOM, tăng memory limit
│   ├─ Exit code 1/255 → Application lỗi, xem CloudWatch Logs
│   └─ CannotPullContainer → Image pull issue (xem phần 3)
│
├─ ECS SERVICE KHÔNG ĐẠT DESIRED COUNT?
│   ├─ describe-services → xem deployments[].rolloutState
│   ├─ Circuit Breaker kích hoạt? → Xem failed tasks
│   └─ Health check fail? → Tăng grace period, sửa endpoint
│
├─ EKS POD PENDING?
│   ├─ kubectl describe pod → xem Events
│   ├─ "Insufficient cpu/memory" → Scale node group
│   └─ "Unschedulable" → Xem node taints/tolerations
│
├─ EKS POD CRASHLOOPBACKOFF?
│   ├─ kubectl logs --previous → Xem logs lần crash trước
│   ├─ Exit 137 → OOM, tăng memory limit
│   └─ Exit 1 → App error, kiểm tra config/secrets
│
└─ NETWORKING?
    ├─ ECS → Kiểm tra Security Group của task ENI
    ├─ ECS Service Connect → Xem service endpoint config
    └─ EKS → Kiểm tra NetworkPolicy, Security Group cho pods
```

---

## 📖 Tham Khảo Thêm

- [../04-containers-ecs/3-ecs-services.md](../04-containers-ecs/3-ecs-services.md) — ECS Service deployment strategies
- [../04-containers-ecs/5-ecs-networking.md](../04-containers-ecs/5-ecs-networking.md) — ECS networking chi tiết
- [../05-containers-eks/5-eks-security.md](../05-containers-eks/5-eks-security.md) — EKS RBAC, IRSA
- [../09-monitoring/1-cloudwatch-metrics.md](../09-monitoring/1-cloudwatch-metrics.md) — ECS/EKS metrics

---

**Cập Nhật Lần Cuối:** 2026-05-15
