# Capacity Planning — Lập Kế Hoạch Năng Lực

> Capacity Planning (Lập Kế Hoạch Năng Lực) là quá trình đảm bảo hệ thống có đủ tài nguyên để xử lý tải hiện tại và tương lai. Không đủ capacity → downtime. Dư thừa quá nhiều → lãng phí chi phí. Mục tiêu: đúng lượng tài nguyên, đúng thời điểm.

## 📚 Mục Lục

1. [Tại Sao Capacity Planning Quan Trọng](#tại-sao-quan-trọng)
2. [Load Testing — Kiểm Thử Tải](#load-testing)
3. [Capacity Forecasting — Dự Báo Năng Lực](#forecasting)
4. [Pre-scaling — Mở Rộng Trước](#pre-scaling)
5. [Bottleneck Analysis — Phân Tích Nút Thắt](#bottleneck)
6. [AWS Compute Optimizer](#compute-optimizer)
7. [Rightsizing Strategy](#rightsizing)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Tại Sao Capacity Planning Quan Trọng {#tại-sao-quan-trọng}

### Chi Phí Của Lập Kế Hoạch Sai

| Vấn Đề                      | Hậu Quả                                           |
| --------------------------- | ------------------------------------------------- |
| Underprovisioning (thiếu)   | Downtime, latency tăng, mất khách hàng            |
| Overprovisioning (dư thừa)  | Lãng phí 30-70% chi phí cloud (rất phổ biến)     |
| Scale quá chậm              | Spike traffic gây chậm/crash trước khi scale kịp  |
| Scale quá nhanh             | Terminate instances đang có active connections     |

### Capacity Planning Process (Quy Trình Lập Kế Hoạch)

```
1. Baseline (Đo Lường Cơ Sở)
   └── Đo performance hiện tại: CPU, memory, latency, throughput

2. Load Testing (Kiểm Thử Tải)
   └── Tìm điểm giới hạn của hệ thống

3. Forecasting (Dự Báo)
   └── Dự đoán growth và traffic patterns

4. Rightsizing (Chọn Đúng Kích Thước)
   └── Chọn instance type và count phù hợp

5. Pre-scaling (Mở Rộng Trước)
   └── Cấu hình auto scaling và warm pools

6. Monitoring & Review (Giám Sát & Xem Xét)
   └── Theo dõi thực tế và điều chỉnh định kỳ
```

---

## 🔨 Load Testing — Kiểm Thử Tải {#load-testing}

### Các Loại Load Test

#### Smoke Test (Kiểm Thử Khói)

Xác nhận hệ thống hoạt động với tải tối thiểu. Chạy trước mỗi deploy.

```
Tải: 1-5% production traffic
Thời gian: 5-10 phút
Mục tiêu: Không có lỗi cơ bản, app đang chạy
```

#### Load Test (Kiểm Thử Tải Thông Thường)

Mô phỏng traffic dự kiến trong điều kiện bình thường và cao điểm.

```
Tải: 100-120% expected production traffic
Thời gian: 30-60 phút (đủ để auto scaling ổn định)
Mục tiêu:
  - Response time P95 < 500ms
  - Error rate < 0.1%
  - CPU < 70% khi stable
```

#### Stress Test (Kiểm Thử Áp Lực)

Tìm điểm gãy — hệ thống bắt đầu lỗi ở mức tải nào.

```
Tải: Tăng dần từ 100% đến 200-300% expected load
Thời gian: Đến khi hệ thống fail hoặc đạt giới hạn
Mục tiêu:
  - Tìm throughput tối đa
  - Xác nhận hệ thống fail gracefully (không crash toàn bộ)
  - Verify auto scaling hoạt động đúng
```

#### Spike Test (Kiểm Thử Đột Biến)

Mô phỏng traffic tăng đột ngột không báo trước.

```
Tải: 0% → 300% trong vài giây
Thời gian: 30-60 giây spike, sau đó giảm
Mục tiêu:
  - Auto scaling đủ nhanh không?
  - Error rate spike < 1%?
  - System recover sau spike không?
```

#### Soak Test / Endurance Test (Kiểm Thử Ngâm)

Phát hiện memory leaks, connection pool exhaustion, disk fill-up.

```
Tải: 80% production traffic
Thời gian: 8-24 giờ
Mục tiêu:
  - Memory usage không tăng liên tục
  - Connection count ổn định
  - Không có gradual degradation
```

### Công Cụ Load Testing

| Công Cụ             | Loại          | Tốt Cho                                           |
| ------------------- | ------------- | ------------------------------------------------- |
| **k6**              | OSS           | Developer-friendly, scripting bằng JavaScript     |
| **Apache JMeter**   | OSS           | Phức tạp, enterprise features, GUI                |
| **Locust**          | OSS           | Python scripting, distributed testing             |
| **Artillery**       | OSS           | YAML-based, Node.js                               |
| **AWS Load Testing**| Managed       | Distributed Jmeter trên Fargate, tích hợp AWS     |
| **Gatling**         | OSS/Enterprise| High performance, Scala DSL                       |

### k6 Script Ví Dụ

```javascript
// k6 load test script
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Tăng lên 100 users trong 2 phút
    { duration: '5m', target: 100 },   // Giữ 100 users trong 5 phút
    { duration: '2m', target: 200 },   // Spike lên 200 users
    { duration: '5m', target: 200 },   // Giữ 200 users
    { duration: '2m', target: 0 },     // Giảm xuống 0
  ],
  thresholds: {
    'http_req_duration': ['p(95)<500'],  // 95% requests < 500ms
    'http_req_failed': ['rate<0.01'],    // Error rate < 1%
  },
};

export default function () {
  const response = http.get('https://api.example.com/products');
  
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  sleep(1);
}
```

### CloudWatch Metrics Cần Theo Dõi Khi Test

```
EC2:
├── CPUUtilization                  → < 70% stable
├── NetworkIn/NetworkOut            → Bandwidth limits
└── StatusCheckFailed               → 0

ELB:
├── RequestCount                    → Throughput
├── TargetResponseTime P95          → < 500ms
├── HTTPCode_ELB_5XX_Count          → ~0
└── UnHealthyHostCount              → 0

ASG:
├── GroupDesiredCapacity            → Scale đúng không?
├── GroupInServiceInstances         → Healthy instances
└── WarmPoolMinSize                 → Warm pool hoạt động

Application:
├── Custom latency metric           → P50/P95/P99
└── Error rate                      → < 0.1%
```

---

## 📈 Capacity Forecasting — Dự Báo Năng Lực {#forecasting}

### Phân Tích Traffic Patterns (Mô Hình Traffic)

```
Trích xuất patterns từ CloudWatch Logs Insights:

fields @timestamp, @message
| filter @message like /GET \/api/
| stats count() as requests by bin(1h)
| sort @timestamp asc
```

**Các Pattern Cần Nhận Diện:**

```
Daily Pattern (Mẫu Hàng Ngày):
  00:00─02:00: 10% peak (traffic đêm thấp)
  08:00─10:00: 60% peak (bắt đầu ngày làm việc)
  12:00─14:00: 80% peak (trưa cao điểm)
  17:00─19:00: 100% peak (tan làm, mua sắm online)
  22:00─00:00: 40% peak (giải trí tối)

Weekly Pattern (Mẫu Hàng Tuần):
  Mon-Fri: 100% baseline
  Sat: 70% baseline
  Sun: 50% baseline

Seasonal Pattern (Mẫu Theo Mùa):
  Tháng 11-12: 200-400% (mùa mua sắm)
  Tháng 1-2: 60-70% (sau Tết)
```

### Growth Rate Calculation (Tính Tốc Độ Tăng Trưởng)

```python
# Ước tính capacity cần thiết trong 6 tháng tới
def forecast_capacity(current_traffic, monthly_growth_rate, months):
    """
    current_traffic: Lượng requests/giây hiện tại
    monthly_growth_rate: 0.15 = 15% tăng mỗi tháng
    months: Số tháng dự báo
    """
    forecast = current_traffic * (1 + monthly_growth_rate) ** months
    
    # Thêm safety buffer (bộ đệm an toàn) 30%
    safe_forecast = forecast * 1.3
    
    # Tính số instances cần (mỗi instance xử lý 100 req/s)
    instances_needed = math.ceil(safe_forecast / 100)
    
    return {
        "month": months,
        "forecast_rps": round(forecast),
        "safe_forecast_rps": round(safe_forecast),
        "instances_needed": instances_needed
    }

# Ví dụ: 1000 req/s hiện tại, tăng 15%/tháng
for m in [1, 3, 6, 12]:
    print(forecast_capacity(1000, 0.15, m))
```

### Little's Law — Định Luật Little

Công thức cơ bản cho capacity planning:

```
L = λ × W

L = Số requests đang được xử lý (concurrency)
λ = Throughput (requests per second)
W = Average response time (giây)

Ví dụ:
  1000 req/s × 0.5 giây/request = 500 concurrent requests

Nếu mỗi instance xử lý 50 concurrent → cần 10 instances
```

---

## ⚡ Pre-scaling — Mở Rộng Trước {#pre-scaling}

### Khi Nào Cần Pre-scaling

- Sự kiện marketing đã được lên kế hoạch (flash sale, campaign)
- Ra mắt tính năng mới với PR lớn
- Thời điểm cao điểm theo mùa (11/11, Black Friday, Tết)
- Sau khi viral trên mạng xã hội (dự đoán)

### Warm Pool (Nhóm Khởi Động Sẵn)

Warm Pool giữ instances đã bootstrapped nhưng chưa nhận traffic — sẵn sàng đưa vào phục vụ ngay.

```bash
aws autoscaling put-warm-pool \
  --auto-scaling-group-name my-asg \
  --min-size 5 \
  --pool-state Stopped    # Stopped (tiết kiệm ~70% chi phí) hoặc Running

# Instance lifecycle trong Warm Pool:
# Pending → Warmed:Pending → Warmed:Running → InService
```

**Warm Pool States:**

| State     | Chi Phí               | Thời Gian Chuyển Sang InService |
| --------- | --------------------- | -------------------------------- |
| `Running` | Full EC2 cost         | < 30 giây                        |
| `Stopped` | Chỉ trả EBS (70% rẻ) | 1-2 phút (cần start)             |
| `Hibernated` | Minimal          | ~1 phút (resume từ hibernate)    |

### Scheduled Pre-scaling cho Sự Kiện

```bash
# Flash sale 11/11 - scale up 2 giờ trước
# 10pm 10/11 ICT = 15:00 UTC 10/11
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name prod-asg \
  --scheduled-action-name 11-11-flash-sale \
  --start-time "2026-11-10T15:00:00Z" \
  --min-size 50 \
  --max-size 200 \
  --desired-capacity 80

# Sau flash sale - scale down
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name prod-asg \
  --scheduled-action-name 11-11-end \
  --start-time "2026-11-11T17:00:00Z" \  # Midnight ICT
  --min-size 10 \
  --max-size 50 \
  --desired-capacity 15
```

### Lambda Provisioned Concurrency Pre-warming

```bash
# Trước sự kiện lớn, provision concurrency cho Lambda
aws lambda put-provisioned-concurrency-config \
  --function-name my-api-function \
  --qualifier production \
  --provisioned-concurrent-executions 100

# Tạo Application Auto Scaling cho Provisioned Concurrency
aws application-autoscaling register-scalable-target \
  --service-namespace lambda \
  --resource-id function:my-api-function:production \
  --scalable-dimension lambda:function:ProvisionedConcurrency \
  --min-capacity 10 \
  --max-capacity 200
```

---

## 🔍 Bottleneck Analysis — Phân Tích Nút Thắt {#bottleneck}

### USE Method (Phương Pháp USE)

USE = **U**tilization + **S**aturation + **E**rrors

```
Với mỗi resource (CPU, Memory, Network, Disk):

Utilization (Tỷ Lệ Sử Dụng):
  CPU: < 70% stable, < 85% peak
  Memory: < 80%
  Network: < 70% bandwidth

Saturation (Bão Hòa — Queue đang build up):
  CPU run queue > CPU count → bão hòa
  Memory swap usage > 0 → bão hòa
  Network drops/retransmits → bão hòa

Errors (Lỗi):
  HTTP 5xx rate
  Connection refused
  Disk I/O errors
```

### Bottleneck Identification Checklist

```
Step 1: Kiểm tra ELB metrics
  □ TargetResponseTime tăng?
  □ 5XX error tăng?
  □ ActiveConnectionCount tăng đột biến?

Step 2: Kiểm tra EC2 metrics
  □ CPUUtilization > 85%? → Scale out hoặc upgrade
  □ Memory cao? → Upgrade instance type
  □ Network throughput giới hạn? → Instance có Enhanced Networking?

Step 3: Kiểm tra RDS metrics
  □ DBLoad > số CPU? → Query optimization hoặc scale up
  □ ReadIOPS / WriteIOPS cao? → IOPS provisioned đủ chưa?
  □ FreeStorageSpace thấp? → Tăng storage
  □ ConnectionCount cao? → RDS Proxy, connection pooling

Step 4: Kiểm tra ứng dụng
  □ GC (Garbage Collection) pause lâu? (JVM)
  □ Thread pool exhaustion?
  □ External API calls chậm?
  □ N+1 query problem?
```

### CloudWatch Container Insights cho ECS/EKS

```bash
# Enable Container Insights
aws ecs update-cluster-settings \
  --cluster my-cluster \
  --settings name=containerInsights,value=enabled

# Key metrics để bottleneck analysis:
# CpuUtilized / CpuReserved    → CPU efficiency
# MemoryUtilized / MemoryReserved → Memory efficiency
# NetworkRxBytes / NetworkTxBytes → Network usage
# StorageReadBytes              → I/O patterns
```

---

## 🤖 AWS Compute Optimizer {#compute-optimizer}

AWS Compute Optimizer (Bộ Tối Ưu Tính Toán AWS) dùng Machine Learning để phân tích CloudWatch metrics và đề xuất instance type phù hợp hơn.

### Bật Compute Optimizer

```bash
# Opt-in account
aws compute-optimizer update-enrollment-status \
  --status Active

# Xem recommendations sau 12-24 giờ phân tích
aws compute-optimizer get-ec2-instance-recommendations \
  --filters '[{"name":"Finding","values":["OVER_PROVISIONED","UNDER_PROVISIONED"]}]'
```

### Giải Thích Findings (Kết Quả Phân Tích)

| Finding               | Ý Nghĩa                                  | Hành Động                        |
| --------------------- | ---------------------------------------- | --------------------------------- |
| `OVER_PROVISIONED`    | Instance lớn hơn cần thiết              | Downgrade để tiết kiệm chi phí   |
| `UNDER_PROVISIONED`   | Instance quá nhỏ, đang bị nghẽn         | Upgrade để cải thiện performance  |
| `OPTIMIZED`           | Instance đang phù hợp                   | Không cần thay đổi               |
| `NOT_OPTIMIZED`       | Không đủ data (instance mới)            | Chờ thêm data                    |

### Compute Optimizer cho ECS on Fargate

```bash
# Recommendations cho Fargate task CPU/Memory
aws compute-optimizer get-ecs-service-recommendations \
  --service-arns arn:aws:ecs:ap-southeast-1:123:service/cluster/my-service
```

---

## 📐 Rightsizing Strategy {#rightsizing}

### Quy Trình Rightsizing (Chọn Đúng Kích Cỡ)

```
1. Thu Thập Data (2-4 tuần)
   CloudWatch metrics: CPU, Memory, Network, Disk

2. Phân Tích
   - Peak usage ở mức nào?
   - Average utilization ra sao?
   - Có burst ngắn hay sustained high load?

3. Chọn Instance Type Mới
   Quy tắc đơn giản:
   - CPU < 20% average → Downsize
   - CPU > 70% thường xuyên → Upsize
   - Memory > 80% → Upsize
   - Burst CPU thường xuyên → Xem xét Graviton hoặc burstable

4. Test Trước Khi Apply
   - Deploy một số instances mới
   - Monitor trong 1-2 tuần
   - So sánh cost và performance

5. Apply Dần Dần
   - Rolling replacement, không thay tất cả cùng lúc
```

### Savings Estimate (Ước Tính Tiết Kiệm)

```python
# Ước tính savings khi rightsizing
current_instance = "m5.2xlarge"   # $0.384/h, 8 vCPU, 32GB RAM
recommended = "m5.xlarge"         # $0.192/h, 4 vCPU, 16GB RAM

hours_per_month = 720
instances_count = 10

current_cost = 0.384 * hours_per_month * instances_count  # $2,764.80
new_cost = 0.192 * hours_per_month * instances_count      # $1,382.40
monthly_savings = current_cost - new_cost                 # $1,382.40

print(f"Monthly savings: ${monthly_savings:,.2f}")        # $1,382.40
print(f"Annual savings: ${monthly_savings * 12:,.2f}")    # $16,588.80
```

---

## 📊 Capacity Planning Dashboard

### CloudWatch Dashboard cho Capacity

```bash
aws cloudwatch put-dashboard \
  --dashboard-name capacity-planning \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "title": "EC2 CPU Utilization Fleet",
        "properties": {
          "metrics": [["AWS/EC2", "CPUUtilization", "AutoScalingGroupName", "prod-asg"]],
          "stat": "Average",
          "period": 300,
          "view": "timeSeries"
        }
      },
      {
        "type": "metric",
        "title": "ASG Instance Count",
        "properties": {
          "metrics": [
            ["AWS/AutoScaling", "GroupDesiredCapacity", "AutoScalingGroupName", "prod-asg"],
            ["AWS/AutoScaling", "GroupInServiceInstances", "AutoScalingGroupName", "prod-asg"]
          ]
        }
      }
    ]
  }'
```

---

## ❓ Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa Load Testing và Stress Testing?**
> Load Testing: Mô phỏng traffic dự kiến, verify hệ thống đáp ứng SLA. Stress Testing: Tăng tải vượt quá dự kiến để tìm điểm gãy và verify system fails gracefully. Load test kiểm tra "bình thường hoạt động đúng không", stress test tìm giới hạn.

**Q: Warm Pool khác gì Provisioned Concurrency của Lambda?**
> Warm Pool (EC2): Instances đã bootstrap, đứng chờ ở trạng thái Stopped/Running — sẵn sàng đưa vào ASG nhanh. Provisioned Concurrency (Lambda): Execution environments đã khởi tạo, luôn ở trạng thái sẵn sàng — loại bỏ cold start. Cả hai đều "pre-warm" nhưng cho compute khác nhau.

**Q: Khi nào nên dùng Compute Optimizer thay vì tự rightsizing?**
> Compute Optimizer tốt hơn khi có fleet lớn (hàng chục đến hàng trăm instances) — khó theo dõi thủ công. Tự rightsizing phù hợp khi số lượng instances nhỏ và bạn hiểu rõ workload pattern. Compute Optimizer tự động phát hiện patterns từ 14 ngày data CloudWatch.

**Q: Little's Law áp dụng vào capacity planning thế nào?**
> L = λ × W: Số concurrent requests = throughput × response time. Ví dụ: 1000 req/s × 0.2 giây = 200 concurrent. Nếu mỗi instance chịu 50 concurrent → cần 4 instances tối thiểu. Khi response time tăng (vì overload), cần nhiều instances hơn để xử lý cùng throughput.

---

## 🎯 Capacity Planning Checklist

```
Trước Sự Kiện Lớn (1-2 tuần trước):
□ Chạy Load Test với tải 200% dự kiến
□ Xác nhận Auto Scaling hoạt động đúng
□ Cấu hình Warm Pool hoặc tăng min capacity
□ Enable Predictive Scaling (nếu có pattern)
□ Tạo Scheduled Scaling Action cho sự kiện
□ Test rollback procedure

Hàng Tháng:
□ Review Compute Optimizer recommendations
□ Kiểm tra CPU/Memory utilization trends
□ So sánh actual vs forecast traffic
□ Điều chỉnh Auto Scaling policies nếu cần
□ Review RDS metrics (DBLoad, connections)

Hàng Quý:
□ Chạy Stress Test để validate limits
□ Review và cập nhật capacity forecast
□ Rightsizing review cho các instances không hiệu quả
□ Benchmark với instance generation mới hơn
```

---

**Trở Về:** [README.md](./README.md) — Tổng Quan High Availability
