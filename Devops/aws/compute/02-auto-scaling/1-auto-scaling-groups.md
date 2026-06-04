# 📦 Auto Scaling Groups — Nhóm Tự Động Co Giãn

> **ASG — Auto Scaling Group** là tập hợp EC2 instance được quản lý như một đơn vị với cấu hình scaling chung. ASG tự động duy trì số lượng instance mong muốn, thay thế instance lỗi, và phân phối instance trên nhiều AZ (Availability Zone — Vùng Khả Dụng).

---

## 📚 Mục Lục

1. [Cấu Hình Capacity](#cấu-hình-capacity)
2. [Health Checks — Kiểm Tra Sức Khỏe](#health-checks)
3. [Multi-AZ Distribution — Phân Phối Đa Vùng](#multi-az-distribution)
4. [Vòng Đời Instance Trong ASG](#vòng-đời-instance-trong-asg)
5. [Termination Policies — Chính Sách Xóa Instance](#termination-policies)
6. [ASG Với Load Balancer](#asg-với-load-balancer)
7. [Tạo ASG Bằng AWS CLI](#tạo-asg-bằng-aws-cli)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 📐 Cấu Hình Capacity

### Ba Thông Số Quan Trọng Nhất

```
┌─────────────────────────────────────────────────────────────┐
│                   ASG Capacity Settings                     │
│                                                             │
│  Minimum Capacity (Năng Lực Tối Thiểu)      = 2           │
│  ├── Luôn chạy ít nhất 2 instance                          │
│  ├── Đảm bảo High Availability                             │
│  └── ASG sẽ launch instance mới nếu có instance bị xóa    │
│                                                             │
│  Desired Capacity (Năng Lực Mong Muốn)      = 4           │
│  ├── Số instance ASG đang cố duy trì tại thời điểm này    │
│  ├── Scaling policies thay đổi giá trị này                │
│  └── Phải nằm trong khoảng [min, max]                     │
│                                                             │
│  Maximum Capacity (Năng Lực Tối Đa)         = 10          │
│  ├── Giới hạn trên, ASG không tạo quá số này              │
│  ├── Bảo vệ khỏi chi phí ngoài kiểm soát                 │
│  └── Cần đủ lớn để xử lý peak traffic                    │
└─────────────────────────────────────────────────────────────┘
```

### Quy Tắc Capacity

```
Min ≤ Desired ≤ Max

Ví dụ hợp lệ:   Min=2, Desired=4, Max=10  ✅
Ví dụ không hợp lệ: Min=5, Desired=3, Max=10  ❌ (Desired < Min)
```

### Khi Nào Desired Thay Đổi?

| Nguyên Nhân                          | Ví Dụ                                         |
| ------------------------------------- | --------------------------------------------- |
| **Scaling Policy triggered**          | CPU > 70% → Desired tăng từ 4 lên 6          |
| **Manual adjustment** (thủ công)     | Operator set Desired = 8 trước event          |
| **Scheduled action** (lịch trình)    | Mỗi thứ 2 lúc 8h: set Desired = 10          |
| **Instance terminated** (instance lỗi)| 1 instance die → ASG launch thêm 1 để bù    |

---

## 🏥 Health Checks

### Hai Loại Health Check

#### 1. EC2 Health Check (Mặc Định)

AWS kiểm tra trạng thái hệ thống và instance:

```
Status Check Types:
├── System Status Check — Kiểm tra phần cứng AWS bên dưới
│   └── Fail: lỗi nguồn điện, mạng vật lý, phần cứng host
└── Instance Status Check — Kiểm tra OS và network instance
    └── Fail: OS crash, cấu hình mạng sai, OOM killer

ASG chỉ replace instance khi:
  Instance status = "impaired" (bị suy giảm)
  Không phải khi "stopped" hay "terminated" thông thường
```

#### 2. ELB Health Check — Kiểm Tra Từ Load Balancer

Yêu cầu HTTP/HTTPS đến endpoint health check của ứng dụng:

```
GET /health HTTP/1.1
Host: instance-ip:8080

Response 200 OK → Instance HEALTHY ✅
Response 500    → Instance UNHEALTHY ❌ → ASG terminate & replace
Timeout (>30s)  → Instance UNHEALTHY ❌ → ASG terminate & replace
```

**Ưu điểm ELB Health Check so với EC2 Health Check:**
- Phát hiện ứng dụng bị crash dù OS vẫn chạy
- Phát hiện ứng dụng bị deadlock (treo)
- Phù hợp hơn cho production workload

### Cấu Hình Health Check

```json
{
  "HealthCheckType": "ELB",
  "HealthCheckGracePeriod": 300
}
```

**Health Check Grace Period (Thời Gian Ân Hạn):** Số giây ASG chờ sau khi instance launch trước khi bắt đầu kiểm tra health. Cần đủ lớn để ứng dụng bootstrap (khởi tạo) hoàn tất. Thường đặt = thời gian boot + thời gian app start.

### Luồng Xử Lý Instance Unhealthy

```
Instance bị đánh dấu Unhealthy
         │
         ▼
ASG chờ Grace Period (nếu còn)
         │
         ▼
ASG terminate instance unhealthy
         │
         ▼
ASG launch instance mới (theo Launch Template)
         │
         ▼
Instance mới pass health check
         │
         ▼
ALB bắt đầu route traffic đến instance mới
```

---

## 🌍 Multi-AZ Distribution

### Tại Sao Cần Nhiều AZ?

```
Không có Multi-AZ:
  AZ-a: [EC2][EC2][EC2][EC2]    AZ-b: (trống)
  → AZ-a bị sự cố → TOÀN BỘ hệ thống down ❌

Có Multi-AZ:
  AZ-a: [EC2][EC2]    AZ-b: [EC2][EC2]
  → AZ-a bị sự cố → AZ-b vẫn phục vụ 50% capacity ✅
```

### Cách ASG Phân Phối Instance

ASG sử dụng **rebalancing** (cân bằng lại) để phân phối đều instance trên các AZ:

```
Trạng thái ban đầu (mất cân bằng):
  AZ-a: [EC2][EC2][EC2]    AZ-b: [EC2]

ASG Rebalancing:
  1. Launch 1 instance ở AZ-b (ưu tiên AZ thiếu)
  2. Sau khi AZ-b instance healthy → terminate 1 instance ở AZ-a

Trạng thái sau rebalancing (cân bằng):
  AZ-a: [EC2][EC2]    AZ-b: [EC2][EC2]
```

**Rebalancing được kích hoạt khi:**
- Thêm AZ mới vào ASG
- AZ bị outage và phục hồi lại
- Instance bị terminate thủ công
- Spot Instance bị reclaim (thu hồi)

### Availability Zone Rebalancing vs Scaling

```
Scaling (thêm instance): Desired tăng → Launch instance ở AZ có ít instance nhất
Rebalancing:             Desired không đổi → Launch ở AZ thiếu, terminate ở AZ dư
```

---

## 🔄 Vòng Đời Instance Trong ASG

```
                    ┌─────────────────────────────────────┐
                    │         EC2 Instance Lifecycle       │
                    │         (Vòng Đời Instance)          │
                    └─────────────────────────────────────┘

Pending ──────────────────────────────────────────────────────┐
  │  Instance đang khởi động                                  │
  │  Lifecycle Hook: "EC2_INSTANCE_LAUNCHING" (nếu có)       │
  ▼                                                           │
Pending:Wait (nếu có Lifecycle Hook)                         │
  │  Chờ custom action (ví dụ: đăng ký vào Consul)          │
  ▼                                                           │
Pending:Proceed                                               │
  │                                                           │
  ▼                                                           │
InService ◄───────────────────────────────────────────────────┘
  │  Instance đang phục vụ traffic
  │  Health checks đang chạy
  │
  ├──── Scale In → Terminating
  ├──── Health Check Fail → Terminating
  └──── Manual terminate → Terminating
  
Terminating
  │  Instance đang được xóa
  │  Lifecycle Hook: "EC2_INSTANCE_TERMINATING" (nếu có)
  ▼
Terminating:Wait (nếu có Lifecycle Hook)
  │  Chờ custom action (ví dụ: drain connections, backup data)
  ▼
Terminating:Proceed
  │
  ▼
Terminated ──── Instance đã bị xóa hoàn toàn


Trạng Thái Đặc Biệt:
Standby ──── Instance tạm thời offline để maintenance
Detached ─── Instance tách khỏi ASG (vẫn chạy nhưng không managed)
```

---

## 🗑️ Termination Policies — Chính Sách Xóa Instance

Khi scale in (thu nhỏ), ASG cần quyết định terminate instance nào. Termination Policy xác định thứ tự ưu tiên:

### Các Loại Termination Policy

| Policy                               | Mô Tả                                                           |
| ------------------------------------- | --------------------------------------------------------------- |
| **Default** (mặc định)               | Cân bằng AZ trước, rồi xóa instance cũ nhất (oldest)          |
| **OldestInstance**                   | Xóa instance có tuổi thọ lớn nhất trước                       |
| **NewestInstance**                   | Xóa instance mới nhất trước (testing rolling update)           |
| **OldestLaunchTemplate**             | Xóa instance dùng Launch Template cũ nhất trước                |
| **OldestLaunchConfiguration**        | Xóa instance dùng Launch Configuration cũ nhất                |
| **ClosestToNextInstanceHour**        | Xóa instance gần kết thúc giờ billing nhất (tiết kiệm chi phí)|
| **AllocationStrategy**               | Dựa trên chiến lược phân bổ (dùng cho Mixed Instance)         |
| **Custom Lambda** (tùy chỉnh)        | Gọi Lambda function để quyết định instance nào bị terminate    |

### Default Termination Policy — Thuật Toán Chi Tiết

```
Khi cần terminate 1 instance:

Bước 1: Tìm AZ có nhiều instance nhất
  AZ-a: 3 instances    AZ-b: 1 instance
  → Chọn AZ-a

Bước 2: Trong AZ-a, xem có instance nào dùng Launch Configuration cũ không?
  → Nếu có: terminate instance đó (ưu tiên cập nhật)

Bước 3: Nếu không, chọn instance closest to next billing hour (gần kết thúc giờ billing nhất)
  → Tiết kiệm chi phí On-Demand

Bước 4: Nếu vẫn tie (bằng nhau): chọn ngẫu nhiên
```

---

## ⚖️ ASG Với Load Balancer

### Tích Hợp ALB + ASG

```
                    ┌──────────────────────────┐
    Users           │  Application Load Balancer│
    ─────────►      │  (ALB — Cân Bằng Tải)   │
                    │  Target Group: web-tg     │
                    └────────────┬─────────────┘
                                 │ Route to healthy targets
              ┌──────────────────┼──────────────────┐
              │                  │                  │
    ┌─────────▼──────┐ ┌────────▼───────┐ ┌────────▼───────┐
    │   EC2 Instance  │ │  EC2 Instance  │ │  EC2 Instance  │
    │   (InService)   │ │  (InService)   │ │  (InService)   │
    └─────────────────┘ └────────────────┘ └────────────────┘
              └──────────────────┴──────────────────┘
                         Auto Scaling Group
                      (Managed together as unit)
```

### Attach ASG Vào Target Group

```bash
aws autoscaling attach-load-balancer-target-groups \
  --auto-scaling-group-name my-asg \
  --target-group-arns arn:aws:elasticloadbalancing:us-east-1:123456789:targetgroup/my-tg/abc123
```

### Connection Draining — Drain Kết Nối Trước Khi Terminate

Khi instance chuẩn bị bị terminate, ALB không gửi request mới và chờ các request đang xử lý hoàn thành:

```
Instance sắp bị terminate:
  1. ALB đánh dấu instance "draining" (đang drain)
  2. ALB ngừng gửi request MỚI đến instance này
  3. ALB chờ request ĐANG XỬ LÝ hoàn thành
  4. Timeout (mặc định 300s) → ASG terminate instance dù còn request
  5. Instance bị terminate hoàn toàn
```

---

## 💻 Tạo ASG Bằng AWS CLI

### Bước 1: Tạo Launch Template

```bash
aws ec2 create-launch-template \
  --launch-template-name "web-server-lt" \
  --version-description "Initial version" \
  --launch-template-data '{
    "ImageId": "ami-0abcdef1234567890",
    "InstanceType": "t3.medium",
    "SecurityGroupIds": ["sg-0123456789abcdef0"],
    "IamInstanceProfile": {
      "Name": "WebServerInstanceProfile"
    },
    "UserData": "IyEvYmluL2Jhc2gKeXVtIHVwZGF0ZSAteQo="
  }'
```

### Bước 2: Tạo Auto Scaling Group

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "web-servers-asg" \
  --launch-template "LaunchTemplateName=web-server-lt,Version=$Latest" \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 4 \
  --vpc-zone-identifier "subnet-111aaa,subnet-222bbb,subnet-333ccc" \
  --health-check-type "ELB" \
  --health-check-grace-period 300 \
  --target-group-arns "arn:aws:elasticloadbalancing:..." \
  --tags "Key=Environment,Value=Production,PropagateAtLaunch=true"
```

### Bước 3: Xem Trạng Thái ASG

```bash
# Xem thông tin tổng quan
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names "web-servers-asg" \
  --query 'AutoScalingGroups[0].{Min:MinSize,Desired:DesiredCapacity,Max:MaxSize,Instances:Instances[*].HealthStatus}'

# Xem các hoạt động scaling gần đây
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name "web-servers-asg" \
  --max-items 10
```

### Thay Đổi Desired Capacity Thủ Công

```bash
# Scale up thủ công
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name "web-servers-asg" \
  --desired-capacity 8 \
  --honor-cooldown

# Scale in thủ công  
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name "web-servers-asg" \
  --desired-capacity 2
```

---

## ✅ Best Practices

### Capacity Planning — Lập Kế Hoạch Năng Lực

```
Min Capacity:
  → Đặt đủ để xử lý baseline traffic (lưu lượng nền)
  → Phải đảm bảo High Availability (ít nhất 2 AZ × ít nhất 1 instance/AZ)
  → Ví dụ: min=2 cho 2 AZ

Max Capacity:
  → Đặt đủ lớn để xử lý peak traffic (lưu lượng đỉnh)
  → Không quá lớn để tránh chi phí runaway (không kiểm soát được)
  → Nên test với load testing trước khi đặt max

Desired Capacity:
  → Đặt dựa trên current expected load (tải dự kiến hiện tại)
  → Scaling policies sẽ tự động điều chỉnh
```

### Health Check Configuration

```
Health Check Type: ELB (luôn dùng ELB cho production)
Grace Period: 
  = Boot time + App startup time + Buffer
  Ví dụ: 30s (boot) + 60s (app start) + 30s (buffer) = 120s
  → Đặt 180-300s để an toàn
```

### Tag Management — Quản Lý Thẻ

```bash
# Tags quan trọng cho tracking và billing
Tags:
  - Key: Environment,   Value: Production,  PropagateAtLaunch: true
  - Key: Application,   Value: WebServer,   PropagateAtLaunch: true
  - Key: Team,          Value: Platform,    PropagateAtLaunch: true
  - Key: CostCenter,    Value: Engineering, PropagateAtLaunch: true
```

**`PropagateAtLaunch: true`**: Tag tự động được áp dụng cho tất cả instance mới launch bởi ASG.

### Multi-AZ Best Practice

```
Luôn chỉ định ít nhất 2 subnet (ở 2 AZ khác nhau):
  subnet-111aaa (us-east-1a)
  subnet-222bbb (us-east-1b)
  subnet-333ccc (us-east-1c)  # Tốt nhất là 3 AZ

Khi AZ bị outage:
  → ASG tự động launch instance ở các AZ còn lại
  → Sau khi AZ phục hồi → ASG rebalance lại
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Instance trong ASG bị terminate đột ngột (do Spot reclaim hoặc lỗi) — điều gì xảy ra?**

> ASG phát hiện số lượng InService instance giảm xuống dưới Desired Capacity. ASG ngay lập tức launch instance mới (theo Launch Template) ở AZ phù hợp để đưa số lượng về lại Desired. Nếu instance được gắn với ALB, instance mới phải pass health check trước khi nhận traffic. Toàn bộ quá trình thường mất 2-5 phút.

**Q: Tại sao không nên đặt Max Capacity quá nhỏ?**

> Nếu Max Capacity quá nhỏ, trong giờ peak traffic ASG không thể tạo thêm instance dù có scaling policy. Kết quả: instance hiện tại bị overload → tăng latency → timeout → user experience kém hoặc outage. Max nên được đặt dựa trên load test với thêm 20-30% buffer.

**Q: Health Check Grace Period quá ngắn gây ra vấn đề gì?**

> Nếu Grace Period quá ngắn, ASG bắt đầu kiểm tra health trước khi ứng dụng kịp khởi động. Instance bị đánh dấu unhealthy, bị terminate. ASG launch instance mới → cũng bị terminate → vòng lặp vô hạn. Kết quả: không có instance nào tồn tại đủ lâu để phục vụ traffic (thrashing loop — vòng lặp dao động).

**Q: Cooldown Period (Thời Gian Hạ Nhiệt) và Grace Period khác nhau như thế nào?**

> - **Grace Period**: Áp dụng cho instance vừa launch — thời gian ASG chờ trước khi kiểm tra health của instance mới đó.
> - **Cooldown Period**: Áp dụng cho ASG sau một scaling activity — thời gian ASG chờ trước khi thực hiện scaling activity tiếp theo. Mục đích: tránh scale liên tục trong khi metric chưa ổn định sau lần scale trước.

---

## 🔗 Điều Hướng

| Trước                       | Tiếp Theo                                         |
| --------------------------- | ------------------------------------------------- |
| [← README](README.md)       | [Launch Templates →](2-launch-templates.md)       |

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
