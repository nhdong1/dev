# 📅 Kế Hoạch Học 90 Ngày — AWS Compute Mastery

> Lộ trình học chi tiết từ beginner đến advanced AWS Compute trong 90 ngày, với mục tiêu cụ thể từng tuần, bài tập thực hành, và checkpoint (điểm kiểm tra tiến độ) hàng tuần.

## 📋 Tổng Quan Lộ Trình

```
Tháng 1 (Ngày 1-30): Nền Tảng & Kỹ Năng Cốt Lõi
Tháng 2 (Ngày 31-60): Nâng Cao & Thực Hành Chuyên Sâu
Tháng 3 (Ngày 61-90): Chuyên Sâu & Sẵn Sàng Phỏng Vấn
```

### Điều Kiện Tiên Quyết

- [ ] Có tài khoản AWS (Free Tier đủ cho 80% labs)
- [ ] Đã cài AWS CLI (Command Line Interface — Giao Diện Dòng Lệnh)
- [ ] Biết Linux cơ bản và lệnh bash
- [ ] Hiểu networking cơ bản (TCP/IP, HTTP, DNS)
- [ ] Biết ít nhất 1 ngôn ngữ lập trình (Python, Node.js, hoặc Java)

### Cam Kết Thời Gian

```
Ngày thường: 1-1.5 giờ/ngày (đọc + thực hành)
Cuối tuần:   2-3 giờ/ngày (labs + ôn tập)
Tổng:        ~100-120 giờ trong 90 ngày
```

---

## 🗓️ Tháng 1: Nền Tảng — Foundation (Ngày 1-30)

### Tuần 1 (Ngày 1-7): EC2 Fundamentals — Nền Tảng EC2

**Mục tiêu tuần:** Hiểu EC2 từ đầu đến cuối, khởi chạy instance thành thục

#### Ngày 1-2: EC2 Instance Types và Pricing

**Lý thuyết (1 giờ):**
- Đọc: `01-ec2-fundamentals/1-instance-types.md`
- Nắm: General Purpose, Compute/Memory/Storage/GPU Optimized families
- Nắm: Naming convention (ví dụ: m6g.xlarge — m=general, 6=thế hệ, g=ARM Graviton)

**Thực hành (30 phút):**
```bash
# Liệt kê tất cả instance types trong region
aws ec2 describe-instance-types \
  --filters "Name=instance-type,Values=m5.*" \
  --query 'InstanceTypes[*].{Type:InstanceType,vCPU:VCpuInfo.DefaultVCpus,MemGB:MemoryInfo.SizeInMiB}' \
  --output table

# So sánh giá On-Demand
aws pricing get-products \
  --service-code AmazonEC2 \
  --filters "Type=TERM_MATCH,Field=instanceType,Value=m5.large" "Type=TERM_MATCH,Field=operatingSystem,Value=Linux"
```

**Kiểm tra hiểu biết:**
- [ ] Có thể giải thích khi nào chọn C-series vs M-series vs R-series
- [ ] Biết T-series Burstable CPU Credit (Tín Chỉ CPU Bùng Nổ) hoạt động thế nào

#### Ngày 3-4: AMI, EBS, và Storage

**Lý thuyết (1 giờ):**
- Đọc: `01-ec2-fundamentals/2-ami-storage.md`
- Nắm: AMI (Amazon Machine Image) types — owned, public, marketplace
- Nắm: EBS (Elastic Block Store) volume types — gp3, io2, st1, sc1

**Thực hành (45 phút):**
```bash
# Khởi chạy EC2 instance từ CLI
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.micro \
  --key-name lab-keypair \
  --security-group-ids sg-xxxx \
  --subnet-id subnet-xxxx \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=my-first-ec2}]'

# Tạo EBS volume và attach
aws ec2 create-volume \
  --availability-zone ap-southeast-1a \
  --volume-type gp3 \
  --size 20 \
  --iops 3000

# Tạo snapshot
aws ec2 create-snapshot --volume-id vol-xxxx --description "My first snapshot"
```

#### Ngày 5: Security Groups và Key Pairs

**Lý thuyết (45 phút):**
- Đọc: `01-ec2-fundamentals/3-security-keypairs.md`
- Nắm: Security Groups là stateful firewall — tại sao không cần outbound rule cho response

**Thực hành (30 phút):**
- Tạo Security Group cho web server (port 80, 443 public; port 22 từ IP của bạn)
- SSH vào EC2 instance và kiểm tra metadata: `curl http://169.254.169.254/latest/meta-data/`

#### Ngày 6-7: User Data, IMDS, và Placement Groups

**Lý thuyết (1 giờ):**
- Đọc: `01-ec2-fundamentals/4-user-data-metadata.md` và `5-placement-groups.md`
- Nắm: IMDSv2 — tại sao quan trọng về bảo mật

**Thực hành + Ôn tập:**
- Tạo EC2 với User Data script tự động cài web server
- Tự kiểm tra: giải thích Cluster vs Spread vs Partition Placement Group không cần tài liệu

**🔵 Checkpoint Tuần 1:**
- [ ] Đã khởi chạy ít nhất 3 EC2 instances bằng CLI
- [ ] Hiểu sự khác biệt giữa EBS volume types và khi nào dùng
- [ ] Có thể giải thích Security Group stateful behavior
- [ ] Biết cách lấy IAM credentials từ IMDS trong EC2

---

### Tuần 2 (Ngày 8-14): Auto Scaling — Tự Động Co Giãn

**Mục tiêu tuần:** Thiết kế và deploy Auto Scaling Group production-ready

#### Ngày 8-9: ASG Cơ Bản và Launch Templates

**Lý thuyết (1 giờ):**
- Đọc: `02-auto-scaling/1-auto-scaling-groups.md` và `2-launch-templates.md`
- Nắm: min/max/desired capacity, health check types, cooldown

**Thực hành:** Hoàn thành Lab 1 từ `4-hands-on-exercises.md`

#### Ngày 10-11: Scaling Policies

**Lý thuyết (1 giờ):**
- Đọc: `02-auto-scaling/3-scaling-policies.md`
- Nắm: Target Tracking (dễ nhất, recommend), Step Scaling, Scheduled Scaling
- Nắm: Predictive Scaling — khi nào dùng

**Thực hành:**
```bash
# Tạo Target Tracking Policy (Chính Sách Theo Dõi Mục Tiêu)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name "webserver-asg" \
  --policy-name "target-cpu-50" \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration "{
    \"PredefinedMetricSpecification\": {
      \"PredefinedMetricType\": \"ASGAverageCPUUtilization\"
    },
    \"TargetValue\": 50.0,
    \"DisableScaleIn\": false
  }"

# Simulate load để test scaling
# Dùng stress tool hoặc Apache Benchmark
sudo yum install -y stress
stress --cpu 2 --timeout 300  # Gây load 5 phút
```

#### Ngày 12-13: Spot Instances và Mixed Instance Policy

**Lý thuyết (45 phút):**
- Đọc: `02-auto-scaling/4-spot-mixed-instances.md`
- Nắm: Spot interruption 2-minute notice, chiến lược diversification

**Thực hành:**
```bash
# Tạo ASG với mixed On-Demand + Spot
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "mixed-asg" \
  --mixed-instances-policy "{
    \"InstancesDistribution\": {
      \"OnDemandBaseCapacity\": 2,
      \"OnDemandPercentageAboveBaseCapacity\": 30,
      \"SpotAllocationStrategy\": \"capacity-optimized\"
    },
    \"LaunchTemplate\": {
      \"LaunchTemplateSpecification\": {
        \"LaunchTemplateName\": \"webserver-template\",
        \"Version\": \"\$Latest\"
      },
      \"Overrides\": [
        {\"InstanceType\": \"m5.large\"},
        {\"InstanceType\": \"m5a.large\"},
        {\"InstanceType\": \"m4.large\"}
      ]
    }
  }" \
  --min-size 3 --max-size 10 --desired-capacity 5
```

#### Ngày 14: Lifecycle Hooks và Ôn Tập

- Đọc: `02-auto-scaling/5-lifecycle-hooks.md`
- Ôn tập tuần 2: tự giải thích tất cả khái niệm

**🔵 Checkpoint Tuần 2:**
- [ ] Đã tạo và test Auto Scaling Group với Target Tracking Policy
- [ ] Hiểu sự khác biệt giữa Target Tracking, Step, và Scheduled Scaling
- [ ] Có thể giải thích Spot interruption và cách handle
- [ ] Biết khi nào dùng Lifecycle Hooks

---

### Tuần 3 (Ngày 15-21): Lambda & Serverless

**Mục tiêu tuần:** Viết và deploy Lambda function, hiểu concurrency và cold start

#### Ngày 15-16: Lambda Fundamentals

**Lý thuyết (1 giờ):**
- Đọc: `03-serverless-lambda/1-lambda-fundamentals.md`
- Nắm: execution model, handler, execution role, runtime

**Thực hành:** Hoàn thành Lab 2 từ `4-hands-on-exercises.md`

#### Ngày 17-18: Event Sources và Triggers

**Lý thuyết (1 giờ):**
- Đọc: `03-serverless-lambda/2-event-sources.md`
- Nắm: synchronous (API Gateway, ALB) vs asynchronous (S3, SNS) vs polling (SQS, Kinesis)

**Thực hành:**
```python
# Viết Lambda xử lý S3 event (ảnh resize khi upload)
import boto3
from PIL import Image
import io

s3 = boto3.client('s3')

def handler(event, context):
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        
        # Tải ảnh từ S3
        response = s3.get_object(Bucket=bucket, Key=key)
        image = Image.open(io.BytesIO(response['Body'].read()))
        
        # Resize về 800x600
        resized = image.resize((800, 600))
        
        # Upload thumbnail lên S3
        buffer = io.BytesIO()
        resized.save(buffer, format='JPEG')
        buffer.seek(0)
        
        thumbnail_key = f"thumbnails/{key}"
        s3.put_object(
            Bucket=bucket,
            Key=thumbnail_key,
            Body=buffer,
            ContentType='image/jpeg'
        )
        
        print(f"Created thumbnail: {thumbnail_key}")
```

#### Ngày 19: Concurrency và Throttling

**Lý thuyết (45 phút):**
- Đọc: `03-serverless-lambda/4-concurrency-throttling.md`
- Nắm: Reserved vs Provisioned Concurrency — khi nào dùng cái nào

**Thực hành:**
```bash
# Set Reserved Concurrency
aws lambda put-function-concurrency \
  --function-name "order-processor" \
  --reserved-concurrent-executions 50

# Bật Provisioned Concurrency cho hot function
aws lambda put-provisioned-concurrency-config \
  --function-name "critical-api" \
  --qualifier "production" \
  --provisioned-concurrent-executions 10
```

#### Ngày 20-21: Cold Start Optimization và Best Practices

- Đọc: `03-serverless-lambda/5-performance-best-practices.md`
- Thực hành: Đo cold start thực tế, tối ưu initialization code

**🔵 Checkpoint Tuần 3:**
- [ ] Đã viết và deploy ít nhất 3 Lambda functions với các triggers khác nhau
- [ ] Hiểu cold start và đã thử 2+ cách giảm thiểu
- [ ] Có thể giải thích Reserved vs Provisioned Concurrency
- [ ] Biết cách set up DLQ (Dead Letter Queue) cho Lambda async

---

### Tuần 4 (Ngày 22-30): ECS & Containers Cơ Bản

**Mục tiêu tuần:** Deploy containerized application trên ECS Fargate

#### Ngày 22-24: ECS Architecture và Task Definitions

**Lý thuyết:**
- Đọc: `04-containers-ecs/README.md`, `1-ecs-architecture.md`, `2-task-definitions.md`
- Nắm: Cluster → Service → Task → Container hierarchy

**Thực hành:** Hoàn thành Lab 3 từ `4-hands-on-exercises.md`

#### Ngày 25-26: ECS Services và Deployment

**Lý thuyết:**
- Đọc: `04-containers-ecs/3-ecs-services.md`, `4-fargate-vs-ec2.md`
- Nắm: Rolling update, Blue/Green deployment, deployment circuit breaker

**Thực hành:**
```bash
# Simulate rolling deployment
# 1. Cập nhật task definition với image mới
# 2. Update service
aws ecs update-service \
  --cluster lab-cluster \
  --service sample-api-service \
  --task-definition sample-api:2 \
  --deployment-configuration "minimumHealthyPercent=100,maximumPercent=200"

# Theo dõi deployment
aws ecs describe-services \
  --cluster lab-cluster \
  --services sample-api-service \
  --query 'services[0].deployments'
```

#### Ngày 27-30: ECS Networking và Tổng Kết Tháng 1

- Đọc: `04-containers-ecs/5-ecs-networking.md`
- Ôn tập toàn bộ tháng 1
- Tự kiểm tra: trả lời 10 câu hỏi đầu trong `1-INTERVIEW_GUIDE.md` không nhìn đáp án

**🔵 Checkpoint Tháng 1 (Ngày 30):**
- [ ] Đã deploy web app trên EC2 với Auto Scaling Group
- [ ] Đã viết Lambda function xử lý SQS hoặc S3 events
- [ ] Đã deploy containerized app trên ECS Fargate
- [ ] Có thể giải thích trade-offs EC2 vs Lambda vs ECS
- [ ] Tự trả lời được 10/20 câu hỏi trong INTERVIEW_GUIDE

---

## 🗓️ Tháng 2: Nâng Cao — Advanced (Ngày 31-60)

### Tuần 5 (Ngày 31-37): EKS — Kubernetes trên AWS

**Mục tiêu tuần:** Hiểu EKS architecture, deploy cluster, và cấu hình IRSA

#### Ngày 31-33: EKS Architecture Deep Dive

**Lý thuyết:**
- Đọc: `05-containers-eks/README.md`, `1-eks-architecture.md`
- Nắm: control plane vs data plane, managed vs self-managed nodes, Fargate profiles

**Thực hành:** Bắt đầu Lab 5 từ `4-hands-on-exercises.md` (tạo EKS cluster)

#### Ngày 34-35: EKS Networking và Storage

**Lý thuyết:**
- Đọc: `05-containers-eks/3-eks-networking.md`, `4-eks-storage.md`
- Nắm: VPC CNI (Container Network Interface), EBS CSI Driver, EFS CSI Driver

**Thực hành:**
```bash
# Cài EBS CSI Driver
aws eks create-addon \
  --cluster-name lab-eks \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn <IRSA-role-arn>

# Deploy StatefulSet với EBS volume
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_PASSWORD
          value: mypassword
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp2
      resources:
        requests:
          storage: 10Gi
EOF
```

#### Ngày 36-37: IRSA và EKS Security

**Lý thuyết:**
- Đọc: `05-containers-eks/5-eks-security.md`, `2-node-groups.md`
- Nắm: IRSA (IAM Roles for Service Accounts), RBAC (Role-Based Access Control)

**Thực hành:**
```bash
# Tạo IRSA role cho S3 access
eksctl create iamserviceaccount \
  --cluster lab-eks \
  --name s3-reader \
  --namespace default \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve

# Verify IRSA hoạt động
kubectl run test-irsa \
  --image=amazon/aws-cli \
  --serviceaccount=s3-reader \
  --restart=Never \
  -- aws s3 ls
```

#### Hoàn thành Lab 5 và HPA

**🔵 Checkpoint Tuần 5:**
- [ ] Đã tạo EKS cluster với managed node group
- [ ] Đã deploy application và cấu hình HPA
- [ ] Đã cài Cluster Autoscaler và test auto-scaling
- [ ] Hiểu IRSA và tại sao tốt hơn node IAM role

---

### Tuần 6 (Ngày 38-44): High Availability — Tính Sẵn Sàng Cao

**Mục tiêu tuần:** Thiết kế và implement Multi-AZ architecture

#### Ngày 38-39: Multi-AZ Design Patterns

**Lý thuyết:**
- Đọc: `06-high-availability/README.md`, `1-multi-az-design.md`
- Vẽ sơ đồ kiến trúc 3-tier HA từ đầu không nhìn tài liệu

**Thực hành:**
```bash
# Kiểm tra ASG distribution qua các AZ
aws ec2 describe-instances \
  --filters "Name=tag:aws:autoscaling:groupName,Values=webserver-asg" \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,AZ:Placement.AvailabilityZone}' \
  --output table
```

#### Ngày 40-41: Health Checks và Deployment Strategies

**Lý thuyết:**
- Đọc: `06-high-availability/3-health-checks-recovery.md`, `4-deployment-strategies.md`
- Nắm: Blue/Green vs Canary vs Rolling — khi nào dùng loại nào

**Thực hành:** Thực hiện Blue/Green deployment với ECS và CodeDeploy

#### Ngày 42-44: Scaling Strategies và Capacity Planning

- Đọc: `06-high-availability/2-scaling-strategies.md`, `5-capacity-planning.md`
- Thực hành load testing với AWS Distributed Load Testing solution

**🔵 Checkpoint Tuần 6:**
- [ ] Có thể vẽ HA architecture từ đầu mà không cần tài liệu
- [ ] Đã thực hiện Blue/Green hoặc Canary deployment
- [ ] Hiểu trade-offs giữa các deployment strategies

---

### Tuần 7 (Ngày 45-51): Cost Optimization — Tối Ưu Chi Phí

**Mục tiêu tuần:** Tính toán và tối ưu chi phí AWS Compute thực tế

#### Ngày 45-47: Pricing Models và Savings Plans

**Lý thuyết:**
- Đọc: `07-cost-optimization/README.md`, `1-pricing-models.md`, `4-savings-plans.md`
- Tính toán: So sánh chi phí On-Demand vs Reserved vs Savings Plans cho một workload cụ thể

**Thực hành:**
```
Bài toán thực tế:
- Bạn có 10 m5.xlarge instances chạy 24/7
- 6 instance luôn cần, 4 instance theo workload
- Tính toán chi phí optimal cho 1 năm
```

#### Ngày 48-49: Spot Strategy và Rightsizing

- Đọc: `07-cost-optimization/2-spot-strategy.md`, `3-rightsizing.md`
- Bật AWS Compute Optimizer và xem recommendations
- Thực hành: Identify over-provisioned instances trong account của bạn

#### Ngày 50-51: Cost Monitoring

- Đọc: `07-cost-optimization/5-cost-monitoring.md`
- Thiết lập Budget Alert (Cảnh Báo Ngân Sách): cảnh báo khi chi phí > 80% budget

**🔵 Checkpoint Tuần 7:**
- [ ] Có thể tính toán chi phí cho một workload cụ thể
- [ ] Hiểu khi nào dùng Savings Plans vs Reserved Instances
- [ ] Đã thiết lập Cost Budget với email alerts

---

### Tuần 8 (Ngày 52-60): Security và Monitoring

**Mục tiêu tuần:** Thiết lập security và monitoring hoàn chỉnh

#### Ngày 52-54: IAM, SSM, và Secrets Management

**Lý thuyết:**
- Đọc: `08-security/1-iam-instance-profiles.md`, `2-systems-manager.md`, `3-secrets-management.md`
- Nắm: Least Privilege principle, SSM Session Manager (không cần SSH)

**Thực hành:**
```bash
# Dùng SSM Session Manager thay vì SSH
aws ssm start-session --target i-xxxx

# Lưu secret vào Secrets Manager
aws secretsmanager create-secret \
  --name "myapp/database/password" \
  --secret-string '{"username":"admin","password":"SuperSecret123"}'

# Đọc secret trong Lambda/EC2
aws secretsmanager get-secret-value \
  --secret-id "myapp/database/password" \
  --query 'SecretString' --output text
```

#### Ngày 55-57: Monitoring và Observability

**Lý thuyết:**
- Đọc: `09-monitoring/README.md` và tất cả files
- Nắm: CloudWatch metrics, logs, alarms, X-Ray tracing

**Thực hành:** Hoàn thành Lab 4 từ `4-hands-on-exercises.md`

#### Ngày 58-60: Troubleshooting Practice và Tổng Kết Tháng 2

- Đọc: `10-troubleshooting/` tất cả files
- Thực hành: Cố tình break một component và tự debug
- Ôn tập tháng 2 và kiểm tra lại checkpoint

**🔵 Checkpoint Tháng 2 (Ngày 60):**
- [ ] Đã cấu hình security theo least privilege principles
- [ ] Đã thiết lập CloudWatch monitoring với alarms
- [ ] Có thể debug EC2, Lambda, ECS issues một cách hệ thống
- [ ] Tự trả lời được 15/20 câu hỏi trong INTERVIEW_GUIDE

---

## 🗓️ Tháng 3: Sẵn Sàng Phỏng Vấn — Interview Ready (Ngày 61-90)

### Tuần 9 (Ngày 61-67): Hệ Thống Hóa Kiến Thức

**Mục tiêu tuần:** Nắm vững tất cả trade-offs và có thể giải thích không cần tài liệu

#### Ngày 61-63: Trade-offs Deep Dive

**Bài tập:** Với mỗi cặp dịch vụ, viết ra trade-offs trong 5 phút:

```
EC2 vs Lambda:
+ EC2: ...
+ Lambda: ...
→ Chọn EC2 khi: ...
→ Chọn Lambda khi: ...

ECS Fargate vs EKS:
...

Reserved vs Savings Plans:
...

Spot vs On-Demand:
...
```

#### Ngày 64-65: System Design Practice (Luyện Thiết Kế Hệ Thống)

- Đọc và luyện tất cả 5 kịch bản trong `3-system-design-scenarios.md`
- Vẽ sơ đồ kiến trúc trên giấy/whiteboard, không nhìn tài liệu
- Tự giải thích từng decision trong 30 giây

#### Ngày 66-67: STAR Stories Practice

- Đọc `2-star-stories.md`
- Chọn 3 stories phù hợp nhất với kinh nghiệm của bạn
- Điều chỉnh theo kinh nghiệm thực tế
- Tập kể to — đảm bảo mỗi story < 3 phút

**🔵 Checkpoint Tuần 9:**
- [ ] Có thể vẽ kiến trúc cho cả 5 kịch bản không nhìn tài liệu
- [ ] Có 3 STAR stories sẵn sàng, kể được < 3 phút mỗi story
- [ ] Có thể giải thích trade-offs của mọi AWS Compute service

---

### Tuần 10 (Ngày 68-74): Mock Interviews (Phỏng Vấn Thử)

**Mục tiêu tuần:** Luyện phỏng vấn thực tế

#### Cách Tổ Chức Mock Interview

**Tìm partner:**
- Bạn bè / đồng nghiệp cùng học
- Online: Reddit r/cscareerquestions, Discord server AWS, Pramp.com

**Format mock interview (45-60 phút):**
```
5 phút:  Giới thiệu, warm-up
10 phút: 3-4 câu hỏi kiến thức AWS Compute
10 phút: 1 STAR story question ("Kể về lần bạn...")
20 phút: System design question
5 phút:  Câu hỏi của candidate cho interviewer
5 phút:  Feedback
```

#### Ngày 68-70: Mock Interview Round 1

Dùng các câu hỏi từ `1-INTERVIEW_GUIDE.md`:
- Câu 1, 3, 7, 11, 19 (chọn 5 câu ngẫu nhiên)
- System design: Kịch bản 2 (Ride-sharing)

**Feedback checklist:**
- [ ] Trả lời rõ ràng, có structure
- [ ] Đề cập đến trade-offs
- [ ] Giải thích được technical depth
- [ ] STAR story: đủ 4 yếu tố S-T-A-R
- [ ] System design: làm rõ yêu cầu trước khi vẽ

#### Ngày 71-72: Cải Thiện Điểm Yếu

- Xem lại feedback từ mock interview round 1
- Đọc lại sections liên quan đến câu trả lời chưa tốt
- Luyện lại những câu hỏi trả lời chưa tốt

#### Ngày 73-74: Mock Interview Round 2

Dùng câu hỏi khác và system design khác:
- Câu 8, 12, 15, 17, 20
- System design: Kịch bản 4 (Data Pipeline)

**🔵 Checkpoint Tuần 10:**
- [ ] Đã làm ít nhất 2 mock interviews đầy đủ
- [ ] Đã nhận và incorporate feedback
- [ ] Tự đánh giá: sẵn sàng 80%+ cho phỏng vấn thực

---

### Tuần 11 (Ngày 75-81): Thực Hành Chuyên Sâu

**Mục tiêu tuần:** Build một project thực tế tích hợp nhiều dịch vụ

#### Project: 3-Tier Serverless E-commerce API

**Kiến trúc:**
```
API Gateway → Lambda → DynamoDB
                    → SQS → Lambda (order processing)
                           → SNS → Lambda (notifications)
S3 (product images) → Lambda (image resize) → CloudFront
```

**Tuần 11 tasks:**
- Ngày 75-76: Setup infra (API Gateway, Lambda, DynamoDB)
- Ngày 77-78: Implement order flow (SQS, payment Lambda)
- Ngày 79-80: Add monitoring (CloudWatch alarms, X-Ray tracing)
- Ngày 81: Deploy và test end-to-end

**🔵 Checkpoint Tuần 11:**
- [ ] Project hoạt động end-to-end
- [ ] Có monitoring và alerting
- [ ] Có thể giải thích kiến trúc cho người mới

---

### Tuần 12 (Ngày 82-90): Final Review và Interview Polish

**Mục tiêu tuần:** Finalize preparation, đảm bảo sẵn sàng 100%

#### Ngày 82-84: Final Q&A Review

```bash
# Ngày 82: Review câu 1-7 (EC2, Auto Scaling)
# Ngày 83: Review câu 8-15 (Lambda, ECS, EKS)
# Ngày 84: Review câu 16-20 (HA, Cost, Trade-offs)

# Với mỗi câu:
# 1. Đọc câu hỏi, đóng đáp án
# 2. Tự trả lời trong 2-3 phút
# 3. So sánh với đáp án mẫu
# 4. Note những điểm còn thiếu
```

#### Ngày 85-86: System Design Final Practice

- Luyện lại kịch bản 1 (Video Streaming) và kịch bản 5 (Serverless API)
- Focus: giải thích trade-offs tự nhiên, không vội vàng

#### Ngày 87-88: STAR Stories Final Polish

- Kể 3 stories của bạn cho người khác nghe
- Đảm bảo: mỗi story < 3 phút, kết quả có số liệu cụ thể
- Chuẩn bị thêm 2 câu hỏi sẽ hỏi ngược interviewer

#### Ngày 89-90: Final Preparation

```
Ngày 89:
- Ôn nhanh top 10 câu hỏi quan trọng nhất
- Review 3 STAR stories lần cuối
- Nghỉ ngơi, đừng học thêm quá nhiều

Ngày 90 (hoặc ngày phỏng vấn):
- Đọc lại JD (Job Description) — highlight AWS terms
- Breakfast tốt, ngủ đủ giấc đêm trước
- Sẵn sàng notebook/giấy để ghi chú
```

**🔵 Checkpoint Cuối (Ngày 90):**
- [ ] Trả lời được 18+/20 câu hỏi trong INTERVIEW_GUIDE tự tin
- [ ] Có thể vẽ và giải thích 5 system design scenarios
- [ ] Có 3 STAR stories hoàn chỉnh, kể tự nhiên < 3 phút
- [ ] Đã làm ít nhất 3 mock interviews với feedback
- [ ] Đã build ít nhất 1 project tích hợp nhiều AWS services

---

## 📊 Tracker Tiến Độ Hàng Tuần

Copy và điền vào cuối mỗi tuần:

```markdown
### Tuần __: [Ngày bắt đầu] — [Ngày kết thúc]

**Đã học:**
- [ ] ...
- [ ] ...

**Đã thực hành:**
- [ ] Lab/exercise: ...
- [ ] Project: ...

**Điểm tự đánh giá (1-10):**
- Lý thuyết: __/10
- Thực hành: __/10
- Khả năng giải thích: __/10

**Điểm yếu cần cải thiện tuần sau:**
- ...

**Mock interview feedback (nếu có):**
- ...
```

---

## 🎯 Mục Tiêu Sau 90 Ngày

Khi hoàn thành lộ trình này, bạn có thể:

**Kiến Thức Kỹ Thuật:**
- Giải thích mọi AWS Compute service và khi nào dùng
- Thiết kế scalable, highly available architecture từ đầu
- Tính toán và tối ưu chi phí cho production workload
- Debug và troubleshoot issues một cách hệ thống

**Kỹ Năng Phỏng Vấn:**
- Trả lời tự tin top 20 câu hỏi AWS Compute
- Kể 3 STAR stories ấn tượng từ kinh nghiệm thực tế
- Thiết kế hệ thống trong buổi phỏng vấn với trade-offs rõ ràng
- Hỏi ngược interviewer những câu hỏi thông minh

**Kỹ Năng Thực Tế:**
- Deploy production-grade applications trên EC2, Lambda, ECS, EKS
- Thiết lập monitoring và alerting hoàn chỉnh
- Implement security best practices
- Tối ưu chi phí với Spot, Savings Plans, rightsizing

---

## 💡 Lời Khuyên Để Không Bỏ Cuộc

1. **Học 1 giờ/ngày đều đặn tốt hơn 7 giờ cuối tuần:** Não bộ cần thời gian để consolidate (củng cố) kiến thức

2. **Thực hành ngay sau khi đọc:** Đừng để quá 24 giờ giữa đọc lý thuyết và thực hành

3. **Dọn dẹp AWS resources sau mỗi lab:** Tránh chi phí bất ngờ, tạo thói quen tốt

4. **Giải thích cho người khác:** Rubber duck debugging — giải thích cho "con vịt cao su" giúp phát hiện chỗ chưa hiểu

5. **Join AWS community:** r/aws, AWS User Group Vietnam — học từ kinh nghiệm thực tế của người khác

6. **Đừng so sánh với người khác:** Mỗi người có pace học khác nhau — 90 ngày là gợi ý, không phải cứng nhắc

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành — Kế hoạch học 90 ngày đầy đủ
