# 📈 Scaling Policies — Chính Sách Co Giãn

> **Scaling Policy** — Chính Sách Co Giãn — là quy tắc định nghĩa **khi nào** và **bao nhiêu** instance được thêm vào hoặc xóa khỏi Auto Scaling Group. Chọn đúng loại scaling policy là yếu tố quan trọng để vừa đảm bảo performance vừa tối ưu chi phí.

---

## 📚 Mục Lục

1. [Tổng Quan Các Loại Policy](#tổng-quan-các-loại-policy)
2. [Target Tracking Scaling](#target-tracking-scaling)
3. [Step Scaling](#step-scaling)
4. [Simple Scaling](#simple-scaling)
5. [Scheduled Scaling — Co Giãn Theo Lịch](#scheduled-scaling)
6. [Predictive Scaling — Co Giãn Dự Đoán](#predictive-scaling)
7. [Cooldown Period — Thời Gian Hạ Nhiệt](#cooldown-period)
8. [Metrics Thường Dùng](#metrics-thường-dùng)
9. [Kết Hợp Nhiều Policies](#kết-hợp-nhiều-policies)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🗺️ Tổng Quan Các Loại Policy

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Scaling Policy Types                              │
├──────────────────────┬───────────────────────────────────────────────┤
│ Target Tracking      │ "Giữ CPU ở mức 70%"                          │
│ (Theo Dõi Mục Tiêu) │ → AWS tự tính toán số instance cần           │
│                      │ → Đơn giản nhất, khuyến nghị mặc định        │
├──────────────────────┼───────────────────────────────────────────────┤
│ Step Scaling         │ "CPU 70-80%: thêm 2; CPU 80-90%: thêm 4"    │
│ (Theo Bậc Thang)    │ → Bạn định nghĩa từng bước                   │
│                      │ → Linh hoạt hơn cho workload không tuyến tính │
├──────────────────────┼───────────────────────────────────────────────┤
│ Simple Scaling       │ "CPU > 70%: thêm 2 instance"                 │
│ (Đơn Giản)          │ → Chờ cooldown giữa các lần scale            │
│                      │ → Legacy, ít dùng (dùng Step hoặc Target)    │
├──────────────────────┼───────────────────────────────────────────────┤
│ Scheduled Scaling    │ "Mỗi thứ 2 lúc 8h: set desired=10"         │
│ (Theo Lịch)         │ → Biết trước thời điểm traffic tăng           │
│                      │ → Pre-scaling trước event                    │
├──────────────────────┼───────────────────────────────────────────────┤
│ Predictive Scaling   │ ML dự báo traffic dựa trên lịch sử           │
│ (Dự Đoán)           │ → Tự động pre-scale trước 15 phút            │
│                      │ → Cần ít nhất 24h dữ liệu lịch sử           │
└──────────────────────┴───────────────────────────────────────────────┘
```

---

## 🎯 Target Tracking Scaling — Co Giãn Theo Mục Tiêu

### Cách Hoạt Động

Target Tracking giống như **cruise control** (kiểm soát hành trình) của xe hơi: bạn đặt mục tiêu (ví dụ: CPU = 70%), AWS tự động tính và điều chỉnh số instance để duy trì mục tiêu đó.

```
Mục tiêu: CPU = 70%

Trạng thái hiện tại:
  4 instances, CPU trung bình = 85%
  → CPU > 70% → Cần scale out

AWS tính toán:
  Instances cần = ceil(4 × 85/70) = ceil(4.86) = 5 instances
  → Launch thêm 1 instance

Sau khi launch:
  5 instances, CPU trung bình = 68%
  → CPU ≈ 70% → Ổn định ✅
```

### Các Predefined Metrics — Metrics Có Sẵn

| Metric                                          | Mô Tả                                            | Dùng Khi                           |
| ------------------------------------------------ | ------------------------------------------------ | ---------------------------------- |
| `ASGAverageCPUUtilization`                      | CPU trung bình của tất cả instances trong ASG    | Phổ biến nhất                      |
| `ASGAverageNetworkIn`                           | Bytes nhận vào trung bình mỗi instance          | Network-bound workload             |
| `ASGAverageNetworkOut`                          | Bytes gửi ra trung bình mỗi instance            | Network-bound workload             |
| `ALBRequestCountPerTarget`                      | Số request ALB mỗi instance (từ Target Group)   | Web servers với load balancer      |

### Tạo Target Tracking Policy

```bash
# Scale khi CPU trung bình > 70%
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name "web-servers-asg" \
  --policy-name "cpu-target-tracking" \
  --policy-type "TargetTrackingScaling" \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 70.0,
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60,
    "DisableScaleIn": false
  }'
```

### Dùng Custom Metric — Metric Tùy Chỉnh

```bash
# Scale dựa trên số messages trong SQS queue
# (Số message / số instances = messages per instance cần xử lý)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name "worker-asg" \
  --policy-name "sqs-target-tracking" \
  --policy-type "TargetTrackingScaling" \
  --target-tracking-configuration '{
    "CustomizedMetricSpecification": {
      "MetricName": "ApproximateNumberOfMessagesVisible",
      "Namespace": "AWS/SQS",
      "Dimensions": [
        {"Name": "QueueName", "Value": "my-processing-queue"}
      ],
      "Statistic": "Average"
    },
    "TargetValue": 100.0
  }'
```

### DisableScaleIn — Vô Hiệu Hóa Scale In

```bash
# Chỉ scale out, không tự động scale in
# Hữu ích khi muốn giữ nguyên capacity sau spike
"DisableScaleIn": true
```

Dùng khi: workload có spike ngắn nhưng terminate instance tốn kém (boot time dài, data cần drain).

---

## 📊 Step Scaling — Co Giãn Theo Bậc Thang

### Khi Nào Dùng Step Scaling?

- Traffic tăng không tuyến tính (linear) — cần thêm nhiều instance hơn khi tải rất cao
- Muốn kiểm soát chính xác số instance thêm ở mỗi mức tải
- Workload có khả năng spike đột ngột cần phản ứng nhanh (không chờ cooldown)

### Cách Hoạt Động

```
CloudWatch Alarm trigger → Step Scaling kiểm tra metric hiện tại → Áp dụng bước phù hợp

CPU Utilization Steps (Các Bước Theo CPU):
  ┌─────────────────────────────────────────────────────────┐
  │ [0%, 60%)    → Không làm gì                            │
  │ [60%, 70%)   → Thêm 1 instance (scale out nhẹ)         │
  │ [70%, 85%)   → Thêm 3 instances (scale out vừa)        │
  │ [85%, 100%]  → Thêm 5 instances (scale out mạnh)       │
  └─────────────────────────────────────────────────────────┘

Scale In Steps (Thu Hẹp):
  ┌─────────────────────────────────────────────────────────┐
  │ [40%, 60%)   → Không làm gì                            │
  │ [20%, 40%)   → Xóa 1 instance                          │
  │ [0%, 20%)    → Xóa 3 instances                         │
  └─────────────────────────────────────────────────────────┘
```

### Tạo CloudWatch Alarm + Step Scaling Policy

```bash
# Bước 1: Tạo Scale Out Policy
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name "web-servers-asg" \
  --policy-name "step-scale-out" \
  --policy-type "StepScaling" \
  --adjustment-type "ChangeInCapacity" \
  --step-adjustments '[
    {
      "MetricIntervalLowerBound": 0,
      "MetricIntervalUpperBound": 10,
      "ScalingAdjustment": 1
    },
    {
      "MetricIntervalLowerBound": 10,
      "MetricIntervalUpperBound": 25,
      "ScalingAdjustment": 3
    },
    {
      "MetricIntervalLowerBound": 25,
      "ScalingAdjustment": 5
    }
  ]' \
  --estimated-instance-warmup 300

# Bước 2: Tạo CloudWatch Alarm để trigger policy
POLICY_ARN=$(aws autoscaling describe-policies \
  --auto-scaling-group-name "web-servers-asg" \
  --policy-names "step-scale-out" \
  --query 'ScalingPolicies[0].PolicyARN' \
  --output text)

aws cloudwatch put-metric-alarm \
  --alarm-name "high-cpu-alarm" \
  --metric-name "CPUUtilization" \
  --namespace "AWS/EC2" \
  --dimensions "Name=AutoScalingGroupName,Value=web-servers-asg" \
  --statistic "Average" \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 60 \
  --comparison-operator "GreaterThanOrEqualToThreshold" \
  --alarm-actions "$POLICY_ARN"
```

### Adjustment Types — Loại Điều Chỉnh

| Adjustment Type           | Ý Nghĩa                                   | Ví Dụ                                       |
| ------------------------- | ----------------------------------------- | ------------------------------------------- |
| `ChangeInCapacity`        | Thêm/bớt số lượng tuyệt đối             | `+3` → thêm 3 instances                    |
| `ExactCapacity`           | Đặt Desired Capacity về giá trị cụ thể   | `8` → set desired = 8                       |
| `PercentChangeInCapacity` | Thêm/bớt theo phần trăm hiện tại        | `+50%` → thêm 50% số instance hiện có      |

---

## ⚡ Simple Scaling — Co Giãn Đơn Giản (Legacy)

Simple Scaling là loại cũ nhất — chỉ thực hiện **một hành động duy nhất** khi alarm trigger, sau đó **chờ cooldown** trước khi có thể scale tiếp.

```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name "web-servers-asg" \
  --policy-name "simple-scale-out" \
  --policy-type "SimpleScaling" \
  --adjustment-type "ChangeInCapacity" \
  --scaling-adjustment 2 \
  --cooldown 300
```

**Nhược điểm của Simple Scaling:**
- Chờ cooldown (300s) dù metric vẫn đang tăng nhanh
- Không thể scale nhanh hơn 1 bước mỗi 5 phút
- **Step Scaling và Target Tracking không có giới hạn này** → luôn ưu tiên dùng Step hoặc Target Tracking

---

## 🗓️ Scheduled Scaling — Co Giãn Theo Lịch

### Khi Nào Dùng Scheduled Scaling?

Khi bạn **biết trước** traffic pattern:
- E-commerce: Thứ 6, thứ 7 traffic tăng gấp 3
- B2B SaaS: Giờ làm việc (8h-18h) cần nhiều instance hơn
- Batch jobs: Tăng capacity trước khi chạy batch lúc 2h sáng
- Marketing events: Black Friday, launch sản phẩm

### Tạo Scheduled Actions

```bash
# Scale up mỗi sáng thứ 2-6 lúc 7h30 UTC (trước giờ làm)
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name "web-servers-asg" \
  --scheduled-action-name "weekday-scale-up" \
  --recurrence "30 7 * * 1-5" \
  --min-size 5 \
  --max-size 20 \
  --desired-capacity 10

# Scale down mỗi tối thứ 2-6 lúc 19h UTC
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name "web-servers-asg" \
  --scheduled-action-name "weekday-scale-down" \
  --recurrence "0 19 * * 1-5" \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 3

# Scale up một lần cho sự kiện cụ thể (không recurrence)
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name "web-servers-asg" \
  --scheduled-action-name "black-friday-2024" \
  --start-time "2024-11-29T00:00:00Z" \
  --end-time "2024-11-30T00:00:00Z" \
  --min-size 20 \
  --max-size 100 \
  --desired-capacity 50
```

### Cú Pháp Cron — Biểu Thức Thời Gian

```
Cron format: "phút giờ ngày tháng ngày_tuần"

"0 8 * * 1-5"    = 8:00 sáng thứ 2-6 (UTC)
"30 7 * * 1-5"   = 7:30 sáng thứ 2-6 (UTC)
"0 0 * * 0"      = 0:00 sáng chủ nhật (UTC)
"0 */6 * * *"    = Mỗi 6 giờ một lần
"0 18 * * 5"     = 18:00 thứ 6 (UTC)

Lưu ý: AWS Scheduled Scaling dùng UTC. Nếu team ở GMT+7 (Việt Nam):
  7:30 UTC → 14:30 Việt Nam
  → Thêm 7 giờ vào giờ mong muốn khi tính UTC
```

---

## 🤖 Predictive Scaling — Co Giãn Dự Đoán

### Tại Sao Cần Predictive Scaling?

```
Vấn đề với Reactive Scaling (Scale phản ứng):
  Traffic đến → CPU tăng → Alarm trigger → Launch instance → Chờ boot (2-5 phút)
  → Trong 2-5 phút này: instance cũ bị overload → latency tăng → user experience kém

Predictive Scaling giải quyết:
  ML phân tích lịch sử → Dự báo traffic sẽ tăng lúc 9h → Pre-scale lúc 8:45
  → Khi traffic thực sự đến 9h: instances đã sẵn sàng ✅
```

### Cách Hoạt Động

```
                    ┌─────────────────────────────────────┐
                    │  AWS ML Model (Mô Hình Học Máy)     │
                    │                                     │
                    │  Phân tích:                         │
                    │  - 14 ngày lịch sử traffic          │
                    │  - Weekly patterns (tuần lặp lại)   │
                    │  - Daily patterns (ngày lặp lại)    │
                    └──────────────┬──────────────────────┘
                                   │ Forecast (Dự báo)
                    ┌──────────────▼──────────────────────┐
                    │  Scaling Schedule (Lịch Scale)       │
                    │                                     │
                    │  08:45: Desired = 8                 │
                    │  09:00: Desired = 12                │
                    │  12:00: Desired = 10                │
                    │  18:00: Desired = 5                 │
                    └──────────────┬──────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────┐
                    │  Auto Scaling Group                  │
                    │  Pre-scales 15 phút trước forecast  │
                    └─────────────────────────────────────┘
```

### Tạo Predictive Scaling Policy

```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name "web-servers-asg" \
  --policy-name "predictive-scaling" \
  --policy-type "PredictiveScaling" \
  --predictive-scaling-configuration '{
    "MetricSpecifications": [
      {
        "TargetValue": 70.0,
        "PredefinedMetricPairSpecification": {
          "PredefinedMetricType": "ASGCPUUtilization"
        }
      }
    ],
    "Mode": "ForecastAndScale",
    "SchedulingBufferTime": 300,
    "MaxCapacityBreachBehavior": "IncreaseMaxCapacity",
    "MaxCapacityBuffer": 10
  }'
```

| Tham Số                      | Ý Nghĩa                                                   |
| ----------------------------- | --------------------------------------------------------- |
| `Mode: ForecastOnly`         | Chỉ hiển thị dự báo, không tự động scale (để review)     |
| `Mode: ForecastAndScale`     | Dự báo và tự động scale theo dự báo                      |
| `SchedulingBufferTime: 300`  | Pre-scale trước 5 phút so với dự báo                     |
| `MaxCapacityBreachBehavior`  | Hành động khi dự báo vượt MaxCapacity                    |

---

## ⏰ Cooldown Period — Thời Gian Hạ Nhiệt

### Mục Đích Của Cooldown

```
Không có Cooldown:
  CPU = 80% → Scale out → Launch 2 instances
  CPU vẫn > threshold (instances chưa boot xong) → Scale out → Launch 2 nữa
  → Over-provisioning (cấp phát quá mức cần thiết) ❌

Có Cooldown (300s):
  CPU = 80% → Scale out → Launch 2 instances
  Chờ 300 giây (instances boot và stabilize)
  CPU = 65% (đã ổn định) → Không scale tiếp ✅
```

### Cooldown Settings — Cấu Hình Thời Gian Hạ Nhiệt

```
Default Cooldown (Mặc Định ASG):
  Áp dụng cho mọi scaling activity nếu không có override
  Thường: 300 giây

Scaling Policy Cooldown (Override):
  Scale Out Cooldown: Thường ngắn hơn (60-120s) — muốn phản ứng nhanh
  Scale In Cooldown:  Thường dài hơn (300-600s) — không muốn terminate vội

Instance Warmup (cho Step/Target Tracking):
  Thời gian chờ instance mới "warm up" (ổn định) trước khi tính vào metric
  Tránh: Instance mới boot → CPU tạm thời cao → trigger scale out thêm
```

```bash
# Target Tracking với cooldown riêng
aws autoscaling put-scaling-policy \
  --policy-type "TargetTrackingScaling" \
  --target-tracking-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }'
```

### Instance Warmup vs Cooldown

| Khái Niệm         | Áp Dụng Cho              | Mục Đích                                                  |
| ------------------ | ------------------------ | --------------------------------------------------------- |
| **Cooldown**       | ASG (toàn bộ nhóm)       | Chặn scaling activity tiếp theo của ASG                  |
| **Instance Warmup**| Instance mới được launch | Không tính metric của instance mới vào aggregate metric  |

---

## 📊 Metrics Thường Dùng

### EC2 Metrics (Đo Bởi CloudWatch Tự Động)

| Metric                  | Namespace       | Ý Nghĩa                          | Scale Out Khi           |
| ----------------------- | --------------- | -------------------------------- | ----------------------- |
| `CPUUtilization`        | `AWS/EC2`       | % CPU đang sử dụng               | > 70%                   |
| `NetworkIn`             | `AWS/EC2`       | Bytes nhận vào/giây              | > threshold             |
| `NetworkOut`            | `AWS/EC2`       | Bytes gửi ra/giây                | > threshold             |
| `StatusCheckFailed`     | `AWS/EC2`       | 1 nếu health check fail          | > 0 (thường dùng cho alarm)|

### ALB/Target Group Metrics

| Metric                              | Namespace          | Ý Nghĩa                          |
| ----------------------------------- | ------------------ | -------------------------------- |
| `RequestCountPerTarget`             | `AWS/ApplicationELB`| Số request mỗi target trong TG  |
| `TargetResponseTime`                | `AWS/ApplicationELB`| Latency trung bình (giây)        |
| `HTTPCode_Target_5XX_Count`         | `AWS/ApplicationELB`| Số lỗi 5xx từ targets           |

### SQS Metrics (Cho Worker ASG)

| Metric                                   | Ý Nghĩa                                    | Scale Out Khi   |
| ---------------------------------------- | ------------------------------------------ | --------------- |
| `ApproximateNumberOfMessagesVisible`     | Số messages đang chờ trong queue           | > target/worker |
| `ApproximateAgeOfOldestMessage`          | Tuổi message cũ nhất (giây)               | > SLA timeout   |

### Custom Application Metrics

```python
# Gửi custom metric từ ứng dụng (Python boto3)
import boto3

cloudwatch = boto3.client('cloudwatch', region_name='us-east-1')

# Ví dụ: số active sessions (phiên hoạt động) của ứng dụng
cloudwatch.put_metric_data(
    Namespace='MyApp/WebServer',
    MetricData=[
        {
            'MetricName': 'ActiveSessions',
            'Value': 1250,
            'Unit': 'Count',
            'Dimensions': [
                {'Name': 'AutoScalingGroupName', 'Value': 'web-servers-asg'}
            ]
        }
    ]
)
```

---

## 🔀 Kết Hợp Nhiều Policies

### Pattern Phổ Biến: Target Tracking + Scheduled

```
Scenario: E-commerce platform

Scheduled Scaling (biết trước):
  Thứ 6 lúc 20h: set min=10, desired=15 (trước weekend sale)
  Thứ 2 lúc 8h:  set min=3,  desired=5  (back to normal)

Target Tracking (phản ứng với thực tế):
  CPU target = 65% → tự động thêm/bớt instance trong khoảng [min, max]

Kết quả:
  → Scheduled đảm bảo capacity sàn cho weekend
  → Target Tracking tự điều chỉnh trong ngưỡng đã đặt
  → Best of both worlds ✅
```

### Pattern Cho Unpredictable Spikes — Spike Đột Ngột

```
Layer 1: Predictive Scaling
  → Xử lý traffic pattern có tính chu kỳ (daily/weekly)

Layer 2: Target Tracking
  → Xử lý biến động trong chu kỳ

Layer 3: CloudWatch Alarm + Step Scaling
  → Phản ứng nhanh với spike đột ngột (< 60 giây)
  → Alarm dùng 1-minute interval thay vì 5-minute
```

### Ưu Tiên Khi Nhiều Policies Cùng Active

```
Quy tắc: AWS luôn ưu tiên hành động bảo đảm availability:
  → Scale Out: Policy nào scale out NHIỀU NHẤT sẽ được dùng
  → Scale In:  Policy nào scale in ÍT NHẤT sẽ được dùng

Ví dụ:
  Policy A: "Thêm 2 instances"
  Policy B: "Thêm 5 instances"
  → ASG sẽ thêm 5 instances (chọn max để đảm bảo capacity)

  Policy C: "Xóa 3 instances"
  Policy D: "Xóa 1 instance"
  → ASG sẽ xóa 1 instance (chọn min để đảm bảo availability)
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Target Tracking Policy hoạt động như thế nào? Khi nào chọn nó thay vì Step Scaling?**

> Target Tracking sử dụng thuật toán PID-controller-like (bộ điều khiển tỉ lệ) để tính số instance cần để đưa metric về mức target. AWS tự động tạo CloudWatch Alarms và quản lý scaling ở cả hai chiều (scale out và scale in). Chọn Target Tracking khi: muốn cấu hình đơn giản, workload tăng giảm tuyến tính với metric. Chọn Step Scaling khi: cần phản ứng khác nhau ở các mức tải khác nhau, hoặc metric không tuyến tính với capacity cần thiết.

**Q: Tại sao Scale In Cooldown thường dài hơn Scale Out Cooldown?**

> Scale Out: Muốn phản ứng nhanh khi tải tăng để tránh user impact → cooldown ngắn (60s). Scale In: Nếu terminate instance quá nhanh khi tải vừa giảm, có thể tải tăng lại ngay sau đó (flash traffic) → cần terminate lại → tốn kém và gây instability. Scale In cooldown dài hơn (300-600s) đảm bảo metric thực sự đã ổn định ở mức thấp trước khi terminate instance. Asymmetric cooldown (không đối xứng) giúp cân bằng giữa chi phí và stability.

**Q: Predictive Scaling có thể thay thế hoàn toàn Target Tracking không?**

> Không. Predictive Scaling chỉ xử lý được traffic patterns có tính **chu kỳ** (ngày/tuần). Nó không xử lý được: spike đột ngột từ viral content, DDoS, hoặc unexpected events. Nên kết hợp cả hai: Predictive Scaling lo phần có thể dự báo, Target Tracking lo phần không dự báo được. Ngoài ra, Predictive Scaling cần ít nhất 24 giờ lịch sử data để dự báo chính xác.

**Q: Khi cả Scheduled Action và Target Tracking cùng active, điều gì xảy ra?**

> Scheduled Action thay đổi `MinSize`, `MaxSize`, hoặc `DesiredCapacity`. Target Tracking sau đó hoạt động trong phạm vi `[min, max]` mới. Ví dụ: Scheduled Action set min=10 vào tối thứ 6 → Target Tracking sẽ không scale in dưới 10 instance dù CPU thấp. Target Tracking vẫn tự do scale out lên đến max nếu cần. Hai cơ chế bổ sung cho nhau, không xung đột.

---

## 🔗 Điều Hướng

| Trước                                           | Tiếp Theo                                              |
| ----------------------------------------------- | ------------------------------------------------------ |
| [← Launch Templates](2-launch-templates.md)     | [Spot & Mixed Instances →](4-spot-mixed-instances.md)  |

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
