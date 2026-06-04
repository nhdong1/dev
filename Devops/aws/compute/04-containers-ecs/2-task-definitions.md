# ECS Task Definitions — Định Nghĩa Task: Container Specs, CPU/Memory, Biến Môi Trường

> Hướng dẫn chuyên sâu về ECS Task Definition (Định Nghĩa Task — Bản Thiết Kế Container): cách cấu hình container specs, CPU/memory sizing, environment variables (biến môi trường), secrets management (quản lý bí mật), logging, health checks và sidecar patterns.

## 📚 Mục Lục

1. [Task Definition Là Gì?](#task-definition-là-gì)
2. [Task-Level Configuration](#task-level-configuration)
3. [Container Definition — Định Nghĩa Container](#container-definition--định-nghĩa-container)
4. [CPU và Memory Sizing](#cpu-và-memory-sizing)
5. [Environment Variables và Secrets](#environment-variables-và-secrets)
6. [Logging Configuration](#logging-configuration)
7. [Health Checks](#health-checks)
8. [Sidecar Pattern — Mẫu Container Đồng Hành](#sidecar-pattern--mẫu-container-đồng-hành)
9. [Volumes — Lưu Trữ Cho Container](#volumes--lưu-trữ-cho-container)
10. [IAM Roles Cho Task](#iam-roles-cho-task)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 📋 Task Definition Là Gì?

Task Definition là **bản thiết kế bất biến** (immutable blueprint) khai báo mọi thứ cần thiết để chạy một hoặc nhiều container trong ECS. Giống như `docker-compose.yml` nhưng cho môi trường AWS production.

```
Task Definition ≈ Kubernetes Pod Spec ≈ docker-compose.yml (AWS version)

Task Definition định nghĩa:
├── Loại launch (Fargate hay EC2)
├── CPU và Memory tổng cộng
├── IAM Roles (Execution Role và Task Role)
├── Network mode (awsvpc, bridge, host)
├── Volumes (EFS, bind mounts)
└── Container Definitions (1 hoặc nhiều container)
    ├── Docker image
    ├── Port mappings
    ├── CPU/Memory per container
    ├── Environment variables
    ├── Secrets
    ├── Logging
    ├── Health checks
    └── Startup order (dependsOn)
```

### Versioning — Quản Lý Phiên Bản

```
Family: "web-api"          ← Tên nhóm, không thay đổi
Revision 1: nginx:1.19    ← Bất biến sau khi tạo
Revision 2: nginx:1.20    ← Bất biến
Revision 3: nginx:1.21    ← Active ← ECS Service trỏ vào đây

Khi cần deploy version mới:
1. Tạo Revision 4 với image mới
2. Update ECS Service để dùng Revision 4
3. Service thực hiện rolling deployment

Xóa Task Definition:
• Không thể xóa, chỉ có thể DEREGISTER (đánh dấu inactive)
• Revision vẫn tồn tại trong AWS, chỉ không dùng được nữa
```

---

## ⚙️ Task-Level Configuration

### Cấu Hình Cấp Task (Áp Dụng Cho Toàn Task)

```json
{
  "family": "web-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "taskRoleArn": "arn:aws:iam::123456789:role/ecs-task-role",
  "executionRoleArn": "arn:aws:iam::123456789:role/ecs-execution-role",
  "containerDefinitions": [ ... ],
  "volumes": [ ... ],
  "tags": [
    {"key": "Environment", "value": "production"},
    {"key": "Service", "value": "web-api"}
  ]
}
```

### Network Mode — Chế Độ Mạng

```
awsvpc (Khuyến Nghị — Bắt Buộc Với Fargate):
• Mỗi task nhận ENI (Elastic Network Interface — Giao Diện Mạng Co Giãn) riêng
• ENI có IP riêng trong VPC subnet
• Security Group áp dụng ở cấp task (không phải host)
• Containers trong cùng task giao tiếp qua localhost
• Ưu điểm: bảo mật tốt nhất, không port conflict

bridge (EC2 Only — Chế Độ Cầu Nối):
• Container giao tiếp qua Docker bridge network trên host
• Port mapping: container port → random host port
• Nhiều tasks chia sẻ EC2 instance networking
• Cần dynamic port mapping với ALB (ALB tự resolve)
• Ít bảo mật hơn awsvpc

host (EC2 Only — Chế Độ Host):
• Container dùng trực tiếp network interface của EC2 host
• Hiệu suất cao nhất, latency thấp nhất
• Không thể chạy nhiều tasks trên cùng host nếu dùng cùng port
• Dùng cho: monitoring agents, high-performance networking
```

---

## 🐋 Container Definition — Định Nghĩa Container

### Cấu Hình Đầy Đủ Một Container

```json
{
  "name": "web-api",
  "image": "123456789.dkr.ecr.ap-southeast-1.amazonaws.com/web-api:v1.5.0",
  "essential": true,
  "cpu": 512,
  "memory": 1024,
  "memoryReservation": 512,
  "portMappings": [
    {
      "containerPort": 8080,
      "protocol": "tcp",
      "name": "web-api-8080-tcp",
      "appProtocol": "http"
    }
  ],
  "environment": [
    {"name": "APP_ENV", "value": "production"},
    {"name": "DB_HOST", "value": "db.internal"}
  ],
  "secrets": [
    {
      "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:secretsmanager:...:secret:db-password"
    }
  ],
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": {
      "awslogs-group": "/ecs/web-api",
      "awslogs-region": "ap-southeast-1",
      "awslogs-stream-prefix": "ecs"
    }
  },
  "healthCheck": {
    "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
    "interval": 30,
    "timeout": 5,
    "retries": 3,
    "startPeriod": 60
  },
  "essential": true,
  "readonlyRootFilesystem": true,
  "user": "1000"
}
```

### Essential Flag — Cờ Thiết Yếu

```
essential: true  → Nếu container này dừng (exit), toàn bộ task dừng
essential: false → Nếu container này dừng, task tiếp tục chạy

Use Case:
├── App container: essential=true (core của task)
├── Nginx proxy: essential=true (cần thiết để phục vụ traffic)
├── Datadog agent: essential=false (monitoring, không critical)
└── Log forwarder: essential=false (optional sidecar)

Init Container Pattern (Mẫu Container Khởi Tạo):
• Init container: essential=false, dependsOn: COMPLETE
• App container: dependsOn init container: SUCCESS
• Init container chạy migration/setup rồi thoát
• App container chỉ khởi động sau khi init thành công
```

### Port Mappings — Ánh Xạ Cổng

```
awsvpc mode:
• Chỉ cần khai báo containerPort
• hostPort tự động = containerPort (vì task có IP riêng)
• containerPort: 8080 → ENI IP:8080

bridge mode:
• containerPort: 8080
• hostPort: 0 (random) → EC2:random_port → container:8080
• ALB dùng dynamic port registration để biết port

Ví dụ với awsvpc (đơn giản hơn):
"portMappings": [{"containerPort": 8080}]

Ví dụ với bridge (phức tạp hơn):
"portMappings": [{"containerPort": 8080, "hostPort": 0}]
```

---

## 📏 CPU và Memory Sizing

### Fargate CPU/Memory Combinations — Tổ Hợp CPU/Memory

```
Fargate chỉ cho phép các tổ hợp CPU-Memory sau:

CPU (vCPU)    Memory (GB)
──────────    ─────────────────────────────────────────
0.25 vCPU     0.5 GB, 1 GB, 2 GB
0.5 vCPU      1 GB, 2 GB, 3 GB, 4 GB
1 vCPU        2 GB, 3 GB, 4 GB, 5 GB, 6 GB, 7 GB, 8 GB
2 vCPU        4 GB đến 16 GB (1 GB increment)
4 vCPU        8 GB đến 30 GB (1 GB increment)
8 vCPU        16 GB đến 60 GB (4 GB increment)
16 vCPU       32 GB đến 120 GB (8 GB increment)

Trong JSON: cpu tính bằng units (1 vCPU = 1024 units)
• 0.25 vCPU = 256
• 0.5 vCPU = 512
• 1 vCPU = 1024
• 2 vCPU = 2048
```

### Task Level vs Container Level

```
Task Level (bắt buộc với Fargate):
  cpu: 1024     = 1 vCPU tổng cho toàn task
  memory: 2048  = 2 GB RAM tổng cho toàn task

Container Level (tùy chọn, tổng ≤ task level):
  container "app":
    cpu: 512              ← Soft limit (guaranteed minimum)
    memory: 1024          ← Hard limit (OOM kill nếu vượt)
    memoryReservation: 768 ← Soft limit (scheduler dùng để placement)
  
  container "nginx":
    cpu: 256
    memory: 512
    memoryReservation: 256
  
  Còn lại: cpu=256, memory=512 cho burst

Khái Niệm memory vs memoryReservation:
• memory (hard limit): Container bị OOM kill nếu vượt quá
• memoryReservation (soft limit): ECS Scheduler dùng để tính
  available memory trên host. Container có thể dùng nhiều hơn
  nếu host còn RAM trống.
```

### CPU Sizing Best Practice — Thực Hành Tốt Nhất

```
Web API (xử lý HTTP requests):
• Traffic thấp: 0.25 vCPU, 0.5 GB
• Traffic trung bình: 0.5 vCPU, 1 GB
• Traffic cao: 1 vCPU, 2 GB

Worker (xử lý background jobs):
• Jobs nhẹ: 0.25 vCPU, 0.5 GB
• Jobs nặng (data processing): 2-4 vCPU, 4-8 GB

ML Inference (suy diễn học máy):
• Model nhỏ: 2 vCPU, 4 GB
• Model lớn: 4+ vCPU, 16+ GB

Cách đo thực tế:
1. Deploy với size nhỏ
2. Monitor CloudWatch Container Insights
3. Xem p95 CPU utilization và memory usage
4. Điều chỉnh dựa trên data thực
```

---

## 🔐 Environment Variables và Secrets

### Plain Text Environment Variables

```json
"environment": [
  {"name": "APP_ENV",     "value": "production"},
  {"name": "LOG_LEVEL",   "value": "info"},
  {"name": "DB_HOST",     "value": "db.cluster.local"},
  {"name": "REDIS_HOST",  "value": "redis.cluster.local"},
  {"name": "PORT",        "value": "8080"}
]
```

**Dùng cho:** Non-sensitive configuration (cấu hình không nhạy cảm)
- APP_ENV, LOG_LEVEL, feature flags
- Service endpoints (URLs, hostnames)
- Port numbers, timeouts

**Không bao giờ dùng cho:** Passwords, API keys, certificates, private keys

### Secrets Management — Quản Lý Bí Mật

```json
"secrets": [
  {
    "name": "DB_PASSWORD",
    "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:123456789:secret:prod/db-password"
  },
  {
    "name": "API_KEY",
    "valueFrom": "arn:aws:ssm:ap-southeast-1:123456789:parameter/prod/api-key"
  },
  {
    "name": "DB_USERNAME",
    "valueFrom": "arn:aws:secretsmanager:...:secret:prod/db-creds:username::"
  }
]
```

**Cách hoạt động:**
1. ECS Execution Role có quyền đọc từ Secrets Manager / SSM
2. Trước khi container khởi động, ECS Agent fetch secret value
3. Inject vào container như environment variable
4. Container code đọc qua `os.getenv("DB_PASSWORD")`

**So sánh AWS Secrets Manager vs SSM Parameter Store:**

```
AWS Secrets Manager (Trình Quản Lý Bí Mật):
✓ Auto rotation (xoay vòng tự động) — tích hợp RDS, Redshift
✓ Built-in versioning
✓ Higher security: mã hóa với KMS mặc định
✗ Tốn phí: $0.40/secret/tháng + $0.05 per 10K API calls
→ Dùng cho: DB passwords, API keys cần rotate

SSM Parameter Store (Kho Tham Số):
✓ Free tier cho Standard parameters
✓ SecureString: mã hóa với KMS
✓ Hierarchical naming: /prod/app/db/password
✗ Không tự rotate
→ Dùng cho: Config values, feature flags, non-rotating secrets
```

### Secrets Injection Best Practice

```
Tổ Chức Secret Theo Môi Trường:
/prod/web-api/db-password    ← Secrets Manager
/prod/web-api/redis-url      ← SSM SecureString
/prod/web-api/jwt-secret     ← Secrets Manager
/staging/web-api/db-password ← Secrets Manager
/dev/web-api/db-password     ← SSM Standard (không cần rotate)

IAM Policy Cho Execution Role:
{
  "Effect": "Allow",
  "Action": [
    "secretsmanager:GetSecretValue",
    "ssm:GetParameters",
    "kms:Decrypt"
  ],
  "Resource": [
    "arn:aws:secretsmanager:...:secret:prod/web-api/*",
    "arn:aws:ssm:...:parameter/prod/web-api/*"
  ]
}
```

---

## 📊 Logging Configuration

### CloudWatch Logs — Nhật Ký CloudWatch

```json
"logConfiguration": {
  "logDriver": "awslogs",
  "options": {
    "awslogs-group": "/ecs/web-api",
    "awslogs-region": "ap-southeast-1",
    "awslogs-stream-prefix": "ecs",
    "awslogs-create-group": "true",
    "mode": "non-blocking",
    "max-buffer-size": "25m"
  }
}
```

**Log Stream Pattern:**
```
Log Group: /ecs/web-api
Log Stream: ecs/web-api/[task-id]

Ví dụ:
ecs/web-api/abc123def456
ecs/nginx/abc123def456
```

**mode: non-blocking (Chế Độ Không Chặn):**
```
Mặc định (blocking): Nếu CloudWatch Logs chậm → container block
non-blocking: Container không block, log có thể bị drop nếu buffer đầy
→ Khuyến nghị non-blocking cho production để tránh app bị chậm vì logging
```

### Các Log Drivers Khác

```
FireLens (Khuyến Nghị Cho Enterprise):
• Dùng Fluent Bit hoặc Fluentd như sidecar
• Route logs đến nhiều destinations: S3, Elasticsearch, Datadog, Splunk
• Enrichment: thêm metadata (task ID, cluster name) vào log

"logConfiguration": {
  "logDriver": "awsfirelens",
  "options": {
    "Name": "datadog",
    "dd_service": "web-api",
    "dd_source": "nodejs",
    "dd_tags": "env:production"
  }
}

awslogs: Đơn giản, CloudWatch-only, đủ cho hầu hết use cases
awsfirelens: Linh hoạt, cho phép route log đến nhiều nơi
splunk: Nếu công ty dùng Splunk
gelf: Cho Graylog
```

---

## 🏥 Health Checks

### Container Health Check

```json
"healthCheck": {
  "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
  "interval": 30,
  "timeout": 5,
  "retries": 3,
  "startPeriod": 60
}
```

**Các tham số:**
```
command:     Lệnh để kiểm tra health
             CMD: truyền vào exec trực tiếp
             CMD-SHELL: chạy qua /bin/sh -c

interval:    Khoảng thời gian giữa các lần check (giây)
             Mặc định: 30, Khuyến nghị: 15-30

timeout:     Thời gian chờ response (giây)
             Mặc định: 5, nên nhỏ hơn interval

retries:     Số lần fail liên tiếp trước khi mark UNHEALTHY
             Mặc định: 3

startPeriod: Thời gian "grace period" (ân hạn) sau khi khởi động
             Container không bị đánh fail trong thời gian này
             Dùng cho: app cần thời gian warmup, DB connection setup
```

### Health Check vs ALB Health Check

```
Container Health Check:
• ECS kiểm tra container có healthy không
• Nếu fail: ECS stop task, Service launch task mới
• Độc lập với network traffic

ALB (Application Load Balancer — Cân Bằng Tải Ứng Dụng) Health Check:
• ALB gửi HTTP request đến container
• Nếu fail: ALB ngừng gửi traffic đến task đó
• ECS Service dùng để deregister task từ target group

Best Practice:
• Dùng cả hai
• Container health check cho self-healing (tự phục hồi)
• ALB health check cho traffic routing
• Đặt startPeriod đủ dài để app hoàn tất startup
```

### Health Check Patterns

```bash
# HTTP endpoint check (phổ biến nhất)
["CMD-SHELL", "curl -sf http://localhost:8080/health || exit 1"]

# TCP port check (không cần HTTP)
["CMD-SHELL", "nc -z localhost 5432 || exit 1"]

# Custom script
["CMD-SHELL", "/usr/local/bin/healthcheck.sh"]

# Python application
["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8080/health')"]

# Node.js
["CMD-SHELL", "node /app/healthcheck.js"]
```

---

## 🔗 Sidecar Pattern — Mẫu Container Đồng Hành

### Sidecar Là Gì?

Sidecar là container chạy cùng trong một Task với app container chính, cung cấp các chức năng bổ trợ như logging, metrics, proxy.

```
Task (1 Task = nhiều containers):
┌─────────────────────────────────────────────┐
│                                             │
│  ┌─────────────┐    ┌─────────────────┐    │
│  │  APP        │    │  DATADOG AGENT  │    │
│  │  CONTAINER  │    │  (Sidecar)      │    │
│  │             │    │                 │    │
│  │ :8080 →     │    │ Collect metrics │    │
│  │ business    │    │ Collect logs    │    │
│  │ logic       │    │ Send to Datadog │    │
│  └──────┬──────┘    └────────▲────────┘    │
│         │ localhost           │             │
│         └────────────────────┘             │
│                                             │
│  Shared: localhost network, volumes        │
└─────────────────────────────────────────────┘
```

### Init Container Pattern

```json
"containerDefinitions": [
  {
    "name": "db-migration",
    "image": "app:v2.0",
    "command": ["python", "manage.py", "migrate"],
    "essential": false,
    "logConfiguration": { ... }
  },
  {
    "name": "web-app",
    "image": "app:v2.0",
    "essential": true,
    "dependsOn": [
      {
        "containerName": "db-migration",
        "condition": "SUCCESS"
      }
    ],
    ...
  }
]
```

**dependsOn conditions:**
```
START:   Container A phải trong trạng thái RUNNING trước khi B start
COMPLETE: Container A phải kết thúc (bất kỳ exit code) trước khi B start
SUCCESS:  Container A phải kết thúc với exit code 0 trước khi B start
HEALTHY:  Container A phải HEALTHY (pass health check) trước khi B start
```

### FireLens Logging Sidecar

```json
"containerDefinitions": [
  {
    "name": "log-router",
    "image": "906394416424.dkr.ecr.ap-southeast-1.amazonaws.com/aws-for-fluent-bit:stable",
    "essential": true,
    "firelensConfiguration": {
      "type": "fluentbit",
      "options": {"enable-ecs-log-metadata": "true"}
    },
    "memoryReservation": 50
  },
  {
    "name": "app",
    "image": "my-app:latest",
    "essential": true,
    "logConfiguration": {
      "logDriver": "awsfirelens",
      "options": {
        "Name": "cloudwatch",
        "region": "ap-southeast-1",
        "log_group_name": "/ecs/my-app",
        "log_stream_prefix": "ecs/"
      }
    }
  }
]
```

---

## 💾 Volumes — Lưu Trữ Cho Container

### Volume Types

```
Ephemeral Storage (Lưu Trữ Tạm Thời — Mặc Định):
• Fargate: 20 GB mặc định, tăng tối đa 200 GB ($0.10/GB-month)
• Dữ liệu mất khi task stop
• Dùng cho: temp files, build artifacts, cache

Bind Mount (Gắn Kết Thư Mục — EC2 Only):
• Mount thư mục từ EC2 host vào container
• Chia sẻ giữa các containers trong task
• Dữ liệu tồn tại trên EC2 (không mất khi task restart)
• Không phù hợp với Fargate

EFS (Elastic File System — Hệ Thống File Co Giãn):
• NFS-based shared storage
• Nhiều tasks/containers có thể mount cùng filesystem
• Dữ liệu persistent (bền vững), survive task restarts
• Hỗ trợ cả Fargate và EC2 launch type
• Chi phí cao hơn, latency cao hơn local storage
• Dùng cho: shared config files, user uploads, ML model files
```

### EFS Volume Configuration

```json
{
  "volumes": [
    {
      "name": "efs-data",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-12345678",
        "rootDirectory": "/data",
        "transitEncryption": "ENABLED",
        "authorizationConfig": {
          "accessPointId": "fsap-12345678",
          "iam": "ENABLED"
        }
      }
    }
  ],
  "containerDefinitions": [
    {
      "name": "app",
      "mountPoints": [
        {
          "sourceVolume": "efs-data",
          "containerPath": "/app/data",
          "readOnly": false
        }
      ]
    }
  ]
}
```

---

## 🔑 IAM Roles Cho Task

### Hai Roles Hoàn Toàn Khác Nhau

```
┌─────────────────────────────────────────────────────────────────┐
│                         ECS TASK                                │
│                                                                 │
│  ┌──────────────────────────────────┐                          │
│  │  EXECUTION ROLE (Vai Trò Thực Thi│                          │
│  │  Dùng bởi: ECS Container Agent   │                          │
│  │  Mục đích: Infrastructure ops    │                          │
│  │  • Pull image từ ECR             │                          │
│  │  • Ghi logs vào CloudWatch       │                          │
│  │  • Fetch secrets (inject vào env)│                          │
│  └──────────────────────────────────┘                          │
│                                                                 │
│  ┌─────────────────────────────────┐                           │
│  │  TASK ROLE (Vai Trò Task)        │                           │
│  │  Dùng bởi: Code trong container  │                           │
│  │  Mục đích: Application API calls │                           │
│  │  • Đọc/ghi S3 bucket             │                           │
│  │  • Query DynamoDB table          │                           │
│  │  • Gửi message SQS               │                           │
│  │  • Gọi bất kỳ AWS API nào        │                           │
│  └─────────────────────────────────┘                           │
└─────────────────────────────────────────────────────────────────┘
```

### Execution Role Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:log-group:/ecs/*"
    },
    {
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:*:*:secret:prod/*"
    }
  ]
}
```

### Task Role Policy (Principle of Least Privilege)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-app-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/my-app-table"
    },
    {
      "Effect": "Allow",
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:*:*:my-queue"
    }
  ]
}
```

---

## ❓ Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: Task Definition revision là gì và tại sao bất biến?**
> Mỗi lần update Task Definition tạo một revision mới với số thứ tự tăng. Revisions cũ không thể chỉnh sửa — chỉ có thể tạo revision mới. Tính bất biến đảm bảo audit trail đầy đủ và khả năng rollback chính xác về bất kỳ version nào.

**Q: Memory vs memoryReservation khác nhau thế nào?**
> `memory` là hard limit — container bị OOM kill nếu vượt quá. `memoryReservation` là soft limit — ECS Scheduler dùng để quyết định có đủ chỗ cho task trên host không, nhưng container có thể dùng nhiều hơn nếu host còn RAM trống. Best practice: đặt `memoryReservation` ở mức bình thường và `memory` ở mức peak để tránh OOM.

**Q: Secrets trong ECS được inject như thế nào?**
> Trước khi container khởi động, ECS Container Agent (với quyền từ Execution Role) gọi AWS Secrets Manager hoặc SSM Parameter Store để lấy secret value, sau đó inject vào container như environment variable. Container code đọc qua `os.getenv()` bình thường. Secret không bao giờ xuất hiện trong Task Definition dưới dạng plain text.

### Câu Hỏi Nâng Cao

**Q: Khi nào dùng `dependsOn` với containers?**
> `dependsOn` dùng khi có thứ tự phụ thuộc giữa containers trong một task. Ví dụ: init container chạy DB migration phải SUCCESS trước khi app container start. Hoặc proxy container (Envoy) phải HEALTHY trước khi app container nhận traffic. Thiếu `dependsOn` có thể gây race condition — app start trước khi dependency sẵn sàng.

**Q: Tại sao readonlyRootFilesystem là security best practice?**
> `readonlyRootFilesystem: true` ngăn attacker ghi file vào filesystem của container ngay cả khi exploit được container. Kết hợp với chạy container với non-root user (`user: "1000"`), giảm drastically attack surface. App cần ghi file phải dùng named volumes hoặc tmpfs mounts riêng.

**Q: awsvpc mode có nhược điểm gì so với bridge mode?**
> Với awsvpc, mỗi task cần một ENI riêng. EC2 instance có giới hạn số ENI tùy instance type (ví dụ: t3.medium chỉ có 3 ENI). Nếu chạy nhiều tasks trên EC2, dễ bị hết ENI — đây là lý do EC2 launch type với awsvpc đôi khi dùng **trunk ENI** (ENI Trung Gian) để tăng mật độ task. Fargate không có vấn đề này vì mỗi task đã chạy trên compute riêng.

---

**Tiếp Theo:** [3-ecs-services.md](3-ecs-services.md) — ECS Services: Deployment strategies, circuit breaker, Service Auto Scaling.
