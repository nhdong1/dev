# 🎯 Spot Instance Strategy — Chiến Lược Sử Dụng Spot

> Hướng dẫn chiến lược sử dụng Spot Instances hiệu quả — từ cơ chế Interruption (Gián Đoạn), Instance Diversification (Đa Dạng Hóa), Checkpointing (Lưu Tiến Trình), đến triển khai Spot trong Auto Scaling Groups, ECS, và EKS.

---

## 📚 Mục Lục

1. [Cơ Chế Spot Instance](#cơ-chế)
2. [Interruption Handling — Xử Lý Gián Đoạn](#interruption-handling)
3. [Instance Diversification — Đa Dạng Hóa](#diversification)
4. [Checkpointing — Lưu Tiến Trình](#checkpointing)
5. [Spot trong Auto Scaling Groups](#spot-asg)
6. [Spot trong ECS & EKS](#spot-containers)
7. [Spot Advisor & Chọn Instance Type](#spot-advisor)
8. [Anti-Patterns — Cạm Bẫy Cần Tránh](#anti-patterns)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## ⚙️ Cơ Chế Spot Instance {#cơ-chế}

### Spot Pools (Nhóm Spot)

Mỗi Spot Pool là một tập hợp EC2 capacity chưa sử dụng được xác định bởi:
- **Instance type** (loại instance): ví dụ m5.large
- **Operating system** (hệ điều hành): Linux, Windows
- **Availability Zone — AZ** (Vùng Khả Dụng): us-east-1a, us-east-1b, ...

```
Spot Pool = {instance_type} × {OS} × {AZ}

Ví dụ:
├── m5.large / Linux / us-east-1a  ← một pool
├── m5.large / Linux / us-east-1b  ← pool khác
├── m5.xlarge / Linux / us-east-1a ← pool khác
└── ...
```

### Spot Price (Giá Spot)

Spot Price thay đổi theo supply/demand của từng pool:
- Thường **ổn định trong nhiều giờ hoặc ngày**
- AWS tính phí theo giá Spot tại thời điểm sử dụng
- Khi AWS cần capacity → **Spot Interruption (Gián Đoạn Spot)** xảy ra

### Spot Instance Lifecycle (Vòng Đời)

```
Request Spot
    ↓
[OPEN] — Request đang chờ fulfillment (hoàn thành)
    ↓ (capacity có sẵn)
[ACTIVE] — Instance đang chạy
    ↓ (một trong các lý do dưới)
    ├── AWS cần capacity → [INTERRUPTED] → Terminate/Stop/Hibernate
    ├── Bạn terminate → [CLOSED]
    └── Request expired → [CANCELLED]
```

### Interruption Notice (Thông Báo Gián Đoạn)

Khi AWS quyết định lấy lại Spot Instance:
1. **Instance Metadata Service** trả về interruption notice **2 phút** trước
2. **EventBridge** (trước đây là CloudWatch Events) phát sự kiện `EC2 Spot Instance Interruption Warning`
3. **Instance Action Metadata** được set tại: `http://169.254.169.254/latest/meta-data/spot/instance-action`

---

## 🛡️ Interruption Handling — Xử Lý Gián Đoạn {#interruption-handling}

### Pattern 1: Graceful Shutdown (Dừng Ưu Tiên)

```bash
#!/bin/bash
# Script chạy dưới dạng cron job mỗi 5 giây để kiểm tra interruption notice

check_spot_interruption() {
  # Kiểm tra Instance Metadata Service v2 (IMDSv2)
  TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
  
  ACTION=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    "http://169.254.169.254/latest/meta-data/spot/instance-action" 2>/dev/null)
  
  if [[ "$ACTION" == *"terminate"* ]] || [[ "$ACTION" == *"stop"* ]]; then
    echo "SPOT INTERRUPTION DETECTED! Starting graceful shutdown..."
    
    # 1. Dừng nhận request mới (drain từ Load Balancer)
    deregister_from_load_balancer
    
    # 2. Hoàn thành các request đang xử lý (drain connections)
    drain_connections 90  # chờ tối đa 90 giây
    
    # 3. Lưu state/checkpoint nếu cần
    save_checkpoint
    
    # 4. Gửi thông báo
    notify_team "Spot instance $INSTANCE_ID interrupted"
    
    echo "Graceful shutdown complete."
  fi
}

# Chạy mỗi 5 giây
while true; do
  check_spot_interruption
  sleep 5
done
```

### Pattern 2: EventBridge Rule (Quy Tắc EventBridge)

```json
// EventBridge Rule tự động gọi Lambda khi Spot bị interrupted
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Spot Instance Interruption Warning"],
  "detail": {
    "instance-action": ["terminate"]
  }
}
```

```python
# Lambda handler xử lý Spot interruption
import boto3
import json

def handler(event, context):
    instance_id = event['detail']['instance-id']
    action = event['detail']['instance-action']
    
    ec2 = boto3.client('ec2')
    ssm = boto3.client('ssm')
    
    # Deregister từ Load Balancer target group
    elb = boto3.client('elbv2')
    # ... deregister logic
    
    # Gửi SSM Run Command để save state
    ssm.send_command(
        InstanceIds=[instance_id],
        DocumentName='AWS-RunShellScript',
        Parameters={'commands': ['python3 /app/save_checkpoint.py']}
    )
    
    print(f"Handled interruption for {instance_id}: {action}")
```

### Pattern 3: SQS Queue cho Worker Pattern

```
Worker Architecture (Kiến Trúc Worker) với Spot:

[SQS Queue] ←── producer adds jobs
     ↓
[Spot Instance Worker]
     ├── Poll SQS message
     ├── Process job (với visibility timeout đủ dài)
     ├── Nếu interrupted → job trở về SQS sau visibility timeout
     └── Delete message only khi hoàn thành

Ưu điểm: Job không bị mất khi instance bị interrupted.
Visibility Timeout (Thời Gian Hiển Thị): đặt >= job duration + buffer
```

---

## 🎲 Instance Diversification — Đa Dạng Hóa {#diversification}

### Tại Sao Cần Diversification?

```
Vấn đề nếu chỉ dùng 1 instance type/AZ:
├── Nếu pool đó có giá cao hoặc bị interrupted nhiều → không có capacity
├── Không có fallback khi 1 pool khan hàng
└── Toàn bộ fleet có thể bị interrupted cùng lúc

Giải pháp: Diversify across multiple pools
├── Nhiều instance types (same family hoặc across families)
├── Nhiều Availability Zones
└── Mix Spot + On-Demand
```

### Chiến Lược Chọn Instance Types

```
Nguyên tắc "Instance Flexibility" (Linh Hoạt Instance):

1. SAME vCPU/Memory ratio — chọn 4-6 instance types tương đương:
   Thay vì chỉ m5.large, hãy dùng:
   ├── m5.large   (2 vCPU, 8 GB)
   ├── m5a.large  (2 vCPU, 8 GB) — AMD variant, thường rẻ hơn
   ├── m6i.large  (2 vCPU, 8 GB) — Thế hệ mới hơn
   ├── m6a.large  (2 vCPU, 8 GB) — AMD + Gen 6
   └── m4.large   (2 vCPU, 8 GB) — Thế hệ cũ, pool thường ít cạnh tranh

2. Ưu tiên instance types có Interruption Rate < 5%
   → Kiểm tra tại https://aws.amazon.com/ec2/spot/instance-advisor/

3. Tránh dùng instance types quá mới (high demand) hoặc quá niche
```

### Allocation Strategy (Chiến Lược Phân Bổ) Trong Spot Fleet

| Strategy                 | Mô Tả                                                    | Dùng Khi                          |
| ------------------------ | -------------------------------------------------------- | --------------------------------- |
| `price-capacity-optimized` | Chọn pool có capacity nhiều nhất + giá tốt (khuyến nghị) | Mặc định cho mọi workload         |
| `capacity-optimized`     | Chọn pool có capacity nhiều nhất                         | Khi availability quan trọng hơn giá |
| `lowest-price`           | Chọn pool rẻ nhất                                        | Không khuyến nghị — hay bị interrupted |
| `diversified`            | Phân phối đều qua tất cả pools                           | Spot Fleet, khi cần predictability |

### Ví Dụ: Mixed Instance Policy trong ASG

```json
{
  "MixedInstancesPolicy": {
    "LaunchTemplate": {
      "LaunchTemplateSpecification": {
        "LaunchTemplateId": "lt-0123456789",
        "Version": "$Latest"
      },
      "Overrides": [
        {"InstanceType": "m5.large"},
        {"InstanceType": "m5a.large"},
        {"InstanceType": "m6i.large"},
        {"InstanceType": "m6a.large"},
        {"InstanceType": "m4.large"}
      ]
    },
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 2,
      "OnDemandPercentageAboveBaseCapacity": 20,
      "SpotAllocationStrategy": "price-capacity-optimized",
      "SpotInstancePools": 0
    }
  }
}
```

---

## 💾 Checkpointing — Lưu Tiến Trình {#checkpointing}

### Khi Nào Cần Checkpointing

```
Workload cần checkpointing:
├── ML Training jobs (chạy hàng giờ/ngày)
├── Video transcoding (file lớn)
├── Data processing (ETL nhiều GB/TB)
├── Genome sequencing
└── Financial simulations (Monte Carlo)

Workload KHÔNG cần checkpointing:
├── Web request handling (< 1 giây/request)
├── Short Lambda functions
└── Stateless API calls
```

### Checkpointing Pattern với S3

```python
import boto3
import json
import os
import time

s3 = boto3.client('s3')
CHECKPOINT_BUCKET = os.environ['CHECKPOINT_BUCKET']
JOB_ID = os.environ['JOB_ID']

def save_checkpoint(current_offset, processed_count, state):
    """Lưu checkpoint lên S3 mỗi N records hoặc mỗi T giây"""
    checkpoint = {
        'offset': current_offset,
        'processed': processed_count,
        'state': state,
        'timestamp': time.time()
    }
    
    s3.put_object(
        Bucket=CHECKPOINT_BUCKET,
        Key=f'checkpoints/{JOB_ID}/latest.json',
        Body=json.dumps(checkpoint)
    )
    print(f"Checkpoint saved: offset={current_offset}, processed={processed_count}")

def load_checkpoint():
    """Load checkpoint để resume sau khi bị interrupted"""
    try:
        response = s3.get_object(
            Bucket=CHECKPOINT_BUCKET,
            Key=f'checkpoints/{JOB_ID}/latest.json'
        )
        checkpoint = json.loads(response['Body'].read())
        print(f"Resumed from checkpoint: offset={checkpoint['offset']}")
        return checkpoint
    except s3.exceptions.NoSuchKey:
        print("No checkpoint found, starting from beginning")
        return {'offset': 0, 'processed': 0, 'state': None}

def process_job():
    checkpoint = load_checkpoint()
    current_offset = checkpoint['offset']
    processed_count = checkpoint['processed']
    
    CHECKPOINT_INTERVAL = 1000  # Lưu mỗi 1000 records
    
    for i, record in enumerate(get_records(start_offset=current_offset)):
        process_record(record)
        processed_count += 1
        
        # Checkpoint định kỳ
        if processed_count % CHECKPOINT_INTERVAL == 0:
            save_checkpoint(current_offset + i, processed_count, get_current_state())
    
    # Checkpoint cuối cùng khi hoàn thành
    save_checkpoint(total_records, processed_count, 'completed')
    print(f"Job completed! Total processed: {processed_count}")
```

### Checkpointing với AWS Batch (Batch Job — Công Việc Hàng Loạt)

```yaml
# Job Definition với retry + checkpoint
JobDefinition:
  Type: container
  ContainerProperties:
    Image: "my-batch-job:latest"
    Environment:
      - Name: CHECKPOINT_BUCKET
        Value: !Ref CheckpointBucket
    JobRoleArn: !GetAtt BatchJobRole.Arn
  RetryStrategy:
    Attempts: 5  # Retry tối đa 5 lần
    EvaluateOnExit:
      - OnStatusReason: "Host EC2 terminated"
        Action: RETRY
      - OnReason: "CannotPullContainerError"
        Action: RETRY
      - OnExitCode: "0"
        Action: EXIT
```

---

## 🔄 Spot trong Auto Scaling Groups {#spot-asg}

### Kiến Trúc Được Khuyến Nghị

```
Production ASG với Spot:

┌────────────────────────────────────────────────────────────┐
│                  Auto Scaling Group                        │
│                                                            │
│  On-Demand Base: 2 instances (luôn chạy)                  │
│  Spot Instances: tối đa 18 instances (80% fleet)          │
│                                                            │
│  AZ Distribution:                                          │
│  ├── us-east-1a: 3-7 instances                            │
│  ├── us-east-1b: 3-7 instances                            │
│  └── us-east-1c: 3-7 instances                            │
└────────────────────────────────────────────────────────────┘
         ↕ Scale in/out
[Application Load Balancer — ALB] (Cân Bằng Tải Ứng Dụng)
```

### Capacity Rebalancing (Cân Bằng Lại Năng Lực)

AWS giới thiệu **Capacity Rebalancing** giúp ASG proactively replace (chủ động thay thế) Spot instances có nguy cơ bị interrupted:

```bash
# Bật Capacity Rebalancing khi tạo ASG
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "my-spot-asg" \
  --capacity-rebalance \
  --min-size 2 \
  --max-size 20 \
  --desired-capacity 6 \
  --mixed-instances-policy file://mixed-instances.json \
  --vpc-zone-identifier "subnet-aaa,subnet-bbb,subnet-ccc"

# Khi ASG nhận được "rebalance recommendation" từ EC2:
# 1. Launch new instance TRƯỚC khi Spot bị interrupted
# 2. Sau khi instance mới healthy → terminate instance cũ
# 3. Kết quả: zero downtime khi Spot pool thay đổi
```

### Lifecycle Hook để Handle Interruption

```bash
# Tạo Lifecycle Hook để drain connections trước khi terminate
aws autoscaling put-lifecycle-hook \
  --lifecycle-hook-name "spot-termination-hook" \
  --auto-scaling-group-name "my-spot-asg" \
  --lifecycle-transition "autoscaling:EC2_INSTANCE_TERMINATING" \
  --heartbeat-timeout 120 \
  --default-result "CONTINUE" \
  --notification-target-arn "arn:aws:sqs:us-east-1:123456789:termination-queue"

# SQS consumer script (chạy trong instance hoặc Lambda riêng):
# 1. Nhận message từ SQS
# 2. Deregister instance khỏi Load Balancer target group
# 3. Chờ in-flight requests drain (connection draining)
# 4. Complete lifecycle action
aws autoscaling complete-lifecycle-action \
  --lifecycle-action-result CONTINUE \
  --auto-scaling-group-name "my-spot-asg" \
  --lifecycle-hook-name "spot-termination-hook" \
  --instance-id "$INSTANCE_ID"
```

---

## 🐳 Spot trong ECS & EKS {#spot-containers}

### ECS với Spot (Fargate Spot)

**Fargate Spot** cho phép chạy ECS tasks trên Spot capacity với giá thấp hơn 70%:

```json
// Capacity Provider Strategy (Chiến Lược Nhà Cung Cấp Năng Lực)
{
  "capacityProviderStrategy": [
    {
      "capacityProvider": "FARGATE",
      "base": 1,
      "weight": 1
    },
    {
      "capacityProvider": "FARGATE_SPOT",
      "base": 0,
      "weight": 4
    }
  ]
}
// Kết quả: 80% tasks chạy trên Fargate Spot, 20% trên Fargate thường
// Base = 1: Luôn có ít nhất 1 task FARGATE thường
```

```python
# ECS Task cần handle SIGTERM khi Fargate Spot interrupted
import signal
import sys

def handle_sigterm(signum, frame):
    """Xử lý SIGTERM gracefully khi Fargate Spot bị thu hồi"""
    print("Received SIGTERM — Fargate Spot interrupted. Saving state...")
    save_current_state()
    cleanup_resources()
    sys.exit(0)

signal.signal(signal.SIGTERM, handle_sigterm)

# Main application loop
while True:
    process_next_batch()
```

### EKS với Spot Nodes

```yaml
# Managed Node Group với Spot
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: us-east-1
nodeGroups:
  - name: spot-workers
    instancesDistribution:
      instanceTypes:
        - m5.large
        - m5a.large
        - m6i.large
        - m6a.large
      onDemandBaseCapacity: 0
      onDemandPercentageAboveBaseCapacity: 20
      spotAllocationStrategy: price-capacity-optimized
    minSize: 2
    maxSize: 20
    labels:
      lifecycle: Ec2Spot
    taints:
      - key: spot
        value: "true"
        effect: NoSchedule
```

```yaml
# Pod Toleration để chạy trên Spot nodes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-worker
spec:
  template:
    spec:
      tolerations:
        - key: spot
          operator: Equal
          value: "true"
          effect: NoSchedule
      nodeSelector:
        lifecycle: Ec2Spot
      terminationGracePeriodSeconds: 120  # 2 phút để graceful shutdown
      containers:
        - name: worker
          image: my-batch-worker:latest
```

### AWS Node Termination Handler (NTH) — Xử Lý Khi Node Bị Thu Hồi

```bash
# Cài đặt Node Termination Handler bằng Helm
helm repo add eks https://aws.github.io/eks-charts
helm install aws-node-termination-handler \
  eks/aws-node-termination-handler \
  --namespace kube-system \
  --set enableSpotInterruptionDraining=true \
  --set enableRebalanceMonitoring=true \
  --set enableScheduledEventDraining=true

# NTH làm gì khi Spot node bị interrupted:
# 1. Nhận interruption notice từ IMDS
# 2. Cordon node (đánh dấu không schedule pods mới)
# 3. Drain node (evict pods hiện có để re-schedule trên nodes khác)
# 4. Xóa node khỏi cluster sau khi drain xong
```

---

## 🔍 Spot Advisor & Chọn Instance Type {#spot-advisor}

### Spot Instance Advisor

AWS cung cấp [Spot Instance Advisor](https://aws.amazon.com/ec2/spot/instance-advisor/) hiển thị:
- **Savings vs On-Demand (Tiết Kiệm so với On-Demand):** % discount
- **Frequency of interruption (Tần Suất Gián Đoạn):** < 5%, 5-10%, 10-15%, 15-20%, > 20%

```bash
# Kiểm tra Spot Placement Score — đánh giá khả năng Spot được fulfill
aws ec2 get-spot-placement-scores \
  --instance-types m5.large m5a.large m6i.large \
  --target-capacity 10 \
  --single-availability-zone-flag false \
  --region-names us-east-1
```

### Tiêu Chí Chọn Instance Types Tốt Cho Spot

```
1. Interruption Rate < 5% (theo Spot Advisor)
2. Nhiều Spot Pools có sẵn cho instance family
3. Instance generations mới hơn thường có nhiều pools hơn
4. Tránh quá nhiều GPU instances (ít pools, hay khan)
5. Tránh z1d, h1 (rất specialized, ít pool)

Lựa chọn tốt cho general-purpose workloads:
├── m5, m5a, m6i, m6a, m7i (General Purpose)
├── c5, c5a, c6i, c6a, c7i (Compute Optimized)
└── r5, r5a, r6i, r6a (Memory Optimized)
```

---

## ⚠️ Anti-Patterns — Cạm Bẫy Cần Tránh {#anti-patterns}

### ❌ Anti-Pattern 1: Chỉ Dùng 1 Instance Type

```
Vấn đề: Nếu pool đó hết capacity → fleet bị interrupted hoàn toàn
Giải pháp: Luôn specify ít nhất 4-6 instance types tương đương
```

### ❌ Anti-Pattern 2: Đặt Spot Price Cap Thấp

```
Vấn đề: Spot Price vượt cap → không launch được instance
Thực tế: Spot Price hiếm khi vượt On-Demand price
Giải pháp: Để AWS tự chọn price (không set max price) hoặc set = On-Demand
```

### ❌ Anti-Pattern 3: Dùng Spot cho Stateful Workloads

```
Vấn đề: Database, session server không chịu được interruption đột ngột
Giải pháp: On-Demand hoặc Reserved cho stateful workloads
          Chỉ dùng Spot cho stateless, fault-tolerant workloads
```

### ❌ Anti-Pattern 4: Không Handle Interruption

```
Vấn đề: Instance bị terminate đột ngột → data loss, partial processing
Giải pháp:
├── Implement graceful shutdown handlers
├── Dùng SQS visibility timeout để job tự retry
└── Checkpoint dữ liệu quan trọng lên S3 định kỳ
```

### ❌ Anti-Pattern 5: Dùng Spot cho Critical Deadline Jobs

```
Vấn đề: Job phải xong trước deadline → Spot có thể bị interrupted đúng lúc
Giải pháp: Mix Spot + On-Demand fallback, hoặc dùng On-Demand thuần
          Spot tốt cho "best-effort" (cố gắng tốt nhất) không có deadline cứng
```

---

## 🎤 Câu Hỏi Phỏng Vấn {#câu-hỏi-phỏng-vấn}

**Q: Giải thích cách bạn sẽ chạy một ML training job 8 tiếng trên Spot?**
> (1) Implement checkpointing — lưu model weights lên S3 mỗi 30 phút. (2) Dùng Spot Fleet với `price-capacity-optimized` allocation strategy, chọn 4-5 instance types phù hợp. (3) Script startup tự load checkpoint mới nhất để resume. (4) EventBridge rule gọi Lambda khi interrupted để save final checkpoint. (5) Kết quả: tiết kiệm 65-80% và job sẽ tự resume khi Spot capacity có lại.

**Q: Spot Interruption Rate là gì và nó ảnh hưởng thế nào đến kiến trúc?**
> Interruption Rate là tỷ lệ phần trăm Spot instances bị thu hồi trong một khoảng thời gian. Nếu dùng instances có rate > 15%, cần: (1) nhiều loại instance để fallback, (2) checkpointing thường xuyên hơn, (3) giảm job chunk size. Nếu rate < 5%, kiến trúc đơn giản hơn vì interruption hiếm.

**Q: Capacity Rebalancing trong ASG là gì?**
> Khi AWS dự đoán một Spot instance sắp bị interrupted, nó gửi "rebalance recommendation". Với Capacity Rebalancing bật, ASG tự động launch replacement instance TRƯỚC khi instance hiện tại bị terminate, đảm bảo fleet luôn đủ capacity. Khác với việc chờ instance bị terminated rồi mới launch (phản ứng chậm hơn).

**Q: Fargate Spot khác EC2 Spot thế nào?**
> Về interruption handling: cả hai đều gửi 2-minute notice. Sự khác biệt chính: Fargate Spot không cần bạn quản lý node infrastructure, chỉ cần handle SIGTERM trong application. EC2 Spot cần thêm: instance-level graceful shutdown scripts, deregister từ load balancer, drain connections. Fargate Spot đơn giản hơn về vận hành nhưng ít tuỳ chỉnh hơn.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
