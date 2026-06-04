# CloudWatch Metrics cho Amazon EFS

> EFS — Elastic File System — Hệ Thống Tệp Linh Hoạt — có cơ chế burst throughput phức tạp, đòi hỏi monitoring chặt chẽ để tránh performance giảm đột ngột

---

## 📋 Tổng Quan

Amazon EFS cung cấp metrics qua CloudWatch namespace `AWS/EFS`. Metrics được gửi **tự động mỗi phút** cho tất cả file systems.

```
Namespace: AWS/EFS
Dimension: FileSystemId
```

Khác với EBS (block storage), EFS là shared file system — nhiều client cùng mount và truy cập. Metrics phản ánh tổng tải từ **tất cả client** kết nối đến file system.

---

## 📊 Nhóm Metrics Chính

### 1. Throughput Metrics — Chỉ Số Thông Lượng

| Metric | Đơn Vị | Ý Nghĩa |
|--------|--------|---------|
| `DataReadIOBytes` | Bytes | Dữ liệu đọc từ EFS (tổng tất cả client) |
| `DataWriteIOBytes` | Bytes | Dữ liệu ghi vào EFS (tổng tất cả client) |
| `DataIOBytes` | Bytes | Tổng đọc + ghi |
| `MetadataIOBytes` | Bytes | I/O cho metadata operations (ls, stat, chmod...) |
| `TotalIOBytes` | Bytes | `DataIOBytes + MetadataIOBytes` |

**Tính throughput thực tế:**

```
Read Throughput  = DataReadIOBytes  / Period(giây)   → MB/s
Write Throughput = DataWriteIOBytes / Period(giây)   → MB/s
Total Throughput = TotalIOBytes     / Period(giây)   → MB/s
```

### 2. PermittedThroughput — Thông Lượng Được Phép

```
Metric: PermittedThroughput
Đơn vị: Bytes/second
Mô tả: Throughput tối đa EFS cho phép tại thời điểm đó
       (thay đổi theo chế độ throughput và BurstCreditBalance)
```

**Đây là metric quan trọng nhất để hiểu khả năng hiện tại của EFS.**

Khi `TotalIOBytes/Period` tiệm cận `PermittedThroughput` → EFS đang gần đạt giới hạn.

**Tính % Throughput đang dùng:**

```
Throughput Utilization % = (TotalIOBytes / Period) / PermittedThroughput × 100
```

Khi > 80% → cần xem xét tăng throughput hoặc đổi throughput mode.

### 3. BurstCreditBalance — Số Dư Credit Burst

```
Metric: BurstCreditBalance
Đơn vị: Bytes
Mô tả: Lượng credit burst còn lại (chỉ áp dụng cho Bursting throughput mode)
       Tích lũy khi dùng dưới baseline, tiêu khi burst
```

**Cơ chế tích lũy credit:**

```
Tích lũy khi: Throughput thực tế < Baseline throughput
Tiêu thụ khi: Throughput thực tế > Baseline throughput

Baseline = 50 KB/s per GB lưu trữ (Standard storage class)
Maximum credit = 2.1 TB (tương đương ~12 giờ burst tốc độ tối đa)
```

**Ví dụ:**

```
EFS có 100 GB dữ liệu:
  Baseline throughput = 100 GB × 50 KB/s = 5 MB/s
  Burst tối đa        = 100 MB/s

Khi ứng dụng dùng 2 MB/s (< 5 MB/s baseline):
  → tích lũy 3 MB/s dưới dạng credit

Khi ứng dụng cần 80 MB/s (> 5 MB/s baseline):
  → tiêu 75 MB/s credit
  → sau ~30 phút burst, credit hết → throughput giới hạn về 5 MB/s
```

**Ngưỡng cảnh báo BurstCreditBalance:**

| Mức | Ý Nghĩa | Hành Động |
|-----|---------|----------|
| `> 1 TB` | An toàn, nhiều dự phòng | Theo dõi định kỳ |
| `500 GB – 1 TB` | Bình thường | Tăng tần suất theo dõi |
| `< 500 GB` | Cần chú ý | Giảm tải hoặc xem xét đổi mode |
| `< 100 GB` | Nguy hiểm | Sắp hết burst capacity |
| `~ 0` | Throughput bị cap | Ứng dụng bị chậm nghiêm trọng |

**Alarm cho BurstCreditBalance:**

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EFS-BurstCreditBalance-Low" \
  --alarm-description "EFS burst credit balance sắp cạn" \
  --metric-name BurstCreditBalance \
  --namespace AWS/EFS \
  --dimensions Name=FileSystemId,Value=fs-0123456789abcdef0 \
  --period 300 \
  --evaluation-periods 3 \
  --threshold 1099511627776 \
  --comparison-operator LessThanThreshold \
  --statistic Average \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:efs-alerts
```

> `1099511627776 bytes = 1 TB` — ngưỡng cảnh báo khi còn dưới 1 TB credit

---

## 🔌 Connection Metrics — Chỉ Số Kết Nối

### ClientConnections — Số Kết Nối Client

```
Metric: ClientConnections
Đơn vị: Count
Mô tả: Số lượng NFS client connections đang hoạt động
       Cập nhật mỗi phút
```

**Dùng để:**
- Phát hiện khi số client tăng đột biến (deploy mới, scaling)
- Xác định file system có đang được dùng không
- Correlation với throughput để tính per-client throughput

**Ví dụ: Phân tích tải per client:**

```
Nếu ClientConnections = 50 và TotalIOBytes/Period = 100 MB/s
→ Trung bình mỗi client dùng 2 MB/s
```

---

## ⚡ PercentIOLimit — % Giới Hạn I/O

```
Metric: PercentIOLimit
Đơn vị: Percent (0–100%)
Áp dụng: CHỈ cho General Purpose performance mode
Mô tả: % đang dùng so với giới hạn I/O operations per second
```

**Đây là metric quan trọng để biết khi nào cần đổi performance mode.**

| Giá Trị | Ý Nghĩa | Hành Động |
|---------|---------|----------|
| `< 50%` | Dư dả dung lượng | Bình thường |
| `50–80%` | Đang dùng nhiều | Theo dõi chặt |
| `> 80%` liên tục | Sắp đến giới hạn | Cân nhắc chuyển Max I/O mode |
| `~100%` | Đạt giới hạn | Chuyển Max I/O ngay hoặc dùng EFS Elastic |

> **Lưu ý:** Max I/O performance mode không có metric này vì không có giới hạn IOPS cứng. Nhưng Max I/O có latency cao hơn General Purpose.

---

## 🏗️ Throughput Modes và Metrics Tương Ứng

EFS có 3 throughput modes — chế độ thông lượng:

### Bursting Mode — Chế Độ Bùng Nổ

```
Metrics cần theo dõi:
  ✓ BurstCreditBalance  → Còn bao nhiêu credit burst
  ✓ PermittedThroughput → Throughput cho phép tại thời điểm đó
  ✓ TotalIOBytes        → Throughput đang dùng

Dấu hiệu cần đổi mode:
  BurstCreditBalance liên tục dưới 500 GB
  và BurstCreditBalance không hồi phục vào ban đêm
```

### Provisioned Throughput — Thông Lượng Được Cấp Phát

```
Metrics cần theo dõi:
  ✓ PermittedThroughput  → Throughput được cấp phát (cố định)
  ✓ TotalIOBytes         → So sánh thực tế vs được cấp phát
  ✓ Utilization %        → Tính: TotalIOBytes/Period / PermittedThroughput

Dấu hiệu đang lãng phí:
  Utilization < 30% liên tục → đang trả tiền thừa
  → Giảm provisioned throughput hoặc chuyển Elastic
```

### Elastic Throughput — Thông Lượng Linh Hoạt

```
Metrics cần theo dõi:
  ✓ TotalIOBytes          → Throughput thực tế (tự scale)
  ✓ MeteredIOBytes        → Bytes được tính phí (khác với TotalIOBytes)
  ✓ PermittedThroughput   → Giới hạn tại thời điểm

Lợi thế: Không cần quản lý credit hay provisioning
Hạn chế: Chi phí cao hơn khi dùng nhiều
```

---

## 📊 CloudWatch Dashboard EFS

```bash
aws cloudwatch put-dashboard \
  --dashboard-name "EFS-Performance-Monitor" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "title": "Throughput — MB/s",
          "metrics": [
            [{"expression": "m1/1048576/PERIOD(m1)", "label": "Read MB/s", "id": "e1"}],
            [{"expression": "m2/1048576/PERIOD(m2)", "label": "Write MB/s", "id": "e2"}],
            ["AWS/EFS", "DataReadIOBytes", "FileSystemId", "fs-0123456789", {"id": "m1", "visible": false}],
            ["AWS/EFS", "DataWriteIOBytes", "FileSystemId", "fs-0123456789", {"id": "m2", "visible": false}]
          ],
          "view": "timeSeries",
          "period": 60,
          "stat": "Sum"
        }
      },
      {
        "type": "metric",
        "properties": {
          "title": "BurstCreditBalance (GB)",
          "metrics": [
            [{"expression": "m1/1073741824", "label": "Credit Balance (GB)", "id": "e1"}],
            ["AWS/EFS", "BurstCreditBalance", "FileSystemId", "fs-0123456789", {"id": "m1", "visible": false}]
          ],
          "view": "timeSeries",
          "period": 300,
          "stat": "Average"
        }
      },
      {
        "type": "metric",
        "properties": {
          "title": "PercentIOLimit (%) — General Purpose Mode",
          "metrics": [
            ["AWS/EFS", "PercentIOLimit", "FileSystemId", "fs-0123456789"]
          ],
          "view": "timeSeries",
          "period": 60,
          "stat": "Average",
          "yAxis": {"left": {"min": 0, "max": 100}}
        }
      },
      {
        "type": "metric",
        "properties": {
          "title": "Client Connections",
          "metrics": [
            ["AWS/EFS", "ClientConnections", "FileSystemId", "fs-0123456789"]
          ],
          "view": "timeSeries",
          "period": 60,
          "stat": "Sum"
        }
      }
    ]
  }'
```

---

## 🚨 Alarm Set Đầy Đủ cho EFS Production

```bash
#!/bin/bash
FS_ID="fs-0123456789abcdef0"
SNS_ARN="arn:aws:sns:us-east-1:123456789012:efs-critical"

# 1. BurstCreditBalance thấp (1 TB threshold)
aws cloudwatch put-metric-alarm \
  --alarm-name "EFS-${FS_ID}-BurstCredit-Low" \
  --metric-name BurstCreditBalance \
  --namespace AWS/EFS \
  --dimensions Name=FileSystemId,Value=$FS_ID \
  --period 300 --evaluation-periods 3 \
  --threshold 1099511627776 \
  --comparison-operator LessThanThreshold \
  --statistic Average --alarm-actions $SNS_ARN

# 2. PercentIOLimit cao (General Purpose mode)
aws cloudwatch put-metric-alarm \
  --alarm-name "EFS-${FS_ID}-IOLimit-High" \
  --metric-name PercentIOLimit \
  --namespace AWS/EFS \
  --dimensions Name=FileSystemId,Value=$FS_ID \
  --period 60 --evaluation-periods 10 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --statistic Average --alarm-actions $SNS_ARN

# 3. Throughput gần PermittedThroughput
# Dùng Metric Math để tính utilization %
aws cloudwatch put-metric-alarm \
  --alarm-name "EFS-${FS_ID}-Throughput-High" \
  --metrics "[
    {\"Id\":\"m1\",\"MetricStat\":{\"Metric\":{\"Namespace\":\"AWS/EFS\",\"MetricName\":\"TotalIOBytes\",\"Dimensions\":[{\"Name\":\"FileSystemId\",\"Value\":\"$FS_ID\"}]},\"Period\":60,\"Stat\":\"Sum\"},\"ReturnData\":false},
    {\"Id\":\"m2\",\"MetricStat\":{\"Metric\":{\"Namespace\":\"AWS/EFS\",\"MetricName\":\"PermittedThroughput\",\"Dimensions\":[{\"Name\":\"FileSystemId\",\"Value\":\"$FS_ID\"}]},\"Period\":60,\"Stat\":\"Average\"},\"ReturnData\":false},
    {\"Id\":\"utilization\",\"Expression\":\"(m1/60)/m2*100\",\"Label\":\"ThroughputUtilization\",\"ReturnData\":true}
  ]" \
  --threshold 80 --comparison-operator GreaterThanThreshold \
  --evaluation-periods 5 --alarm-actions $SNS_ARN
```

---

## 🔍 Troubleshooting — Chẩn Đoán Sự Cố EFS

### Kịch Bản 1: EFS chậm đột ngột vào giờ cao điểm

```
Triệu chứng: Application timeout khi đọc/ghi file
Kiểm tra:
  1. BurstCreditBalance → < 100 GB? → Đây là nguyên nhân
  2. PercentIOLimit → > 80%? → General Purpose đang bão hòa
  3. ClientConnections → Tăng đột biến? → Nhiều client hơn bình thường

Giải pháp:
  BurstBalance hết → Chuyển Provisioned hoặc Elastic throughput
  PercentIOLimit cao → Chuyển Max I/O hoặc EFS Elastic
```

### Kịch Bản 2: BurstCreditBalance không hồi phục

```
Triệu chứng: BurstCreditBalance giảm dần qua nhiều ngày
Nguyên nhân: Throughput trung bình > baseline (50 KB/s per GB)

Phân tích:
  1. Kiểm tra TotalIOBytes trung bình 24 giờ
  2. Tính baseline = EFS Size × 50 KB/s
  3. Nếu TotalIOBytes/Period > baseline → Credit mất nhiều hơn tích lũy

Giải pháp:
  - Tăng dung lượng EFS (tăng baseline) — không hiệu quả nếu dữ liệu ít
  - Chuyển Provisioned throughput với mức đủ dùng
  - Chuyển Elastic throughput (tự điều chỉnh, trả theo dùng)
```

### Kịch Bản 3: MetadataIOBytes cao bất thường

```
Triệu chứng: EFS chậm dù DataIOBytes bình thường
Nguyên nhân: Quá nhiều metadata operations (ls -R, find, stat liên tục)

Phân tích:
  MetadataIOBytes / TotalIOBytes × 100 > 30% → metadata overhead cao

Giải pháp:
  - Review application — giảm số lần list directory
  - Cache directory listing ở application layer
  - Xem xét tổ chức lại cấu trúc thư mục (ít file per directory)
```

---

## 📊 So Sánh Metrics theo Performance Mode

| Metric | General Purpose | Max I/O |
|--------|----------------|---------|
| `PercentIOLimit` | Có | Không có |
| `BurstCreditBalance` | Có (nếu Bursting mode) | Có |
| `DataReadIOBytes` | Có | Có |
| `Latency` | Thấp hơn | Cao hơn (sub-millisecond vs millisecond) |
| `TotalIOBytes` | Có | Có |

---

## 📝 Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa BurstCreditBalance của EFS và EBS?**
A: EBS BurstBalance đo theo % (0–100%), áp dụng cho gp2 IOPS. EFS BurstCreditBalance đo theo Bytes, áp dụng cho throughput trong Bursting mode. Cả hai đều tích lũy khi dùng dưới baseline và tiêu khi burst vượt baseline.

**Q: PercentIOLimit = 100% có nghĩa là gì và làm gì tiếp theo?**
A: EFS General Purpose đang đạt giới hạn IOPS. Ứng dụng sẽ thấy latency tăng. Giải pháp: chuyển sang Max I/O performance mode (chấp nhận latency cao hơn) hoặc dùng EFS Elastic throughput.

**Q: Tại sao MetadataIOBytes quan trọng?**
A: Metadata operations (ls, stat, open, close) tiêu thụ throughput giống data operations nhưng không chuyển dữ liệu. Application với nhiều small files hoặc frequent directory listing sẽ tốn nhiều throughput vào metadata, làm giảm throughput cho data thực sự.

---

**Tiếp Theo:** [4-s3-server-access-logging.md](4-s3-server-access-logging.md) — S3 Access Logging và phân tích với Athena

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
