# AWS Compute Services — Chỉ Mục Đầy Đủ

> Hướng dẫn toàn diện về AWS Compute Services — từ EC2 (Elastic Compute Cloud) và Auto Scaling (Tự Động Co Giãn) đến Serverless Lambda, ECS/EKS Container và tối ưu chi phí vận hành đám mây

## 📁 Cấu Trúc Thư Mục

```
Devops/aws/compute/
├── README.md                                        [BẮT ĐẦU TỪ ĐÂY] Lộ trình & tổng quan
├── INDEX.md                                         Chỉ mục đầy đủ (file này)
│
├── 01-ec2-fundamentals/
│   ├── README.md                                    Tổng quan EC2 & kiến trúc máy chủ ảo
│   ├── 1-instance-types.md                          (Sẽ tạo) Instance families, naming, chọn đúng loại
│   ├── 2-ami-storage.md                             (Sẽ tạo) AMI, EBS volumes, instance store, S3
│   ├── 3-security-keypairs.md                       (Sẽ tạo) Security Groups, Key Pairs, IAM profiles
│   ├── 4-user-data-metadata.md                      (Sẽ tạo) Bootstrap scripts, IMDS v1 vs v2
│   └── 5-placement-groups.md                        (Sẽ tạo) Cluster, Spread, Partition placement
│
├── 02-auto-scaling/
│   ├── README.md                                    Tổng quan Auto Scaling & tự động co giãn
│   ├── 1-auto-scaling-groups.md                     (Sẽ tạo) ASG cơ bản, min/max/desired capacity
│   ├── 2-launch-templates.md                        (Sẽ tạo) Launch Templates vs Launch Configurations
│   ├── 3-scaling-policies.md                        (Sẽ tạo) Target Tracking, Step, Scheduled policies
│   ├── 4-spot-mixed-instances.md                    (Sẽ tạo) Spot trong ASG, Mixed Instance Policy
│   └── 5-lifecycle-hooks.md                         (Sẽ tạo) Lifecycle Hooks, warm pools, instance refresh
│
├── 03-serverless-lambda/
│   ├── README.md                                    Tổng quan Lambda & kiến trúc serverless
│   ├── 1-lambda-fundamentals.md                     Function anatomy, runtime, handler, role
│   ├── 2-event-sources.md                           API Gateway, SQS, SNS, S3, DynamoDB Streams
│   ├── 3-layers-extensions.md                       Lambda Layers, Extensions, Container images
│   ├── 4-concurrency-throttling.md                  Reserved/Provisioned Concurrency, throttling
│   └── 5-performance-best-practices.md              Cold start, SnapStart, memory tuning, VPC
│
├── 04-containers-ecs/
│   ├── README.md                                    Tổng quan ECS & container orchestration
│   ├── 1-ecs-architecture.md                        (Sẽ tạo) Cluster, Service, Task, Container Agent
│   ├── 2-task-definitions.md                        (Sẽ tạo) Task Definition, container specs, env vars
│   ├── 3-ecs-services.md                            (Sẽ tạo) Service deployment, circuit breaker, rolling
│   ├── 4-fargate-vs-ec2.md                          (Sẽ tạo) Fargate vs EC2 launch type, trade-offs
│   └── 5-ecs-networking.md                          (Sẽ tạo) awsvpc mode, Service Connect, Service Discovery
│
├── 05-containers-eks/
│   ├── README.md                                    Tổng quan EKS & Kubernetes trên AWS
│   ├── 1-eks-architecture.md                        (Sẽ tạo) Control plane, data plane, add-ons
│   ├── 2-node-groups.md                             (Sẽ tạo) Managed/Self-managed nodes, Fargate Profiles
│   ├── 3-eks-networking.md                          (Sẽ tạo) VPC CNI, CoreDNS, kube-proxy, Ingress
│   ├── 4-eks-storage.md                             (Sẽ tạo) EBS CSI, EFS CSI, StatefulSets
│   └── 5-eks-security.md                            (Sẽ tạo) RBAC, IRSA, Pod Security, Secrets encryption
│
├── 06-high-availability/
│   ├── README.md                                    Tổng quan HA & fault tolerance trên AWS Compute
│   ├── 1-multi-az-design.md                         (Sẽ tạo) Multi-AZ EC2, RDS, ECS patterns
│   ├── 2-scaling-strategies.md                      (Sẽ tạo) Reactive, predictive, scheduled scaling
│   ├── 3-health-checks-recovery.md                  (Sẽ tạo) EC2/ELB/ECS health checks, auto recovery
│   ├── 4-deployment-strategies.md                   (Sẽ tạo) Blue/Green, Rolling, Canary, Immutable
│   └── 5-capacity-planning.md                       (Sẽ tạo) Load testing, forecasting, pre-scaling
│
├── 07-cost-optimization/
│   ├── README.md                                    Tổng quan tối ưu chi phí Compute
│   ├── 1-pricing-models.md                          (Sẽ tạo) On-Demand, Reserved, Spot, Savings Plans
│   ├── 2-spot-strategy.md                           (Sẽ tạo) Spot interruption, diversification, checkpointing
│   ├── 3-rightsizing.md                             (Sẽ tạo) Compute Optimizer, CloudWatch utilization
│   ├── 4-savings-plans.md                           (Sẽ tạo) Compute vs EC2 Instance vs SageMaker plans
│   └── 5-cost-monitoring.md                         (Sẽ tạo) Cost Explorer, Budgets, Trusted Advisor
│
├── 08-security/
│   ├── README.md                                    Tổng quan bảo mật Compute
│   ├── 1-iam-instance-profiles.md                   (Sẽ tạo) IAM Roles, Instance Profiles, least privilege
│   ├── 2-systems-manager.md                         (Sẽ tạo) SSM Session Manager, Run Command, Patch Manager
│   ├── 3-secrets-management.md                      (Sẽ tạo) Secrets Manager, Parameter Store, rotation
│   ├── 4-encryption.md                              (Sẽ tạo) EBS encryption, KMS, data-in-transit
│   └── 5-compliance-patching.md                     (Sẽ tạo) Inspector, Security Hub, patch baselines
│
├── 09-monitoring/
│   ├── README.md                                    Tổng quan giám sát Compute
│   ├── 1-cloudwatch-metrics.md                      EC2, Lambda, ECS/EKS default & custom metrics
│   ├── 2-cloudwatch-logs.md                         Log Groups, Log Insights, Metric Filters
│   ├── 3-cloudwatch-alarms.md                       Alarms, composite alarms, SNS notifications
│   ├── 4-xray-tracing.md                            X-Ray, distributed tracing, service map
│   └── 5-observability-checklist.md                 SLI/SLO/SLA, dashboards, runbooks
│
├── 10-troubleshooting/
│   ├── README.md                                    Tổng quan xử lý sự cố Compute
│   ├── 1-ec2-issues.md                              (Sẽ tạo) SSH/RDP fail, status checks, instance recovery
│   ├── 2-auto-scaling-issues.md                     (Sẽ tạo) Launch failures, scaling imbalance, cooldown
│   ├── 3-lambda-debugging.md                        (Sẽ tạo) Timeouts, OOM, permissions, cold start
│   ├── 4-container-issues.md                        (Sẽ tạo) Task failures, OOM, networking, image pull
│   └── 5-production-checklist.md                    (Sẽ tạo) Checklist trước khi go-live
│
└── 11-interview-prep/
    ├── README.md                                    Tổng quan chuẩn bị phỏng vấn AWS Compute
    ├── 1-INTERVIEW_GUIDE.md                         (Sẽ tạo) Top 20 câu hỏi AWS Compute kèm đáp án
    ├── 2-star-stories.md                            (Sẽ tạo) Mẫu câu chuyện incident theo STAR (5 stories)
    ├── 3-system-design-scenarios.md                 (Sẽ tạo) 5 kịch bản thiết kế hệ thống thực tế
    ├── 4-hands-on-exercises.md                      (Sẽ tạo) 5 bài tập thực hành có hướng dẫn step-by-step
    └── 5-90-day-study-plan.md                       (Sẽ tạo) Kế hoạch học 90 ngày có lộ trình chi tiết
```

---

## ✅ Đã Được Tạo

| Chủ Đề                        | File                | Trạng Thái | Chất Lượng |
| ----------------------------- | ------------------- | ---------- | ---------- |
| **Tổng Quan & Lộ Trình**      | README.md           | ✅         | Toàn diện  |
| **Chỉ Mục Đầy Đủ**            | INDEX.md            | ✅         | Đầy đủ     |
| **EC2 — Tổng Quan & Vòng Đời** | 01-ec2-fundamentals/README.md | ✅ | Toàn diện |
| **EC2 — Instance Types**      | 01-ec2-fundamentals/1-instance-types.md | ✅ | Toàn diện |
| **EC2 — AMI & Storage**       | 01-ec2-fundamentals/2-ami-storage.md | ✅ | Toàn diện |
| **EC2 — Security & Key Pairs** | 01-ec2-fundamentals/3-security-keypairs.md | ✅ | Toàn diện |
| **EC2 — User Data & IMDS**    | 01-ec2-fundamentals/4-user-data-metadata.md | ✅ | Toàn diện |
| **EC2 — Placement Groups**    | 01-ec2-fundamentals/5-placement-groups.md | ✅ | Toàn diện |
| **Auto Scaling — Tổng Quan** | 02-auto-scaling/README.md | ✅ | Toàn diện |
| **Auto Scaling — ASG Cơ Bản** | 02-auto-scaling/1-auto-scaling-groups.md | ✅ | Toàn diện |
| **Auto Scaling — Launch Templates** | 02-auto-scaling/2-launch-templates.md | ✅ | Toàn diện |
| **Auto Scaling — Scaling Policies** | 02-auto-scaling/3-scaling-policies.md | ✅ | Toàn diện |
| **Auto Scaling — Spot & Mixed Instances** | 02-auto-scaling/4-spot-mixed-instances.md | ✅ | Toàn diện |
| **Auto Scaling — Lifecycle Hooks & Warm Pools** | 02-auto-scaling/5-lifecycle-hooks.md | ✅ | Toàn diện |
| **Lambda — Tổng Quan & Kiến Trúc Serverless** | 03-serverless-lambda/README.md | ✅ | Toàn diện |
| **Lambda — Fundamentals & Execution Model** | 03-serverless-lambda/1-lambda-fundamentals.md | ✅ | Toàn diện |
| **Lambda — Event Sources & Triggers** | 03-serverless-lambda/2-event-sources.md | ✅ | Toàn diện |
| **Lambda — Layers, Extensions & Container Images** | 03-serverless-lambda/3-layers-extensions.md | ✅ | Toàn diện |
| **Lambda — Concurrency & Throttling** | 03-serverless-lambda/4-concurrency-throttling.md | ✅ | Toàn diện |
| **Lambda — Performance & Best Practices** | 03-serverless-lambda/5-performance-best-practices.md | ✅ | Toàn diện |
| **ECS — Tổng Quan & Container Orchestration** | 04-containers-ecs/README.md | ✅ | Toàn diện |
| **ECS — Architecture: Cluster, Service, Task** | 04-containers-ecs/1-ecs-architecture.md | ✅ | Toàn diện |
| **ECS — Task Definitions Deep Dive** | 04-containers-ecs/2-task-definitions.md | ✅ | Toàn diện |
| **ECS — Services, Deployment & Auto Scaling** | 04-containers-ecs/3-ecs-services.md | ✅ | Toàn diện |
| **ECS — Fargate vs EC2 Launch Type** | 04-containers-ecs/4-fargate-vs-ec2.md | ✅ | Toàn diện |
| **ECS — Networking: awsvpc, Service Connect** | 04-containers-ecs/5-ecs-networking.md | ✅ | Toàn diện |
| **EKS — Tổng Quan & Kiến Trúc Kubernetes** | 05-containers-eks/README.md | ✅ | Toàn diện |
| **EKS — Control Plane, Data Plane & Add-ons** | 05-containers-eks/1-eks-architecture.md | ✅ | Toàn diện |
| **EKS — Managed Nodes, Self-managed & Fargate Profiles** | 05-containers-eks/2-node-groups.md | ✅ | Toàn diện |
| **EKS — VPC CNI, CoreDNS, Ingress & Network Policies** | 05-containers-eks/3-eks-networking.md | ✅ | Toàn diện |
| **EKS — EBS CSI, EFS CSI & StatefulSets** | 05-containers-eks/4-eks-storage.md | ✅ | Toàn diện |
| **EKS — RBAC, IRSA, Pod Security & Secrets Encryption** | 05-containers-eks/5-eks-security.md | ✅ | Toàn diện |
| **High Availability — Tổng Quan & Kiến Trúc HA** | 06-high-availability/README.md | ✅ | Toàn diện |
| **High Availability — Multi-AZ Design** | 06-high-availability/1-multi-az-design.md | ✅ | Toàn diện |
| **High Availability — Scaling Strategies** | 06-high-availability/2-scaling-strategies.md | ✅ | Toàn diện |
| **High Availability — Health Checks & Recovery** | 06-high-availability/3-health-checks-recovery.md | ✅ | Toàn diện |
| **High Availability — Deployment Strategies** | 06-high-availability/4-deployment-strategies.md | ✅ | Toàn diện |
| **High Availability — Capacity Planning** | 06-high-availability/5-capacity-planning.md | ✅ | Toàn diện |
| **Cost Optimization — Tổng Quan & FinOps** | 07-cost-optimization/README.md | ✅ | Toàn diện |
| **Cost Optimization — Pricing Models** | 07-cost-optimization/1-pricing-models.md | ✅ | Toàn diện |
| **Cost Optimization — Spot Strategy** | 07-cost-optimization/2-spot-strategy.md | ✅ | Toàn diện |
| **Cost Optimization — Rightsizing** | 07-cost-optimization/3-rightsizing.md | ✅ | Toàn diện |
| **Cost Optimization — Savings Plans** | 07-cost-optimization/4-savings-plans.md | ✅ | Toàn diện |
| **Cost Optimization — Cost Monitoring** | 07-cost-optimization/5-cost-monitoring.md | ✅ | Toàn diện |
| **Security — Tổng Quan & Defense in Depth** | 08-security/README.md | ✅ | Toàn diện |
| **Security — IAM Roles & Instance Profiles** | 08-security/1-iam-instance-profiles.md | ✅ | Toàn diện |
| **Security — AWS Systems Manager** | 08-security/2-systems-manager.md | ✅ | Toàn diện |
| **Security — Secrets Management** | 08-security/3-secrets-management.md | ✅ | Toàn diện |
| **Security — Encryption & KMS** | 08-security/4-encryption.md | ✅ | Toàn diện |
| **Security — Compliance & Patching** | 08-security/5-compliance-patching.md | ✅ | Toàn diện |
| **Monitoring — Tổng Quan & Observability** | 09-monitoring/README.md | ✅ | Toàn diện |
| **Monitoring — CloudWatch Metrics** | 09-monitoring/1-cloudwatch-metrics.md | ✅ | Toàn diện |
| **Monitoring — CloudWatch Logs** | 09-monitoring/2-cloudwatch-logs.md | ✅ | Toàn diện |
| **Monitoring — CloudWatch Alarms** | 09-monitoring/3-cloudwatch-alarms.md | ✅ | Toàn diện |
| **Monitoring — AWS X-Ray Distributed Tracing** | 09-monitoring/4-xray-tracing.md | ✅ | Toàn diện |
| **Monitoring — Observability Checklist & SLI/SLO** | 09-monitoring/5-observability-checklist.md | ✅ | Toàn diện |
| **Troubleshooting — Tổng Quan Xử Lý Sự Cố** | 10-troubleshooting/README.md | ✅ | Toàn diện |
| **Troubleshooting — Sự Cố EC2** | 10-troubleshooting/1-ec2-issues.md | ✅ | Toàn diện |
| **Troubleshooting — Sự Cố Auto Scaling** | 10-troubleshooting/2-auto-scaling-issues.md | ✅ | Toàn diện |
| **Troubleshooting — Gỡ Lỗi Lambda** | 10-troubleshooting/3-lambda-debugging.md | ✅ | Toàn diện |
| **Troubleshooting — Sự Cố Container ECS/EKS** | 10-troubleshooting/4-container-issues.md | ✅ | Toàn diện |
| **Troubleshooting — Checklist Trước Go-Live** | 10-troubleshooting/5-production-checklist.md | ✅ | Toàn diện |
| **Interview Prep — Tổng Quan & Chiến Lược** | 11-interview-prep/README.md | ✅ | Toàn diện |
| **Interview Prep — Top 20 Q&A** | 11-interview-prep/1-INTERVIEW_GUIDE.md | ✅ | Toàn diện |
| **Interview Prep — 5 STAR Stories** | 11-interview-prep/2-star-stories.md | ✅ | Toàn diện |
| **Interview Prep — 5 System Design Scenarios** | 11-interview-prep/3-system-design-scenarios.md | ✅ | Toàn diện |
| **Interview Prep — 5 Hands-on Exercises** | 11-interview-prep/4-hands-on-exercises.md | ✅ | Toàn diện |
| **Interview Prep — Kế Hoạch Học 90 Ngày** | 11-interview-prep/5-90-day-study-plan.md | ✅ | Toàn diện |

---

## 🎯 Cần Tạo Tiếp Theo (Theo Thứ Tự Ưu Tiên)

### Ưu Tiên Cao — Kỹ Năng Cốt Lõi

- [x] `01-ec2-fundamentals/README.md` — Tổng quan EC2, instance lifecycle ✅
- [x] `01-ec2-fundamentals/1-instance-types.md` — Instance families & chọn đúng loại ✅
- [x] `01-ec2-fundamentals/2-ami-storage.md` — AMI, EBS, snapshot management ✅
- [x] `01-ec2-fundamentals/3-security-keypairs.md` — Security Groups, Key Pairs, IAM profiles ✅
- [x] `01-ec2-fundamentals/4-user-data-metadata.md` — Bootstrap scripts, IMDS v2 ✅
- [x] `01-ec2-fundamentals/5-placement-groups.md` — Cluster/Spread/Partition ✅
- [x] `02-auto-scaling/README.md` — Auto Scaling tổng quan ✅
- [x] `02-auto-scaling/1-auto-scaling-groups.md` — ASG cơ bản, capacity settings ✅
- [x] `02-auto-scaling/2-launch-templates.md` — Launch Templates chi tiết ✅
- [x] `02-auto-scaling/3-scaling-policies.md` — Tất cả scaling policies ✅
- [x] `03-serverless-lambda/README.md` — Lambda & Serverless tổng quan ✅
- [x] `03-serverless-lambda/1-lambda-fundamentals.md` — Function anatomy, execution model ✅
- [x] `03-serverless-lambda/2-event-sources.md` — Event triggers phổ biến ✅
- [x] `04-containers-ecs/README.md` — ECS & container orchestration tổng quan ✅
- [x] `04-containers-ecs/1-ecs-architecture.md` — Cluster, Service, Task architecture ✅

### Ưu Tiên Trung Bình — Kỹ Năng Nâng Cao

- [x] `02-auto-scaling/4-spot-mixed-instances.md` — Spot trong ASG ✅
- [x] `02-auto-scaling/5-lifecycle-hooks.md` — Lifecycle hooks & warm pools ✅
- [x] `03-serverless-lambda/3-layers-extensions.md` — Lambda Layers & Extensions ✅
- [x] `03-serverless-lambda/4-concurrency-throttling.md` — Concurrency management ✅
- [x] `03-serverless-lambda/5-performance-best-practices.md` — Cold start optimization ✅
- [x] `04-containers-ecs/2-task-definitions.md` — Task Definition deep dive ✅
- [x] `04-containers-ecs/3-ecs-services.md` — Service deployment strategies ✅
- [x] `04-containers-ecs/4-fargate-vs-ec2.md` — Trade-offs & decision guide ✅
- [x] `04-containers-ecs/5-ecs-networking.md` — awsvpc, Service Connect ✅
- [x] `05-containers-eks/README.md` — EKS tổng quan ✅
- [x] `05-containers-eks/1-eks-architecture.md` — Control plane & data plane ✅
- [x] `06-high-availability/README.md` — HA patterns tổng quan ✅
- [x] `06-high-availability/1-multi-az-design.md` — Multi-AZ architecture ✅
- [x] `06-high-availability/2-scaling-strategies.md` — Reactive, predictive, scheduled scaling ✅
- [x] `06-high-availability/3-health-checks-recovery.md` — EC2/ELB/ECS health checks, auto recovery ✅
- [x] `06-high-availability/4-deployment-strategies.md` — Blue/Green, Canary, Rolling ✅
- [x] `06-high-availability/5-capacity-planning.md` — Load testing, forecasting, pre-scaling ✅

### Ưu Tiên Thấp — Tài Liệu Tham Khảo

- [x] `05-containers-eks/2-node-groups.md` — Node group management ✅
- [x] `05-containers-eks/3-eks-networking.md` — VPC CNI & networking ✅
- [x] `05-containers-eks/4-eks-storage.md` — EBS/EFS CSI drivers ✅
- [x] `05-containers-eks/5-eks-security.md` — RBAC, IRSA, pod security ✅
- [x] `07-cost-optimization/README.md` — Cost optimization tổng quan ✅
- [x] `07-cost-optimization/1-pricing-models.md` — Tất cả pricing models ✅
- [x] `07-cost-optimization/2-spot-strategy.md` — Spot best practices ✅
- [x] `07-cost-optimization/3-rightsizing.md` — Compute Optimizer, utilization ✅
- [x] `07-cost-optimization/4-savings-plans.md` — Compute vs EC2 Instance vs SageMaker plans ✅
- [x] `07-cost-optimization/5-cost-monitoring.md` — Cost Explorer, Budgets, Trusted Advisor ✅
- [x] `08-security/README.md` — Security tổng quan ✅
- [x] `08-security/1-iam-instance-profiles.md` — IAM Roles, Instance Profiles, least privilege ✅
- [x] `08-security/2-systems-manager.md` — SSM Session Manager, Run Command, Patch Manager ✅
- [x] `08-security/3-secrets-management.md` — Secrets Manager, Parameter Store, rotation ✅
- [x] `08-security/4-encryption.md` — EBS encryption, KMS, data-in-transit ✅
- [x] `08-security/5-compliance-patching.md` — Inspector, Security Hub, patch baselines ✅
- [x] `09-monitoring/README.md` — Monitoring tổng quan ✅
- [x] `09-monitoring/1-cloudwatch-metrics.md` — Key metrics ✅
- [x] `09-monitoring/2-cloudwatch-logs.md` — Log Groups, Insights, Filters ✅
- [x] `09-monitoring/3-cloudwatch-alarms.md` — Alarms, Composite, Actions ✅
- [x] `09-monitoring/4-xray-tracing.md` — Distributed tracing ✅
- [x] `09-monitoring/5-observability-checklist.md` — SLI/SLO/SLA, Runbooks ✅
- [x] `10-troubleshooting/README.md` — Troubleshooting tổng quan ✅
- [x] `10-troubleshooting/1-ec2-issues.md` — SSH/RDP fail, status checks, recovery ✅
- [x] `10-troubleshooting/2-auto-scaling-issues.md` — Launch failures, imbalance, cooldown ✅
- [x] `10-troubleshooting/3-lambda-debugging.md` — Timeouts, OOM, permissions, cold start ✅
- [x] `10-troubleshooting/4-container-issues.md` — Task failures, OOM, networking, image pull ✅
- [x] `10-troubleshooting/5-production-checklist.md` — Checklist trước khi go-live ✅
- [x] `11-interview-prep/README.md` — Interview prep tổng quan ✅
- [x] `11-interview-prep/1-INTERVIEW_GUIDE.md` — Top 20 Q&A ✅
- [x] `11-interview-prep/2-star-stories.md` — 5 câu chuyện STAR ✅
- [x] `11-interview-prep/3-system-design-scenarios.md` — 5 kịch bản thiết kế hệ thống ✅
- [x] `11-interview-prep/4-hands-on-exercises.md` — 5 bài tập thực hành step-by-step ✅
- [x] `11-interview-prep/5-90-day-study-plan.md` — Kế hoạch học 90 ngày ✅

---

## 🚀 Cách Sử Dụng Knowledge Base Này

### Cho Mục Đích Tự Học

```
1. Bắt đầu với README.md
2. Chọn lộ trình học (Beginner / Intermediate / Advanced)
3. Đi qua từng section theo thứ tự
4. Thực hành hands-on trên AWS Free Tier sau mỗi chủ đề
5. Vẽ sơ đồ kiến trúc để kiểm tra hiểu biết
```

### Cho Chuẩn Bị Phỏng Vấn

```
1. Đọc 11-interview-prep/1-INTERVIEW_GUIDE.md
2. Nắm vững EC2, Auto Scaling, Lambda, ECS (luôn được hỏi)
3. Hiểu rõ trade-offs giữa EC2 / Lambda / ECS / EKS
4. Chuẩn bị 2-3 incident stories từ kinh nghiệm thực tế
5. Luyện tập giải thích architecture design decisions
```

### Cho Công Việc Thực Tế

```
Dùng làm tài liệu tham khảo:
- Trước khi deploy: Xem 10-troubleshooting/5-production-checklist.md
- Khi có sự cố: Vào 10-troubleshooting/ để chẩn đoán từng bước
- Tối ưu chi phí: Tham khảo 07-cost-optimization/
- Bảo mật: Kiểm tra 08-security/ checklist
- Monitoring setup: Xem 09-monitoring/ để thiết lập
```

### Cho System Design

```
1. Đọc README.md để nắm tổng quan các dịch vụ và khi nào dùng
2. Dùng bảng so sánh để chọn EC2 / Lambda / ECS / EKS
3. Thiết kế HA từ 06-high-availability/
4. Tối ưu chi phí từ 07-cost-optimization/
5. Áp dụng bảo mật từ 08-security/
```

---

## 📊 Ước Tính Thời Gian Học

| Section                       | Thời Gian  | Độ Khó | Ưu Tiên         |
| ----------------------------- | ---------- | ------ | --------------- |
| EC2 Fundamentals              | 6-8 giờ    | ⭐⭐   | Bắt buộc        |
| Auto Scaling                  | 4-6 giờ    | ⭐⭐   | Bắt buộc        |
| Serverless Lambda             | 8-10 giờ   | ⭐⭐⭐ | Bắt buộc        |
| ECS & Fargate                 | 8-10 giờ   | ⭐⭐⭐ | Bắt buộc        |
| EKS Kubernetes                | 10-15 giờ  | ⭐⭐⭐ | Nên học         |
| High Availability             | 4-6 giờ    | ⭐⭐   | Nên học         |
| Cost Optimization             | 4-6 giờ    | ⭐⭐   | Nên học         |
| Security                      | 4-6 giờ    | ⭐⭐   | Nên học         |
| Monitoring & Observability    | 4-6 giờ    | ⭐⭐   | Nên học         |
| Troubleshooting               | 3-4 giờ    | ⭐⭐   | Nên học         |
| Interview Prep                | 4-6 giờ    | ⭐     | Trước phỏng vấn |

**Tổng cộng: 60-85 giờ để có kiến thức AWS Compute toàn diện**

---

## 🎓 Cấp Độ Kỹ Năng Hỗ Trợ

### Người Mới — Beginner (0-1 năm kinh nghiệm)

- [ ] Khởi chạy EC2 instance, cấu hình Security Group, SSH vào server
- [ ] Gán IAM Role cho EC2 instance, hiểu Instance Profile
- [ ] Deploy ứng dụng đơn giản lên EC2 với User Data
- [ ] Tạo Lambda Function đầu tiên với trigger từ API Gateway
- [ ] Hiểu sự khác biệt On-Demand vs Reserved vs Spot

**Thời gian để thành thạo:** 1-2 tháng

### Trung Cấp — Intermediate (1-3 năm kinh nghiệm)

- [ ] Thiết kế Auto Scaling Group với Target Tracking Policy
- [ ] Viết Lambda Function xử lý SQS queue với error handling
- [ ] Tạo ECS Service với Fargate và Application Load Balancer
- [ ] Cấu hình CloudWatch Alarms kết nối Auto Scaling policies
- [ ] Triển khai Blue/Green Deployment với ECS hoặc CodeDeploy
- [ ] Tính toán và chọn Savings Plans phù hợp cho workload

**Thời gian để nâng cấp:** 2-3 tháng thực hành

### Nâng Cao — Advanced (3-5+ năm kinh nghiệm)

- [ ] Kiến trúc EKS cluster với managed node groups + Fargate profiles
- [ ] Tối ưu chi phí với Spot + On-Demand mixed fleet + Savings Plans
- [ ] Thiết kế event-driven serverless architecture phức tạp
- [ ] Triển khai multi-region active-active compute với Route 53
- [ ] Implement FinOps (Quản Lý Tài Chính Đám Mây) — tagging, chargeback, rightsizing

**Thời gian để nâng cấp:** Học liên tục

---

## 🔗 Điều Hướng Nhanh

| Nhu Cầu                    | Vị Trí                                                                                       |
| -------------------------- | -------------------------------------------------------------------------------------------- |
| Tổng quan nhanh            | [README.md](README.md)                                                                       |
| EC2 cơ bản                 | [01-ec2-fundamentals/README.md](01-ec2-fundamentals/README.md)                               |
| Auto Scaling               | [02-auto-scaling/README.md](02-auto-scaling/README.md)                                       |
| Serverless Lambda          | [03-serverless-lambda/README.md](03-serverless-lambda/README.md)                             |
| ECS & Fargate              | [04-containers-ecs/README.md](04-containers-ecs/README.md)                                   |
| EKS Kubernetes             | [05-containers-eks/README.md](05-containers-eks/README.md)                                   |
| High Availability          | [06-high-availability/README.md](06-high-availability/README.md)                             |
| Tối ưu chi phí             | [07-cost-optimization/README.md](07-cost-optimization/README.md)                             |
| Bảo mật Compute            | [08-security/README.md](08-security/README.md)                                               |
| Giám sát & CloudWatch      | [09-monitoring/README.md](09-monitoring/README.md)                                           |
| Xử lý sự cố               | [10-troubleshooting/README.md](10-troubleshooting/README.md)                                 |
| Câu hỏi phỏng vấn          | [11-interview-prep/1-INTERVIEW_GUIDE.md](11-interview-prep/1-INTERVIEW_GUIDE.md)             |

---

## 📈 Theo Dõi Tiến Độ Học Tập

Sao chép và theo dõi tiến độ của bạn:

```markdown
## AWS Compute — Tiến Độ Học

### Giai Đoạn 1: Nền Tảng (Tuần 1-2)

- [ ] EC2 instance types & families
- [ ] AMI, EBS volumes, snapshots
- [ ] Security Groups & IAM Instance Profiles
- [ ] User Data & Instance Metadata (IMDS v2)
- [ ] Pricing models: On-Demand / Reserved / Spot

### Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3-6)

- [ ] Auto Scaling Groups với Target Tracking Policy
- [ ] Launch Templates & mixed instance policies
- [ ] Lambda Functions với event sources phổ biến
- [ ] ECS Cluster, Task Definition, Service (Fargate)
- [ ] CloudWatch Alarms & Auto Scaling integration

### Giai Đoạn 3: Nâng Cao (Tuần 7-10)

- [ ] EKS Cluster với managed node groups
- [ ] Lambda concurrency: reserved vs provisioned
- [ ] ECS Blue/Green Deployment với CodeDeploy
- [ ] Spot Instance strategy & interruption handling
- [ ] X-Ray distributed tracing cho Lambda/ECS

### Giai Đoạn 4: Chuyên Sâu (Tuần 11+)

- [ ] EKS IRSA, RBAC, pod security policies
- [ ] Savings Plans vs Reserved Instances analysis
- [ ] Multi-region HA compute architecture
- [ ] FinOps: rightsizing, tagging, cost allocation
- [ ] Mock interviews & system design practice
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này, bạn có thể:

### ✅ Năng Lực Nền Tảng

- [ ] Khởi chạy và cấu hình EC2 instance từ đầu (không cần tham khảo)
- [ ] Thiết kế Auto Scaling Group production-ready với đúng scaling policies
- [ ] Giải thích khi nào dùng Lambda / ECS / EKS / EC2 và trade-offs
- [ ] Tính toán chi phí và chọn đúng pricing model cho từng workload

### ✅ Năng Lực Vận Hành

- [ ] Debug EC2 connectivity issues một cách có hệ thống
- [ ] Phân tích CloudWatch metrics để tìm root cause của sự cố
- [ ] Tối ưu Lambda cold start và memory settings
- [ ] Triển khai ECS service update không gây downtime

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời tự tin 20 câu hỏi AWS Compute hàng đầu
- [ ] Kể 2-3 incident stories theo định dạng STAR
- [ ] Thiết kế scalable, highly available compute architecture
- [ ] Thảo luận trade-offs giữa các AWS Compute services

---

## 🚀 Các Bước Tiếp Theo

### Ngay Lập Tức (Tuần Này)

1. Đọc README.md đầy đủ
2. Chọn lộ trình học phù hợp với level hiện tại
3. Tạo AWS Free Tier account (nếu chưa có)
4. Bắt đầu với `01-ec2-fundamentals/`

### Ngắn Hạn (2 Tuần Tới)

1. Hoàn thành EC2 Fundamentals + Auto Scaling
2. Thực hành tạo web server trên EC2 với Auto Scaling Group
3. Viết Lambda Function đầu tiên và deploy
4. Labs hands-on cho mỗi chủ đề

### Trung Hạn (4 Tuần Tới)

1. Hoàn thành Lambda + ECS Fargate deep dive
2. Triển khai microservice đơn giản trên ECS Fargate
3. Chuẩn bị 2-3 incident stories theo STAR
4. Mock interviews với bạn bè / đồng nghiệp

### Dài Hạn (3 Tháng Tới)

1. Nắm vững toàn bộ AWS Compute services
2. Xây dựng lab project: 3-tier app với EC2 + Lambda + ECS
3. Lấy chứng chỉ AWS (SAA-C03 — Solutions Architect Associate)
4. Ứng dụng vào công việc thực tế hoặc phỏng vấn vị trí mới

---

## 💡 Lời Khuyên Thực Tế

1. **Học qua thực hành:** Tạo EC2 và ECS Fargate thực tế trước khi học Terraform
2. **Hiểu pricing trước:** Biết chi phí của mỗi service giúp đưa ra quyết định đúng
3. **Hiểu trade-offs:** Với mỗi service, hỏi "khi nào dùng cái này thay vì cái kia?"
4. **Monitor mọi thứ:** Bật CloudWatch từ ngày đầu — metric là "sự thật"
5. **Security by default:** Luôn dùng IAM Roles thay vì access keys cho EC2
6. **Test auto scaling:** Simulate load để verify scaling hoạt động đúng
7. **Know your costs:** Dùng Cost Explorer hàng tuần để không bị shock cuối tháng
8. **Document kiến trúc:** Architecture diagram là tài sản quý giá cho team

---

## 📞 Đóng Góp

Phát hiện lỗi? Muốn thêm nội dung?

Đây là tài liệu sống. Đóng góp được chào đón:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Thêm sections cho chủ đề chưa được đề cập
- [ ] Ví dụ thực tế từ kinh nghiệm của bạn
- [ ] Giải thích rõ hơn các khái niệm phức tạp
- [ ] Bổ sung hands-on exercises
- [ ] Cập nhật các tính năng AWS mới

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 3.0 (Interview Prep hoàn thành — Knowledge Base đầy đủ)
**Trạng Thái:** ✅ README.md hoàn thành | ✅ INDEX.md hoàn thành | ✅ 01-ec2-fundamentals hoàn thành | ✅ 02-auto-scaling hoàn thành | ✅ 03-serverless-lambda hoàn thành | ✅ 04-containers-ecs hoàn thành | ✅ 05-containers-eks hoàn thành | ✅ 06-high-availability hoàn thành | ✅ 07-cost-optimization hoàn thành | ✅ 08-security hoàn thành | ✅ 09-monitoring hoàn thành | ✅ 10-troubleshooting hoàn thành | ✅ 11-interview-prep hoàn thành
