# Deployment Strategies — Chiến Lược Triển Khai Không Gây Downtime

> Deployment Strategy (Chiến Lược Triển Khai) quyết định cách cập nhật ứng dụng sang version mới. Chọn sai strategy có thể gây downtime, ảnh hưởng đến trải nghiệm người dùng, và khó rollback khi có lỗi. Mục tiêu: deploy nhanh, an toàn, dễ rollback.

## 📚 Mục Lục

1. [Tổng Quan Các Chiến Lược](#tổng-quan)
2. [Rolling Update — Cập Nhật Cuộn](#rolling-update)
3. [Blue/Green Deployment — Triển Khai Xanh/Lam](#blue-green)
4. [Canary Release — Phát Hành Canary](#canary-release)
5. [Immutable Deployment — Triển Khai Bất Biến](#immutable)
6. [A/B Testing](#ab-testing)
7. [Triển Khai trên AWS Services](#aws-deployment-tools)
8. [So Sánh & Chọn Strategy](#so-sánh)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🗺️ Tổng Quan Các Chiến Lược {#tổng-quan}

```
Strategy          Risk   Speed  Cost    Rollback     Downtime
─────────────────────────────────────────────────────────────
Rolling Update    Trung  Trung  Thấp    Chậm (vài min)  Không
Blue/Green        Thấp   Nhanh  Cao     Tức thì (giây)  Không
Canary            Rất thấp Chậm Trung   Tức thì         Không
Immutable         Thấp   Trung  Cao     Nhanh           Không
In-place          Cao    Rất nhanh Rất thấp Không có   Có
```

---

## 🔄 Rolling Update — Cập Nhật Cuộn {#rolling-update}

### Cách Hoạt Động

Thay thế dần dần instances cũ bằng instances mới, từng nhóm nhỏ (batch) một.

```
Trạng Thái Ban Đầu: 4 instances [v1] [v1] [v1] [v1]

Batch 1: Replace 2 instances
         [v2] [v2] [v1] [v1]   ← 2 instance v2, 2 v1 đang chạy

Batch 2: Replace 2 instances còn lại
         [v2] [v2] [v2] [v2]   ← Hoàn thành

Trong suốt quá trình: ALB route traffic đến instances healthy
```

### ASG Rolling Update (Instance Refresh)

```bash
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name my-asg \
  --strategy Rolling \
  --preferences '{
    "MinHealthyPercentage": 75,    # Giữ ít nhất 75% instances healthy
    "InstanceWarmup": 300,          # Chờ 300 giây sau khi instance healthy
    "MaxHealthyPercentage": 110,    # Tạm thời tăng 10% capacity
    "CheckpointPercentages": [20, 50, 100],  # Dừng để verify tại mỗi checkpoint
    "CheckpointDelay": 600          # Chờ 10 phút tại mỗi checkpoint
  }'
```

**MinHealthyPercentage ảnh hưởng tốc độ deploy:**
- 100%: Thêm instances mới trước, xóa cũ sau (cần thêm capacity)
- 90%: Replace 10% mỗi lần
- 50%: Replace 50% cùng lúc (nhanh nhưng rủi ro hơn)

### ECS Rolling Update

```json
{
  "deploymentConfiguration": {
    "minimumHealthyPercent": 100,    // Không giảm dưới 100% healthy tasks
    "maximumPercent": 200            // Cho phép tăng gấp đôi tasks tạm thời
  }
}
```

**ECS Rolling Deployment Flow:**

```
Desired: 4 tasks [v1][v1][v1][v1]

Phase 1: Launch 4 new tasks (maximumPercent=200)
         [v1][v1][v1][v1][v2][v2][v2][v2]  → 8 tasks running

Phase 2: Stop old tasks khi new tasks healthy
         [v2][v2][v2][v2]  → Back to 4 tasks

Total downtime: 0
Traffic disruption: 0 (ELB health check guards)
```

### Kubernetes Rolling Update

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Tối đa 1 pod thêm vào (5 pods tổng)
      maxUnavailable: 0    # Không cho phép pod nào down
```

### Nhược Điểm Rolling Update

- **Hai version chạy song song** trong quá trình deploy — cần backward compatible
- **Không thể test version mới** trước khi nhận production traffic
- **Rollback chậm** — phải deploy lại version cũ theo cùng rolling process

---

## 🔵🟢 Blue/Green Deployment — Triển Khai Xanh/Lam {#blue-green}

### Cách Hoạt Động

Duy trì **hai môi trường production** đồng thời. Một môi trường đang chạy (Blue — xanh), một môi trường với version mới (Green — lam). Khi sẵn sàng → chuyển 100% traffic.

```
Trước Deploy:
  Blue (v1) ←── 100% traffic    Green (idle)
  
Sau Deploy v2 lên Green:
  Blue (v1)                     Green (v2) ← smoke test, QA
  
Khi sẵn sàng, chuyển traffic:
  Blue (v1) (giữ lại để rollback)   Green (v2) ←── 100% traffic
  
Rollback nếu cần:
  Blue (v1) ←── 100% traffic    Green (v2) (tắt xuống)
```

### Blue/Green với ALB Target Groups

```bash
# Tạo hai target groups: blue và green
aws elbv2 create-target-group --name my-app-blue --port 8080 ...
aws elbv2 create-target-group --name my-app-green --port 8080 ...

# Hiện tại blue đang nhận traffic
# Forward 100% đến blue
aws elbv2 modify-rule \
  --rule-arn $RULE_ARN \
  --actions '[{
    "Type": "forward",
    "ForwardConfig": {
      "TargetGroups": [
        {"TargetGroupArn": "BLUE_TG_ARN", "Weight": 100},
        {"TargetGroupArn": "GREEN_TG_ARN", "Weight": 0}
      ]
    }
  }]'

# Deploy v2 lên green, test xong
# Chuyển traffic sang green (một lần)
aws elbv2 modify-rule \
  --rule-arn $RULE_ARN \
  --actions '[{
    "Type": "forward",
    "ForwardConfig": {
      "TargetGroups": [
        {"TargetGroupArn": "BLUE_TG_ARN", "Weight": 0},
        {"TargetGroupArn": "GREEN_TG_ARN", "Weight": 100}
      ]
    }
  }]'
```

### Blue/Green với CodeDeploy trên ECS

```yaml
# appspec.yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: "my-app"
          ContainerPort: 8080
        PlatformVersion: "LATEST"

Hooks:
  - BeforeAllowTraffic: "LambdaFuncBeforeAllowTraffic"
  - AfterAllowTraffic: "LambdaFuncAfterAllowTraffic"
```

```bash
# deployment-config với traffic shifting
aws deploy create-deployment-config \
  --deployment-config-name MyBlueGreen \
  --traffic-routing-config '{
    "type": "AllAtOnce"    # Hoặc "TimeBasedLinear", "TimeBasedCanary"
  }'
```

**ECS Blue/Green với CodeDeploy:**
- CodeDeploy tạo task set mới (Green)
- Chờ Green healthy
- Dịch chuyển traffic (AllAtOnce hoặc theo % dần)
- Giữ Blue 1 giờ để rollback (configurable)
- Sau đó terminate Blue

### Ưu/Nhược Điểm Blue/Green

| Ưu Điểm                              | Nhược Điểm                                    |
| ------------------------------------ | --------------------------------------------- |
| Rollback tức thì (chuyển DNS/LB)    | Chi phí gấp đôi trong thời gian deploy        |
| Test version mới trước khi live      | Database migration phức tạp (cần backward compat) |
| Zero downtime tuyệt đối              | Cần đủ capacity cho cả hai môi trường          |
| Môi trường production chính xác      | Warming up instances mất thời gian             |

---

## 🐦 Canary Release — Phát Hành Canary {#canary-release}

### Cách Hoạt Động

Chuyển một **phần nhỏ traffic** sang version mới, theo dõi, rồi dần dần tăng tỷ lệ.

```
Giai Đoạn 1:  v1=97% / v2=3%     ← Chỉ 3% users test v2
Giai Đoạn 2:  v1=90% / v2=10%    ← Tăng nếu không có lỗi
Giai Đoạn 3:  v1=50% / v2=50%    ← Canary đủ rộng để khẳng định
Giai Đoạn 4:  v1=0%  / v2=100%   ← Hoàn thành
```

**Nguồn gốc tên "Canary":** Thợ mỏ ngày xưa mang chim hoàng yến (canary) vào mỏ. Chim nhạy cảm với khí độc hơn người — nếu chim chết, thợ mỏ biết phải sơ tán. Tương tự, canary release "thăm dò" môi trường production với ít users trước.

### Canary với ALB Weighted Target Groups

```bash
# 5% traffic sang v2 canary
aws elbv2 modify-rule \
  --rule-arn $RULE_ARN \
  --actions '[{
    "Type": "forward",
    "ForwardConfig": {
      "TargetGroups": [
        {"TargetGroupArn": "V1_TG_ARN", "Weight": 95},
        {"TargetGroupArn": "V2_TG_ARN", "Weight": 5}
      ],
      "StickinessDuration": 86400
    }
  }]'
```

### Canary với Route 53 Weighted Routing

```bash
# 5% requests đến canary stack
aws route53 change-resource-record-sets \
  --hosted-zone-id ZONE_ID \
  --change-batch '{
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "api.example.com",
          "Type": "A",
          "SetIdentifier": "stable",
          "Weight": 95,
          "AliasTarget": {"DNSName": "stable-alb.elb.amazonaws.com", ...}
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "api.example.com",
          "Type": "A",
          "SetIdentifier": "canary",
          "Weight": 5,
          "AliasTarget": {"DNSName": "canary-alb.elb.amazonaws.com", ...}
        }
      }
    ]
  }'
```

### Automated Canary với CodeDeploy

```bash
# TimeBasedLinear: 10% mỗi 5 phút
aws deploy create-deployment-config \
  --deployment-config-name Canary10Percent5Minutes \
  --traffic-routing-config '{
    "type": "TimeBasedLinear",
    "timeBasedLinear": {
      "linearPercentage": 10,
      "linearInterval": 5
    }
  }'
```

**Timeline:**
```
Phút 0:   10% → v2
Phút 5:   20% → v2
Phút 10:  30% → v2
...
Phút 45:  100% → v2
```

### Metrics để Theo Dõi Canary

```
Theo Dõi Canary (so sánh v1 vs v2):
✅ Error rate (tỷ lệ lỗi) — target: < 0.1%
✅ P95/P99 latency (độ trễ bách phân vị) — target: không tăng > 20%
✅ Success rate (tỷ lệ thành công) — target: > 99.9%
✅ Custom business metrics (đơn hàng, chuyển đổi)

Nếu bất kỳ metric nào vượt ngưỡng → Auto rollback (tự động quay lại)
```

---

## 🧱 Immutable Deployment — Triển Khai Bất Biến {#immutable}

### Cách Hoạt Động

**Không bao giờ** thay đổi instances đang chạy. Luôn tạo mới hoàn toàn, sau đó loại bỏ cái cũ.

```
Version 1 đang chạy:
[v1-instance-1] [v1-instance-2] [v1-instance-3]
        
Tạo toàn bộ instances mới với v2:
[v1-instance-1] [v1-instance-2] [v1-instance-3]
[v2-instance-1] [v2-instance-2] [v2-instance-3]

Sau khi v2 healthy, route traffic:
[v2-instance-1] [v2-instance-2] [v2-instance-3]
        
Terminate v1 instances:
[v2-instance-1] [v2-instance-2] [v2-instance-3]
```

### Ưu Điểm Immutable

- **Không có configuration drift** (trôi dạt cấu hình) — mỗi deployment bắt đầu từ trạng thái sạch
- **Dễ debug** — "version này chạy gì" luôn rõ ràng
- **Rollback đơn giản** — terminate v2, route lại v1 (nếu còn giữ)
- **Predictable** — test trên instance giống hệt production

### Immutable với AWS Elastic Beanstalk

```bash
# Immutable deployment policy
eb deploy --strategy immutable

# Trong .elasticbeanstalk/config.yml:
deploy:
  artifact: myapp.jar
option_settings:
  aws:elasticbeanstalk:command:
    DeploymentPolicy: Immutable
    HealthThreshold: Ok
    IgnoreHealthCheck: false
```

### Kết Nối với Infrastructure as Code

Immutable deployment phù hợp hoàn toàn với IaC (Infrastructure as Code — Hạ Tầng Dưới Dạng Mã):

```hcl
# Terraform: Luôn tạo ASG mới khi AMI thay đổi
resource "aws_autoscaling_group" "app" {
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
  
  # instance_refresh buộc replace tất cả instances
  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 100
    }
  }
}
```

---

## 🧪 A/B Testing {#ab-testing}

A/B Testing (Kiểm Thử A/B) khác Canary ở mục đích: thay vì test stability (ổn định), A/B testing đo **business outcome** (kết quả kinh doanh).

```
Phân tách theo user segment (phân khúc người dùng):
- Nhóm A (50%): Giao diện cũ → Đo conversion rate (tỷ lệ chuyển đổi)
- Nhóm B (50%): Giao diện mới → Đo conversion rate

Sau 1-2 tuần: Phân tích kết quả thống kê → Giữ phiên bản tốt hơn
```

**Sticky Sessions (Phiên Cố Định) quan trọng cho A/B Testing:**
- Người dùng phải nhất quán thấy cùng một version
- Dùng cookie hoặc header để phân loại user

---

## 🛠️ Triển Khai trên AWS Services {#aws-deployment-tools}

### AWS CodeDeploy — Cho EC2, Lambda, ECS

| Deployment Type     | Mô Tả                                           | Hỗ Trợ                |
| ------------------- | ----------------------------------------------- | --------------------- |
| `In-place`          | Update code trực tiếp trên instance hiện tại    | EC2 / On-premises     |
| `Blue/Green`        | Tạo replacement environment                     | EC2, Lambda, ECS      |

**Deployment Config nội sẵn:**
- `CodeDeployDefault.AllAtOnce`: Deploy tất cả cùng lúc
- `CodeDeployDefault.HalfAtATime`: Deploy 50% mỗi lần
- `CodeDeployDefault.OneAtATime`: Deploy từng instance một (an toàn nhất, chậm nhất)
- `CodeDeployDefault.LambdaLinear10PercentEvery3Minutes`
- `CodeDeployDefault.LambdaCanary10Percent30Minutes`

### ECS Deployment

```json
{
  "deploymentController": {
    "type": "CODE_DEPLOY"    // Hoặc "ECS" (rolling native) hoặc "EXTERNAL"
  }
}
```

**ECS Native Rolling:** Đơn giản, không cần CodeDeploy setup. Dùng `minimumHealthyPercent` và `maximumPercent`.

**ECS với CodeDeploy Blue/Green:** Phức tạp hơn nhưng có rollback tức thì và hooks.

### Lambda Deployment với Aliases & Versions

```bash
# Publish version mới (immutable snapshot)
VERSION=$(aws lambda publish-version \
  --function-name my-function \
  --query 'Version' --output text)

# Tạo alias "production" trỏ đến version mới với Canary 10%
aws lambda update-alias \
  --function-name my-function \
  --name production \
  --function-version $VERSION \
  --routing-config "AdditionalVersionWeights={\"$OLD_VERSION\"=0.9}"
  # 10% → new version, 90% → old version
```

---

## ⚖️ So Sánh & Chọn Strategy {#so-sánh}

### Ma Trận Quyết Định

| Tình Huống                                       | Strategy Phù Hợp             |
| ------------------------------------------------ | ----------------------------- |
| Startup nhỏ, cần đơn giản                        | Rolling Update                |
| Production lớn, rollback phải tức thì            | Blue/Green                    |
| Tính năng có thể ảnh hưởng user (thận trọng)    | Canary Release                |
| Muốn test business metrics trước khi rollout     | A/B Testing                   |
| Database migration phức tạp                      | Blue/Green (với care)         |
| Lambda function updates                          | Canary / Linear (CodeDeploy)  |
| Tần suất deploy cao (nhiều lần/ngày)             | Rolling hoặc Canary tự động   |

### Decision Tree (Cây Quyết Định)

```
Có yêu cầu zero downtime tuyệt đối không?
├── Không → In-place (đơn giản, chấp nhận downtime ngắn)
└── Có → Tiếp tục...
        │
        Cần rollback tức thì (< 1 phút) không?
        ├── Không → Rolling Update (đơn giản, đủ dùng)
        └── Có → Tiếp tục...
                │
                Muốn thử nghiệm với % nhỏ trước không?
                ├── Không → Blue/Green (all-at-once switch)
                └── Có → Canary Release
```

### Chi Phí So Sánh (Tương Đối)

```
Rolling:    1x (không cần thêm capacity lâu dài)
Blue/Green: 2x trong quá trình deploy
Canary:     ~1.1-1.5x (chỉ phần nhỏ chạy song song)
Immutable:  2x trong quá trình deploy (tương tự Blue/Green)
```

---

## ❓ Câu Hỏi Phỏng Vấn

**Q: Blue/Green vs Canary — khi nào chọn cái nào?**
> Blue/Green: Cần chuyển đổi nhanh và rollback ngay (ví dụ: database migration đi kèm). Canary: Khi muốn "thăm dò" dần dần, phát hiện lỗi trước khi ảnh hưởng tất cả users, đặc biệt với tính năng mới có nhiều thay đổi logic.

**Q: Tại sao Rolling Update yêu cầu backward compatibility?**
> Trong rolling update, v1 và v2 chạy song song. Nếu v2 thay đổi API hoặc database schema theo cách không tương thích với v1, instances v1 sẽ bị lỗi khi gọi nhau hoặc đọc dữ liệu từ DB. Cần: API versioning, database migration trước code migration.

**Q: CodeDeploy Canary 10Percent5Minutes nghĩa là gì?**
> 10% traffic chuyển sang version mới, giữ trong 5 phút, theo dõi alarms. Nếu không có alarm → chuyển 100% còn lại. Nếu có alarm → auto rollback 100% về version cũ.

**Q: Làm sao rollback nhanh cho ECS Blue/Green?**
> CodeDeploy tạo Original task set và Replacement task set. Khi cần rollback, chỉ cần chuyển lại ALB target group weight về Original. Thường chỉ mất vài giây. CodeDeploy có thể tự động rollback khi CloudWatch alarm trigger.

---

**Tiếp Theo:** [5-capacity-planning.md](./5-capacity-planning.md) — Lập Kế Hoạch Năng Lực & Load Testing
