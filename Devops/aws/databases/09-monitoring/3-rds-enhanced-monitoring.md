# RDS Enhanced Monitoring — Giám Sát Nâng Cao Cấp OS

> Hướng dẫn toàn diện về RDS Enhanced Monitoring (Giám Sát Nâng Cao RDS) — OS-level metrics (Số Liệu Cấp Hệ Điều Hành), process monitoring (Giám Sát Tiến Trình), và phân tích sâu để debug các vấn đề hiệu năng mà CloudWatch basic metrics không thể phát hiện.

---

## 📚 Mục Lục

1. [Enhanced Monitoring là gì](#enhanced-monitoring-là-gì)
2. [Basic vs Enhanced Monitoring](#basic-vs-enhanced-monitoring)
3. [Bật Enhanced Monitoring](#bật-enhanced-monitoring)
4. [OS Metrics Quan Trọng](#os-metrics-quan-trọng)
5. [Process List Monitoring](#process-list-monitoring)
6. [Phân Tích Sự Cố Thực Tế](#phân-tích-sự-cố-thực-tế)
7. [CloudWatch Logs Integration](#cloudwatch-logs-integration)

---

## Enhanced Monitoring là gì

Enhanced Monitoring (Giám Sát Nâng Cao) là tính năng của RDS cung cấp metrics **cấp hệ điều hành** từ bên trong RDS instance — thứ mà CloudWatch basic metrics không thể nhìn thấy.

### Cơ Chế Hoạt Động

```
CloudWatch Basic Metrics (Không có Enhanced Monitoring):
  ┌─────────────────────────────────────────────┐
  │  AWS Hypervisor (Lớp Ảo Hóa)               │
  │    → CPUUtilization: 78% (toàn bộ instance) │
  │    → FreeableMemory: 2.1 GB                  │
  │    → IOPS: 1,200                             │
  └─────────────────────────────────────────────┘
  Chỉ thấy từ "bên ngoài" instance

Enhanced Monitoring (Có Agent bên trong OS):
  ┌─────────────────────────────────────────────┐
  │  RDS Instance OS (Amazon Linux)             │
  │    ├── CPU per core: core0=85%, core1=12%   │
  │    ├── Per-process CPU: mysqld=75%          │
  │    ├── Memory: buffer=4GB, cache=2GB        │
  │    ├── Swap: 0 MB (tốt) / 512 MB (xấu)     │
  │    ├── Disk I/O per device                  │
  │    └── Network per interface                │
  └─────────────────────────────────────────────┘
  Thấy từ "bên trong" — chi tiết hơn nhiều
```

### Granularity (Độ Chi Tiết) Có Thể Cấu Hình

```
Granularity options: 1, 5, 10, 15, 30, 60 giây

Production thường dùng: 60 giây (cân bằng cost/detail)
High-activity debugging: 1-5 giây (chi tiết nhất, tốn phí nhất)

Chi phí: CloudWatch Logs charges (~$0.50/GB ingested)
  - 60s granularity: ~$0.10/instance/tháng
  - 1s granularity:  ~$6/instance/tháng
```

---

## Basic vs Enhanced Monitoring

| Khía Cạnh                    | Basic CloudWatch Metrics | Enhanced Monitoring        |
| ---------------------------- | ------------------------ | -------------------------- |
| **Nguồn dữ liệu**            | AWS Hypervisor           | Agent trong OS             |
| **Granularity**              | 1 phút                   | 1-60 giây                  |
| **CPU detail**               | Total CPU %              | Per-CPU core, per-process  |
| **Memory detail**            | FreeableMemory (tổng)    | Buffer, cache, swap riêng  |
| **Process list**             | Không có                 | Top 100 processes          |
| **Disk I/O**                 | Tổng IOPS                | Per-device I/O             |
| **Network**                  | Tổng bytes               | Per-interface statistics   |
| **Chi phí**                  | Miễn phí                 | CloudWatch Logs charges    |
| **Nơi xem**                  | CloudWatch console       | CloudWatch Logs + RDS Console |
| **Cần bật thủ công**         | Không (tự động)          | Có (bật trong settings)    |

---

## Bật Enhanced Monitoring

### Yêu Cầu IAM

Enhanced Monitoring cần một IAM role để gửi metrics lên CloudWatch Logs:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": "arn:aws:logs:*:*:log-group:RDSOSMetrics:*"
    }
  ]
}
```

Trust relationship (quan hệ tin tưởng) cho role:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "monitoring.rds.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Bật qua AWS Console

```
1. Vào RDS Console → Databases → chọn instance
2. Click "Modify" (Sửa Đổi)
3. Tìm phần "Monitoring"
4. Bật "Enable Enhanced Monitoring"
5. Chọn Granularity: 60 seconds (khuyến nghị)
6. Chọn Monitoring Role: arn:aws:iam::123456789:role/rds-monitoring-role
7. Apply changes (áp dụng thay đổi)
```

### Bật qua AWS CLI

```bash
# Bật Enhanced Monitoring khi tạo instance mới
aws rds create-db-instance \
  --db-instance-identifier prod-mysql-01 \
  --db-instance-class db.r6g.large \
  --engine mysql \
  --monitoring-interval 60 \
  --monitoring-role-arn arn:aws:iam::123456789:role/rds-monitoring-role

# Bật Enhanced Monitoring cho instance đang chạy
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql-01 \
  --monitoring-interval 60 \
  --monitoring-role-arn arn:aws:iam::123456789:role/rds-monitoring-role \
  --apply-immediately
```

### Xem Enhanced Monitoring Data

```
Dữ liệu được gửi đến CloudWatch Logs:
  Log Group:  RDSOSMetrics
  Log Stream: db-instance-id (mỗi instance có stream riêng)

Format: JSON document mỗi interval

Xem qua:
  1. RDS Console → chọn instance → "Monitoring" tab → "OS Process list"
  2. CloudWatch Logs → RDSOSMetrics log group
  3. CloudWatch Logs Insights (để query)
```

---

## OS Metrics Quan Trọng

### CPU Metrics — Số Liệu CPU

Enhanced Monitoring cung cấp CPU breakdown chi tiết:

```json
// Ví dụ Enhanced Monitoring JSON output
{
  "cpuUtilization": {
    "user":    62.4,   // % CPU cho user space processes (mysqld, postgres...)
    "system":   8.1,   // % CPU cho kernel operations
    "wait":    18.3,   // % CPU bị block chờ I/O (I/O wait — đợi I/O)
    "idle":    11.2,   // % CPU nhàn rỗi
    "steal":    0.0,   // % CPU bị hypervisor "đánh cắp" (VM contention)
    "irq":      0.0,   // % CPU cho hardware interrupts
    "softirq":  0.0,   // % CPU cho software interrupts
    "nice":     0.0    // % CPU cho nice'd processes
  }
}
```

**Phân Tích từ CPU breakdown:**

```
cpu.user cao (> 60%):
  → Database đang làm nhiều CPU work (queries, sorting, hashing)
  → Kiểm tra slow queries, missing indexes

cpu.wait cao (> 20%):
  → I/O wait — database đang chờ disk
  → Storage IOPS không đủ
  → Cần upgrade IOPS hoặc instance có NVMe SSD

cpu.steal cao (> 5%):
  → Hypervisor đang lấy CPU từ instance này cho instance khác
  → Xem xét upgrade lên Dedicated Instance
  → Đây là vấn đề với "noisy neighbors" trên shared host

cpu.system cao:
  → Kernel overhead — có thể là network/disk driver issues
```

### Memory Metrics — Số Liệu Bộ Nhớ

```json
{
  "memory": {
    "free":            524288,    // KB bộ nhớ thực sự trống
    "cached":        4194304,     // KB dùng cho file cache (có thể giải phóng)
    "buffers":        262144,     // KB dùng cho buffer (có thể giải phóng)
    "active":        6291456,     // KB dùng bởi active processes
    "inactive":      1048576,     // KB dùng nhưng có thể swap
    "total":        16777216,     // KB tổng RAM
    "hugePagesFree":       0,     // Huge pages trống
    "hugePagesTotal":      0,     // Tổng huge pages
    "hugePagesSize":    2048      // KB mỗi huge page
  }
}
```

**Phân Tích:**

```
memory.free rất thấp + memory.cached cao → BÌNH THƯỜNG
  → Linux dùng RAM cho disk cache, giải phóng khi cần
  → MySQL/PostgreSQL buffer pool chiếm nhiều RAM là TỐT

memory.free thấp + memory.cached thấp + swap.free thấp → XẤU
  → RAM thực sự thiếu, OOM killer (Out-Of-Memory Killer) có thể kích hoạt
  → Cần upsize instance

Swap đang được dùng → LUÔN XẤU cho production database
  → Disk I/O chậm 100-1000x so với RAM
  → Queries sẽ timeout
  → Upsize instance ngay lập tức
```

### Disk I/O Metrics — Số Liệu Đĩa

```json
{
  "diskIO": [
    {
      "device":          "rdsdev",   // Device name (tên thiết bị)
      "readIOsPS":           1250,   // Read IOPS hiện tại
      "writeIOsPS":           480,   // Write IOPS hiện tại
      "readKbPS":           19200,   // KB đọc/giây
      "writeKbPS":           7680,   // KB ghi/giây
      "readLatency":         0.8,    // ms độ trễ đọc trung bình
      "writeLatency":        1.2,    // ms độ trễ ghi trung bình
      "await":               1.0,    // ms thời gian chờ I/O trung bình
      "tUtil":              58.5     // % thời gian device bận
    }
  ]
}
```

**Phân Tích:**

```
diskIO.tUtil > 90%:
  → I/O saturation (bão hòa) — disk không kịp phục vụ requests
  → Cần upgrade storage type (gp2 → gp3 → io2)
  → Hoặc tăng provisioned IOPS

diskIO.await cao hơn diskIO.readLatency:
  → Requests đang queue (xếp hàng) chờ disk
  → Càng xác nhận I/O saturation

diskIO.readKbPS / readIOsPS = block size (kích thước khối):
  → Nếu lớn (> 128 KB): Sequential I/O (đọc tuần tự) → tốt cho analytics
  → Nếu nhỏ (< 16 KB): Random I/O → OLTP workload, cần IOPS cao
```

### Network Metrics — Số Liệu Mạng

```json
{
  "network": [
    {
      "interface":  "eth0",
      "rx":         157286400,  // Bytes nhận/giây
      "tx":          52428800   // Bytes gửi/giây
    }
  ]
}
```

**Khi nào network là bottleneck:**
```
Network bandwidth (băng thông mạng) giới hạn theo instance class:
  db.t3.micro:   Up to 5 Gbps
  db.r6g.large:  Up to 10 Gbps
  db.r6g.4xlarge: Up to 25 Gbps

Nếu đang gần giới hạn:
  → Xem xét compression cho large result sets
  → Hoặc upsize instance class
  → Hoặc dùng Read Replicas để phân tán traffic
```

---

## Process List Monitoring

### Xem Process List trong RDS Console

RDS Console cho phép xem top 100 processes đang chạy trên OS của RDS instance:

```
RDS Console → Databases → chọn instance
→ Tab "Monitoring" → "OS Process list" (Danh Sách Tiến Trình OS)

Columns (Cột):
  PID        — Process ID (ID Tiến Trình)
  Name       — Tên tiến trình (mysqld, postgres, aurora_ql...)
  CPU (%)    — % CPU của tiến trình này
  Memory (%) — % Memory của tiến trình này
  RSS (KB)   — Resident Set Size (Kích Thước Bộ Nhớ Thực) — RAM đang dùng
  VSZ (KB)   — Virtual Size (Kích Thước Ảo) — bao gồm cả swap
  State      — Trạng thái (S=sleeping, R=running, D=waiting for I/O)
```

### Processes Quan Trọng Cần Theo Dõi

```
mysqld:
  → Main MySQL daemon (tiến trình chính MySQL)
  → Bình thường chiếm phần lớn CPU & Memory
  → Nếu CPU đột ngột > 90% → kiểm tra queries

aurora_ql:
  → Aurora query layer (tầng truy vấn Aurora)
  → Xử lý SQL parsing và optimization

aurora_log_manager:
  → Quản lý redo logs (nhật ký redo) của Aurora
  → Thường thấp

OS background processes:
  → mdaemon, sshd, cron... thường rất thấp
  → Nếu bất thường → cần investigate
```

### Phân Tích Process State (Trạng Thái Tiến Trình)

```
Process states (trạng thái tiến trình):
  R (Running):    Đang thực thi trên CPU
  S (Sleeping):   Chờ sự kiện (bình thường)
  D (Disk wait):  Chờ I/O hoàn thành (KHÔNG ngắt được)
  T (Stopped):    Bị dừng lại
  Z (Zombie):     Đã chết nhưng chưa được cleanup

Khi thấy nhiều processes ở trạng thái D:
  → I/O bottleneck nghiêm trọng
  → Disk không kịp phục vụ
  → Cần tăng IOPS ngay lập tức
```

---

## Phân Tích Sự Cố Thực Tế

### Tình Huống 1: CPU Spike Bí Ẩn

```
Triệu chứng: CPU spike 95% lúc 3 AM, không có user traffic

Điều tra với Enhanced Monitoring:

Bước 1: Xem cpuUtilization breakdown
  user: 88%, system: 5%, wait: 2%
  → CPU spike từ user space (không phải I/O wait)

Bước 2: Xem process list lúc 3 AM
  mysqld: 89% CPU
  → MySQL chính là nguyên nhân

Bước 3: Kết hợp với Performance Insights
  → Tìm thấy: ANALYZE TABLE statement chạy tự động
  → Scheduled maintenance job (công việc bảo trì định kỳ)

Giải pháp:
  → Dời ANALYZE TABLE sang maintenance window
  → Hoặc giảm scope: ANALYZE TABLE orders PARTITION (p2026)
```

### Tình Huống 2: Memory Leak

```
Triệu chứng: FreeableMemory giảm dần trong 3 ngày
  ngày 1: 4 GB free
  ngày 2: 2 GB free
  ngày 3: 800 MB free → swap bắt đầu dùng

Điều tra với Enhanced Monitoring:

Bước 1: Xem memory detail
  memory.active tăng đều
  memory.cached không giảm tương ứng

Bước 2: Xem process RSS qua thời gian
  mysqld RSS: ngày 1=8GB, ngày 2=10GB, ngày 3=12GB
  → MySQL đang dùng nhiều RAM hơn theo thời gian

Nguyên nhân có thể:
  - Performance Schema (Lược Đồ Hiệu Năng) accumulating data
  - Prepared statements (câu lệnh chuẩn bị) không được đóng
  - Connection không được cleanup đúng

Giải pháp:
  → Restart MySQL (maintenance window)
  → Tune performance_schema_max_* parameters
  → Fix application connection management
```

### Tình Huống 3: I/O Wait Cao Bí Ẩn

```
Triệu chứng: Queries chậm bất thường dù CPU < 30%

Điều tra với Enhanced Monitoring:

Bước 1: Xem cpuUtilization
  user: 25%, system: 8%, wait: 58% ← đây rồi!
  → 58% CPU time bị block chờ I/O

Bước 2: Xem diskIO metrics
  readLatency: 45ms (bình thường nên < 1ms!)
  tUtil: 98%
  → Disk bão hòa hoàn toàn

Bước 3: Xem IOPS
  ReadIOPS: 3,200 (đang dùng)
  Provisioned IOPS: 3,000 (giới hạn)
  → Đã vượt provisioned IOPS

Giải pháp:
  Ngay lập tức: Tăng Provisioned IOPS từ 3,000 lên 6,000
  Dài hạn: Upgrade sang gp3 với IOPS cấu hình riêng
           Hoặc thêm Read Replica để giảm read IOPS trên primary
```

---

## CloudWatch Logs Integration

### Truy Vấn Enhanced Monitoring Data

Enhanced Monitoring data được lưu trong CloudWatch Logs, có thể query bằng CloudWatch Logs Insights:

```
# Tìm khi nào swap đạt cao nhất
fields @timestamp, swap.free, swap.total
| parse @message "\"swap\":{*}" as swapJson
| filter swapJson like /free/
| sort @timestamp desc
| limit 100
```

### Export Metrics để Phân Tích Dài Hạn

```python
# Python script để extract Enhanced Monitoring data
import boto3
import json

client = boto3.client('logs', region_name='ap-southeast-1')

response = client.get_log_events(
    logGroupName='RDSOSMetrics',
    logStreamName='prod-mysql-01',
    startTime=int(start_time.timestamp() * 1000),
    endTime=int(end_time.timestamp() * 1000)
)

for event in response['events']:
    metrics = json.loads(event['message'])
    print(f"Time: {metrics['timestamp']}")
    print(f"CPU User: {metrics['cpuUtilization']['user']}%")
    print(f"Swap: {metrics['swap']['free']} KB free")
```

### Tạo CloudWatch Alarm Từ Enhanced Monitoring Logs

```bash
# Tạo metric filter (bộ lọc số liệu) từ Enhanced Monitoring logs
aws logs put-metric-filter \
  --log-group-name RDSOSMetrics \
  --filter-name rds-swap-usage \
  --filter-pattern '{ $.swap.used > 0 }' \
  --metric-transformations \
    metricName=RDSSwapUsed,\
    metricNamespace=Custom/RDS,\
    metricValue=$.swap.used,\
    unit=Kilobytes

# Tạo alarm từ custom metric vừa tạo
aws cloudwatch put-metric-alarm \
  --alarm-name "RDS-swap-usage-detected" \
  --metric-name RDSSwapUsed \
  --namespace Custom/RDS \
  --threshold 256000 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --period 60 \
  --statistic Average \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:db-critical
```

---

## Khi Nào Cần Enhanced Monitoring

```
✅ Bật Enhanced Monitoring khi:
  - Production database
  - Debugging CPU spikes không giải thích được
  - Investigating memory issues
  - Suspicious I/O patterns
  - Compliance yêu cầu OS-level audit

❌ Có thể bỏ qua khi:
  - Development / Test environments (tiết kiệm chi phí)
  - Batch processing databases ít critical
  - Aurora Serverless (không có OS metrics truyền thống)

💡 Tip:
  - Luôn bật cho production, tắt cho dev để tiết kiệm
  - Dùng granularity 60s cho production thường
  - Giảm xuống 1-5s khi đang active debugging
```

---

## Checklist Enhanced Monitoring

```
□ Tạo IAM role rds-monitoring-role với permissions đúng
□ Bật Enhanced Monitoring cho tất cả production instances
□ Granularity 60s cho production (balance cost/detail)
□ Thiết lập CloudWatch Logs retention policy (30 ngày)
□ Tạo CloudWatch Alarms từ Enhanced Monitoring logs cho swap
□ Document baseline metrics (CPU/memory bình thường) cho mỗi instance
□ Biết cách đọc process list để debug CPU spikes
□ Tạo runbook cho từng loại OS-level issue thường gặp
```

---

**Xem Trước:** [2-alerting-strategy.md](2-alerting-strategy.md) — Chiến Lược Cảnh Báo
**Xem Tiếp:** [4-dynamodb-monitoring.md](4-dynamodb-monitoring.md) — Giám Sát DynamoDB

**Cập Nhật Lần Cuối:** 2026-05-15
