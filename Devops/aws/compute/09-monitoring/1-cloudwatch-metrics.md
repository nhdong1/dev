# 1 — CloudWatch Metrics — Chỉ Số Giám Sát AWS Compute

> CloudWatch Metrics (Chỉ Số Giám Sát) là time-series data (dữ liệu chuỗi thời gian) được AWS và ứng dụng của bạn gửi đến CloudWatch — nền tảng để tạo alarms, dashboards và kích hoạt Auto Scaling

## 📚 Mục Lục

1. [Khái Niệm Cốt Lõi](#khái-niệm-cốt-lõi)
2. [EC2 Default Metrics](#ec2-default-metrics)
3. [Lambda Metrics](#lambda-metrics)
4. [ECS Metrics](#ecs-metrics)
5. [EKS Metrics — Container Insights](#eks-metrics)
6. [Custom Metrics — Chỉ Số Tùy Chỉnh](#custom-metrics)
7. [CloudWatch Agent — Agent Giám Sát](#cloudwatch-agent)
8. [Metric Math — Tính Toán Trên Chỉ Số](#metric-math)
9. [Chi Phí Và Tối Ưu](#chi-phí-và-tối-ưu)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khái Niệm Cốt Lõi

### Metric Là Gì?

Metric (Chỉ Số) là một time-series data point gồm:

```
┌────────────────────────────────────────────────────────┐
│  METRIC STRUCTURE (Cấu Trúc Chỉ Số)                  │
├────────────────────────────────────────────────────────┤
│  Namespace  : AWS/EC2                                  │
│  MetricName : CPUUtilization                           │
│  Dimensions : {InstanceId: "i-0abc12345", ...}        │
│  Timestamp  : 2026-05-15T14:30:00Z                     │
│  Value      : 78.5                                     │
│  Unit       : Percent                                  │
└────────────────────────────────────────────────────────┘
```

### Namespace — Không Gian Tên

Namespace phân tách metrics của các dịch vụ khác nhau:

| Namespace              | Dịch Vụ               |
| ---------------------- | --------------------- |
| `AWS/EC2`              | EC2 instances         |
| `AWS/Lambda`           | Lambda functions      |
| `AWS/ECS`              | ECS clusters/services |
| `AWS/EKS`              | EKS clusters          |
| `AWS/ApplicationELB`   | Application Load Balancer |
| `CWAgent`              | CloudWatch Agent (custom) |
| `Custom/MyApp`         | Metrics tự define     |

### Dimensions — Chiều Dữ Liệu

Dimensions là key-value pairs lọc và phân nhóm metrics:

```
# Không có dimension → metric của TOÀN BỘ account
AWS/EC2 CPUUtilization

# Với dimension InstanceId → metric của MỘT instance cụ thể
AWS/EC2 CPUUtilization {InstanceId: "i-0abc12345"}

# Với dimension AutoScalingGroupName → metric của TOÀN BỘ ASG
AWS/EC2 CPUUtilization {AutoScalingGroupName: "my-asg"}
```

### Periods — Chu Kỳ Thu Thập

| Loại Metric          | Granularity (Độ Chi Tiết) | Lưu Giữ       |
| -------------------- | ------------------------- | ------------- |
| Standard (Tiêu Chuẩn) | 5 phút                  | 63 ngày       |
| Detailed (Chi Tiết)  | 1 phút                   | 63 ngày       |
| High-Resolution      | 1–60 giây                | 3 giờ         |
| Extended Retention   | 1 giờ (aggregated)        | 15 tháng      |

---

## EC2 Default Metrics

AWS tự động thu thập các metrics này **miễn phí** (5 phút/lần với Standard Monitoring):

### Metrics CPU & Network

```
┌─────────────────────┬──────────────────┬───────────────────────────────┐
│ Metric Name         │ Unit             │ Mô Tả                         │
├─────────────────────┼──────────────────┼───────────────────────────────┤
│ CPUUtilization      │ Percent          │ % CPU đang dùng               │
│ CPUCreditUsage      │ Count            │ CPU credits tiêu thụ (T types)│
│ CPUCreditBalance    │ Count            │ CPU credits còn lại (T types) │
│ NetworkIn           │ Bytes            │ Bytes nhận vào từ network     │
│ NetworkOut          │ Bytes            │ Bytes gửi ra qua network      │
│ NetworkPacketsIn    │ Count            │ Số packets nhận vào           │
│ NetworkPacketsOut   │ Count            │ Số packets gửi ra             │
└─────────────────────┴──────────────────┴───────────────────────────────┘
```

### Metrics Disk & Status

```
┌──────────────────────────┬──────────────────┬──────────────────────────────────┐
│ Metric Name              │ Unit             │ Mô Tả                            │
├──────────────────────────┼──────────────────┼──────────────────────────────────┤
│ DiskReadBytes            │ Bytes            │ Bytes đọc từ EBS/instance store   │
│ DiskWriteBytes           │ Bytes            │ Bytes ghi vào EBS/instance store  │
│ DiskReadOps              │ Count            │ Số lần đọc I/O                   │
│ DiskWriteOps             │ Count            │ Số lần ghi I/O                   │
│ StatusCheckFailed        │ Count (0 or 1)   │ 1 = có vấn đề (system hoặc instance) │
│ StatusCheckFailed_System │ Count (0 or 1)   │ AWS hardware/hypervisor failure  │
│ StatusCheckFailed_Inst.  │ Count (0 or 1)   │ OS/software failure trong VM     │
└──────────────────────────┴──────────────────┴──────────────────────────────────┘
```

### ⚠️ Metrics KHÔNG CÓ Trong Default (Cần CloudWatch Agent)

```
❌ MemoryUtilization     → RAM usage — PHẢI cài CloudWatch Agent
❌ SwapUtilization       → Swap space usage
❌ DiskSpaceUsed         → Disk capacity (khác với DiskReadBytes!)
❌ DiskSpaceUtilization  → % disk đã dùng
```

### EC2 Detailed Monitoring — Giám Sát Chi Tiết

```bash
# Bật Detailed Monitoring (1 phút/lần) — tốn phí ~$3.50/instance/tháng
aws ec2 monitor-instances --instance-ids i-0abc12345

# Tắt Detailed Monitoring
aws ec2 unmonitor-instances --instance-ids i-0abc12345
```

**Khi nào bật Detailed Monitoring?**
- Auto Scaling Group cần phản ứng nhanh (< 5 phút)
- Production workloads quan trọng
- Debugging performance issues

---

## Lambda Metrics

Lambda tự động publish metrics này vào namespace `AWS/Lambda`:

### Metrics Quan Trọng Nhất

```
┌─────────────────────┬─────────────┬────────────────────────────────────────────┐
│ Metric              │ Unit        │ Ý Nghĩa & Ngưỡng Cảnh Báo                 │
├─────────────────────┼─────────────┼────────────────────────────────────────────┤
│ Invocations         │ Count       │ Tổng số lần gọi function                   │
│ Errors              │ Count       │ Lần gọi bị lỗi (exception, timeout, OOM)   │
│ Duration            │ Milliseconds│ Thời gian thực thi — alert P99 > timeout/2 │
│ Throttles           │ Count       │ Bị throttle vì vượt concurrency limit       │
│ ConcurrentExecutions│ Count       │ Số Lambda đang chạy song song              │
│ DeadLetterErrors    │ Count       │ Lỗi ghi vào DLQ — event bị mất!           │
│ DestinationDelivery │             │ Delivery failures to async destinations    │
│   Failures          │ Count       │                                            │
│ IteratorAge         │ Milliseconds│ Lag của Kinesis/DynamoDB Streams consumer  │
│ UnreservedConcurrency│ Count      │ Concurrency từ pool chung                  │
└─────────────────────┴─────────────┴────────────────────────────────────────────┘
```

### Tính Error Rate Bằng Metric Math

```
Error Rate (%) = (Errors / Invocations) × 100

Trong CloudWatch Metric Math:
  m1 = Lambda Errors (Sum)
  m2 = Lambda Invocations (Sum)
  error_rate = (m1 / m2) * 100
```

### IteratorAge — Lag Của Stream Consumer

```
IteratorAge = thời gian từ khi record được ghi vào stream → Lambda xử lý xong

IteratorAge thấp (~100ms)   → Lambda theo kịp stream
IteratorAge cao (nhiều giây) → Lambda đang bị lag, cần scale up
IteratorAge tăng liên tục   → Không đủ concurrency để xử lý!

Alert: IteratorAge > 60,000ms (1 phút) → Cần tăng Lambda concurrency hoặc shard count
```

---

## ECS Metrics

ECS publish metrics vào namespace `AWS/ECS`:

### Cluster-Level Metrics

```
┌──────────────────────┬─────────────┬──────────────────────────────────────┐
│ Metric               │ Unit        │ Ý Nghĩa                              │
├──────────────────────┼─────────────┼──────────────────────────────────────┤
│ CPUReservation       │ Percent     │ % CPU đã được reserved bởi tasks     │
│ MemoryReservation    │ Percent     │ % Memory đã được reserved bởi tasks  │
│ CPUUtilization       │ Percent     │ % CPU đang thực sự được dùng         │
│ MemoryUtilization    │ Percent     │ % Memory đang thực sự được dùng      │
└──────────────────────┴─────────────┴──────────────────────────────────────┘

Lưu ý quan trọng:
  CPUReservation ≠ CPUUtilization
  Reservation = limit đã khai báo trong Task Definition
  Utilization = thực tế đang dùng
```

### Service-Level Metrics

```
┌──────────────────────┬─────────────┬──────────────────────────────────────┐
│ Metric               │ Unit        │ Ý Nghĩa                              │
├──────────────────────┼─────────────┼──────────────────────────────────────┤
│ RunningTaskCount     │ Count       │ Số tasks đang chạy                   │
│ PendingTaskCount     │ Count       │ Số tasks đang pending (chờ placement) │
│ CPUUtilization       │ Percent     │ CPU usage của service                │
│ MemoryUtilization    │ Percent     │ Memory usage của service             │
└──────────────────────┴─────────────┴──────────────────────────────────────┘
```

### Container Insights — Insights Chi Tiết Hơn

Container Insights cung cấp metrics cấp container (bật thêm, tốn phí):

```bash
# Bật Container Insights cho ECS cluster
aws ecs update-cluster-settings \
  --cluster my-cluster \
  --settings name=containerInsights,value=enabled
```

Metrics có thêm khi bật Container Insights:
- `container_cpu_utilized` — CPU từng container
- `container_memory_utilized` — Memory từng container
- `container_network_rx_bytes` / `container_network_tx_bytes`

---

## EKS Metrics — Container Insights

EKS không có default CloudWatch metrics — phải cài **Container Insights** với CloudWatch Agent hoặc **ADOT** (AWS Distro for OpenTelemetry — Phân Phối AWS Cho OpenTelemetry).

### Cài Container Insights Cho EKS

```bash
# Cài CloudWatch Agent cho EKS qua Helm (Trình Quản Lý Gói Kubernetes)
helm repo add aws-cloudwatch-metrics https://aws.github.io/eks-charts
helm install aws-cloudwatch-metrics \
  --set clusterName=my-cluster \
  aws-cloudwatch-metrics/aws-cloudwatch-metrics
```

### Metrics Sau Khi Bật Container Insights

```
NODE METRICS (Namespace: ContainerInsights):
  node_cpu_utilization         → % CPU của node
  node_memory_utilization      → % RAM của node
  node_filesystem_utilization  → % disk của node
  node_network_total_bytes     → Network throughput

POD METRICS:
  pod_cpu_utilized             → CPU từng pod
  pod_memory_utilized          → Memory từng pod
  pod_number_of_containers     → Số container trong pod
  pod_number_of_container_restarts → Restarts cao → vấn đề!

SERVICE METRICS:
  service_number_of_running_pods → Pod đang chạy
```

---

## Custom Metrics — Chỉ Số Tùy Chỉnh

Publish custom metrics từ ứng dụng để monitor business logic:

### Ví Dụ Business Metrics Quan Trọng

```
✓ OrdersPerMinute         → Throughput kinh doanh
✓ PaymentFailureRate      → Tỷ lệ lỗi thanh toán
✓ CartAbandonmentRate     → Tỷ lệ bỏ giỏ hàng
✓ ActiveSessions          → Người dùng đang online
✓ QueueDepth              → Độ sâu hàng đợi xử lý
✓ CacheHitRate            → Tỷ lệ cache hit
```

### Publish Custom Metric Bằng AWS CLI

```bash
aws cloudwatch put-metric-data \
  --namespace "MyApp/Business" \
  --metric-name "OrdersProcessed" \
  --value 150 \
  --unit Count \
  --dimensions Environment=prod,Service=order-service \
  --timestamp "2026-05-15T14:30:00Z"
```

### Publish Từ Python (boto3)

```python
import boto3

cloudwatch = boto3.client('cloudwatch', region_name='ap-southeast-1')

def publish_metric(metric_name: str, value: float, unit: str = 'Count'):
    cloudwatch.put_metric_data(
        Namespace='MyApp/Business',
        MetricData=[
            {
                'MetricName': metric_name,
                'Value': value,
                'Unit': unit,
                'Dimensions': [
                    {'Name': 'Environment', 'Value': 'prod'},
                    {'Name': 'Service', 'Value': 'order-service'},
                ],
            }
        ]
    )

# Sử dụng
publish_metric('OrdersProcessed', 150)
publish_metric('PaymentLatency', 245.5, 'Milliseconds')
```

### High-Resolution Custom Metrics — Độ Phân Giải Cao

```python
cloudwatch.put_metric_data(
    Namespace='MyApp/Performance',
    MetricData=[
        {
            'MetricName': 'APILatency',
            'Value': 45.2,
            'Unit': 'Milliseconds',
            'StorageResolution': 1,  # 1 giây thay vì 60 giây mặc định
        }
    ]
)
```

**Chi phí High-Resolution:** $0.02/1,000 data points (đắt hơn standard $0.01)

---

## CloudWatch Agent

CloudWatch Agent (Agent Giám Sát CloudWatch) là phần mềm cài trên EC2/ECS/EKS để thu thập metrics hệ thống không có trong default.

### Cài Đặt Trên EC2

```bash
# Tải và cài CloudWatch Agent
sudo yum install amazon-cloudwatch-agent -y

# Chạy wizard cấu hình
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# Hoặc tạo config thủ công tại:
# /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

### Cấu Hình Mẫu (JSON)

```json
{
  "metrics": {
    "namespace": "CWAgent",
    "metrics_collected": {
      "mem": {
        "measurement": [
          "mem_used_percent",
          "mem_available_percent"
        ],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": [
          "used_percent",
          "inodes_free"
        ],
        "resources": ["/", "/data"],
        "metrics_collection_interval": 60
      },
      "cpu": {
        "measurement": [
          "cpu_usage_idle",
          "cpu_usage_iowait",
          "cpu_usage_user",
          "cpu_usage_system"
        ],
        "totalcpu": true,
        "metrics_collection_interval": 60
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/application/*.log",
            "log_group_name": "/ec2/myapp/application",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
```

### Khởi Động Agent

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

---

## Metric Math — Tính Toán Trên Chỉ Số

Metric Math cho phép tính toán giữa nhiều metrics để tạo ra metrics mới:

### Ví Dụ Thực Tế

```
# Tính Lambda Error Rate
m1 = AWS/Lambda Errors (Sum, 5 phút)
m2 = AWS/Lambda Invocations (Sum, 5 phút)
e1 = IF(m2 > 0, (m1/m2)*100, 0)   # Error Rate (%)

# Tính ECS CPU % từ cluster
m3 = AWS/ECS CPUUtilization (cluster my-cluster)
m4 = AWS/ECS CPUReservation (cluster my-cluster)
e2 = m3 / m4 * 100   # % CPU thực tế / đã reserve

# Tính tổng request từ nhiều Lambda
m5 = Lambda-A Invocations
m6 = Lambda-B Invocations
e3 = SUM([m5, m6])   # Tổng invocations
```

### Các Hàm Metric Math Phổ Biến

```
SUM(metrics)         → Tổng của nhiều metrics
AVG(metrics)         → Trung bình
MIN/MAX(metrics)     → Giá trị nhỏ nhất/lớn nhất
RATE(metric)         → Tốc độ thay đổi per second
FILL(metric, value)  → Điền giá trị khi metric không có data
IF(condition, true, false) → Điều kiện
ANOMALY_DETECTION_BAND(metric, stddev) → Dải phát hiện bất thường
```

---

## Chi Phí Và Tối Ưu

### Bảng Chi Phí CloudWatch Metrics (ap-southeast-1)

| Loại                                | Chi Phí                         |
| ----------------------------------- | ------------------------------- |
| Default EC2 metrics (AWS/EC2)       | Miễn phí                        |
| Custom metrics (standard 1 phút)   | $0.30/metric/tháng (10K đầu miễn phí) |
| High-resolution metrics (1 giây)   | $0.30/metric/tháng              |
| API requests (GetMetricData)        | $0.01/1,000 requests            |
| CloudWatch Agent (additional metrics)| Tính như custom metrics        |
| Container Insights                  | $0.50/node/tháng (EKS)         |

### Chiến Lược Tiết Kiệm Chi Phí

```
1. CONSOLIDATE DIMENSIONS:
   Thay vì 10 metrics riêng lẻ → dùng 1 metric với 10 dimensions

2. APPROPRIATE RESOLUTION:
   Chỉ dùng high-resolution khi thực sự cần (1-giây precision)
   Default 60 giây thường đủ cho hầu hết use cases

3. METRIC MATH THAY VÌ CUSTOM METRICS:
   Tính error_rate bằng Metric Math từ Errors + Invocations
   → Không cần publish thêm custom metric

4. ALARM-BASED SAMPLING:
   Chỉ tăng resolution khi có alarm (dùng Lambda tự động điều chỉnh)

5. REVIEW UNUSED METRICS:
   Xóa custom metrics không còn dùng (vẫn bị tính phí!)
```

---

## Câu Hỏi Phỏng Vấn

**Q: Tại sao EC2 không có Memory metric mặc định?**

> Vì AWS không thể "nhìn vào" bên trong guest OS để đọc memory — họ chỉ thấy hypervisor level. Memory là metric của OS, không phải hypervisor. Cần cài CloudWatch Agent trong OS để gửi memory data lên CloudWatch.

**Q: Sự khác biệt giữa CPUReservation và CPUUtilization trong ECS?**

> CPUReservation là % CPU đã được "khai báo" bởi tất cả Task Definitions (soft limit). CPUUtilization là % CPU thực sự đang được dùng. Một cluster có thể có CPUReservation = 90% nhưng CPUUtilization = 30% nếu tasks overprovisioned.

**Q: IteratorAge metric trong Lambda dùng để làm gì?**

> IteratorAge đo độ trễ giữa lúc record được ghi vào Kinesis/DynamoDB Streams và lúc Lambda xử lý xong. Nếu IteratorAge tăng liên tục → Lambda không đủ concurrency để theo kịp stream, cần tăng số shards hoặc Lambda concurrency.

**Q: Khi nào dùng Metric Math thay vì publish custom metric?**

> Dùng Metric Math khi metric mới có thể được tính từ các metrics đã có (vd: error_rate = errors/invocations). Tiết kiệm chi phí vì không phải publish thêm data points. Dùng custom metric khi cần publish business data không thể suy diễn từ infrastructure metrics (vd: orders_per_minute).

**Q: Detailed Monitoring có ý nghĩa gì với Auto Scaling?**

> Với Standard Monitoring (5 phút), Auto Scaling dùng data cũ 5 phút → phản ứng chậm. Với Detailed Monitoring (1 phút), ASG có data mới hơn → scale nhanh hơn trong traffic spikes. Đặc biệt quan trọng với Target Tracking Policy cần phản ứng nhanh.

---

**← Trước:** [README.md](./README.md) | **Tiếp theo →** [2-cloudwatch-logs.md](./2-cloudwatch-logs.md)

**Cập Nhật Lần Cuối:** 2026-05-15 | **Phiên Bản:** 1.0
