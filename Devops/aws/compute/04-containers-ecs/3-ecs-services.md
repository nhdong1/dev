# ECS Services — Dịch Vụ ECS: Deployment Strategies, Circuit Breaker, Auto Scaling

> Hướng dẫn chuyên sâu về ECS Service (Dịch Vụ ECS): rolling deployment (triển khai cuộn), blue/green deployment (triển khai xanh/lam), circuit breaker (ngắt mạch tự động), Service Auto Scaling (tự động co giãn dịch vụ) và tích hợp với Application Load Balancer.

## 📚 Mục Lục

1. [ECS Service Là Gì?](#ecs-service-là-gì)
2. [Service Configuration — Cấu Hình Service](#service-configuration--cấu-hình-service)
3. [Deployment Strategies — Chiến Lược Triển Khai](#deployment-strategies--chiến-lược-triển-khai)
4. [Rolling Deployment — Triển Khai Cuộn](#rolling-deployment--triển-khai-cuộn)
5. [Blue/Green Deployment với CodeDeploy](#bluegreen-deployment-với-codedeploy)
6. [Circuit Breaker — Ngắt Mạch Tự Động](#circuit-breaker--ngắt-mạch-tự-động)
7. [Service Auto Scaling — Tự Động Co Giãn](#service-auto-scaling--tự-động-co-giãn)
8. [Load Balancer Integration — Tích Hợp Cân Bằng Tải](#load-balancer-integration--tích-hợp-cân-bằng-tải)
9. [Service Connect và Service Discovery](#service-connect-và-service-discovery)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 ECS Service Là Gì?

ECS Service là **bộ điều khiển dài hạn** (long-running controller) đảm bảo số lượng Tasks mong muốn luôn đang chạy, tự phục hồi khi có sự cố, và orchestrate (điều phối) quá trình deploy không gây downtime.

```
ECS Service ≈ Kubernetes Deployment + Kubernetes Service kết hợp

Service làm gì:
┌──────────────────────────────────────────────────────────┐
│  1. Duy trì desired_count tasks đang chạy               │
│     → Task fail? Launch task mới thay thế               │
│                                                          │
│  2. Load Balancer integration                            │
│     → Register task IPs vào ALB Target Group             │
│     → Deregister tasks không healthy                     │
│                                                          │
│  3. Deployment orchestration                             │
│     → Rolling: thay thế dần tasks cũ bằng mới           │
│     → Blue/Green: switch traffic sau khi tests pass      │
│                                                          │
│  4. Circuit Breaker (Ngắt Mạch)                         │
│     → Rollback tự động khi deploy quá nhiều lỗi         │
│                                                          │
│  5. Auto Scaling                                         │
│     → Scale out/in dựa trên CloudWatch metrics          │
└──────────────────────────────────────────────────────────┘
```

### Service vs Standalone Task

```
ECS Service:                      Standalone Task (Task Đơn Lẻ):
• Long-running (chạy mãi mãi)     • One-off (chạy một lần)
• Tự restart khi fail             • Không restart khi fail
• Tích hợp Load Balancer          • Không có Load Balancer
• Hỗ trợ Auto Scaling             • Không có Auto Scaling
• Ví dụ: Web API, Worker          • Ví dụ: DB migration, batch job
```

---

## ⚙️ Service Configuration — Cấu Hình Service

### Tạo ECS Service Với AWS CLI

```bash
aws ecs create-service \
  --cluster cluster-production \
  --service-name web-api \
  --task-definition web-api:5 \
  --desired-count 3 \
  --launch-type FARGATE \
  --deployment-configuration '{
    "maximumPercent": 200,
    "minimumHealthyPercent": 100,
    "deploymentCircuitBreaker": {
      "enable": true,
      "rollback": true
    }
  }' \
  --load-balancers '[{
    "targetGroupArn": "arn:aws:elasticloadbalancing:...",
    "containerName": "web-api",
    "containerPort": 8080
  }]' \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-aaa", "subnet-bbb"],
      "securityGroups": ["sg-ccc"],
      "assignPublicIp": "DISABLED"
    }
  }'
```

### Các Tham Số Quan Trọng

```
desired_count: 3
    → ECS luôn cố gắng duy trì đúng 3 tasks

maximumPercent: 200
    → Tối đa 200% desired_count = 6 tasks trong rolling deployment
    → Không thể > 200% để tránh quá tải

minimumHealthyPercent: 100
    → Tối thiểu 100% healthy tasks trong khi deploy
    → Không bao giờ drop dưới 3 healthy tasks
    → Đảm bảo không có downtime

Tổng hợp: Trong rolling deploy, tối đa 6 tasks tồn tại đồng thời
           và luôn có ≥ 3 tasks healthy serving traffic
```

---

## 🚀 Deployment Strategies — Chiến Lược Triển Khai

### So Sánh 3 Chiến Lược

```
┌────────────────────┬──────────────────┬──────────────────┬──────────────────┐
│                    │ ROLLING UPDATE   │  BLUE/GREEN      │  RECREATE        │
│                    │ (Cuộn)           │  (Xanh/Lam)      │  (Tạo Lại)       │
├────────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Downtime           │ Không (zero)     │ Không (zero)     │ Có (downtime)    │
│ Traffic Switching  │ Dần dần          │ Ngay lập tức     │ N/A              │
│ Rollback Speed     │ Chậm (vài phút)  │ Nhanh (<1 phút)  │ Chậm             │
│ Cần thêm infra?    │ Không            │ Có (2x tasks)    │ Không            │
│ Testing trước?     │ Không            │ Có thể           │ Không            │
│ Phức tạp           │ Thấp             │ Cao              │ Thấp             │
│ Chi phí            │ 1x - 2x          │ 2x trong deploy  │ Gián đoạn        │
│ Use case           │ Hầu hết apps     │ Critical services│ Dev/Staging only │
└────────────────────┴──────────────────┴──────────────────┴──────────────────┘
```

---

## 🔄 Rolling Deployment — Triển Khai Cuộn

### Cơ Chế Hoạt Động

```
Trước Deploy: 3 tasks (v1.0) đang nhận traffic

Time → ──────────────────────────────────────────────►

Tasks v1.0: ████████████████░░░░░░░░░░░░░░░░░░░░░░░░
Tasks v1.1: ░░░░░░░░░░░░░░░░████████████████████████
Total tasks:      3      →      6      →      3

Bước 1: ECS launch 3 tasks v1.1 (total: 6 tasks)
Bước 2: Wait v1.1 tasks pass health check
Bước 3: Register v1.1 tasks vào ALB target group
Bước 4: Deregister v1.0 tasks từ ALB
Bước 5: Stop v1.0 tasks
Bước 6: Total trở về 3 tasks (tất cả v1.1)
```

### Tuning Parameters — Điều Chỉnh Tham Số

```
Scenario 1: Zero Downtime, Nhanh
maximumPercent: 200
minimumHealthyPercent: 100
→ Launch toàn bộ tasks mới trước, stop toàn bộ tasks cũ sau
→ Không downtime, nhưng tốn 2x resources trong lúc deploy

Scenario 2: Tiết Kiệm Chi Phí, Có Thể Giảm Throughput
maximumPercent: 150
minimumHealthyPercent: 50
→ Với 4 tasks desired: tối đa 6, tối thiểu 2
→ Deploy từng nhóm nhỏ, tiết kiệm resources hơn

Scenario 3: Dev/Test Environment, Nhanh Nhất
maximumPercent: 100
minimumHealthyPercent: 0
→ Stop toàn bộ tasks cũ trước, launch tasks mới sau
→ Có downtime ngắn, không lãng phí resources
→ KHÔNG dùng cho production
```

### Graceful Shutdown — Tắt Nhẹ Nhàng

```
Khi ECS stop một task:
1. ECS gửi SIGTERM đến container
2. App nhận SIGTERM, bắt đầu graceful shutdown:
   • Stop nhận request mới
   • Finish các request đang xử lý
   • Close DB connections
   • Flush logs
3. Nếu container vẫn chạy sau stopTimeout → SIGKILL

Container Timeout Settings:
• ECS_CONTAINER_STOP_TIMEOUT: 30 giây mặc định (tăng nếu cần)
• ALB deregistration delay: 300 giây (nên giảm xuống 30-60s cho fast deploy)

Code Example (Node.js):
process.on('SIGTERM', async () => {
  console.log('Received SIGTERM, starting graceful shutdown...');
  server.close(() => {
    console.log('HTTP server closed');
    process.exit(0);
  });
  // Finish in-flight requests, close DB connections
});
```

---

## 🔵 Blue/Green Deployment với CodeDeploy

### Kiến Trúc Blue/Green

```
TRƯỚC DEPLOY:
Internet → ALB → Target Group BLUE → 3 Tasks (v1.0) [Blue]

QUÁ TRÌNH DEPLOY:
1. ECS/CodeDeploy tạo Target Group GREEN
2. Launch 3 Tasks mới (v1.1) → đăng ký vào GREEN
3. Wait health checks GREEN pass
4. (Tùy chọn) Chạy automated tests trên GREEN
5. CodeDeploy switch ALB: 100% traffic → GREEN
6. (Tùy chọn) Chờ bake time (thời gian "nướng") để verify
7. Terminate Tasks cũ (BLUE)

SAU DEPLOY:
Internet → ALB → Target Group GREEN → 3 Tasks (v1.1) [Green]

ROLLBACK:
• Bất kỳ lúc nào trước khi terminate BLUE:
  → Switch ALB ngược lại: 100% traffic → BLUE
  → < 1 phút rollback
```

### CodeDeploy AppSpec File

```yaml
# appspec.yaml cho ECS Blue/Green
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "arn:aws:ecs:...:task-definition/web-api:5"
        LoadBalancerInfo:
          ContainerName: "web-api"
          ContainerPort: 8080
        PlatformVersion: "LATEST"

Hooks:
  - BeforeInstall: "arn:aws:lambda:...:function:pre-deploy-validation"
  - AfterInstall: "arn:aws:lambda:...:function:smoke-tests"
  - AfterAllowTestTraffic: "arn:aws:lambda:...:function:integration-tests"
  - BeforeAllowTraffic: "arn:aws:lambda:...:function:final-validation"
  - AfterAllowTraffic: "arn:aws:lambda:...:function:post-deploy-metrics"
```

### Traffic Shifting — Chuyển Dần Traffic

```
CodeDeploy Linear Strategy (Chiến Lược Tuyến Tính):
• Linear10PercentEvery1Minute: +10% traffic/phút
• Linear10PercentEvery3Minutes: +10% traffic/3 phút

CodeDeploy Canary Strategy (Chiến Lược Canary):
• Canary10Percent5Minutes: 10% trong 5 phút, rồi 100%
• Canary10Percent15Minutes: 10% trong 15 phút, rồi 100%

All-at-Once (Toàn Bộ Cùng Lúc):
• 0% → 100% ngay lập tức
• Nhanh nhất, risk cao nhất

Ví Dụ Canary10Percent5Minutes:

Thời gian: 0 phút           5 phút
BLUE (v1.0): ████████████░░░░░░
GREEN (v1.1): ░░░░░░░░░░░░████████

Traffic phân phối:
• 0-5 phút: 10% → GREEN, 90% → BLUE
• Sau 5 phút: 100% → GREEN (nếu không có lỗi)
```

---

## ⚡ Circuit Breaker — Ngắt Mạch Tự Động

### Vấn Đề Không Có Circuit Breaker

```
Tình Huống: Deploy version mới có bug khiến container crash sau 10 giây

Không Có Circuit Breaker:
1. ECS launch task mới (v1.1)
2. Task khởi động, health check pass (bug chưa triggered)
3. 10 giây sau: task crash, exit code 1
4. ECS thấy task STOPPED → launch task mới (v1.1)
5. Task crash lại sau 10 giây
6. ECS launch lại... crash lại...
7. Sau 30 phút: vẫn đang crash loop
8. Kết quả: production có thể không có đủ tasks
9. Cần manual intervention để rollback

Với Circuit Breaker:
1. ECS theo dõi số task failures trong sliding window
2. Sau N failures (mặc định ~10): Circuit Breaker trigger
3. ECS dừng deploy, không launch thêm tasks v1.1
4. Tự động rollback về Task Definition cũ (v1.0)
5. Alert gửi qua CloudWatch Event → SNS → PagerDuty
6. Toàn bộ tự động, không cần can thiệp thủ công
```

### Circuit Breaker Configuration

```bash
# Bật Circuit Breaker khi tạo/update service
aws ecs update-service \
  --cluster cluster-prod \
  --service web-api \
  --deployment-configuration '{
    "deploymentCircuitBreaker": {
      "enable": true,
      "rollback": true
    }
  }'

# Xem trạng thái deployment
aws ecs describe-services \
  --cluster cluster-prod \
  --services web-api \
  --query 'services[0].deployments'
```

### Deployment Status States

```
IN_PROGRESS:  Deployment đang diễn ra
COMPLETED:    Deployment thành công
FAILED:       Deployment thất bại (circuit breaker trigger)
STOPPED:      Deployment bị dừng thủ công

Deployment Failure Reasons:
• TASK_FAILED_TO_START: Container không thể start
• TASK_REPEATEDLY_FAILED: Crash loop
• LOAD_BALANCER_NOT_IN_SERVICE: Health check fail liên tục
• DEPLOYMENT_CIRCUIT_BREAKER_ROLLBACK_REQUESTED: Rollback thủ công
```

### CloudWatch Alarm Cho Deployment

```json
{
  "AlarmName": "ecs-deployment-failed",
  "AlarmDescription": "ECS Service deployment failed",
  "MetricName": "DeploymentFailed",
  "Namespace": "ECS/ContainerInsights",
  "Statistic": "Sum",
  "Period": 300,
  "EvaluationPeriods": 1,
  "Threshold": 1,
  "ComparisonOperator": "GreaterThanOrEqualToThreshold",
  "AlarmActions": ["arn:aws:sns:...:ecs-alerts"]
}
```

---

## 📈 Service Auto Scaling — Tự Động Co Giãn

### Application Auto Scaling Cho ECS

ECS Service tích hợp với **Application Auto Scaling** (Tự Động Co Giãn Ứng Dụng) để tự động điều chỉnh `desired_count` dựa trên CloudWatch metrics.

```
Kiến Trúc Auto Scaling:

CloudWatch Alarm (CPU > 70%) → Application Auto Scaling → ECS Service
                                                         desired_count: 3 → 6

CloudWatch Alarm (CPU < 30%) → Application Auto Scaling → ECS Service
                                                         desired_count: 6 → 3
```

### Target Tracking Scaling — Theo Dõi Mục Tiêu (Khuyến Nghị)

```bash
# Scale để giữ CPU Utilization ở 70%
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/cluster-prod/web-api \
  --policy-name cpu-tracking-policy \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }'
```

**Predefined Metrics Cho ECS:**
```
ECSServiceAverageCPUUtilization    ← % CPU của service
ECSServiceAverageMemoryUtilization ← % Memory của service
ALBRequestCountPerTarget           ← Request/giây per task (cần ALB)
```

### Step Scaling — Co Giãn Theo Bước

```bash
# Định nghĩa step: +2 tasks nếu CPU 70-80%, +4 tasks nếu CPU >80%
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/cluster-prod/web-api \
  --policy-name step-scale-out \
  --policy-type StepScaling \
  --step-scaling-policy-configuration '{
    "AdjustmentType": "ChangeInCapacity",
    "StepAdjustments": [
      {
        "MetricIntervalLowerBound": 0,
        "MetricIntervalUpperBound": 10,
        "ScalingAdjustment": 2
      },
      {
        "MetricIntervalLowerBound": 10,
        "ScalingAdjustment": 4
      }
    ],
    "Cooldown": 60
  }'
```

### Scheduled Scaling — Co Giãn Theo Lịch

```bash
# Scale up trước giờ cao điểm sáng (8:00 AM UTC+7 = 1:00 AM UTC)
aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/cluster-prod/web-api \
  --scheduled-action-name scale-up-morning \
  --schedule "cron(0 1 * * ? *)" \
  --scalable-target-action '{
    "MinCapacity": 5,
    "MaxCapacity": 20
  }'

# Scale down sau giờ thấp điểm đêm (11:00 PM UTC+7 = 4:00 PM UTC)
aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/cluster-prod/web-api \
  --scheduled-action-name scale-down-night \
  --schedule "cron(0 16 * * ? *)" \
  --scalable-target-action '{
    "MinCapacity": 2,
    "MaxCapacity": 10
  }'
```

### Scale-In Protection — Bảo Vệ Khỏi Scale In

```bash
# Ngăn ECS terminate task đang xử lý job dài hạn
aws ecs update-container-instances-state \
  --cluster cluster-prod \
  --container-instances <instance-id> \
  --status DRAINING

# Programmatic scale-in protection (tốt hơn cho worker tasks)
# Trong application code:
import boto3
ecs = boto3.client('ecs')

# Trước khi bắt đầu job dài
ecs.update_task_protection(
    cluster='cluster-prod',
    tasks=['task-id'],
    protectionEnabled=True,
    expiresInMinutes=60
)

# Sau khi hoàn thành job
ecs.update_task_protection(
    cluster='cluster-prod',
    tasks=['task-id'],
    protectionEnabled=False
)
```

---

## ⚖️ Load Balancer Integration — Tích Hợp Cân Bằng Tải

### ECS + ALB Architecture

```
Internet
   │
   ▼
ALB (Application Load Balancer)
   │
   ├── Listener HTTP:80 → Redirect to HTTPS
   └── Listener HTTPS:443 → Forward to Target Group
                                    │
                                    ▼
                          Target Group "web-api"
                          ┌──────────────────────────┐
                          │  10.0.1.5:8080  ← Task 1 │
                          │  10.0.2.7:8080  ← Task 2 │
                          │  10.0.3.9:8080  ← Task 3 │
                          └──────────────────────────┘

• ECS tự động register/deregister task IPs
• ALB health check → quyết định route traffic
• Stickiness (dính): tùy chọn, khuyến nghị stateless app
```

### ALB Target Group Configuration

```bash
# Tạo Target Group cho ECS với awsvpc mode
aws elbv2 create-target-group \
  --name web-api-tg \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-xxx \
  --target-type ip \
  --health-check-protocol HTTP \
  --health-check-path /health \
  --health-check-interval-seconds 15 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --deregistration-delay-attributes \
    "deregistration_delay.timeout_seconds=30"
```

**Target Type IP vs Instance:**
```
ip (awsvpc mode):
• Target = IP address của ENI của Task
• ALB giao tiếp trực tiếp với task qua IP
• Phù hợp với awsvpc network mode (Fargate)

instance (bridge mode):
• Target = EC2 instance ID
• ALB giao tiếp qua EC2 host, dynamic port forwarding
• Phù hợp với bridge network mode
```

### Deregistration Delay — Độ Trễ Hủy Đăng Ký

```
Vấn Đề Không Có Deregistration Delay:
1. ECS quyết định stop task
2. ECS ngay lập tức deregister task từ ALB
3. ALB vẫn đang route requests đến task
4. Task bị stop → 502 Bad Gateway

Với Deregistration Delay:
1. ECS quyết định stop task
2. ALB stop routing NEW requests đến task
3. Task vẫn chạy thêm N giây (deregistration delay)
4. Các in-flight requests hoàn thành
5. Sau N giây: Task bị terminate

Best Practice:
• deregistration_delay: 30-60 giây (không cần 300 mặc định)
• Phải match với SIGTERM grace period của app
• Kết hợp với graceful shutdown trong code
```

---

## 🔗 Service Connect và Service Discovery

### Service Discovery — Khám Phá Dịch Vụ Với Route 53

```
Cơ Chế:
1. ECS tạo Route 53 Private Hosted Zone
2. Mỗi Task được đăng ký một DNS record:
   web-api.local → 10.0.1.5
   web-api.local → 10.0.2.7
3. Services khác gọi: http://web-api.local:8080

Nhược Điểm:
• DNS caching gây delay khi task fail/replace
• Không có load balancing built-in (chỉ DNS round-robin)
• Không có circuit breaking, retries tự động
• Khó quan sát (không có metrics per-service)
```

### ECS Service Connect — Kết Nối Dịch Vụ (Khuyến Nghị)

```
ECS Service Connect ≈ Service Mesh nhẹ, tích hợp sẵn vào ECS

Tính Năng:
• Service-to-service discovery qua DNS name ngắn
• Built-in proxy (Envoy-based) cho mỗi task
• CloudWatch metrics tự động: request rate, latency, errors
• Tích hợp với X-Ray distributed tracing
• Health-based routing: tự bypass unhealthy tasks
• Retry logic tự động

Kiến Trúc Service Connect:
Task A (web-api)                Task B (payment-service)
┌────────────────────┐          ┌────────────────────┐
│ ┌────────────────┐ │          │ ┌────────────────┐ │
│ │  app container │ │          │ │  app container │ │
│ │ :8080          │ │          │ │ :9090          │ │
│ └───────┬────────┘ │          │ └────────────────┘ │
│         │ localhost│          │   ▲                │
│ ┌───────▼────────┐ │          │ ┌─┴──────────────┐ │
│ │ Service Connect│ │ ──────►  │ │ Service Connect│ │
│ │ Proxy (Envoy)  │ │ HTTPS    │ │ Proxy (Envoy)  │ │
│ └────────────────┘ │          │ └────────────────┘ │
└────────────────────┘          └────────────────────┘

web-api gọi: http://payment-service → Proxy handle kết nối
```

### Cấu Hình Service Connect

```json
{
  "serviceConnectConfiguration": {
    "enabled": true,
    "namespace": "production",
    "services": [
      {
        "portName": "web-api-8080-tcp",
        "clientAliases": [
          {
            "port": 8080,
            "dnsName": "web-api"
          }
        ]
      }
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/service-connect",
        "awslogs-region": "ap-southeast-1"
      }
    }
  }
}
```

---

## ❓ Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: minimumHealthyPercent và maximumPercent trong ECS Service có nghĩa gì?**
> `minimumHealthyPercent` là % tối thiểu tasks phải healthy trong quá trình deploy. Ví dụ: 3 tasks desired, minimumHealthyPercent=100 → luôn có ít nhất 3 tasks healthy, không có downtime. `maximumPercent` là % tối đa tasks cho phép tồn tại. maximumPercent=200 → tối đa 6 tasks cùng lúc (khi launch 3 tasks mới, chưa stop 3 tasks cũ). Hai tham số này kiểm soát tốc độ và tính an toàn của deployment.

**Q: Circuit Breaker trong ECS Service hoạt động thế nào?**
> ECS theo dõi số task launch failures trong một deployment. Nếu deployment mới có quá nhiều tasks fail health check hoặc crash (ngưỡng mặc định khoảng 10 failures), Circuit Breaker trigger: dừng deployment và (nếu `rollback: true`) tự động cập nhật service để quay về Task Definition trước đó. Không cần can thiệp thủ công — phù hợp cho CI/CD pipeline.

**Q: Sự khác biệt giữa rolling deployment và blue/green deployment?**
> Rolling: Thay thế dần tasks cũ bằng tasks mới trong cùng một target group. Traffic dần dần shift khi tasks cũ bị deregister. Rollback chậm vì phải redeploy. Blue/Green: Launch toàn bộ tasks mới (green) vào target group riêng, sau khi healthy mới switch traffic 100% từ blue sang green. Rollback cực nhanh (< 1 phút) vì blue fleet vẫn còn. Blue/Green tốt hơn cho critical services nhưng tốn 2x resources trong lúc deploy.

### Câu Hỏi Nâng Cao

**Q: Tại sao deregistration delay quan trọng và nên đặt bao nhiêu?**
> Khi ECS stop một task, có khoảng thời gian ALB vẫn đang route requests tới task đó (do ALB chưa kịp cập nhật backend list). Nếu không có deregistration delay, task bị kill khi vẫn đang xử lý request → 502/504 errors. Nên đặt bằng maximum expected request processing time + buffer, thường 30-60 giây. Giá trị mặc định 300 giây quá dài, làm chậm deployment.

**Q: Khi nào dùng Service Connect thay vì Service Discovery?**
> Với ứng dụng microservices mới, **luôn dùng Service Connect** vì cung cấp: (1) built-in retries và health-based routing, (2) CloudWatch metrics tự động cho service-to-service traffic, (3) tích hợp X-Ray tracing. Service Discovery (Route 53 DNS) chỉ nên dùng khi cần backward compatibility hoặc khi services non-ECS cũng cần resolve DNS name.

**Q: Scale-In Protection (Bảo Vệ Khỏi Scale In) dùng khi nào?**
> Khi task đang xử lý long-running job mà không thể interrupt giữa chừng (ví dụ: video transcoding, data export). Nếu Auto Scaling quyết định scale in và terminate task đó, job sẽ mất. Giải pháp: trước khi bắt đầu job, gọi API `UpdateTaskProtection` để protect task. Sau khi job hoàn thành, remove protection. Task sẽ không bị terminate trong thời gian protected.

---

**Tiếp Theo:** [4-fargate-vs-ec2.md](4-fargate-vs-ec2.md) — Fargate vs EC2 Launch Type: trade-offs chi tiết và hướng dẫn chọn.
