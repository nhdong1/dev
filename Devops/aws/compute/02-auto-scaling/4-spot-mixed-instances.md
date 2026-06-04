# 💰 Spot Instances & Mixed Instance Policy — Tối Ưu Chi Phí ASG

> **Spot Instance** — Máy Chủ Tạm Thời — cho phép bạn dùng EC2 capacity dư thừa của AWS với giá giảm đến 90% so với On-Demand. **Mixed Instance Policy** — Chính Sách Hỗn Hợp — kết hợp Spot và On-Demand trong cùng một ASG để tối ưu chi phí mà vẫn đảm bảo availability.

---

## 📚 Mục Lục

1. [Spot Instance Là Gì?](#spot-instance-là-gì)
2. [Spot Interruption — Gián Đoạn Spot](#spot-interruption)
3. [Mixed Instance Policy — Chính Sách Hỗn Hợp](#mixed-instance-policy)
4. [Allocation Strategies — Chiến Lược Phân Bổ](#allocation-strategies)
5. [Xử Lý Spot Interruption Trong Ứng Dụng](#xử-lý-spot-interruption-trong-ứng-dụng)
6. [Capacity Rebalancing — Cân Bằng Lại Capacity](#capacity-rebalancing)
7. [Tạo ASG Với Mixed Instance Policy](#tạo-asg-với-mixed-instance-policy)
8. [Spot Best Practices](#spot-best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 💡 Spot Instance Là Gì?

### Cơ Chế Hoạt Động

AWS có data centers với EC2 capacity dư thừa (không ai đặt). Thay vì để trống, AWS cho thuê với giá thấp hơn nhiều — đây là **Spot Instances**.

```
Spot vs On-Demand Pricing:

On-Demand m5.xlarge:    $0.192/giờ  = $138/tháng   (100%)
Spot m5.xlarge:         $0.058/giờ  = $42/tháng    (~30%) → Tiết kiệm 70% ✅

Spot c5.2xlarge:        $0.082/giờ
On-Demand c5.2xlarge:   $0.340/giờ  → Tiết kiệm 76%

Mức tiết kiệm trung bình: 60-90% so với On-Demand
```

### Spot vs On-Demand vs Reserved

| Đặc Điểm                   | On-Demand         | Reserved (1 năm)  | Spot              |
| --------------------------- | :---------------: | :---------------: | :---------------: |
| **Giá tương đối**           | 100%              | ~40%              | ~10-40%           |
| **Cam kết thời gian**       | Không             | 1-3 năm           | Không             |
| **Có thể bị terminate?**    | Không             | Không             | **Có** (2 phút notice) |
| **Tốt nhất cho**            | Workload linh hoạt| Baseline workload | Fault-tolerant workload |

---

## ⚡ Spot Interruption — Gián Đoạn Spot

### Khi Nào Spot Bị Terminate?

Khi AWS cần lại capacity (ví dụ: có nhiều On-Demand request), AWS sẽ **reclaim** (thu hồi) Spot instance:

```
Luồng Spot Interruption:

T+0:  AWS quyết định reclaim Spot instance
T+0:  AWS gửi Interruption Notice vào instance metadata
      → GET /latest/meta-data/spot/instance-action
      → Response: {"action": "terminate", "time": "2024-01-15T10:02:00Z"}
T+0:  CloudWatch Event được phát ra (event: EC2 Spot Instance Interruption Warning)
T+2m: AWS terminate instance (đúng 2 phút sau notice)

Thời gian ứng dụng có để:
  → Drain connections hiện tại
  → Save state/checkpoint
  → Deregister từ load balancer
  → Gửi logs cuối cùng
```

### Tần Suất Interruption

Thực tế, mức interruption rate (tỷ lệ gián đoạn) của Spot:
- **< 5% tháng**: Đa số instance types và regions
- **< 1% tháng**: Nhiều instance families phổ biến
- Dùng **Spot Instance Advisor** (công cụ của AWS) để xem tần suất gián đoạn theo từng type

---

## 🔀 Mixed Instance Policy — Chính Sách Hỗn Hợp

### Tại Sao Cần Mixed Instance?

```
Chỉ dùng Spot (100%):
  → Tiết kiệm chi phí tối đa
  → Nhưng: ASG có thể không fulfill đủ capacity nếu Spot không có
  → Risk: Toàn bộ fleet bị interrupt cùng lúc ❌

Chỉ dùng On-Demand (100%):
  → Luôn có capacity
  → Nhưng: Tốn kém ❌

Mixed (Ví dụ 30% On-Demand + 70% Spot):
  → On-Demand đảm bảo min capacity luôn có
  → Spot tiết kiệm cho phần còn lại
  → Tối ưu cả cost lẫn availability ✅
```

### Kiến Trúc Mixed Instance Fleet

```
ASG với Mixed Instance Policy:
  Total Desired: 10 instances

  On-Demand Base (Sàn On-Demand):
    → 2 instances (luôn là On-Demand)
    → Baseline capacity không bao giờ bị interrupt

  On-Demand Percentage (% On-Demand cho phần mở rộng):
    → 20% of (10-2) = 20% of 8 = ~2 instances

  Spot:
    → 8 - 2 = 6 instances (từ nhiều instance types)

  Kết quả:
    → 4 On-Demand instances (luôn ổn định)
    → 6 Spot instances      (tiết kiệm ~65%)
    → Tổng tiết kiệm ≈ 40% so với all On-Demand
```

---

## 🎯 Allocation Strategies — Chiến Lược Phân Bổ

### Spot Allocation Strategies

| Strategy                        | Mô Tả                                                                    | Khi Nào Dùng                                   |
| -------------------------------- | ------------------------------------------------------------------------ | ---------------------------------------------- |
| **`price-capacity-optimized`**  | Chọn pool (nhóm) có capacity nhiều nhất + giá tốt — **Khuyến nghị mới** | Hầu hết workload, ít interruption nhất         |
| **`capacity-optimized`**        | Chọn pool có capacity nhiều nhất, không quan tâm giá                     | Khi cần minimize interruption tuyệt đối        |
| **`lowest-price`**              | Chọn pool rẻ nhất, chia đều trên N pools (N=2 mặc định)                 | Batch jobs ngắn, chi phí tối ưu là ưu tiên số 1|
| **`diversified`**               | Chia đều trên tất cả pools được chỉ định                                | Đa dạng hóa tối đa để tránh single-pool failure|

### On-Demand Allocation Strategies

| Strategy              | Mô Tả                                                |
| ---------------------- | ---------------------------------------------------- |
| **`lowest-price`**    | Dùng instance type On-Demand rẻ nhất đáp ứng yêu cầu |
| **`prioritized`**     | Dùng instance type theo thứ tự ưu tiên bạn định nghĩa |

### Tại Sao `price-capacity-optimized` Tốt Hơn `lowest-price`?

```
lowest-price strategy:
  → Chọn Spot pool rẻ nhất
  → Pool rẻ nhất thường có ÍT capacity dư thừa
  → → Interruption rate cao hơn
  → Cost savings thực tế thấp hơn dự kiến (do phải replace thường xuyên)

price-capacity-optimized:
  → Cân bằng giữa giá và availability
  → Pool có nhiều capacity → interruption rate thấp hơn
  → Tổng cost thực tế thường tốt hơn dù đơn giá có thể cao hơn một chút
```

---

## 🛡️ Xử Lý Spot Interruption Trong Ứng Dụng

### Phương Pháp 1: Polling Instance Metadata

```bash
#!/bin/bash
# Chạy trong background để theo dõi interruption notice

METADATA_URL="http://169.254.169.254/latest/meta-data/spot/instance-action"

while true; do
  HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
    -H "X-aws-ec2-metadata-token: $TOKEN" \
    "$METADATA_URL")

  if [ "$HTTP_CODE" = "200" ]; then
    echo "Spot interruption notice received! Initiating graceful shutdown..."

    # Deregister từ load balancer
    aws elbv2 deregister-targets \
      --target-group-arn "$TARGET_GROUP_ARN" \
      --targets "Id=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)"

    # Drain connections: chờ max 90 giây
    sleep 90

    # Stop ứng dụng gracefully
    systemctl stop myapp

    break
  fi

  sleep 5  # Kiểm tra mỗi 5 giây
done
```

### Phương Pháp 2: EventBridge Rule + Lambda

```python
# Lambda function xử lý Spot interruption event
import boto3
import json

def handler(event, context):
    # event từ EventBridge: EC2 Spot Instance Interruption Warning
    instance_id = event['detail']['instance-id']
    region = event['region']

    ec2 = boto3.client('ec2', region_name=region)
    asg = boto3.client('autoscaling', region_name=region)

    # Tìm ASG chứa instance này
    response = asg.describe_auto_scaling_instances(
        InstanceIds=[instance_id]
    )

    if not response['AutoScalingInstances']:
        return

    asg_name = response['AutoScalingInstances'][0]['AutoScalingGroupName']

    # Detach instance khỏi Load Balancer (ALB sẽ drain connections)
    asg.detach_load_balancer_target_groups(
        AutoScalingGroupName=asg_name,
        TargetGroupARNs=[TARGET_GROUP_ARN]
    )

    print(f"Instance {instance_id} detached from ALB due to Spot interruption")
```

```json
// EventBridge Rule Pattern
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Spot Instance Interruption Warning"]
}
```

### Phương Pháp 3: Lifecycle Hook (Xem File 5)

ASG Lifecycle Hook tạm dừng quá trình terminate, cho ứng dụng thời gian graceful shutdown. Xem `5-lifecycle-hooks.md` để biết chi tiết.

---

## ⚖️ Capacity Rebalancing — Cân Bằng Lại Capacity

**Capacity Rebalancing** — Cân Bằng Lại Capacity — là tính năng giúp ASG chủ động thay thế Spot instance **trước khi** nó bị interrupt:

```
Cách Hoạt Động:

Không có Capacity Rebalancing:
  AWS gửi Interruption Notice → 2 phút sau terminate
  → Bạn chỉ có 2 phút để thay thế

Với Capacity Rebalancing:
  AWS phát hiện Spot instance "at elevated risk" (nguy cơ cao bị interrupt)
  → ASG nhận được "EC2 Instance Rebalance Recommendation" event
  → ASG ngay lập tức launch instance thay thế (Spot khác hoặc On-Demand)
  → Sau khi instance mới healthy → terminate instance cũ
  → Không có gap trong capacity ✅
```

### Kích Hoạt Capacity Rebalancing

```bash
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name "web-servers-asg" \
  --capacity-rebalance
```

---

## 💻 Tạo ASG Với Mixed Instance Policy

### Ví Dụ Hoàn Chỉnh

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "web-servers-mixed-asg" \
  --min-size 4 \
  --max-size 40 \
  --desired-capacity 10 \
  --vpc-zone-identifier "subnet-aaa,subnet-bbb,subnet-ccc" \
  --health-check-type "ELB" \
  --health-check-grace-period 300 \
  --capacity-rebalance \
  --mixed-instances-policy '{
    "LaunchTemplate": {
      "LaunchTemplateSpecification": {
        "LaunchTemplateName": "web-server-lt",
        "Version": "$Latest"
      },
      "Overrides": [
        {
          "InstanceType": "m5.xlarge"
        },
        {
          "InstanceType": "m5a.xlarge"
        },
        {
          "InstanceType": "m5n.xlarge"
        },
        {
          "InstanceType": "m4.xlarge"
        },
        {
          "InstanceType": "c5.2xlarge"
        },
        {
          "InstanceType": "c5a.2xlarge"
        }
      ]
    },
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 2,
      "OnDemandPercentageAboveBaseCapacity": 20,
      "SpotAllocationStrategy": "price-capacity-optimized",
      "OnDemandAllocationStrategy": "lowest-price"
    }
  }'
```

### Giải Thích Cấu Hình

```
OnDemandBaseCapacity: 2
  → Luôn có 2 instance On-Demand, bất kể desired là bao nhiêu

OnDemandPercentageAboveBaseCapacity: 20
  → Trong số instances NGOÀI 2 On-Demand base:
     20% là On-Demand, 80% là Spot

Ví dụ với Desired = 10:
  On-Demand base:       2 instances
  Phần còn lại:        8 instances (10 - 2)
    → On-Demand 20%:   2 instances (8 × 20%)
    → Spot 80%:        6 instances (8 × 80%)
  Tổng:
    → 4 On-Demand + 6 Spot

Overrides (danh sách instance types):
  → ASG tự chọn instance type phù hợp từ danh sách này
  → Dựa theo allocation strategy đã chọn
  → Đa dạng instance types = ít bị impact khi một pool hết capacity
```

### Dùng Instance Requirements Thay Cho Danh Sách Cố Định

```json
"Overrides": [
  {
    "InstanceRequirements": {
      "VCpuCount": {"Min": 4, "Max": 8},
      "MemoryMiB": {"Min": 8192, "Max": 16384},
      "InstanceGenerations": ["current"],
      "ExcludedInstanceTypes": ["t2.*", "t3.*"],
      "SpotMaxPricePercentageOverLowestPrice": 50
    }
  }
]
```

---

## ✅ Spot Best Practices

### Thiết Kế Ứng Dụng

```
1. Stateless (Không Trạng Thái):
   → Không lưu session data trên local disk của instance
   → Session data → ElastiCache / DynamoDB
   → File uploads → S3 trực tiếp (không qua instance)

2. Checkpoint (Điểm Kiểm Tra):
   → Batch/processing jobs: lưu tiến độ định kỳ
   → Khi Spot bị terminate: instance mới tiếp tục từ checkpoint
   → Không cần restart từ đầu

3. Graceful Shutdown (Tắt Máy Tốt Đẹp):
   → Handle SIGTERM signal
   → Drain in-flight requests trước khi terminate
   → Deregister từ ALB trước 2 phút deadline

4. Idempotent Operations (Phép Toán Bất Biến):
   → Nếu job chạy lại từ đầu do interruption: kết quả vẫn đúng
   → Không tạo duplicate records
```

### Chọn Instance Types Đúng

```
Diversification (Đa Dạng Hóa) — Nguyên Tắc Quan Trọng Nhất:

❌ Sai: Chỉ dùng 1 instance type
  → Nếu pool đó hết capacity → Không launch được instance nào
  → Hoặc interruption rate cao khi pool đó bận

✅ Đúng: 5-10 instance types cùng family/size
  → m5.xlarge, m5a.xlarge, m5n.xlarge, m4.xlarge, c5.2xlarge...
  → Nếu pool này hết → Dùng pool khác
  → Interruption rate thấp hơn nhiều

Nguyên Tắc Chọn Instance:
  → Cùng vCPU và RAM (để app chạy đúng)
  → Khác families/generations (để đa dạng pools)
  → Dùng Instance Advisor để tránh pools có interruption rate cao
```

### Monitoring Spot Fleet

```bash
# Xem Spot interruption events trong 24h qua
aws cloudwatch get-metric-statistics \
  --namespace "AWS/AutoScaling" \
  --metric-name "GroupTerminatingInstances" \
  --dimensions "Name=AutoScalingGroupName,Value=web-servers-mixed-asg" \
  --start-time "$(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period 3600 \
  --statistics Sum

# Xem tỷ lệ Spot vs On-Demand trong ASG
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names "web-servers-mixed-asg" \
  --query 'AutoScalingGroups[0].Instances[*].{ID:InstanceId,Type:InstanceType,Market:Market,State:HealthStatus}'
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Ứng dụng nào phù hợp với Spot Instance? Ứng dụng nào không?**

> **Phù hợp:** Batch processing (xử lý hàng loạt), CI/CD workers, rendering, data processing, stateless web servers, development/test environments. **Không phù hợp:** Database chính (primary database), ứng dụng yêu cầu sessions dài, workload có stateful data quan trọng không thể checkpoint, bất kỳ thứ gì không chịu được gián đoạn đột ngột 2 phút.

**Q: Giải thích Mixed Instance Policy và tại sao nên dùng nhiều instance types?**

> Mixed Instance Policy cho phép ASG dùng cả On-Demand (cho base stability) và Spot (cho cost savings) trong cùng một nhóm. Nhiều instance types quan trọng vì mỗi instance type là một Spot pool riêng. Nếu một pool hết capacity hoặc có interruption rate cao, ASG tự động dùng pool khác. Nguyên tắc: diversification (đa dạng hóa) giúp giảm interruption rate tổng thể vì khi pool A bị terminate hàng loạt, ASG ngay lập tức launch ở pool B, C... mà không có gap.

**Q: `capacity-optimized` và `price-capacity-optimized` khác nhau thế nào?**

> `capacity-optimized`: Chọn pool có capacity nhiều nhất, hoàn toàn không quan tâm giá → interruption rate thấp nhất nhưng giá không tối ưu nhất. `price-capacity-optimized` (khuyến nghị): Cân bằng cả hai — chọn pool có capacity nhiều VÀ giá tốt. Trong thực tế, strategy mới này thường cho kết quả tốt hơn cả hai chiều (giá và stability) vì AWS có thêm thông tin để tối ưu.

**Q: Capacity Rebalancing khác gì với việc Spot bị interrupt thông thường?**

> Thông thường, AWS chỉ báo trước **2 phút** trước khi terminate. Capacity Rebalancing hoạt động sớm hơn — khi AWS phát hiện instance "at elevated risk" (nguy cơ cao), ASG nhận được recommendation và **proactively** (chủ động) launch instance thay thế ngay. Sau khi instance mới healthy, instance cũ mới bị terminate. Kết quả: không có capacity gap, user experience liền mạch.

---

## 🔗 Điều Hướng

| Trước                                             | Tiếp Theo                                      |
| ------------------------------------------------- | ---------------------------------------------- |
| [← Scaling Policies](3-scaling-policies.md)       | [Lifecycle Hooks →](5-lifecycle-hooks.md)       |

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
