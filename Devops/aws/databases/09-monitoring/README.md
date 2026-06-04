# 09 — Monitoring & Observability — Giám Sát & Quan Sát Toàn Diện

> Hướng dẫn đầy đủ về giám sát AWS Database Services — từ CloudWatch Metrics (Số Liệu CloudWatch), Enhanced Monitoring (Giám Sát Nâng Cao), Alerting Strategy (Chiến Lược Cảnh Báo), đến Database Activity Streams (Luồng Hoạt Động Cơ Sở Dữ Liệu) và Observability best practices (Thực Hành Tốt Nhất về Quan Sát Hệ Thống).

---

## 📚 Mục Lục

1. [Tại Sao Monitoring Quan Trọng](#tại-sao-monitoring-quan-trọng)
2. [Ba Trụ Cột Observability](#ba-trụ-cột-observability)
3. [Tổng Quan Công Cụ Giám Sát AWS](#tổng-quan-công-cụ-giám-sát-aws)
4. [Lộ Trình Học](#lộ-trình-học)
5. [Kiến Trúc Monitoring Toàn Diện](#kiến-trúc-monitoring-toàn-diện)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Monitoring Quan Trọng

Trong môi trường production, **vấn đề không phải là liệu sự cố có xảy ra hay không, mà là khi nào**. Monitoring tốt giúp:

- **Phát hiện sớm** — Nhận cảnh báo trước khi người dùng báo lỗi
- **Giảm MTTR** — MTTR (Mean Time To Recovery — Thời Gian Phục Hồi Trung Bình) ngắn hơn khi có dữ liệu đầy đủ
- **Capacity Planning** — Lập kế hoạch mở rộng dựa trên xu hướng thực tế
- **Cost Optimization** — Phát hiện tài nguyên lãng phí qua metrics
- **Compliance** — Đáp ứng yêu cầu kiểm toán PCI-DSS, HIPAA, GDPR

### Hậu Quả của Monitoring Kém

```
Không có monitoring đúng → Phản ứng thay vì phòng ngừa

Kịch bản thường gặp:
1. Database chậm → Người dùng phàn nàn → Team nhận ticket
2. Mất vài giờ để xác định nguyên nhân (thiếu dữ liệu lịch sử)
3. Fix khẩn cấp không đúng nguyên nhân gốc
4. Sự cố tái diễn sau 2 tuần
```

---

## Ba Trụ Cột Observability

Observability (Quan Sát Hệ Thống) hiện đại dựa trên **3 trụ cột**:

### 1. Metrics — Số Liệu

**Định nghĩa:** Dữ liệu số học được đo lường theo thời gian (time-series data)

```
Ví dụ:
- CPU Utilization: 78% lúc 14:30:00
- DatabaseConnections: 245 lúc 14:30:00
- ReadIOPS: 1,200 lúc 14:30:00

Công cụ AWS: Amazon CloudWatch Metrics
```

**Ưu điểm:**
- Chi phí thấp để lưu trữ lâu dài
- Dễ tạo dashboards và alerts
- Phù hợp để phát hiện xu hướng

**Hạn chế:**
- Không có ngữ cảnh — chỉ biết "cái gì" không biết "tại sao"

---

### 2. Logs — Nhật Ký

**Định nghĩa:** Bản ghi sự kiện rời rạc với timestamp và thông tin chi tiết

```
Ví dụ:
[2026-05-15 14:32:15] SLOW QUERY: SELECT * FROM orders WHERE user_id = 123
  Execution time: 8,450ms  Rows examined: 2,450,000

Công cụ AWS: Amazon CloudWatch Logs, RDS Enhanced Logging
```

**Ưu điểm:**
- Ngữ cảnh phong phú để debug
- Có thể lọc và tìm kiếm

**Hạn chế:**
- Tốn chi phí lưu trữ nếu log nhiều
- Cần CloudWatch Logs Insights để phân tích hiệu quả

---

### 3. Traces — Vết Theo Dõi

**Định nghĩa:** Theo dõi một request qua toàn bộ hệ thống (distributed tracing — theo dõi phân tán)

```
Ví dụ:
Request ID: abc-123
  ├─ API Gateway: 5ms
  ├─ Lambda function: 45ms
  │   ├─ ElastiCache lookup: 2ms (MISS)
  │   └─ RDS query: 38ms ← bottleneck
  └─ Response: 50ms total

Công cụ AWS: AWS X-Ray
```

**Ưu điểm:**
- Xác định bottleneck (điểm nghẽn) trong hệ thống phức tạp
- End-to-end visibility (Tầm Nhìn Đầu Cuối)

---

## Tổng Quan Công Cụ Giám Sát AWS

### CloudWatch — Trung Tâm Giám Sát AWS

```
Amazon CloudWatch
├── Metrics         — Thu thập số liệu tự động từ tất cả dịch vụ AWS
├── Logs            — Lưu trữ và phân tích nhật ký hệ thống
├── Alarms          — Cảnh báo dựa trên ngưỡng metric
├── Dashboards      — Bảng điều khiển trực quan
├── Logs Insights   — Truy vấn logs bằng ngôn ngữ riêng (như SQL)
└── Contributor Insights — Phân tích top contributors gây tải
```

### RDS-Specific Tools — Công Cụ Chuyên Biệt Cho RDS

| Công Cụ                     | Mục Đích                                               | Granularity (Độ Chi Tiết) |
| --------------------------- | ------------------------------------------------------ | ------------------------- |
| **CloudWatch Basic Metrics** | CPU, Memory, IOPS, Connections                        | 1 phút                    |
| **Enhanced Monitoring**     | OS-level metrics (CPU per process, memory breakdown)  | 1-60 giây                 |
| **Performance Insights**    | SQL-level analysis, wait events, top queries          | 1 giây                    |
| **Database Activity Streams** | Real-time audit log mọi hoạt động DB               | Near real-time             |

### DynamoDB-Specific Tools — Công Cụ Chuyên Biệt Cho DynamoDB

| Công Cụ                        | Mục Đích                                           |
| ------------------------------ | -------------------------------------------------- |
| **CloudWatch Table Metrics**   | ConsumedRCU, ConsumedWCU, Throttled requests       |
| **Contributor Insights**       | Phát hiện hot keys (khóa nóng) gây throttling      |
| **CloudTrail**                 | Audit trail cho tất cả API calls                   |

---

## Lộ Trình Học

### Bước 1: CloudWatch Foundations (Nền Tảng CloudWatch)

```
→ Đọc: 1-cloudwatch-dashboards.md
Mục tiêu:
  - Hiểu CloudWatch Metrics namespace cho RDS/Aurora/DynamoDB/ElastiCache
  - Biết key metrics cần theo dõi cho từng dịch vụ
  - Tạo dashboard tổng hợp
```

### Bước 2: Alerting Strategy (Chiến Lược Cảnh Báo)

```
→ Đọc: 2-alerting-strategy.md
Mục tiêu:
  - Thiết lập thresholds (ngưỡng) dựa trên SLO
  - SLO (Service Level Objective — Mục Tiêu Mức Dịch Vụ) cho database
  - Alert routing — đúng người nhận đúng cảnh báo
  - Tránh alert fatigue (mệt mỏi cảnh báo)
```

### Bước 3: Enhanced Monitoring Deep Dive (Đi Sâu Giám Sát Nâng Cao)

```
→ Đọc: 3-rds-enhanced-monitoring.md
Mục tiêu:
  - Hiểu OS-level metrics vs CloudWatch basic metrics
  - Process list monitoring (giám sát danh sách tiến trình)
  - Phát hiện memory leaks (rò rỉ bộ nhớ), swap (bộ nhớ ảo) usage
```

### Bước 4: DynamoDB Monitoring Chuyên Sâu

```
→ Đọc: 4-dynamodb-monitoring.md
Mục tiêu:
  - Throttling metrics và cách phản ứng
  - Capacity planning với CloudWatch
  - Contributor Insights cho hot partitions
```

### Bước 5: Activity Streams & Audit (Luồng Hoạt Động & Kiểm Toán)

```
→ Đọc: 5-activity-streams.md
Mục tiêu:
  - Database Activity Streams cho Aurora và RDS
  - Integration với SIEM (Security Information and Event Management — Quản Lý Thông Tin & Sự Kiện Bảo Mật)
  - Compliance reporting tự động
```

---

## Kiến Trúc Monitoring Toàn Diện

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AWS Database Monitoring Stack                     │
│                (Bộ Giám Sát Cơ Sở Dữ Liệu AWS)                    │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │     RDS /    │  │   DynamoDB   │  │      ElastiCache         │  │
│  │    Aurora    │  │              │  │       Redis              │  │
│  └──────┬───────┘  └──────┬───────┘  └────────────┬─────────────┘  │
│         │                 │                        │                 │
│  ┌──────▼─────────────────▼────────────────────────▼─────────────┐  │
│  │              Amazon CloudWatch                                  │  │
│  │   Metrics │ Logs │ Alarms │ Dashboards │ Logs Insights         │  │
│  └──────┬────────────────────────────────────────────────────────┘  │
│         │                                                            │
│  ┌──────▼──────────────────────────────────────────────────────┐   │
│  │               Alert Routing (Định Tuyến Cảnh Báo)           │   │
│  │  SNS Topic → PagerDuty / OpsGenie / Slack / Email           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │         Activity Streams → Kinesis → SIEM / S3              │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Tầng Giám Sát Theo Độ Sâu

```
Level 1: Basic CloudWatch Metrics (Tự động, miễn phí cho hầu hết metrics)
  └── CPU, Memory, IOPS, Connections, Latency

Level 2: Enhanced Monitoring (Tốn phí nhỏ, cần bật thủ công)
  └── OS processes, per-CPU metrics, memory detail

Level 3: Performance Insights (Miễn phí 7 ngày, tính phí lưu lâu hơn)
  └── SQL-level analysis, wait events, lock waits

Level 4: Database Activity Streams (Aurora, tính phí)
  └── Every SQL statement, user, IP, timestamp
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Sự khác biệt giữa CloudWatch Basic Metrics và Enhanced Monitoring?

```
Basic CloudWatch Metrics:
  - Dữ liệu từ hypervisor (lớp ảo hóa), không phải OS thực
  - Granularity: 1 phút
  - Miễn phí
  - Metrics: CPU, FreeableMemory, IOPS...

Enhanced Monitoring:
  - Agent chạy bên trong OS của RDS instance
  - Granularity: 1-60 giây (cấu hình được)
  - Tốn phí CloudWatch Logs
  - Thêm: per-process CPU, memory breakdown, OS metrics

→ Khi nào dùng Enhanced Monitoring?
  - Debugging CPU spike — xem process nào gây ra
  - Phát hiện swap usage (database không nên dùng swap)
  - Phân tích memory pressure chi tiết
```

### Câu 2: Làm thế nào để phát hiện DynamoDB hot partition?

```
Cách 1: CloudWatch Metrics
  - SuccessfulRequestLatency tăng đột biến
  - ThrottledRequests > 0 liên tục
  - ConsumedCapacityUnits bất thường

Cách 2: Contributor Insights (khuyến nghị)
  - Bật Contributor Insights cho bảng DynamoDB
  - Tự động báo cáo top keys gây throttling
  - Xem partition key nào chiếm nhiều RCU/WCU nhất

→ Fix: Write sharding (phân mảnh ghi) hoặc redesign partition key
```

### Câu 3: Thiết kế monitoring strategy cho production RDS database?

```
1. Bật Enhanced Monitoring (granularity 60s)
2. Bật Performance Insights (retention 7 ngày miễn phí)
3. Tạo CloudWatch Alarms cho:
   - CPUUtilization > 80% trong 5 phút
   - FreeStorageSpace < 10 GB
   - DatabaseConnections > 80% max_connections
   - ReadLatency / WriteLatency > 20ms
   - ReplicaLag > 30 giây (nếu có Read Replica)
4. Dashboard tổng hợp cho on-call engineer
5. Activity Streams nếu cần audit trail
```

### Câu 4: SLO cho database là gì và cách đo?

```
SLO (Service Level Objective — Mục Tiêu Mức Dịch Vụ) ví dụ:

  Availability: 99.95% uptime/month (~22 phút downtime)
  Latency P99: < 100ms cho read queries
  Error rate: < 0.1% failed transactions

Cách đo bằng CloudWatch:
  - Availability: từ RDS events + CloudWatch
  - Latency: ReadLatency / WriteLatency percentiles
  - Error rate: DatabaseErrorCount / TotalRequests
```

---

## 📁 Các File Trong Module Này

| File                            | Nội Dung                                                   |
| ------------------------------- | ---------------------------------------------------------- |
| `README.md`                     | Tổng quan, kiến trúc, lộ trình học (file này)             |
| `1-cloudwatch-dashboards.md`    | Thiết lập dashboards, key metrics theo dịch vụ            |
| `2-alerting-strategy.md`        | Ngưỡng cảnh báo, SLO, alert routing, giảm noise           |
| `3-rds-enhanced-monitoring.md`  | OS-level metrics, process monitoring, phân tích sâu       |
| `4-dynamodb-monitoring.md`      | DynamoDB CloudWatch alarms, Contributor Insights          |
| `5-activity-streams.md`         | Database Activity Streams, Kinesis, SIEM integration       |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
