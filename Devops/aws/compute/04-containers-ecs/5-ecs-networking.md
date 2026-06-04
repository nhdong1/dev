# ECS Networking — Mạng ECS: awsvpc Mode, Service Connect, Service Discovery

> Hướng dẫn toàn diện về ECS Networking (Mạng ECS): awsvpc network mode (chế độ mạng awsvpc), Service Connect (Kết Nối Dịch Vụ), Service Discovery (Khám Phá Dịch Vụ) với Cloud Map, và các pattern mạng production-ready cho container workloads trên AWS.

## 📚 Mục Lục

1. [Tổng Quan Networking ECS](#tổng-quan-networking-ecs)
2. [Network Modes — Các Chế Độ Mạng](#network-modes--các-chế-độ-mạng)
3. [awsvpc Mode — Chế Độ Mạng Khuyến Nghị](#awsvpc-mode--chế-độ-mạng-khuyến-nghị)
4. [Bridge Mode — Chế Độ Cầu Nối](#bridge-mode--chế-độ-cầu-nối)
5. [VPC và Subnet Design Cho ECS](#vpc-và-subnet-design-cho-ecs)
6. [Security Groups Cho ECS](#security-groups-cho-ecs)
7. [Load Balancer Patterns](#load-balancer-patterns)
8. [Service Discovery Với AWS Cloud Map](#service-discovery-với-aws-cloud-map)
9. [ECS Service Connect](#ecs-service-connect)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🌐 Tổng Quan Networking ECS

### Thách Thức Networking Của Container

```
Vấn Đề Networking Khi Chạy Nhiều Containers:

1. Port Conflict (Xung Đột Cổng):
   Container A: port 8080
   Container B: port 8080
   Trên cùng EC2 host → conflict!

2. Service Discovery (Khám Phá Dịch Vụ):
   Service A cần gọi Service B
   Service B có thể chạy trên nhiều tasks với IPs khác nhau
   Làm sao biết địa chỉ IP để gọi?

3. Network Isolation (Cách Ly Mạng):
   Container của service A không nên access trực tiếp DB
   của service B → cần fine-grained network policies

4. Load Balancing (Cân Bằng Tải):
   Traffic từ internet cần được phân phối đều
   giữa nhiều tasks của cùng service

ECS giải quyết bằng: awsvpc mode + Service Connect + ALB
```

### Kiến Trúc Mạng ECS Hoàn Chỉnh

```
Internet
   │
   ▼
Internet Gateway (Cổng Internet)
   │
   ▼
VPC (Virtual Private Cloud — Đám Mây Riêng Ảo)
   │
   ├── Public Subnet (Mạng Con Công Khai)
   │   └── ALB (Application Load Balancer — Cân Bằng Tải Ứng Dụng)
   │       └── Security Group: chấp nhận 80, 443 từ internet
   │
   └── Private Subnet (Mạng Con Riêng Tư)
       ├── ECS Task A (ENI: 10.0.1.10)
       │   └── Security Group: chấp nhận từ ALB SG
       ├── ECS Task B (ENI: 10.0.2.15)
       │   └── Security Group: chấp nhận từ ALB SG
       └── RDS Database (ENI: 10.0.1.50)
           └── Security Group: chấp nhận từ ECS Task SG
```

---

## 🔌 Network Modes — Các Chế Độ Mạng

### 4 Network Modes Trong ECS

```
┌─────────────────────────────────────────────────────────────────────┐
│ awsvpc (KHUYẾN NGHỊ — Bắt Buộc Với Fargate)                        │
│ • Mỗi task = 1 ENI riêng với IP trong VPC                           │
│ • Security Group cấp task                                           │
│ • Không có port conflict                                            │
│ • Hỗ trợ: Fargate + EC2                                             │
│                                                                     │
│ bridge (EC2 Only — Cầu Nối Docker)                                  │
│ • Containers dùng Docker bridge network trên host                   │
│ • Port mapping: container port → random/fixed host port             │
│ • Security Group cấp EC2 instance (không phải task)                 │
│ • Hỗ trợ: EC2 only                                                  │
│                                                                     │
│ host (EC2 Only — Mạng Host)                                         │
│ • Container dùng trực tiếp network interface của EC2 host           │
│ • Hiệu suất cao nhất, latency thấp nhất                             │
│ • Không thể có nhiều tasks dùng cùng port                           │
│ • Hỗ trợ: EC2 only                                                  │
│                                                                     │
│ none (Không Mạng)                                                   │
│ • Container không có network connectivity                           │
│ • Dùng cho: isolated compute jobs, security-sensitive tasks         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🥇 awsvpc Mode — Chế Độ Mạng Khuyến Nghị

### Cơ Chế Hoạt Động

```
VPC Subnet: 10.0.1.0/24

EC2 Instance (hoặc Fargate compute):
┌──────────────────────────────────────────────────────────┐
│  EC2 Host ENI: 10.0.1.5 (primary network interface)     │
│                                                          │
│  Task A → ENI: 10.0.1.10 (dedicated, attached to task)  │
│  ┌────────────────────────────────────────────────┐     │
│  │  Container 1: localhost:8080                   │     │
│  │  Container 2: localhost:9090                   │     │
│  │  Giao tiếp với nhau qua: localhost             │     │
│  │  Giao tiếp ra ngoài qua: ENI IP 10.0.1.10     │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
│  Task B → ENI: 10.0.1.11 (dedicated, attached to task)  │
│  ┌────────────────────────────────────────────────┐     │
│  │  Container 1: localhost:8080                   │     │
│  │  (Không conflict với Task A vì ENI riêng)      │     │
│  └────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────┘
```

### ENI Trunking — Giải Quyết Giới Hạn ENI Trên EC2

```
Vấn Đề: EC2 instance có giới hạn số ENI có thể attach

Giới Hạn ENI Theo Instance Type:
• t3.micro:  2 ENIs   → tối đa 1 task awsvpc (1 ENI dành cho host)
• t3.medium: 3 ENIs   → tối đa 2 tasks awsvpc
• m5.xlarge: 4 ENIs   → tối đa 3 tasks awsvpc
• c5.4xlarge: 8 ENIs  → tối đa 7 tasks awsvpc

Giải Pháp: ENI Trunking (Gộp ENI)
• Kích hoạt: "awsvpcTrunking" trong account settings
• 1 trunk ENI thêm cho phép gắn thêm nhiều branch ENIs
• m5.xlarge với trunking: có thể chạy 10+ tasks awsvpc
• Yêu cầu: ECS-optimized AMI phiên bản gần đây

Bật ENI Trunking:
aws ecs put-account-setting \
  --name awsvpcTrunking \
  --value enabled
```

### awsvpc Network Configuration

```json
{
  "networkConfiguration": {
    "awsvpcConfiguration": {
      "subnets": [
        "subnet-private-1a",
        "subnet-private-1b",
        "subnet-private-1c"
      ],
      "securityGroups": [
        "sg-ecs-tasks"
      ],
      "assignPublicIp": "DISABLED"
    }
  }
}
```

**assignPublicIp:**
```
DISABLED (Khuyến Nghị Cho Production):
• Tasks chạy trong private subnet
• Không có public IP, không accessible từ internet
• Dùng NAT Gateway để tasks access internet (pull image, call external APIs)
• Traffic từ internet vào qua ALB → tasks

ENABLED (Dev/Test Only):
• Tasks nhận public IP
• Không cần NAT Gateway (tiết kiệm ~$32/tháng per AZ)
• Không an toàn cho production workload
```

---

## 🌉 Bridge Mode — Chế Độ Cầu Nối

### Cơ Chế Bridge Mode

```
EC2 Instance với Bridge Mode:
┌──────────────────────────────────────────────────────────────────┐
│  EC2 Host: eth0 = 10.0.1.5                                       │
│                                                                  │
│  Docker Bridge Network: docker0 = 172.17.0.0/16                 │
│                                                                  │
│  Task A:                                                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Container: 172.17.0.2:8080                              │   │
│  │  Port mapping: 10.0.1.5:32768 → 172.17.0.2:8080         │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  Task B (cùng container port):                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Container: 172.17.0.3:8080                              │   │
│  │  Port mapping: 10.0.1.5:32769 → 172.17.0.3:8080         │   │
│  │  (Không conflict vì dùng random host port khác)          │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### Dynamic Port Mapping Với ALB

```
Với Bridge Mode, ALB cần biết host port (random) của mỗi task:

1. Task register với ALB Target Group:
   Target: 10.0.1.5:32768   ← EC2 IP + random host port
   Target: 10.0.1.6:32770   ← EC2 IP + random host port

2. ALB route traffic → 10.0.1.5:32768 → Container:8080

3. ECS tự động register/deregister khi tasks scale

Yêu cầu Target Group Type = "instance" (không phải "ip")
```

---

## 🏗️ VPC và Subnet Design Cho ECS

### Kiến Trúc VPC Production-Ready

```
VPC: 10.0.0.0/16 (65,536 IPs)
│
├── Public Subnets (Mạng Con Công Khai)
│   ├── subnet-public-1a: 10.0.0.0/24 (256 IPs)
│   ├── subnet-public-1b: 10.0.1.0/24 (256 IPs)
│   └── subnet-public-1c: 10.0.2.0/24 (256 IPs)
│   Dùng cho: Internet Gateway, ALB, NAT Gateway, Bastion Host
│
├── Private App Subnets (Mạng Con Riêng Cho Ứng Dụng)
│   ├── subnet-app-1a: 10.0.10.0/23 (512 IPs)
│   ├── subnet-app-1b: 10.0.12.0/23 (512 IPs)
│   └── subnet-app-1c: 10.0.14.0/23 (512 IPs)
│   Dùng cho: ECS Tasks (Fargate/EC2), Lambda Functions
│   Route: private route table → NAT Gateway (internet access)
│
└── Private Data Subnets (Mạng Con Riêng Cho Dữ Liệu)
    ├── subnet-data-1a: 10.0.20.0/24 (256 IPs)
    ├── subnet-data-1b: 10.0.21.0/24 (256 IPs)
    └── subnet-data-1c: 10.0.22.0/24 (256 IPs)
    Dùng cho: RDS, ElastiCache, OpenSearch
    Route: isolated, không có internet access
```

### IP Address Planning Cho ECS

```
Mỗi ECS Task (awsvpc mode) dùng 1 IP từ subnet.
Với 3 tasks và high-traffic scaling, cần đủ IP.

Tính Toán:
• Max tasks: 100 (peak, khi scale out)
• Mỗi task: 1 IP
• Buffer: 20% dự phòng
• Cần: 100 × 1.2 = 120 IPs minimum

Subnet Size:
• /24 = 251 usable IPs → OK cho 120 tasks
• /23 = 507 usable IPs → OK cho 300 tasks
• /22 = 1019 usable IPs → OK cho 600 tasks

Lưu Ý: AWS giữ lại 5 IPs đầu mỗi subnet (network, router, DNS, future, broadcast)
```

---

## 🔒 Security Groups Cho ECS

### Security Group Hierarchy — Phân Cấp Security Group

```
sg-alb (ALB Security Group):
  Inbound:
  • 80 (HTTP) from 0.0.0.0/0 (internet)
  • 443 (HTTPS) from 0.0.0.0/0 (internet)
  Outbound:
  • 8080 to sg-ecs-web (ALB giao tiếp với ECS tasks)

sg-ecs-web (ECS Web API Tasks Security Group):
  Inbound:
  • 8080 from sg-alb (chỉ nhận traffic từ ALB)
  Outbound:
  • 5432 to sg-rds (gọi PostgreSQL)
  • 6379 to sg-redis (gọi Redis)
  • 443 to 0.0.0.0/0 (gọi external APIs, ECR, Secrets Manager)

sg-ecs-worker (ECS Worker Tasks Security Group):
  Inbound:
  • (không cần inbound — worker pull từ SQS)
  Outbound:
  • 5432 to sg-rds
  • 443 to 0.0.0.0/0

sg-rds (RDS Security Group):
  Inbound:
  • 5432 from sg-ecs-web
  • 5432 from sg-ecs-worker
  Outbound:
  • (none needed)

Nguyên Tắc: Dùng Security Group Reference (tham chiếu SG) thay vì
            IP ranges. Khi tasks scale và IP thay đổi, rules vẫn đúng.
```

### VPC Endpoints — Điểm Cuối VPC

```
Vấn Đề: ECS tasks trong private subnet cần access ECR, Secrets Manager,
         CloudWatch → Traffic đi qua NAT Gateway → tốn tiền + latency

Giải Pháp: VPC Endpoints (Interface Endpoints)
• Traffic đi trong AWS network, không qua internet
• Bảo mật hơn (không ra internet)
• Giảm chi phí NAT Gateway

VPC Endpoints Cần Thiết Cho ECS:
• com.amazonaws.REGION.ecr.api      ← ECR API
• com.amazonaws.REGION.ecr.dkr      ← Docker pull từ ECR
• com.amazonaws.REGION.s3           ← S3 (ECR layer storage)
• com.amazonaws.REGION.logs         ← CloudWatch Logs
• com.amazonaws.REGION.secretsmanager ← Secrets Manager
• com.amazonaws.REGION.ssm          ← Systems Manager (ECS Exec)
• com.amazonaws.REGION.ssmmessages  ← ECS Exec

Chi Phí: ~$0.01/hour/endpoint/AZ (khoảng $7/tháng/endpoint)
Tiết Kiệm: NAT Gateway data processing $0.045/GB → $0 với VPC Endpoint
```

---

## 🔍 Service Discovery Với AWS Cloud Map

### Cơ Chế Service Discovery

```
AWS Cloud Map + Route 53 Private DNS:

1. Tạo Cloud Map namespace: production.local

2. Đăng ký ECS Service:
   Service Name: payment-service
   Namespace: production.local
   → DNS Record: payment-service.production.local

3. Mỗi Task khi start:
   → Đăng ký IP vào Cloud Map
   → Route 53 record: payment-service.production.local → [10.0.1.10, 10.0.2.15, ...]

4. Service A gọi Service B:
   http://payment-service.production.local:8080
   → DNS resolve → 10.0.2.15 (một trong các IPs)
   → Gọi trực tiếp tới task

5. Khi Task stop:
   → ECS tự deregister IP từ Cloud Map
   → Route 53 xóa IP đó khỏi DNS records
```

### Cấu Hình Service Discovery Trong ECS Service

```json
{
  "serviceRegistries": [
    {
      "registryArn": "arn:aws:servicediscovery:ap-southeast-1:123456789:service/srv-abc123",
      "port": 8080
    }
  ]
}
```

```bash
# Tạo Cloud Map namespace
aws servicediscovery create-private-dns-namespace \
  --name production.local \
  --vpc vpc-xxx

# Tạo Cloud Map service
aws servicediscovery create-service \
  --name payment-service \
  --dns-config '{
    "NamespaceId": "ns-xxx",
    "DnsRecords": [{"Type": "A", "TTL": 10}]
  }' \
  --health-check-custom-config '{"FailureThreshold": 1}'
```

### Nhược Điểm Service Discovery

```
1. DNS Caching (Cache DNS):
   • TTL 10-30 giây → Client có thể cache IP cũ của task đã stop
   • Cần code handle connection failures và retry
   
2. Không có Load Balancing Thông Minh:
   • DNS round-robin đơn giản
   • Không biết task nào "kém healthy" để tránh
   • Không phân phối đều theo load thực tế
   
3. Không Có Metrics:
   • Không biết latency, error rate per-service
   • Khó troubleshoot service-to-service calls
   
4. Không Có Circuit Breaking:
   • Nếu Service B overloaded, Service A vẫn tiếp tục gọi
   • Cần tự implement retry logic, timeout, circuit breaker
   
Kết Luận: Service Discovery phù hợp cho simple microservices hoặc
           khi cần backward compatibility. Với microservices mới,
           dùng ECS Service Connect thay thế.
```

---

## 🔗 ECS Service Connect

### Service Connect Là Gì?

ECS Service Connect (Kết Nối Dịch Vụ) là giải pháp **service mesh nhẹ** (lightweight service mesh) tích hợp sẵn vào ECS, cho phép service-to-service communication với observability tự động mà không cần cấu hình phức tạp như Istio hay AWS App Mesh.

```
Service Connect vs Service Discovery vs App Mesh:

Service Discovery:
• DNS-based, đơn giản
• Không có built-in retries, metrics
• Phù hợp: simple use cases

Service Connect (KHUYẾN NGHỊ CHO ECS MỚI):
• Built-in proxy per task (Envoy-based)
• Automatic metrics (request rate, p50/p99 latency, error rate)
• X-Ray tracing integration
• Health-based routing
• Không cần sidecar container riêng

AWS App Mesh (Không Khuyến Nghị Nữa):
• Full service mesh với Envoy sidecar
• Phức tạp, nhiều cấu hình
• AWS đang chuyển sang Service Connect
```

### Kiến Trúc Service Connect

```
Namespace: production (Cloud Map namespace)

Service A (web-api):          Service B (payment-service):
┌───────────────────┐         ┌───────────────────┐
│ ┌───────────────┐ │         │ ┌───────────────┐ │
│ │  app:8080     │ │         │ │  payment:9090 │ │
│ └───────┬───────┘ │         │ └───────────────┘ │
│         │ localhost│         │        ▲          │
│ ┌───────▼───────┐ │         │ ┌──────┴────────┐ │
│ │ Service       │─┼─────────┼►│ Service       │ │
│ │ Connect Proxy │ │ HTTP    │ │ Connect Proxy │ │
│ │ (Port 8080    │ │ to      │ │ (Port 9090    │ │
│ │  outbound)    │ │ payment │ │  inbound)     │ │
│ └───────────────┘ │ :9090   │ └───────────────┘ │
└───────────────────┘         └───────────────────┘

Code trong app container:
• Gọi: http://payment-service:9090/api/charge
• Proxy handle: service discovery, retries, circuit breaking
• Metrics tự động capture vào CloudWatch
```

### Cấu Hình Service Connect

```bash
# Tạo ECS Cluster với Service Connect namespace
aws ecs create-cluster \
  --cluster-name cluster-production \
  --service-connect-defaults '{"namespace": "production"}'

# Tạo ECS Service với Service Connect
aws ecs create-service \
  --cluster cluster-production \
  --service-name payment-service \
  --task-definition payment-service:3 \
  --desired-count 2 \
  --service-connect-configuration '{
    "enabled": true,
    "namespace": "production",
    "services": [
      {
        "portName": "payment-9090-tcp",
        "clientAliases": [
          {
            "port": 9090,
            "dnsName": "payment-service"
          }
        ],
        "discoveryName": "payment-service-discovery",
        "ingressPortOverride": 9090
      }
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/service-connect-proxy",
        "awslogs-region": "ap-southeast-1",
        "awslogs-stream-prefix": "ecs"
      }
    }
  }'
```

### Service Connect Port Naming

```json
// Trong Task Definition, cần đặt tên cho portMappings
{
  "containerDefinitions": [
    {
      "name": "payment",
      "portMappings": [
        {
          "containerPort": 9090,
          "protocol": "tcp",
          "name": "payment-9090-tcp",
          "appProtocol": "http"
        }
      ]
    }
  ]
}

// appProtocol hỗ trợ:
// "http"  → HTTP/1.1 metrics
// "http2" → HTTP/2 metrics
// "grpc"  → gRPC metrics
```

### Metrics Tự Động Với Service Connect

```
CloudWatch Namespace: AWS/ECS/ManagedScaling

Metrics được thu thập tự động:
• ActiveConnections: số kết nối đang hoạt động
• NewConnections: kết nối mới mỗi phút
• ProcessedBytes: bytes đã xử lý
• RequestCount: số request
• RequestCountPerTarget: request per task
• TargetResponseTime: latency p50, p75, p95, p99
• HTTPCode_Target_2XX_Count: số 2xx responses
• HTTPCode_Target_4XX_Count: số 4xx responses
• HTTPCode_Target_5XX_Count: số 5xx responses

Không cần cấu hình gì thêm — Service Connect tự động thu thập.
```

### Service Connect vs Service Discovery So Sánh

| Tính Năng                     | Service Connect              | Service Discovery (Cloud Map) |
| ----------------------------- | ---------------------------- | ----------------------------- |
| **Cơ Chế**                    | Built-in proxy (Envoy)       | DNS (Route 53)                |
| **Metrics Tự Động**           | ✅ CloudWatch metrics         | ❌ Không                       |
| **X-Ray Tracing**             | ✅ Tích hợp                   | ❌ Cần tự tích hợp             |
| **Health-Based Routing**      | ✅ Proxy bypass unhealthy      | ❌ DNS round-robin             |
| **Retries Tự Động**           | ✅ Proxy retry                | ❌ Cần tự code                 |
| **Circuit Breaking**          | ✅ Envoy circuit breaker       | ❌ Không có                    |
| **Startup Latency**           | Thêm ~1 giây cho proxy init  | Không ảnh hưởng               |
| **Phức Tạp Cấu Hình**         | Trung bình                   | Thấp                          |
| **Non-ECS Services**          | ❌ ECS only                   | ✅ Bất kỳ service nào          |
| **Khuyến Nghị**               | ✅ Cho ECS microservices mới  | Chỉ khi cần non-ECS support   |

---

## ⚖️ Load Balancer Patterns

### ALB (Application Load Balancer) — Cân Bằng Tải Tầng 7

```
Phù Hợp Cho: HTTP/HTTPS, gRPC, WebSocket

ALB → ECS Pattern:
Internet → ALB:443 → Target Group → ECS Tasks (awsvpc mode)

ALB Features Quan Trọng Cho ECS:
1. Path-based routing (định tuyến theo đường dẫn):
   /api/* → Target Group: api-service-tg
   /auth/* → Target Group: auth-service-tg
   /* → Target Group: frontend-tg

2. Host-based routing (định tuyến theo host):
   api.example.com → Target Group: api-service-tg
   admin.example.com → Target Group: admin-service-tg

3. Header-based routing (định tuyến theo header):
   X-API-Version: v2 → Target Group: api-v2-tg
   (dùng cho Canary deployment hoặc versioning)

4. HTTP to HTTPS redirect
5. SSL termination (giải mã TLS tại ALB)
6. WAF integration (Web Application Firewall — Tường Lửa Ứng Dụng)
7. Access logs → S3
```

### NLB (Network Load Balancer) — Cân Bằng Tải Tầng 4

```
Phù Hợp Cho: TCP/UDP/TLS, high throughput, static IP

NLB → ECS Pattern:
Client → NLB:TCP/443 → Target Group → ECS Tasks

Dùng NLB khi:
• Cần static IP (NLB có Elastic IP)
• UDP traffic (game server, video streaming)
• Non-HTTP protocols (MQTT, custom binary protocols)
• Cần preserve client IP
• Extreme low latency (<1ms)
• High throughput (millions of requests/second)
```

### Internal Load Balancer — Cân Bằng Tải Nội Bộ

```
Cho Service-to-Service Communication:

Service A → Internal ALB → Service B

Cấu hình:
• ALB scheme: internal (không có internet-facing IP)
• Subnet: private app subnets
• Security Group: chỉ accept từ ECS task SG của Service A

Khi Dùng Internal ALB Thay Vì Service Connect:
• Khi Service A là non-ECS (Lambda, EC2, on-premise)
• Khi cần ALB path-based routing cho microservices
• Khi cần ALB access logs chi tiết
• Khi team chưa sẵn sàng chuyển sang Service Connect
```

---

## 🔧 ECS Exec — Exec Vào Container Đang Chạy

### ECS Exec Là Gì?

ECS Exec cho phép **exec vào container đang chạy** (kể cả Fargate) mà không cần SSH, thông qua AWS Systems Manager Session Manager.

```
Traditional SSH Access:         ECS Exec:
Developer                       Developer
   │                                │
   │ SSH port 22                    │ HTTPS (qua SSM)
   │                                │
EC2 Instance                    ECS Task (Fargate/EC2)
│ SSH Daemon                    │ SSM Agent (embedded)
│                                │
│ Shell access                  │ Shell trong container
```

### Bật ECS Exec

```bash
# 1. Bật ECS Exec cho Service
aws ecs update-service \
  --cluster cluster-prod \
  --service web-api \
  --enable-execute-command

# 2. Task Role cần policy AmazonSSMManagedInstanceCore (hoặc custom)
{
  "Effect": "Allow",
  "Action": [
    "ssmmessages:CreateControlChannel",
    "ssmmessages:CreateDataChannel",
    "ssmmessages:OpenControlChannel",
    "ssmmessages:OpenDataChannel"
  ],
  "Resource": "*"
}

# 3. Exec vào container
aws ecs execute-command \
  --cluster cluster-prod \
  --task abc123def456 \
  --container web-api \
  --command "/bin/bash" \
  --interactive
```

### Use Cases Cho ECS Exec

```
✓ Debug container không có logs
✓ Kiểm tra environment variables thực tế đang chạy
✓ Test network connectivity từ container
✓ Run one-off commands (xem database records, clear cache)
✓ Troubleshoot container health check failures

⚠️ Không Dùng Cho:
✗ Regular maintenance (dùng SSM Run Command thay thế)
✗ Production debugging thường xuyên (risk contamination)
✗ Chạy migrations (dùng ECS RunTask thay thế)
```

---

## ❓ Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: Tại sao awsvpc là network mode được khuyến nghị cho ECS?**
> awsvpc cấp cho mỗi Task một ENI (Elastic Network Interface — Giao Diện Mạng Co Giãn) riêng với IP riêng trong VPC. Điều này mang lại: (1) **Bảo mật**: Security Group áp dụng ở cấp Task thay vì cấp EC2 host, isolation tốt hơn. (2) **Không port conflict**: mỗi task có IP riêng nên nhiều tasks có thể dùng cùng container port. (3) **VPC-native networking**: Task giao tiếp trực tiếp qua VPC như EC2 instance. (4) **Bắt buộc với Fargate**: Fargate chỉ hỗ trợ awsvpc.

**Q: Service Connect khác Service Discovery ở điểm gì quan trọng nhất?**
> Điểm quan trọng nhất là **observability và reliability tự động**. Service Discovery chỉ là DNS — bạn biết IP của service nhưng không biết latency, error rate, không có automatic retry, không có circuit breaking. Service Connect inject một proxy (Envoy) vào mỗi task, proxy này tự động thu thập metrics (request rate, p99 latency, error count) vào CloudWatch, tích hợp X-Ray tracing, và có thể bypass unhealthy tasks. Với microservices production, Service Connect cho observability tốt hơn rất nhiều.

**Q: VPC Endpoint cho ECS có tác dụng gì?**
> Tasks trong private subnet cần access ECR để pull images, Secrets Manager để lấy secrets, CloudWatch để ghi logs. Bình thường traffic này đi qua NAT Gateway (tốn $0.045/GB và có latency). VPC Endpoint tạo kết nối riêng từ VPC đến AWS service, traffic đi hoàn toàn trong AWS network — không qua internet, không qua NAT Gateway. Lợi ích: (1) giảm chi phí NAT Gateway, (2) bảo mật hơn (traffic không ra internet), (3) latency thấp hơn.

### Câu Hỏi Nâng Cao

**Q: ENI Trunking là gì và khi nào cần thiết?**
> Mỗi EC2 instance có giới hạn số ENI có thể attach (ví dụ t3.medium: 3 ENI). Với awsvpc mode, mỗi task cần 1 ENI, nên EC2 nhỏ chỉ chạy được 1-2 tasks — waste compute. ENI Trunking cho phép attach 1 **trunk ENI** vào instance, trunk ENI này có thể handle traffic cho nhiều **branch ENIs** ảo của các tasks. Kết quả: instance có thể chạy nhiều tasks hơn giới hạn ENI vật lý, tăng mật độ task trên EC2, giảm chi phí. Cần bật qua `aws ecs put-account-setting --name awsvpcTrunking`.

**Q: Tại sao internal ALB đôi khi tốt hơn Service Connect cho service-to-service communication?**
> Có những trường hợp internal ALB phù hợp hơn: (1) **Non-ECS callers** — Lambda, EC2 hay on-premise services không thể dùng Service Connect (ECS-only). (2) **Path-based routing** — khi một service cần route `/api/v1/*` và `/api/v2/*` đến các target groups khác nhau. (3) **ALB features** — khi cần sticky sessions, WAF rules, hay access logs chi tiết. (4) **Gradual migration** — team chưa sẵn sàng chuyển sang Service Connect ngay. Trade-off: internal ALB tốn thêm chi phí ($0.008/LCU-hour) và thêm một hop mạng.

**Q: Làm sao đảm bảo ECS tasks không thể access trực tiếp RDS mà chỉ qua một service cụ thể?**
> Dùng **Security Group chaining** (chuỗi Security Group): RDS Security Group chỉ allow inbound từ `sg-data-service` (Security Group của data-service). Web API Security Group (`sg-web-api`) không có quyền connect RDS trực tiếp. Web API phải gọi qua data-service → data-service gọi RDS. Nếu muốn stricter, kết hợp VPC Network ACL (Access Control List — Danh Sách Kiểm Soát Truy Cập) để block ở subnet level. Không dùng IP-based rules vì tasks scale và IP thay đổi liên tục — Security Group reference bền vững hơn.

---

**Hoàn Thành:** Đây là bài cuối trong series `04-containers-ecs/`.

**Quay Lại:** [README.md](README.md) | **Tiếp Theo:** [05-containers-eks/README.md](../05-containers-eks/README.md) — EKS & Kubernetes trên AWS.
