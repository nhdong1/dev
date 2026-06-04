# 📐 Rightsizing — Chọn Đúng Kích Cỡ Instance

> Hướng dẫn toàn diện về Rightsizing (Chọn Đúng Kích Cỡ) cho AWS Compute — sử dụng AWS Compute Optimizer (Công Cụ Tối Ưu Tính Toán), phân tích CloudWatch Utilization (Sử Dụng Tài Nguyên), và quy trình rightsizing cho EC2, Lambda, ECS/EKS.

---

## 📚 Mục Lục

1. [Rightsizing Là Gì?](#rightsizing-là-gì)
2. [AWS Compute Optimizer](#compute-optimizer)
3. [CloudWatch Utilization Analysis](#cloudwatch-analysis)
4. [Quy Trình Rightsizing](#quy-trình)
5. [EC2 Rightsizing Chi Tiết](#ec2-rightsizing)
6. [Lambda Rightsizing](#lambda-rightsizing)
7. [ECS & EKS Rightsizing](#container-rightsizing)
8. [Tự Động Hóa Rightsizing](#automation)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 📌 Rightsizing Là Gì? {#rightsizing-là-gì}

### Định Nghĩa

Rightsizing là quá trình **khớp loại và kích cỡ instance với yêu cầu workload thực tế**, loại bỏ over-provisioning (cấp phát dư thừa) và under-provisioning (cấp phát thiếu).

```
Over-provisioned (Cấp Phát Dư):        Right-sized (Đúng Kích Cỡ):
m5.4xlarge (16 vCPU, 64 GB)            m5.xlarge (4 vCPU, 16 GB)
CPU avg: 8%                            CPU avg: 40-60%
Memory avg: 15%                        Memory avg: 50-70%
Chi phí: $768/tháng                    Chi phí: $192/tháng
                                        Tiết kiệm: $576/tháng (75%)
```

### Tại Sao Over-Provisioning Xảy Ra?

1. **"Just in case" mentality** — Sợ hết capacity nên mua dư
2. **Lift and shift** — Migrate từ on-prem mà không re-evaluate sizing
3. **Thiếu visibility** — Không có monitoring metrics đủ tốt
4. **Tăng trưởng chậm** — Server sizing cho tương lai nhưng traffic chưa đến
5. **Không ai chịu trách nhiệm chi phí** — Không có FinOps culture

---

## 🤖 AWS Compute Optimizer {#compute-optimizer}

### Tổng Quan

AWS Compute Optimizer sử dụng **Machine Learning (Học Máy)** để phân tích 14 ngày CloudWatch metrics và đưa ra recommendations (gợi ý) về:
- EC2 Instances
- EC2 Auto Scaling Groups
- EBS Volumes
- Lambda Functions
- ECS Services trên Fargate

### Kích Hoạt Compute Optimizer

```bash
# Opt in cho AWS Account
aws compute-optimizer update-enrollment-status \
  --status Active

# Opt in cho tất cả accounts trong AWS Organization
aws compute-optimizer update-enrollment-status \
  --status Active \
  --include-member-accounts

# Kiểm tra trạng thái enrollment
aws compute-optimizer get-enrollment-status
```

### Xem EC2 Recommendations

```bash
# Xem tất cả EC2 recommendations
aws compute-optimizer get-ec2-instance-recommendations \
  --query 'instanceRecommendations[*].{
    Instance:instanceArn,
    Current:currentInstanceType,
    Finding:finding,
    Recommended:recommendationOptions[0].instanceType,
    Saving:recommendationOptions[0].estimatedMonthlySavings.value
  }' \
  --output table

# Kết quả finding có thể là:
# - OVER_PROVISIONED: Nên downsize
# - UNDER_PROVISIONED: Nên upsize
# - OPTIMIZED: Đang ổn
# - NOT_OPTIMIZED: Cần xem xét
```

### Xem ASG Recommendations

```bash
# Xem Auto Scaling Group recommendations
aws compute-optimizer get-auto-scaling-group-recommendations \
  --query 'autoScalingGroupRecommendations[*].{
    ASG:autoScalingGroupName,
    Current:currentConfiguration.instanceType,
    Finding:finding,
    Recommended:recommendationOptions[0].configuration.instanceType
  }' \
  --output table
```

### Hiểu Recommendation Reasons

Compute Optimizer giải thích lý do recommendation với **Finding Reason Codes**:

| Code                          | Ý Nghĩa                              |
| ----------------------------- | ------------------------------------ |
| `CPUOverProvisioned`          | CPU utilization thấp hơn ngưỡng      |
| `MemoryOverProvisioned`       | Memory thấp (cần CloudWatch agent)   |
| `EBSThroughputOverProvisioned`| EBS throughput thấp hơn provisioned  |
| `NetworkBandwidthOverProvisioned` | Network usage thấp              |
| `CPUUnderProvisioned`         | CPU maxing out — cần upsize           |
| `MemoryUnderProvisioned`      | Memory gần đầy — cần upsize          |

### Enhanced Infrastructure Metrics (Chỉ Số Hạ Tầng Nâng Cao)

Compute Optimizer mặc định phân tích 14 ngày. Bật Enhanced để phân tích **93 ngày** (3 tháng):

```bash
# Bật Enhanced Infrastructure Metrics (có phí thêm ~$0.0003360/giờ/instance)
aws compute-optimizer put-recommendation-preferences \
  --resource-type "Ec2Instance" \
  --scope name=AccountId,value=$(aws sts get-caller-identity --query Account --output text) \
  --enhanced-infrastructure-metrics Activated
```

---

## 📊 CloudWatch Utilization Analysis {#cloudwatch-analysis}

### Metrics Quan Trọng Cho Rightsizing

#### CPU Utilization (Sử Dụng CPU)

```bash
# CPU Average trong 30 ngày
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time $(date -d '30 days ago' --utc +%FT%TZ) \
  --end-time $(date --utc +%FT%TZ) \
  --period 86400 \
  --statistics Average Maximum \
  --output table

# Ngưỡng đánh giá:
# Average CPU < 5%  → Over-provisioned (rất nhiều)
# Average CPU < 20% → Có thể downsize
# Average CPU 20-70% → Optimal range
# Max CPU > 80% thường xuyên → Cần upsize hoặc scale out
```

#### Memory Utilization (Sử Dụng Bộ Nhớ)

CloudWatch không thu thập Memory metric mặc định — phải cài CloudWatch Agent:

```bash
# Cài CloudWatch Agent để thu thập memory metrics
# File config: /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
{
  "metrics": {
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}"
    },
    "metrics_collected": {
      "mem": {
        "measurement": ["mem_used_percent"],
        "metrics_collection_interval": 60
      }
    }
  }
}

# Sau khi cài agent, xem memory metric:
aws cloudwatch get-metric-statistics \
  --namespace CWAgent \
  --metric-name mem_used_percent \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time $(date -d '30 days ago' --utc +%FT%TZ) \
  --end-time $(date --utc +%FT%TZ) \
  --period 86400 \
  --statistics Average Maximum
```

#### Network & Disk I/O

```bash
# Network In/Out
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name NetworkIn \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time $(date -d '30 days ago' --utc +%FT%TZ) \
  --end-time $(date --utc +%FT%TZ) \
  --period 86400 \
  --statistics Average Maximum

# EBS Read/Write IOPS
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name EBSReadOps \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time $(date -d '30 days ago' --utc +%FT%TZ) \
  --end-time $(date --utc +%FT%TZ) \
  --period 86400 \
  --statistics Average Maximum
```

### Script Tự Động Phân Tích Nhiều Instances

```bash
#!/bin/bash
# rightsizing-analysis.sh — Phân tích CPU utilization tất cả EC2 instances

echo "Instance ID | Instance Type | Avg CPU% | Max CPU% | Recommendation"
echo "------------|---------------|----------|----------|---------------"

# Lấy tất cả running instances
INSTANCES=$(aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType]' \
  --output text)

while read -r INSTANCE_ID INSTANCE_TYPE; do
  START_DATE=$(date -d '30 days ago' --utc +%FT%TZ)
  END_DATE=$(date --utc +%FT%TZ)
  
  # Lấy CPU stats
  STATS=$(aws cloudwatch get-metric-statistics \
    --namespace AWS/EC2 \
    --metric-name CPUUtilization \
    --dimensions Name=InstanceId,Value="$INSTANCE_ID" \
    --start-time "$START_DATE" \
    --end-time "$END_DATE" \
    --period 2592000 \
    --statistics Average Maximum \
    --query 'Datapoints[0].[Average,Maximum]' \
    --output text 2>/dev/null)
  
  AVG_CPU=$(echo "$STATS" | awk '{printf "%.1f", $1}')
  MAX_CPU=$(echo "$STATS" | awk '{printf "%.1f", $2}')
  
  # Đưa ra recommendation
  if (( $(echo "$AVG_CPU < 5" | bc -l) )); then
    REC="HEAVILY OVER-PROVISIONED — xem xét downsize 2 cấp"
  elif (( $(echo "$AVG_CPU < 15" | bc -l) )); then
    REC="OVER-PROVISIONED — cân nhắc downsize 1 cấp"
  elif (( $(echo "$MAX_CPU > 85" | bc -l) )); then
    REC="UNDER-PROVISIONED — cân nhắc upsize"
  else
    REC="OK — trong optimal range"
  fi
  
  echo "$INSTANCE_ID | $INSTANCE_TYPE | $AVG_CPU% | $MAX_CPU% | $REC"
done <<< "$INSTANCES"
```

---

## 🔄 Quy Trình Rightsizing {#quy-trình}

### Bốn Bước Rightsizing

```
BƯỚC 1: ĐO LƯỜNG (Measure)
├── Thu thập 30-90 ngày CloudWatch metrics
├── Bao gồm CPU, Memory, Network, Disk I/O
├── Capture peak periods (ngày cuối tháng, campaigns, etc.)
└── Export data ra CSV để phân tích

BƯỚC 2: PHÂN TÍCH (Analyze)
├── Chạy AWS Compute Optimizer recommendations
├── Nhóm instances theo utilization pattern:
│   ├── < 10% CPU avg → candidate downsizing ngay
│   ├── 10-40% CPU avg → review thêm
│   └── > 70% CPU avg → cân nhắc upsize hoặc scale-out
└── Validate memory, network cũng không bottleneck

BƯỚC 3: HÀNH ĐỘNG (Act)
├── Ưu tiên instances có tiết kiệm cao nhất
├── Test trong staging environment trước
├── Implement trong maintenance window
└── Document trước/sau để rollback nếu cần

BƯỚC 4: KIỂM TRA (Verify)
├── Monitor 1-2 tuần sau khi resize
├── Xác nhận performance không bị ảnh hưởng
├── So sánh chi phí trước/sau
└── Schedule review tiếp theo sau 3 tháng
```

### Risk Assessment (Đánh Giá Rủi Ro)

| Mức Rủi Ro  | Trường Hợp                                      | Hành Động                          |
| ----------- | ----------------------------------------------- | ---------------------------------- |
| **Thấp**    | Dev/test, CPU avg < 5% trong 90 ngày            | Resize ngay                        |
| **Trung Bình** | Production non-critical, CPU avg < 15%       | Test staging → resize trong maintenance window |
| **Cao**     | Production critical, database servers           | Test kỹ + rollback plan + resize khi traffic thấp |
| **Rất Cao** | Real-time payment, trading systems              | Không resize nếu không có data chắc chắn |

---

## 💻 EC2 Rightsizing Chi Tiết {#ec2-rightsizing}

### Instance Family Migration (Chuyển Đổi Dòng Instance)

```
Cùng workload, instance family mới hơn = rẻ hơn + mạnh hơn:

m5.xlarge (4 vCPU, 16 GB) @ $0.192/giờ
   → m6i.xlarge (4 vCPU, 16 GB) @ $0.192/giờ (tương đương giá nhưng mạnh hơn ~10%)
   → m6a.xlarge (4 vCPU, 16 GB) @ $0.1728/giờ (AMD, rẻ hơn 10%)

Lý do nên migrate sang generations mới:
├── Same price hoặc rẻ hơn nhưng performance cao hơn
├── Mạng tốt hơn (ENA, up to 12.5 Gbps vs 10 Gbps)
└── EBS bandwidth tốt hơn
```

### Downsize Calculator

```python
# Tính toán tiết kiệm khi downsize
def calculate_downsize_savings(
    current_type, recommended_type, 
    hours_per_month=730, count=1
):
    # Giá On-Demand us-east-1 (cập nhật từ AWS pricing)
    prices = {
        'm5.2xlarge': 0.384, 'm5.xlarge': 0.192, 'm5.large': 0.096,
        'm6i.2xlarge': 0.384, 'm6i.xlarge': 0.192, 'm6i.large': 0.096,
        'c5.2xlarge': 0.34, 'c5.xlarge': 0.17, 'c5.large': 0.085,
        'r5.2xlarge': 0.504, 'r5.xlarge': 0.252, 'r5.large': 0.126,
    }
    
    current_monthly = prices[current_type] * hours_per_month * count
    recommended_monthly = prices[recommended_type] * hours_per_month * count
    saving = current_monthly - recommended_monthly
    saving_pct = (saving / current_monthly) * 100
    
    print(f"Current: {current_type} × {count} = ${current_monthly:.2f}/tháng")
    print(f"Recommended: {recommended_type} × {count} = ${recommended_monthly:.2f}/tháng")
    print(f"Saving: ${saving:.2f}/tháng ({saving_pct:.1f}%) = ${saving*12:.2f}/năm")

calculate_downsize_savings('m5.2xlarge', 'm5.xlarge', count=5)
# Current: m5.2xlarge × 5 = $1,401.60/tháng
# Recommended: m5.xlarge × 5 = $700.80/tháng
# Saving: $700.80/tháng (50.0%) = $8,409.60/năm
```

---

## ⚡ Lambda Rightsizing {#lambda-rightsizing}

### Lambda Pricing (Định Giá Lambda)

Lambda tính phí theo:
- **Number of requests (Số Lượng Request):** $0.20 per 1 million requests
- **Duration (Thời Gian Thực Thi):** tính theo GB-second (gigabyte × giây)
  - Price per GB-second: $0.0000166667

```
Chi phí = (Memory GB × Duration giây × Invocations) × $0.0000166667
Ví dụ: 512 MB = 0.5 GB, duration 100ms = 0.1 giây, 1M invocations/tháng
Chi phí = 0.5 × 0.1 × 1,000,000 × $0.0000166667 = $0.833/tháng
```

### Lambda Memory Optimization

```
More Memory = More CPU + Higher cost per invocation
BUT potentially shorter duration → lower total cost

Ví dụ thực tế (image processing function):
Memory 128 MB: Duration 8,000ms → Cost = 0.125 × 8 × invocations × rate
Memory 512 MB: Duration 1,200ms → Cost = 0.5  × 1.2 × invocations × rate
Memory 1024 MB: Duration 800ms  → Cost = 1.0  × 0.8 × invocations × rate

Relative cost:  128 MB = 1.0 (baseline)
                512 MB = 0.75 (tiết kiệm 25%!)
               1024 MB = 1.0 (tương đương baseline)

→ 512 MB là sweet spot cho function này
```

### AWS Lambda Power Tuning Tool

```bash
# Deploy Lambda Power Tuning (open source tool từ Alex Casalboni)
git clone https://github.com/alexcasalboni/aws-lambda-power-tuning
cd aws-lambda-power-tuning
npm install
npm run deploy

# Chạy power tuning cho function của bạn
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:us-east-1:123456789:stateMachine:powerTuningStateMachine \
  --input '{
    "lambdaARN": "arn:aws:lambda:us-east-1:123456789:function:my-function",
    "powerValues": [128, 256, 512, 1024, 2048],
    "num": 50,
    "payload": "{}",
    "parallelInvocation": true,
    "strategy": "cost"
  }'

# Tool sẽ chạy function ở mỗi memory size 50 lần và tìm optimal point
```

### Lambda Timeout Optimization

```python
# Phân tích Lambda duration metrics để tối ưu timeout
import boto3

def analyze_lambda_duration(function_name, days=30):
    cw = boto3.client('cloudwatch')
    
    response = cw.get_metric_statistics(
        Namespace='AWS/Lambda',
        MetricName='Duration',
        Dimensions=[{'Name': 'FunctionName', 'Value': function_name}],
        StartTime=datetime.now() - timedelta(days=days),
        EndTime=datetime.now(),
        Period=86400 * days,  # Một data point cho toàn bộ period
        Statistics=['Average', 'p95', 'p99', 'Maximum']
    )
    
    if response['Datapoints']:
        dp = response['Datapoints'][0]
        avg_ms = dp.get('Average', 0)
        p95_ms = dp.get('p95', 0)  # 95th percentile
        p99_ms = dp.get('p99', 0)  # 99th percentile
        max_ms = dp.get('Maximum', 0)
        
        print(f"Function: {function_name}")
        print(f"Average duration: {avg_ms:.0f}ms")
        print(f"P95 duration: {p95_ms:.0f}ms")
        print(f"P99 duration: {p99_ms:.0f}ms")
        print(f"Max duration: {max_ms:.0f}ms")
        
        # Khuyến nghị timeout
        recommended_timeout = int(p99_ms * 1.5 / 1000) + 1  # P99 × 1.5, tính bằng giây
        print(f"Recommended timeout: {recommended_timeout}s")
        print(f"(P99 × 1.5 buffer = {p99_ms * 1.5:.0f}ms)")
```

---

## 🐳 ECS & EKS Rightsizing {#container-rightsizing}

### ECS Task CPU & Memory Rightsizing

```python
# Xem CloudWatch Container Insights metrics cho ECS
import boto3
from datetime import datetime, timedelta

def get_ecs_utilization(cluster_name, service_name):
    cw = boto3.client('cloudwatch')
    
    metrics = ['CpuUtilized', 'MemoryUtilized', 'CpuReserved', 'MemoryReserved']
    results = {}
    
    for metric in metrics:
        response = cw.get_metric_statistics(
            Namespace='ECS/ContainerInsights',
            MetricName=metric,
            Dimensions=[
                {'Name': 'ClusterName', 'Value': cluster_name},
                {'Name': 'ServiceName', 'Value': service_name}
            ],
            StartTime=datetime.now() - timedelta(days=14),
            EndTime=datetime.now(),
            Period=86400,
            Statistics=['Average', 'Maximum']
        )
        if response['Datapoints']:
            results[metric] = response['Datapoints'][-1]  # Lấy data point mới nhất
    
    # Tính utilization ratio
    if 'CpuUtilized' in results and 'CpuReserved' in results:
        cpu_util = results['CpuUtilized']['Average'] / results['CpuReserved']['Average'] * 100
        print(f"CPU Utilization: {cpu_util:.1f}%")
    
    if 'MemoryUtilized' in results and 'MemoryReserved' in results:
        mem_util = results['MemoryUtilized']['Average'] / results['MemoryReserved']['Average'] * 100
        print(f"Memory Utilization: {mem_util:.1f}%")
```

### Kubernetes Resource Requests & Limits

```yaml
# TRƯỚC khi rightsizing — over-provisioned
containers:
  - name: api-server
    resources:
      requests:
        cpu: "2000m"    # 2 vCPU requested
        memory: "4Gi"   # 4 GB requested
      limits:
        cpu: "4000m"
        memory: "8Gi"
# Actual usage (từ metrics-server):
# CPU: 200m avg (10% of request)
# Memory: 800Mi avg (20% of request)

---
# SAU khi rightsizing — right-sized
containers:
  - name: api-server
    resources:
      requests:
        cpu: "500m"     # 0.5 vCPU — đủ cho workload
        memory: "1Gi"   # 1 GB — đủ với headroom
      limits:
        cpu: "1000m"    # 2× request cho burst
        memory: "2Gi"   # 2× request cho burst
```

### Vertical Pod Autoscaler — VPA (Tự Động Điều Chỉnh Resource Pod Theo Chiều Dọc)

```yaml
# VPA tự động recommend và set resource requests cho pods
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-server-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  updatePolicy:
    updateMode: "Auto"    # Tự động update (cần pod restart)
    # Hoặc "Off" để chỉ xem recommendations, không tự update
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 4
          memory: 8Gi
```

```bash
# Xem VPA recommendations (khi updateMode: "Off")
kubectl describe vpa api-server-vpa

# Output sẽ hiển thị:
# Recommendation:
#   Container Recommendations:
#     Container Name: api-server
#     Lower Bound:    cpu: 200m, memory: 512Mi
#     Target:         cpu: 450m, memory: 900Mi    ← recommended
#     Upper Bound:    cpu: 1000m, memory: 2Gi
```

---

## 🤖 Tự Động Hóa Rightsizing {#automation}

### AWS Systems Manager Automation

```yaml
# SSM Automation Document để tự động resize instance
# Chạy trong maintenance window hàng tuần
schemaVersion: '0.3'
description: 'Automatic EC2 rightsizing based on Compute Optimizer'
assumeRole: '{{ AutomationAssumeRole }}'
parameters:
  InstanceId:
    type: String
  NewInstanceType:
    type: String
mainSteps:
  - name: StopInstance
    action: 'aws:changeInstanceState'
    inputs:
      InstanceIds: ['{{ InstanceId }}']
      DesiredState: stopped
  
  - name: ChangeInstanceType
    action: 'aws:executeAwsApi'
    inputs:
      Service: ec2
      Api: ModifyInstanceAttribute
      InstanceId: '{{ InstanceId }}'
      InstanceType:
        Value: '{{ NewInstanceType }}'
  
  - name: StartInstance
    action: 'aws:changeInstanceState'
    inputs:
      InstanceIds: ['{{ InstanceId }}']
      DesiredState: running
```

### Lambda Tự Động Phân Tích Và Báo Cáo

```python
import boto3
import json

def generate_rightsizing_report(event, context):
    """Lambda chạy weekly để generate rightsizing report và gửi qua SNS"""
    
    optimizer = boto3.client('compute-optimizer')
    sns = boto3.client('sns')
    
    # Lấy EC2 recommendations
    response = optimizer.get_ec2_instance_recommendations(
        filters=[{'name': 'Finding', 'values': ['OVER_PROVISIONED']}]
    )
    
    recommendations = response['instanceRecommendations']
    
    if not recommendations:
        return {'message': 'No over-provisioned instances found'}
    
    # Tính tổng potential savings
    total_monthly_saving = sum(
        rec['recommendationOptions'][0]['estimatedMonthlySavings']['value']
        for rec in recommendations
        if rec.get('recommendationOptions')
    )
    
    # Tạo report
    report_lines = [
        f"AWS Rightsizing Report — {len(recommendations)} Over-provisioned Instances",
        f"Total Potential Monthly Savings: ${total_monthly_saving:.2f}",
        f"Annual Savings Potential: ${total_monthly_saving * 12:.2f}",
        "",
        "Top 10 Recommendations:"
    ]
    
    sorted_recs = sorted(
        recommendations,
        key=lambda x: x.get('recommendationOptions', [{}])[0].get('estimatedMonthlySavings', {}).get('value', 0),
        reverse=True
    )[:10]
    
    for rec in sorted_recs:
        instance_id = rec['instanceArn'].split('/')[-1]
        current_type = rec['currentInstanceType']
        if rec.get('recommendationOptions'):
            recommended = rec['recommendationOptions'][0]
            saving = recommended.get('estimatedMonthlySavings', {}).get('value', 0)
            new_type = recommended.get('instanceType', 'N/A')
            report_lines.append(
                f"  {instance_id}: {current_type} → {new_type} (save ${saving:.2f}/month)"
            )
    
    report = "\n".join(report_lines)
    
    # Gửi report qua SNS
    sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789:cost-optimization-alerts',
        Subject='Weekly AWS Rightsizing Report',
        Message=report
    )
    
    return {'total_recommendations': len(recommendations), 'monthly_savings': total_monthly_saving}
```

---

## 🎤 Câu Hỏi Phỏng Vấn {#câu-hỏi-phỏng-vấn}

**Q: Làm sao tìm và xử lý over-provisioned EC2 instances trong production?**
> (1) Bật AWS Compute Optimizer và chờ 14 ngày data. (2) Xem recommendations có finding OVER_PROVISIONED. (3) Validate với CloudWatch: CPU avg < 20%, memory < 40%. (4) Test downsize trong staging. (5) Thực hiện resize trong maintenance window với rollback plan. Không resize nhiều instances cùng lúc — theo batch nhỏ để dễ rollback.

**Q: Compute Optimizer cần bao lâu để có recommendations?**
> Minimum 14 ngày CloudWatch data (30+ ngày chính xác hơn). Nếu bật Enhanced Infrastructure Metrics, phân tích 93 ngày để capture seasonal patterns (chu kỳ theo mùa). Với workloads mới, cần chờ đủ data — không rightsizing khi chưa có đủ baseline.

**Q: VPA và HPA — Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang — có thể dùng cùng nhau không?**
> Có nhưng cần cẩn thận. VPA thay đổi CPU/memory requests của pods (vertical), HPA thay đổi số lượng pods (horizontal). Conflict xảy ra khi VPA muốn tăng request nhưng HPA đang scale in. Best practice: dùng VPA ở mode "Off" (chỉ recommend) để tự tay apply, hoặc dùng Goldilocks tool để quản lý. Với workloads có traffic spike, HPA + Spot thường hiệu quả hơn VPA.

**Q: Khi nào KHÔNG nên rightsizing?**
> (1) Instance đang ở memory/CPU spikes không regular — đợi thêm data. (2) Workload có latency SLA strict — downsize có thể ảnh hưởng P99. (3) Instance có Reserved Instance gắn — rightsizing sẽ lãng phí RI. (4) Trước peak season (Black Friday, Tết) — không phải lúc giảm size. Luôn validate trong staging trước production.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
