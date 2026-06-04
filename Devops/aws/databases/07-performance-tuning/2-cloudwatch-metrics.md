# CloudWatch Metrics & Enhanced Monitoring — Giám Sát Hiệu Năng Database

> CloudWatch (Giám Sát Đám Mây) cung cấp metrics hệ thống cho tất cả dịch vụ database AWS, trong khi Enhanced Monitoring (Giám Sát Nâng Cao) cung cấp thêm số liệu cấp hệ điều hành (OS-level metrics) giúp chẩn đoán sâu hơn.

## 📚 Mục Lục

1. [Tổng Quan CloudWatch](#tổng-quan-cloudwatch)
2. [Metrics RDS Quan Trọng](#metrics-rds-quan-trọng)
3. [Metrics Aurora Quan Trọng](#metrics-aurora-quan-trọng)
4. [Metrics DynamoDB Quan Trọng](#metrics-dynamodb-quan-trọng)
5. [Metrics ElastiCache Quan Trọng](#metrics-elasticache-quan-trọng)
6. [Enhanced Monitoring — Giám Sát Nâng Cao](#enhanced-monitoring--giám-sát-nâng-cao)
7. [Thiết Lập Dashboard & Alarms](#thiết-lập-dashboard--alarms)
8. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🎯 Tổng Quan CloudWatch

### Luồng Thu Thập Metrics

```
┌───────────────────────────────────────────────────────────────────┐
│  Database Service (RDS / DynamoDB / ElastiCache)                  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  CloudWatch Agent / Native Integration                      │  │
│  │  - Thu thập metrics mỗi 1-5 phút (mặc định)                │  │
│  │  - Enhanced Monitoring: mỗi 1-60 giây                       │  │
│  └─────────────────────────┬───────────────────────────────────┘  │
└────────────────────────────┼──────────────────────────────────────┘
                             │ HTTPS / Xử lý nội bộ AWS
                             ▼
         ┌─────────────────────────────────┐
         │  Amazon CloudWatch              │
         │  - Lưu trữ time-series data     │
         │  - Giữ 15 tháng                 │
         │  - Hỗ trợ math expressions      │
         └────────────────┬────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
     Dashboard        Alarms          Metric Streams
  (Bảng Điều Khiển)  (Cảnh Báo)     (Luồng Dữ Liệu)
          │               │
          ▼               ▼
    Grafana /          SNS / Lambda /
    QuickSight         PagerDuty / Slack
```

### Hai Loại Metrics

| Loại | Mô Tả | Chi Phí |
|------|-------|--------|
| **Basic Monitoring** | 1 metric/phút, built-in | Miễn phí |
| **Enhanced Monitoring** | OS-level, mỗi 1-60 giây | Theo CloudWatch Logs pricing |

---

## 📊 Metrics RDS Quan Trọng

### Nhóm 1 — Compute (Tính Toán)

| Metric | Namespace | Ngưỡng Cảnh Báo | Ý Nghĩa |
|--------|-----------|----------------|--------|
| `CPUUtilization` | `AWS/RDS` | > 80% liên tục 5 phút | CPU đang bị saturated |
| `FreeableMemory` | `AWS/RDS` | < 256MB | Sắp hết RAM, swap sẽ tăng |
| `SwapUsage` | `AWS/RDS` | > 256MB | Đang dùng swap — rất chậm |

### Nhóm 2 — Storage & I/O

| Metric | Ngưỡng Cảnh Báo | Ý Nghĩa |
|--------|----------------|--------|
| `FreeStorageSpace` | < 10% dung lượng | Nguy cơ đầy disk |
| `DiskQueueDepth` | > 1 liên tục | I/O requests đang bị queue — bottleneck |
| `ReadIOPS` / `WriteIOPS` | Gần provisioned IOPS | Sắp đạt giới hạn IOPS |
| `ReadLatency` / `WriteLatency` | > 20ms | I/O latency cao bất thường |
| `ReadThroughput` / `WriteThroughput` | Gần max bandwidth | Bandwidth bottleneck |

### Nhóm 3 — Connections (Kết Nối)

| Metric | Ngưỡng Cảnh Báo | Ý Nghĩa |
|--------|----------------|--------|
| `DatabaseConnections` | > 80% `max_connections` | Sắp đạt giới hạn kết nối |
| `LoginFailures` | > 0 | Có thể là brute force hoặc config sai |

**Cách tính `max_connections` theo instance class:**
```
MySQL: max_connections ≈ DBInstanceClassMemory / 12,582,880
PostgreSQL: max_connections ≈ DBInstanceClassMemory / 9,531,392

Ví dụ — db.r5.xlarge (32 GB RAM):
MySQL: 32 * 1024 / 12 ≈ 2,730 connections
```

### Nhóm 4 — Replication (Sao Chép)

| Metric | Ngưỡng Cảnh Báo | Ý Nghĩa |
|--------|----------------|--------|
| `ReplicaLag` | > 60 giây | Read replica đang bị trễ |
| `BinLogDiskUsage` | > 10 GB | Binlog không được cleanup |

### Nhóm 5 — Transactions

| Metric | Ý Nghĩa |
|--------|--------|
| `CommitThroughput` | Số transactions commit/giây |
| `RollbackSegmentHistoryListLength` | PostgreSQL: dead rows cần vacuum |
| `Deadlocks` | Số deadlocks/giây — thường phải = 0 |
| `BlockedTransactions` | MySQL: transactions đang bị block |

---

## 🌟 Metrics Aurora Quan Trọng

Aurora có thêm các metrics đặc thù nhờ kiến trúc shared storage:

| Metric | Ngưỡng | Ý Nghĩa |
|--------|--------|--------|
| `AuroraReplicaLag` | > 100ms | Replica lag của Aurora (thường rất thấp) |
| `AuroraReplicaLagMaximum` | > 200ms | Lag của replica chậm nhất |
| `AuroraReplicaLagMinimum` | Theo dõi xu hướng | Lag của replica nhanh nhất |
| `BufferCacheHitRatio` | < 90% | Cache hit rate thấp → tăng instance size |
| `Queries` | Theo dõi xu hướng | Query throughput tổng thể |
| `RollbackSegmentHistoryListLength` | Tăng liên tục | Long-running transactions |
| `BinlogReplicationThroughput` | Theo dõi xu hướng | Throughput binlog replication |
| `VolumeBytesUsed` | Gần max (128 TB) | Dung lượng Aurora volume |
| `VolumeWriteIOPs` | Gần provisioned | IOPS trên Aurora storage |

### Aurora Serverless — Metrics Đặc Thù

| Metric | Ý Nghĩa |
|--------|--------|
| `ServerlessDatabaseCapacity` | ACU (Aurora Capacity Units — Đơn Vị Năng Lực Aurora) hiện tại |
| `ACUUtilization` | Phần trăm ACU đang dùng so với max |

---

## 📊 Metrics DynamoDB Quan Trọng

### Nhóm 1 — Capacity (Năng Lực)

| Metric | Ngưỡng Cảnh Báo | Ý Nghĩa |
|--------|----------------|--------|
| `ConsumedReadCapacityUnits` | Gần provisioned RCU | Sắp bị throttle đọc |
| `ConsumedWriteCapacityUnits` | Gần provisioned WCU | Sắp bị throttle ghi |
| `ProvisionedReadCapacityUnits` | - | Capacity đang provision |
| `ProvisionedWriteCapacityUnits` | - | Capacity đang provision |

### Nhóm 2 — Throttling (Giới Hạn)

| Metric | Ngưỡng Cảnh Báo | Ý Nghĩa |
|--------|----------------|--------|
| `ReadThrottleEvents` | > 0 | Requests bị throttle khi đọc |
| `WriteThrottleEvents` | > 0 | Requests bị throttle khi ghi |
| `ThrottledRequests` | > 0 | Tổng throttled requests |

### Nhóm 3 — Latency (Độ Trễ)

| Metric | Ngưỡng Cảnh Báo | Ý Nghĩa |
|--------|----------------|--------|
| `SuccessfulRequestLatency` | > 10ms (p99) | Latency cao bất thường |
| `GetItem.Latency` | > 5ms | GetItem chậm |
| `Query.Latency` | > 20ms | Query chậm |
| `Scan.Latency` | Theo dõi xu hướng | Scan thường chậm — hạn chế Scan |

### Nhóm 4 — Errors

| Metric | Ngưỡng Cảnh Báo | Ý Nghĩa |
|--------|----------------|--------|
| `SystemErrors` | > 0 | Lỗi phía DynamoDB |
| `UserErrors` | > 0 | Lỗi phía client (request không hợp lệ) |
| `ConditionalCheckFailedRequests` | Tăng đột biến | Có thể là race condition |

### Nhóm 5 — GSI (Global Secondary Index — Chỉ Mục Phụ Toàn Cầu)

| Metric | Ý Nghĩa |
|--------|--------|
| `OnlineIndexPercentageProgress` | Tiến độ tạo GSI mới |
| `OnlineIndexConsumedWriteCapacity` | WCU dùng khi tạo GSI |
| `OnlineIndexThrottleEvents` | Throttle trong quá trình tạo GSI |

---

## 🚀 Metrics ElastiCache Quan Trọng

### Redis — Metrics Quan Trọng

| Metric | Ngưỡng Cảnh Báo | Ý Nghĩa |
|--------|----------------|--------|
| `CacheHits` / `CacheMisses` | Hit Rate < 80% | Hiệu quả cache thấp |
| `BytesUsedForCache` | > 80% maxmemory | Sắp đầy memory |
| `Evictions` | > 0 liên tục | Keys bị xóa sớm do đầy memory |
| `CurrConnections` | Gần `maxclients` | Sắp đạt giới hạn connections |
| `ReplicationLag` | > 1 giây | Replica lag cao |
| `EngineCPUUtilization` | > 90% | Redis CPU saturated |
| `SwapUsage` | > 50MB | Nguy hiểm — Redis thường không nên swap |

### Memcached — Metrics Quan Trọng

| Metric | Ngưỡng | Ý Nghĩa |
|--------|--------|--------|
| `BytesUsedForCacheItems` | > 80% | Sắp đầy memory |
| `Evictions` | > 0 | Keys bị evict |
| `CurrConnections` | Gần `max_simultaneous_connections` | Kết nối sắp đầy |
| `GetMisses` | Cao liên tục | Cache miss rate cao |

---

## 🔬 Enhanced Monitoring — Giám Sát Nâng Cao

### Enhanced Monitoring Cung Cấp Gì?

Enhanced Monitoring cho phép nhìn vào **bên trong OS** của DB instance, bổ sung những gì CloudWatch metrics không có:

```
CloudWatch Metrics:              Enhanced Monitoring thêm:
- CPU tổng của instance          - CPU từng process (mysqld, postgres, etc.)
- Memory tổng                    - Memory từng process
- Disk I/O tổng                  - I/O từng volume
                                 - Số processes đang chạy
                                 - Swap activity chi tiết
                                 - Network interface stats
```

### Kích Hoạt Enhanced Monitoring

```bash
# Tạo IAM role trước
aws iam create-role \
  --role-name rds-monitoring-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "monitoring.rds.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name rds-monitoring-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole

# Kích hoạt cho DB instance
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --monitoring-interval 60 \
  --monitoring-role-arn arn:aws:iam::123456789012:role/rds-monitoring-role
```

### Monitoring Intervals — Khoảng Thời Gian Lấy Mẫu

| Interval | Use Case | Overhead |
|---------|---------|---------|
| 0 | Tắt Enhanced Monitoring | Không có |
| 1 giây | Debug sự cố hiệu năng nghiêm trọng | Cao |
| 5 giây | Production troubleshooting | Trung bình |
| 15 giây | Production monitoring | Nhẹ |
| 30 giây | Cân bằng detail/cost | Rất nhẹ |
| **60 giây** | **Mặc định cho production** | Tối thiểu |

### Enhanced Monitoring Metrics — Chi Tiết

Enhanced Monitoring gửi metrics qua **CloudWatch Logs** (không phải CloudWatch Metrics thông thường):

```
Log Group: RDSOSMetrics
Log Stream: db-instance-identifier

Payload (JSON):
{
  "engine": "MySQL",
  "instanceID": "mydb",
  "instanceResourceID": "db-XXXXX",
  "timestamp": "2024-01-01T12:00:00Z",
  "version": 1.00,
  "uptime": "0 days, 12:00:00",
  "numVCPUs": 4,
  "cpuUtilization": {
    "guest": 0.0,
    "irq": 0.0,
    "system": 2.1,
    "wait": 0.5,
    "idle": 92.0,
    "user": 5.4,
    "total": 8.0,
    "steal": 0.0,
    "nice": 0.0
  },
  "memory": {
    "writeback": 0,
    "hugePagesFree": 0,
    "hugePagesRsvd": 0,
    "hugePagesSize": 2048,
    "free": 8192000,
    "hugePagesTotal": 0,
    "cached": 16384000,
    "ram": 32768000,
    "total": 28672000,
    "active": 12288000
  },
  "tasks": {
    "sleeping": 456,
    "zombie": 0,
    "running": 2,
    "stopped": 0,
    "total": 458,
    "blocked": 0
  },
  "processList": [
    {
      "vss": 12345678,
      "name": "mysqld",
      "tgid": 1234,
      "parentID": 1,
      "memoryUsedPc": 45.2,
      "cpuUsedPc": 5.4,
      "id": 1234,
      "rss": 8765432
    }
  ]
}
```

---

## 📈 Thiết Lập Dashboard & Alarms

### Dashboard RDS Toàn Diện

```json
{
  "widgets": [
    {
      "type": "metric",
      "title": "CPU Utilization",
      "properties": {
        "metrics": [
          ["AWS/RDS", "CPUUtilization", "DBInstanceIdentifier", "mydb"],
          ["AWS/RDS", "CPUUtilization", "DBInstanceIdentifier", "mydb-replica"]
        ],
        "period": 60,
        "stat": "Average",
        "threshold": 80
      }
    },
    {
      "type": "metric",
      "title": "Database Connections",
      "properties": {
        "metrics": [
          ["AWS/RDS", "DatabaseConnections", "DBInstanceIdentifier", "mydb"]
        ],
        "period": 60,
        "stat": "Maximum"
      }
    },
    {
      "type": "metric",
      "title": "Read/Write Latency",
      "properties": {
        "metrics": [
          ["AWS/RDS", "ReadLatency", "DBInstanceIdentifier", "mydb"],
          ["AWS/RDS", "WriteLatency", "DBInstanceIdentifier", "mydb"]
        ],
        "period": 60,
        "stat": "p99"
      }
    },
    {
      "type": "metric",
      "title": "IOPS",
      "properties": {
        "metrics": [
          ["AWS/RDS", "ReadIOPS", "DBInstanceIdentifier", "mydb"],
          ["AWS/RDS", "WriteIOPS", "DBInstanceIdentifier", "mydb"]
        ],
        "period": 60,
        "stat": "Average"
      }
    }
  ]
}
```

### Thiết Lập Alarms Chuẩn Production

```bash
# Script tạo alarms cho RDS instance
DB_ID="mydb"
SNS_ARN="arn:aws:sns:ap-southeast-1:123456789012:db-alerts"

# Alarm 1: CPU cao
aws cloudwatch put-metric-alarm \
  --alarm-name "${DB_ID}-HighCPU" \
  --alarm-description "CPU > 80% trong 5 phút" \
  --metric-name CPUUtilization \
  --namespace AWS/RDS \
  --dimensions Name=DBInstanceIdentifier,Value=$DB_ID \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions $SNS_ARN

# Alarm 2: Connections gần đầy
aws cloudwatch put-metric-alarm \
  --alarm-name "${DB_ID}-HighConnections" \
  --alarm-description "Connections > 80% max" \
  --metric-name DatabaseConnections \
  --namespace AWS/RDS \
  --dimensions Name=DBInstanceIdentifier,Value=$DB_ID \
  --statistic Maximum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 200 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions $SNS_ARN

# Alarm 3: Free storage thấp
aws cloudwatch put-metric-alarm \
  --alarm-name "${DB_ID}-LowStorage" \
  --alarm-description "Free storage < 10 GB" \
  --metric-name FreeStorageSpace \
  --namespace AWS/RDS \
  --dimensions Name=DBInstanceIdentifier,Value=$DB_ID \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 10737418240 \
  --comparison-operator LessThanThreshold \
  --alarm-actions $SNS_ARN

# Alarm 4: Replica lag
aws cloudwatch put-metric-alarm \
  --alarm-name "${DB_ID}-ReplicaLag" \
  --alarm-description "ReplicaLag > 60 giây" \
  --metric-name ReplicaLag \
  --namespace AWS/RDS \
  --dimensions Name=DBInstanceIdentifier,Value="${DB_ID}-replica" \
  --statistic Average \
  --period 60 \
  --evaluation-periods 3 \
  --threshold 60 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions $SNS_ARN
```

### Composite Alarms — Cảnh Báo Tổng Hợp

```bash
# Cảnh báo chỉ khi CẢ HAI CPU cao VÀ IOPS cao (tránh false alarm)
aws cloudwatch put-composite-alarm \
  --alarm-name "${DB_ID}-PerformanceDegraded" \
  --alarm-description "CPU AND IOPS đều cao — có thể bottleneck nghiêm trọng" \
  --alarm-rule "ALARM(${DB_ID}-HighCPU) AND ALARM(${DB_ID}-HighIOPS)" \
  --alarm-actions $SNS_ARN
```

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Q: Metric nào quan trọng nhất để theo dõi cho RDS?

```
Top 5 metrics không thể thiếu:

1. CPUUtilization — CPU đang làm gì?
   Ngưỡng: 80% sustained

2. DatabaseConnections — Connections sắp đầy chưa?
   Ngưỡng: 80% max_connections

3. FreeStorageSpace — Storage còn bao nhiêu?
   Ngưỡng: < 20% cần cảnh báo, < 10% cần hành động

4. ReadLatency / WriteLatency — Response time thực tế
   Ngưỡng: > 20ms cần điều tra

5. ReplicaLag — Read replica có bị trễ không?
   Ngưỡng: > 60 giây cần cảnh báo
```

### Q: Enhanced Monitoring bổ sung gì mà CloudWatch không có?

```
CloudWatch metrics là "instance-level":
- Tổng CPU của toàn bộ instance
- Tổng memory của instance

Enhanced Monitoring cung cấp "process-level":
- CPU của mysqld process riêng biệt
- CPU của OS processes
- Memory breakdown chi tiết
- Zombie processes
- I/O wait breakdown

Use case thực tế:
- CPU 95% nhưng MySQL chỉ dùng 30% → Vấn đề ở OS hoặc process khác
- Enhanced Monitoring cho thấy process nào đang gây vấn đề
```

### Q: Làm thế nào để phát hiện DynamoDB hot partition qua CloudWatch?

```
DynamoDB không expose per-partition metrics trực tiếp.
Phương pháp:

1. CloudWatch Contributor Insights (bật trong DynamoDB):
   - Hiển thị top partition keys tiêu thụ capacity nhiều nhất
   - Xác định partition keys gây throttling

2. Xem tỷ lệ giữa consumed/provisioned:
   - ConsumedWriteCapacityUnits / ProvisionedWriteCapacityUnits
   - Nếu > 80% mà vẫn có throttling → hot partition

3. WriteThrottleEvents tăng dù capacity còn:
   → Dấu hiệu hot partition không phân phối đều

Giải pháp: Redesign partition key, thêm random suffix,
hoặc chuyển sang on-demand capacity mode.
```

---

**Tiếp Theo:** [3-slow-query-analysis.md](3-slow-query-analysis.md) — Slow Query Log & EXPLAIN Plans

**Cập Nhật Lần Cuối:** 2026-05-15
