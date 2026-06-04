# ⚖️ Sự Cố Auto Scaling — Chẩn Đoán & Khắc Phục

> Hướng dẫn xử lý toàn diện các sự cố phổ biến trong Auto Scaling Groups (ASG — Nhóm Tự Động Co Giãn): launch failures, scaling không hoạt động đúng, cooldown quá dài, và instance bị terminate nhầm.

## 📚 Mục Lục

1. [Launch Failures — Không Thể Khởi Chạy Instance](#1-launch-failures)
2. [Scaling Không Xảy Ra Đúng Lúc](#2-scaling-không-xảy-ra-đúng-lúc)
3. [Scaling Imbalance — Mất Cân Bằng AZ](#3-scaling-imbalance)
4. [Instance Bị Terminate Nhầm](#4-instance-bị-terminate-nhầm)
5. [Cooldown Period Gây Vấn Đề](#5-cooldown-period-gây-vấn-đề)
6. [Capacity Không Đạt Desired](#6-capacity-không-đạt-desired)

---

## 1. Launch Failures

### Xem Activity History Để Tìm Lỗi

```bash
# Xem 10 activity gần nhất của ASG
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name my-asg \
  --max-items 10 \
  --query 'Activities[*].[StatusCode,StatusMessage,Description,StartTime]' \
  --output table

# Lọc chỉ những activity thất bại
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name my-asg \
  --query 'Activities[?StatusCode==`Failed`].[StatusMessage,StartTime]'
```

### Nguyên Nhân Launch Failure Phổ Biến

#### A. Insufficient Instance Capacity (Không Đủ Tài Nguyên EC2)

```
Thông báo lỗi: "We currently do not have sufficient capacity in the
Availability Zone you requested."

Giải pháp:
  1. Mở rộng sang nhiều AZ hơn trong ASG
  2. Thêm nhiều instance types vào Launch Template (Mixed Instance Policy)
  3. Dùng Spot Instances để tăng khả năng được cấp phát
  4. Cân nhắc EC2 Capacity Reservations (Đặt Trước Năng Lực EC2)
```

**Cấu Hình Mixed Instance Policy** (Chính Sách Instance Hỗn Hợp):

```json
{
  "MixedInstancesPolicy": {
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 2,
      "OnDemandPercentageAboveBaseCapacity": 30,
      "SpotAllocationStrategy": "capacity-optimized"
    },
    "LaunchTemplate": {
      "LaunchTemplateSpecification": {
        "LaunchTemplateId": "lt-xxxxxxxx",
        "Version": "$Latest"
      },
      "Overrides": [
        {"InstanceType": "m5.large"},
        {"InstanceType": "m5a.large"},
        {"InstanceType": "m4.large"},
        {"InstanceType": "m5d.large"}
      ]
    }
  }
}
```

#### B. VPC Subnet Đầy IP

```
Triệu chứng: "There are not enough free addresses in the subnet to
satisfy your minimum request of 1 private IP address."

Xử lý:
  1. Kiểm tra số IP còn trống trong subnet
     aws ec2 describe-subnets --subnet-ids subnet-xxxxxxxx \
       --query 'Subnets[].AvailableIpAddressCount'

  2. Nếu subnet /24 (254 IPs) đầy → Tạo subnet mới, thêm vào ASG
  3. Xem xét dùng CIDR lớn hơn ngay từ đầu (/23 = 510 IPs)
```

#### C. IAM Permissions Thiếu

```
Lỗi: "User: arn:aws:sts::...AutoScaling... is not authorized to perform:
ec2:RunInstances"

Auto Scaling Service Role cần có:
  - ec2:RunInstances
  - ec2:DescribeInstances
  - iam:PassRole (để pass IAM Instance Profile cho EC2)
  - elasticloadbalancing:RegisterTargets (nếu dùng ELB)
```

#### D. Launch Template / AMI Không Hợp Lệ

```bash
# Kiểm tra AMI còn tồn tại và available trong region hiện tại
aws ec2 describe-images --image-ids ami-xxxxxxxx \
  --query 'Images[].State'
# Kết quả "available" = tốt, trống hoặc không có = AMI đã bị xóa

# Kiểm tra Launch Template version
aws ec2 describe-launch-template-versions \
  --launch-template-id lt-xxxxxxxx \
  --query 'LaunchTemplateVersions[*].[VersionNumber,DefaultVersion,LaunchTemplateData.ImageId]'
```

---

## 2. Scaling Không Xảy Ra Đúng Lúc

### Scale Out Không Xảy Ra Khi CPU Cao

```
Triệu chứng: CPU đang 90% nhưng ASG không thêm instance

Checklist kiểm tra:
  □ ASG đã đạt MaxSize chưa?
  □ Scaling Policy có đúng không? (Target Tracking vs Step Policy)
  □ CloudWatch Alarm có ở trạng thái ALARM không?
  □ Cooldown period còn đang chạy không?
  □ Có Scaling Activity nào đang Pending không?
```

```bash
# Kiểm tra trạng thái ASG
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names my-asg \
  --query 'AutoScalingGroups[*].[MinSize,MaxSize,DesiredCapacity,Instances[].LifecycleState]'

# Kiểm tra scaling policies được gắn với ASG
aws autoscaling describe-policies \
  --auto-scaling-group-name my-asg

# Kiểm tra CloudWatch Alarm liên kết với policy
aws cloudwatch describe-alarms \
  --alarm-names my-cpu-alarm \
  --query 'MetricAlarms[*].[AlarmName,StateValue,MetricName,Threshold]'
```

### Scale In Không Xảy Ra Khi CPU Thấp

```
Triệu chứng: Tải giảm xuống 5% nhưng ASG vẫn duy trì instance cũ

Nguyên nhân:
  1. Scale-in protection — Instance được bảo vệ khỏi terminate
  2. MinSize setting giữ minimum instances
  3. Target Tracking policy có scale-in delay mặc định 15 phút
  4. Instance đang trong Lifecycle Hook → Pending:Wait state
```

```bash
# Kiểm tra instance có bật scale-in protection không
aws autoscaling describe-auto-scaling-instances \
  --query 'AutoScalingInstances[?ProtectedFromScaleIn==`true`]'

# Kiểm tra lifecycle hooks đang pending
aws autoscaling describe-lifecycle-hooks \
  --auto-scaling-group-name my-asg

# Xem instance đang ở lifecycle state nào
aws autoscaling describe-auto-scaling-instances \
  --query 'AutoScalingInstances[*].[InstanceId,LifecycleState,AutoScalingGroupName]'
```

---

## 3. Scaling Imbalance — Mất Cân Bằng AZ

### Triệu Chứng

```
Mong muốn: Mỗi AZ có 3 instances
Thực tế:   AZ-a: 5 instances, AZ-b: 1 instance, AZ-c: 0 instances
```

### Nguyên Nhân Mất Cân Bằng

```
1. Một AZ bị hết capacity (Spot hoặc On-Demand) → ASG dồn sang AZ khác
2. Subnet của một AZ bị xóa khỏi cấu hình ASG
3. Rebalancing bị tắt hoặc không được trigger
4. Instance failures ở một AZ không được thay thế kịp thời
```

### Kích Hoạt Rebalancing Thủ Công

```bash
# ASG tự động rebalance khi nhận ra imbalance
# Để force rebalance ngay lập tức:

# Cách 1: Tăng MaxSize tạm thời (ASG sẽ launch instances ở AZ thiếu)
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --max-size 15  # Tăng tạm thời

# Cách 2: Terminate một instance ở AZ dư thừa (ASG sẽ rebalance)
aws autoscaling terminate-instance-in-auto-scaling-group \
  --instance-id i-xxxxxxxx \
  --should-decrement-desired-capacity false  # Giữ nguyên desired capacity
```

### Cấu Hình ASG Để Cân Bằng Tốt Hơn

```bash
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --availability-zone-rebalancing enabled \  # Tự động rebalance khi mất cân bằng
  --default-instance-warmup 120               # Thời gian warmup trước khi tính metrics
```

---

## 4. Instance Bị Terminate Nhầm

### Tìm Hiểu Tại Sao Instance Bị Terminate

```bash
# Xem lịch sử terminate
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name my-asg \
  --query 'Activities[?Description!=null].[StatusCode,Description,StartTime]' \
  --output table

# Ví dụ description:
# "Terminating EC2 instance: i-xxx" (scale in bình thường)
# "Terminating EC2 instance due to failed ELB health check" (health check fail)
# "Replacing unhealthy instance: i-xxx" (EC2 status check fail)
```

### Health Check False Positives (Phát Hiện Lỗi Sai)

```
Tình huống: Instance khỏe mạnh nhưng bị đánh dấu unhealthy và terminate

Nguyên nhân phổ biến:
  1. Health check path trả về 500 khi ứng dụng đang khởi động (startup period)
  2. Health check timeout quá ngắn — response chưa kịp trả về
  3. ELB health check và EC2 health check có kết quả khác nhau
  4. Application trả về status code không nằm trong healthy range
```

```bash
# Xem cấu hình health check của ASG
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names my-asg \
  --query 'AutoScalingGroups[*].[HealthCheckType,HealthCheckGracePeriod]'

# Tăng Health Check Grace Period để instance có thời gian khởi động
# (Thời Gian Ân Hạn Kiểm Tra Sức Khỏe)
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --health-check-grace-period 300  # 5 phút để ứng dụng khởi động xong

# Tạm thời đặt instance ở trạng thái Standby để tránh bị terminate khi debug
aws autoscaling enter-standby \
  --instance-ids i-xxxxxxxx \
  --auto-scaling-group-name my-asg \
  --should-decrement-desired-capacity false
```

### Termination Policy — Chính Sách Terminate

```
Khi ASG cần scale in, thứ tự ưu tiên terminate instance:

Default Termination Policy:
  1. AZ nào có nhiều instance nhất → xử lý AZ đó trước (rebalance)
  2. Trong AZ đó, instance nào dùng Launch Configuration cũ nhất
  3. Nếu tất cả dùng Launch Template mới nhất → instance nào gần billing hour nhất

Các policy khác:
  - OldestInstance → Terminate instance cũ nhất trước
  - NewestInstance → Terminate instance mới nhất trước (ít dùng)
  - ClosestToNextInstanceHour → Tiết kiệm chi phí nhất
  - Default → Theo thứ tự mô tả ở trên
```

---

## 5. Cooldown Period Gây Vấn Đề

### Cooldown Quá Dài — Không Scale Kịp

```
Vấn đề: Traffic spike (Đợt Tăng Đột Biến) trong 2 phút nhưng cooldown 5 phút
        → ASG không scale out kịp thời

Giải pháp: Điều chỉnh cooldown phù hợp với workload

Loại Cooldown:
  - Default Cooldown (Mặc Định): Áp dụng cho Simple/Step Scaling — thường 300s
  - Scaling-specific Cooldown: Có thể override per-policy
  - Target Tracking: Không dùng cooldown — tự quản lý
  - Instance Warmup Period: Thời gian chờ instance "sẵn sàng" trước khi tính metric
```

```bash
# Cập nhật default cooldown của ASG
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --default-cooldown 120  # Giảm xuống 2 phút

# Cập nhật cooldown cho specific scaling policy
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-asg \
  --policy-name scale-out-policy \
  --scaling-adjustment 2 \
  --adjustment-type ChangeInCapacity \
  --cooldown 60  # 1 phút cooldown cho scale out nhanh hơn

# Default Instance Warmup — ảnh hưởng đến Target Tracking
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --default-instance-warmup 60  # Sau 60s, instance được tính vào metrics
```

### Cooldown Quá Ngắn — Flapping (Co Giãn Liên Tục)

```
Vấn đề: Scale out → scale in → scale out liên tục
        → Tốn chi phí, gây instability

Dấu hiệu trong Activity History:
  10:00 - Launched i-aaa (CPU 80%)
  10:02 - Terminated i-aaa (CPU 40%)
  10:04 - Launched i-bbb (CPU 80%)
  ...

Giải pháp:
  1. Tăng cooldown (300-600 giây cho production)
  2. Tăng InstanceWarmup period
  3. Dùng Target Tracking thay vì Step Scaling (ít flapping hơn)
  4. Xem xét scale-in cooldown khác scale-out cooldown:
     Scale out: cooldown = 60s (cần nhanh)
     Scale in: cooldown = 300s (cần thận trọng)
```

---

## 6. Capacity Không Đạt Desired

### Desired Capacity vs Actual Running Instances

```bash
# Kiểm tra trạng thái chi tiết
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names my-asg \
  --query 'AutoScalingGroups[*].{
    Desired:DesiredCapacity,
    Min:MinSize,
    Max:MaxSize,
    RunningCount:length(Instances[?LifecycleState==`InService`]),
    PendingCount:length(Instances[?LifecycleState==`Pending`]),
    TerminatingCount:length(Instances[?LifecycleState==`Terminating`])
  }'
```

### Tại Sao Không Đạt Desired Capacity?

```
1. Launch Failures liên tục (xem phần 1)
   → Kiểm tra Activity History cho failed launches

2. Account-level EC2 limits (Giới Hạn EC2 Cấp Tài Khoản)
   → aws service-quotas get-service-quota \
       --service-code ec2 \
       --quota-code L-1216C47A  # Running On-Demand Standard Instances

3. Instance bị terminate ngay sau khi launch (bootstrap lỗi)
   → Health check grace period quá ngắn
   → User Data script crash

4. Spot Interruptions liên tục (không đủ capacity cho Spot)
   → Dùng diversified instance types và AZs
   → Kết hợp On-Demand base capacity
```

### Force Update Desired Capacity Thủ Công

```bash
# Tăng desired capacity thủ công (vượt qua scaling policies tạm thời)
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name my-asg \
  --desired-capacity 5 \
  --honor-cooldown false  # Bỏ qua cooldown để áp dụng ngay

# Thay thế tất cả instances trong ASG (rolling replacement)
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name my-asg \
  --preferences '{
    "MinHealthyPercentage": 80,
    "InstanceWarmup": 120,
    "CheckpointDelay": 3600
  }'
```

---

## 🔁 Tổng Kết — Quy Trình Chẩn Đoán Auto Scaling

```
ASG có vấn đề?
│
├─ KHÔNG LAUNCH ĐƯỢC?
│   └─ aws autoscaling describe-scaling-activities
│       ├─ InsufficientInstanceCapacity → Thêm AZ, thêm instance types
│       ├─ Subnet đầy IP → Mở rộng hoặc tạo subnet mới
│       ├─ AMI không tìm thấy → Update Launch Template
│       └─ IAM permission denied → Kiểm tra Service Role
│
├─ KHÔNG SCALE OUT?
│   ├─ Đã đạt MaxSize → Tăng MaxSize nếu cần
│   ├─ Cooldown đang chạy → Chờ hoặc giảm cooldown
│   └─ CloudWatch Alarm chưa ALARM → Kiểm tra metric & threshold
│
├─ INSTANCE BỊ TERMINATE NHẦM?
│   ├─ Health check grace period ngắn → Tăng lên
│   ├─ Health check path trả 500 → Sửa application
│   └─ Bật scale-in protection cho critical instances
│
└─ MẤT CÂN BẰNG AZ?
    └─ Bật availability-zone-rebalancing
    └─ Kiểm tra subnet của từng AZ trong ASG config
```

---

## 📖 Tham Khảo Thêm

- [1-ec2-issues.md](1-ec2-issues.md) — Sự cố EC2 instance trong ASG
- [../02-auto-scaling/3-scaling-policies.md](../02-auto-scaling/3-scaling-policies.md) — Scaling policies chi tiết
- [../02-auto-scaling/5-lifecycle-hooks.md](../02-auto-scaling/5-lifecycle-hooks.md) — Lifecycle hooks & warm pools

---

**Cập Nhật Lần Cuối:** 2026-05-15
