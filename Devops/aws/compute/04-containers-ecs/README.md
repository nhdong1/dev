# 🐳 ECS — Elastic Container Service — Dịch Vụ Container AWS

> Hướng dẫn toàn diện về Amazon ECS (Elastic Container Service — Dịch Vụ Container Co Giãn): kiến trúc Cluster/Service/Task, Task Definitions, Fargate vs EC2, deployment strategies, và networking. Nắm vững ECS để xây dựng và vận hành container workload production-ready trên AWS.

## 📚 Mục Lục

1. [Tại Sao ECS?](#tại-sao-ecs)
2. [Tổng Quan Kiến Trúc](#tổng-quan-kiến-trúc)
3. [Launch Types — Fargate vs EC2](#launch-types--fargate-vs-ec2)
4. [Các Thành Phần Cốt Lõi](#các-thành-phần-cốt-lõi)
5. [Lộ Trình Học ECS](#lộ-trình-học-ecs)
6. [Câu Hỏi Phỏng Vấn Trọng Tâm](#câu-hỏi-phỏng-vấn-trọng-tâm)

---

## 🎯 Tại Sao ECS?

### Container trên AWS — Tại Sao Không Chỉ Dùng EC2?

Trước khi có container orchestration, các team deploy ứng dụng trực tiếp lên EC2 instances:

```
Vấn Đề Khi Chạy Container "Thủ Công" Trên EC2:

EC2 Instance 1          EC2 Instance 2          EC2 Instance 3
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ container-A     │    │ container-A     │    │  (đang trống)   │
│ container-B     │    │                 │    │                 │
│ container-C     │    │                 │    │                 │
│ (quá tải 90%)   │    │ (chỉ dùng 30%)  │    │ (lãng phí 0%)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘

Vấn đề: Không biết container nào chạy ở đâu, không tự cân bằng tải,
        container chết không tự khởi động lại, deploy thủ công mất thời gian.
```

**ECS giải quyết tất cả những vấn đề trên:**

| Vấn Đề                              | Giải Pháp ECS                                    |
| ----------------------------------- | ------------------------------------------------ |
| Không biết container chạy ở đâu     | ECS Scheduler tự đặt container lên instance tối ưu |
| Không tự cân bằng tải               | ECS tích hợp với ALB/NLB tự động                |
| Container chết không tự restart     | ECS Service tự thay thế container failed         |
| Deploy thủ công tốn thời gian       | ECS rolling deployment / blue-green tự động      |
| Khó scale container                 | ECS Service Auto Scaling theo metric CloudWatch  |

### ECS vs Các Lựa Chọn Khác

```
Container Orchestration Trên AWS:

┌──────────────────────────────────────────────────────────────────┐
│                   CONTAINER ORCHESTRATION                        │
│                                                                  │
│  ECS (AWS Native)          EKS (Kubernetes)    Self-managed K8s  │
│  ┌──────────────┐          ┌──────────────┐    ┌──────────────┐  │
│  │ AWS-managed  │          │ K8s on AWS   │    │ DIY K8s      │  │
│  │ API          │          │ control plane│    │ on EC2       │  │
│  │ Simple setup │          │ Standard K8s │    │ Full control │  │
│  │ AWS-native   │          │ Portable     │    │ Complex ops  │  │
│  └──────────────┘          └──────────────┘    └──────────────┘  │
│                                                                  │
│  Khuyến Nghị:                                                    │
│  • ECS Fargate: Bắt đầu nhanh, ít ops overhead                  │
│  • EKS: Khi cần Kubernetes standard, đội DevOps mạnh            │
│  • Tránh self-managed K8s: Quá phức tạp trên production         │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ Tổng Quan Kiến Trúc

### Kiến Trúc ECS Toàn Cảnh

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          ECS CLUSTER                                     │
│  (Cluster — Cụm Máy Chủ Container: nhóm logical tài nguyên compute)     │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐     │
│  │                    ECS SERVICE A                                │     │
│  │  (Service — Dịch Vụ: duy trì số lượng Tasks mong muốn)         │     │
│  │                                                                 │     │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │     │
│  │  │   TASK 1    │  │   TASK 2    │  │   TASK 3    │            │     │
│  │  │ (đơn vị     │  │             │  │             │            │     │
│  │  │  thực thi)  │  │             │  │             │            │     │
│  │  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │            │     │
│  │  │ │app:1.0  │ │  │ │app:1.0  │ │  │ │app:1.0  │ │            │     │
│  │  │ │nginx    │ │  │ │nginx    │ │  │ │nginx    │ │            │     │
│  │  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │            │     │
│  │  └─────────────┘  └─────────────┘  └─────────────┘            │     │
│  └─────────────────────────────────────────────────────────────────┘     │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐     │
│  │                    ECS SERVICE B (Worker)                       │     │
│  │  ┌─────────────┐  ┌─────────────┐                              │     │
│  │  │   TASK 1    │  │   TASK 2    │                              │     │
│  │  │ ┌─────────┐ │  │ ┌─────────┐ │                              │     │
│  │  │ │worker   │ │  │ │worker   │ │                              │     │
│  │  │ └─────────┘ │  │ └─────────┘ │                              │     │
│  │  └─────────────┘  └─────────────┘                              │     │
│  └─────────────────────────────────────────────────────────────────┘     │
│                                                                          │
│  Compute: EC2 Instances hoặc Fargate (Serverless)                        │
└──────────────────────────────────────────────────────────────────────────┘
                              │
                     ┌────────▼────────┐
                     │  Task Definition │
                     │ (Bản Thiết Kế   │
                     │  cho Task)       │
                     └─────────────────┘
```

### Các Lớp Trong ECS

```
Lớp 1: ECS API / Control Plane (Mặt Phẳng Điều Khiển)
├── Cluster management
├── Service scheduling
├── Task placement
└── Health monitoring

Lớp 2: Data Plane (Mặt Phẳng Dữ Liệu) — Nơi Container Thực Sự Chạy
├── EC2 Launch Type: EC2 instances với ECS Container Agent
└── Fargate Launch Type: AWS-managed compute (bạn không thấy EC2)

Lớp 3: Container Runtime
├── Docker containers
├── Isolated networking (mỗi task có ENI riêng với awsvpc)
└── Shared/dedicated compute resources
```

---

## 🚀 Launch Types — Fargate vs EC2

### So Sánh Nhanh

```
FARGATE LAUNCH TYPE                    EC2 LAUNCH TYPE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━         ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• AWS quản lý compute                  • Bạn quản lý EC2 instances
• Trả tiền theo vCPU + Memory          • Trả tiền theo EC2 instance size
• Không cần chọn instance type         • Chọn instance type phù hợp
• Khởi động hơi chậm hơn (~30s)       • Khởi động nhanh hơn
• Không có GPU support (hạn chế)      • Hỗ trợ GPU, custom AMI
• Đơn giản hơn cho dev team           • Phức tạp hơn, cần ops
• Chi phí cao hơn cho workload cố định • Tiết kiệm hơn với Reserved/Spot

Dùng Fargate khi:                      Dùng EC2 khi:
✓ Muốn quản lý ít nhất                ✓ Workload dự đoán được, cần tiết kiệm
✓ Workload spiky hoặc burst           ✓ Cần GPU hoặc custom kernel
✓ Dev team nhỏ                        ✓ Cần nhiều quyền kiểm soát
✓ Microservices đa dạng               ✓ Spot Instances để giảm chi phí
```

---

## 🧩 Các Thành Phần Cốt Lõi

### 1. ECS Cluster — Cụm Container

```
Cluster là ranh giới logic nhóm các Service và Task lại.
Không phải ranh giới bảo mật — các cluster khác nhau vẫn dùng
chung IAM account, VPC (nếu không cấu hình khác).

Thông thường tổ chức theo:
• Môi trường: cluster-prod, cluster-staging, cluster-dev
• Ứng dụng: cluster-backend, cluster-frontend, cluster-data
```

### 2. Task Definition — Bản Thiết Kế Task

```json
{
  "family": "web-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "nginx:latest",
      "portMappings": [{"containerPort": 80}],
      "environment": [{"name": "ENV", "value": "production"}],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/web-app",
          "awslogs-region": "ap-southeast-1"
        }
      }
    }
  ]
}
```

**Task Definition quan trọng vì:**
- Định nghĩa image nào chạy
- CPU/Memory được phân bổ bao nhiêu
- Environment variables, secrets
- Log configuration
- Network mode (awsvpc là best practice)

### 3. ECS Task — Đơn Vị Thực Thi

```
Task = 1 instance đang chạy của Task Definition
      = 1 hoặc nhiều container chạy cùng nhau (sidecar pattern)

Task có thể chạy theo 2 cách:
• Qua ECS Service: Duy trì desired count, tự restart khi fail
• Standalone Task (Run Task): Chạy một lần rồi kết thúc (batch job)
```

### 4. ECS Service — Bộ Quản Lý Task

```
Service = Controller đảm bảo luôn có N tasks đang chạy

Các trách nhiệm của Service:
• Duy trì số lượng Task theo desired count
• Tích hợp với Load Balancer để phân phối traffic
• Rolling deployment khi cập nhật Task Definition
• Circuit Breaker (Ngắt Mạch) khi deploy thất bại
• Service Auto Scaling dựa trên CloudWatch metrics
```

---

## 📖 Lộ Trình Học ECS

### Bước 1: Nền Tảng Container (Tuần 1)

- [ ] Hiểu Docker cơ bản: build image, run container, port mapping
- [ ] ECR (Elastic Container Registry — Kho Lưu Trữ Container): push/pull images
- [ ] Tạo ECS Cluster đầu tiên (Fargate)
- [ ] Viết Task Definition đơn giản
- [ ] Chạy standalone Task thành công

### Bước 2: ECS Service & Load Balancer (Tuần 2)

- [ ] Tạo ECS Service với Fargate
- [ ] Tích hợp Application Load Balancer (ALB — Cân Bằng Tải Ứng Dụng)
- [ ] Hiểu awsvpc networking mode
- [ ] Cấu hình Health Check cho Service
- [ ] Test rolling deployment khi update image

### Bước 3: Production-Ready (Tuần 3-4)

- [ ] Service Auto Scaling với CloudWatch metrics
- [ ] Blue/Green Deployment với CodeDeploy
- [ ] Secrets management với AWS Secrets Manager
- [ ] CloudWatch Logs và Container Insights
- [ ] IAM Task Role (Vai Trò IAM Cho Task) — least privilege

### Bước 4: Nâng Cao (Tuần 5+)

- [ ] ECS Service Connect (Kết Nối Service) — service mesh nhẹ
- [ ] EC2 Launch Type với Spot Instances
- [ ] Multi-container Tasks (sidecar patterns)
- [ ] ECS Anywhere — chạy ECS trên on-premise
- [ ] Capacity Providers (Nhà Cung Cấp Năng Lực) — FARGATE + EC2 kết hợp

---

## 🎤 Câu Hỏi Phỏng Vấn Trọng Tâm

### Câu Hỏi Cơ Bản

**Q: ECS Cluster, Service, Task Definition, Task khác nhau thế nào?**
> Cluster là ranh giới logic nhóm tài nguyên. Task Definition là blueprint (bản thiết kế) định nghĩa container specs. Task là 1 instance đang chạy. Service là controller duy trì N tasks và tích hợp load balancer.

**Q: Fargate vs EC2 launch type — khi nào chọn cái nào?**
> Fargate khi muốn giảm ops overhead, workload unpredictable, hoặc team nhỏ. EC2 khi cần tiết kiệm chi phí với Reserved/Spot, cần GPU, hoặc custom AMI.

**Q: awsvpc network mode là gì và tại sao là best practice?**
> Mỗi Task nhận một ENI (Elastic Network Interface — Giao Diện Mạng) riêng biệt với IP riêng trong VPC. Bảo mật hơn (Security Group cấp Task thay vì cấp Host), không có port conflict giữa các task trên cùng EC2.

### Câu Hỏi Nâng Cao

**Q: Circuit Breaker trong ECS Service hoạt động thế nào?**
> Nếu rolling deployment mới có quá nhiều Task failed (không pass health check), ECS tự động rollback về Task Definition cũ. Tránh deploy faulty code lên production.

**Q: Sự khác biệt giữa Task Role và Execution Role trong ECS?**
> Execution Role: ECS dùng để pull image từ ECR và ghi logs vào CloudWatch (infrastructure operations). Task Role: Container code dùng để gọi AWS API như S3, DynamoDB (application permissions). Hai role hoàn toàn độc lập theo nguyên tắc least privilege.

**Q: ECS Service Connect vs Service Discovery khác nhau thế nào?**
> Service Discovery dùng Route 53 (DNS) để resolve service endpoint. Service Connect là solution mới hơn, built-in proxy, hỗ trợ observability (metrics, traces) và circuit breaking tốt hơn. Khuyến nghị Service Connect cho microservices mới.

---

## 📁 Các File Trong Section Này

| File                        | Nội Dung                                              | Trạng Thái |
| --------------------------- | ----------------------------------------------------- | ---------- |
| `README.md`                 | Tổng quan ECS, kiến trúc, lộ trình học                | ✅         |
| `1-ecs-architecture.md`     | Cluster, Service, Task, Container Agent chi tiết      | ✅         |
| `2-task-definitions.md`     | Task Definition deep dive, container specs, env vars  | ✅         |
| `3-ecs-services.md`         | Service deployment strategies, circuit breaker        | ✅         |
| `4-fargate-vs-ec2.md`       | Fargate vs EC2 launch type, trade-offs chi tiết       | ✅         |
| `5-ecs-networking.md`       | awsvpc mode, Service Connect, Service Discovery       | ✅         |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
