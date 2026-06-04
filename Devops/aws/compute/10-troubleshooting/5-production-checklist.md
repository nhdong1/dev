# ✅ Production Checklist — Checklist Trước Khi Go-Live

> Danh sách kiểm tra toàn diện trước khi triển khai ứng dụng lên môi trường production — bao gồm bảo mật, monitoring, khả năng chịu lỗi, hiệu suất, và kế hoạch rollback.

## 📚 Mục Lục

1. [Checklist Hạ Tầng — Infrastructure](#1-checklist-hạ-tầng)
2. [Checklist Bảo Mật — Security](#2-checklist-bảo-mật)
3. [Checklist Monitoring & Alerting](#3-checklist-monitoring--alerting)
4. [Checklist Hiệu Suất — Performance](#4-checklist-hiệu-suất)
5. [Checklist Khả Năng Phục Hồi — Resilience](#5-checklist-khả-năng-phục-hồi)
6. [Kế Hoạch Rollback — Rollback Plan](#6-kế-hoạch-rollback)
7. [Post-Deployment Validation — Xác Nhận Sau Triển Khai](#7-post-deployment-validation)

---

## 1. Checklist Hạ Tầng

### EC2 & Auto Scaling

```
□ Auto Scaling Group có ít nhất 2 AZ (Availability Zone — Vùng Khả Dụng)
□ MinSize >= 2 để đảm bảo luôn có instance dự phòng
□ MaxSize đủ lớn để xử lý tải peak (Đỉnh Tải Tính Toán)
□ Health Check Grace Period đủ dài (≥ ứng dụng khởi động)
□ ELB (Elastic Load Balancer — Cân Bằng Tải) health check path trả về 200
□ Launch Template đang dùng AMI được patched mới nhất
□ Instance store data không chứa trạng thái quan trọng (ephemeral)
□ EBS volumes có encryption bật
□ EC2 Auto Recovery được cấu hình
□ Deployment strategy là Rolling hoặc Blue/Green (không phải All-at-Once)
```

### Lambda

```
□ Timeout được đặt đúng (không quá ngắn, không để mặc định 3 giây cho function nặng)
□ Memory được điều chỉnh qua Lambda Power Tuning hoặc kinh nghiệm thực tế
□ Reserved Concurrency (Đồng Thời Dự Phòng) được đặt nếu cần bảo vệ downstream
□ Dead Letter Queue (Hàng Đợi Thư Chết — DLQ) hoặc Destinations được cấu hình cho async invocations
□ Provisioned Concurrency cho functions nhạy cảm với latency
□ Lambda Versions và Aliases được dùng cho phép rollback dễ dàng
□ VPC endpoints được cấu hình nếu Lambda trong VPC cần gọi AWS services
□ Error rate alarm được thiết lập
□ Concurrency alarm được thiết lập (> 80% reserved limit)
```

### ECS / Fargate

```
□ Service chạy trên ít nhất 2 AZ
□ Desired count >= 2 (không single point of failure)
□ Deployment Circuit Breaker (Cơ Chế Ngắt Mạch) được bật
□ Deployment với rollback on alarm được cấu hình
□ Task CPU/Memory phù hợp với workload thực tế
□ Container health check endpoint hoạt động đúng
□ ECS Task Role chỉ có minimum permissions cần thiết
□ ECS Exec được bật cho môi trường non-production (debug khi cần)
□ Log driver là awslogs với retention policy
□ Image tag không dùng :latest — dùng immutable tag (git SHA hoặc version)
```

### EKS

```
□ Cluster có ít nhất 3 nodes trải đều qua 3 AZ
□ Node group có Auto Scaling (min/max/desired)
□ RBAC (Role-Based Access Control — Kiểm Soát Truy Cập Dựa Trên Vai Trò) được cấu hình đúng
□ IRSA (IAM Roles for Service Accounts — Vai Trò IAM Cho Service Accounts) thay vì node-level IAM role
□ Pod Disruption Budget (Ngân Sách Gián Đoạn Pod — PDB) được đặt cho critical services
□ Resource requests và limits được đặt cho mọi container
□ Liveness và Readiness Probes được cấu hình
□ Network Policies (Chính Sách Mạng) được áp dụng
□ Cluster Autoscaler hoặc Karpenter (Tự Động Mở Rộng Cluster) được cài
□ HPA (Horizontal Pod Autoscaler) cho services cần scale
□ EKS managed node groups đang dùng AMI mới nhất
```

---

## 2. Checklist Bảo Mật

### IAM & Permissions

```
□ Không có hardcoded credentials (thông tin xác thực mã cứng) trong code hoặc config
□ EC2/Lambda/ECS/EKS dùng IAM Roles, không dùng Access Keys
□ IAM Roles tuân theo Principle of Least Privilege (Nguyên Tắc Đặc Quyền Tối Thiểu)
□ Không có wildcard (*) trong IAM actions nếu có thể tránh
□ MFA (Multi-Factor Authentication — Xác Thực Đa Yếu Tố) bật cho IAM users quan trọng
□ Root account không có Access Keys
□ AWS Organizations SCP (Service Control Policies) được dùng nếu multi-account
```

### Network Security

```
□ Security Groups tuân theo least privilege (chỉ mở port cần thiết)
□ Không có Security Group mở 0.0.0.0/0 cho port 22 (SSH) hoặc 3389 (RDP)
□ Dùng SSM Session Manager (Quản Lý Phiên) thay vì SSH trực tiếp
□ VPC có Flow Logs (Nhật Ký Luồng) được bật
□ Public và Private subnets được tách biệt rõ ràng
□ Database, cache chỉ nằm trong Private Subnets
□ NACLs (Network ACLs) được cấu hình nếu cần thêm tầng bảo mật
□ WAF (Web Application Firewall — Tường Lửa Ứng Dụng Web) được gắn vào ALB/CloudFront
```

### Data & Encryption

```
□ EBS volumes được encrypt với AWS KMS (Key Management Service)
□ S3 buckets có SSE (Server-Side Encryption — Mã Hóa Phía Máy Chủ) bật
□ RDS instances được encrypt
□ Data in transit được mã hóa bằng TLS 1.2+
□ Secrets Manager hoặc Parameter Store SecureString cho mọi secret
□ Không có secrets trong environment variables dạng plain text
□ S3 buckets không có Public Access (trừ khi có lý do rõ ràng)
□ CloudTrail (Nhật Ký Kiểm Toán) được bật cho mọi region
```

### Compliance

```
□ AWS Config rules được bật để phát hiện configuration drift
□ AWS Inspector (Công Cụ Kiểm Tra Bảo Mật) quét vulnerabilities trên EC2/Lambda
□ Security Hub (Trung Tâm Bảo Mật) tổng hợp findings từ nhiều nguồn
□ GuardDuty (Phát Hiện Mối Đe Dọa) được bật
□ Patch Management (Quản Lý Vá Lỗi) schedule được thiết lập
```

---

## 3. Checklist Monitoring & Alerting

### CloudWatch Metrics — Chỉ Số Cần Theo Dõi

```
EC2:
  □ CPUUtilization > 80% (cảnh báo sớm) / > 95% (critical)
  □ StatusCheckFailed_Instance > 0
  □ StatusCheckFailed_System > 0
  □ DiskReadOps/WriteOps bất thường
  □ NetworkIn/Out để phát hiện traffic anomaly

Auto Scaling:
  □ GroupDesiredCapacity vs GroupInServiceInstances (phát hiện launch failures)
  □ Scaling activities trong activity log

Lambda:
  □ Errors > X per minute
  □ Duration P99 > X ms
  □ Throttles > 0
  □ ConcurrentExecutions > 80% reserved limit
  □ IteratorAge (SQS/Kinesis) tăng cao

ECS:
  □ CPUUtilization > 70% (trigger scale out)
  □ MemoryUtilization > 80%
  □ RunningTaskCount giảm xuống dưới desired
  □ ALB TargetResponseTime > threshold
  □ ALB HTTPCode_Target_5XX_Count

EKS:
  □ Node CPU/Memory utilization
  □ Pod OOM events
  □ API Server errors
```

### Alarms & Notifications

```
□ SNS Topics (Chủ Đề Thông Báo) được cấu hình để gửi alert
□ Composite Alarms (Cảnh Báo Tổng Hợp) cho P0 scenarios
□ PagerDuty/OpsGenie integration nếu cần on-call rotation
□ CloudWatch Dashboard (Bảng Điều Khiển) cho war room view
□ Alarm actions: Scale-out policy, EC2 Auto Recovery
□ Alarm notifications đến đúng team channel (Slack, email)
□ Alarm suppression windows cho maintenance periods
```

### Logging

```
□ CloudWatch Log Groups cho tất cả services (EC2, Lambda, ECS, EKS)
□ Log retention policy được đặt (không retain mãi mãi — tốn chi phí)
□ Structured logging (JSON) để Log Insights query dễ hơn
□ Error logs được filter và alert
□ Access logs của Load Balancer được bật và gửi vào S3
□ VPC Flow Logs cho network troubleshooting
□ CloudTrail logs cho audit trail
```

### Tracing

```
□ AWS X-Ray (Theo Dõi Phân Tán) được bật cho Lambda/ECS/API Gateway
□ Service Map cho thấy toàn bộ request flow
□ X-Ray sampling rule được cấu hình (không sample 100% trong production)
□ Custom annotations/metadata để filter traces
```

---

## 4. Checklist Hiệu Suất

### Load Testing — Kiểm Tra Tải

```
□ Load test đã được thực hiện với ít nhất 1.5x expected peak traffic
□ Kết quả load test document rõ ràng (latency P50/P95/P99, error rate)
□ Auto Scaling đã được verify là hoạt động đúng khi tải tăng
□ Database connection pool không bị exhausted khi tải cao
□ Cache hit rate > threshold kỳ vọng

Tools để load test:
  - Apache JMeter (Công Cụ Kiểm Tra Tải Apache)
  - Locust (Python-based load testing)
  - k6 (Modern load testing tool)
  - AWS Distributed Load Testing Solution
```

### Caching Strategy — Chiến Lược Bộ Nhớ Đệm

```
□ ElastiCache (Redis/Memcached) được dùng cho dữ liệu truy cập thường xuyên
□ CloudFront (CDN — Mạng Phân Phối Nội Dung) cho static assets
□ Lambda/ECS reuse connections và SDK clients ngoài handler
□ DynamoDB DAX (DynamoDB Accelerator — Bộ Tăng Tốc DynamoDB) nếu latency quan trọng
□ Cache invalidation strategy được define rõ ràng
```

### Database

```
□ RDS Multi-AZ được bật (không phải Single-AZ trong production)
□ Read Replicas cho read-heavy workloads
□ Connection pooling (RDS Proxy — Proxy RDS) cho Lambda/serverless
□ Database indexes cho các query phổ biến
□ Slow query log được bật và monitored
□ Database maintenance window không trùng business hours
```

---

## 5. Checklist Khả Năng Phục Hồi

### High Availability — Tính Sẵn Sàng Cao

```
□ Không có single point of failure (Điểm Lỗi Duy Nhất) trong architecture
□ Multi-AZ deployment cho tất cả stateful services
□ Load Balancer distributes traffic qua nhiều healthy targets
□ Graceful degradation (Xuống Cấp Duyên Dáng) khi service phụ thuộc không available
□ Circuit Breaker pattern được implement cho external calls
□ Retry với exponential backoff cho transient failures
□ Timeout được đặt cho mọi external calls (không để mặc định vô hạn)
```

### Disaster Recovery — Khắc Phục Thảm Họa

```
□ RTO (Recovery Time Objective — Mục Tiêu Thời Gian Phục Hồi) đã được define
□ RPO (Recovery Point Objective — Mục Tiêu Điểm Phục Hồi) đã được define
□ Backup được tự động hóa và được test restore
□ EBS Snapshot schedule được cấu hình
□ RDS automated backups được bật với retention phù hợp
□ Multi-Region strategy (nếu RPO/RTO yêu cầu)
□ Runbook (Sách Hướng Dẫn Vận Hành) cho các failure scenarios phổ biến được viết
```

### Testing

```
□ Health check endpoints không trả về false positives
□ Chaos Engineering (Kỹ Thuật Hỗn Loạn) đã được thực hiện ít nhất ở mức cơ bản
□ Failover đã được test: Stop instance trong một AZ, verify traffic chuyển sang AZ khác
□ Scale out đã được test: Tăng tải, verify ASG/HPA thêm instances
□ Rollback đã được test: Deploy version cũ, verify rollback thành công
```

---

## 6. Kế Hoạch Rollback

### Chuẩn Bị Rollback Trước Khi Deploy

```
Trước mỗi deployment production, document sẵn:
  1. Version/commit đang chạy hiện tại
  2. Lệnh để rollback về version trước
  3. Thời gian rollback ước tính
  4. Người chịu trách nhiệm approve rollback
  5. Điều kiện trigger rollback tự động
```

### Lệnh Rollback Nhanh Theo Dịch Vụ

**Lambda:**
```bash
# Rollback bằng alias pointer (nhanh nhất — gần như instant)
aws lambda update-alias \
  --function-name my-function \
  --name production \
  --function-version PREVIOUS_VERSION_NUMBER

# Ví dụ: Đang dùng version 42, rollback về 41
aws lambda update-alias \
  --function-name my-function \
  --name production \
  --function-version 41
```

**ECS:**
```bash
# Rollback về task definition revision trước
PREVIOUS_REVISION=$(aws ecs describe-task-definition \
  --task-definition my-task-def \
  --query 'taskDefinition.revision' \
  --output text)
PREVIOUS_REVISION=$((PREVIOUS_REVISION - 1))

aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --task-definition "my-task-def:${PREVIOUS_REVISION}" \
  --force-new-deployment
```

**EKS:**
```bash
# Rollback deployment về revision trước
kubectl rollout undo deployment/my-deployment -n my-namespace

# Rollback về revision cụ thể
kubectl rollout history deployment/my-deployment -n my-namespace
kubectl rollout undo deployment/my-deployment --to-revision=3 -n my-namespace

# Xem trạng thái rollback
kubectl rollout status deployment/my-deployment -n my-namespace
```

**EC2 / Auto Scaling:**
```bash
# Rollback Launch Template về version trước
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --launch-template LaunchTemplateId=lt-xxxxxxxx,Version=PREVIOUS_VERSION

# Trigger instance refresh để replace instances hiện tại
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name my-asg \
  --preferences '{"MinHealthyPercentage": 90, "InstanceWarmup": 60}'
```

### Điều Kiện Trigger Rollback Tự Động

```
Xác định trước khi deploy:
  □ Error rate > X% trong Y phút → Auto rollback
  □ P99 latency > X ms trong Y phút → Alert + manual rollback
  □ Health check failure rate > X% → Auto rollback (ECS circuit breaker)
  □ Business metric giảm > X% (conversion rate, orders) → Manual rollback

CodeDeploy Automatic Rollback:
  aws deploy update-deployment-group \
    --application-name my-app \
    --deployment-group-name production \
    --auto-rollback-configuration '{
      "enabled": true,
      "events": ["DEPLOYMENT_FAILURE", "DEPLOYMENT_STOP_ON_ALARM"]
    }' \
    --alarm-configuration '{
      "enabled": true,
      "alarms": [{"name": "high-error-rate"}]
    }'
```

---

## 7. Post-Deployment Validation

### Synthetic Monitoring — Giám Sát Tổng Hợp

```bash
# CloudWatch Synthetics — Canary (Tự Động Kiểm Tra API/UI định kỳ)
# Chạy script Canary mỗi 5 phút để verify critical user flows

aws synthetics create-canary \
  --name production-health-check \
  --artifact-s3-location s3://my-canary-bucket/results \
  --execution-role-arn arn:aws:iam::123456789:role/CloudWatchSyntheticsRole \
  --schedule Expression='rate(5 minutes)' \
  --run-config TimeoutInSeconds=30 \
  --runtime-version syn-nodejs-puppeteer-7.0 \
  --code Handler=index.handler,S3Bucket=my-canary-bucket,S3Key=canary.zip
```

### Smoke Test Ngay Sau Deploy

```
Trong 5-10 phút đầu sau deploy, verify:
  □ Health check endpoint trả về 200
  □ Authentication flow hoạt động
  □ Critical API endpoints trả về đúng response
  □ Error rate không tăng đột biến
  □ Latency P50/P95 không tệ hơn baseline

Script smoke test cơ bản:
  curl -f https://api.example.com/health || exit 1
  curl -f https://api.example.com/api/v1/status | jq '.status == "ok"'
```

### Theo Dõi 24 Giờ Đầu Sau Deploy

```
Giờ 0-1:
  □ Error rate normal
  □ Latency normal
  □ Auto Scaling hoạt động đúng (nếu là high-traffic time)
  □ Không có OOM, crash, timeout bất thường
  □ CloudWatch Alarms không fire

Giờ 1-24:
  □ Memory leak không xảy ra (memory usage ổn định theo thời gian)
  □ Database connections không tăng không kiểm soát
  □ Cost không tăng bất thường
  □ Business metrics ổn định hoặc cải thiện

Sau 24 giờ:
  □ Declare deployment thành công
  □ Update runbook với bài học rút ra
  □ Cleanup: Xóa resources cũ (AMI cũ, task definitions cũ)
  □ Document post-deployment review
```

---

## 📋 Tổng Hợp — Quick Reference Checklist

```
PRE-DEPLOYMENT (Trước Khi Triển Khai):
  □ Infrastructure reviewed: Multi-AZ, Min capacity >= 2
  □ Security reviewed: IAM least privilege, no hardcoded secrets
  □ Monitoring setup: Alarms, dashboards, log groups
  □ Load test passed: 1.5x peak traffic
  □ Rollback plan documented và tested
  □ Deployment window communicated to team

DEPLOYMENT:
  □ Deploy to staging first (nếu có)
  □ Monitor error rate in real-time
  □ Keep terminal open với lệnh rollback sẵn
  □ Notify team khi bắt đầu và kết thúc

POST-DEPLOYMENT (Sau Khi Triển Khai):
  □ Smoke tests passed
  □ Monitor for 1 hour (hoặc qua peak traffic period)
  □ No unexpected alarms
  □ Declare success sau 24 giờ ổn định
  □ Cleanup old resources
  □ Retrospective nếu có issues
```

---

## 📖 Tham Khảo Thêm

- [../06-high-availability/4-deployment-strategies.md](../06-high-availability/4-deployment-strategies.md) — Blue/Green, Canary, Rolling
- [../08-security/README.md](../08-security/README.md) — Security best practices
- [../09-monitoring/5-observability-checklist.md](../09-monitoring/5-observability-checklist.md) — SLI/SLO/SLA & Runbooks
- [1-ec2-issues.md](1-ec2-issues.md) — EC2 troubleshooting
- [3-lambda-debugging.md](3-lambda-debugging.md) — Lambda debugging

---

**Cập Nhật Lần Cuối:** 2026-05-15
