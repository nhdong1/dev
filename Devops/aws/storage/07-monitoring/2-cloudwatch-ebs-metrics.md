# CloudWatch Metrics cho Amazon EBS

> EBS — Elastic Block Store — Lưu Trữ Khối Linh Hoạt — cung cấp metrics chi tiết để giám sát hiệu suất I/O, trạng thái burst và hàng đợi

---

## 📋 Tổng Quan

Khác với S3, tất cả EBS metrics được gửi đến CloudWatch **tự động mỗi 5 phút** (mặc định) hoặc **1 phút** nếu bật detailed monitoring — giám sát chi tiết (tốn phí thêm).

| Chế Độ | Tần Suất | Chi Phí | Kích Hoạt |
|--------|---------|---------|-----------|
| **Basic Monitoring** | 5 phút | Miễn phí | Mặc định |
| **Detailed Monitoring** — Giám Sát Chi Tiết | 1 phút | Tốn phí | Opt-in qua EC2 |

```
Namespace: AWS/EBS
```

---

## 📊 Nhóm Metrics Chính

### 1. Volume Read/Write Operations — Thao Tác Đọc/Ghi

| Metric | Đơn Vị | Ý Nghĩa |
|--------|--------|---------|
| `VolumeReadOps` | Count | Tổng số thao tác đọc trong kỳ |
| `VolumeWriteOps` | Count | Tổng số thao tác ghi trong kỳ |
| `VolumeReadBytes` | Bytes | Tổng dữ liệu đọc |
| `VolumeWriteBytes` | Bytes | Tổng dữ liệu ghi |

**Tính IOPS thực tế:**

```
IOPS = VolumeReadOps / Period(giây) + VolumeWriteOps / Period(giây)
```

Ví dụ: Nếu `VolumeReadOps = 3000` và `VolumeWriteOps = 1500` trong 300 giây:

```
IOPS = (3000 + 1500) / 300 = 15 IOPS/giây
```

**Tính Throughput — Thông Lượng:**

```
Throughput = (VolumeReadBytes + VolumeWriteBytes) / Period(giây)
```

### 2. Latency Metrics — Chỉ Số Độ Trễ

| Metric | Đơn Vị | Ý Nghĩa |
|--------|--------|---------|
| `VolumeTotalReadTime` | Seconds | Tổng thời gian cho tất cả read operations |
| `VolumeTotalWriteTime` | Seconds | Tổng thời gian cho tất cả write operations |

**Tính Average Read Latency — Độ Trễ Đọc Trung Bình:**

```
ReadLatency = VolumeTotalReadTime / VolumeReadOps × 1000  (ms)
```

> **Ngưỡng tốt:** gp3 thường < 1ms. Nếu > 20ms cần điều tra.

### 3. VolumeQueueLength — Độ Dài Hàng Đợi I/O

```
Metric: VolumeQueueLength
Đơn vị: Count
Mô tả: Số I/O requests đang chờ xử lý (chưa hoàn thành)
```

**Đây là metric quan trọng nhất để phát hiện EBS bị nghẽn cổ chai (I/O bottleneck).**

| Giá Trị | Ý Nghĩa | Hành Động |
|---------|---------|----------|
| `0` | EBS xử lý ngay, không có chờ | Bình thường |
| `< 1` | Hàng đợi nhỏ, vẫn tốt | Theo dõi |
| `1 – 5` | Bắt đầu có áp lực | Xem xét tăng IOPS |
| `> 5` (kéo dài) | I/O bottleneck — nghẽn cổ chai | Cần hành động ngay |

**Công thức tính VolumeQueueLength lý tưởng:**

```
Lý tưởng: VolumeQueueLength ≤ 1 per 1000 IOPS được provision
Ví dụ: gp3 với 3000 IOPS → VolumeQueueLength nên ≤ 3
```

---

## ⚡ BurstBalance — Số Dư Burst

BurstBalance là metric **đặc biệt quan trọng** cho gp2 và st1/sc1 volumes — các loại có cơ chế tích lũy credit burst.

### Cơ Chế Burst Credit

```
Metric: BurstBalance
Đơn vị: Percent (0–100%)
Áp dụng: gp2, st1, sc1 (KHÔNG áp dụng cho gp3, io1, io2)
```

**Nguyên tắc hoạt động:**

```
[Khi tải nhẹ]  → IOPS < baseline → tích lũy credit vào "bucket"
[Khi tải cao]  → IOPS > baseline → tiêu thụ credit từ "bucket"
[Khi hết credit] → IOPS bị giới hạn xuống baseline → performance giảm mạnh
```

**Baseline IOPS cho gp2:**

```
Baseline = 3 IOPS/GB
Volume 100GB → baseline 300 IOPS
Volume 500GB → baseline 1500 IOPS

Maximum burst = 3000 IOPS (khi BurstBalance = 100%)
Credit tích lũy tối đa = 5.4 triệu credits (= 30 phút burst full)
```

**Ví dụ thực tế:**

```
Volume: gp2 100GB, baseline = 300 IOPS
Ban đêm (thấp tải): xài 50 IOPS → tích lũy 250 IOPS/giây dưới dạng credit
Ban ngày (cao tải): xài 3000 IOPS → rút credit
Nếu burst liên tục > 30 phút → BurstBalance về 0% → IOPS giảm xuống 300
```

### Ngưỡng Cảnh Báo BurstBalance

| Mức BurstBalance | Trạng Thái | Hành Động |
|-----------------|-----------|----------|
| `> 50%` | Bình thường | Không cần làm gì |
| `20% – 50%` | Cần chú ý | Theo dõi chặt hơn |
| `< 20%` | Cảnh báo | Xem xét upgrade volume |
| `< 10%` | Nguy hiểm | Upgrade ngay hoặc tắt ứng dụng tải cao |
| `0%` | IOPS bị cap | Ứng dụng bị chậm |

**CloudWatch Alarm cho BurstBalance:**

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EBS-BurstBalance-Low" \
  --alarm-description "BurstBalance dưới 20% — cần kiểm tra" \
  --metric-name BurstBalance \
  --namespace AWS/EBS \
  --dimensions Name=VolumeId,Value=vol-0123456789abcdef0 \
  --period 300 \
  --evaluation-periods 3 \
  --threshold 20 \
  --comparison-operator LessThanThreshold \
  --statistic Average \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts
```

> **Giải pháp dứt điểm:** Migrate từ gp2 sang gp3. gp3 không có cơ chế burst — IOPS được provision cố định, không bị giảm đột ngột. Vẫn rẻ hơn gp2 khoảng 20%.

---

## 🔥 io1/io2 — Provisioned IOPS Volumes

io1 và io2 không có BurstBalance nhưng có metrics riêng để giám sát:

### VolumeConsumedReadWriteOps — IOPS Thực Tế

```
Metric: VolumeConsumedReadWriteOps
Đơn vị: Count
Mô tả: Tổng IOPS thực sự được dùng
```

**Kiểm tra xem có đang "lãng phí" IOPS không:**

```
IOPS Utilization % = VolumeConsumedReadWriteOps / (ProvisionedIOPS × Period) × 100
```

Nếu utilization < 30% liên tục → đang over-provision → lãng phí tiền.

### VolumeThroughputPercentage — Tỉ Lệ Throughput

```
Metric: VolumeThroughputPercentage
Đơn vị: Percent
Mô tả: % throughput so với maximum của volume type
Chỉ có: io1/io2
```

---

## 📈 Volume Status Checks — Kiểm Tra Trạng Thái Volume

EBS cũng báo cáo trạng thái volume vào CloudWatch:

```
Metric: VolumeIdleTime
Đơn vị: Seconds
Mô tả: Thời gian volume không có I/O request nào
```

**Dùng để phát hiện:**
- Volume không được dùng (xem xét xóa để tiết kiệm chi phí)
- Application ngừng ghi vào database bất thường

---

## 🛠️ Alarm Thực Tế

### Alarm Set hoàn chỉnh cho EBS Production

```bash
#!/bin/bash
VOLUME_ID="vol-0123456789abcdef0"
SNS_ARN="arn:aws:sns:us-east-1:123456789012:ops-critical"

# 1. BurstBalance thấp (chỉ cho gp2/st1/sc1)
aws cloudwatch put-metric-alarm \
  --alarm-name "EBS-${VOLUME_ID}-BurstBalance-Critical" \
  --metric-name BurstBalance \
  --namespace AWS/EBS \
  --dimensions Name=VolumeId,Value=$VOLUME_ID \
  --period 300 --evaluation-periods 2 \
  --threshold 10 --comparison-operator LessThanThreshold \
  --statistic Average --alarm-actions $SNS_ARN

# 2. Queue Length cao
aws cloudwatch put-metric-alarm \
  --alarm-name "EBS-${VOLUME_ID}-QueueLength-High" \
  --metric-name VolumeQueueLength \
  --namespace AWS/EBS \
  --dimensions Name=VolumeId,Value=$VOLUME_ID \
  --period 60 --evaluation-periods 5 \
  --threshold 10 --comparison-operator GreaterThanThreshold \
  --statistic Average --alarm-actions $SNS_ARN

# 3. Read Latency cao (dùng Metric Math)
aws cloudwatch put-metric-alarm \
  --alarm-name "EBS-${VOLUME_ID}-ReadLatency-High" \
  --metrics '[
    {"Id":"readTime","MetricStat":{"Metric":{"Namespace":"AWS/EBS","MetricName":"VolumeTotalReadTime","Dimensions":[{"Name":"VolumeId","Value":"'$VOLUME_ID'"}]},"Period":60,"Stat":"Average"},"ReturnData":false},
    {"Id":"readOps","MetricStat":{"Metric":{"Namespace":"AWS/EBS","MetricName":"VolumeReadOps","Dimensions":[{"Name":"VolumeId","Value":"'$VOLUME_ID'"}]},"Period":60,"Stat":"Average"},"ReturnData":false},
    {"Id":"latency","Expression":"(readTime/readOps)*1000","Label":"ReadLatencyMs","ReturnData":true}
  ]' \
  --threshold 20 --comparison-operator GreaterThanThreshold \
  --evaluation-periods 5 --alarm-actions $SNS_ARN
```

---

## 📋 Dashboard EBS — Terraform

Tạo CloudWatch Dashboard bằng Terraform — Công Cụ Infrastructure as Code:

```hcl
resource "aws_cloudwatch_dashboard" "ebs_monitoring" {
  dashboard_name = "EBS-Performance-Dashboard"

  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric"
        properties = {
          title   = "IOPS — Read vs Write"
          metrics = [
            ["AWS/EBS", "VolumeReadOps", "VolumeId", var.volume_id, {label = "Read IOPS"}],
            ["AWS/EBS", "VolumeWriteOps", "VolumeId", var.volume_id, {label = "Write IOPS"}]
          ]
          period = 60
          stat   = "Sum"
          view   = "timeSeries"
        }
      },
      {
        type = "metric"
        properties = {
          title   = "BurstBalance %"
          metrics = [
            ["AWS/EBS", "BurstBalance", "VolumeId", var.volume_id]
          ]
          period = 300
          stat   = "Average"
          yAxis  = { left = { min = 0, max = 100 } }
        }
      },
      {
        type = "metric"
        properties = {
          title   = "Queue Length — I/O Bottleneck Indicator"
          metrics = [
            ["AWS/EBS", "VolumeQueueLength", "VolumeId", var.volume_id]
          ]
          period = 60
          stat   = "Average"
        }
      }
    ]
  })
}
```

---

## 🔍 Chẩn Đoán I/O Bottleneck — Quy Trình

Khi ứng dụng báo chậm, quy trình chẩn đoán EBS:

```
Bước 1: Kiểm tra VolumeQueueLength
         > 5 sustained → CONFIRM: có bottleneck

Bước 2: Kiểm tra IOPS thực tế
         VolumeReadOps + VolumeWriteOps / Period
         So sánh với ProvisionedIOPS của volume type

Bước 3: Kiểm tra BurstBalance (nếu là gp2)
         < 20% → đây là nguyên nhân
         Giải pháp: chuyển sang gp3

Bước 4: Kiểm tra Latency
         VolumeTotalReadTime / VolumeReadOps
         > 20ms → bất thường

Bước 5: Kiểm tra EC2 instance type
         Có instance store bandwidth limit không?
         Dùng: aws ec2 describe-instance-types
```

---

## 📊 So Sánh: gp2 vs gp3 từ góc độ Monitoring

| Tiêu Chí | gp2 | gp3 |
|---------|-----|-----|
| **BurstBalance** | Có — phải monitor | Không — không có burst |
| **IOPS ổn định** | Không — biến động theo burst | Có — luôn ở mức provision |
| **Predictability** — Dự đoán được | Thấp | Cao |
| **Metric cần theo dõi** | BurstBalance, QueueLength | QueueLength, Latency |
| **Chi phí** | Cao hơn | Thấp hơn 20% |

---

## 📝 Câu Hỏi Phỏng Vấn

**Q: EBS gp2 bị chậm đột ngột vào giờ cao điểm, làm sao debug?**
A: Kiểm tra ngay CloudWatch metric BurstBalance. Nếu về 0% tức là đã cạn credit, IOPS bị cap về baseline (3 IOPS/GB). Giải pháp ngắn hạn: tăng kích thước volume. Dài hạn: migrate sang gp3.

**Q: VolumeQueueLength = 0 có nghĩa là gì?**
A: Volume đang idle — không có I/O request nào đang chờ. Hoặc EBS xử lý kịp ngay lập tức không có hàng đợi. Đây là trạng thái lý tưởng.

**Q: Sự khác biệt giữa VolumeReadOps và VolumeConsumedReadWriteOps?**
A: VolumeReadOps là số thao tác đọc thực tế. VolumeConsumedReadWriteOps chỉ có trên io1/io2, đo IOPS so với ngưỡng provision — giúp kiểm tra xem có đang lãng phí IOPS không.

---

**Tiếp Theo:** [3-cloudwatch-efs-metrics.md](3-cloudwatch-efs-metrics.md) — EFS throughput và burst credit

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
