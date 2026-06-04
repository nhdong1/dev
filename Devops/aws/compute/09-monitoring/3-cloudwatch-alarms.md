# 3 — CloudWatch Alarms — Hệ Thống Cảnh Báo Tự Động

> CloudWatch Alarms (Cảnh Báo CloudWatch) theo dõi metrics và kích hoạt hành động tự động khi vượt ngưỡng — từ gửi notification đến trigger Auto Scaling hay khôi phục EC2

## 📚 Mục Lục

1. [Alarm States và Cách Hoạt Động](#alarm-states)
2. [Tạo Alarm Cơ Bản](#tạo-alarm-cơ-bản)
3. [Alarm Types — Các Loại Cảnh Báo](#alarm-types)
4. [Composite Alarms — Cảnh Báo Tổng Hợp](#composite-alarms)
5. [Alarm Actions — Hành Động Tự Động](#alarm-actions)
6. [Anomaly Detection — Phát Hiện Bất Thường](#anomaly-detection)
7. [Alarm Best Practices](#alarm-best-practices)
8. [Alarm Patterns Theo Dịch Vụ](#alarm-patterns-theo-dịch-vụ)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Alarm States

CloudWatch Alarm có **3 trạng thái**:

```
┌──────────────────────────────────────────────────────────────────┐
│                      ALARM STATE MACHINE                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────┐                                        │
│   │   INSUFFICIENT_DATA │ ← Trạng thái ban đầu khi vừa tạo     │
│   │  (Không Đủ Dữ Liệu) │   hoặc metric không có data          │
│   └──────────┬──────────┘                                        │
│              │ Data bắt đầu xuất hiện                           │
│              ▼                                                   │
│   ┌─────────────────────┐      Metric vượt threshold            │
│   │        OK           │ ──────────────────────────────────→   │
│   │   (Bình Thường)     │                                        │
│   │   ✅ Green          │ ←──────────────────────────────────   │
│   └─────────────────────┘      Metric quay về ngưỡng an toàn   │
│                                                                  │
│   ┌─────────────────────┐                                        │
│   │       ALARM         │ ← Metric vi phạm threshold đủ lâu    │
│   │   (Cảnh Báo)        │   → Trigger actions (SNS, scaling...) │
│   │   🔴 Red            │                                        │
│   └─────────────────────┘                                        │
└──────────────────────────────────────────────────────────────────┘
```

### Evaluation Period — Kỳ Đánh Giá

Alarm không kích hoạt ngay khi metric vượt ngưỡng — phải vi phạm đủ số lần:

```
Configuration:
  Period      = 60 giây (mỗi data point = 1 phút)
  Evaluation  = 3 periods (xét 3 data points liên tiếp)
  Datapoints  = 2 out of 3 (2/3 data points phải vượt ngưỡng)
  Threshold   = CPU > 80%

Timeline:
  T+0  : CPU = 85%  → vi phạm 1/3
  T+1m : CPU = 83%  → vi phạm 2/3 → ALARM kích hoạt! (2/3 thỏa điều kiện)
  T+2m : CPU = 78%  → không vi phạm

Điều này tránh "false positive" do CPU spike ngắn.
```

---

## Tạo Alarm Cơ Bản

### Bằng AWS CLI

```bash
# Alarm khi EC2 CPU > 80% trong 2/3 lần kiểm tra (mỗi lần 5 phút)
aws cloudwatch put-metric-alarm \
  --alarm-name "EC2-High-CPU-i-0abc12345" \
  --alarm-description "CPU utilization > 80% on production EC2" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 3 \
  --datapoints-to-alarm 2 \
  --threshold 80 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --dimensions Name=InstanceId,Value=i-0abc12345 \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789012:ops-alerts \
  --ok-actions arn:aws:sns:ap-southeast-1:123456789012:ops-alerts \
  --treat-missing-data notBreaching
```

### Tham Số Quan Trọng

| Tham Số                     | Ý Nghĩa                                         | Giá Trị Phổ Biến              |
| --------------------------- | ----------------------------------------------- | ----------------------------- |
| `period`                    | Số giây mỗi data point                          | 60, 300, 3600                 |
| `evaluation-periods`        | Xét bao nhiêu data points                       | 1-5                           |
| `datapoints-to-alarm`       | Bao nhiêu data points phải vi phạm              | >= 1, <= evaluation-periods   |
| `statistic`                 | Cách tính giá trị                               | Average, Sum, Maximum, Minimum|
| `comparison-operator`       | Phép so sánh                                    | GreaterThan, LessThan...      |
| `treat-missing-data`        | Xử lý khi không có data                         | notBreaching, breaching, ignore|

### treat-missing-data Options

```
notBreaching  → Missing data = OK (không vi phạm) — dùng cho metrics tùy chọn
breaching     → Missing data = ALARM — dùng khi metric PHẢI có (nếu không có = vấn đề)
ignore        → Giữ nguyên state hiện tại
missing       → Chuyển sang INSUFFICIENT_DATA
```

**Ví dụ:** Lambda Errors với `notBreaching` — khi không có invocations → không có errors → không alert

---

## Alarm Types — Các Loại Cảnh Báo

### 1. Static Threshold Alarm — Cảnh Báo Ngưỡng Tĩnh

Alarm đơn giản nhất: so sánh metric với con số cố định.

```bash
# Lambda error rate > 5%
aws cloudwatch put-metric-alarm \
  --alarm-name "Lambda-High-Error-Rate" \
  --metrics '[
    {
      "Id": "e1",
      "Expression": "(m1/m2)*100",
      "Label": "ErrorRate"
    },
    {
      "Id": "m1",
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/Lambda",
          "MetricName": "Errors",
          "Dimensions": [{"Name": "FunctionName", "Value": "order-handler"}]
        },
        "Period": 300,
        "Stat": "Sum"
      }
    },
    {
      "Id": "m2",
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/Lambda",
          "MetricName": "Invocations",
          "Dimensions": [{"Name": "FunctionName", "Value": "order-handler"}]
        },
        "Period": 300,
        "Stat": "Sum"
      }
    }
  ]' \
  --comparison-operator GreaterThanThreshold \
  --threshold 5 \
  --evaluation-periods 2 \
  --datapoints-to-alarm 1 \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789012:ops-alerts
```

### 2. Anomaly Detection Alarm — Cảnh Báo Phát Hiện Bất Thường

Xem chi tiết ở phần [Anomaly Detection](#anomaly-detection) bên dưới.

### 3. Metric Math Alarm — Cảnh Báo Dựa Trên Tính Toán

Dùng Metric Math expressions để tạo alarm từ nhiều metrics:

```bash
# Alarm khi tổng error rate của 3 Lambda functions > 2%
# Dùng SUM để aggregate, rồi so sánh
```

---

## Composite Alarms — Cảnh Báo Tổng Hợp

Composite Alarm (Cảnh Báo Tổng Hợp) kết hợp nhiều alarms với logic AND/OR để giảm false positives và tạo alert hierarchy.

### Ví Dụ: Alarm Chỉ Khi CPU Cao VÀ Error Rate Cao

```bash
# Bước 1: Tạo alarm con
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPU" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=InstanceId,Value=i-0abc12345

aws cloudwatch put-metric-alarm \
  --alarm-name "HighErrorRate" \
  --metric-name Errors \
  --namespace AWS/Lambda \
  --statistic Sum \
  --period 300 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=FunctionName,Value=order-handler

# Bước 2: Tạo Composite Alarm từ 2 alarms trên
aws cloudwatch put-composite-alarm \
  --alarm-name "ProductionCritical" \
  --alarm-description "High CPU AND high error rate — likely production issue" \
  --alarm-rule "ALARM(HighCPU) AND ALARM(HighErrorRate)" \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789012:pagerduty-critical
```

### Ví Dụ: Multi-Region Health Check

```bash
# Alert khi BẤT KỲ region nào có vấn đề
aws cloudwatch put-composite-alarm \
  --alarm-name "GlobalServiceHealth" \
  --alarm-rule "ALARM(HealthCheck-US-East) OR ALARM(HealthCheck-AP-SE) OR ALARM(HealthCheck-EU-West)" \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:global-alerts
```

### Suppression — Tắt Cảnh Báo Có Kiểm Soát

```bash
# Tắt alert trong maintenance window (cửa sổ bảo trì)
# Dùng Composite Alarm với ALARM(MaintenanceMode) OR ALARM(ServiceIssue)
# Khi bật MaintenanceMode alarm thủ công → composite không fire

aws cloudwatch set-alarm-state \
  --alarm-name "MaintenanceMode" \
  --state-value ALARM \
  --state-reason "Planned maintenance 14:00-16:00"
```

---

## Alarm Actions — Hành Động Tự Động

### 1. SNS Notification — Thông Báo

```bash
# Gửi email/SMS/webhook qua SNS
--alarm-actions arn:aws:sns:ap-southeast-1:123456789012:ops-alerts

# SNS có thể forward đến:
# - Email (đăng ký subscriber)
# - SMS (số điện thoại)
# - HTTP/HTTPS webhook (Slack, PagerDuty, OpsGenie)
# - Lambda (xử lý tùy chỉnh)
# - SQS (message queue)
```

### 2. Auto Scaling Action — Tự Động Co Giãn

```bash
# Scale out khi CPU > 70%
aws cloudwatch put-metric-alarm \
  --alarm-name "ASG-Scale-Out" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 70 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 2 \
  --dimensions Name=AutoScalingGroupName,Value=my-asg \
  --alarm-actions arn:aws:autoscaling:ap-southeast-1:123456789012:scalingPolicy:abc123:autoScalingGroupName/my-asg:policyName/scale-out-policy
```

### 3. EC2 Actions — Hành Động Trên EC2

```bash
# Auto-recover EC2 khi Status Check Failed
aws cloudwatch put-metric-alarm \
  --alarm-name "EC2-Auto-Recover-i-0abc12345" \
  --metric-name StatusCheckFailed_System \
  --namespace AWS/EC2 \
  --statistic Maximum \
  --period 60 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 2 \
  --dimensions Name=InstanceId,Value=i-0abc12345 \
  --alarm-actions "arn:aws:automate:ap-southeast-1:ec2:recover"
  # Các actions EC2 khác:
  # arn:aws:automate:{region}:ec2:stop
  # arn:aws:automate:{region}:ec2:terminate
  # arn:aws:automate:{region}:ec2:reboot
```

### 4. Lambda Action — Trigger Lambda

```bash
# Trigger Lambda để tự động khắc phục sự cố
--alarm-actions arn:aws:lambda:ap-southeast-1:123456789012:function:auto-remediate
```

---

## Anomaly Detection — Phát Hiện Bất Thường

CloudWatch Anomaly Detection (Phát Hiện Bất Thường) dùng Machine Learning để học pattern bình thường của metric và alert khi có bất thường.

### Cách Hoạt Động

```
┌───────────────────────────────────────────────────────────┐
│  ANOMALY DETECTION BAND                                   │
│                                                           │
│   Value                                                   │
│   │        ╭──────────── Upper Band                      │
│   │    ╭───╯   Normal                                     │
│   │   ╱       Range                                       │
│   │  │     ╮                                              │
│   │  │     ╰──────────── Lower Band                      │
│   │                                                       │
│   │              ↗ ANOMALY! (vượt band)                  │
│   │             ╱                                         │
│   └─────────────────────────────────────────────Time      │
│                                                           │
│  ML model học từ:                                         │
│  - Trend (xu hướng)                                       │
│  - Seasonality (tính mùa vụ — cao thứ 2, thấp CN)        │
│  - Time-of-day patterns (giờ cao điểm)                    │
└───────────────────────────────────────────────────────────┘
```

### Tạo Anomaly Detection Alarm

```bash
# Bước 1: Tạo anomaly detector
aws cloudwatch put-anomaly-detector \
  --namespace AWS/Lambda \
  --metric-name Duration \
  --dimensions Name=FunctionName,Value=order-handler \
  --stat Average

# Bước 2: Tạo alarm dựa trên anomaly band
aws cloudwatch put-metric-alarm \
  --alarm-name "Lambda-Duration-Anomaly" \
  --metrics '[
    {
      "Id": "m1",
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/Lambda",
          "MetricName": "Duration",
          "Dimensions": [{"Name": "FunctionName", "Value": "order-handler"}]
        },
        "Period": 300,
        "Stat": "Average"
      }
    },
    {
      "Id": "ad1",
      "Expression": "ANOMALY_DETECTION_BAND(m1, 2)"
    }
  ]' \
  --comparison-operator GreaterThanUpperThreshold \
  --threshold-metric-id ad1 \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789012:ops-alerts
```

**Số `2` trong `ANOMALY_DETECTION_BAND(m1, 2)` là standard deviations** — band rộng hơn = ít sensitive hơn.

---

## Alarm Best Practices

### 1. Tránh Alert Fatigue (Mệt Mỏi Do Cảnh Báo Quá Nhiều)

```
❌ SAI: Alert ngay khi metric vượt ngưỡng 1 lần
✅ ĐÚNG: evaluation_periods=3, datapoints_to_alarm=2
         → Phải vi phạm 2/3 lần mới alert

❌ SAI: Alert mọi thứ (CPU > 50%)
✅ ĐÚNG: Chỉ alert những gì cần hành động ngay

❌ SAI: Ngưỡng quá thấp (CPU > 30% alert)
✅ ĐÚNG: Ngưỡng dựa trên baseline thực tế
```

### 2. Alert On Symptoms, Not Causes

```
❌ Alert: "CPU > 80%"               (cause — nguyên nhân)
✅ Alert: "HTTP 5xx Error Rate > 1%" (symptom — triệu chứng người dùng gặp)
✅ Alert: "P99 Latency > 2 giây"    (symptom — trải nghiệm người dùng)
✅ Alert: "Payment failure > 5/phút" (business impact)
```

### 3. Runbook Links Trong Alarm Description

```bash
--alarm-description "High CPU on production EC2.
Runbook: https://wiki.company.com/runbooks/ec2-high-cpu
On-call: #ops-team-slack
Escalate after 15 min: @on-call-manager"
```

### 4. Phân Cấp Alarm (Alarm Hierarchy)

```
TIER 1 — P1 (Immediate)     : Production down, SLA breach
TIER 2 — P2 (Urgent)        : Degraded performance, error spike
TIER 3 — P3 (Warning)       : Approaching capacity, trend alerts
TIER 4 — P4 (Informational) : Daily summary, capacity planning
```

### 5. OK Actions — Thông Báo Khi Hết Sự Cố

```bash
# Quan trọng: bổ sung ok-actions để team biết sự cố đã resolved
--alarm-actions arn:aws:sns:...:ops-pagerduty \   # fire khi ALARM
--ok-actions arn:aws:sns:...:ops-notifications     # fire khi OK (sự cố kết thúc)
```

---

## Alarm Patterns Theo Dịch Vụ

### EC2 — Danh Sách Alarm Cần Có

```bash
# ✅ 1. CPU cao
MetricName: CPUUtilization, Threshold: >= 80, Period: 300, DP: 2/3

# ✅ 2. Status check failed → auto-recover
MetricName: StatusCheckFailed_System, Threshold: >= 1, Action: ec2:recover

# ✅ 3. Instance status check (OS level)
MetricName: StatusCheckFailed_Instance, Threshold: >= 1

# ✅ 4. Network anomaly (nếu có baseline)
MetricName: NetworkIn, AnomalyDetection band

# ⚠️ Cần CloudWatch Agent:
# 5. Memory > 85%: mem_used_percent >= 85
# 6. Disk > 80%: disk_used_percent >= 80
```

### Lambda — Danh Sách Alarm Cần Có

```bash
# ✅ 1. Error rate cao (dùng Metric Math)
Expression: (Errors/Invocations)*100, Threshold: >= 1%

# ✅ 2. Throttles
MetricName: Throttles, Threshold: >= 10, Period: 60

# ✅ 3. Duration tiếp cận timeout
MetricName: Duration (Maximum), Threshold: >= function_timeout * 0.9

# ✅ 4. DLQ errors (nếu có Dead Letter Queue)
MetricName: DeadLetterErrors, Threshold: >= 1

# ✅ 5. Iterator age (nếu là stream consumer)
MetricName: IteratorAge, Threshold: >= 60000 (1 phút)

# ✅ 6. Concurrent executions gần limit
MetricName: ConcurrentExecutions, Threshold: >= account_limit * 0.8
```

### ECS Fargate — Danh Sách Alarm Cần Có

```bash
# ✅ 1. Running task count thấp hơn desired
MetricName: RunningTaskCount, Threshold: < desired_count

# ✅ 2. Service CPU cao
MetricName: CPUUtilization (Service), Threshold: >= 80%

# ✅ 3. Service Memory cao
MetricName: MemoryUtilization (Service), Threshold: >= 85%

# ✅ 4. Pending tasks (capacity issue)
MetricName: PendingTaskCount, Threshold: >= 3, Period: 300
```

---

## Câu Hỏi Phỏng Vấn

**Q: Composite Alarm là gì? Khi nào dùng?**

> Composite Alarm kết hợp nhiều alarms với logic AND/OR. Dùng để giảm alert noise: thay vì alert mỗi khi CPU > 80% (có thể bình thường trong batch jobs), chỉ alert khi CPU > 80% VÀ Error rate > 5% cùng lúc — lúc đó mới là vấn đề thực sự. Cũng dùng để tạo "umbrella alarm" đại diện cho health của toàn bộ service.

**Q: Sự khác biệt giữa treat-missing-data notBreaching và breaching?**

> `notBreaching` — khi metric không có data, alarm coi như "OK" (không vi phạm). Phù hợp cho Lambda Errors — nếu không có invocations thì không có errors. `breaching` — khi metric không có data, alarm coi như đang vi phạm. Phù hợp cho metrics PHẢI có liên tục — nếu không có data là có vấn đề (vd: heartbeat metric từ ứng dụng).

**Q: Tại sao nên dùng `datapoints_to_alarm < evaluation_periods`?**

> Để tránh flapping (alarm liên tục ON/OFF) và false positives từ spike ngắn. Ví dụ `3 of 5`: alarm chỉ fire khi metric vi phạm 3 lần trong 5 phút liên tiếp — spike 1 phút không trigger. Điều này đặc biệt quan trọng với CPU (có thể spike ngắn 100% khi garbage collection).

**Q: Anomaly Detection khác gì Static Threshold Alarm?**

> Static threshold cần bạn biết trước ngưỡng "bình thường" và set con số cố định. Anomaly Detection ML model tự học pattern (trend, seasonality, time-of-day) và alert khi có điều gì đó bất thường so với lịch sử. Hữu ích cho metrics có pattern phức tạp: traffic cao thứ Hai, thấp Chủ Nhật — static threshold sẽ cần nhiều alarms khác nhau, anomaly detection tự xử lý.

**Q: Làm sao tránh alert fatigue?**

> (1) Chỉ alert những gì cần hành động ngay. (2) Alert on symptoms (error rate, latency) thay vì causes (CPU%). (3) Dùng `evaluation_periods` đủ dài để lọc spikes ngắn. (4) Dùng Composite Alarms để kết hợp điều kiện. (5) Review và xóa alarms không ai xử lý. (6) Phân cấp severity — P1 wake on-call, P3 Slack channel next morning.

---

**← Trước:** [2-cloudwatch-logs.md](./2-cloudwatch-logs.md) | **Tiếp theo →** [4-xray-tracing.md](./4-xray-tracing.md)

**Cập Nhật Lần Cuối:** 2026-05-15 | **Phiên Bản:** 1.0
