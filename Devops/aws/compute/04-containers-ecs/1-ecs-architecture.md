# ECS Architecture — Kiến Trúc ECS: Cluster, Service, Task, Container Agent

> Hiểu sâu về các thành phần cốt lõi của Amazon ECS (Elastic Container Service — Dịch Vụ Container Co Giãn): cách Cluster, Service, Task Definition, Task và Container Agent phối hợp với nhau để chạy container workload production-ready.

## 📚 Mục Lục

1. [Tổng Quan Kiến Trúc](#tổng-quan-kiến-trúc)
2. [ECS Cluster](#ecs-cluster)
3. [ECS Task Definition — Bản Thiết Kế](#ecs-task-definition--bản-thiết-kế)
4. [ECS Task — Đơn Vị Thực Thi](#ecs-task--đơn-vị-thực-thi)
5. [ECS Service — Bộ Quản Lý Task](#ecs-service--bộ-quản-lý-task)
6. [Container Agent — Tác Nhân Container](#container-agent--tác-nhân-container)
7. [ECS Scheduler — Bộ Lên Lịch](#ecs-scheduler--bộ-lên-lịch)
8. [Control Plane vs Data Plane](#control-plane-vs-data-plane)
9. [Luồng Thực Thi Hoàn Chỉnh](#luồng-thực-thi-hoàn-chỉnh)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🏗️ Tổng Quan Kiến Trúc

### Sơ Đồ Toàn Cảnh ECS

```
                    ┌──────────────────────────────────────┐
                    │         AWS ECS CONTROL PLANE         │
                    │    (Mặt Phẳng Điều Khiển — Managed)   │
                    │                                      │
                    │  ┌─────────────┐  ┌──────────────┐  │
                    │  │  Scheduler  │  │  API Server  │  │
                    │  │ (Bộ Lên     │  │  (Cổng API)  │  │
                    │  │  Lịch Task) │  │              │  │
                    │  └─────────────┘  └──────────────┘  │
                    └──────────────┬───────────────────────┘
                                   │ Giao tiếp qua HTTPS
                    ┌──────────────▼───────────────────────┐
                    │         ECS CLUSTER                   │
                    │  (Cụm — Ranh Giới Logic Tài Nguyên)   │
                    │                                      │
                    │  ┌────────────────────────────────┐  │
                    │  │         ECS SERVICE            │  │
                    │  │  (Duy trì desired_count Tasks) │  │
                    │  │  ┌────────┐ ┌────────┐         │  │
                    │  │  │ Task 1 │ │ Task 2 │ ...     │  │
                    │  │  │ ┌────┐ │ │ ┌────┐ │         │  │
                    │  │  │ │app │ │ │ │app │ │         │  │
                    │  │  │ └────┘ │ │ └────┘ │         │  │
                    │  │  └────────┘ └────────┘         │  │
                    │  └────────────────────────────────┘  │
                    │                                      │
                    │  Data Plane: EC2 + Agent | Fargate   │
                    └──────────────────────────────────────┘
```

### Phân Biệt 5 Khái Niệm Cốt Lõi

| Thành Phần           | Vai Trò                                      | Tương Tự                    |
| -------------------- | -------------------------------------------- | --------------------------- |
| **Cluster**          | Ranh giới logic nhóm tài nguyên              | Kubernetes Namespace/Cluster |
| **Task Definition**  | Blueprint (bản thiết kế) cho container       | Kubernetes Pod Spec / Dockerfile |
| **Task**             | 1 instance đang chạy của Task Definition     | Kubernetes Pod               |
| **Service**          | Controller duy trì N tasks + load balancing  | Kubernetes Deployment + Service |
| **Container Agent**  | Agent trên EC2 giao tiếp với ECS Control Plane | Kubernetes kubelet          |

---

## 📦 ECS Cluster

### Cluster Là Gì?

ECS Cluster là **ranh giới logic** (không phải ranh giới vật lý hay bảo mật) nhóm các ECS Services và Tasks lại với nhau. Cluster không có địa chỉ IP hay network boundary riêng.

```
Cluster chứa:
├── Services (Dịch Vụ) — long-running workload
├── Standalone Tasks (Task Riêng Lẻ) — batch/one-off jobs
├── Capacity Providers (Nhà Cung Cấp Năng Lực) — FARGATE, FARGATE_SPOT, EC2 ASG
└── Settings — Container Insights, default capacity provider
```

### Tổ Chức Cluster Theo Môi Trường

```
Mô Hình Phổ Biến — Một Cluster Mỗi Môi Trường:

cluster-production
├── service-api (3 tasks)
├── service-worker (2 tasks)
└── service-cron (1 task)

cluster-staging
├── service-api (1 task)
└── service-worker (1 task)

cluster-development
└── service-api (1 task)

Ưu điểm:
✓ Tách biệt hoàn toàn giữa môi trường
✓ IAM permissions dễ quản lý hơn
✓ Cost tracking rõ ràng theo tag

Nhược điểm:
✗ Nhiều cluster cần quản lý hơn
✗ Không chia sẻ được Capacity Provider EC2
```

### Cluster Settings Quan Trọng

```
Container Insights (Thông Tin Chi Tiết Container):
• Bật: CloudWatch thu thập metrics CPU, memory, network per task/service
• Tốn thêm chi phí CloudWatch (~$0.35/GB log + metrics)
• Khuyến nghị bật cho production

Default Capacity Provider Strategy (Chiến Lược Nhà Cung Cấp Mặc Định):
• FARGATE: Luôn dùng Fargate on-demand
• FARGATE_SPOT: Dùng Fargate Spot (rẻ hơn ~70%, có thể bị thu hồi)
• Mixed: Một phần FARGATE, một phần FARGATE_SPOT
```

---

## 📋 ECS Task Definition — Bản Thiết Kế

### Task Definition Là Gì?

Task Definition là **bản thiết kế bất biến** (immutable blueprint) định nghĩa mọi thứ về cách chạy container. Mỗi lần cập nhật tạo ra một revision (phiên bản) mới.

```
Task Definition: family "web-app"
├── Revision 1: image nginx:1.19, cpu 256, memory 512
├── Revision 2: image nginx:1.20, cpu 256, memory 512   ← Bất biến
├── Revision 3: image nginx:1.21, cpu 512, memory 1024
└── Revision 4 (ACTIVE): image nginx:1.24, cpu 512, memory 1024
```

### Các Thành Phần Task Definition

```
Task-Level Settings (Cài Đặt Cấp Task):
├── family: tên nhóm (ví dụ: "web-app")
├── networkMode: awsvpc | bridge | host | none
├── requiresCompatibilities: ["FARGATE"] hoặc ["EC2"]
├── cpu: 256 | 512 | 1024 | 2048 | 4096 (vCPU * 1024)
├── memory: 512 đến 30720 MB
├── taskRoleArn: IAM Role cho code trong container
├── executionRoleArn: IAM Role cho ECS Agent (pull image, push logs)
└── volumes: EFS mounts, bind mounts

Container-Level Settings (Cài Đặt Cấp Container):
├── name: tên container
├── image: ECR URI hoặc Docker Hub image
├── cpu: soft limit (có thể dùng nhiều hơn nếu còn)
├── memory / memoryReservation: hard/soft limit
├── portMappings: container port → host port (nếu bridge mode)
├── environment: biến môi trường plain text
├── secrets: từ Secrets Manager / Parameter Store
├── logConfiguration: CloudWatch Logs config
├── healthCheck: lệnh kiểm tra sức khỏe container
├── essential: true = task dừng nếu container này fail
└── dependsOn: thứ tự khởi động container (sidecar pattern)
```

### CPU và Memory: Task Level vs Container Level

```
Task Level (bắt buộc với Fargate):
  cpu: 1024    = 1 vCPU tổng cộng cho toàn task
  memory: 2048 = 2 GB RAM tổng cộng

Container Level (tùy chọn):
  containerA: cpu: 512, memory: 1024   ← guaranteed
  containerB: cpu: 256, memory: 512    ← guaranteed
  Còn lại (256 cpu, 512 MB): burst khi cần

Lưu ý: Container có thể dùng quá memory reservation
nhưng bị OOM kill nếu vượt hard limit.
```

### Essential Container — Container Thiết Yếu

```
Task gồm 3 containers:
├── app (essential: true)   ← Nếu chết → toàn task dừng
├── nginx (essential: true) ← Nếu chết → toàn task dừng
└── datadog-agent (essential: false) ← Nếu chết → task tiếp tục

Sidecar Pattern (Mẫu Container Đồng Hành):
• Logging sidecar: Fluentd, Datadog agent
• Proxy sidecar: Envoy, nginx
• Init container: chạy migration DB trước khi app khởi động
```

---

## ⚙️ ECS Task — Đơn Vị Thực Thi

### Task Lifecycle — Vòng Đời Task

```
                PROVISIONING
                     │ (allocate compute resources)
                     ▼
                  PENDING
                     │ (pull image, start containers)
                     ▼
                  RUNNING ◄─────────────────┐
                     │                      │ (restart nếu
                  (stop/fail)               │  là Service)
                     ▼                      │
                 DEPROVISIONING             │ Service
                     │                      │ Controller
                     ▼                      │
                  STOPPED ─────────────────►┘
                  
Task Status có thể là: PROVISIONING, PENDING, ACTIVATING, RUNNING,
                       DEACTIVATING, STOPPING, DEPROVISIONING, STOPPED
```

### Task vs Service Task

```
Standalone Task (Task Đơn Lẻ):
• Chạy một lần, xong việc thì dừng
• Dùng cho: DB migrations, data processing, batch jobs
• Lệnh: aws ecs run-task ...
• Không tự restart nếu fail

Service Task (Task Trong Service):
• ECS Service giám sát, tự khởi động lại khi fail
• Duy trì desired count (số lượng mong muốn)
• Tích hợp với Load Balancer
• Hỗ trợ Auto Scaling, deployment strategies
```

### Task ARN và Task ID

```
Task ARN (Amazon Resource Name — Tên Tài Nguyên Amazon):
arn:aws:ecs:ap-southeast-1:123456789:task/cluster-prod/abc123def456

Dùng Task ID để:
• exec vào container: aws ecs execute-command --task abc123def456
• xem logs: filter by container ID trong CloudWatch
• troubleshoot task failures
```

---

## 🎯 ECS Service — Bộ Quản Lý Task

### Service Là Bộ Điều Phối

```
ECS Service = Kubernetes Deployment + Kubernetes Service kết hợp

Trách Nhiệm Của Service:
1. Duy trì desired_count tasks đang chạy
2. Thay thế tasks failed bằng tasks mới
3. Đăng ký/hủy đăng ký tasks với Load Balancer Target Group
4. Thực hiện rolling deployment khi cập nhật Task Definition
5. Circuit Breaker (Ngắt Mạch) — rollback khi deploy thất bại
6. Service Auto Scaling dựa trên metrics
```

### Service Configuration — Cấu Hình Service

```yaml
# Các tham số quan trọng khi tạo ECS Service

ServiceName: web-api
Cluster: cluster-production
TaskDefinition: web-app:4          # revision 4
DesiredCount: 3                    # số tasks mong muốn
LaunchType: FARGATE                # hoặc EC2

# Deployment Configuration (Cấu Hình Triển Khai)
DeploymentConfiguration:
  MaximumPercent: 200              # tối đa 200% tasks (6/3) trong rolling
  MinimumHealthyPercent: 100       # giữ ít nhất 100% tasks healthy
  
# Circuit Breaker (Ngắt Mạch Tự Động)
DeploymentCircuitBreaker:
  Enable: true
  Rollback: true                   # tự rollback nếu deploy fail

# Load Balancer Integration
LoadBalancers:
  - TargetGroupArn: arn:aws:...
    ContainerName: web
    ContainerPort: 80

# Network Configuration (awsvpc mode)
NetworkConfiguration:
  AwsvpcConfiguration:
    Subnets: [subnet-xxx, subnet-yyy]
    SecurityGroups: [sg-zzz]
    AssignPublicIp: DISABLED
```

### Desired Count và Deployment

```
Trước Deployment:  3 tasks đang chạy (version 1.0)

Rolling Deployment với MaximumPercent=200, MinimumHealthyPercent=100:

Bước 1: Launch 3 tasks mới (version 1.1) → tổng 6 tasks
Bước 2: Wait health check pass cho tasks mới
Bước 3: Deregister và stop 3 tasks cũ (version 1.0)
Kết quả: 3 tasks đang chạy (version 1.1)

                    Time →
tasks(v1.0): ████████████░░░░░░
tasks(v1.1): ░░░░░░░████████████
Total:        ────────6──────3──
```

### Circuit Breaker — Ngắt Mạch Tự Động

```
Vấn Đề Không Có Circuit Breaker:
• Deploy version mới với bug → containers crash liên tục
• ECS tiếp tục launch containers mới → crash tiếp
• Production down cho đến khi manual rollback

Với Circuit Breaker:
1. ECS theo dõi task launch successes và failures
2. Nếu tỷ lệ thất bại cao (threshold vượt ngưỡng)
3. → Dừng deployment
4. → Tự động rollback về Task Definition cũ
5. → Gửi CloudWatch event alert

Rollback Trigger:
• Mặc định: nếu ≥ 10 task failures trong sliding window
• Hoặc: % health tasks giảm xuống dưới MinimumHealthyPercent
```

---

## 🤖 Container Agent — Tác Nhân Container

### Container Agent Là Gì?

Container Agent là **phần mềm chạy trên mỗi EC2 instance** trong ECS Cluster (EC2 launch type). Nó hoạt động như cầu nối giữa EC2 instance và ECS Control Plane.

```
                ECS Control Plane (AWS Managed)
                         │
                    HTTPS API calls
                         │
EC2 Instance             ▼
┌──────────────────────────────────────┐
│  ECS Container Agent                 │
│  (Tác Nhân Container)                │
│  • Đăng ký EC2 vào Cluster           │
│  • Nhận lệnh start/stop Task         │
│  • Báo cáo trạng thái Container      │
│  • Thu thập metrics (CPU, Memory)    │
│  • Pull image từ ECR                 │
│                                      │
│  Docker Daemon (Trình Quản Lý Docker)│
│  ┌──────────┐  ┌──────────┐          │
│  │Container1│  │Container2│          │
│  └──────────┘  └──────────┘          │
└──────────────────────────────────────┘
```

### Container Agent Không Tồn Tại Với Fargate

```
Fargate Launch Type:
• AWS quản lý compute infrastructure bên dưới
• Không có EC2 instance để SSH vào
• Không cần Container Agent
• AWS tự handle việc scheduling task lên compute

EC2 Launch Type:
• Bạn quản lý EC2 instances
• Container Agent chạy như systemd service trên EC2
• AMI: Amazon ECS-optimized AMI (đã có sẵn Agent)
• Cần đảm bảo Agent cập nhật, healthy, có đủ quyền IAM
```

### Container Agent IAM Role

```
EC2 Instance Profile cần có policy AmazonEC2ContainerServiceforEC2Role:
• ecs:RegisterContainerInstance     ← đăng ký vào cluster
• ecs:DeregisterContainerInstance   ← hủy đăng ký khi terminate
• ecs:DiscoverPollEndpoint          ← tìm endpoint để poll
• ecs:Poll                          ← nhận lệnh từ Control Plane
• ecs:StartTelemetrySession         ← gửi metrics
• ecr:GetAuthorizationToken         ← đăng nhập ECR
• ecr:BatchGetImage                 ← pull image
• logs:CreateLogStream              ← ghi CloudWatch Logs
• logs:PutLogEvents                 ← ghi CloudWatch Logs

Lưu ý: Đây là Instance Profile (Hồ Sơ Máy Chủ), khác với
        Task Role (Vai Trò Task) dành cho code trong container.
```

### Container Agent Configuration

```bash
# File cấu hình: /etc/ecs/ecs.config

ECS_CLUSTER=cluster-production        # Cluster mà instance đăng ký vào
ECS_ENGINE_AUTH_TYPE=docker           # Xác thực private registry
ECS_ENABLE_CONTAINER_METADATA=true   # Metadata endpoint cho container
ECS_CONTAINER_STOP_TIMEOUT=30        # Giây chờ graceful shutdown
ECS_ENABLE_SPOT_INSTANCE_DRAINING=true # Tự draining khi Spot bị thu hồi
ECS_RESERVED_MEMORY=256              # MB RAM dành riêng cho Agent
```

---

## 📐 ECS Scheduler — Bộ Lên Lịch

### Scheduler Quyết Định Task Chạy Ở Đâu

ECS Scheduler là thành phần trong Control Plane quyết định **task nào** chạy trên **compute resource nào**.

```
Scheduler nhận đầu vào:
├── Task Definition requirements (CPU, Memory, GPU, ...)
├── Placement Constraints (Ràng Buộc Vị Trí)
│   ├── distinctInstance: mỗi task trên EC2 khác nhau
│   └── memberOf: chỉ chạy trên instance có attribute X
├── Placement Strategy (Chiến Lược Vị Trí)
│   ├── spread: phân tán đều theo AZ / instanceId
│   ├── binpack: đóng gói dày đặc (tiết kiệm instances)
│   └── random: chọn ngẫu nhiên
└── Capacity Provider Strategy (với EC2 launch type)

Scheduler cho ra:
└── Quyết định: Task X chạy trên Instance Y, AZ Z
```

### Placement Strategy — Chiến Lược Phân Bổ Task

```
Spread by AZ (Phân Tán Theo Vùng Khả Dụng) — Khuyến Nghị:
• Task được phân đều giữa các AZ
• us-east-1a: 2 tasks, us-east-1b: 2 tasks, us-east-1c: 1 task
• Giảm thiểu downtime khi một AZ có sự cố

BinPack by Memory (Đóng Gói Theo Bộ Nhớ) — Tiết Kiệm Chi Phí:
• Lấp đầy một instance trước khi sang instance tiếp theo
• Giảm số instances cần thiết
• Trade-off: ít HA hơn

Kết Hợp (Best Practice):
placement_strategy = [
  {type: "spread", field: "attribute:ecs.availability-zone"},
  {type: "binpack", field: "memory"}
]
• Trước tiên spread across AZ, sau đó binpack trong mỗi AZ
```

---

## 🔄 Control Plane vs Data Plane

### Phân Biệt Rõ Ràng

```
CONTROL PLANE (Mặt Phẳng Điều Khiển) — AWS Managed, Bạn Không Thấy:
┌─────────────────────────────────────────────────────────┐
│ • ECS API Server: nhận lệnh từ CLI/Console/SDK          │
│ • Scheduler: quyết định task chạy ở đâu                 │
│ • Controller: duy trì desired state (trạng thái mong muốn)│
│ • Health Monitor: theo dõi task health                  │
│ • Deployment Manager: orchestrate rolling updates       │
└─────────────────────────────────────────────────────────┘

DATA PLANE (Mặt Phẳng Dữ Liệu) — Nơi Container Thực Sự Chạy:
┌─────────────────────────────────────────────────────────┐
│ EC2 Launch Type:                                        │
│ • EC2 instances bạn quản lý trong Auto Scaling Group    │
│ • Container Agent chạy trên mỗi instance                │
│ • Docker daemon chạy containers                         │
│                                                         │
│ Fargate Launch Type:                                    │
│ • AWS-managed micro-VM (Firecracker technology)         │
│ • Không có EC2 instance bạn có thể SSH vào              │
│ • Mỗi task chạy trên compute riêng biệt (better isolation)│
└─────────────────────────────────────────────────────────┘
```

---

## 🔁 Luồng Thực Thi Hoàn Chỉnh

### Từ Lệnh Deploy Đến Container Đang Chạy

```
1. Developer chạy: aws ecs update-service --desired-count 3 --task-definition web:5

2. ECS API Server nhận request, validate, lưu desired state

3. ECS Controller nhận thấy: hiện tại 3 tasks chạy web:4
                             mong muốn: 3 tasks chạy web:5

4. ECS Scheduler tính toán:
   • Cần launch 3 tasks mới (web:5) dựa trên deployment config
   • Chọn AZ, instance/Fargate compute để chạy

5a. Với EC2 Launch Type:
   • Scheduler gửi lệnh đến Container Agent trên EC2
   • Container Agent yêu cầu Docker pull image từ ECR
   • Docker chạy containers theo Task Definition
   • Container Agent báo cáo task RUNNING về Control Plane

5b. Với Fargate Launch Type:
   • AWS provision micro-VM cho task
   • Pull image, start containers
   • Báo cáo trạng thái về Control Plane

6. ECS Service đăng ký Task IPs vào ALB Target Group

7. ALB Health Check confirm tasks healthy

8. ECS Service deregister và stop 3 tasks cũ (web:4)

9. Deployment hoàn thành — 3 tasks web:5 đang nhận traffic
```

---

## ❓ Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: ECS Cluster có phải là network boundary không?**
> Không. Cluster chỉ là ranh giới logic để nhóm Services/Tasks. Nhiều Cluster cùng nằm trong một VPC, dùng chung subnet. Ranh giới network thực sự là VPC, subnet, và Security Group ở cấp Task (với awsvpc mode).

**Q: Task Definition revision (phiên bản) hoạt động thế nào?**
> Mỗi Task Definition family có nhiều revisions. Mỗi lần update tạo revision mới, revision cũ vẫn giữ nguyên (immutable). ECS Service trỏ đến specific revision. Khi muốn deploy code mới, tạo revision mới và update Service trỏ vào revision đó.

**Q: Sự khác biệt giữa Task Role và Execution Role?**
> **Execution Role** được ECS Agent dùng để thực hiện các tác vụ infrastructure: pull image từ ECR, ghi logs vào CloudWatch, lấy secrets từ Secrets Manager để inject vào container. **Task Role** được code bên trong container dùng để gọi AWS APIs như S3, DynamoDB, SQS. Đây là hai role hoàn toàn độc lập — best practice là grant least privilege cho mỗi role.

### Câu Hỏi Nâng Cao

**Q: Khi một ECS Task fail, Service phản ứng thế nào?**
> ECS Service liên tục poll trạng thái tasks. Khi task chuyển sang STOPPED, Service tự động launch task mới để duy trì desired_count. Nếu task fail liên tục (crash loop), Circuit Breaker có thể trigger rollback deployment về Task Definition cũ.

**Q: Container Agent cần quyền IAM gì và tại sao?**
> Container Agent cần Instance Profile với quyền để: (1) đăng ký/hủy đăng ký EC2 vào Cluster, (2) nhận lệnh từ Control Plane, (3) pull container images từ ECR, (4) gửi logs và metrics lên CloudWatch. Đây là IAM Instance Profile cho EC2 — hoàn toàn khác với Task Role dành cho ứng dụng.

**Q: Tại sao Fargate cung cấp isolation tốt hơn EC2 launch type?**
> Fargate chạy mỗi task trên **dedicated micro-VM** sử dụng Firecracker technology (công nghệ máy ảo nhẹ). Không có tenant khác chia sẻ kernel với task của bạn. Với EC2 launch type, nhiều tasks của bạn chia sẻ cùng EC2 instance — nếu một container bị compromise, nó có thể ảnh hưởng containers khác trên cùng host.

**Q: Placement Strategy nào phù hợp cho production?**
> Best practice là kết hợp: `spread` theo `attribute:ecs.availability-zone` trước, rồi `binpack` theo `memory` sau. Spread đảm bảo High Availability (khi một AZ down, tasks vẫn chạy ở AZ khác). BinPack giảm số instances EC2 cần thiết trong mỗi AZ, tiết kiệm chi phí.

---

**Tiếp Theo:** [2-task-definitions.md](2-task-definitions.md) — Task Definition Deep Dive: container specs, CPU/memory sizing, environment variables và secrets management.
