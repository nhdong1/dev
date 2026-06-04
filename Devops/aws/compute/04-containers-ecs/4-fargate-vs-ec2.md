# Fargate vs EC2 Launch Type — Fargate và EC2: Trade-offs và Hướng Dẫn Chọn

> Hướng dẫn toàn diện so sánh Fargate (Serverless Container — Container Không Máy Chủ) và EC2 (Elastic Compute Cloud) launch type trong Amazon ECS: kiến trúc, chi phí, hiệu suất, bảo mật và khi nào nên chọn loại nào.

## 📚 Mục Lục

1. [Tổng Quan So Sánh](#tổng-quan-so-sánh)
2. [Kiến Trúc Fargate](#kiến-trúc-fargate)
3. [Kiến Trúc EC2 Launch Type](#kiến-trúc-ec2-launch-type)
4. [So Sánh Chi Tiết](#so-sánh-chi-tiết)
5. [Chi Phí — Cost Analysis](#chi-phí--cost-analysis)
6. [Hiệu Suất và Giới Hạn](#hiệu-suất-và-giới-hạn)
7. [Bảo Mật](#bảo-mật)
8. [Capacity Providers — Nhà Cung Cấp Năng Lực](#capacity-providers--nhà-cung-cấp-năng-lực)
9. [Hướng Dẫn Chọn Launch Type](#hướng-dẫn-chọn-launch-type)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🔍 Tổng Quan So Sánh

```
FARGATE                              EC2 LAUNCH TYPE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
"Serverless Container"               "Self-managed Container Host"

• AWS quản lý compute (bạn không     • Bạn quản lý EC2 instances
  thấy EC2 instances)                • Cần Auto Scaling Group cho hosts
                                     
• Trả tiền theo vCPU + Memory        • Trả tiền theo EC2 instance size
  chính xác cho mỗi task               (kể cả lúc không dùng hết)
                                     
• Không cần chọn instance type       • Cần chọn đúng instance type
                                     
• Mỗi task: compute riêng biệt       • Nhiều tasks chia sẻ EC2 host
  (Firecracker micro-VM)               (shared kernel)
                                     
• Không hỗ trợ GPU                   • Hỗ trợ GPU (p3, g4, g5)
  (hạn chế cho ML workload)            Custom AMI, kernel modules
                                     
• Không có persistent local storage  • Có instance store storage
  (chỉ EFS cho shared storage)         (NVMe SSD gắn trực tiếp)
                                     
• Task startup: ~30-60 giây          • Task startup: ~10-20 giây
                                     
• Chi phí cao hơn per-hour           • Chi phí thấp hơn với Reserved/Spot
  nhưng không lãng phí                 nhưng cần quản lý nhiều hơn
```

---

## 🟡 Kiến Trúc Fargate

### Cách Fargate Hoạt Động

```
Khi bạn launch một Fargate Task:

1. ECS Scheduler nhận yêu cầu: "Chạy task với 1 vCPU, 2 GB RAM"

2. AWS Fargate Fleet provisioning:
   • AWS chọn một Fargate Worker (micro-VM host)
   • Tạo Firecracker micro-VM (máy ảo siêu nhẹ) riêng cho task
   • Micro-VM có kernel riêng, hoàn toàn isolated

3. Container startup:
   • Pull image từ ECR (nếu chưa cached)
   • Start containers theo Task Definition
   • Attach ENI (Elastic Network Interface) với IP trong VPC subnet

4. Task đang chạy:
   • Bạn không thể SSH vào Fargate compute
   • Chỉ có thể dùng ECS Exec để exec vào container
   • Compute node transparent với bạn

Fargate Architecture:
┌─────────────────────────────────────────────────┐
│              AWS FARGATE FLEET                   │
│         (Ẩn, AWS-managed, bạn không thấy)        │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │  Firecracker micro-VM cho Task A         │   │
│  │  ┌────────────────────────────────────┐  │   │
│  │  │  Your ECS Task                     │  │   │
│  │  │  ┌────────────┐  ┌──────────────┐  │  │   │
│  │  │  │ Container A│  │ Container B  │  │  │   │
│  │  │  │ (app)      │  │ (sidecar)    │  │  │   │
│  │  │  └────────────┘  └──────────────┘  │  │   │
│  │  └────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────┐   │
│  │  Firecracker micro-VM cho Task B         │   │
│  │  (hoàn toàn độc lập với Task A)          │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### Fargate Storage

```
Ephemeral Storage (Lưu Trữ Tạm Thời — Mặc Định):
• 20 GB free mặc định mỗi task
• Tăng lên tối đa 200 GB: $0.10/GB/tháng
• Dữ liệu mất khi task stop

EFS (Elastic File System) — Persistent Shared Storage:
• Mount vào container qua NFSv4
• Nhiều tasks/services chia sẻ cùng filesystem
• Dữ liệu persistent, survive task restarts
• Chi phí: $0.30/GB/tháng (Standard storage)

Fargate KHÔNG hỗ trợ:
• Instance store (NVMe SSD gắn vật lý)
• EBS volumes (gắn trực tiếp vào instance)
→ Phải dùng EFS nếu cần persistent storage
```

### Fargate Spot — Fargate Giá Rẻ

```
Fargate Spot = Tận dụng Fargate capacity dư thừa của AWS
               Giá thấp hơn 70% so với Fargate on-demand
               Nhưng có thể bị interrupted với 2 phút notice

Khi Nào Dùng Fargate Spot:
✓ Batch processing jobs
✓ CI/CD pipeline runners
✓ Data processing tasks có thể retry
✓ Non-critical background workers
✓ Dev/testing environments

Khi KHÔNG Nên Dùng Fargate Spot:
✗ Web API production service (unacceptable interruption)
✗ Tasks không thể tolerate interruption
✗ Long-running jobs không có checkpoint

Xử Lý Interruption:
• AWS gửi SIGTERM 120 giây trước khi terminate
• App cần handle SIGTERM và checkpoint state nếu có
• Kết hợp với SQS: message visibility timeout > task duration
```

---

## 🔵 Kiến Trúc EC2 Launch Type

### Cách EC2 Launch Type Hoạt Động

```
EC2 Launch Type yêu cầu bạn quản lý EC2 instances:

1. Bạn tạo Auto Scaling Group với ECS-optimized AMI
2. ECS Container Agent chạy trên mỗi EC2 instance
3. Agent đăng ký EC2 vào ECS Cluster
4. ECS Scheduler đặt Tasks lên EC2 instances có đủ tài nguyên
5. Nhiều Tasks chạy trên cùng một EC2 instance

EC2 Launch Type Architecture:
┌──────────────────────────────────────────────────────────────┐
│                     ECS CLUSTER                              │
│                                                              │
│  EC2 Instance (t3.xlarge: 4 vCPU, 16 GB RAM)                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  ECS Container Agent  (chiếm ~256 MB RAM reserved)     │  │
│  │  Docker Daemon                                         │  │
│  │                                                        │  │
│  │  Task A          Task B           Task C              │  │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐         │  │
│  │  │ Container │  │ Container │  │ Container │         │  │
│  │  │ 1vCPU,2GB │  │ 1vCPU,4GB │  │ 2vCPU,8GB │         │  │
│  │  └───────────┘  └───────────┘  └───────────┘         │  │
│  │                                                        │  │
│  │  Shared: EC2 kernel, network interface, storage       │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  EC2 Instance 2 (t3.xlarge)                                 │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Task D, Task E, ...                                   │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### Quản Lý EC2 Hosts

```
Bạn chịu trách nhiệm:
1. Chọn instance type phù hợp:
   • CPU-bound: c5, c6i (Compute Optimized)
   • Memory-bound: r5, r6i (Memory Optimized)
   • GPU: g4dn, p3 (Accelerated Computing)
   • General purpose: m5, t3 (Tiết kiệm chi phí)

2. ECS-optimized AMI:
   • Amazon cung cấp AMI với ECS Agent + Docker pre-installed
   • Cần theo dõi và update AMI định kỳ
   • Có thể tự build custom AMI nếu cần thêm packages

3. Instance Patching (Vá Lỗi):
   • Dùng AWS Systems Manager Patch Manager
   • Instance refresh trong Auto Scaling Group

4. Cluster capacity management:
   • Capacity Providers với Auto Scaling Group
   • Đảm bảo đủ EC2 hosts khi ECS cần scale tasks
```

### Spot Instances Với EC2 Launch Type

```
Kết Hợp EC2 Spot + ECS = Chi Phí Thấp Nhất:

Capacity Provider Strategy:
• 70% tasks → FARGATE (ổn định)
• 30% tasks → EC2 Spot (tiết kiệm)

Hoặc:
• Base: 2 On-Demand EC2 (luôn sẵn sàng)
• Scale-out: EC2 Spot (khi cần burst capacity)

Xử Lý Spot Interruption:
1. EC2 gửi interruption notice 2 phút trước
2. EC2 Spot Interruption Handler (Lambda) nhận EC2 event
3. Gọi ECS DrainContainer để gracefully drain tasks
4. Tasks di chuyển sang EC2 On-Demand khác hoặc Fargate
```

---

## 📊 So Sánh Chi Tiết

### Ma Trận So Sánh Đầy Đủ

| Tiêu Chí                       | Fargate                    | EC2 Launch Type           |
| ------------------------------ | -------------------------- | ------------------------- |
| **Quản Lý Server**             | AWS quản lý                | Bạn quản lý               |
| **Provisioning Time**          | ~30-60 giây                | ~10-20 giây               |
| **Cách Tính Chi Phí**          | vCPU + Memory per second   | EC2 instance per hour     |
| **GPU Support**                | ❌ Không                   | ✅ Có (g4dn, p3, p4)      |
| **Instance Store**             | ❌ Không                   | ✅ Có                     |
| **Custom Kernel / AMI**        | ❌ Không                   | ✅ Có                     |
| **SSH vào Host**               | ❌ Không thể               | ✅ Có thể                 |
| **Isolation Level**            | Mạnh (micro-VM per task)   | Trung bình (shared host)  |
| **Spot Pricing**               | ✅ Fargate Spot (~70% off) | ✅ EC2 Spot (~90% off)    |
| **Reserved Pricing**           | ✅ Compute Savings Plans   | ✅ Reserved Instances      |
| **Density (tasks per host)**   | N/A                        | Nhiều tasks/instance      |
| **Cold Start**                 | ~30-60 giây                | ~10-20 giây               |
| **Windows Container**          | ✅ Hỗ trợ                  | ✅ Hỗ trợ                 |
| **ARM (Graviton)**             | ✅ Fargate ARM              | ✅ EC2 Graviton            |
| **Ops Overhead**               | Thấp                       | Cao (patching, AMI, ASG)  |

---

## 💰 Chi Phí — Cost Analysis

### Fargate Pricing (Giá Fargate)

```
Fargate tính phí theo:
• vCPU per second: $0.04048/vCPU-hour
• Memory per second: $0.004445/GB-hour

Ví Dụ Task: 1 vCPU, 2 GB RAM, chạy 24/7
Chi phí/tháng = (0.04048 × 1 × 24 × 30) + (0.004445 × 2 × 24 × 30)
              = $29.15 + $6.40
              = $35.55/task/tháng

3 tasks: $106.65/tháng
```

### EC2 Pricing (Giá EC2)

```
Ví Dụ: t3.medium (2 vCPU, 4 GB RAM)
• On-Demand: ~$0.0416/hour = $30/tháng
• Reserved 1-year: ~$0.026/hour = $18.7/tháng
• Spot: ~$0.013/hour = $9.4/tháng

Chạy 4 tasks (1 vCPU, 2 GB each) trên 2× t3.xlarge:
• t3.xlarge On-Demand: $0.1664/hour × 2 = $0.33/hour = $240/tháng
• vs Fargate: 4 tasks × $35.55 = $142.2/tháng

→ Fargate rẻ hơn khi EC2 utilization thấp
→ EC2 rẻ hơn khi EC2 được tận dụng gần 100%
```

### Khi Nào Fargate Rẻ Hơn

```
Fargate tiết kiệm hơn EC2 khi:

1. Spiky/unpredictable workload:
   • EC2 phải provision cho peak → wasteful ngoài peak
   • Fargate chỉ trả cho thời gian task thực sự chạy

2. Nhiều microservices nhỏ:
   • Mỗi service ít tasks (1-2 tasks)
   • EC2 instance chạy không tận dụng hết
   • Fargate cost granular theo actual usage

3. Dev/Staging environments:
   • Không chạy 24/7, chỉ chạy trong business hours
   • Fargate tắt khi không cần, không tốn tiền

Break-Even Analysis:
• EC2 On-Demand thường hòa vốn khi EC2 utilization > 60-70%
• EC2 Reserved hòa vốn khi EC2 utilization > 40-50%
• EC2 Spot thường rẻ hơn Fargate Spot nếu quản lý được
```

---

## ⚡ Hiệu Suất và Giới Hạn

### Task Startup Time — Thời Gian Khởi Động

```
Fargate Task Startup:
1. AWS provision micro-VM: ~10-15 giây
2. Pull image (nếu chưa cached): ~15-30 giây
3. Container startup: ~5-10 giây
Total: 30-60 giây (hoặc hơn với large images)

EC2 Task Startup (image đã có trên host):
1. Docker run container: ~1-5 giây
2. Container startup: ~5-10 giây
Total: 10-20 giây

Tác Động:
• EC2 nhanh hơn khi cần scale out gấp
• Fargate cần predict và scale out sớm hơn (proactive scaling)
• Dùng Scheduled Scaling trước giờ cao điểm để giảm tác động
```

### Fargate Task Size Limits

```
Fargate CPU Limits:
• Min: 0.25 vCPU
• Max: 16 vCPU

Fargate Memory Limits:
• Min: 0.5 GB
• Max: 120 GB

Fargate Storage:
• Ephemeral: 20 GB mặc định, tối đa 200 GB

EC2 Task Limits:
• Phụ thuộc vào EC2 instance type
• p4d.24xlarge: 96 vCPU, 1152 GB RAM, 8× NVIDIA A100 GPU
• Không giới hạn storage (gắn nhiều EBS volumes)
```

### Network Performance — Hiệu Suất Mạng

```
Fargate:
• Bandwidth tỷ lệ theo vCPU allocated
• 0.25 vCPU: baseline 0.5 Gbps
• 4 vCPU+: up to 40 Gbps
• Enhanced Networking mặc định (ENA)

EC2:
• Phụ thuộc instance type
• c5n.18xlarge: 100 Gbps
• Placement Group (Cluster) cho low-latency HPC
• Jumbo frames (9000 MTU) nếu cần

Trong Thực Tế:
• Hầu hết web applications: Fargate bandwidth đủ dùng
• HPC, data-intensive: EC2 với instance family phù hợp
```

---

## 🔒 Bảo Mật

### Isolation Model — Mô Hình Cách Ly

```
Fargate Isolation (MẠNH HƠN):
┌─────────────────────────────────────────────────────┐
│  Task A của bạn          Task B của bạn              │
│  ┌─────────────────┐     ┌─────────────────┐        │
│  │ Firecracker VM  │     │ Firecracker VM  │        │
│  │ Kernel riêng    │     │ Kernel riêng    │        │
│  │ Memory riêng    │     │ Memory riêng    │        │
│  └─────────────────┘     └─────────────────┘        │
│  Không thể ảnh hưởng lẫn nhau (hypervisor boundary) │
└─────────────────────────────────────────────────────┘

EC2 Isolation (YẾU HƠN):
┌─────────────────────────────────────────────────────┐
│  EC2 Instance của bạn                               │
│  ┌──────────────────────────────────────────────┐   │
│  │  Task A      Task B      Task C              │   │
│  │  Container   Container   Container           │   │
│  │  ┌───────┐   ┌───────┐   ┌───────┐           │   │
│  │  │ app   │   │ app   │   │ app   │           │   │
│  │  └───────┘   └───────┘   └───────┘           │   │
│  │  Shared Linux kernel, shared memory bus      │   │
│  └──────────────────────────────────────────────┘   │
│  Container escape → ảnh hưởng tasks khác trên host  │
└─────────────────────────────────────────────────────┘
```

### Security Best Practices

```
Fargate:
• Không có SSH access → attack surface nhỏ hơn
• Task role per service (least privilege)
• Secrets từ Secrets Manager (không hardcode)
• readonlyRootFilesystem: true
• User non-root (user: "1000")
• VPC security groups cấp task level

EC2:
• Tất cả Fargate practices +
• SSM Session Manager thay vì SSH (không mở port 22)
• IMDSv2 bắt buộc (ngăn SSRF attacks)
• Patch EC2 instances thường xuyên
• Amazon Inspector cho vulnerability scanning
• aws ec2 modify-instance-metadata-options --http-tokens required
```

---

## 🏭 Capacity Providers — Nhà Cung Cấp Năng Lực

### Capacity Provider Là Gì?

Capacity Provider là tầng abstraction giữa ECS Service và compute infrastructure, cho phép kết hợp nhiều loại compute trong một cluster.

```
ECS Cluster
├── Capacity Provider: FARGATE          ← On-demand Fargate
├── Capacity Provider: FARGATE_SPOT     ← Spot Fargate
└── Capacity Provider: my-ec2-asg       ← EC2 Auto Scaling Group

ECS Service A Strategy:
  70% FARGATE (base) + 30% FARGATE_SPOT (burst)

ECS Service B Strategy:
  100% my-ec2-asg (EC2 only, muốn tiết kiệm)
```

### Mixed Strategy — Chiến Lược Hỗn Hợp

```json
{
  "capacityProviderStrategy": [
    {
      "capacityProvider": "FARGATE",
      "weight": 1,
      "base": 2
    },
    {
      "capacityProvider": "FARGATE_SPOT",
      "weight": 4,
      "base": 0
    }
  ]
}
```

**Giải Thích:**
- `base: 2`: Luôn chạy ít nhất 2 tasks trên FARGATE (ổn định)
- `weight`: Khi scale ra, tỷ lệ phân bổ = 1:4 (FARGATE:FARGATE_SPOT)
- Ví dụ 10 tasks tổng: 2 FARGATE (base) + 8 FARGATE_SPOT (scale-out)

### Cluster Auto Scaling Với EC2 Capacity Provider

```
EC2 Capacity Provider + Managed Scaling:

Khi ECS cần thêm tasks nhưng không đủ EC2:
1. ECS Capacity Provider nhận thấy tasks đang PENDING
2. Gửi tín hiệu scale-out đến Auto Scaling Group
3. ASG launch EC2 instances mới
4. Container Agent đăng ký instances vào cluster
5. ECS Scheduler đặt PENDING tasks lên instances mới

Managed Termination Protection:
• Ngăn ASG terminate EC2 instance đang chạy tasks
• Đợi tasks drain xong trước khi terminate
• Tương tự như scale-in protection nhưng ở cấp EC2
```

---

## 🎯 Hướng Dẫn Chọn Launch Type

### Decision Tree — Cây Quyết Định

```
Bạn cần chạy container trên ECS. Chọn launch type nào?

START
  │
  ▼
Cần GPU (NVIDIA/AMD)?
  ├── Có → EC2 Launch Type (g4dn, p3, p4d)
  └── Không → tiếp tục
          │
          ▼
       Cần custom kernel module hoặc privileged mode?
          ├── Có → EC2 Launch Type
          └── Không → tiếp tục
                  │
                  ▼
               Cần instance store (NVMe SSD) cho I/O cao?
                  ├── Có → EC2 Launch Type (i3, i4i)
                  └── Không → tiếp tục
                          │
                          ▼
                       Team có kinh nghiệm quản lý EC2 không?
                       Workload có thể dự đoán/cố định không?
                       Chi phí có phải ưu tiên hàng đầu?
                          ├── Tất cả Có → EC2 với Reserved Instances
                          └── Khác → tiếp tục
                                  │
                                  ▼
                               → FARGATE (khuyến nghị mặc định)
                               → Xem xét thêm Fargate Spot
                                 cho non-critical workloads
```

### Use Case Guide — Hướng Dẫn Theo Use Case

```
Fargate (Ưu Tiên Mặc Định):
✓ Web APIs và microservices
✓ Background workers (non-GPU)
✓ Event-driven processing (SQS consumers)
✓ Dev/Staging environments
✓ Batch jobs không cần GPU
✓ Team nhỏ, ít DevOps resources
✓ Startup/early stage (iterate nhanh, ít ops)

EC2 (Khi Có Lý Do Cụ Thể):
✓ Machine Learning inference/training (cần GPU)
✓ Real-time data processing cần throughput cao (i3en)
✓ Games với custom networking
✓ Legacy apps cần system-level access
✓ Large, stable workload cần tối ưu chi phí tối đa
✓ HPC (High-Performance Computing — Điện Toán Hiệu Suất Cao)

Kết Hợp Cả Hai (Mixed Capacity Provider):
✓ Core service: Fargate on-demand (reliability)
✓ Scale-out: Fargate Spot hoặc EC2 Spot (cost saving)
✓ Batch processing: EC2 Spot với large instances
```

---

## ❓ Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: Fargate và EC2 launch type khác nhau ở điểm gì cơ bản nhất?**
> Sự khác biệt cốt lõi là **ai chịu trách nhiệm quản lý compute infrastructure**. Với Fargate, AWS provision và quản lý compute — bạn chỉ định nghĩa Task và trả tiền theo vCPU/Memory thực tế dùng. Với EC2, bạn tự provision và quản lý EC2 instances, cần Auto Scaling Group, Container Agent, patching, AMI updates, và trả tiền cho EC2 instance kể cả khi không dùng hết.

**Q: Khi nào chắc chắn phải chọn EC2 thay vì Fargate?**
> Ba trường hợp chắc chắn phải dùng EC2: (1) Cần **GPU** (ML training/inference, graphics rendering) — Fargate không hỗ trợ GPU. (2) Cần **instance store** (NVMe SSD gắn trực tiếp) cho I/O throughput cực cao — Fargate chỉ có ephemeral storage giới hạn. (3) Cần **custom kernel modules** hoặc privileged mode — Fargate không cho phép.

**Q: Fargate có bao giờ rẻ hơn EC2 không?**
> Có. Fargate rẻ hơn khi: (1) Workload spiky — EC2 phải provision cho peak nhưng hầu hết thời gian idle. (2) Nhiều microservices nhỏ — mỗi service dùng ít tài nguyên, EC2 instance không tận dụng hết. (3) Dev/Staging không chạy 24/7 — Fargate chỉ tốn tiền khi task đang chạy. Khi EC2 utilization > 60-70%, EC2 On-Demand bắt đầu rẻ hơn Fargate.

### Câu Hỏi Nâng Cao

**Q: Firecracker trong Fargate là gì và tại sao quan trọng về bảo mật?**
> Firecracker là công nghệ **micro-VM** (máy ảo siêu nhẹ) do AWS phát triển và dùng cho Fargate. Mỗi Fargate task chạy trong micro-VM riêng với kernel riêng, isolated hoàn toàn qua hypervisor boundary. Ngay cả khi attacker escape được container, chúng chỉ có thể access micro-VM của task đó, không thể ảnh hưởng tasks khác hay host. Với EC2 launch type, nhiều tasks chia sẻ kernel — container escape nguy hiểm hơn.

**Q: Capacity Provider Strategy kết hợp FARGATE và FARGATE_SPOT như thế nào trong production?**
> Chiến lược hiệu quả: `base: 2, FARGATE` + `weight: 4, FARGATE_SPOT`. Ý nghĩa: luôn có 2 tasks on-demand FARGATE (không bị interrupt), khi scale ra, 80% tasks mới là FARGATE_SPOT (tiết kiệm ~70% chi phí). Application cần handle SIGTERM gracefully với 120 giây timeout. Dùng cho stateless services — nếu một Spot task bị interrupt, 2 on-demand tasks vẫn serving traffic trong khi ECS launch replacement.

**Q: Tại sao startup time của Fargate chậm hơn EC2 và cách giảm thiểu?**
> Fargate chậm hơn vì phải provision micro-VM (~15 giây) và pull image (~15-30 giây). EC2 chỉ cần Docker run vì image thường đã cached trên host. Cách giảm thiểu: (1) **Proactive scaling** — dùng Scheduled Scaling scale out trước giờ cao điểm. (2) **Tối ưu image size** — smaller image pull nhanh hơn. (3) **ECR trong cùng region** — tránh cross-region pull. (4) **Target tracking với low cooldown** — scale out sớm trước khi quá tải. (5) **Desired count tối thiểu đủ cao** — không scale về 0 nếu có cold start concern.

---

**Tiếp Theo:** [5-ecs-networking.md](5-ecs-networking.md) — ECS Networking: awsvpc mode, Service Connect, Service Discovery.
