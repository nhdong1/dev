# 🔗 Lifecycle Hooks, Warm Pools & Instance Refresh

> **Lifecycle Hook** — Móc Vòng Đời — cho phép ứng dụng thực hiện các hành động tùy chỉnh khi instance đang được launch hoặc terminate, bằng cách tạm dừng quá trình và chờ tín hiệu "CONTINUE" hoặc "ABANDON". **Warm Pool** — Bể Instance Ấm — giữ sẵn các instance đã được khởi tạo để giảm thời gian scale out.

---

## 📚 Mục Lục

1. [Lifecycle Hooks — Móc Vòng Đời](#lifecycle-hooks)
2. [Launch Lifecycle Hook — Hook Khi Launch](#launch-lifecycle-hook)
3. [Terminate Lifecycle Hook — Hook Khi Terminate](#terminate-lifecycle-hook)
4. [Warm Pools — Bể Instance Ấm](#warm-pools)
5. [Instance Refresh — Làm Mới Instance](#instance-refresh)
6. [Kết Hợp Lifecycle Hooks + EventBridge](#kết-hợp-lifecycle-hooks--eventbridge)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🔗 Lifecycle Hooks — Móc Vòng Đời

### Vấn Đề Lifecycle Hooks Giải Quyết

```
Không có Lifecycle Hooks:

Scale Out:
  ASG launch instance → Instance pass health check → Nhận traffic
  Vấn đề: Ứng dụng chưa kịp đăng ký vào service discovery (Consul/Eureka)
           → Request đến nhưng service chưa sẵn sàng → 502 errors

Scale In (Terminate):
  ASG terminate instance ngay → Connections bị cắt đột ngột
  Vấn đề: In-flight requests (request đang xử lý) bị mất
           → User nhận được lỗi hoặc dữ liệu không nhất quán

Với Lifecycle Hooks:
  → Launch Hook: "Chờ đó, tôi cần đăng ký service discovery trước"
  → Terminate Hook: "Chờ đó, tôi cần drain connections và backup data trước"
```

### Luồng Vòng Đời Đầy Đủ Với Hooks

```
LAUNCH FLOW (Luồng Khởi Chạy):

Pending ──► Pending:Wait ──► [Custom Actions] ──► Pending:Proceed ──► InService
                │                                          │
                │ Timeout (HeartbeatTimeout)               │ CONTINUE signal
                │ hoặc ABANDON signal                      │
                ▼                                          ▼
           Terminated                               InService ✅
           (Instance bị xóa nếu ABANDON)

TERMINATE FLOW (Luồng Xóa):

Terminating ──► Terminating:Wait ──► [Custom Actions] ──► Terminating:Proceed ──► Terminated
                     │                                              │
                     │ Timeout hoặc ABANDON signal                  │ CONTINUE signal
                     ▼                                              ▼
              Force terminate                              Terminated ✅
```

### Tham Số Lifecycle Hook

| Tham Số                 | Mô Tả                                                         | Giá Trị Mặc Định |
| ----------------------- | ------------------------------------------------------------- | :---------------: |
| `HeartbeatTimeout`      | Số giây chờ tối đa trước khi timeout                         | 3600 giây (1 giờ) |
| `DefaultResult`         | Hành động khi timeout: `CONTINUE` hoặc `ABANDON`            | `ABANDON`         |
| `NotificationTargetARN` | SNS topic hoặc SQS queue nhận thông báo khi hook triggered   | Không bắt buộc    |
| `RoleARN`               | IAM Role ASG dùng để publish thông báo                       | Không bắt buộc    |

---

## 🚀 Launch Lifecycle Hook — Hook Khi Launch

### Use Cases Phổ Biến

| Use Case                         | Mô Tả                                                                 |
| --------------------------------- | --------------------------------------------------------------------- |
| **Service Discovery Registration**| Đăng ký instance vào Consul, Eureka, hoặc Route 53                  |
| **Configuration Pull**           | Tải cấu hình từ S3, Parameter Store, Secrets Manager                |
| **Custom Health Check**          | Kiểm tra ứng dụng đã thực sự sẵn sàng phục vụ (beyond OS health)    |
| **Security Scan**                | Chạy vulnerability scan (quét lỗ hổng) trước khi nhận traffic       |
| **Data Loading**                 | Pre-load cache hoặc data cần thiết cho ứng dụng                     |

### Ví Dụ: Đăng Ký Vào Service Discovery

```bash
# Tạo Launch Lifecycle Hook
aws autoscaling put-lifecycle-hook \
  --auto-scaling-group-name "web-servers-asg" \
  --lifecycle-hook-name "register-service-discovery" \
  --lifecycle-transition "autoscaling:EC2_INSTANCE_LAUNCHING" \
  --heartbeat-timeout 300 \
  --default-result "ABANDON" \
  --notification-target-arn "arn:aws:sqs:us-east-1:123456789:asg-launch-queue" \
  --role-arn "arn:aws:iam::123456789:role/asg-lifecycle-role"
```

```python
# Lambda function xử lý Launch Hook
import boto3
import json
import time

def handler(event, context):
    """
    Triggered từ SQS/SNS khi instance đang ở Pending:Wait state
    Nhiệm vụ: Đăng ký instance vào service discovery
    """
    asg_client = boto3.client('autoscaling')

    # Parse thông tin từ event
    message = json.loads(event['Records'][0]['body'])
    instance_id = message['EC2InstanceId']
    asg_name = message['AutoScalingGroupName']
    lifecycle_token = message['LifecycleActionToken']
    hook_name = message['LifecycleHookName']

    try:
        # Thực hiện custom action: đăng ký service discovery
        register_to_consul(instance_id)

        # Chờ ứng dụng sẵn sàng (custom health check)
        wait_for_app_ready(instance_id)

        # Gửi CONTINUE signal → Instance chuyển sang InService
        asg_client.complete_lifecycle_action(
            AutoScalingGroupName=asg_name,
            LifecycleHookName=hook_name,
            LifecycleActionToken=lifecycle_token,
            LifecycleActionResult='CONTINUE'
        )
        print(f"Instance {instance_id} successfully registered, sending CONTINUE")

    except Exception as e:
        print(f"Failed to register instance {instance_id}: {e}")

        # Gửi ABANDON signal → Instance bị terminate, ASG sẽ thử lại
        asg_client.complete_lifecycle_action(
            AutoScalingGroupName=asg_name,
            LifecycleHookName=hook_name,
            LifecycleActionToken=lifecycle_token,
            LifecycleActionResult='ABANDON'
        )


def register_to_consul(instance_id: str):
    """Đăng ký instance vào Consul service discovery"""
    import requests
    import subprocess

    # Lấy private IP của instance
    ec2 = boto3.client('ec2')
    instance = ec2.describe_instances(InstanceIds=[instance_id])
    private_ip = instance['Reservations'][0]['Instances'][0]['PrivateIpAddress']

    # Đăng ký vào Consul
    payload = {
        "ID": instance_id,
        "Name": "web-server",
        "Address": private_ip,
        "Port": 8080,
        "Check": {
            "HTTP": f"http://{private_ip}:8080/health",
            "Interval": "10s"
        }
    }
    requests.put("http://consul:8500/v1/agent/service/register", json=payload)


def wait_for_app_ready(instance_id: str, max_wait: int = 120):
    """Chờ application endpoint trả về 200 OK"""
    import requests

    ec2 = boto3.client('ec2')
    instance = ec2.describe_instances(InstanceIds=[instance_id])
    private_ip = instance['Reservations'][0]['Instances'][0]['PrivateIpAddress']

    for attempt in range(max_wait // 5):
        try:
            response = requests.get(f"http://{private_ip}:8080/health", timeout=5)
            if response.status_code == 200:
                return True
        except:
            pass
        time.sleep(5)

    raise Exception(f"Application not ready after {max_wait} seconds")
```

### Heartbeat — Tín Hiệu Còn Sống

Nếu custom action tốn nhiều hơn `HeartbeatTimeout` giây, cần gửi heartbeat để gia hạn:

```python
# Gửi heartbeat mỗi 60 giây trong khi đang xử lý
def send_heartbeat(asg_name, hook_name, token):
    asg_client = boto3.client('autoscaling')
    asg_client.record_lifecycle_action_heartbeat(
        AutoScalingGroupName=asg_name,
        LifecycleHookName=hook_name,
        LifecycleActionToken=token
    )
    # Heartbeat gia hạn thêm HeartbeatTimeout giây
```

---

## 💀 Terminate Lifecycle Hook — Hook Khi Terminate

### Use Cases Phổ Biến

| Use Case                    | Mô Tả                                                                       |
| ---------------------------- | --------------------------------------------------------------------------- |
| **Connection Draining**      | Chờ in-flight requests (request đang xử lý) hoàn thành                    |
| **Log Flushing**            | Đảm bảo logs được gửi đến CloudWatch/Elasticsearch trước khi terminate     |
| **Deregistration**          | Xóa instance khỏi service discovery (Consul, Eureka)                       |
| **Data Backup**             | Backup dữ liệu tạm thời trên local disk lên S3                            |
| **Spot Interruption**       | Xử lý graceful shutdown khi Spot bị reclaim                                |

### Ví Dụ: Graceful Drain + Deregister

```bash
#!/bin/bash
# /usr/local/bin/graceful-shutdown.sh
# Chạy trong instance khi nhận SIGTERM

set -e

# Lấy thông tin instance từ metadata
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)

# Biến từ environment (set bởi cloud-init)
ASG_NAME="${AUTO_SCALING_GROUP_NAME}"
HOOK_NAME="graceful-terminate"
TARGET_GROUP_ARN="${TARGET_GROUP_ARN}"

echo "[$(date)] Starting graceful shutdown for $INSTANCE_ID"

# Bước 1: Deregister từ Load Balancer Target Group
echo "[$(date)] Deregistering from target group..."
aws elbv2 deregister-targets \
  --region "$REGION" \
  --target-group-arn "$TARGET_GROUP_ARN" \
  --targets "Id=$INSTANCE_ID"

# Bước 2: Chờ connection drain (tối đa 90 giây)
echo "[$(date)] Waiting for connections to drain..."
sleep 90

# Bước 3: Dừng ứng dụng gracefully
echo "[$(date)] Stopping application..."
systemctl stop myapp

# Bước 4: Flush logs còn trong buffer
echo "[$(date)] Flushing log buffer..."
pkill -SIGUSR1 fluentd  # Signal fluentd flush buffer

# Bước 5: Deregister từ Consul
echo "[$(date)] Deregistering from Consul..."
curl -X PUT "http://consul:8500/v1/agent/service/deregister/$INSTANCE_ID"

# Bước 6: Backup dữ liệu quan trọng (nếu cần)
if [ -d /var/app/data ]; then
  echo "[$(date)] Backing up application data..."
  aws s3 sync /var/app/data "s3://my-backup-bucket/instances/$INSTANCE_ID/"
fi

# Bước 7: Gửi CONTINUE signal → ASG terminate instance
echo "[$(date)] Sending CONTINUE signal to ASG..."
LIFECYCLE_TOKEN=$(cat /tmp/lifecycle-token)
aws autoscaling complete-lifecycle-action \
  --region "$REGION" \
  --auto-scaling-group-name "$ASG_NAME" \
  --lifecycle-hook-name "$HOOK_NAME" \
  --instance-id "$INSTANCE_ID" \
  --lifecycle-action-token "$LIFECYCLE_TOKEN" \
  --lifecycle-action-result "CONTINUE"

echo "[$(date)] Graceful shutdown complete"
```

```bash
# Tạo Terminate Lifecycle Hook
aws autoscaling put-lifecycle-hook \
  --auto-scaling-group-name "web-servers-asg" \
  --lifecycle-hook-name "graceful-terminate" \
  --lifecycle-transition "autoscaling:EC2_INSTANCE_TERMINATING" \
  --heartbeat-timeout 180 \
  --default-result "CONTINUE"
```

---

## 🌡️ Warm Pools — Bể Instance Ấm

### Vấn Đề Warm Pools Giải Quyết

```
Vấn đề với Cold Start (Khởi Động Lạnh):

Scale Out Event:
  1. ASG quyết định launch instance → 0 giây
  2. EC2 instance boot (khởi động) → 30-60 giây
  3. OS initializes → 30-60 giây
  4. Application starts → 30-120 giây
  5. Application warm up (cache, connections) → 30-60 giây
  ──────────────────────────────────────────────────────
  Tổng: 2-5 phút trước khi instance sẵn sàng nhận traffic

Nếu scale out xảy ra khi traffic đang spike:
  → 2-5 phút latency tăng → user experience kém
  → Trong thời gian này: instances cũ bị overload

Với Warm Pool:
  → Pool của pre-initialized instances (đã boot, app đã start)
  → Scale out → Lấy instance từ warm pool → Ready trong < 30 giây ✅
```

### Kiến Trúc Warm Pool

```
                    ┌──────────────────────────────────────┐
                    │         Auto Scaling Group           │
                    │                                      │
                    │  InService: [EC2][EC2][EC2][EC2]    │
                    │  (Đang nhận traffic)                 │
                    └──────────────────────────────────────┘
                                      ▲
                         Khi scale out: lấy từ Warm Pool
                                      │
                    ┌──────────────────────────────────────┐
                    │              Warm Pool               │
                    │                                      │
                    │  [EC2-stopped][EC2-stopped][EC2-running] │
                    │                                      │
                    │  Instance state tùy chọn:            │
                    │  - Stopped (tiết kiệm chi phí)       │
                    │  - Running (sẵn sàng ngay nhất)      │
                    │  - Hibernated (trung gian)           │
                    └──────────────────────────────────────┘
                                      ▲
                         Khi pool thiếu: ASG launch mới
```

### Instance States Trong Warm Pool

| State          | Thời Gian Để InService | Chi Phí         | Dùng Khi                              |
| -------------- | :---------------------: | :-------------: | ------------------------------------- |
| **Stopped**    | 20-60 giây              | Chỉ EBS         | Bootstrap time dài nhưng muốn tiết kiệm |
| **Running**    | < 10 giây               | EC2 + EBS       | Cần sẵn sàng cực nhanh               |
| **Hibernated** | 10-30 giây              | EC2 + EBS (thấp)| Trung gian (RAM được lưu vào EBS)    |

### Tạo Warm Pool

```bash
# Tạo Warm Pool với tối đa 5 instances ở trạng thái Stopped
aws autoscaling put-warm-pool \
  --auto-scaling-group-name "web-servers-asg" \
  --pool-state "Stopped" \
  --min-size 2 \
  --max-group-prepared-capacity 5

# Tạo Warm Pool với instances Running (sẵn sàng ngay)
aws autoscaling put-warm-pool \
  --auto-scaling-group-name "web-servers-asg" \
  --pool-state "Running" \
  --min-size 1 \
  --max-group-prepared-capacity 3
```

### Chi Phí Warm Pool

```
Warm Pool = Stopped Instances:
  Chi phí: Chỉ trả tiền EBS volume (không trả EC2 instance hours)
  Ví dụ: 3 stopped instances với 50GB EBS mỗi cái
    = 3 × 50GB × $0.10/GB/tháng = $15/tháng
    (So với 3 running instances: 3 × $0.096/giờ × 720 giờ = $207/tháng)

Warm Pool = Running Instances:
  Chi phí: EC2 + EBS (như instance bình thường)
  Nhưng: Không tính vào production load → không cần instance to
```

### Lifecycle Hook Với Warm Pool

Warm Pool hỗ trợ riêng lifecycle hooks cho quá trình "warm up":

```bash
# Hook khi instance đang được thêm vào Warm Pool (từ Cold)
aws autoscaling put-lifecycle-hook \
  --auto-scaling-group-name "web-servers-asg" \
  --lifecycle-hook-name "warm-pool-init" \
  --lifecycle-transition "autoscaling:EC2_INSTANCE_LAUNCHING" \
  --heartbeat-timeout 600

# Hook khi instance chuyển từ Warm Pool vào ASG (InService)
# Dùng để "final preparation" trước khi nhận traffic
```

---

## 🔄 Instance Refresh — Làm Mới Instance

### Mục Đích

Instance Refresh cho phép rolling replacement (thay thế cuộn) tất cả instances trong ASG với cấu hình mới (AMI, instance type, security patch...) mà không cần recreate ASG hay gây downtime.

### Cách Hoạt Động

```
Desired = 10 instances, MinHealthyPercentage = 80%

Bước 1: Tính số instances có thể thay thế cùng lúc
  10 × (100% - 80%) = 2 instances có thể được terminate cùng lúc

Bước 2: Vòng lặp thay thế
  Round 1: Terminate 2 instances cũ → Launch 2 instances mới
           Chờ 2 instances mới healthy
           (8 instances cũ + 2 instances mới = 10 tổng)
  Round 2: Terminate 2 instances cũ nữa → Launch 2 instances mới
  ...
  Round 5: Tất cả 10 instances đã được thay thế ✅

Tổng thời gian: 5 rounds × (launch time + warmup) ≈ 20-30 phút
```

### Tạo Instance Refresh

```bash
# Instance Refresh cơ bản
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name "web-servers-asg" \
  --preferences '{
    "MinHealthyPercentage": 90,
    "InstanceWarmup": 300
  }'

# Instance Refresh với Checkpoints — Dừng để review tại các mốc
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name "web-servers-asg" \
  --preferences '{
    "MinHealthyPercentage": 90,
    "InstanceWarmup": 300,
    "CheckpointPercentages": [20, 50, 100],
    "CheckpointDelay": 600
  }'
```

### Checkpoints — Điểm Dừng Để Kiểm Tra

```
CheckpointPercentages: [20, 50, 100]
CheckpointDelay: 600 (10 phút)

Luồng:
  Refresh 0%  → 20% instances replaced → PAUSE 10 phút
               (Review: metrics OK? errors tăng không?)
               → [Tự động tiếp tục sau 10 phút, hoặc manual resume]
  
  Refresh 20% → 50% instances replaced → PAUSE 10 phút
               (Review: performance OK với 50% fleet mới?)
  
  Refresh 50% → 100% instances replaced → DONE
```

### Cancel Instance Refresh — Hủy Làm Mới

```bash
# Cancel nếu phát hiện vấn đề
aws autoscaling cancel-instance-refresh \
  --auto-scaling-group-name "web-servers-asg"

# Cancel chỉ dừng instances mới từ bị replace
# Instances đã được replace vẫn giữ nguyên (không rollback)
# Phải tạo Instance Refresh mới với version cũ để rollback
```

### Rollback Instance Refresh

```bash
# Để rollback: tạo Launch Template version mới trỏ về AMI cũ
aws ec2 create-launch-template-version \
  --launch-template-name "web-server-lt" \
  --source-version "3" \
  --launch-template-data '{"ImageId": "ami-OLD_AMI_ID"}'

# Cập nhật $Default về version rollback
aws ec2 modify-launch-template \
  --launch-template-name "web-server-lt" \
  --default-version "4"

# Chạy Instance Refresh với version rollback
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name "web-servers-asg" \
  --preferences '{"MinHealthyPercentage": 90, "InstanceWarmup": 120}'
```

---

## ⚡ Kết Hợp Lifecycle Hooks + EventBridge

### Architecture Pattern — Mẫu Kiến Trúc

```
                    ┌────────────────────┐
                    │  Auto Scaling Group│
                    │  Launch/Terminate  │
                    └─────────┬──────────┘
                              │ Lifecycle Event
                    ┌─────────▼──────────┐
                    │   Amazon EventBridge│
                    │   (Event Bus)       │
                    └─────────┬──────────┘
                              │ Rule match
              ┌───────────────┼───────────────┐
              │               │               │
    ┌─────────▼───┐ ┌────────▼──────┐ ┌──────▼──────┐
    │   Lambda    │ │  Step Functions│ │    SQS      │
    │  (Nhanh,    │ │  (Workflow     │ │  (Worker    │
    │   đơn giản) │ │   phức tạp)   │ │   queue)    │
    └─────────────┘ └───────────────┘ └─────────────┘
```

### EventBridge Rule Cho ASG Lifecycle Events

```json
// Rule pattern cho Launch events
{
  "source": ["aws.autoscaling"],
  "detail-type": ["EC2 Instance-launch Lifecycle Action"],
  "detail": {
    "AutoScalingGroupName": ["web-servers-asg"]
  }
}

// Rule pattern cho Terminate events
{
  "source": ["aws.autoscaling"],
  "detail-type": ["EC2 Instance-terminate Lifecycle Action"],
  "detail": {
    "AutoScalingGroupName": ["web-servers-asg"]
  }
}
```

### Step Functions Cho Workflow Phức Tạp

```json
// State Machine cho Launch workflow
{
  "Comment": "Instance Launch Workflow",
  "StartAt": "PullConfiguration",
  "States": {
    "PullConfiguration": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:pull-config",
      "Next": "RegisterServiceDiscovery"
    },
    "RegisterServiceDiscovery": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:register-consul",
      "Retry": [{"ErrorEquals": ["States.ALL"], "MaxAttempts": 3}],
      "Next": "WaitForAppReady"
    },
    "WaitForAppReady": {
      "Type": "Wait",
      "Seconds": 30,
      "Next": "HealthCheck"
    },
    "HealthCheck": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:health-check",
      "Next": "SendContinue"
    },
    "SendContinue": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:complete-lifecycle",
      "Parameters": {
        "LifecycleActionResult": "CONTINUE"
      },
      "End": true
    }
  }
}
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Khi nào cần Lifecycle Hook? Cho ví dụ thực tế.**

> Cần Lifecycle Hook khi quá trình launch hoặc terminate instance yêu cầu thời gian để hoàn thành các tác vụ quan trọng mà ASG không tự xử lý. Ví dụ thực tế: (1) Launch Hook — Ứng dụng microservice cần đăng ký vào Consul service discovery và load cấu hình từ Secrets Manager trước khi nhận traffic; nếu ASG đưa instance vào InService ngay, load balancer có thể route request đến khi ứng dụng chưa sẵn sàng. (2) Terminate Hook — Ứng dụng xử lý message từ SQS cần drain (xử lý hết) message đang xử lý và dequeue bất kỳ message đang hold trước khi terminate để tránh mất data.

**Q: Warm Pool phù hợp với workload nào? Chi phí ra sao?**

> Warm Pool phù hợp khi bootstrap time (thời gian khởi tạo) của instance dài — ứng dụng Java Spring với JVM warm-up, hoặc instances cần tải model ML lớn. Khi scale out xảy ra, instance từ warm pool (đã initialized) được sử dụng ngay thay vì phải chờ cold start 3-5 phút. Chi phí: nếu Warm Pool dùng `Stopped` state, chỉ trả tiền EBS storage (rất rẻ) chứ không trả instance hours. Trade-off: cần thiết kế ứng dụng để handle việc instance chuyển từ stopped/warm-pool state sang InService.

**Q: Instance Refresh và Blue/Green Deployment khác nhau như thế nào?**

> Instance Refresh là **in-place rolling replacement** (thay thế cuộn tại chỗ) trong cùng một ASG — thay thế từng batch instance cũ bằng instance mới. Không cần tạo environment mới. Blue/Green Deployment tạo **environment hoàn toàn mới** (Green) song song với environment cũ (Blue), sau khi Green verified → switch traffic 100%. Instance Refresh đơn giản hơn và ít tốn kém hơn, nhưng rollback chậm hơn (phải refresh ngược lại). Blue/Green cho phép rollback instant (chỉ cần switch traffic về Blue) nhưng tốn gấp đôi chi phí trong thời gian chuyển đổi.

**Q: Default result của Lifecycle Hook nên là CONTINUE hay ABANDON?**

> Cho **Launch Hook**: `ABANDON` là an toàn hơn — nếu custom action timeout (ví dụ: Lambda bị lỗi, Consul không đăng ký được), instance bị terminate thay vì được đưa vào service ở trạng thái chưa sẵn sàng. ASG sẽ launch instance mới để thử lại. Cho **Terminate Hook**: `CONTINUE` là an toàn hơn — nếu timeout, instance vẫn bị terminate (tránh "zombie instance" sống mãi). Trong cả hai trường hợp, nên thiết kế custom action để gửi tín hiệu rõ ràng thay vì dựa vào timeout.

---

## 🔗 Điều Hướng

| Trước                                                 | Tiếp Theo                                            |
| ----------------------------------------------------- | ---------------------------------------------------- |
| [← Spot & Mixed Instances](4-spot-mixed-instances.md) | [Serverless Lambda →](../03-serverless-lambda/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
